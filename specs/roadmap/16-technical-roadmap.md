# Technical Roadmap — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## Phase Overview

```
Phase 1: Engine Offline          (Months 1–3)
Phase 2: Single Player Web       (Months 4–9)
Phase 3: Backend Persistent      (Months 10–15)
Phase 4: Multiplayer             (Months 16–24)
Phase 5: Mobile                  (Months 25–32)
Phase 6: Ecosystem               (Months 33–48)
```

## Guiding Principle: Playable Before Perfect

This roadmap is ambitious for a small team — the single biggest project risk is
burnout/over-engineering before anyone plays the game (see Risk BR-03). Therefore:

- **The goal of Phase 1 is the earliest playable loop, not a perfect engine.** A
  rough but fun match → season → transfer loop that a human can play and enjoy beats a
  statistically immaculate engine with no game around it. Fun retains users; rigor does
  not. Statistical validation runs continuously but does not gate "is this fun?".
- **Scope is cut aggressively at every phase boundary.** Any feature not required to
  make the current phase *playable and enjoyable* is deferred. The 1M-user architecture
  is the destination, not the Phase 1 deliverable.
- **Each phase must end with something a real person can play end-to-end.** No phase
  ships pure infrastructure with no playable surface.

## Recurring Operation: Annual Update Cycle

"Annual data update" is a core product promise (Brasfoot heritage), not a Phase 6
feature. From **Phase 3 onward** it is a recurring yearly operation, not a one-off:

- Each year: import the new season's data via the Data Pipeline (doc 11), recalculate
  attributes/CA, version the dataset (`EngineTuning` + data version pinned per
  season/league), and publish.
- Active careers and multiplayer leagues pin to the data version at season start —
  they never change mid-season.
- The pipeline and `EngineTuning` config (doc 12 §1.1) make this a config + import
  operation, never a code release. Target: annual update ships without redeploying the
  engine.

---

## Phase 1 — Engine Offline

**Duration**: 3 months  
**Goal**: Build and validate the match engine and domain model. No UI, no server, no database.

### Objectives
- Implement complete player model (all attributes, CA/PA, development curves)
- Implement match simulation engine (hybrid event-driven)
- Implement AI club lineup selection and basic transfer AI
- Implement competition format engine (league, cup, group+knockout)
- Implement 1-season and 10-season simulation loop
- Validate engine with Monte Carlo statistical suite
- Validate economic model (no inflation collapse, no single-club dominance)

### Deliverables
- `packages/match-engine`: fully tested, pure TypeScript module
- `packages/game-engine`: domain model, season loop, AI
- `packages/shared-types`: all domain types (shared across packages)
- Monte Carlo test suite passing all statistical assertions
- README documenting how to run a simulation from command line

### Risks
- Engine calibration may require multiple iterations (expect 2–3 calibration cycles)
- PA estimation algorithm may be harder to balance than expected

### Dependencies
- None (fully offline)

---

## Phase 2 — Single Player Web

**Duration**: 6 months  
**Goal**: Full career mode playable in the browser. Local save, no backend.

### Objectives
- Web UI with full career mode (club selection → squad management → season progression)
- Transfer market UI (search, offers, negotiations)
- Tactics and lineup editor
- Match report viewer (full event log, stats, player ratings)
- Financial dashboard
- Training management
- Youth academy view
- Local save/load via IndexedDB (no backend required)
- Desktop packaging (Tauri — lightweight alternative to Electron)
- Fictitious player/club database (1,000+ players, 100+ clubs, 5 leagues)

### Deliverables
- `apps/web`: Next.js app (App Router) with full career UI
- `apps/desktop`: Tauri wrapper for Windows/macOS/Linux
- `packages/ui`: shared React component library (Tailwind)
- `packages/sdk`: client SDK for game engine state management
- Fictitious data pack (JSON format, serves as first mod pack template)

### Technical Milestones
- Month 4: Core UI scaffolded, squad view and match simulation working in browser
- Month 5: Full season loop playable (advance week, simulate matches, standings update)
- Month 6: Transfer market, tactics, and training functional
- Month 7: Financial system, youth academy, multi-season careers
- Month 8: Desktop app packaged, UX polish pass
- Month 9: Beta release to closed group (50–100 users), feedback iteration

### Risks
- Game state management in browser can become complex (Zustand architecture must be right)
- Tauri desktop packaging has learning curve
- UX decisions in this phase are hard to reverse

### Dependencies
- Phase 1 complete (game engine and domain model stable)

---

## Phase 3 — Backend Persistent

**Duration**: 6 months  
**Goal**: User accounts, cloud saves, persistent careers, leaderboards.

