# CLAUDE.md — Dugout Manager

This file is loaded automatically at the start of every session. It is the entry point
for continuing work across chats.

## What this project is

**Dugout Manager** — a modern football management game inspired by Brasfoot, intended for
real production. Multiplayer-first long-term vision, but bootstrapped solo for now.

GitHub: https://github.com/jaoros27/dugout-manager

## Hard constraints (these govern every decision)

- **Team**: solo — the user directs and reviews, Claude implements. Review bandwidth is
  the bottleneck. Work in small, reviewable increments and explain every change.
- **Budget**: effectively zero. Free tiers only. Cost never precedes revenue.
- **First launch target**: 3-4 months, browser-only MVP.
- **Revenue**: freemium + subscription, activated only after retention is proven.

## Conventions

- **Commits**: title only — no body, no "Co-Authored-By" attribution.
- Code must stay simple and readable so the user can review it.
- Engine-first: prove the match engine is fun before building UI on top.

## Working agreement: keep status current (MANDATORY)

**Whenever a milestone or meaningful step is completed, update the "Current status"
section below BEFORE ending the session** — mark what is done (✅) and set the next step
(⬜). This is part of the Definition of Done for every step, not an optional extra. A new
chat relies on this section being accurate to know where to continue. If status here is
stale, the handoff to the next session is broken.

## Where the plan lives

- **Near-term execution**: `specs/product/` (docs 18-21) — MVP, bootstrap stack, business
  model, execution plan. **Start here.**
- **10-year vision/architecture**: `specs/` (docs 01-17) — what the MVP grows into.

## Current status

- ✅ Full spec-kit complete (21 documents)
- ✅ Monorepo folder structure created
- ⬜ **No implementation started yet**
- ⬜ Next step: **Milestone 1 — The Engine Heart** (see `specs/product/21-execution-plan.md`):
  `packages/shared-types` + `packages/match-engine` (hybrid simulation, doc 12) +
  fictional data, validated by a Monte Carlo sanity check (best team wins more, upsets happen).

## How to continue in a new chat

Just say: *"Continue Dugout Manager — we're starting Milestone 1, the match engine."*
This file + the saved memory will give the full context. Update the "Current status"
section above as milestones complete.
