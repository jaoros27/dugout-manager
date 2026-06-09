# Bounded Contexts — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## Context Map

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DUGOUT MANAGER                                  │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │   Identity   │    │   Career     │    │    Club Management       │  │
│  │   Context    │───▶│   Context    │───▶│    Context               │  │
│  └──────────────┘    └──────────────┘    └──────────────────────────┘  │
│                             │                         │                 │
│                             ▼                         ▼                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  Competition │◀───│    Season    │    │   Transfer Market        │  │
│  │   Context    │    │   Context    │    │   Context                │  │
│  └──────────────┘    └──────────────┘    └──────────────────────────┘  │
│         │                   │                         │                 │
│         ▼                   ▼                         ▼                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │    Match     │    │   Finance    │    │    Player Development    │  │
│  │   Engine     │    │   Context    │    │    Context               │  │
│  └──────────────┘    └──────────────┘    └──────────────────────────┘  │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  Multiplayer │    │ Data Pipeline│    │    Mod System            │  │
│  │   Context    │    │   Context    │    │    Context               │  │
│  └──────────────┘    └──────────────┘    └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## BC-01 — Identity Context

**Responsibility**: User authentication, authorization, profile management.

**Owns**:
- User (email, passwordHash, role, createdAt)
- Session (token, refreshToken, expiresAt)
- ManagerProfile (displayName, avatarUrl, reputation)

**Publishes**:
- `UserRegistered`
- `UserAuthenticated`

**Consumes**:
- Nothing (upstream, no dependencies on game domain)

**External dependencies**:
- Email provider (registration, password reset)
- OAuth providers (optional: Google, Discord)

**Note**: Identity is intentionally isolated. The game domain knows a `managerId` (UUID), never a `User` entity. This allows identity to be swapped without touching game logic.

---

## BC-02 — Career Context

**Responsibility**: The manager's game state — their ongoing journey within a single save.

**Owns**:
- Career (managerId, clubId, seasonId, startDate, status)
- CareerSettings (difficulty, simulationSpeed)

**Publishes**:
- `CareerStarted`
- `CareerEnded`

**Consumes**:
- `ClubAssigned` (from Club Management)
- `SeasonEnded` (from Season Context)

**Anti-Corruption Layer**:
- Translates Club domain model into Career's local view (CareerClubSnapshot)

**Note**: Career is the "session" aggregate. In multiplayer, a Career maps to one manager slot in a MultiplayerLeague.

---

## BC-03 — Club Management Context

**Responsibility**: Everything about a football club as an organizational unit.

**Owns**:
- Club (name, reputation, tier, stadiumCapacity, facilities)
- Squad (clubId, players[], coachingStaff[])
- CoachingStaff (headCoach, assistants, fitness, youth, goalkeeper)
- Contract (playerId, clubId, wage, expiresAt, clauses[])

**Publishes**:
- `PlayerAddedToSquad`
- `PlayerRemovedFromSquad`
- `ContractSigned`
- `ContractExpired`
- `CoachHired`
- `CoachFired`

**Consumes**:
- `TransferCompleted` (from Transfer Market)
- `PlayerAgedUp` (from Player Development)
- `SeasonStarted` (from Season)

**Key Invariant**: Squad wage bill must not exceed board-approved wage budget.

---

## BC-04 — Player Domain Context

**Responsibility**: The canonical player model — attributes, state, history.

**Owns**:
- Player (id, name, nationality, dateOfBirth, position, attributes, state)
- PlayerAttributes (technical[], physical[], mental[])
- PlayerState (morale, fatigue, form, injuryStatus)
- PlayerHistory (clubHistory[], seasonStats[])

**Publishes**:
- `PlayerCreated`
- `PlayerAttributeChanged`
- `PlayerInjured`
- `PlayerRecovered`
- `PlayerRetired`

**Consumes**:
- `TrainingSessionCompleted` (from Training)
- `MatchSimulated` (from Match Engine — for form/fatigue updates)
- `PlayerAgedUp` (internal)

