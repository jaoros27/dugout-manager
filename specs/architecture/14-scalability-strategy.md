# Scalability Strategy — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. Scalability Targets

| Milestone | Active Users | Concurrent | Matches/Day | Data Size |
|---|---|---|---|---|
| Phase 3 Launch | 10,000 | 1,000 | 50,000 | ~50 GB |
| Phase 4 | 100,000 | 10,000 | 500,000 | ~500 GB |
| Phase 6 | 1,000,000 | 100,000 | 5,000,000 | ~5 TB |

---

## 2. Bottleneck Analysis

### Match Simulation (Primary Bottleneck)
- Single match: < 50ms (target)
- 5,000 matches/day (Phase 3) = low load
- 5,000,000 matches/day (Phase 6) = ~58 matches/second average, 10× burst = 580/second
- **Solution**: Stateless match engine workers, horizontally scalable via NATS queue

### API Layer
- 10,000 concurrent users → ~5,000 req/sec assuming 2 req/user/second
- NestJS handles ~10,000 req/sec per instance under typical load
- **Solution**: 2–5 API replicas at Phase 4, HPA scaling at Phase 6

### Database (PostgreSQL)
- Primary bottleneck at Phase 4+: write amplification from events
- 1M active careers × 1 write/week = manageable
- Match events: 100 events × 500,000 matches/day = 50M rows/day at Phase 6
- **Solution**: Partitioned events table + read replicas + Redis projection caching

### Transfer Market (Spike Load)
- Transfer windows create write spikes: 100,000 users submitting offers in 72 hours
- Peak: ~5,000 offers/hour = ~1.4 offers/second (very low!)
- Optimistic locking handles concurrency well at this scale

---

## 3. Horizontal Scaling Design

### Match Engine Workers

```
Properties:
  - Fully stateless (no shared state between workers)
  - Receives job from NATS queue: {matchId, homeContext, awayContext, seed}
  - Writes result to PostgreSQL + publishes MatchSimulated event
  - No worker-to-worker communication

Scaling:
  Phase 3: 2 workers
  Phase 4: 5–20 workers (HPA based on queue depth)
  Phase 6: 20–100 workers (HPA, multi-region)

NATS queue configuration:
  Subject: match.simulate
  Consumer group: match-engine-workers
  Max deliver: 3 (retry on failure)
  Ack wait: 120s (match simulation timeout)
```

### API Service

```
Scaling triggers (Kubernetes HPA):
  CPU > 60%: scale up
  CPU < 30% for 5 min: scale down
  Min replicas: 2 (Phase 3), 3 (Phase 4+)
  Max replicas: 10 (Phase 3), 50 (Phase 4+)

Stateless design requirements:
  - No in-memory session state (all in Redis)
  - No sticky sessions (except WebSocket gateway)
  - JWT validation is stateless (signature verification only)
```

### WebSocket Gateway

```
Scaling challenge: sticky sessions required for WebSocket connections
Solution: 
  - nginx with ip_hash load balancing (Phase 4)
  - Or: WebSocket via Redis pub/sub fan-out (workers subscribe to Redis, push to clients)
  
Redis pub/sub fan-out pattern:
  NATS event → API subscriber → Redis PUBLISH channel:league:{id}
  WebSocket workers → Redis SUBSCRIBE channel:league:{id}
  WebSocket workers → push to connected clients

This allows WebSocket workers to be stateless (connection affinity via Redis)
```

---

## 4. Caching Strategy

### Three-Layer Cache

```
Layer 1: Browser cache (Next.js static generation)
  - Competition definitions: 24h
  - Club metadata: 1h
  - Player profiles (non-active): 30 min

Layer 2: Redis (application cache)
  - Active squad: 10 min
  - League standings: 1 min (during season)
  - Transfer market: 30 sec
  - Player search results: 2 min

Layer 3: PostgreSQL materialized views
  - Financial balances: refreshed every 5 min
  - Season statistics: refreshed every hour
```

### Cache Invalidation Rules

```
Event → Invalidate
PlayerAttributeChanged → proj:player:{playerId}
ContractSigned → proj:squad:{clubId}
TransferCompleted → proj:squad:{old_club}, proj:squad:{new_club}, proj:market:*
MatchSimulated → proj:standings:{seasonId} (delay 10 sec for batch invalidation)
WeekAdvanced → proj:squad:{all clubs with matches}
```

