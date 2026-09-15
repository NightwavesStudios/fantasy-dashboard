# Fantasy Football Dashboard Technical Reference

## 1. Application Shape

The dashboard is a browser-only application implemented in `index.html`.

- HTML defines the tabs, forms, tables, modals, and empty render targets.
- CSS is embedded in the `<style>` block.
- JavaScript is embedded in the main `<script>` block.
- `player-value-adjustments.js` is loaded before the main script and provides optional `window.PLAYER_VALUE_ADJUSTMENTS` entries.
- There is no application server, database, build step, or backend proxy in this repository.
- Data is fetched directly from public APIs by the browser.
- `localStorage` stores theme, active/recent leagues, external ranking sources, and ranking movement history.

The main runtime state is held in globals such as `LEAGUE_ID`, `SEASON`, `PLATFORM`, `leagueData`, `powerModelsCache`, and the FantasyCalc lookup maps.

## 2. User Inputs and Runtime Configuration

The home screen accepts:

- Platform: `espn` or `sleeper`.
- League ID.
- Season, defaulting to `2026`.

`applyLeague()` validates that the league ID is numeric, updates the global configuration, rebuilds API URLs, clears relevant caches, loads the league, and saves it to recent leagues.

The active league is stored under `leagueToolsActive`. Recent leagues are stored under `leagueToolsLeagues`.

## 3. Network Sources

### ESPN league data

Base URL:

```text
https://lm-api-reads.fantasy.espn.com/apis/v3/games/ffl/seasons/{SEASON}/segments/0/leagues/{LEAGUE_ID}
```

The initial URL requests:

90% starter FantasyCalc value
10% bench FantasyCalc value
```

The weekly projection lineup is separate from the eight-player FantasyCalc starter/bench split. The former models weekly lineup scoring; the latter models roster value allocation.

- Teams and team names.
- Owners and members.
- Rosters and player objects.
- Player positions, injury status, stats, and roster slots.
- Records and points for.
- Current scoring period.
- Basic scoreboard data.
- ESPN trade-block status.

`fetchLeague()` performs the initial ESPN request. It also separately loads bye weeks and projection enrichment.

### ESPN league settings and schedule

The playoff URL adds:

```text
&view=mSettings&view=mMatchup
```

Used by `setupPlayoffOdds()` for playoff settings, matchup schedules, and schedule data.

### ESPN pro-team schedules and bye weeks

```text
https://lm-api-reads.fantasy.espn.com/apis/v3/games/ffl/seasons/{SEASON}?view=proTeamSchedules_wl
```

`fetchByeWeeks()` reads `settings.proTeams` and creates `byeByProTeam`, mapping ESPN `proTeamId` values to bye weeks.

`getByeWeek()` checks, in order:

1. Sleeper-specific `_sleeperBye`.
2. A direct `bye_week` property.
3. The ESPN pro-team bye map.

Failure is non-fatal. The map becomes empty and the UI displays unavailable bye information.

### ESPN waiver pool

The waiver URL requests `view=kona_player_info`.

`fetchWaiverPool()` sends an `X-Fantasy-Filter` header requesting free agents and waivers, with supported slot IDs, a 250-player limit, and descending ownership sorting.

Used for:

- Free-agent and waiver lists.
- Ownership percentage and ownership change.
- Injury labels.
- Bye weeks.
- FantasyCalc value and overall rank.
- Same-position replacement suggestions for a selected team.

### ESPN weekly projections

`enrichEspnProjections()` requests each week from 1 through 17:

```text
?view=mMatchup&view=mRoster&scoringPeriodId={WEEK}
```

Each request is made concurrently with `Promise.allSettled()`.

The function collects weekly projected stats, actual stats, ceilings, and variance by player ID. It stores the result on roster players as:

- `_weeklyProjections`
- `_weeklyActuals`
- `_projEnrich`

If every request fails, projection enrichment fails gracefully and normal fallback projection logic remains available.

### ESPN live matchup data

`fetchLiveMatchupWeek(week)` requests:

```text
{API_URL}&view=mMatchup&view=mRoster&scoringPeriodId={week}&_live={timestamp}
```

The timestamp and `cache: "no-store"` are used to avoid stale browser responses.

Used for:

- Locked player status.
- Actual points already scored.
- Live roster totals.
- Current matchup projections.
- Snapshot cards.
- Matchup advice and late swaps.
- Refreshing playoff inputs.

### ESPN league communication history

`setupLeagueHistory()` requests:

```text
{API_URL}&view=kona_league_communication
```

`parseTradesFromTopics()` filters transaction topics for message type `230`, then groups player movements into historical trades.

Used for the trade-history grading feature. Historical trade grades use current FantasyCalc values, not historical values.

### FantasyCalc

Current provider:

```text
https://api.fantasycalc.com/values/current?isDynasty=false&numQbs=2&numTeams=10&ppr=1
```

Configuration means:

- Redraft values.
- 10 teams.
- Superflex-style two-QB value settings.
- Full PPR.

`fetchFantasyCalcEntries()` converts each response row into:

```js
{
  espnId,
  name,
  value,
  overallRank,
  positionRank
}
```

`applyFantasyCalcValueAdjustments()` adds local name-based adjustments from `player-value-adjustments.js` before values are indexed.

`ingestEntries()` creates:

- `fcByEspnId`: exact ESPN ID to metadata.
- `fcByName`: normalized name to metadata.

Name matching uses both strict normalization and loose normalization that removes punctuation, suffixes such as Jr/Sr/II, and trailing jersey-style numbers.

FantasyCalc is used for:

- Trade values.
- Roster value.
- Starter and bench value.
- Position strength.
- VORP/scarcity signals.
- Waiver replacement values.
- Trade fairness and trade suggestions.
- Historical trade grades.

### Sleeper league data

Sleeper endpoints use:

```text
https://api.sleeper.app/v1/league/{LEAGUE_ID}
https://api.sleeper.app/v1/league/{LEAGUE_ID}/users
https://api.sleeper.app/v1/league/{LEAGUE_ID}/rosters
https://api.sleeper.app/v1/state/nfl
https://api.sleeper.app/v1/league/{LEAGUE_ID}/matchups/{WEEK}
https://api.sleeper.app/v1/players/nfl
```

`fetchSleeperLeagueData()` loads league metadata, users, rosters, NFL state, the current matchup week, and available weeks of schedule data.

The complete Sleeper player dump is compacted and cached in `localStorage` for 24 hours under:

- `sleeperPlayersNfl_v1`
- `sleeperPlayersNfl_v1_ts`

Sleeper data is transformed into an ESPN-like internal shape so the rest of the application can reuse the same model and render functions.

Sleeper does not provide the same ESPN projection fields to this app. `applySleeperProjectionProxies()` creates synthetic weekly projections from position baselines and FantasyCalc overall rank.

## 4. Request Handling and Failure Behavior

All requests go through `fetchWithTimeout()`.

- It creates an `AbortController`.
- It aborts after the requested timeout.
- It always clears the timeout in `finally`.
- HTTP errors are converted into thrown errors by the caller.

Typical timeouts:

- FantasyCalc: 15 seconds.
- ESPN league data: 20 seconds.
- ESPN bye weeks: 15 seconds.
- ESPN weekly projections: 20 seconds each.
- Sleeper league requests: 20 seconds.
- Sleeper player dump: 60 seconds.
- Live matchup data: 20 seconds.

The main league load treats FantasyCalc, bye weeks, and projection enrichment as partially optional. If one optional source fails, the dashboard continues with warnings and fallbacks where possible.

## 5. Player and Position Modeling

`POS_MAP` converts ESPN position IDs:

```text
1  QB
2  RB
3  WR
4  TE
5  K
16 D/ST
```

The dashboard's primary skill-position model uses QB, RB, WR, and TE. K and D/ST are generally excluded from the ranking lineup calculations.

`injuryLabel()` converts ESPN injury strings into display labels such as Out, IR, Doubtful, Q, and Probable.

## 6. FantasyCalc Starter and Bench Split

The FantasyCalc roster-value split is controlled by `getStarterPlayers()` and `getBenchPlayers()`.

The starter set is exactly:

- Top 2 QBs by FantasyCalc value.
- Top 2 RBs by FantasyCalc value.
- Top 2 WRs by FantasyCalc value.
- Top 2 remaining RB/WR flex players by FantasyCalc value.

This produces eight FantasyCalc starters. All roster players left out of that selection count as bench, including TEs, K/D/ST, IR, and any additional QB/RB/WR players.

`starterValue()` sums the selected eight players. `benchValue()` sums every unselected player.

The power model's FantasyCalc component uses:

```text
90% starter FantasyCalc value
10% bench FantasyCalc value
```

## 7. Completed-Week PPG

`buildTeamModels()` calculates actual PPG separately from projected PPG.

The completed-game count is based on:

```text
completedGames = max(0, scoringPeriodId - 1)
```

The current scoring period is deliberately excluded because it may still be in progress.

The season points source is:

1. `_directEspnPointsFor`, if available.
2. `team.record.overall.pointsFor`.
3. `team.points`.

Starter player actual points from the current scoring period are summed from `statSourceId === 0` and removed from the season points total. Therefore:

```text
completedPointsFor = seasonPoints - currentWeekStarterPoints
PPG = completedPointsFor / completedGames
```

If there are no completed games, points for and PPG are zero for the actual-performance signal.

The direct ESPN game count, when available, is capped at `completedGames` so it cannot reintroduce the in-progress week.

## 8. Projection Profiles

`espnProjectionFields()` returns projected points, floor, ceiling, season average, boom score, and standard deviation.

Projection source priority:

1. Enriched current-week ESPN projection.
2. Weekly player stat projection.
3. Season average.
4. Position-based fallback.

Injury behavior:

- OUT or IR: projection, floor, and ceiling become zero.
- DOUBTFUL: floor is reduced to 55% of normal.
- QUESTIONABLE: floor is reduced to 85% of normal.

Position fallback profiles are defined in `BOOM_BUST`. QBs have tighter ranges, while RBs and WRs have wider floor/ceiling ranges.

## 9. Lineup Calculations

`optimalLineupTotal()` and `optimalLineupTotalForWeek()` select a lineup by a requested numeric field such as `proj`, `projFloor`, `projCeil`, or `val`.

The weekly lineup shape is:

- 1 QB.
- 2 RB.
- 2 WR.
- 1 TE.
- 2 RB/WR/TE flex.
- Best remaining eligible player as a superflex slot.

Bye-week players are excluded by `optimalLineupTotalForWeek()`.

This weekly projection lineup is separate from the eight-player FantasyCalc starter/bench split. The former models weekly lineup scoring; the latter models roster value allocation.

`buildWeeklyTeamModel()` calculates weeks 1 through 17 and stores, for every week:

- Expected lineup points.
- Floor lineup points.
- Ceiling lineup points.
- Team standard deviation.
- Selected weekly lineup.

It also calculates:

- Mean projected week.
- Median projected week.
- Worst week.
- Best week.
- Average floor.
- Average ceiling.
- Season projected points.
- Week-to-week volatility.
- Rest-of-season mean projection after the current scoring week.
- Mean projection for the next three available weeks.

## 10. Power Ranking Pipeline

The ranking pipeline is:

```text
buildTeamModels(data)
  -> finalizePowerScores(models)
  -> applyHybridRanks(models)
  -> renderPowerRankings(models)
