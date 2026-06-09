# Data Architecture — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. Data Storage Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA LAYER OVERVIEW                           │
│                                                                  │
│  PostgreSQL (primary)           Redis (cache + projections)      │
│  ├── identity schema            ├── session:{userId}            │
│  ├── game schema                ├── projection:league:{id}      │
│  ├── events schema              ├── projection:squad:{clubId}   │
│  └── finance schema             ├── leaderboard:{competitionId} │
│                                 └── ratelimit:{ip}              │
│                                                                  │
│  NATS JetStream (event bus)     Object Storage (S3-compatible)  │
│  ├── game.events.*              ├── match-replays/              │
│  ├── match.simulate.*           ├── season-exports/             │
│  └── notifications.*            └── mod-packages/              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. PostgreSQL Schema Design

### Schema Separation by Bounded Context

Each bounded context owns its own schema. Cross-context references use IDs only (no foreign keys across schemas).

```sql
-- Schemas
CREATE SCHEMA identity;    -- User accounts, sessions
CREATE SCHEMA game;        -- Players, clubs, careers, competitions
CREATE SCHEMA events;      -- Domain event store
CREATE SCHEMA finance;     -- Financial ledger (event-sourced)
CREATE SCHEMA pipeline;    -- Data import pipeline state
```

---

### identity schema

```sql
identity.users {
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email        TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role         TEXT NOT NULL DEFAULT 'player',
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at   TIMESTAMPTZ
}

identity.manager_profiles {
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES identity.users(id),
  display_name TEXT NOT NULL,
  reputation   INTEGER NOT NULL DEFAULT 1,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
}

identity.sessions {
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL,
  refresh_token TEXT NOT NULL,
  expires_at    TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
}
```

---

### game schema (key tables)

