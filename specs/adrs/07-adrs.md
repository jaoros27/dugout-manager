# Architecture Decision Records — Dugout Manager

> Version: 1.0  
> Format: MADR (Markdown Architecture Decision Records)

---

## ADR-001 — Monorepo Tooling: Turborepo

**Status**: Accepted  
**Date**: 2026-06-01

### Context
We have a monorepo with: 3 apps (web, api, admin), 6 packages (game-engine, match-engine, shared-domain, shared-types, sdk, ui). We need:
- Incremental builds (only rebuild what changed)
- Parallel task execution
- Shared dependency management
- TypeScript path aliases across packages

### Options Considered

| Tool | Pros | Cons |
|---|---|---|
| **Turborepo** | Fast caching, simple config, Vercel ecosystem, great TS support | Younger project, less enterprise features |
| **Nx** | Mature, generators, plugins, enterprise grade | Complex config, steep learning curve, heavier |
| **pnpm Workspaces alone** | Minimal overhead, no lock-in | No task orchestration, no caching, manual build order |

### Decision
**Turborepo + pnpm Workspaces**.

Turborepo handles task orchestration and remote caching. pnpm handles dependency management. This combination is the industry standard for TypeScript monorepos in 2025–2026.

### Rationale
- Turborepo config is ~50 lines vs Nx's 300+
- pnpm's strict dependency isolation prevents phantom dependencies
- Remote caching (Vercel or self-hosted) will dramatically speed CI from Phase 3 onward
- Nx is overkill until we have 20+ packages and dedicated platform engineers

### Consequences
- Must define task dependency graph in `turbo.json`
- Remote cache server needed in CI from Phase 3
- pnpm lockfile must be committed

---

## ADR-002 — Frontend Framework: Next.js + React

**Status**: Accepted  
**Date**: 2026-06-01

### Context
We need a web application that:
- Has server-side rendering for first load performance
- Supports future API routes
- Works well with TypeScript and Tailwind
- Can share components with a future mobile app

### Options Considered
| Option | Pros | Cons |
|---|---|---|
| **Next.js** | SSR/SSG, App Router, great TS, Vercel deploy, ecosystem | Vendor pull toward Vercel |
| Vite + React SPA | Simpler, faster dev, no SSR complexity | No SSR, manual routing, manual API handling |
| Remix | Excellent data loading model | Smaller ecosystem, less familiar |
| SvelteKit | Performance, simplicity | Smaller ecosystem, team reskilling needed |

### Decision
**Next.js (App Router) + React 19 + TypeScript + Tailwind + Zustand**.

### Rationale
- App Router enables server components — critical for performance at 100k+ users
- Zustand is lighter than Redux, simpler than Jotai for game state management
- The team knows React; switching to Svelte would add ramp-up time
- Next.js API routes can be used for BFF pattern (Backend-for-Frontend) in Phase 2

### Consequences
- App Router mental model differs from Pages Router — team needs training
- SSR adds complexity for real-time game state (must be careful about hydration)
- Vercel deployment is easiest path but not mandatory — Docker works fine

---

## ADR-003 — Backend Framework: NestJS

**Status**: Accepted  
**Date**: 2026-06-01

### Context
We need a backend that:
- Supports CQRS pattern natively
- Has strong TypeScript support
- Supports dependency injection for testability
- Can integrate with event buses (NATS/Kafka)
- Scales to microservices if needed

### Options Considered
| Option | Pros | Cons |
|---|---|---|
| **NestJS** | CQRS module, DI, decorators, structured, modular | Opinionated, magic annotations, heavier startup |
| Fastify + custom | Fastest raw performance, full control | No structure, must build CQRS/DI by hand |
| Hono | Ultra-lightweight, edge-ready | Too minimal for enterprise game backend |
| tRPC | Type-safe APIs, excellent DX | Not designed for event-driven / CQRS |

### Decision
**NestJS with `@nestjs/cqrs` module**.

### Rationale
- `@nestjs/cqrs` provides Command Bus, Query Bus, and Event Bus out of the box
- NestJS modules map cleanly to Bounded Contexts
- Testability via DI is essential for a system with complex domain rules
- Performance is sufficient: 10k rps per instance, horizontally scalable

### Challenges Acknowledged
- NestJS decorators can become "magic" — we will enforce clear module boundaries
- Start-up time is ~500ms — acceptable for server, not for edge functions
- We will NOT use NestJS for the match engine (it runs as a pure TS module)

---

