# Data Pipeline Architecture — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. Pipeline Philosophy

**Fundamental principle**: External data is a replaceable input. Domain rules are permanent.

The data pipeline's job is to:
1. Fetch raw data from external sources (APIs, scrapers, community packs)
2. Validate against a strict schema
3. Normalize into the internal domain model
4. Version and persist the normalized dataset
5. Publish a "season data ready" event for the game to consume

The game domain **never calls the pipeline at runtime**. Data is pre-materialized.

---

## 2. Architecture Overview

```
External Sources
  ├── Licensed APIs (SportsDB, SportMonks, API-Football)
  ├── Open data (OpenFootball GitHub, football-data.org)
  ├── Community packs (mod system uploads)
  └── Manual editor (admin tool for corrections)
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│                   DATA PIPELINE                              │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐ │
│  │   Ingestor   │──▶│  Validator   │──▶│   Normalizer     │ │
│  │ (per source) │   │  (schema +   │   │ (external →      │ │
│  │              │   │   business)  │   │  internal model) │ │
│  └──────────────┘   └──────────────┘   └────────┬─────────┘ │
│                                                  │           │
│  ┌───────────────────────────────────────────────▼─────────┐ │
│  │                   DATA VERSIONING                        │ │
│  │  DataVersion { season, provider, hash, importedAt }      │ │
│  └───────────────────────────────────────────────┬─────────┘ │
│                                                  │           │
│  ┌───────────────────────────────────────────────▼─────────┐ │
│  │                  PUBLISH STAGE                           │ │
│  │  Writes normalized records to game schema                │ │
│  │  Publishes SeasonDataPublished event                     │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
         │
         ▼
     Game Domain
  (reads from game schema, never calls pipeline)
```

---

## 3. Source Adapters (Anti-Corruption Layer)

Each external source has its own adapter implementing the `DataProvider` port:

```typescript
interface DataProvider {
  fetchPlayers(season: number, league: string): Promise<RawPlayerRecord[]>
  fetchClubs(season: number, country: string): Promise<RawClubRecord[]>
  fetchCompetitions(season: number): Promise<RawCompetitionRecord[]>
}
```

### Available Adapters

| Adapter | Source | Data Coverage | Cost | Legal Risk |
|---|---|---|---|---|
| `SportMonksAdapter` | SportMonks API | 2,500+ leagues, 500k+ players | €499/mo | Low (licensed) |
| `ApiFootballAdapter` | API-Football | 800+ leagues, 300k+ players | $25/mo | Low (licensed) |
| `OpenFootballAdapter` | openfoot.org / GitHub | Major leagues, less depth | Free | Low (public) |
| `FDotOrgAdapter` | football-data.org | Top 12 leagues | Free tier | Low (CC license) |
| `CommunityPackAdapter` | Mod system uploads | Community-defined | Free | Mod author's responsibility |
| `ManualEditAdapter` | Admin tool | Admin-corrected data | Internal | Internal |

### Selection Strategy per Phase

| Phase | Primary Source | Rationale |
|---|---|---|
| 1–2 | Generated (fictitious) | Zero licensing risk, validate engine |
| 3 | API-Football (free tier) + OpenFootball | Cheapest real-data path |
| 4 | SportMonks (production) | Coverage and quality at scale |
| 5+ | Licensed (CBF first) + SportMonks | First-party Brazilian data |

---

## 4. Validation Layer

Two levels of validation before data enters the system:

### 4.1 Schema Validation
```typescript
// Example: player record schema
const rawPlayerSchema = z.object({
  externalId: z.string().min(1),
  name: z.object({
    first: z.string(),
    last: z.string(),
    known: z.string().optional()
  }),
  nationality: z.string().length(2),   // ISO 3166
  dateOfBirth: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  position: z.enum(['GK','CB','LB','RB','CDM','CM','CAM','LW','RW','ST']),
  height: z.number().min(150).max(220),
  weight: z.number().min(50).max(120),
  attributes: z.record(z.number().min(1).max(20))
})
```

### 4.2 Business Validation
```
Player age must be 15–45
CA must be 1–200
PA must be >= CA
Position must be valid for the assigned league type
Club must exist before player can be assigned
Competition format must have minimum club count
```