**Key Invariant**: CA is always ≤ PA. Attributes cannot exceed position-specific maximums.

**Note**: Player is the most cross-cut aggregate. All other contexts reference players by ID only — they never embed the full Player entity.

---

## BC-05 — Transfer Market Context

**Responsibility**: Negotiation and execution of player transfers between clubs.

**Owns**:
- TransferListing (playerId, askingPrice, type, windowId)
- TransferOffer (offerId, buyerClubId, sellerClubId, playerId, amount, status)
- TransferNegotiation (offerId, rounds[], currentStatus)
- TransferWindow (id, type, openDate, closeDate)

**Publishes**:
- `TransferOfferSubmitted`
- `TransferOfferAccepted`
- `TransferOfferRejected`
- `TransferCompleted`
- `LoanAgreed`
- `TransferWindowOpened`
- `TransferWindowClosed`

**Consumes**:
- `TransferWindowOpened/Closed` (from Season)
- `ClubBudgetUpdated` (from Finance)

**Key Invariant**: Transfer cannot complete outside an open transfer window (except free transfers / pre-contracts).

**Multiplayer note**: In shared leagues, TransferOffer must be processed with optimistic concurrency — two managers bidding simultaneously must result in exactly one winner.

---

## BC-06 — Finance Context

**Responsibility**: Financial ledger for each club — income, expenses, budget management.

**Owns**:
- FinancialLedger (clubId, entries[])
- LedgerEntry (type, amount, date, reference)
- Budget (clubId, transferBudget, wageBudget, season)
- FinancialReport (clubId, season, revenue, expenses, profit)

**Publishes**:
- `RevenueCollected`
- `ExpenseRecorded`
- `BudgetUpdated`
- `FinancialWarningIssued`
- `ClubEnteredAdministration`
- `PrizeMoneyDistributed`

**Consumes**:
- `MatchSimulated` (for ticket revenue calculation)
- `TransferCompleted` (for fee recording)
- `ContractSigned` (for wage obligations)
- `SeasonEnded` (for prize money)

**Key Invariant**: Wage obligations never silently fail — warnings cascade to administration if unresolved.

---

## BC-07 — Competition Context

**Responsibility**: Competition definitions, fixtures, standings, and results.

**Owns**:
- Competition (id, name, type, tier, region)
- Season (competitionId, year, clubs[], fixtures[], standings)
- Fixture (id, homeClubId, awayClubId, date, matchDayId, result)
- LeagueTable (competitionId, season, rows[])
- KnockoutDraw (competitionId, rounds[], pairs[])

**Publishes**:
- `FixtureScheduled`
- `MatchResultRecorded`
- `LeagueTableUpdated`
- `CompetitionWinnerDetermined`
- `PromotionConfirmed`
- `RelegationConfirmed`

**Consumes**:
- `MatchSimulated` (from Match Engine)
- `NewSeasonStarted` (from Season)

**Key Invariant**: A fixture result can only be recorded once. No replays unless explicitly authorized.

---

## BC-08 — Match Engine Context

**Responsibility**: Stateless simulation of a single football match given two club states and tactical configurations.

**Owns** (ephemeral, not persisted):
- MatchContext (homeLineup, awayLineup, homeTactics, awayTactics, conditions)
- MatchEvent (type, minute, player, outcome)
- MatchResult (homeScore, awayScore, events[], stats[], seed)

**Publishes**:
- `MatchSimulated` (with full event log and seed for replay)

**Consumes**:
- Nothing persistent — fully stateless, all inputs passed in command

**Key invariants**:
- Given the same seed and inputs, must produce identical output
- Must never read from or write to a database during simulation
- Must be runnable offline (no network calls)

**Note**: This is the most critical and most tested context. The match engine is a pure function: `simulate(matchContext, seed) → MatchResult`.

---

## BC-09 — Season Context

**Responsibility**: Orchestrates time progression across a single career or multiplayer league.

**Owns**:
- GameCalendar (seasonId, weeks[], events[])
- SeasonPhase (PRE_SEASON | SEASON | TRANSFER_WINDOW | POST_SEASON)
- MatchDay (id, week, fixtures[])

