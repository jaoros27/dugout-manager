# Game Engine Design — Dugout Manager

> Version: 1.0  
> Status: Draft

---

## 1. Engine Principles

1. **Pure function**: `simulate(context, seed) → result`. No I/O, no side effects.
2. **Deterministic**: Same seed + same inputs always produce same output.
3. **Probabilistic**: Best team wins more often — zebras exist.
4. **Position-aware**: A striker's attributes are weighted differently than a goalkeeper's.
5. **Context-sensitive**: Tactics, morale, fatigue, weather, crowd pressure all matter.
6. **Auditable**: Full event log produced per match for debugging and analytics.
7. **Tunable without recompiling**: Every magic number lives in config, not code (see §1.1).

### 1.1 Tunable Constants Principle (MANDATORY)

Every numeric constant in this document — `baseRate = 1.2`, `injuryProbability = 0.003`,
shot thresholds (`> 14`), home-advantage modifiers (`+0.05`), fatigue decay rates,
momentum weights, position weight tables — is a **calibration parameter, not a code
literal**.

All such constants live in a single versioned config object (`EngineTuning`), passed
into `simulate()` as part of the context (or bound at engine construction). Rules:

- **No stochastic or weighting constant is hardcoded** in an algorithm function.
- The Monte Carlo calibration suite (§13) tunes `EngineTuning`, never patches code.
- Each `EngineTuning` version is hashed and stored — a career/league pins its tuning
  version so results never silently change under a player mid-season.