### Objectives
- User registration, authentication (JWT + refresh tokens)
- Cloud save system (careers synced to PostgreSQL)
- Leaderboards (reputation, goals scored, season winners)
- Admin panel (user management, system health)
- Observability stack (OpenTelemetry, Prometheus, Grafana)
- Production deployment (Docker Compose initially → Kubernetes prep)
- Public API documentation (Swagger/OpenAPI)
- Data pipeline: first real data import (API-Football free tier)
- LGPD/GDPR compliance: data deletion, privacy policy

### Deliverables
- `apps/api`: NestJS API (all domain services, CQRS, event bus)
- `apps/admin`: Admin Next.js app
- `infrastructure/docker`: Docker Compose for local dev + production
- `infrastructure/monitoring`: Prometheus + Grafana dashboards
- Data pipeline MVP (import + validate + normalize)

### Technical Milestones
- Month 10: Auth system + cloud save basic flow
- Month 11: All career features working via API (not just local)
- Month 12: Leaderboards, admin panel, observability
- Month 13: Data pipeline MVP + first real data import
- Month 14: Security audit (OWASP), performance testing
- Month 15: Launch to public (1,000 user target)

### Risks
- LR-04 (LGPD compliance) must be complete before launch
- Performance testing may reveal PostgreSQL indexing gaps

### Dependencies
- Phase 2 complete (UI must be stable)

---

## Phase 4 — Multiplayer

**Duration**: 9 months  
**Goal**: Async multiplayer leagues with shared transfer market.

### Objectives
- Private and public league creation
- Match day gate pattern (lineup submission + deadline + atomic processing)
- Shared transfer market with optimistic concurrency
- WebSocket notifications (match results, transfer updates, deadlines)
- Anti-cheat: server-side simulation, lineup opacity
- Commissioner tools
- League chat
- Kubernetes deployment with HPA
- Match engine worker pool (separate from API)

### Deliverables
- Multiplayer service (NestJS module)
- WebSocket gateway (separate deployment)
- Match engine workers (NATS-based, stateless)
- Kubernetes manifests (`infrastructure/kubernetes/`)
- Anti-cheat test suite

### Technical Milestones
- Month 16–17: Match day gate infrastructure + single-league prototype
- Month 18–19: Shared transfer market + concurrent bid handling
- Month 20: WebSocket notifications + commissioner tools
- Month 21: Anti-cheat implementation + security testing
- Month 22: Kubernetes deployment + load testing (10k users simulation)
- Month 23: Beta with 50 leagues (500 managers)
- Month 24: Public launch

### Risks
- TR-05 (multiplayer consistency bug) is the highest risk in this phase
- WebSocket scaling requires careful architecture (sticky sessions)

### Dependencies
- Phase 3 backend stable and production-tested

---

## Phase 5 — Mobile

**Duration**: 8 months  
**Goal**: Native mobile app (iOS + Android) with same backend.

### Objectives
- React Native app (same domain logic via SDK package)
- Mobile-optimized UX (touch-first, progressive disclosure)
- Push notifications (match day deadlines, transfer updates)
- Offline mode (single player career without connectivity)
- App Store + Google Play submission + review process

### Deliverables
- `apps/mobile`: React Native app
- Push notification service (Firebase Cloud Messaging)
- Mobile-specific SDK adaptations

### Dependencies
- Phase 4 API stable and documented
- `packages/sdk` mature enough for mobile

---

## Phase 6 — Ecosystem

**Duration**: 16 months  
**Goal**: Mod system, global leagues, community data, first licensed data.

### Objectives
- Mod SDK (documentation, tooling, validator)
- Mod browser in-game (upload, download, rate)
- Community data pipeline (accept community-built season packs)
- First licensed data deal (target: Brazilian Série A / CBF)
- Multi-region infrastructure (Americas, Europe)
- Public developer API (third-party integrations)
- Global leaderboards and esports infrastructure
- Season schedule: annual data update cycle

### Deliverables
- Mod SDK and documentation
- Mod validation pipeline
- Community portal (web)
- Licensed data integration (first region)
- Multi-region Kubernetes
- Developer API + API keys

---

## Milestone Summary

| Milestone | Target Month | Success Criteria |
|---|---|---|
| Engine validated (Monte Carlo) | Month 3 | All 7 statistical assertions pass |
| Beta single player | Month 9 | 50 users play 3+ seasons without bugs |
| Backend launch | Month 15 | 1,000 registered users |
| Multiplayer beta | Month 23 | 500 managers across 50 leagues |
| Multiplayer public | Month 24 | 10,000 users |
| Mobile launch | Month 33 | 100,000 installs |
| First licensed data | Month 40 | Brazilian Série A in base game |
| Global ecosystem | Month 48 | 1,000,000 users, 50+ community mods |
