# Domain Model — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## Core Aggregates

### Player (Aggregate Root)

```
Player {
  id: PlayerId                     // UUID, immutable
  personalInfo: PersonalInfo       // Value Object
  position: Position               // Value Object (enum)
  attributes: PlayerAttributes     // Value Object
  state: PlayerState               // Value Object (mutable per week)
  ability: PlayerAbility           // Value Object (CA + PA)
  contract: Contract | null        // Entity
}

PersonalInfo {
  name: PlayerName                 // Value Object (first, last, known as)
  nationality: Nationality         // Value Object (ISO 3166)
  dateOfBirth: Date
  height: Centimeters              // 150–210
  weight: Kilograms                // 55–110
}

Position {
  primary: PositionCode            // GK | CB | LB | RB | CDM | CM | CAM | LW | RW | ST
  secondary: PositionCode[]        // max 2 secondary positions
}

PlayerAttributes {
  technical: TechnicalAttributes
  physical: PhysicalAttributes
  mental: MentalAttributes
}

TechnicalAttributes {
  finishing:    Attribute  // 1–20
  passing:      Attribute  // 1–20
  crossing:     Attribute  // 1–20
  dribbling:    Attribute  // 1–20
  marking:      Attribute  // 1–20
  tackling:     Attribute  // 1–20
  heading:      Attribute  // 1–20
  ballControl:  Attribute  // 1–20
}

PhysicalAttributes {
  speed:        Attribute  // 1–20
  acceleration: Attribute  // 1–20
  stamina:      Attribute  // 1–20
  strength:     Attribute  // 1–20
  jumping:      Attribute  // 1–20
}

MentalAttributes {
  leadership:    Attribute  // 1–20
  concentration: Attribute  // 1–20
  determination: Attribute  // 1–20
  decisions:     Attribute  // 1–20
  positioning:   Attribute  // 1–20
  teamwork:      Attribute  // 1–20
}

Attribute: newtype(integer, range: 1–20)  // cannot be constructed outside range

PlayerState {
  morale:       MoraleLevel     // VERY_LOW | LOW | NEUTRAL | HIGH | VERY_HIGH
  fatigue:      Percentage      // 0–100 (0 = fresh, 100 = exhausted)
  form:         FormRating      // 1–10 rolling average of recent performances
  injury:       Injury | null
}

PlayerAbility {
  currentAbility:   CA  // 1–200, derived from weighted attributes
  potentialAbility: PA  // 1–200, hidden from API, never exposed to client
}

Injury {
  type:           InjuryType    // MINOR | MODERATE | SEVERE | CAREER_ENDING
  bodyPart:       BodyPart
  occurredAt:     Date
  expectedReturn: Date
}
```

**Invariants**:
- `CA <= PA` always
- `fatigue` cannot exceed 100 after a single match
- `injury != null` implies player is unavailable for selection
- `PA` is never modified after creation (it is destiny, not outcome)
- Attributes in range 1–20; `Attribute` is a value object that enforces this

---

### Club (Aggregate Root)

```
Club {
  id:           ClubId
  identity:     ClubIdentity       // Value Object
  tier:         LeagueTier         // 1 (top) → 5 (amateur)
  reputation:   Reputation         // 1–100 scalar
  facilities:   ClubFacilities     // Value Object
  finances:     ClubFinanceSnapshot // Value Object (denormalized read)
  squad:        Squad              // Entity
}

ClubIdentity {
  name:         ClubName
  shortName:    Abbreviation       // max 3 chars
  founded:      Year
  city:         City
  country:      Country
}

ClubFacilities {
  stadiumCapacity: integer          // 1,000–100,000
  trainingGrade:   FacilityGrade   // 1–5
  youthGrade:      FacilityGrade   // 1–5
  medicalGrade:    FacilityGrade   // 1–5
}

Squad {
  players:     SquadPlayer[]       // max 40 registered
  coaching:    CoachingStaff
}

SquadPlayer {
  playerId:   PlayerId
  jerseyNumber: integer
  squadRole:  SquadRole            // FIRST_TEAM | ROTATION | YOUTH | LOANED_OUT
}
```

**Invariants**:
- Squad max 40 players
- At least 16 fit first-team players required for match day (else forfeit)
- Cannot register same player twice in same squad

---

### Contract (Entity, owned by Club)

```
Contract {
  id:            ContractId
  playerId:      PlayerId
  clubId:        ClubId
  wage:          Money              // weekly wage
  startDate:     Date
  expiryDate:    Date
  clauses:       ContractClause[]
}

ContractClause {
  type: SELL_ON_PERCENTAGE | RELEASE_CLAUSE | LOAN_RECALL | BUY_OPTION
  value: Money | Percentage
}
```

**Invariants**:
- A player can have at most one active contract per club
- Release clause must be > current market value to be valid
- Expired contract transitions player to free agent status

---

### Match (Aggregate, ephemeral in engine — persisted as event log)

