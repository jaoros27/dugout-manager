# Non-Functional Requirements — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## NFR-01 — Performance

| ID | Requirement | Target | Rationale |
|---|---|---|---|
| NFR-01.1 | Single match simulation | < 50ms | Real-time feedback during bulk simulation |
| NFR-01.2 | Full 38-match season simulation | < 2s | Career mode batch processing |
| NFR-01.3 | 100-club, 10-season full simulation | < 30s | Engine validation and AI processing |
| NFR-01.4 | API response time (p95) | < 200ms | Web/mobile UX threshold |
| NFR-01.5 | API response time (p99) | < 500ms | Acceptable tail latency |
| NFR-01.6 | Transfer market query | < 100ms | Responsive search experience |
| NFR-01.7 | UI time-to-interactive (first load) | < 3s | Web performance baseline |
| NFR-01.8 | UI time-to-interactive (subsequent) | < 1s | Cached navigation |

---

## NFR-02 — Scalability

| ID | Requirement | Target |
|---|---|---|
| NFR-02.1 | Concurrent active users (Phase 3) | 10,000 |
| NFR-02.2 | Concurrent active users (Phase 4) | 100,000 |
| NFR-02.3 | Concurrent active users (Phase 6) | 1,000,000 |
| NFR-02.4 | Match engine must be horizontally scalable | Stateless, parallelizable |
| NFR-02.5 | API services scale independently | Per-service horizontal scaling |
| NFR-02.6 | Database read replicas for query-heavy paths | At least 2 read replicas in Phase 4+ |
| NFR-02.7 | Match processing queue handles burst | 10,000 matches/minute burst capacity |

---

## NFR-03 — Reliability

| ID | Requirement | Target |
|---|---|---|
| NFR-03.1 | API availability | 99.5% (Phase 3), 99.9% (Phase 4+) |
| NFR-03.2 | Match engine must be deterministic given same seed | 100% reproducible |
| NFR-03.3 | No data loss on service restart | Write-ahead log + event sourcing |
| NFR-03.4 | Graceful degradation: match engine failure must not crash API | Circuit breaker pattern |
| NFR-03.5 | Player save state never corrupted by partial writes | Transactional writes only |
| NFR-03.6 | Multiplayer match day must complete atomically | All-or-nothing processing |

---

## NFR-04 — Security

| ID | Requirement |
|---|---|
| NFR-04.1 | Authentication via JWT with refresh tokens |
| NFR-04.2 | All API endpoints authenticated except public league listings |
| NFR-04.3 | Rate limiting on all public endpoints (100 req/min per IP) |
| NFR-04.4 | Match engine inputs validated server-side — client cannot inject match results |
| NFR-04.5 | No player attributes accessible before match result is processed (anti-cheat) |
| NFR-04.6 | All user data encrypted at rest |
| NFR-04.7 | PII (email, IP) stored separately from game data |
| NFR-04.8 | OWASP Top 10 compliance required before Phase 3 launch |
| NFR-04.9 | Transfer offers validated server-side — cannot bypass budget constraints client-side |

---

## NFR-05 — Modularit and Maintainability

| ID | Requirement |
|---|---|
| NFR-05.1 | Match engine has no compile-time dependency on API layer |
| NFR-05.2 | Frontend has no direct dependency on database models |
| NFR-05.3 | Domain model has no dependency on any framework (NestJS, React, etc.) |
| NFR-05.4 | Each bounded context has its own database schema namespace |
| NFR-05.5 | Adding a new competition type requires no changes to match engine |
| NFR-05.6 | Adding a new data provider requires changes only in the data pipeline layer |
| NFR-05.7 | All inter-service communication via defined contracts (events/DTOs) |
| NFR-05.8 | Each package/app independently testable and deployable |

---

## NFR-06 — Testability

| ID | Requirement | Target |
|---|---|---|
| NFR-06.1 | Match engine unit test coverage | ≥ 90% |
| NFR-06.2 | Domain layer unit test coverage | ≥ 85% |
| NFR-06.3 | API integration test coverage (happy path + error cases) | ≥ 80% |
| NFR-06.4 | Match engine statistical validation (Monte Carlo) | 100-season, 1000-season runs |
| NFR-06.5 | CI pipeline runs all tests on every PR | < 10 min total CI time |
| NFR-06.6 | Match simulation is deterministic when seeded | 100% |
| NFR-06.7 | Economic simulation tests for inflation/collapse | 1000-season economic validation |

---

## NFR-07 — Observability

| ID | Requirement |
|---|---|
| NFR-07.1 | All services emit structured logs (JSON) with trace IDs |
| NFR-07.2 | All API endpoints instrumented with OpenTelemetry traces |
| NFR-07.3 | Match engine emits timing metrics per simulation step |
| NFR-07.4 | Prometheus metrics exposed by all services |
| NFR-07.5 | Grafana dashboards for: API latency, match queue depth, active users, error rate |
| NFR-07.6 | Alerting on: p99 latency > 1s, error rate > 1%, queue depth > 1000 |
| NFR-07.7 | Distributed tracing across match request → engine → result |

---

## NFR-08 — Developer Experience

| ID | Requirement |
|---|---|
| NFR-08.1 | Full local dev environment starts with a single command |
| NFR-08.2 | Hot reload for all apps in development mode |
| NFR-08.3 | Shared types package eliminates runtime type mismatches between services |
| NFR-08.4 | ADRs document all major architectural decisions |
| NFR-08.5 | New developer productive in < 1 day with CLAUDE.md or README |
| NFR-08.6 | No circular dependencies between packages |

---

## NFR-09 — Portability

| ID | Requirement |
|---|---|
| NFR-09.1 | Game engine runs in: browser (WASM), Node.js, and native (Rust FFI optional) |
| NFR-09.2 | Web app works on Chrome, Firefox, Safari (last 2 major versions) |
| NFR-09.3 | Desktop app (Phase 2) packaged for Windows, macOS, Linux |
| NFR-09.4 | API containerized with Docker; no host OS dependencies |
| NFR-09.5 | Database migrations versioned and reversible |

---

## NFR-10 — Data Compliance

| ID | Requirement |
|---|---|
| NFR-10.1 | No FIFA, UEFA, CBF, or CONMEBOL logos/branding without explicit license |
| NFR-10.2 | Real player names used only under license or via mod system |
| NFR-10.3 | User data deletion request processed within 30 days (LGPD / GDPR) |
| NFR-10.4 | Data provider contracts reviewed by legal before production use |
| NFR-10.5 | Mod system validates that mods do not inject licensed data into base game |