## ADR-004 — Message Broker: NATS JetStream

**Status**: Accepted  
**Date**: 2026-06-01

### Context
We need async messaging for:
- Match day processing (fan-out to multiple consumers)
- Season progression events
- Multiplayer coordination
- Notification delivery

### Options Considered
| Option | Pros | Cons |
|---|---|---|
| **NATS JetStream** | Lightweight, fast, at-least-once, pub/sub + queue, easy Docker | Smaller ecosystem than Kafka, less tooling |
| Kafka | Battle-tested, huge ecosystem, excellent for event log | Operationally complex, overkill for Phase 1–3 |
| RabbitMQ | Mature, familiar, good DX | Fewer primitives for event sourcing patterns |
| Redis Streams | Already in stack, simple | Limited consumer group semantics, not a full broker |

### Decision
**NATS JetStream** for Phase 1–4. Re-evaluate Kafka at 500k+ active users.

### Rationale
- NATS JetStream adds persistence and at-least-once delivery on top of NATS core
- Single binary, trivial Docker setup — critical for developer experience
- JetStream subjects map naturally to domain events
- Kafka requires Zookeeper or KRaft + more operational expertise we don't have in Phase 1

### Migration Path
If we outgrow NATS, the NATS consumer/publisher code is isolated behind event bus ports — swapping to Kafka is a ~2-day infrastructure change, not a domain rewrite.

---

## ADR-005 — Match Engine Simulation Model: Hybrid Event-Driven

**Status**: Accepted  
**Date**: 2026-06-01

### Context
The match engine is the most critical system. We evaluated:
1. **Turn-based**: divide match into N turns, resolve each turn probabilistically
2. **Event-based**: generate significant events (goals, cards, injuries) with probability distribution
3. **Hybrid**: generate a sequence of events with time, influenced by team states throughout the match

### Analysis

**Turn-based (rejected)**:
- Simple to implement
- Too granular → either too many turns (slow) or too few (unrealistic)
- Doesn't model momentum shifts naturally

**Event-based (pure)**:
- Fast (just generate 5–20 events per match)
- Loses match texture: no possession, no chance build-up
- Feels like a random number generator with stats wrapper

**Hybrid (chosen)**:
- Divide match into phases (5-minute intervals = 18 phases + extra time)
- Each phase determines possession dominance and chance generation
- Chances converted to events (goal, save, miss, corner, foul) based on player attributes
- Momentum: a team winning a phase has increased chance generation in next phase
- Fatigue: physical attributes degraded in phases 15–18

### Decision
**Hybrid Event-Driven Simulation**.

### Algorithm Sketch
```
for each phase (5 minutes):
  1. Compute possession share (tactic + midfield attributes)
  2. Compute chance count for each team (possession × attack rating)
  3. For each chance:
     a. Is it a shot? (dribbling, positioning attributes)
     b. Is it on target? (finishing, heading)
     c. Is it a goal? (finishing vs GK rating)
  4. Apply momentum modifier
  5. Check for cards, injuries based on fouls (tackling, strength differential)
  6. Apply fatigue decay to physical attributes
```

### Consequences
- Engine has ~200–400 simulation steps per match → must complete in < 50ms
- Seeded PRNG required at each stochastic step for determinism
- Rich event log produced (18+ phases × multiple events) = good data for analytics

---

## ADR-006 — Database Primary Store: PostgreSQL + Redis

**Status**: Accepted  
**Date**: 2026-06-01

### Context
We need storage for:
- Persistent game state (players, clubs, seasons, careers)
- Event store (domain events for CQRS)
- Session/cache (active game sessions, leaderboards)
- Read projections (league tables, squad views)

### Decision
- **PostgreSQL 16+**: primary write store, event store, transactional operations
- **Redis 7+**: session cache, read projections, pub/sub notifications, leaderboards
- No separate event store database — PostgreSQL with append-only events table is sufficient through Phase 4

### Rationale
- PostgreSQL JSON/JSONB support handles domain event payloads natively
- Single database engine reduces operational complexity
- Redis is already in stack — using it for projections avoids a second DB technology
- Dedicated event store (EventStoreDB) is an option at Phase 5+, when event replay latency becomes a bottleneck

### Consequences
- Event sourcing via PostgreSQL requires careful index design
- Redis eviction policy must be `allkeys-lru` for cache use cases
- PostgreSQL connection pooling required (PgBouncer) from Phase 3

---

## ADR-007 — Architecture Pattern: CQRS + Event Sourcing (Selective)