```
Match {
  id:           MatchId
  fixture:      FixtureRef
  homeLineup:   Lineup
  awayLineup:   Lineup
  conditions:   MatchConditions
  seed:         MatchSeed          // reproducibility
  events:       MatchEvent[]       // ordered by minute
  result:       MatchResult
}

Lineup {
  clubId:       ClubId
  startingXI:   PlayerSlot[]       // exactly 11
  substitutes:  PlayerSlot[]       // max 9
  tactics:      TacticsSnapshot
}

PlayerSlot {
  playerId:     PlayerId
  position:     FieldPosition      // actual position played
  captain:      boolean
}

MatchConditions {
  venue:        VenueType          // HOME | AWAY | NEUTRAL
  weather:      Weather            // CLEAR | RAIN | SNOW | WIND
  pitchQuality: PitchQuality       // PERFECT | GOOD | POOR | WATERLOGGED
  attendance:   integer
}

MatchEvent {
  minute:       integer            // 1–120 (inc. stoppage + extra time)
  type:         EventType          // GOAL | YELLOW | RED | INJURY | SUBSTITUTION | SAVE | MISS
  actorId:      PlayerId
  assistId:     PlayerId | null
  metadata:     EventMetadata
}

MatchResult {
  homeScore:    integer
  awayScore:    integer
  homeStats:    MatchStats
  awayStats:    MatchStats
}

MatchStats {
  possession:   Percentage
  shots:        integer
  shotsOnTarget: integer
  corners:      integer
  fouls:        integer
  yellowCards:  integer
  redCards:     integer
}
```

---

### Competition (Aggregate Root)

```
Competition {
  id:           CompetitionId
  definition:   CompetitionDefinition
  format:       CompetitionFormat       // LEAGUE | CUP | GROUP_KNOCKOUT
  region:       Region
  tier:         integer
}

CompetitionDefinition {
  name:         string
  shortName:    string
  clubs:        integer               // min clubs for this format
  promotionSpots: integer
  relegationSpots: integer
  continentalSpots: integer
}
```

---

### Season (Aggregate Root)

```
Season {
  id:           SeasonId
  competitionId: CompetitionId
  year:         integer
  clubs:        ClubId[]
  fixtures:     Fixture[]
  standings:    Standing[]
  phase:        SeasonPhase
}

Fixture {
  id:           FixtureId
  matchDay:     integer
  homeClubId:   ClubId
  awayClubId:   ClubId
  scheduledDate: Date
  result:       MatchResult | null
}

Standing {
  clubId:       ClubId
  played:       integer
  won:          integer
  drawn:        integer
  lost:         integer
  goalsFor:     integer
  goalsAgainst: integer
  points:       integer
}
```

---

### Transfer (Aggregate Root)

```
Transfer {
  id:           TransferId
  type:         TransferType    // PERMANENT | LOAN | FREE | PRE_CONTRACT
  buyerClubId:  ClubId
  sellerClubId: ClubId | null   // null for free agent
  playerId:     PlayerId
  fee:          Money
  status:       TransferStatus  // PENDING | NEGOTIATING | AGREED | COMPLETED | REJECTED
  window:       TransferWindowId
  agreedAt:     Date | null
}
```

---

## Domain Services

### MatchSimulationService
```
simulate(context: MatchContext, seed: MatchSeed): MatchResult
```
Pure function. No side effects. No I/O. Deterministic given same seed.

### PlayerValuationService
```
estimate(player: Player, market: MarketConditions): Money
```
Domain service computing transfer value from CA, age, position, contract length, and inflation.

### AbilityComputationService
```
computeCA(attributes: PlayerAttributes, position: Position): CA
```
Weighted sum by position profile. GK weights technical attributes differently than ST.

### DevelopmentProgressionService
```
applyWeeklyDevelopment(player: Player, training: TrainingSession, weekOfSeason: integer): PlayerAttributeDeltas
```
Returns attribute deltas. Caller applies them. Service has no side effects.

### EconomicBalanceService
```
validateEconomy(clubs: Club[], season: integer): EconomicHealthReport
```
Runs inflation, Gini coefficient, and dominance checks. Used in engine validation suite.

---

## Value Objects Summary

| Value Object | Rule |
|---|---|
| `Attribute` | integer 1–20, immutable once set |
| `CA` / `PA` | integer 1–200; CA ≤ PA invariant enforced at creation |
| `Money` | non-negative decimal with currency code; arithmetic operations return new Money |
| `Percentage` | decimal 0–100; cannot be constructed outside range |
| `MatchSeed` | UUID v4 generated at match creation; never changed |
| `MoraleLevel` | enum with 5 levels; transitions only via defined domain events |
| `FormRating` | rolling average of last 5 performances, 1–10 |

---

## Domain Events Summary

| Event | Aggregate | Key Payload |
|---|---|---|
| `PlayerCreated` | Player | playerId, attributes, position |
| `PlayerAttributeChanged` | Player | playerId, attribute, oldValue, newValue |
| `PlayerInjured` | Player | playerId, injuryType, expectedReturn |
| `PlayerRecovered` | Player | playerId, recoveredAt |
| `PlayerRetired` | Player | playerId, retiredAt |
| `ContractSigned` | Club | playerId, clubId, wage, expiry |
| `ContractExpired` | Club | playerId, clubId |
| `TransferCompleted` | Transfer | playerId, from, to, fee |
| `MatchSimulated` | Match | matchId, result, events[], seed |
| `SeasonStarted` | Season | seasonId, competitionId, year |
| `SeasonEnded` | Season | seasonId, standings[], winner |
| `ClubPromoted` | Season | clubId, fromTier, toTier |
| `ClubRelegated` | Season | clubId, fromTier, toTier |
| `FinancialWarningIssued` | Finance | clubId, deficit |
| `ClubEnteredAdministration` | Finance | clubId |
