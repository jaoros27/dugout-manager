# Bootstrap Stack — Dugout Manager

> Version: 1.0  
> Status: Draft  
> Goal: ship the MVP with R$0/month infrastructure using only free tiers

---

## 1. Principle

Every tool in the MVP stack must have a **genuinely free tier sufficient for launch**.
No credit card traps, no "free for 30 days". We add paid services only when revenue
covers them.

---

## 2. MVP Stack (browser-only, R$0/month)

| Concern | Tool | Free tier | Why |
|---|---|---|---|
| Framework | Next.js + React + TypeScript | Open source | Same as the long-term spec — no rework later |
| Styling | Tailwind CSS | Open source | Fast, consistent UI |
| State | Zustand | Open source | Lightweight game-state management |
| Match engine | Pure TypeScript package | Open source | Runs client-side, no server |
| Local save | IndexedDB (via `idb` or Dexie) | Browser built-in | No database needed |
| Hosting | **Vercel Hobby** | Free (non-commercial) | One-command deploy of Next.js |
| Monorepo | Turborepo + pnpm | Open source | As per ADR-001 |
| Source control | GitHub | Free | Already set up |
| CI | GitHub Actions | 2,000 min/month free | Run tests on every push |
| Analytics | **Plausible (self-host) or Umami** | Free / open source | Privacy-friendly, LGPD-safe, no cookie banner |
| Error tracking | **Sentry** | Free tier (5k events/mo) | Catch runtime bugs in production |
| Domain (optional) | Registro.br `.com.br` | ~R$40/year | Only real cost; can launch on `*.vercel.app` first |

**Total monthly cost: R$0.** (Optional domain: ~R$3/month amortized.)

> ⚠️ Vercel's Hobby tier is for non-commercial use. The free MVP (no payments) is fine.
> The day you add a subscription, move to Vercel Pro (~US$20/mo) or Cloudflare Pages
> (free, allows commercial). Plan this switch with monetization, not before.

---

## 3. Stack When You Add Accounts (still ~R$0)

When the MVP proves fun and you need cloud saves + login:

| Concern | Tool | Free tier |
|---|---|---|
| Database + Auth + Storage | **Supabase** | 500MB DB, 50k monthly active users, auth included |
| Alternative DB | **Neon** (Postgres) | 0.5GB, generous free tier |
| Hosting (commercial-OK) | **Cloudflare Pages** | Unlimited requests, free |

Supabase is the single best bootstrap choice: Postgres + authentication + file storage +
row-level security, all free up to a real scale. It replaces NestJS + a separate auth
system + a separate DB for the early backend — dramatically less to build and maintain
solo.

> This **temporarily supersedes ADR-003 (NestJS)** for the bootstrap phase. NestJS
> becomes worth it only when the domain logic outgrows what Supabase edge functions can
> handle cleanly — likely around multiplayer (original Phase 4). Documented as a
> conscious, reversible trade-off.

---

## 4. Stack When You Add Payments

| Concern | Tool | Cost |
|---|---|---|
| Payments (Brazil) | **Mercado Pago** or **Stripe** | % per transaction, no fixed monthly fee |
| Subscription mgmt | Stripe Billing / Mercado Pago Assinaturas | Included in transaction % |

No fixed cost — you only pay a percentage when you actually earn. This is the right model
for a bootstrapped product: infrastructure cost scales with revenue, never ahead of it.

---

## 5. What We Deliberately Defer

These are in the long-term spec but have **no place in a zero-budget MVP**:

| Deferred | Until |
|---|---|
| Kubernetes | 100k+ users (original Phase 4+) — likely never self-managed; use managed platforms |
| NATS / Kafka | Real multiplayer with async match days |
| Redis | When DB query load actually needs caching (Supabase handles early scale) |
| PgBouncer | When connection count is a real bottleneck |
| Multi-region | 1M users — a champagne problem |
| Prometheus/Grafana self-hosted | Sentry + Vercel/Supabase dashboards cover early needs for free |
| OpenTelemetry full tracing | When you have multiple services to trace across |

**Rule**: every item above is added only when a concrete, measured problem demands it —
never preemptively. Premature infrastructure is the fastest way to burn a solo budget and
solo time.

---

## 6. Cost Evolution Summary

```
MVP (browser-only, free):           R$0/month
+ Accounts (Supabase free):         R$0/month
+ Payments (Mercado Pago/Stripe):   % of revenue only — starts paying for itself
+ Scale (paid tiers):               only when revenue > cost
```

The entire bootstrap path is designed so that **cost never precedes revenue**. You can
take this from idea to paying users without ever paying a fixed monthly bill out of pocket
(beyond an optional ~R$40/year domain).
