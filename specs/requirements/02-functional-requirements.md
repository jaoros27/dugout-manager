# Functional Requirements — Dugout Manager

> Version: 1.0  
> Status: Draft  
> Phase coverage: All phases (tagged per feature)

---

## FR-01 — Club Management

| ID | Requirement | Phase |
|---|---|---|
| FR-01.1 | Manager can view full squad list with attributes summary | 2 |
| FR-01.2 | Manager can hire players from the global transfer market | 2 |
| FR-01.3 | Manager can sell players to other clubs (AI or human) | 2 |
| FR-01.4 | Manager can loan players in/out with configurable terms | 2 |
| FR-01.5 | Manager can negotiate and renew player contracts | 2 |
| FR-01.6 | Manager can hire/fire coaching staff | 2 |
| FR-01.7 | Manager can view and upgrade stadium capacity/facilities | 3 |
| FR-01.8 | Manager can view financial statements (P&L, balance sheet) | 2 |
| FR-01.9 | Manager can manage youth academy intake | 2 |
| FR-01.10 | Manager can set club-wide tactical philosophy | 2 |

---

## FR-02 — Player Model

| ID | Requirement | Phase |
|---|---|---|
| FR-02.1 | Each player has: Name, Nationality, Age, Height, Weight, Position | 1 |
| FR-02.2 | Technical attributes (8): Finishing, Passing, Crossing, Dribbling, Marking, Tackling, Heading, Ball Control | 1 |
| FR-02.3 | Physical attributes (5): Speed, Acceleration, Stamina, Strength, Jumping | 1 |
| FR-02.4 | Mental attributes (6): Leadership, Concentration, Determination, Decisions, Positioning, Teamwork | 1 |
| FR-02.5 | State attributes: Morale, Fatigue, Form, Injury status | 1 |
| FR-02.6 | CA (Current Ability): scalar 1–200 derived from weighted attributes | 1 |
| FR-02.7 | PA (Potential Ability): scalar 1–200 representing ceiling | 1 |
| FR-02.8 | Players develop toward PA based on age curve and training | 1 |
| FR-02.9 | Players regress from age 29–32 based on position and physical attributes | 1 |
| FR-02.10 | Player morale affects performance with ±15% variance | 1 |
| FR-02.11 | Player injuries reduce CA temporarily with position-specific recovery curves | 1 |
| FR-02.12 | Player value is computed from CA, age, contract length, and market conditions | 2 |

---

## FR-03 — Coaching Staff

| ID | Requirement | Phase |
|---|---|---|
| FR-03.1 | Head Coach has: tactical style, reputation (1–5), experience (years), coaching attributes | 2 |
| FR-03.2 | Coach tactical style influences match engine weighting | 1 |
| FR-03.3 | Coach reputation affects player morale and transfer negotiations | 2 |
| FR-03.4 | Coaching staff includes: assistant manager, fitness coach, youth coach, goalkeeping coach | 2 |
| FR-03.5 | Assistant manager provides pre-match analysis with probability hints | 3 |

---

## FR-04 — Financial System

| ID | Requirement | Phase |
|---|---|---|
| FR-04.1 | Revenue streams: ticket sales, sponsorship, TV rights, prize money, player sales, merchandise | 2 |
| FR-04.2 | Expense streams: wages, transfer fees, facilities, youth academy, staff | 2 |
| FR-04.3 | Weekly wage budget displayed with remaining capacity | 2 |
| FR-04.4 | Club reputation tier gates maximum transfer budget (prevents small clubs buying superstars) | 2 |
| FR-04.5 | Clubs can take board-approved loans with interest obligations | 3 |
| FR-04.6 | Financial Fair Play rules enforced: wage-to-revenue ratio capped at 70% | 3 |
| FR-04.7 | Clubs falling below financial thresholds enter administration with penalties | 3 |
| FR-04.8 | AI clubs have realistic financial profiles matching real-world tiers | 2 |

---

## FR-05 — Transfer Market

| ID | Requirement | Phase |
|---|---|---|
| FR-05.1 | Summer and winter transfer windows per competition calendar | 2 |
| FR-05.2 | Transfer types: permanent buy, permanent sell, loan in, loan out, free transfer, pre-contract | 2 |
| FR-05.3 | Offer system: bid → counter-offer → accept/reject with AI negotiation | 2 |
| FR-05.4 | Contract clauses: sell-on percentage, release clause, loan recall option | 2 |
| FR-05.5 | Player agents negotiated implicitly (wage demands affected by reputation) | 2 |
| FR-05.6 | In multiplayer: shared transfer market across all human managers | 4 |
| FR-05.7 | Transfer history log per player visible to all managers | 2 |

---

## FR-06 — Youth Academy

| ID | Requirement | Phase |
|---|---|---|
| FR-06.1 | Each club generates 3–8 youth players per season | 2 |
| FR-06.2 | Youth player PA is hidden; partially revealed over time and with better coaching | 2 |
| FR-06.3 | Youth development quality scales with academy facility investment | 2 |
| FR-06.4 | Youth players can be promoted to first team or sold | 2 |
| FR-06.5 | Nations have youth talent weighting (Brazil > Germany > etc.) | 2 |

