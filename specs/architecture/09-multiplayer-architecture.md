# Multiplayer Architecture — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. Multiplayer Model

Dugout Manager uses **asynchronous turn-based multiplayer** — not real-time.

This is a deliberate design choice:

| Concern | Real-time | Async Turn-based (our model) |
|---|---|---|
| Server cost | Very high (open connections) | Low (stateless per request) |
| Fairness | Time-zone disadvantage | Equal — everyone plays same match day |
| Complexity | Distributed locking, CRDT | Simpler event ordering |
| Player experience | Requires constant availability | Play at your own pace |
| Cheat surface | Timing exploits, connection abuse | Much smaller |

Football management is inherently asynchronous — you don't watch every second of the match.

---

## 2. League Types

### Private League
- Created by one manager (commissioner)
- Invite-only via link/code
- Full commissioner control: pause, kick, replace with AI
- 4–20 human managers

### Public League
- Matchmaking by reputation tier
- Auto-filled with AI clubs when not enough humans
- No commissioner — automated moderation

### Solo Campaign (hybrid)
- Single human manager
- All other clubs are AI
- Can "open" to multiplayer later (migration path)

---

## 3. Match Day Gate Pattern

The core coordination primitive of multiplayer:

```
State machine per MatchDay:

PENDING
  │  All lineups submitted OR deadline elapsed
  ▼
LOCKED (no more changes)
  │  MatchDayOrchestrator processes all matches
  ▼
PROCESSING
  │  All MatchSimulated events received
  ▼
COMPLETE (results published to all managers)
```

### Deadline Policy
- Default: 48 hours from match day open
- Configurable by commissioner (24h–72h)
- Auto-submit uses last saved lineup when deadline elapses
- If no lineup ever saved: system selects best CA XI automatically

### Atomicity Guarantee
- All matches in a match day are processed in a single NATS batch job
- Results are committed to the database in a single PostgreSQL transaction
- No manager can query match day results until ALL results are committed
- If any match simulation fails: entire match day is retried (idempotent)

---

## 4. Shared Transfer Market

All human managers in a league share the same transfer market.

### Concurrent Bid Problem

**Scenario**: Manager A and Manager B both submit offers for Player X at the same moment.

**Solution: First-Accepted Wins with Optimistic Concurrency**

```
TransferOffer table has a unique constraint:
  (playerId, windowId, status=ACCEPTED) is unique

Transaction flow:
  1. Load player's current transfer record (with version)
  2. Check no ACCEPTED offer exists for this player in this window
  3. If conflict: UPDATE player version WHERE version = :expectedVersion
  4. If version mismatch: transaction fails with CONCURRENT_MODIFICATION error
  5. Client receives 409 Conflict → "Another manager beat you to it"
```

### Transfer Window Rules
- Transfer windows are global within a league (all managers share open/close dates)
- Free agent signings allowed outside windows
- Loan recalls allowed mid-season (if clause agreed)
- Pre-contracts can be agreed from January 1 for the following season

---

## 5. Synchronization Model

### What is synchronized?
| Data | Sync Model |
|---|---|
| League table standings | After each match day completes |
| Match results | After full match day is committed |
| Transfer market listings | Real-time (eventual, < 5s) |
| Squad state | After each match day |
| Player form/fatigue/injuries | After each match day |
| Financial balances | After wages/transfers processed |
| Chat messages | Real-time via WebSocket |

### What is NOT synchronized in real-time?
- Tactics (per-manager, submitted at lineup time)
- Training plans (per-manager, not visible to others)
- Youth players (per-manager)

---

## 6. WebSocket Architecture

Used exclusively for **push notifications**, not game state.

```
Client connects to WebSocket Gateway
  │  Authenticates with JWT on handshake
  │  Subscribes to league channel + personal channel
  │
Channels:
  - league:{leagueId}     → match day opened/closed, results available
  - user:{managerId}      → transfer offer received/accepted, personal alerts
  - system               → maintenance notices

Message format:
  {
    type: "MATCH_DAY_COMPLETE" | "TRANSFER_ACCEPTED" | "DEADLINE_REMINDER",
    payload: { ... },
    leagueId: string,
    timestamp: ISO8601
  }
```

WebSocket connections are handled by dedicated WebSocket Gateway instances (separate from API service) because:
- They maintain long-lived connections → different scaling needs
- API service is stateless; WebSocket gateway needs sticky sessions

---

## 7. Anti-Cheat Design

### Attack Surface Analysis

| Attack | Risk | Mitigation |
|---|---|---|
| Client sends modified match result | High | Server always re-simulates; client result is ignored |
| Client modifies player attributes | High | All simulation inputs fetched server-side, never trusted from client |
| Manager views opponent lineup before submitting own | Medium | Lineups encrypted server-side until match day gate closes |
| Timing exploit: submit lineup after seeing partial results | Medium | Gate closes atomically; no partial results visible |
| Multiple accounts in same league | Low | IP + device fingerprint checks at league join |
| Exploiting AI clubs for cheap transfers | Low | AI evaluates offers against realistic valuation model |
| Save file manipulation (single player) | Low (single player — acceptable) | Hash verification on load for online saves |

### Core Anti-Cheat Principles
1. **Never trust the client** for any simulation input
2. **Match simulation always server-side** from Phase 3 onward
3. **Lineup opacity**: opponent lineup not readable until match day locked
4. **Idempotent match processing**: replaying same match day produces same result
5. **Audit log**: every transfer, lineup, and simulation recorded with timestamps

---

## 8. Multiplayer Consistency Guarantees

| Operation | Consistency Level | Why |
|---|---|---|
| Submit lineup | Eventual (confirmed within 1s) | Low stakes, can be resubmitted |
| Transfer offer | Strong (transactional) | Must prevent double-acceptance |
| Match day results | Strong (atomic batch commit) | All-or-nothing visibility |
| League table update | Strong (after match day commit) | Integrity-critical |
| Chat message | Eventual | Low stakes |
| Financial balance | Strong (after transaction) | Money must balance |

---

## 9. Inactive Manager Handling

| Scenario | Action |
|---|---|
| Manager misses 1 match day | Auto-submit best XI; warning sent |
| Manager misses 3 consecutive match days | Commissioner can replace with AI |
| Manager account deleted | Replaced with AI; historical results preserved |
| Commissioner inactive | Longest-tenured manager promoted to commissioner |

---

## 10. Scalability Targets

| Metric | Phase 4 target | Phase 6 target |
|---|---|---|
| Concurrent leagues | 500 | 50,000 |
| Concurrent WebSocket connections | 10,000 | 500,000 |
| Match simulations per minute (burst) | 5,000 | 200,000 |
| Transfer market concurrent users | 1,000 | 50,000 |

### Scaling the Match Day Gate
- Each league's match day processing is independent
- Match Engine Worker Pool scales horizontally via Kubernetes HPA
- NATS JetStream queue absorbs burst: 50,000 jobs/second is achievable on a 3-node cluster
- PostgreSQL batch insert for match results: one transaction per match day (not per match)
