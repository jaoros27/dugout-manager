# Product Vision — Dugout Manager

## 1. Problem Statement

Football management games in the Brazilian market are either:

- **Too lightweight** (Brasfoot): shallow simulation, no multiplayer, Windows-only, no live data pipeline.
- **Too heavy** (Football Manager): expensive, complex, PC-only, English-language, no mod-friendly architecture.
- **Too arcade** (FIFA Career Mode): entertainment over depth, no community ownership.

There is no modern, multiplayer-first, community-driven football manager built for Portuguese-speaking markets and the global football fanbase that combines deep simulation with accessible UX.

---

## 2. Vision Statement

> **Dugout Manager** is the football management simulation for those who want to feel like the real boss — deep enough to reward mastery, simple enough to pick up in minutes, multiplayer-ready from day one, and built to grow for decades.

---

## 3. Target Audience

### Primary
- Brazilian and Latin American football fans (18–40)
- Former Brasfoot players seeking a modern successor
- Football Manager fans who want a lighter, more accessible experience

### Secondary
- Competitive gamers interested in async multiplayer leagues
- Modding communities (FIFA data editors, Football Manager researchers)
- Esports organizers running virtual football leagues

---

## 4. Key Differentiators

| Dimension | Brasfoot | Football Manager | Dugout Manager |
|---|---|---|---|
| Platform | Windows only | PC/Mac | Web + Desktop + Mobile |
| Multiplayer | None | Limited | First-class, async + real-time |
| UX Complexity | Low | Very high | Medium (progressive disclosure) |
| Price | Free | ~R$200 | Free-to-play + premium |
| Data model | Static database | Licensed data | Open + licensed hybrid |
| Mod support | Manual edits | Workshop | First-class SDK |
| Architecture | Monolith | Monolith | Distributed, modular |
| Match Engine | Probabilistic | Full simulation | Hybrid event-driven |
| Language | Portuguese | English | Multi-language (PT first) |

---

## 5. Core Values

1. **Depth without complexity** — simulation depth accessible through progressive disclosure
2. **Community first** — mods, leagues, and data driven by the community
3. **Long-term reliability** — architecture built to evolve for 10+ years
4. **Fair competition** — anti-cheat and economy balance from the start
5. **Data transparency** — all simulation rules are auditable and testable

---

## 6. Success Metrics

### Phase 1 (Engine + Single Player)
- Match engine passes 100-season statistical validation
- Player attribute system correlates with real football outcomes
- Build time < 5 minutes on any dev machine

### Phase 2 (Backend Persistent)
- 1,000 registered users in first 3 months
- Save/load cycle < 2 seconds
- 99.5% uptime

### Phase 3 (Multiplayer)
- 10,000 active players
- Async league cycle completes in < 72 hours
- Anti-cheat blocks > 95% of detected exploits

### Phase 4 (Mobile)
- 100,000 installs
- Session retention D7 > 30%

### Phase 5 (Mods)
- 50+ community mod packs
- Mod SDK rated "easy to use" by 80%+ of modders

### Phase 6 (Global)
- 1,000,000 registered users
- 5+ official licensed leagues

---

## 7. Six-Phase Evolution

```
Phase 1: Engine Offline
  └─ Match engine, player model, AI, single-season simulation
  └─ No backend, no persistence, pure logic validation

Phase 2: Single Player
  └─ Full career mode, save system, UI, seasons, competitions
  └─ Web + Desktop (Electron or Tauri)

Phase 3: Backend Persistent
  └─ User accounts, cloud saves, leaderboards, async leagues
  └─ NestJS API + PostgreSQL + Redis

Phase 4: Multiplayer
  └─ Real-time async leagues, shared transfer market
  └─ NATS messaging, distributed match scheduling

Phase 5: Mobile
  └─ React Native or Flutter app
  └─ Same API, adapted UX for mobile-first flows

Phase 6: Ecosystem
  └─ Mod SDK, community data pipelines, API for third parties
  └─ Global leagues, esports infrastructure
```

---

## 8. Positioning Statement

For **football fans and competitive gamers** who want **deep management simulation without the steep learning curve**, Dugout Manager is a **multiplayer-first football management game** that delivers **Brasfoot's accessibility with Football Manager's depth** — playable on any device, evolved by the community, and built to last decades.

---

## 9. Non-Goals (What We Will NOT Build)

- 3D match visualization (Phase 1–3)
- Licensed player images/faces
- Pay-to-win mechanics
- Real-money transfer market
- Single-platform lock-in
- Hard dependency on any single data provider
