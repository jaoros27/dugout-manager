# Database Strategy — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. Technology Choices

| Database | Role | Rationale |
|---|---|---|
| PostgreSQL 16+ | Primary write store, event store, finance ledger | ACID, JSONB, partitioning, row-level locking, extensible |
| Redis 7+ | Sessions, read projections, leaderboards, pub/sub | Sub-millisecond reads, TTL, sorted sets, lightweight pub/sub |

No other databases are introduced without an ADR justifying the addition.

---

## 2. PostgreSQL Schema Strategy

### Principle: Schema-per-Context

Each bounded context owns one or more PostgreSQL schemas. Cross-context reads use views. Cross-context writes use events (never direct table access).

```
identity    — user accounts, sessions, manager profiles
game        — core game entities: players, clubs, careers, fixtures, standings
events      — append-only domain event log
finance     — financial ledger (event-sourced, append-only)
pipeline    — data import state and versioning
```

### Connection Pooling

| Phase | Connection Pool |
|---|---|
| 1–2 | Direct connections (max 20) |
| 3 | PgBouncer in transaction mode (pool_size=50) |
| 4+ | PgBouncer in transaction mode (pool_size=200) + read replicas |

PgBouncer is mandatory from Phase 3 — PostgreSQL's connection overhead at scale is a known bottleneck.

---

## 3. Indexing Strategy

### Critical Indexes

```sql
-- Player lookup (most common query in transfer market)
CREATE INDEX CONCURRENTLY idx_players_position_ca
  ON game.players (position_primary, current_ability DESC);

-- Contract expiry (weekly job)
CREATE INDEX CONCURRENTLY idx_contracts_expiry
  ON game.contracts (expiry_date, status)
  WHERE status = 'ACTIVE';

-- Fixture scheduling
CREATE INDEX CONCURRENTLY idx_fixtures_season_matchday
  ON game.fixtures (season_id, match_day, status);

-- Event log stream replay
CREATE INDEX CONCURRENTLY idx_events_stream
  ON events.domain_events (stream_id, sequence_number);

-- Event log projections
CREATE INDEX CONCURRENTLY idx_events_type_time
  ON events.domain_events (event_type, occurred_at DESC);

-- Finance ledger per club
CREATE INDEX CONCURRENTLY idx_ledger_club_time
  ON finance.ledger_events (club_id, occurred_at DESC);

-- Standings sort (constant access during active season)
CREATE INDEX CONCURRENTLY idx_standings_season_points
  ON game.league_standings (season_id, points DESC, goals_for DESC);
```

---

## 4. Optimistic Locking

All write aggregates use a version column for optimistic concurrency:

```sql
-- Example: update player attributes
UPDATE game.players
SET attr_finishing = 15,
    current_ability = 142,
    version = version + 1,
    updated_at = now()
WHERE id = :playerId
  AND version = :expectedVersion;

-- 0 rows affected = concurrent modification → throw OptimisticLockException
```

This eliminates pessimistic locking overhead while preventing lost updates.

---

## 5. Transactional Integrity Rules

1. **Match day results** committed in a single transaction per match day (atomic)
2. **Transfers** use a serializable transaction for the "accept offer" step
3. **Wage payments** processed in batch with a single transaction per club per week
4. **Event writes** always co-committed with state writes (same transaction)
5. **Financial ledger** append-only — never UPDATE/DELETE, only INSERT

---

## 6. Read Replica Strategy

### Phase 3+

```
Primary (writes): career saves, transfers, match results, events
Replica 1 (reads): transfer market queries, player search, standings
Replica 2 (reads): analytics, admin dashboard, data pipeline reads
```

Lag SLA: replicas must be < 5 seconds behind primary for game-critical reads. For non-critical analytics, up to 60 seconds is acceptable.

Application routing:
- All commands → primary
- CQRS queries → replica (with retry to primary on replication lag detected)

---

## 7. Partitioning Strategy

```sql
-- Events partitioned by month (append-heavy)
CREATE TABLE events.domain_events (
  id             BIGSERIAL,
  occurred_at    TIMESTAMPTZ NOT NULL,
  ...
) PARTITION BY RANGE (occurred_at);

-- Auto-partition creation via pg_partman (or manual quarterly)
CREATE TABLE events.domain_events_2026_q1
  PARTITION OF events.domain_events
  FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');

-- Fixtures partitioned by season_year (query pattern: always per season)
CREATE TABLE game.fixtures (
  ...
  season_year SMALLINT NOT NULL
) PARTITION BY LIST (season_year);
```

Partitioning starts at Phase 4. Before that, indexes are sufficient.

---

## 8. Migration Strategy

- All schema changes via Flyway or custom migration runner
- Migrations are sequential and versioned: `V001__create_players.sql`
- Destructive migrations (DROP COLUMN, DROP TABLE) are preceded by a deprecation migration
- Zero-downtime migrations use `ADD COLUMN ... DEFAULT NULL` first, then backfill, then add NOT NULL constraint
- Never run migrations on application startup in production (separate CI step)

---

## 9. Backup Strategy

| Phase | Strategy |
|---|---|
| 1–2 | pg_dump daily, stored locally |
| 3 | pg_dump daily to S3, WAL archival for PITR, 30-day retention |
| 4+ | Continuous WAL streaming, PITR to 5-minute precision, 90-day retention, multi-region |

RTO target (Phase 4): < 30 minutes  
RPO target (Phase 4): < 5 minutes

---

## 10. Redis Strategy

### Key Patterns

```
# Sessions (TTL: 30 days)
session:{userId} → JWT refresh + metadata

# Read projections (TTL: varies)
proj:squad:{clubId}         → 10 min TTL, invalidated on squad change
proj:standings:{seasonId}   → 1 min TTL during active season
proj:market:{windowId}      → 30 sec TTL, high churn during transfer window
proj:player:{playerId}      → 5 min TTL

# Sorted sets (leaderboards, no TTL — permanent structures)
lb:reputation                     → ZADD score=reputation member=managerId
lb:goals:{seasonId}:{leagueId}    → ZADD score=goals member=playerId

# Rate limiting (TTL: 60 sec)
rl:ip:{ipAddress}   → counter
rl:user:{userId}    → counter

# Multiplayer gates (TTL: 48h + buffer)
gate:{matchDayId}:lineups  → Set of submitted managerIds
gate:{matchDayId}:deadline  → Unix timestamp
```

### Eviction Policy

Production Redis configured with `maxmemory-policy allkeys-lru`.

Critical data (sessions, gates) must also be backed by PostgreSQL — Redis is a cache, not a source of truth.

### Redis Cluster (Phase 4+)

3-node cluster with hash slots:
- No manual sharding required — hash slot distribution is automatic
- Cluster-aware client required (ioredis with cluster mode)
- MULTI/EXEC transactions not available across cluster nodes — use Lua scripts for atomic operations on single keys

---

## 11. Performance Targets per Query Type

| Query | Target Latency | Implementation |
|---|---|---|
| Player search by position + CA range | < 50ms | PostgreSQL indexed query |
| League standings | < 20ms | Redis cached projection |
| Active squad for a club | < 30ms | Redis cached projection |
| Transfer market listings | < 40ms | Redis cached projection |
| Match event log replay | < 100ms | PostgreSQL stream query with index |
| Financial balance for a club | < 30ms | PG materialized view |
| Leaderboard top 100 | < 5ms | Redis sorted set ZREVRANGE |
| Login / JWT issue | < 200ms | PG session write + Redis write |
