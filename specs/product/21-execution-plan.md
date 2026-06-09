# Execution Plan — Dugout Manager MVP

> Version: 1.0  
> Status: Draft  
> Reality: solo (user + Claude), zero budget, ~14 weeks to public launch

---

## 1. The Honest Framing

The 48-month, 6-phase roadmap (doc 16) is the **10-year vision**. This document is the
**actual execution plan** for the next ~14 weeks, calibrated to the real constraints:
one person directing, Claude implementing, no money, browser-only.

The two are not in conflict — this plan builds the foundation (engine, domain model,
web app) that the long-term vision extends. Nothing here is throwaway.

---

## 2. How "solo (user + Claude)" Actually Works

| Role | Who | Responsibility |
|---|---|---|
| Product owner / director | **User** | Decides scope, reviews every change, judges "is this fun?", playtests |
| Implementer | **Claude** | Writes code, tests, docs; explains every change so the user stays in control |

**The real bottleneck is review bandwidth, not typing speed.** Therefore:
- Work in small, reviewable increments — never giant unreviewable drops
- Every change comes with a plain explanation of what and why
- Code stays simple and readable on purpose — the user must be able to understand it
- The user playtests at every milestone; fun is judged by playing, not by specs

---

## 3. The 14-Week Plan (milestone-based, not date-locked)

> Weeks are effort milestones, not deadlines. Solo pace varies — the order matters more
> than the calendar.

### Milestone 1 — The Engine Heart (Weeks 1-4)
**Goal**: a match simulates and feels right.
- `packages/shared-types`: player, club, match types (from doc 06)
- `packages/match-engine`: hybrid simulation (doc 12) with tunable `EngineTuning`
- Generate fictional players + 20 clubs for one league
- CLI/test harness: simulate a match, print the result
- Monte Carlo sanity check: best team wins more, upsets happen
- **Playable surface**: none yet, but you can run a simulated season in the terminal

### Milestone 2 — A Season You Can Watch (Weeks 5-7)
**Goal**: a full season runs and produces a believable table.
- `packages/game-engine`: season loop, fixtures, standings, AI lineups
- Simulate all 38 matches, produce a final table
- Validate: the table looks realistic across 10 simulated seasons
- **Playable surface**: terminal — "run a season, see who won"

### Milestone 3 — The Game in a Browser (Weeks 8-11)
**Goal**: a stranger can play a season with a mouse.
- `apps/web`: Next.js app, deploy to Vercel (live URL from day one)
- Screens: club pick → squad view → lineup/formation → simulate match → match report → table
- Local save with IndexedDB (close the tab, come back, continue)
- **Playable surface**: a real, clickable game online

### Milestone 4 — Make It a Game, Not a Sim (Weeks 12-14)
**Goal**: meaningful decisions + the loop that retains.
- Simple transfer market (buy/sell vs AI, a transfer budget to respect)
- Multi-season continuity (players age, career continues)
- Basic analytics (Plausible/Umami) + error tracking (Sentry)
- UX polish pass; fix the rough edges playtesters hit
- **Playable surface**: the full MVP loop from doc 18 — ship it

### Launch
Share the URL. No install, no signup. Watch the doc 18 metrics. Decide next step from
real data, not guesses.

---

## 4. Production Readiness Checklist (before public launch)

Even a free, zero-budget MVP needs these to be "in production" responsibly:

| Item | Why | Effort |
|---|---|---|
| Error tracking (Sentry) | You will not see bugs users hit otherwise | Low |
| Privacy-friendly analytics | Know if the loop works; LGPD-safe | Low |
| Privacy Policy + basic Terms | Legally required even when free, esp. with analytics | Low |
| Save data is local + exportable | Don't lose a user's career to a browser clear | Low |
| Works on mobile browser | Most Brazilian traffic is mobile — responsive web is mandatory | Medium |
| CI runs tests on every push | Don't ship a broken build | Low |
| A landing page that explains the game | First impression, basic SEO | Low |

> Note: even though the **mobile app** is deferred to the far future, the **web MVP must
> work in a mobile browser**. Brazil is mobile-first. A desktop-only MVP would miss most
> of its audience. Responsive design is an MVP requirement, not a later phase.

---

## 5. Legal Minimum (free MVP)

Charging money raises the bar, but even a free game with analytics needs:
- **Privacy Policy** — what data is collected (anonymous analytics, error logs), LGPD basis
- **Terms of Use** — basic liability, acceptable use
- **No personal data collected without consent** — anonymous analytics only at MVP stage
- Done as simple, honest documents — not expensive lawyer work at this stage

When monetization turns on (doc 20), revisit with proper review: payment terms,
subscription cancellation, refund policy, possibly a CNPJ/MEI for receiving payments.

---

## 6. Risks Specific to This Plan

| Risk | Mitigation |
|---|---|
| **Solo burnout / scope creep** (BR-03) | Tiny milestones, ruthless scope cuts, ship the loop before adding anything |
| Engine feels random or unfair | Milestone 1 gates on "best team wins more" before building UI on top |
| Over-engineering for the 10-year vision | This plan is the contract: build only the MVP; defer everything else |
| Building UI before the engine is fun | Engine-first order (Milestones 1-2 before 3) prevents this |
| Losing user saves | Local save + export/import from Milestone 3 |
| Mobile audience missed | Responsive web required at launch |

---

## 7. The One Rule That Governs Everything

> **Ship the smallest fun thing, get real people playing, then let their behavior — not
> the spec — decide what comes next.**

The spec is the map of where this *can* go. This execution plan is how we take the first
real step without spending money or burning out. Every decision defers to: *does this get
us to a fun, playable, live game faster?*