**Status**: Accepted  
**Date**: 2026-06-01

### Context
Full event sourcing (all state derived from events) is powerful but complex. We evaluated:
1. Full Event Sourcing everywhere
2. CQRS with traditional state + event log
3. CQRS only (no event sourcing)

### Decision
**CQRS everywhere + Event Sourcing selectively** (Match and Finance contexts only).

### Rationale
- Full ES for Match: every match must be replayable for debugging, analytics, and anti-cheat
- Full ES for Finance: every money movement must be auditable forever
- Traditional state for Club, Player, Competition: simpler, faster writes, no replay needed
- CQRS throughout: write model (commands) and read model (queries) separated for performance

### Consequences
- Match events stored indefinitely → plan for data growth (partitioning by season)
- Finance events stored indefinitely → legal requirement
- All other aggregates use optimistic locking on version number, not full event replay

---

## ADR-008 — Multiplayer Consistency Model: Eventual Consistency with Turn Gates

**Status**: Accepted  
**Date**: 2026-06-01

### Context
In multiplayer, we must decide between:
1. **Strong consistency**: every operation visible to all managers immediately
2. **Eventual consistency**: changes propagate asynchronously
3. **Turn-gated consistency**: within a match day "turn", operations are eventually consistent; the turn gate enforces global ordering

### Decision
**Turn-gated consistency model**.

### Rules
- Transfer market: optimistic concurrency (first accepted bid wins, others rejected)
- Lineup submission: can be changed until gate closes
- Match results: not visible until ALL matches in the match day are processed
- Standings: updated atomically after all results posted

### Rationale
- Football management is inherently turn-based — managers don't need real-time updates
- Strong consistency would require distributed transactions across match simulations (too complex)
- Turn gates are natural in the football calendar (transfer windows, match days)
- Anti-cheat: no manager can see results before their own match is processed

### Consequences
- Turn gate adds latency (up to 48h per match day)
- Need dead-letter handling for managers who fail to submit lineup
- Requires careful UI to show pending vs confirmed match day state

---

## ADR-009 — Game Runtime Targets: TypeScript (WASM-upgradeable)

**Status**: Accepted  
**Date**: 2026-06-01

### Context
The match engine needs to run on:
- Browser (Phase 2: offline single-player in web)
- Node.js server (Phase 3+: backend simulation)
- Mobile WebView (Phase 5)

Options:
1. TypeScript only
2. Rust compiled to WASM + TypeScript bindings
3. C++ with Emscripten

### Decision
**TypeScript for Phase 1–3. Design match engine as a pure module with clean interface for future WASM port.**

### Rationale
- TypeScript is fast enough: 50ms per match is achievable in Node.js
- A pure function `simulate(ctx, seed): result` with no I/O is trivially portable to WASM later
- Rust WASM port becomes an option if bulk simulation exceeds 10,000 matches/second requirement
- C++ adds toolchain complexity with no benefit at current scale

### Trigger for Re-evaluation
If a 1,000-club simulation of 38 match days (38,000 matches) takes > 60 seconds, evaluate Rust/WASM.

---

## ADR-010 — Data Licensing Strategy: Open + Modded

**Status**: Accepted  
**Date**: 2026-06-01

### Context
Real player names, club names, and competition data require licensing from:
- FIFA / FIFPRO (player likeness and name rights)
- UEFA / CONMEBOL / CBF (competition names and branding)
- Individual clubs (logos, kit colors)

Cost of official licensing:
- FIFPRO license: €50,000–€500,000/year depending on scope
- UEFA license: negotiated case-by-case, typically 6-figure
- CBF license: unknown, Brazil-specific

### Decision
**Base game ships with fictitious data. Community mod packs provide real data. First-party licensed data added when revenue justifies cost.**

### Rationale
- Football Manager, PES (eFootball), and FIFA all started or still rely on community editors for data
- Fictitious data eliminates all licensing risk for launch
- Mod system (Phase 6) enables community to contribute real data packs
- First licensed content: Brazilian Série A (CBF) — most important market

### Naming Convention
- Fictitious names follow cultural patterns (e.g., "Felipe Santos" for Brazilian players)
- Club names are thinly disguised (e.g., "Vermelho e Preto FC" instead of "Flamengo")
- Once a license is acquired, the licensed names replace the fictitious ones

### Consequences
- Community expects real names at launch — manage expectations via mod system
- Legal review required before any real-name data ships in base game
- Mod system must include DMCA takedown support for unlicensed content