**Publishes**:
- `WeekAdvanced`
- `SeasonStarted`
- `SeasonEnded`
- `TransferWindowOpened`
- `TransferWindowClosed`
- `PreSeasonStarted`

**Consumes**:
- `LineupSubmitted` (in multiplayer — gates week advancement)

**Key Invariant**: In multiplayer, week cannot advance until all managers have submitted lineups or the deadline has passed.

---

## BC-10 — Player Development Context

**Responsibility**: Training logic, attribute progression, age curves, and youth generation.

**Owns**:
- TrainingPlan (clubId, week, schedule[])
- TrainingSession (clubId, focus, intensity, date)
- DevelopmentCurve (position, peakAge, regressionOnset)
- YouthIntake (clubId, season, players[])

**Publishes**:
- `TrainingSessionCompleted`
- `PlayerAttributeImproved`
- `PlayerAttributeDeclined`
- `YouthPlayerGenerated`

**Consumes**:
- `WeekAdvanced` (triggers training)
- `SeasonEnded` (triggers youth intake)

---

## BC-11 — Multiplayer Context

**Responsibility**: Multi-manager league coordination, turn management, and shared state synchronization.

**Owns**:
- MultiplayerLeague (id, managers[], settings, status)
- LeagueSeason (leagueId, seasonId, matchDayQueue[])
- ManagerSlot (leagueId, managerId, clubId, isReady, lastActivity)
- MatchDayGate (matchDayId, submissionsReceived[], allReady, deadline)

**Publishes**:
- `LeagueCreated`
- `ManagerJoinedLeague`
- `MatchDayGateOpened`
- `MatchDayGateClosed`
- `MultiplayerMatchDayProcessed`

**Consumes**:
- `LineupSubmitted` (gates match day processing)
- `SeasonEnded` (triggers next season setup)

---

## BC-12 — Data Pipeline Context

**Responsibility**: Importing, validating, normalizing, and versioning external data (players, clubs, competitions).

**Owns**:
- DataImportJob (id, provider, status, createdAt)
- RawDataRecord (jobId, source, payload, validationStatus)
- NormalizedRecord (type, externalId, internalId, data, version)
- DataVersion (season, provider, importedAt, recordCount)

**Publishes**:
- `DataImportCompleted`
- `DataValidationFailed`
- `SeasonDataPublished`

**Consumes**:
- External data providers (via Anti-Corruption Layer adapters)

**Note**: This context is entirely separate from the game domain. The game never calls the data pipeline at runtime — data is pre-imported and materialized into the Player/Club domains before gameplay begins.

---

## BC-13 — Mod System Context

**Responsibility**: Community-created content distribution, validation, and loading.

**Owns**:
- ModPackage (id, type, author, version, schema, content)
- ModRegistry (published mods, ratings, downloads)
- ModValidationResult (modId, errors[], warnings[], status)

**Publishes**:
- `ModPublished`
- `ModValidationPassed`
- `ModValidationFailed`

**Consumes**:
- Data Pipeline events (mods follow same normalization rules)

---

## Context Relationships Summary

| From | To | Relationship |
|---|---|---|
| Career | Club Management | Partnership (Career owns the manager's view of the club) |
| Club Management | Player Domain | Customer/Supplier (Club consumes Player entities by ID) |
| Transfer Market | Finance | Customer/Supplier (Transfer fees recorded by Finance) |
| Transfer Market | Club Management | Customer/Supplier (Squad updated after transfer) |
| Match Engine | Player Domain | Anti-Corruption Layer (reads a snapshot, never writes) |
| Competition | Match Engine | Customer/Supplier (Competition requests simulation) |
| Season | All contexts | Conformist (all contexts react to season time events) |
| Data Pipeline | Player Domain | Anti-Corruption Layer (translates external → internal model) |
| Multiplayer | Season | Partnership (multiplayer gates season progression) |
| Mod System | Data Pipeline | Conformist (mods go through same pipeline) |