Batch invalidation: after a match day completes, invalidate in bulk (single Redis pipeline call).

---

## 5. Database Scaling Path

### Phase 3 (10k users)
```
Single primary PostgreSQL
+ 1 read replica for transfer market and player search
+ PgBouncer (pool_size = 50)
+ Redis single node
```

### Phase 4 (100k users)
```
Primary PostgreSQL
+ 2 read replicas (load-balanced by pgbouncer)
+ PgBouncer (pool_size = 200)
+ Events table partitioned by month
+ Redis cluster (3 nodes)
+ Materialized view refresh moved to background job
```

### Phase 6 (1M users)
```
Multi-region deployment:
  Americas (primary write region)
    └── Primary PostgreSQL
    └── 3 read replicas
  Europe (read region)
    └── 2 read replicas (streaming from Americas primary)
  Asia (read region)
    └── 2 read replicas

Redis Cluster: 6 nodes (3 primary + 3 replica), hash-sharded
Event archive: events > 2 years old moved to cold storage (S3 Parquet)
OLAP queries: dbt pipeline materializes daily aggregates for analytics
```

---

## 6. CDN Strategy

```
Static assets (Next.js build artifacts): CDN from Phase 2
  - Cache-Control: max-age=31536000, immutable (versioned filenames)
  - CDN: Cloudflare (free tier), Vercel Edge (if on Vercel), or AWS CloudFront

API responses: NOT cached at CDN by default
  Exception: competition metadata, club metadata, player profiles (public data)
  → Edge caching with 1h TTL + stale-while-revalidate

WebSocket: NOT CDN-compatible → direct to WebSocket gateway
Match replays: stored on S3, served via CDN (immutable after creation)
```

---

## 7. Cost Modeling

### Phase 3 (10k users) — ~$400/month

| Service | Spec | Cost |
|---|---|---|
| API server | 2× 2vCPU/4GB | $80 |
| Match engine workers | 2× 2vCPU/4GB | $80 |
| PostgreSQL (primary + replica) | 2vCPU/8GB each | $120 |
| Redis | 1vCPU/2GB | $30 |
| NATS | 1vCPU/1GB | $20 |
| CDN + egress | — | $30 |
| Monitoring | Grafana Cloud free + OTEL | $40 |

### Phase 4 (100k users) — ~$3,000/month

| Service | Spec | Cost |
|---|---|---|
| API servers | 6× 4vCPU/8GB | $600 |
| Match engine workers | 10× 4vCPU/8GB | $1,000 |
| PostgreSQL (1 primary + 2 replicas) | 8vCPU/32GB each | $800 |
| Redis cluster (3 nodes) | 4vCPU/16GB each | $300 |
| NATS cluster (3 nodes) | 2vCPU/4GB each | $120 |
| CDN + egress | high volume | $100 |
| Monitoring + alerting | Grafana Cloud | $80 |

### Phase 6 (1M users) — ~$25,000/month

Requires multi-region, dedicated DB instances, and ops team. Estimate based on similar-scale game backends. Revenue at this scale should comfortably cover infrastructure costs.

---

## 8. Kubernetes HPA Configuration (Phase 4+)

```yaml
# Match engine workers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    name: match-engine-worker
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: External
      external:
        metric:
          name: nats_consumer_pending_messages
          selector:
            matchLabels:
              consumer: match-engine-workers
        target:
          type: AverageValue
          averageValue: "100"   # scale up when > 100 pending per worker
```

Match engine scales on NATS queue depth — the most accurate signal for this workload.

---

## 9. Observability-Driven Scaling

Scaling decisions must be data-driven:

```
Metrics collected:
  - NATS queue depth per consumer group (primary match engine scaling signal)
  - API p95/p99 latency per route (scale API if > 200ms p95)
  - PostgreSQL active connections (scale PgBouncer pool if > 80% used)
  - Redis memory usage (scale cluster if > 70% used)
  - Match simulation time distribution (alert if p99 > 500ms)

Dashboards:
  - Match day processing: queue depth, worker count, time to complete
  - Transfer window: offer rate, acceptance rate, conflict rate
  - System health: CPU, memory, error rates per service
  - Business metrics: daily active users, seasons started, matches simulated
```
