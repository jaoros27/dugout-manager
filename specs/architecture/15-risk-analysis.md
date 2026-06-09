# Risk Analysis — Dugout Manager

> Version: 1.0  
> Status: Draft  
> Format: Risk ID | Description | Probability | Impact | Mitigation

---

## 1. Technical Risks

### TR-01 — Match Engine Calibration Failure
- **Description**: Engine produces unrealistic results (dominant teams win 90%+ of matches, or random outcomes dominate CA quality)
- **Probability**: Medium (calibration is iterative and subjective)
- **Impact**: High (player trust, game credibility)
- **Mitigation**: Monte Carlo test suite with hard assertions as CI gate. Any change to engine weights must pass 1,000-season statistical tests before merge. Calibration team role defined from Phase 1.

### TR-02 — PostgreSQL Write Bottleneck at Scale
- **Description**: At 1M users, event log insertion rate exceeds PostgreSQL write capacity
- **Probability**: Low in Phase 1–4, Medium in Phase 5+
- **Impact**: Medium (performance degradation, not data loss)
- **Mitigation**: Partitioned tables from Phase 4. EventStoreDB migration path documented and architecturally prepared (event bus abstraction in place). Monitoring alert at 70% write capacity.

### TR-03 — NATS JetStream Scalability Ceiling
- **Description**: NATS cannot handle 500k+ messages/day required at Phase 6
- **Probability**: Low (NATS handles millions/second in benchmarks)
- **Impact**: Medium (requires Kafka migration)
- **Mitigation**: Event bus abstracted behind interface. Kafka migration is infra-level change, not domain change. Re-evaluate at 200k active users.

### TR-04 — Match Engine WASM Port Complexity
- **Description**: TypeScript match engine cannot meet < 50ms target at 10k+ concurrent simulations
- **Probability**: Low (benchmarks suggest TS is sufficient through Phase 4)
- **Impact**: Medium (requires Rust WASM port)
- **Mitigation**: Engine designed as pure function with clean interface from day 1. WASM port is a drop-in replacement, not a rewrite. Trigger: 1,000-match batch takes > 60 seconds.

### TR-05 — Multiplayer Consistency Bug
- **Description**: A race condition allows a manager to see opponent lineup before submitting their own, or match results are published partially
- **Probability**: Low-Medium (distributed systems are hard)
- **Impact**: Very High (trust destruction, competitive integrity)
- **Mitigation**: Match day gate pattern with atomic commit. Integration tests simulate concurrent submissions. Manual penetration testing of the gate before Phase 4 launch.

### TR-06 — Data Provider API Breaking Change
- **Description**: SportMonks or API-Football changes their schema or terminates service without notice
- **Probability**: Medium (APIs change over time)
- **Impact**: Medium (breaks data import, not live game)
- **Mitigation**: Ports and Adapters pattern — provider change is isolated to adapter layer. Dual-source strategy: always have 2 active providers. Community data pack as fallback. Never expose external IDs to the game domain.

### TR-07 — Single Player Save Corruption
- **Description**: A bug causes a career save to be partially written, corrupting game state
- **Probability**: Low (transactional writes)
- **Impact**: High (player data loss, trust)
- **Mitigation**: All writes in PostgreSQL transactions. Event sourcing on match engine means state is always reconstructible from events. Automatic save backups (last 3 states) for single player mode.

---

## 2. Legal and Licensing Risks

### LR-01 — Unauthorized Use of Player Names/Likenesses
- **Description**: Base game ships with real player names without FIFPRO license
- **Probability**: High if real names used without license
- **Impact**: Very High (cease and desist, financial penalties)
- **Mitigation**: Ship with fictitious data only. Mod system distributes real names with mod author's legal responsibility. First-party licensing negotiated per region when revenue justifies.

### LR-02 — Competition Branding Infringement
- **Description**: Using "Série A", "Champions League", or CBF/UEFA logos without license
- **Probability**: High without license
- **Impact**: High (legal action from rights holders)
- **Mitigation**: Generic competition names in base game ("Liga Brasil", "Continental Cup"). Branded versions only under license. Clear naming convention documented.