```sql
game.players {
  id              UUID PRIMARY KEY,
  name_first      TEXT NOT NULL,
  name_last       TEXT NOT NULL,
  name_known_as   TEXT,
  nationality     CHAR(2) NOT NULL,    -- ISO 3166-1 alpha-2
  date_of_birth   DATE NOT NULL,
  height_cm       SMALLINT NOT NULL,
  weight_kg       SMALLINT NOT NULL,
  position_primary TEXT NOT NULL,
  position_secondary TEXT[],
  -- Technical attributes
  attr_finishing      SMALLINT NOT NULL CHECK (attr_finishing BETWEEN 1 AND 20),
  attr_passing        SMALLINT NOT NULL CHECK (attr_passing BETWEEN 1 AND 20),
  attr_crossing       SMALLINT NOT NULL CHECK (attr_crossing BETWEEN 1 AND 20),
  attr_dribbling      SMALLINT NOT NULL CHECK (attr_dribbling BETWEEN 1 AND 20),
  attr_marking        SMALLINT NOT NULL CHECK (attr_marking BETWEEN 1 AND 20),
  attr_tackling       SMALLINT NOT NULL CHECK (attr_tackling BETWEEN 1 AND 20),
  attr_heading        SMALLINT NOT NULL CHECK (attr_heading BETWEEN 1 AND 20),
  attr_ball_control   SMALLINT NOT NULL CHECK (attr_ball_control BETWEEN 1 AND 20),
  -- Physical attributes
  attr_speed          SMALLINT NOT NULL CHECK (attr_speed BETWEEN 1 AND 20),
  attr_acceleration   SMALLINT NOT NULL CHECK (attr_acceleration BETWEEN 1 AND 20),
  attr_stamina        SMALLINT NOT NULL CHECK (attr_stamina BETWEEN 1 AND 20),
  attr_strength       SMALLINT NOT NULL CHECK (attr_strength BETWEEN 1 AND 20),
  attr_jumping        SMALLINT NOT NULL CHECK (attr_jumping BETWEEN 1 AND 20),
  -- Mental attributes
  attr_leadership     SMALLINT NOT NULL CHECK (attr_leadership BETWEEN 1 AND 20),
  attr_concentration  SMALLINT NOT NULL CHECK (attr_concentration BETWEEN 1 AND 20),
  attr_determination  SMALLINT NOT NULL CHECK (attr_determination BETWEEN 1 AND 20),
  attr_decisions      SMALLINT NOT NULL CHECK (attr_decisions BETWEEN 1 AND 20),
  attr_positioning    SMALLINT NOT NULL CHECK (attr_positioning BETWEEN 1 AND 20),
  attr_teamwork       SMALLINT NOT NULL CHECK (attr_teamwork BETWEEN 1 AND 20),
  -- Ability
  current_ability  SMALLINT NOT NULL,
  potential_ability SMALLINT NOT NULL,   -- NEVER returned in API responses
  -- State
  morale          TEXT NOT NULL DEFAULT 'NEUTRAL',
  fatigue         SMALLINT NOT NULL DEFAULT 0,
  form            NUMERIC(4,2) NOT NULL DEFAULT 5.0,
  injury_type     TEXT,
  injury_return   DATE,
  -- Meta
  version         INTEGER NOT NULL DEFAULT 0,  -- optimistic locking
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
}

game.clubs {
  id                  UUID PRIMARY KEY,
  name                TEXT NOT NULL,
  short_name          CHAR(3) NOT NULL,
  country             CHAR(2) NOT NULL,
  city                TEXT NOT NULL,
  founded_year        SMALLINT,
  reputation          SMALLINT NOT NULL DEFAULT 50,
  league_tier         SMALLINT NOT NULL DEFAULT 3,
  stadium_capacity    INTEGER NOT NULL DEFAULT 20000,
  training_grade      SMALLINT NOT NULL DEFAULT 2,
  youth_grade         SMALLINT NOT NULL DEFAULT 2,
  medical_grade       SMALLINT NOT NULL DEFAULT 2,
  version             INTEGER NOT NULL DEFAULT 0,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
}

game.contracts {
  id            UUID PRIMARY KEY,
  player_id     UUID NOT NULL,   -- no FK across schemas; validated at app layer
  club_id       UUID NOT NULL,
  wage_weekly   NUMERIC(12,2) NOT NULL,
  start_date    DATE NOT NULL,
  expiry_date   DATE NOT NULL,
  clauses       JSONB NOT NULL DEFAULT '[]',
  status        TEXT NOT NULL DEFAULT 'ACTIVE',   -- ACTIVE | EXPIRED | TERMINATED
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
}

game.careers {
  id              UUID PRIMARY KEY,
  manager_id      UUID NOT NULL,
  club_id         UUID NOT NULL,
  season_year     SMALLINT NOT NULL,
  status          TEXT NOT NULL DEFAULT 'ACTIVE',
  difficulty      TEXT NOT NULL DEFAULT 'NORMAL',
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  ended_at        TIMESTAMPTZ
}

game.competitions {
  id                   UUID PRIMARY KEY,
  name                 TEXT NOT NULL,
  short_name           TEXT NOT NULL,
  format               TEXT NOT NULL,    -- LEAGUE | CUP | GROUP_KNOCKOUT
  region               TEXT NOT NULL,
  tier                 SMALLINT NOT NULL,
  promotion_spots      SMALLINT NOT NULL DEFAULT 0,
  relegation_spots     SMALLINT NOT NULL DEFAULT 0,
  continental_spots    SMALLINT NOT NULL DEFAULT 0
}

game.seasons {
  id                   UUID PRIMARY KEY,
  competition_id       UUID NOT NULL,
  year                 SMALLINT NOT NULL,
  status               TEXT NOT NULL DEFAULT 'PENDING',
  phase                TEXT NOT NULL DEFAULT 'PRE_SEASON',
  started_at           TIMESTAMPTZ,
  ended_at             TIMESTAMPTZ,
  UNIQUE (competition_id, year)
}

game.fixtures {
  id                UUID PRIMARY KEY,
  season_id         UUID NOT NULL,
  match_day         SMALLINT NOT NULL,
  home_club_id      UUID NOT NULL,
  away_club_id      UUID NOT NULL,
  scheduled_date    DATE NOT NULL,
  home_score        SMALLINT,
  away_score        SMALLINT,
  match_seed        UUID,
  simulated_at      TIMESTAMPTZ,
  status            TEXT NOT NULL DEFAULT 'PENDING'
}

game.league_standings {
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  season_id       UUID NOT NULL,
  club_id         UUID NOT NULL,
  played          SMALLINT NOT NULL DEFAULT 0,
  won             SMALLINT NOT NULL DEFAULT 0,
  drawn           SMALLINT NOT NULL DEFAULT 0,
  lost            SMALLINT NOT NULL DEFAULT 0,
  goals_for       SMALLINT NOT NULL DEFAULT 0,
  goals_against   SMALLINT NOT NULL DEFAULT 0,
  points          SMALLINT NOT NULL DEFAULT 0,
  UNIQUE (season_id, club_id)
}
```

---

### events schema (Event Store)

