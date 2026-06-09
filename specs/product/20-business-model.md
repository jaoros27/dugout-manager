# Business Model — Dugout Manager

> Version: 1.0  
> Status: Draft  
> Model: Freemium + subscription, activated only after retention is proven

---

## 1. Core Principle

**Never charge before the game is proven fun and retentive.** The MVP is 100% free.
Monetization is switched on only when the success metrics in doc 18 are met (people
finish seasons and come back). Charging too early kills a fragile early community.

A second hard rule, inherited from the spec: **no pay-to-win, ever.** Paying users never
get sporting advantages. This protects competitive integrity (critical once multiplayer
exists) and the game's reputation.

---

## 2. The Free / Paid Line

The line is drawn on **convenience, scale, and depth of history** — never on competitive
advantage.

| Capability | Free | Premium (subscription) |
|---|---|---|
| Single-player career | ✅ Unlimited | ✅ |
| Number of simultaneous save slots | 1 | Unlimited |
| Leagues available | Base league(s) | All leagues |
| Season history / statistics depth | Last 1 season | Full career history + analytics |
| Cloud save (cross-device) | ❌ (local only) | ✅ |
| Private multiplayer leagues (future) | Join only | **Create + commission** |
| Cosmetic customization | Basic | Full |
| Early access to new features | ❌ | ✅ |
| Match engine / player attributes | Identical | Identical (no P2W) |

The free game must be genuinely complete and fun on its own. Premium sells **more**
(more saves, more leagues, more history, cloud, hosting multiplayer) — not a better
chance of winning.

---

## 3. Pricing (initial hypothesis)

| Plan | Price | Notes |
|---|---|---|
| Free | R$0 | The whole core game |
| Premium Monthly | ~R$9,90/month | Targets impulse, low commitment |
| Premium Annual | ~R$79/year | ~33% discount, better cash flow + retention |

Prices are hypotheses to be tested, calibrated to the Brazilian market (the primary
audience). Mercado Pago is the preferred gateway for Brazil (Pix support is essential —
Pix is how Brazil pays).

> **Pix matters**: a large share of the Brazilian audience does not use credit cards.
> Supporting Pix (via Mercado Pago) can be the difference between a 1% and a 5%
> conversion rate. This is a first-class requirement, not a nice-to-have.

---

## 4. When to Turn Monetization On

```
Gate to enable subscriptions (ALL must be true):
  □ MVP retention metrics met (doc 18 §6)
  □ Accounts + cloud save shipped (the first premium hook)
  □ At least 2-3 "premium-worthy" features exist (extra slots, more leagues, full history)
  □ Mercado Pago/Stripe integration tested end-to-end including Pix
  □ Terms of Use + Privacy Policy published (see doc 21 legal section)
```

Turning it on before these are true risks charging for an experience that is not yet
worth paying for.

---

## 5. Revenue Sustainability Math (illustrative)

The point of monetization is to make the project self-funding, not to get rich early.

```
Assume free tier on Cloudflare Pages + Supabase free = ~R$0 until ~50k users
First paid infra need (~R$200-500/mo) arises around 10k-50k active users

Break-even at R$9,90/mo:
  Need ~30-50 paying subscribers to cover R$300-500/mo infra
  At a conservative 2% conversion: 30 payers = 1,500 active users
  At 5% conversion (good Pix-enabled freemium): 30 payers = 600 active users

Conclusion: the project becomes self-funding at a very small scale.
A few hundred to a couple thousand active users can cover its own costs.
```

This is the whole reason for the bootstrap strategy: keep fixed costs at zero until a
tiny paying base can cover them, then grow within revenue.

---

## 6. What We Will NOT Do (monetization anti-patterns)

| Anti-pattern | Why we avoid it |
|---|---|
| Pay-to-win (buy better players/results) | Destroys competitive integrity and trust |
| Loot boxes / gacha | Gambling-classification risk (Risk LR-05), predatory |
| Energy/timer mechanics ("wait or pay") | Hostile to the relaxed manager experience |
| Selling user data | LGPD risk, reputation suicide |
| Ads in core gameplay | Cheapens the product; minimal revenue at this scale |
| Paywalling the core loop | The free game must be complete — that's what drives word of mouth |

---

## 7. Long-Term Revenue Streams (post-MVP, in likely order)

1. **Subscription** (primary) — recurring, predictable, aligns with ongoing service
2. **Cosmetics** — club crests, kit designers, manager avatars (one-time purchases)
3. **Premium multiplayer hosting** — creating/commissioning private leagues
4. **Official licensed data packs** (far future) — once revenue justifies licensing cost

Subscription is the backbone. Everything else is supplementary and only added when it
doesn't complicate the core experience.
