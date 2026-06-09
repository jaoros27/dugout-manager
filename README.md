# Dugout Manager

> A modern football management game — deep simulation, multiplayer-first, built to last decades.

---

## What is this?

Dugout Manager is a football management simulation game inspired by the simplicity of Brasfoot and the depth of Football Manager. It is multiplayer-first, community-driven, and designed to evolve across multiple platforms over the next decade.

**Status**: Specification phase (Spec-Kit complete — implementation not started)

---

## Repository Structure

```
dugout-manager/
├── specs/                  — Full Spec-Kit (17 documents)
│   ├── vision/             — Product Vision
│   ├── requirements/       — Functional + Non-Functional Requirements
│   ├── architecture/       — System, Multiplayer, Data, Engine Design
│   ├── adrs/               — Architecture Decision Records
│   ├── event-storming/     — Event Storming
│   └── roadmap/            — Technical Roadmap
│
├── apps/
│   ├── web/                — Next.js web application
│   ├── api/                — NestJS backend API
│   └── admin/              — Admin dashboard
│
├── packages/
│   ├── game-engine/        — Season loop, AI, domain model
│   ├── match-engine/       — Match simulation (pure function, no I/O)
│   ├── shared-domain/      — Shared domain types and value objects
│   ├── shared-types/       — API contracts and DTOs
│   ├── sdk/                — Client SDK for game state management
│   └── ui/                 — Shared React component library
│
├── infrastructure/
│   ├── docker/             — Docker Compose configurations
│   ├── kubernetes/         — K8s manifests
│   ├── terraform/          — Infrastructure as code
│   └── monitoring/         — Prometheus + Grafana
│
└── docs/                   — Additional documentation
```

---

## Spec-Kit Documents

| # | Document | Location |
|---|---|---|
| 01 | Product Vision | [specs/vision/01-product-vision.md](specs/vision/01-product-vision.md) |
| 02 | Functional Requirements | [specs/requirements/02-functional-requirements.md](specs/requirements/02-functional-requirements.md) |
| 03 | Non-Functional Requirements | [specs/requirements/03-non-functional-requirements.md](specs/requirements/03-non-functional-requirements.md) |
| 04 | Event Storming | [specs/event-storming/04-event-storming.md](specs/event-storming/04-event-storming.md) |
| 05 | Bounded Contexts | [specs/architecture/05-bounded-contexts.md](specs/architecture/05-bounded-contexts.md) |
| 06 | Domain Model | [specs/architecture/06-domain-model.md](specs/architecture/06-domain-model.md) |
| 07 | Architecture Decision Records | [specs/adrs/07-adrs.md](specs/adrs/07-adrs.md) |
| 08 | System Architecture | [specs/architecture/08-system-architecture.md](specs/architecture/08-system-architecture.md) |
| 09 | Multiplayer Architecture | [specs/architecture/09-multiplayer-architecture.md](specs/architecture/09-multiplayer-architecture.md) |
| 10 | Data Architecture | [specs/architecture/10-data-architecture.md](specs/architecture/10-data-architecture.md) |
| 11 | Data Pipeline Architecture | [specs/architecture/11-data-pipeline-architecture.md](specs/architecture/11-data-pipeline-architecture.md) |
| 12 | Game Engine Design | [specs/architecture/12-game-engine-design.md](specs/architecture/12-game-engine-design.md) |
| 13 | Database Strategy | [specs/architecture/13-database-strategy.md](specs/architecture/13-database-strategy.md) |
| 14 | Scalability Strategy | [specs/architecture/14-scalability-strategy.md](specs/architecture/14-scalability-strategy.md) |
| 15 | Risk Analysis | [specs/architecture/15-risk-analysis.md](specs/architecture/15-risk-analysis.md) |
| 16 | Technical Roadmap | [specs/roadmap/16-technical-roadmap.md](specs/roadmap/16-technical-roadmap.md) |
| 17 | Technical Debt Prevention | [specs/architecture/17-technical-debt-prevention.md](specs/architecture/17-technical-debt-prevention.md) |

---

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15, React 19, TypeScript, Tailwind, Zustand |
| Backend | NestJS, TypeScript, CQRS |
| Database | PostgreSQL 16, Redis 7 |
| Messaging | NATS JetStream |
| Monorepo | Turborepo + pnpm Workspaces |
| Infra | Docker, Kubernetes |
| Observability | OpenTelemetry, Prometheus, Grafana |

---

## Development Phases

| Phase | Description | Status |
|---|---|---|
| 1 | Engine Offline | Not started |
| 2 | Single Player Web | Not started |
| 3 | Backend Persistent | Not started |
| 4 | Multiplayer | Not started |
| 5 | Mobile | Not started |
| 6 | Global Ecosystem | Not started |

---

## Getting Started

> Implementation has not started. Check back after Spec-Kit is approved.