- Mods (Phase 6) may ship alternative `EngineTuning` profiles (e.g., "high-scoring
  arcade" vs "realistic").

```typescript
interface EngineTuning {
  version: string
  possession: { homeAdvantage: number; varianceSigma: number; clampMin: number; clampMax: number }
  chances:    { baseRatePerPhase: number; poissonScale: number }
  shooting:   { onTargetThreshold: number; goalThreshold: number; noiseSigma: number }
  injuries:   { baseProbabilityPerPhase: number; fatigueWeight: number; ageWeight: number }
  fatigue:    { decayPerPhase: number; highPressPenalty: number }
  momentum:   { scoringBoost: number; phaseWinBoost: number }
  positionWeights: Record<PositionCode, AttributeWeightTable>
}
```

The consequence: calibrating the engine is editing a config and re-running the
statistical suite — **never recompiling**. This is what makes "annual update" and
community tuning operationally cheap.

---

## 2. Match Simulation Architecture

### Top-Level Flow

```
simulate(MatchContext, seed: UUID) → MatchResult

MatchContext {
  home: TeamContext
  away: TeamContext
  conditions: MatchConditions
}

TeamContext {
  lineup: PlayerSlot[11]
  substitutes: PlayerSlot[9]
  tactics: TacticsSnapshot
  coach: CoachSnapshot
}
```

### Phase-Based Simulation

The match is divided into **18 phases of 5 minutes** (0–90 min) + stoppage time phases.

```
for phase = 1 to 18:
  1. computePossession(home, away, tactics) → PossessionSplit
  2. generateChances(home, away, possession, phase) → Chance[]
  3. resolveChances(chances, players, seed) → Event[]
  4. applyCardLogic(events, players, seed) → CardEvent[]
  5. checkForInjuries(phase, intensity, players, seed) → InjuryEvent[]
  6. applyFatigueDecay(phase, players) → updated PlayerStates
  7. applyMomentum(phase, score, home, away) → MomentumModifier

if extraTime:
  run phases 19–24 (4x5 min = 30 min extra)
  if still tied: runPenalties(home, away, seed)
```

---

## 3. Possession Algorithm

```
basePossession(home) = midfield_rating(home) / (midfield_rating(home) + midfield_rating(away))

midfield_rating(team) =
  avg(passing, positioning, decisions) of CM/CDM/CAM players
  × coach_tactical_modifier(pressing_style)
  × home_advantage_modifier(venue)   -- home: +0.05, away: -0.05

final_possession = clamp(basePossession + random_variance, 0.30, 0.70)
```

Possession split is recalculated each phase. A team that scores gains momentum → increased possession in next phase.

---

## 4. Chance Generation Algorithm

```
chancesForTeam(team, possession, phase):
  baseRate = 1.2 chances per phase at 50% possession
  
  attackRating(team) =
    avg(finishing, dribbling, crossing) of attacking players
    weighted by position (ST > LW/RW > CAM)
    
  defenseRating(opponent) =
    avg(tackling, marking, positioning, strength) of defensive players
    weighted by position (CB > CDM > LB/RB)
    
  chanceCount = Poisson(λ = baseRate × possession × (attackRating / defenseRating))
  
  for each chance:
    chanceQuality = Beta(attackRating, defenseRating)  -- 0.0–1.0
    chanceType = weighted_random([SHOT, HEADER, FREE_KICK, CORNER])
```

---

## 5. Shot Resolution Algorithm

```
resolveShot(chance, attacker, goalkeeper, seed):
  
  shotAccuracy =
    attacker.finishing × position_modifier(chanceType)
    × form_modifier(attacker.form)
    × fatigue_modifier(attacker.fatigue)
    + noise(seed, σ=2)   -- seeded random variance
    
  saveChance =
    goalkeeper.reflexes × goalkeeper.positioning
    × goalkeeper.form_modifier
    + noise(seed, σ=2)

  if shotAccuracy > 14:
    onTarget = true
    if shotAccuracy > saveChance:
      return GOAL
    else:
      return SAVE
  else if shotAccuracy > 9:
    onTarget = Random(seed) < 0.4
    return onTarget ? SAVE : MISS
  else:
    return MISS
```

---

## 6. Player Attribute System

### Current Ability (CA)

CA is a single scalar (1–200) that represents overall quality. It is derived from weighted attributes per position:

```
CA_GK  = w1×reflexes + w2×aerial + w3×distribution + w4×concentration + ...
CA_CB  = w1×tackling + w2×marking + w3×heading + w4×strength + w5×positioning + ...
CA_ST  = w1×finishing + w2×dribbling + w3×decisions + w4×speed + w5×heading + ...
```

Weights are position-specific and sum to 1.0. CA is recomputed after every attribute change.

### Potential Ability (PA)

PA is fixed at player creation. It represents the ceiling the player can reach under ideal conditions.

- Elite: PA 170–200 (world-class players)
- Good: PA 130–169 (solid top-flight players)
- Average: PA 100–129 (mid-table quality)
- Development: PA 60–99 (youth with potential)

PA is **never exposed in API responses** — discovering a player's PA is part of the game.

### Development Curve

```
growthRate(player, week):
  if player.age < peakAge(player.position):
    rate = BASE_GROWTH × (1 - player.CA / player.PA)  -- faster growth when far from PA
    rate × training_modifier(trainingFocus, player.position)
    rate × determination_modifier(player.determination)
    rate × facility_modifier(club.trainingGrade)
  else:
    return 0  -- at peak, no growth

regressionRate(player, week):
  if player.age > regressionAge(player.position):
    rate = BASE_REGRESSION × (player.age - regressionAge) × physical_weight(player.position)
    -- physical positions (ST, LW, LB) regress faster than technical/mental positions (GK, CB)
  else:
    return 0
```

### Peak Age by Position

| Position | Peak Age | Regression Start |
|---|---|---|
| GK | 28–33 | 36 |
| CB | 26–31 | 33 |
| LB/RB | 25–29 | 31 |
| CDM | 26–30 | 32 |
| CM | 25–29 | 31 |
| CAM | 24–28 | 30 |
| LW/RW | 23–27 | 29 |
| ST | 23–27 | 29 |

---

## 7. Morale and Form System

### Morale
- Updated weekly based on: results, playing time, wage satisfaction, relationship with coach
- Effects match performance: VERY_HIGH +8% to all stats, VERY_LOW -15%
- Cannot be directly set — only changed by events

### Form
- Rolling average of last 5 performance ratings (1–10)
- Performance rating computed post-match from: match events (goals, assists, key passes, errors), minutes played, defensive actions
- Form affects chance of selection and individual match contribution weight

---

## 8. Tactical Impact on Simulation

Tactics affect the simulation through modifiers, not absolute overrides:

| Tactic Setting | Effect |
|---|---|
| High Press | +10% chance generation when possession lost, +15% fatigue accumulation |
| Deep Block | -20% opponent chance quality when in defensive third, -5% own possession |
| Short Passing | +5% possession, +3% build-up chance count, -10% direct attack efficiency |
| Long Ball | -5% possession, +15% aerial chance count, boosted by aerial physical stats |
| Wide Play | +20% crossing chances, boosted by crossing + LW/RW dribbling |
| Central Play | -10% wide chances, +15% central penetration weight |

---

## 9. Home Advantage and Conditions

```
HomeAdvantage:
  +5% possession base
  +5% morale start
  +8% crowd pressure boost in high-attendance games (> 80% stadium capacity)
  Referee modifier: not explicitly modeled (absorbed into random variance)

Weather modifiers:
  RAIN:        -10% to passing accuracy, +5% injury risk
  SNOW:        -15% passing, -10% speed, +10% injury risk
  STRONG_WIND: -20% long passes, -15% crossing, +10% aerial variance
  CLEAR:       No modifier

Pitch quality modifiers:
  WATERLOGGED: -25% dribbling effectiveness, +5% fatigue
  POOR:        -10% technical skills
  GOOD:        No modifier
  PERFECT:     +5% technical play effectiveness
```

---

## 10. Injury System

```
injuryProbability(player, phase, matchIntensity):
  base = 0.003  -- ~0.3% per phase ≈ 2 injuries per match per 100 matches
  
  modifiers:
    + (player.fatigue / 100) × 0.002    -- fatigued players more vulnerable
    + max(0, (player.age - 30) × 0.0005) -- older players more fragile
    - (player.strength / 20) × 0.001    -- stronger players slightly less
    + (matchIntensity - 0.5) × 0.002    -- tough matches more injuries
    
  if random(seed) < injuryProbability:
    severity = weighted_random_by_probability:
      MINOR    (1–14 days):   60%
      MODERATE (15–60 days):  30%
      SEVERE   (61–180 days):  9%
      CAREER_ENDING:           1%
```

---

## 11. AI Club Intelligence

### Lineup Selection
```
selectXI(squad, formation, opposition):
  1. Remove unavailable players (injured, suspended)
  2. Score each player for each position slot:
     slot_score = CA × position_suitability × form × morale
  3. Assign best scorer to each slot via greedy bipartite matching
  4. If multiple formations available, select formation that maximizes total slot score
```

### Transfer Strategy
```
identifyNeeds(squad):
  for each position:
    if < 2 players of acceptable CA: position is a NEED
    
evaluateOffer(offer, player, club):
  fair_value = valuationService.estimate(player)
  if offer >= fair_value × 0.90: ACCEPT_LIKELY
  if offer >= fair_value × 0.70: NEGOTIATE
  if offer < fair_value × 0.70: REJECT
  
decideTarget(budget, needs):
  candidates = database.find(position=NEED, CA_range, wage_range=budget)
  sort by (CA - current_equivalent) / estimated_fee  -- efficiency score
  bid on top 3 candidates
```

---

## 12. Economic System Design

### Goals
- Prevent one club from dominating indefinitely (but allow dynasties over 5–10 seasons)
- Prevent mass bankruptcies in lower leagues
- Keep transfer inflation under control

### Mechanisms

**Wage Cap Escalation**: Wage budgets grow proportionally with league TV money. Top-flight TV money distributed with diminishing returns (1st gets 2.5× last place, not 10×).

**Prize Money Scaling**: Continental prize money prevents total top-flight dominance — a 2nd-tier club winning its cup earns enough to challenge tier-1 for one season.

**Transfer Levy**: A 5% transaction tax on all permanent transfers above €5M. Proceeds distributed to lower-tier clubs as development grants.

**Soft Wage Cap**: No hard cap. But clubs spending >75% wages-to-revenue for 3 consecutive seasons face board spending restrictions.

**Youth Injection**: Each season, every club generates youth players (more with better facilities). This ensures a steady supply of new talent and prevents a talent vacuum.

### Economic Validation Tests (Monte Carlo)

```
Test suite runs 1,000 simulated seasons:
  - Gini coefficient of club wealth must stay 0.35–0.65 (not too equal, not too extreme)
  - At least 5 different clubs must win top division per 20 seasons
  - < 5% of clubs enter administration per 20 seasons
  - Average player value must not grow > 15% per season (inflation check)
  - Bottom-tier clubs must not consistently have zero viable transfer targets
```

---

## 13. Engine Validation Strategy

### Unit Tests
- Each algorithm function tested with known inputs/outputs
- Edge cases: goalkeeper with CA=1 vs striker with CA=200 → score heavily weighted
- Determinism test: run same match 1,000 times with same seed → identical results

### Statistical Validation (Monte Carlo)
```
Simulation: 100-season run, 20 clubs, top tier
Assertions:
  - Top CA team wins league in 45–65% of seasons (not 80%+, not 30%-)
  - Expected goals per match: 2.5–3.0
  - Home win rate: 45–55%
  - Draw rate: 20–30%
  - Away win rate: 20–30%
  - Upset rate (lower CA team wins): 25–35% of matches
  - Zero-zero results: 5–10%
  - Average goals by GK CA=20: < 0.8/match
  - Average goals by GK CA=5: > 2.2/match
```

### Calibration Loop
If any statistical assertion fails after a code change: the change is blocked until calibrated.
Calibration happens by adjusting modifier weights — never by patching specific scenarios.