### Validation Failure Policy
- `ERROR`: Record rejected, logged, pipeline continues
- `WARNING`: Record accepted with flag, human review queue
- `CRITICAL`: Pipeline halts, alert sent to team

---

## 5. Normalization Layer

Translates provider-specific formats to the internal domain model:

```typescript
class PlayerNormalizer {
  normalize(raw: RawPlayerRecord, source: DataSource): NormalizedPlayer {
    return {
      internalId: this.resolveOrCreate(raw.externalId, source),
      name: this.normalizeName(raw.name),
      nationality: this.resolveNationality(raw.nationality),
      dateOfBirth: new Date(raw.dateOfBirth),
      position: this.mapPosition(raw.position, source),
      attributes: this.mapAttributes(raw.attributes, source),
      currentAbility: this.computeCA(raw.attributes, raw.position),
      potentialAbility: this.estimatePA(raw.attributes, raw.age, source)
    }
  }
}
```

### Ability Estimation from Real Data

Real-world player data providers give stats (goals, assists, pass accuracy) — not abstract "finishing = 14" attributes. We must map:

```
finishing   ← goals/shot_on_target_ratio, xG outperformance
passing     ← pass_accuracy, key_passes_per_90
speed       ← sprint_speed data (if available) else age-position estimate
stamina     ← minutes_per_season / available_minutes
CA          ← weighted composite of all mapped attributes + position weight
PA          ← CA + youth bonus based on age curve
```

This mapping is version-controlled and auditable. Different attribute maps per data source.

---

## 6. Data Versioning

Every import produces a versioned dataset:

```sql
pipeline.data_versions {
  id              UUID PRIMARY KEY,
  season_year     SMALLINT NOT NULL,
  provider        TEXT NOT NULL,
  import_hash     TEXT NOT NULL,    -- SHA256 of normalized dataset
  player_count    INTEGER,
  club_count      INTEGER,
  competition_count INTEGER,
  status          TEXT NOT NULL,    -- PENDING | VALIDATING | COMPLETE | FAILED
  imported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at    TIMESTAMPTZ
}
```

### Version Rules
- A new season data version cannot replace the previous until validated
- Active careers can pin to a specific data version (no mid-career attribute changes)
- Multiplayer leagues use the data version active at season start — never updated mid-season

---

## 7. Incremental Update Strategy

Full reimport per season is expensive. For in-season updates:

```
Week 1 of each month:
  1. Fetch injury/suspension updates (lightweight)
  2. Fetch transfer completions (new club assignments)
  3. Do NOT update player attributes mid-season
  4. Publish IncrementalUpdateApplied event
```

Full attribute recalculation happens only at season boundary.

---

## 8. Data Pipeline Execution

```
Trigger: Cron job (every Sunday night, off-peak hours)
         OR: Manual trigger via admin tool
         OR: Webhook from data provider (if supported)

Execution environment:
  - Isolated Docker container (pipeline worker)
  - Dedicated database connection pool (does not share with API)
  - Maximum run time: 4 hours (alert if exceeded)
  - Idempotent: re-running same import produces same result

Pipeline steps:
  1. Fetch (30–120 min depending on source)
  2. Validate (5–15 min)
  3. Normalize (10–30 min)
  4. Diff against previous version (5 min)
  5. Write to staging tables (10 min)
  6. Promote staging to live tables (< 1 min, transactional)
  7. Publish SeasonDataPublished event
  8. Invalidate Redis projections for affected clubs/players
```

---

## 9. Data Quality Monitoring

```
Checks run after every import:
  - Player count delta < 20% vs previous import (alert if exceeded)
  - Average CA distribution within expected range (15–17 mean, σ ≈ 2)
  - No club has more than 40 players assigned
  - All competition clubs present in club dataset
  - Nationality distribution plausible (Brazil > 10% of all players)

Dashboard: Grafana panel showing per-source import health
Alert: Slack/email if any CRITICAL validation fails
```

---

## 10. Licensing and Compliance Checks

Before any data version is published to production:

```
Legal checklist (automated where possible):
  □ Provider license valid and not expired
  □ No player names from FIFPRO-protected list (if no FIFPRO license)
  □ No official competition logos embedded in data
  □ Player image URLs not included (only text attributes)
  □ Data usage within contracted volume limits

If any check fails: import is held in PENDING, alert sent to legal/ops team.
```