```sql
events.domain_events {
  id              BIGSERIAL PRIMARY KEY,
  event_id        UUID NOT NULL UNIQUE,
  stream_id       TEXT NOT NULL,           -- e.g., "match:abc-123"
  event_type      TEXT NOT NULL,           -- e.g., "MatchSimulated"
  aggregate_type  TEXT NOT NULL,           -- e.g., "Match"
  aggregate_id    UUID NOT NULL,
  payload         JSONB NOT NULL,
  metadata        JSONB NOT NULL DEFAULT '{}',
  occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  sequence_number BIGINT NOT NULL
}

-- Index for stream replay
CREATE INDEX ON events.domain_events (stream_id, sequence_number);
-- Index for projections
CREATE INDEX ON events.domain_events (event_type, occurred_at);
-- Partition by month for large tables
CREATE TABLE events.domain_events_2026_01 PARTITION OF events.domain_events
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

---

### finance schema (Event-Sourced)

```sql
finance.ledger_events {
  id          BIGSERIAL PRIMARY KEY,
  event_id    UUID NOT NULL UNIQUE,
  club_id     UUID NOT NULL,
  type        TEXT NOT NULL,    -- REVENUE_COLLECTED | WAGE_PAID | TRANSFER_FEE | etc.
  amount      NUMERIC(14,2) NOT NULL,
  direction   TEXT NOT NULL,    -- CREDIT | DEBIT
  reference   TEXT,             -- e.g., matchId, transferId
  season_year SMALLINT NOT NULL,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
}

-- Materialized view: current balance per club
CREATE MATERIALIZED VIEW finance.club_balances AS
SELECT
  club_id,
  SUM(CASE WHEN direction = 'CREDIT' THEN amount ELSE -amount END) as balance,
  MAX(occurred_at) as last_updated
FROM finance.ledger_events
GROUP BY club_id;
```

---

## 3. Redis Key Design

```
# Sessions
session:{userId}                → JWT refresh token + metadata (TTL: 30 days)

# Projections (read models, computed from events)
projection:squad:{clubId}       → JSON squad summary (TTL: 10 min)
projection:league:{seasonId}    → JSON league table (TTL: 1 min in active season)
projection:market:{windowId}    → JSON transfer listings (TTL: 30 sec)

# Leaderboards
leaderboard:reputation          → Sorted set: managerId → reputation score
leaderboard:goals:{seasonId}    → Sorted set: playerId → goals scored

# Rate limiting
ratelimit:{ip}                  → counter (TTL: 60 sec)
ratelimit:user:{userId}         → counter (TTL: 60 sec)

# Multiplayer gates
gate:{matchDayId}:submitted     → Set of managerIds who submitted
gate:{matchDayId}:deadline      → timestamp (TTL: 48h)

# Pub/sub channels (for WebSocket distribution)
channel:league:{leagueId}
channel:user:{managerId}
```

---

## 4. CQRS Read/Write Model Separation

```
Write Model (Commands → Aggregates → Events → PostgreSQL)
  └── Normalized tables, transactional writes, event log

Read Model (Queries → Projections → Redis or PostgreSQL views)
  └── Denormalized, optimized for query patterns

Projection update strategies:
  SYNC:  Updated in same transaction as write (for critical projections)
  ASYNC: Updated by event subscriber (for most projections, eventual consistency)
  MATERIALIZED: PostgreSQL materialized view refreshed every 5 min
```

### Key Read Models and Where They Live

| Read Model | Storage | Update Trigger | Staleness |
|---|---|---|---|
| League Table | Redis (hot) + PG (authoritative) | MatchSimulated event | < 1 min |
| Squad Summary | Redis | ContractSigned/Expired | < 10 min |
| Transfer Market | Redis | Offer events | < 30 sec |
| Player Profile | PostgreSQL query | PlayerAttributeChanged | Instant |
| Financial Balance | PG Materialized View | Ledger event | 5 min refresh |
| Match Report | PostgreSQL | MatchSimulated (written once) | Immutable |

---

## 5. Data Partitioning Strategy

```
events.domain_events    → Partitioned by month (range on occurred_at)
game.fixtures           → Partitioned by season_year
finance.ledger_events   → Partitioned by season_year + club_id (hash)

Retention policy:
  - domain_events: keep forever (audit + replay)
  - finance.ledger_events: keep forever (legal)
  - game.fixtures: keep forever (history)
  - redis projections: TTL per key type (see above)
```

---

## 6. Data Growth Projections

| Table | Rows/year (10k users) | Rows/year (1M users) |
|---|---|---|
| domain_events | ~5M | ~500M |
| fixtures | ~200k | ~20M |
| ledger_events | ~2M | ~200M |
| players | ~500k (mostly static) | ~2M |
| league_standings | ~50k | ~5M |

At 1M users, domain_events requires partitioning + archival strategy. PostgreSQL handles this with partition pruning — queries automatically skip irrelevant partitions.
