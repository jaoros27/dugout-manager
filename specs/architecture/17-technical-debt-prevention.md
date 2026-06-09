# Technical Debt Prevention Plan — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. Philosophy

Technical debt is not inherently bad — it is a trade-off. This plan distinguishes:

- **Acceptable debt**: conscious shortcuts with documented repayment plan
- **Unacceptable debt**: ignorance-driven shortcuts, no tracking, no plan
- **Debt that kills projects**: architectural violations that compound over time

The goal is to make unacceptable debt hard to introduce and easily detectable.

---

## 2. Architecture Guardrails

### 2.1 Dependency Rules (Enforced by CI)

```
Allowed dependency directions:
  apps/ → packages/    (apps consume packages)
  packages/sdk → packages/shared-types
  packages/game-engine → packages/shared-domain
  packages/match-engine → packages/shared-types   (NOT game-engine)
  packages/ui → packages/shared-types

Forbidden:
  packages/match-engine → packages/game-engine   (engine must be isolated)
  packages/shared-domain → any app               (domain must not know about apps)
  packages/shared-types → anything               (types are leaves)
  Any package → apps/                             (no circular upward dependencies)

Enforcement: eslint-plugin-import + custom circular dependency check in CI
Tool: madge or dependency-cruiser as CI gate
```

### 2.2 Domain Purity Rule

No framework code in domain model:
- No `@Injectable()`, `@Column()`, or NestJS decorators in `packages/shared-domain`
- No TypeORM/Prisma entities in the domain layer
- Domain objects are plain TypeScript classes/types
- Repository implementations live in infrastructure layer, not domain

### 2.3 Match Engine Isolation Rule

`packages/match-engine` must:
- Have zero runtime dependencies on databases, HTTP clients, or ORMs
- Import only from `packages/shared-types`
- Export only pure functions
- Be runnable in a browser, Node.js, and Deno without modification

Verification: CI runs `node --input-type=module < packages/match-engine/src/index.ts` in a sandboxed environment.

---

## 3. Code Quality Standards

### 3.1 Test Coverage Requirements

| Layer | Minimum Coverage | Enforcement |
|---|---|---|
| Match Engine | 90% branch | CI gate (fail PR) |
| Domain Services | 85% branch | CI gate (fail PR) |
| API handlers | 80% line | CI gate (warn, not fail) |
| UI components | Not measured | Storybook visual tests |

### 3.2 Engine Statistical Tests as CI Gate

The Monte Carlo suite (100-season validation) runs on every PR that touches `packages/match-engine` or `packages/game-engine`. If any statistical assertion fails, the PR cannot merge.

This is the single most important quality gate in the project.

### 3.3 Mutation Testing (Phase 3+)

Pitest/Stryker mutation testing on the match engine and domain layer quarterly. Goal: mutation score > 70%. Prevents tests that cover lines without actually testing behavior.

---

## 4. API Contract Stability

### 4.1 Versioned API from Day One

All API routes are versioned:
```
/api/v1/careers
/api/v1/players
/api/v1/transfers
```

Rationale: Never break clients (mobile app, third-party integrations) with server changes.

### 4.2 Breaking Change Policy

A change is breaking if it:
- Removes a field from a response
- Changes a field type
- Removes an endpoint
- Changes authentication requirements

Breaking changes require:
1. New API version (`/v2/`)
2. Old version maintained for minimum 6 months
3. Deprecation header added to old version responses
4. Migration guide published

### 4.3 Shared Types Package

`packages/shared-types` is the source of truth for all API contracts. Frontend and backend both import from it. Runtime type mismatches are caught at compile time.

```typescript
// shared-types/src/api/player.ts
export interface PlayerSummaryDto {
  id: string
  name: string
  position: PositionCode
  nationality: string
  age: number
  currentAbility: number
  // NOTE: potentialAbility is intentionally absent — never exposed in API
}
```

---

## 5. Event Schema Versioning

