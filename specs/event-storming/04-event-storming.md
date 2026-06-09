# Event Storming — Dugout Manager

> Version: 1.0  
> Status: Draft  
> Method: Big Picture + Process Level Event Storming

---

## Legend

```
[EVENT]     — Orange: Domain Event (past tense, something that happened)
[COMMAND]   — Blue:   Command (intent, triggers an event)
[AGGREGATE] — Yellow: Aggregate (entity that enforces invariants)
[POLICY]    — Purple: Policy (when EVENT → COMMAND reaction)
[READ MODEL]— Green:  Read Model (query projection)
[EXTERNAL]  — Pink:   External System
[HOTSPOT]   — Red:    Area of uncertainty / complexity
```

---

## 1. Career Start Flow

```
[COMMAND] StartCareer(clubId, managerId)
    → [AGGREGATE] Career
    → [EVENT] CareerStarted(careerId, clubId, managerId, season)

[COMMAND] SelectFormation(careerId, formation)
    → [AGGREGATE] Tactics
    → [EVENT] TacticsConfigured(careerId, formation, instructions)
```

---

## 2. Transfer Market Flow

```
[COMMAND] SubmitTransferOffer(buyerClubId, sellerClubId, playerId, amount)
    → [AGGREGATE] TransferNegotiation
    → [EVENT] TransferOfferSubmitted(offerId, buyerClubId, sellerClubId, playerId, amount)

[POLICY] When TransferOfferSubmitted
    → [COMMAND] EvaluateTransferOffer(offerId)
    → [AGGREGATE] TransferNegotiation
    → [EVENT] TransferOfferAccepted | TransferOfferRejected | TransferCounterOffered

[COMMAND] SignPlayer(playerId, clubId, contractTerms)
    → [AGGREGATE] Contract
    → [EVENT] PlayerSigned(playerId, clubId, wage, duration, clauses)

[POLICY] When PlayerSigned
    → [COMMAND] RemovePlayerFromSquad(sellerClubId, playerId)
    → [COMMAND] AddPlayerToSquad(buyerClubId, playerId)
    → [EVENT] SquadUpdated(clubId)
    → [EVENT] TransferFeePaid(amount, from, to)
    → [EVENT] FinancialRecordCreated(type=TRANSFER_FEE, amount)

[HOTSPOT] Multiplayer shared transfer market: consistency when two managers
          bid for same player simultaneously.
```

---

## 3. Match Day Flow

```
[COMMAND] ProcessMatchDay(matchDayId, homeClubId, awayClubId)
    → [AGGREGATE] MatchSchedule
    → [EVENT] MatchDayStarted(matchDayId, date)

[COMMAND] SelectStartingEleven(careerId, matchId, lineup)
    → [AGGREGATE] Lineup
    → [EVENT] LineupSubmitted(matchId, clubId, lineup)

[COMMAND] SimulateMatch(matchId, homeTactics, awayTactics)
    → [AGGREGATE] Match (via Match Engine — stateless service)
    → [EVENT] MatchSimulated(matchId, events[], homeScore, awayScore, stats)

[POLICY] When MatchSimulated
    → [COMMAND] UpdatePlayerForm(players[], matchStats)
    → [COMMAND] UpdatePlayerFatigue(players[], minutesPlayed)
    → [COMMAND] UpdateLeagueTable(competitionId, matchResult)
    → [COMMAND] CheckForInjuries(players[], matchIntensity)
    → [EVENT] PlayerFormUpdated(playerId, newForm)
    → [EVENT] PlayerFatigueUpdated(playerId, fatigueLevel)
    → [EVENT] LeagueTableUpdated(competitionId, standings)
    → [EVENT] InjuryOccurred(playerId, type, recoveryWeeks) [conditional]

[POLICY] When InjuryOccurred
    → [COMMAND] UpdatePlayerAvailability(playerId, returnDate)
    → [EVENT] PlayerUnavailable(playerId, until)

[READ MODEL] MatchReport(matchId) ← MatchSimulated events
[READ MODEL] LeagueTable(competitionId) ← LeagueTableUpdated events
[READ MODEL] SquadHealth(clubId) ← PlayerUnavailable, PlayerFormUpdated
```

---

## 4. Season Progression Flow