```

### Team model fields

`buildTeamModels()` creates one model per team containing:

- Team identity and owner.
- Record and completed-game PPG.
- Raw FantasyCalc roster value.
- Starter and bench entries.
- Position values and position scores.
- Starter values.
- Weekly projections.
- Floor and ceiling projections.
- Bye-aware weekly model inputs.

### Actual and projected strength

`finalizePowerScores()` blends projected strength with actual performance.

Projected strength is:

```text
50% median weekly projection
30% mean weekly projection
15% 75th-percentile weekly projection
5% 25th-percentile weekly projection
```

Actual strength is:

```text
70% actual PPG
30% actual PPG / projected PPG * league average projected points
```

The actual-performance blend weight depends on scoring week:

```text
Week:  0    1     2     3     4     5     6     7     8     9    10
Weight 0  .10   .20   .30   .40   .50   .55   .60   .65   .70   .75

Week: 11   12    13    14    15    16    17
Weight .78 .81   .84   .87   .90   .92   .94
```

If no team has completed games, actual weight is zero.

### Component scores

Each component uses a league-distribution percentile rather than dividing by the single best team. The lowest value maps near `0`, the median maps near `50`, and the highest value maps to `100`; tied values share the midpoint rank. This prevents one outlier team from defining every other team's score.

```text
componentScore = midpointPercentile(teamValue, leagueValues)
```

The best team is therefore `1.0`, displayed as `100/100`. A team at 4,000 when the leader is at 5,000 is scored according to its position in the league distribution rather than automatically receiving `80/100`.

The current component calculations are:

- `currentPowerScore`: normalized current-week legal lineup projection.
- `rosterAdvantageScore`: normalized weekly starter projection above positional replacement.
- `rosScore`: normalized rest-of-season projected lineup points.
- `actualPerformanceScore`: normalized actual strength.
- `qbSecurityScore`: normalized QB1/QB2 strength plus a smaller QB3 contribution.
- `depthScore`: 60% normalized projected depth plus 40% normalized bench value.
- `reliabilityScore`: normalized floor-to-expected stability and lower weekly volatility.
- `upsideScore`: normalized bench ceiling surplus.
- `recordScore`: normalized record win rate.

### Composite power formula

The current raw composite is:

```text
rawPower =
  0.30 * currentPowerScore