### LR-03 — Mod System Hosting Liability
- **Description**: Community mod hosting copyrighted content (player photos, licensed logos)
- **Probability**: Medium (community will try to upload unlicensed content)
- **Impact**: Medium (DMCA takedowns, platform liability)
- **Mitigation**: Mod validation pipeline rejects image/media uploads. DMCA takedown process documented. Mod authors accept ToS liability. Text-only data packs only in Phase 5–6.

### LR-04 — LGPD / GDPR Non-Compliance
- **Description**: Failure to handle user data deletion requests or data breaches properly
- **Probability**: Low (with proper implementation)
- **Impact**: High (regulatory fines up to 2% of revenue, reputation)
- **Mitigation**: PII stored separately from game data. Deletion pipeline built in Phase 3. Privacy policy reviewed by legal before Phase 3 launch. No third-party analytics without explicit consent.

### LR-05 — Gambling Classification Risk
- **Description**: Certain jurisdictions may classify virtual leagues/prediction features as gambling
- **Probability**: Low (no real-money exchange in current design)
- **Impact**: Very High (platform bans, regulatory action)
- **Mitigation**: No real-money wagering, ever. No loot boxes. All in-game purchases are cosmetic or one-time. Legal review of any monetization feature before launch.

---

## 3. Business Risks

### BR-01 — Community Doesn't Adopt Without Real Player Names
- **Description**: Players expect real names at launch; fictitious data drives them away
- **Probability**: Medium
- **Impact**: High (adoption barrier)
- **Mitigation**: Launch with mod-ready architecture and community pack on day 1. Transparent roadmap showing licensing plan. Partnership with Football Manager data community early.

### BR-02 — Football Manager Enters Same Market
- **Description**: Sports Interactive releases a web/mobile version targeting our exact market
- **Probability**: Medium-Low (they have shown no interest in Brazil-first or async multiplayer)
- **Impact**: High
- **Mitigation**: Differentiate on: Brazilian-first content, async multiplayer depth, open data/mod ecosystem. Build community loyalty before SI reaches the market.

### BR-03 — Key Person Risk (Single Developer)
- **Description**: Project stalls if core developer leaves or is unavailable
- **Probability**: Medium (early stage)
- **Impact**: Very High (project failure)
- **Mitigation**: Architecture documentation from day 1 (this spec kit). Spec Kit as onboarding guide. Engine designed for contributor-friendly participation. Open source components where appropriate.

### BR-04 — Monetization Model Fails to Cover Infrastructure Costs
- **Description**: Freemium model doesn't convert enough paying users to cover ~$3k/month at Phase 4
- **Probability**: Medium
- **Impact**: High (sustainability)
- **Mitigation**: Infrastructure costs kept minimal through Phase 1–2. Paid features: premium leagues, extended career history, cosmetics. No pay-to-win. Target 3–5% conversion rate at 100k users = 3,000–5,000 paying users. $5/month = $15k–$25k/month at Phase 4.

### BR-05 — Player Retention (Session Design)
- **Description**: Async multiplayer has long gaps between sessions; players churn before match day
- **Probability**: Medium
- **Impact**: High (multiplayer league collapse)
- **Mitigation**: Push notifications for match day deadlines. Daily tasks (training, scouting) give reasons to return between match days. AI auto-pilot mode for inactive managers prevents league collapse.

---

## 4. Risk Matrix

```
           LOW IMPACT    MEDIUM IMPACT    HIGH IMPACT    VERY HIGH IMPACT
LOW PROB       —              TR-03           TR-04             TR-07
MEDIUM PROB   BR-05          TR-01           TR-06            BR-01, BR-02
HIGH PROB      —             LR-03           BR-04          LR-01, LR-02
CRITICAL       —              —             TR-05           BR-03, LR-04
```

---

## 5. Risk Review Schedule

| Phase | Review Trigger |
|---|---|
| Phase 1 | Engine calibration results (monthly) |
| Phase 2 | Beta user feedback (monthly) |
| Phase 3 | Pre-launch legal review (mandatory) |
| Phase 4 | Multiplayer stress test (pre-launch) |
| Phase 5 | Mobile platform compliance review |
| Phase 6 | Licensing audit (annual) |

High and Very High impact risks are reviewed before every major phase transition.