---

## FR-07 — Competitions

| ID | Requirement | Phase |
|---|---|---|
| FR-07.1 | Support league format (round-robin, home/away) | 1 |
| FR-07.2 | Support cup format (knockout, two-legged) | 1 |
| FR-07.3 | Support group stage + knockout format (continental) | 2 |
| FR-07.4 | Promotion and relegation between tiers | 2 |
| FR-07.5 | Continental qualification spots from league position | 2 |
| FR-07.6 | Season calendar auto-generated without fixture conflicts | 2 |
| FR-07.7 | Prize money distributed by competition round reached | 2 |
| FR-07.8 | National team competitions run in parallel to club calendar | 3 |

---

## FR-08 — Season Management

| ID | Requirement | Phase |
|---|---|---|
| FR-08.1 | Season progresses week by week | 2 |
| FR-08.2 | Manager can simulate a week (auto-play all matches) | 2 |
| FR-08.3 | Manager can simulate a single match | 2 |
| FR-08.4 | End-of-season: award ceremonies, contract renewals, squad reassessment | 2 |
| FR-08.5 | Multi-season career mode (20+ seasons) | 2 |

---

## FR-09 — Tactics

| ID | Requirement | Phase |
|---|---|---|
| FR-09.1 | Formation selection (4-3-3, 4-4-2, 3-5-2, etc.) | 1 |
| FR-09.2 | Team instructions: pressing intensity, defensive line, tempo, width | 1 |
| FR-09.3 | Individual player instructions per position | 2 |
| FR-09.4 | Set-piece routines (corners, free kicks) | 2 |
| FR-09.5 | In-match tactical adjustment triggers (simulated, not real-time) | 2 |
| FR-09.6 | Tactical familiarity: players improve in their role over time | 2 |

---

## FR-10 — Training

| ID | Requirement | Phase |
|---|---|---|
| FR-10.1 | Weekly training schedule with focus: technical, physical, tactical, recovery | 2 |
| FR-10.2 | Training intensity affects fatigue accumulation | 2 |
| FR-10.3 | Targeted training accelerates specific attribute improvement | 2 |
| FR-10.4 | Overtraining increases injury risk | 2 |
| FR-10.5 | Match sharpness degrades if player does not play regularly | 2 |

---

## FR-11 — Injuries

| ID | Requirement | Phase |
|---|---|---|
| FR-11.1 | Injury types: minor (1–2 weeks), moderate (1–2 months), severe (3+ months), career-ending | 1 |
| FR-11.2 | Injury probability influenced by: fatigue, age, physical attributes, pitch condition | 1 |
| FR-11.3 | Injured players cannot be selected; CA temporarily reduced during recovery | 1 |
| FR-11.4 | Recovery speed influenced by fitness coaching quality | 2 |

---

## FR-12 — AI Club Behavior

| ID | Requirement | Phase |
|---|---|---|
| FR-12.1 | AI clubs sign players to fill squad gaps each transfer window | 2 |
| FR-12.2 | AI clubs sell surplus players when over wage budget | 2 |
| FR-12.3 | AI clubs renew expiring contracts proactively | 2 |
| FR-12.4 | AI clubs set starting XI based on best available CA for their formation | 1 |
| FR-12.5 | AI clubs make tactical adjustments if losing at half-time | 2 |
| FR-12.6 | AI financial planning: clubs never consistently overspend wages | 2 |
| FR-12.7 | AI youth promotion: top youth players promoted after 2–3 seasons | 2 |

---

## FR-13 — Multiplayer

| ID | Requirement | Phase |
|---|---|---|
| FR-13.1 | Players can create private leagues (invite-only) | 4 |
| FR-13.2 | Players can join public leagues (matchmaking by reputation tier) | 4 |
| FR-13.3 | Shared transfer market across all human managers in a league | 4 |
| FR-13.4 | Async match processing: all match days processed simultaneously when all managers ready | 4 |
| FR-13.5 | Turn timer: maximum 48h per match day before auto-simulate | 4 |
| FR-13.6 | Chat system per league | 4 |
| FR-13.7 | League commissioner tools: pause season, remove inactive manager, replace with AI | 4 |

---

## FR-14 — Mod System

| ID | Requirement | Phase |
|---|---|---|
| FR-14.1 | Season pack: replace player/club/competition data for a specific year | 6 |
| FR-14.2 | League pack: add new national leagues not in base game | 6 |
| FR-14.3 | Club pack: custom club crests, kits, stadium names | 6 |
| FR-14.4 | Competition pack: custom cup formats | 6 |
| FR-14.5 | Mods distributed via in-game mod browser | 6 |
| FR-14.6 | Mod validation: schema-checked before loading | 6 |

---

## FR-15 — Observability and Admin

| ID | Requirement | Phase |
|---|---|---|
| FR-15.1 | Admin panel to view active users, active seasons, system health | 3 |
| FR-15.2 | Match replay log: full event log per match exportable as JSON | 2 |
| FR-15.3 | Season simulation statistics: goals/game, upsets ratio, winner distribution | 2 |