```
[COMMAND] AdvanceWeek(seasonId)
    → [AGGREGATE] Season
    → [EVENT] WeekAdvanced(seasonId, week)

[POLICY] When WeekAdvanced
    → [COMMAND] ProcessTrainingSessions(clubIds[])
    → [COMMAND] ProcessScheduledMatches(matchIds[])
    → [COMMAND] ProcessContractExpirations()
    → [COMMAND] GenerateTransferRumors()

[COMMAND] EndSeason(seasonId)
    → [AGGREGATE] Season
    → [EVENT] SeasonEnded(seasonId, standings, winners)

[POLICY] When SeasonEnded
    → [COMMAND] ProcessPromotion(leagueId, promotedClubs[])
    → [COMMAND] ProcessRelegation(leagueId, relegatedClubs[])
    → [COMMAND] DistributePrizeMoney(competitionId, placements)
    → [COMMAND] GenerateYouthPlayers(clubs[])
    → [COMMAND] AgeUpAllPlayers()
    → [COMMAND] StartNewSeason()

    → [EVENT] ClubPromoted(clubId, fromLeague, toLeague)
    → [EVENT] ClubRelegated(clubId, fromLeague, toLeague)
    → [EVENT] PrizeMoneyDistributed(clubId, amount)
    → [EVENT] YouthPlayerGenerated(playerId, clubId, attributes)
    → [EVENT] PlayerAgedUp(playerId, newAge)
    → [EVENT] NewSeasonStarted(seasonId, year)
```

---

## 5. Player Development Flow

```
[COMMAND] RunTrainingSession(clubId, focus, intensity)
    → [AGGREGATE] TrainingSession
    → [EVENT] TrainingSessionCompleted(clubId, focus, playerResults[])

[POLICY] When TrainingSessionCompleted
    → [COMMAND] UpdatePlayerAttributes(playerId, attributeDeltas)
    → [EVENT] PlayerAttributeImproved(playerId, attribute, delta) [conditional on age/PA]
    → [EVENT] PlayerAttributeDeclined(playerId, attribute, delta) [conditional on age regression]

[COMMAND] AgeUpPlayer(playerId)
    → [AGGREGATE] Player
    → [EVENT] PlayerAgedUp(playerId, newAge)

[POLICY] When PlayerAgedUp AND player.age > regression_threshold(position)
    → [COMMAND] ApplyAgeRegression(playerId)
    → [EVENT] PlayerRegressed(playerId, affectedAttributes[])
```

---

## 6. Finance Flow

```
[COMMAND] CollectTicketRevenue(matchId, attendance, ticketPrice)
    → [AGGREGATE] Finance
    → [EVENT] RevenueCollected(clubId, type=TICKET, amount, matchId)

[COMMAND] PayWages(clubId, month)
    → [AGGREGATE] Finance
    → [EVENT] WagesPaid(clubId, totalAmount, month)

[POLICY] When WagesPaid AND balance < 0
    → [COMMAND] TriggerFinancialWarning(clubId)
    → [EVENT] FinancialWarningIssued(clubId, deficit)

[POLICY] When FinancialWarningIssued (consecutive 3 months)
    → [COMMAND] EnterAdministration(clubId)
    → [EVENT] ClubEnteredAdministration(clubId)

[HOTSPOT] Financial Fair Play enforcement in multiplayer — when to check and who enforces.
```

---

## 7. Multiplayer Flow

```
[EXTERNAL] Human Manager (Browser/Mobile)

[COMMAND] CreateLeague(hostManagerId, settings)
    → [AGGREGATE] MultiplayerLeague
    → [EVENT] LeagueCreated(leagueId, hostManagerId, settings)

[COMMAND] JoinLeague(leagueId, managerId)
    → [AGGREGATE] MultiplayerLeague
    → [EVENT] ManagerJoinedLeague(leagueId, managerId)

[COMMAND] SubmitMatchDayLineup(leagueId, managerId, matchDayId, lineup)
    → [AGGREGATE] MultiplayerMatchDay
    → [EVENT] LineupSubmitted(leagueId, managerId, matchDayId)

[POLICY] When ALL managers in league submitted lineup for matchDayId
    → [COMMAND] ProcessMultiplayerMatchDay(leagueId, matchDayId)
    → [EVENT] MultiplayerMatchDayProcessed(leagueId, matchDayId, results[])

[POLICY] When timer expires AND NOT all lineups submitted
    → [COMMAND] AutoSubmitDefaultLineup(managerId, matchDayId)
    → [EVENT] DefaultLineupSubmitted(managerId, matchDayId)

[HOTSPOT] Ordering guarantee: all match results in a match day must be
          processed atomically to prevent one manager seeing results
          before others in the same match day.
```

---

## 8. Domain Hotspots Summary

| # | Hotspot | Risk | Mitigation |
|---|---|---|---|
| H-01 | Shared transfer market concurrent bids | Race condition, unfair outcomes | Optimistic locking + event ordering |
| H-02 | Multiplayer match day atomicity | Partial results visible | Saga pattern, all-or-nothing commit |
| H-03 | Financial Fair Play enforcement | Timing of checks | End-of-season enforced batch check |
| H-04 | Match engine determinism with randomness | Replay inconsistency | Seeded PRNG, seed stored in event |
| H-05 | PA disclosure fairness | Exploit: players hiding true potential | PA never exposed in API response |
| H-06 | AI club behavior vs multiplayer balance | AI exploited or too passive | Separate AI strategy per competition type |
| H-07 | Data provider license expiry | Sudden data loss | Ports and Adapters, community fallback |