Domain events are stored permanently. Event schemas MUST be versioned.

### Event Versioning Strategy

```typescript
// Version 1 — initial
interface MatchSimulatedV1 {
  version: 1
  matchId: string
  homeScore: number
  awayScore: number
  // events was added in V2
}

// Version 2 — added event log
interface MatchSimulatedV2 {
  version: 2
  matchId: string
  homeScore: number
  awayScore: number
  events: MatchEvent[]
}

// Upcaster: converts V1 to V2 at read time
function upcastMatchSimulated(event: MatchSimulatedV1): MatchSimulatedV2 {
  return { ...event, version: 2, events: [] }
}
```

Rule: Event handlers always receive the latest version. Upcasters run at projection time.

---

## 6. Database Migration Discipline

Rules enforced by code review and CI:

1. **No `ALTER TABLE ... DROP COLUMN`** without a prior migration that marks the column deprecated
2. **No `NOT NULL` additions** without a default value (zero-downtime migration)
3. **All migrations reversible** (include `down` migration)
4. **Migration naming**: `V001__describe_change.sql` — numbered and descriptive
5. **No migrations on application startup** in production — separate CI/CD step
6. **Migration tested on a copy of production data** before deployment (Phase 4+)

---

## 7. Observability as a Quality Gate

Technical debt often hides in unmonitored code paths. Required instrumentation:

- Every API route: duration histogram, error rate counter
- Every NATS consumer: processing time, failure count, retry count
- Every match simulation: step timing, result distribution
- Every database query > 100ms: logged with query plan

A feature that ships without observability instrumentation is considered incomplete.

---

## 8. Refactoring Budget

| Phase | Refactoring budget (% of engineering capacity) |
|---|---|
| 1–2 | 10% (engine still evolving — defer refactoring) |
| 3 | 20% (stabilize before multiplayer) |
| 4 | 15% |
| 5+ | 20% (mobile brings new requirements, old patterns need review) |

Tracked in backlog with explicit "tech debt" label. Reviewed in monthly architecture review.

---

## 9. ADR Process

Every significant architectural decision must have an ADR before implementation.

### When to write an ADR
- Choosing between two or more viable technical options
- Adding a new dependency (library, service, protocol)
- Changing a data model that affects multiple contexts
- Introducing a new pattern not yet in the codebase

### ADR States
- `Draft` → `Proposed` → `Accepted` → `Deprecated` → `Superseded`

Superseded ADRs are kept in the `specs/adrs/` folder with a pointer to the replacement. History is never deleted.

---

## 10. Developer Onboarding as a Quality Metric

If a new developer cannot:
- Run the full local dev environment in < 30 minutes
- Run the full test suite in < 10 minutes
- Understand the system architecture from `specs/architecture/` in < 2 hours

...then the documentation or developer experience has unacceptable debt.

Measured by: onboarding checklist run with each new contributor. Failures become backlog items.

---

## 11. Deprecation Protocol

When removing a feature, API, or internal abstraction:

1. Mark deprecated with notice period (minimum: next major phase)
2. Log deprecation warning at runtime (API: deprecated header, internal: console.warn)
3. Add to "deprecation tracker" in `/specs/architecture/deprecation-tracker.md`
4. Remove in planned sprint — never leave deprecated code indefinitely
5. Reference the original ADR that justified the original design

---

## 12. Long-Term Architecture Fitness Functions

These are automated tests that verify the architecture stays healthy over time:

| Fitness Function | Tool | Frequency |
|---|---|---|
| No circular package dependencies | dependency-cruiser | Every PR |
| Match engine statistical validity | Monte Carlo suite | Every engine PR |
| Domain layer has no framework imports | custom ESLint rule | Every PR |
| API response times within SLA | k6 load test | Weekly |
| Database query count per request < 20 | query counter middleware | Every integration test |
| No new `any` types in shared-types | TypeScript strict | Every PR |
| No direct cross-schema SQL writes | custom SQL linter | Pre-merge |
