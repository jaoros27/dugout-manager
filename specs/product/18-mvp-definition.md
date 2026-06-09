# MVP Definition — Dugout Manager

> Version: 1.0  
> Status: Draft  
> Constraint context: solo (user + Claude), zero budget, 3-4 month launch, browser-only

---

## 1. The One Question the MVP Must Answer

> **"Is managing a club through a season fun enough that people come back?"**

Nothing else matters yet. Not multiplayer, not real data, not mobile, not mods, not
accounts, not money. If the core loop is not fun, none of those save the project. If it
is fun, all of those become worth building.

---

## 2. What the MVP IS

A single-player football manager that runs entirely in the browser, with a local save.

### Core Loop (the entire game)
```
1. Pick a club from one league (20 fictional Brazilian-style clubs)
2. See your squad (players with attributes)
3. Set your formation and starting XI
4. Simulate the next match → watch a text match report
5. See the updated league table
6. Repeat through a full season (38 matches)
7. Between matches: buy/sell players in a simple transfer market
8. End of season: champion crowned, see if you finished where you wanted
9. Start a new season (squad ages, you continue)
```

That is the whole game. It is genuinely playable and genuinely fun if the match engine
and squad-building feel right.

---

## 3. MVP Feature List (the ONLY features)

| Feature | Included? | Notes |
|---|---|---|
| 1 league, 20 clubs | ✅ | Fictional clubs and players (zero licensing risk) |
| Player attributes + CA | ✅ | Full attribute model from doc 06 — this is the heart |
| Match simulation | ✅ | Hybrid engine from doc 12 (the most important code) |
| Text match report | ✅ | Goals, cards, key events — no 2D/3D |
| Formation + starting XI | ✅ | A handful of formations (4-4-2, 4-3-3, 3-5-2) |
| League table | ✅ | Standings update after each match |
| Simple transfer market | ✅ | Buy/sell vs AI clubs, basic fee negotiation |
| Club finances (basic) | ✅ | One number: transfer budget. Spend it, don't overspend |
| Season progression | ✅ | Play 38 matches, crown a champion |
| Multi-season | ✅ | Players age, you keep your career going |
| AI club lineups | ✅ | AI picks its best XI (doc 12 §11) |
| Local save (IndexedDB) | ✅ | No account, no server |

### Explicitly NOT in the MVP
Multiplayer · user accounts · backend/database · real player data · mobile app · mods ·
youth academy · training schedules · injuries (beyond a simple flag) · coaching staff ·
detailed finances · cup competitions · promotion/relegation · continental competitions ·
payments/subscription · push notifications.

All of these are real features from the spec — they come **after** the loop is proven fun.

---

## 4. Why Browser-Only, No Backend

| Decision | Reason |
|---|---|
| Runs in browser | Match engine is pure TS — runs client-side with zero server |
| Save in IndexedDB | No database, no account, no server cost |
| Deploy on Vercel free | R$0 hosting for a Next.js app |
| No login | Removes a whole subsystem; play instantly |

**Infrastructure cost of the MVP: R$0/month.** (Only optional cost: a domain, ~R$40/year.)

This is what makes a 3-4 month, zero-budget launch realistic.

---

## 5. Definition of "Done" for the MVP

The MVP ships when a stranger can:
1. Open a URL (no install, no signup)
2. Pick a club and understand their squad within 2 minutes
3. Play a full 38-match season without hitting a bug or dead end
4. Win or lose based on decisions that feel meaningful (not random)
5. Want to start a second season

If a playtester finishes a season and immediately starts another **without being asked**,
the MVP has succeeded.

---

## 6. Success Metrics (post-launch)

Since there are no accounts, metrics are lightweight and privacy-friendly (e.g.,
self-hosted Plausible or simple anonymous events):

| Metric | Target | Meaning |
|---|---|---|
| Completes ≥1 full season | > 30% of players | The loop holds attention |
| Starts a 2nd season | > 15% of players | The loop is genuinely fun |
| Avg session length | > 15 min | Engagement is real |
| Return within 7 days | > 20% | Retention signal — green light for accounts/monetization |

If these hit: build accounts + cloud save + subscription (Phase 3 of the original spec).
If they miss: fix the fun before building anything else.

---

## 7. What Comes Right After the MVP (in order)

Only once the loop is proven fun:

1. **More content** — more leagues, cup competition, promotion/relegation (still browser-only, still free)
2. **Accounts + cloud save** — Supabase free tier (the first backend, still R$0)
3. **Monetization** — freemium limits + subscription (see doc 20)
4. **Then** the big-spec items: multiplayer, mobile, mods, real data

This sequence keeps cost at R$0 until the moment revenue can cover it.