+ 0.20 * rosterAdvantageScore
+ 0.15 * rosScore
+ 0.12 * actualPerformanceScore
+ 0.08 * qbSecurityScore
+ 0.05 * depthScore
+ 0.05 * reliabilityScore
+ 0.03 * upsideScore
+ 0.02 * recordScore
```

The weights total 1.00.

The displayed overall score is normalized against the highest composite score in the league:

```text
totalScore = rawPower / max(rawPower) * 100
```

It is rounded to one decimal place. The best overall team is always `100.0` unless there is no usable model data.

The visible score breakdown uses the same component weights and shows each component's contribution as a percentage of that team's composite score.

Expanded team details also expose the raw starter and bench FantasyCalc totals, current lineup score, ROS projection, lineup advantage score, and QB security score so the composite can be audited at the team level.

## 11. Positional Rankings, Needs, and Depth

Position rankings compare each team's FantasyCalc position total against other teams.

`needsSurplusForTeam()` marks a position as:

- A need when its rank is in roughly the bottom 40% or roster depth is below the minimum.
- A surplus when its rank is in roughly the top 30% and the team has enough depth.

Minimum depth assumptions:

- QB: 2.
- RB: 3.
- WR: 3.
- TE: 1.

`computeDepthScore()` selects the five best projected players outside the normal weekly starters and sums their projections.

## 12. Hybrid Rankings

External ranking sources are local-only data entered by the user. They are stored under a league/season-specific key.

Each source contains:

- Source name.
- Team order.
- Team rank map.
- Weight.
- Enabled state.

`applyHybridRanks()` calculates a weighted ordinal average:

```text
averageRank = sum(sourceRank * sourceWeight) / sum(sourceWeight)
```

Lower average rank is better. Sources must contain at least half of the league's teams to be included. The site ranking can also be disabled or given a custom weight.

## 13. Trade Desk

The trade desk builds working roster maps from the same team models.

Trade selections are tracked in:

- `calcSelected.a`.
- `calcSelected.b`.
- `calcSelected.dropA`.
- `calcSelected.dropB`.

Trade fairness compares the FantasyCalc value sent by each side. The UI classifies the trade using the absolute gap and percentage gap.

Trades can be chained. `applySwap()` exchanges selected players and applies required drops when a team receives more players than it sends and would exceed `MAX_ROSTER` of 16.

Trade impact is recalculated by:

1. Cloning the team models.
2. Swapping the selected entries.
3. Recomputing position values and starter values.
4. Re-running rank maps.
5. Comparing ranks, starter projection, roster value, needs, and depth before and after.

The share link stores team IDs and selected player IDs in a `trade` query parameter. Chain state itself is session-local and is not encoded.

## 14. Trade Suggestions

`renderTradeSuggestions()` creates candidate packages from each partner team.

Candidate generation supports:

- 1-for-1 through multi-player package shapes.
- Position filters.
- Partner-team filters.
- Explicit players to get.
- Explicit players to give.
- Tradeable-player filtering.
- Value gap and ratio limits.
- Need-fill scoring.
- Lateral-trade penalties.
- Position downgrade rejection.
- Starter projection gain/loss.

Packages are rejected when they cause an excessive positional drop or fail both sides' minimum improvement tests. Remaining ideas are sorted by a lower-is-better score based on value gap, positional penalties, need fulfillment, and projection gains.

## 15. Playoff Simulation

`runPlayoffSimulations()` runs 10,000 simulations.

Inputs include:

- Completed record.
- Remaining schedule.
- Weekly lineup means.
- Weekly floor and ceiling.
- Weekly standard deviations.
- Playoff-team count.
- Playoff start week.
- Bye slots.

The current week is excluded from projected future schedule games. If no usable schedule remains, a synthetic round-robin schedule is created.

Each simulated future week:

1. Draws an independent score for every team.
2. Applies head-to-head results.
3. Applies a median-style result where the top half of scorers receive a win and the bottom half receive a loss.
4. Sorts teams by wins, points, and head-to-head wins.
5. Counts playoff and bye appearances.
6. Runs single-elimination playoff rounds using the relevant playoff-week projections.

Reported probabilities include:

- Playoffs.
- First-round bye.
- Championship.

Very early-season probabilities are curved toward priors to avoid overconfident results with little actual data.

Strength of schedule is the average projected points of remaining head-to-head opponents. Positive SOS delta means harder than league average.

## 16. Matchups and Live Views

`setupMatchup()` loads a selected team's live matchup and opponent.

It calculates:

- Current starter points.
- Remaining starter projection.
- Standard and optimal lineup totals.
- Projection margin.
- Win probability from independent normal score draws.
- Floor/ceiling estimates.
- Late-swap recommendations.

Late-swap advice compares unplayed starters with unplayed bench players and chooses floor, ceiling, or ordinary projection depending on whether the team is ahead or behind.

`renderSnapshotDeck()` creates screenshot-oriented summaries for:

- Power rankings.
- Live matchup scores.
- Top and bottom teams versus the current median projection.

## 17. Rendering and Refresh Flow

`renderAll()` performs the main post-load rendering:

1. Trade board.
2. External ranking sources.
3. Power models and rankings.
4. Snapshot deck.
5. Trade calculator.
6. Power filters.

Tabs are event-driven. Snapshot and playoff tabs establish one-minute refresh timers while active. Matchup views also refresh while open.

The header status shows team count, on-block count, scoring week, and active value source.

## 18. Local Storage

Keys currently used include:

- `leagueToolsTheme`: dark/light theme.
- `leagueToolsLeagues`: recent leagues.
- `leagueToolsActive`: active league.
- `sleeperPlayersNfl_v1`: compact Sleeper player cache.
- `sleeperPlayersNfl_v1_ts`: Sleeper cache timestamp.
- `leagueToolsPowerRanks_{platform}_{league}_{season}`: previous ranking order for movement indicators.
- `leagueToolsExtRanks_{league}_{season}`: external ranking sources.
- `leagueToolsExtRanks_{league}_{season}_siteW`: site ranking weight.

## 19. Important Fallbacks and Limitations

- ESPN public league access is required for ESPN data.
- API availability and browser CORS behavior can affect loading.
- Bye-week data is optional.
- ESPN weekly projection enrichment is optional.
- Sleeper projections are synthetic proxies based on position and FantasyCalc rank.
- Historical trade grades use today's FantasyCalc values.
- Live projections are estimates when ESPN does not expose a locked player total.
- The weekly lineup model and the FantasyCalc eight-player value split intentionally use different roster rules.
- There are no automated unit tests in the repository; the primary low-cost validation is parsing the inline script with `node --check`.
