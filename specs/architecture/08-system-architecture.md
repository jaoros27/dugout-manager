# System Architecture — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                             CLIENTS                                        │
│                                                                            │
│   ┌────────────┐    ┌────────────────┐    ┌──────────────┐                │
│   │  Web App   │    │  Desktop App   │    │  Mobile App  │                │
│   │ (Next.js)  │    │ (Tauri/Electron│    │ (React Native│                │
│   └─────┬──────┘    └───────┬────────┘    └──────┬───────┘                │
│         └──────────────────┼──────────────────────┘                       │
└─────────────────────────────┼──────────────────────────────────────────────┘
                              │ HTTPS / WSS
┌─────────────────────────────┼──────────────────────────────────────────────┐
│                          API GATEWAY                                       │
│                     (Nginx / Traefik)                                      │
│              Rate Limiting │ Auth │ Routing │ TLS                          │
└──────────────────┬──────────┴────────────────────────────────────────────-─┘
                   │
        ┌──────────┼──────────────────────────────────────┐
        │          │                                      │
        ▼          ▼                                      ▼
┌───────────┐ ┌────────────┐                    ┌─────────────────┐
│  API      │ │ WebSocket  │                    │  Admin API      │
│  Service  │ │  Gateway   │                    │  (NestJS)       │
│ (NestJS)  │ │ (NestJS)   │                    └─────────────────┘
└─────┬─────┘ └─────┬──────┘
      │             │
      └──────┬───────┘
             │
┌────────────▼──────────────────────────────────────────────────────────────┐
│                         INTERNAL SERVICES                                  │
│                                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │    Career    │  │    Club      │  │   Transfer   │  │   Finance    │  │
│  │   Service    │  │   Service    │  │    Service   │  │   Service    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                  │                  │          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Competition │  │    Season    │  │ Multiplayer  │  │  Notification│  │
│  │   Service    │  │   Service    │  │   Service    │  │   Service    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         └─────────────────┴──────────────────┴──────────────────┘          │
│                                    │                                        │
│                         ┌──────────▼──────────┐                            │
│                         │   Match Engine      │                            │
│                         │   Worker Pool       │                            │
│                         │ (stateless workers) │                            │
│                         └──────────┬──────────┘                            │
└────────────────────────────────────┼─────────────────────────────────────-─┘
                                     │
┌────────────────────────────────────┼──────────────────────────────────────┐
│                       DATA LAYER                                           │
│                                                                            │
│  ┌───────────────────────────────┐ ┌──────────────────────────────────┐   │
│  │   PostgreSQL (primary)        │ │   Redis                           │   │
│  │   ├── game_schema            │ │   ├── sessions                    │   │
│  │   ├── events_schema          │ │   ├── projections                 │   │
│  │   ├── finance_schema         │ │   ├── leaderboards                │   │
│  │   └── identity_schema        │ │   └── pub/sub                     │   │
│  └───────────────────────────────┘ └──────────────────────────────────┘   │
│                                                                            │
│  ┌───────────────────────────────┐                                        │
│  │   NATS JetStream              │                                        │
│  │   (async event bus)           │                                        │
│  └───────────────────────────────┘                                        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Responsibilities

### API Gateway
- TLS termination
- Rate limiting (100 req/min per IP, 1000 req/min authenticated)
- JWT validation (stateless, validates signature + expiry)
- Routing to downstream services
- Request/response logging with trace IDs

### API Service (NestJS — main)
- Handles all game domain HTTP requests
- Issues commands via NestJS Command Bus
- Returns read model projections via Query Bus
- Does NOT contain business logic — delegates to domain layer

### WebSocket Gateway (NestJS)
- Real-time notifications: match results, transfer updates, match day gates
- Not used for game state synchronization (too complex for async turns)
- Phase 4 only

### Match Engine Worker Pool
- Stateless Node.js worker processes
- Receives `SimulateMatch` jobs from NATS queue
- Runs pure match simulation
- Publishes `MatchSimulated` event with full result
- Scales horizontally — spin up more workers for batch simulation
- No database access during simulation

### Domain Services (per Bounded Context)
- Each service owns its own database schema
- Communicates with other services via NATS events (async) or direct HTTP (sync, query only)
- No shared database access between services

---

## 3. Request Flow Example — Submit Transfer Offer

```
Client
  │  POST /transfers/offers
  ▼
API Gateway
  │  JWT validation, rate limit check
  ▼
API Service (TransferController)
  │  Parse DTO, validate schema
  ▼
NestJS Command Bus
  │  SubmitTransferOfferCommand
  ▼
SubmitTransferOfferHandler
  │  Load TransferWindow aggregate (check window is open)
  │  Load buyer Club (check budget available)
  │  Load Player (check not already under offer)
  │  Create TransferOffer aggregate
  │  Persist to PostgreSQL
  │  Publish TransferOfferSubmitted event to NATS
  ▼
NATS → AI Club Evaluation Subscriber (if seller is AI club)
  │  AI evaluates offer
  │  Publishes TransferOfferAccepted/Rejected
  ▼
NATS → API Service subscriber
  │  Updates TransferNegotiation state
  │  Sends notification via WebSocket to buyer manager
```

---

## 4. Match Day Processing Flow (Multiplayer)

```
MatchDayGate receives all lineups (or deadline expires)
  │
  ▼
NATS: MatchDayGateClosed event published
  │
  ▼
Match Engine Orchestrator
  │  Reads all fixtures for this match day
  │  Publishes N SimulateMatch jobs to NATS queue
  │
  ▼
Match Engine Worker Pool (parallel processing)
  │  Each worker picks up one SimulateMatch job
  │  Simulates match (pure function)
  │  Publishes MatchSimulated event
  │
  ▼
Competition Service (subscriber)
  │  Aggregates all MatchSimulated events
  │  When all matches complete: publishes MatchDayCompleted
  │
  ▼
Season Service (subscriber)
  │  Advances season week
  │  Updates standings, form, fatigue, injuries
  │  Notifies clients via WebSocket
```

---

## 5. Phase Deployment Topology

### Phase 1–2 (Single Player, Web)
```
Single Docker Compose:
  - next.js app (serves frontend)
  - nestjs api (monolith, all services in one process)
  - postgresql
  - redis

Match engine: runs in-process (no worker pool needed)
NATS: not yet deployed (events handled in-memory via NestJS event bus)
```

### Phase 3 (Backend Persistent, 10k users)
```
Docker Compose or single Kubernetes cluster:
  - web (2 replicas)
  - api (3 replicas, load balanced)
  - match-engine-worker (2 replicas)
  - postgresql (primary + 1 read replica)
  - redis (single node)
  - nats (single node)
  - prometheus + grafana

Horizontal scaling via replica count.
```

### Phase 4 (Multiplayer, 100k users)
```
Kubernetes:
  - web (autoscaled 2–10 replicas)
  - api (autoscaled 3–15 replicas)
  - match-engine-worker (autoscaled 5–50 replicas)
  - websocket-gateway (autoscaled 2–8 replicas, sticky sessions)
  - postgresql (primary + 2 read replicas + PgBouncer)
  - redis cluster (3 nodes)
  - nats cluster (3 nodes, JetStream enabled)
  - data-pipeline worker (1–3 replicas, cron-based)
```

### Phase 6 (Global, 1M users)
```
Multi-region Kubernetes:
  - CDN for static assets
  - Region-affinity for multiplayer leagues (EU, Americas, Asia)
  - Read replicas per region
  - NATS supercluster (multi-region)
  - Global leaderboards via Redis Cluster
```

---

## 6. Inter-Service Communication Rules

| Pattern | Used For | Protocol |
|---|---|---|
| Sync HTTP | Query: read projections needed immediately | HTTP/2 REST |
| Async NATS | State changes: domain events | NATS JetStream |
| WebSocket | Real-time client notifications | WSS |
| Database views | Cross-context read queries | PostgreSQL views (read-only) |

**Rule**: A service NEVER writes to another service's database schema. All cross-context writes go via events.

---

## 7. Security Boundaries

```
Internet → API Gateway (TLS, rate limit, JWT)
         → API Service (authenticated)
         → Domain Services (internal network only, no public exposure)
         → Match Engine Workers (internal network only)
         → Databases (internal network only, no public exposure)
```

The match engine worker and all internal services are never directly accessible from the internet. All traffic routes through the API Gateway.
