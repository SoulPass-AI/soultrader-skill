# /soulhunter evolve — Weekly Strategy Evolution

## Two Modes of Evolution

**Real-time evolution (anytime a trade closes or a mistake happens):**
- Immediately ask: what went wrong? What's the root cause?
- Update the skill rules RIGHT NOW — don't wait for Sunday
- Save to memory so future sessions inherit the lesson
- This is how mistakes lead to rule changes within hours, not weeks

**Scheduled deep review (this document — weekly, Sunday):**
Run every Sunday after `/soulhunter research`, or when the user says `/soulhunter evolve`.
This is what separates SoulHunter from a static bot — it learns from its own performance, adapts to market regimes, and becomes antifragile over time.

**The principle:** Discipline means principles don't change (follow SM, use stops, respect Layer 1). But RULES are just implementations of principles — they should evolve the moment you learn they're wrong. Waiting a week to fix a broken rule is not discipline, it's stubbornness.

## Pre-flight

1. Load `~/.soulhunter/portfolio.json` — need closed trades from this week
2. Load `~/.soulhunter/rule-history.json` — cumulative rule effectiveness
3. Load `~/.soulhunter/oracle-wallets.json` — oracle performance tracking
4. Load `~/.soulhunter/funding-tracker.json` — news dedup + yield tracking

If fewer than 3 closed trades this week, still run evolution but note "insufficient data for statistical significance" in the report.

## Design Philosophy

Evolution is not just "adjust weights." It's a multi-layered self-improvement loop:

1. **Review** — what happened, honestly
2. **Attribution** — which rules caused which outcomes (including thesis quality and news impact)
3. **Diagnose** — are you making systematic errors?
4. **Calibrate** — adjust parameters with guardrails against overfitting
5. **Mutate** — occasionally try something new to discover better strategies
6. **Report** — transparent record for accountability (including Layer 1 accuracy)

## Phase 0: Missed Opportunity Tracking

Before reviewing what happened, check what you DIDN'T do. This is where the best traders learn the most.

### 0a. Tokens that were screened but not traded

Load the previous strategy.json and compare:
- Tokens in `flow_acceleration_table` that didn't make the `watchlist` — why were they excluded?
- Tokens in `watchlist` that passed conviction but were blocked by entry rules (price change, pullback) — what happened to them?
- Tokens in `blacklist` — did they crash (validating the blacklist) or moon (missed opportunity)?

```bash
# For each screened-but-not-traded token, check what happened:
soulpass price <TOKEN>
# Compare current price vs price at strategy generation
```

### 0b. Record missed opportunities

```json
{
  "missed_opportunities": [
    {
      "token": "MPLX",
      "reason_excluded": "blacklisted — single Fund trader risk",
      "price_at_screening": 0.0365,
      "price_now": 0.0520,
      "would_have_pnl_percent": 42.5,
      "lesson": "single-Fund-trader blacklist may be too conservative"
    }
  ],
  "validated_blacklist": [
    {
      "token": "JUP",
      "reason_excluded": "DECELERATING flow",
      "price_at_screening": 0.154,
      "price_now": 0.142,
      "avoided_loss_percent": -7.8,
      "lesson": "DECELERATING blacklist correctly avoided a loser"
    }
  ]
}
```

This data feeds into Phase 3 — if the blacklist is consistently blocking winners, loosen it. If it's consistently saving you from losers, keep it tight.

### 0c. Edge benchmark — what would pure beta have returned?

Calculate: "If we had bought every token with positive SM netflow equally weighted, what would the return be?"

This benchmark tells you whether SoulHunter's alpha layer (flow acceleration, oracles, conviction scoring) is actually adding value above simple SM-following. If pure beta beats us for 3+ weeks, something is wrong with the alpha layer.

```
pure_beta_return = average price change of all SM netflow positive tokens this week
soulhunter_return = our actual portfolio return
alpha_generated = soulhunter_return - pure_beta_return
```

Track `alpha_generated` over time. It should be consistently positive. If it goes negative, the conviction scoring is subtracting value, not adding it.

## Phase 1: Review — What happened this week?

### 1a. Load trade data
Read `~/.soulhunter/portfolio.json` and extract:
- All trades closed this week (from `closed_trades` where exit_time is within this week)
- All currently open positions
- Daily PnL history
- Price history for volatility analysis

### 1b. Generate performance summary

Calculate:
```
total_trades:           count of closed trades this week
wins:                   trades with pnl_percent > 0
losses:                 trades with pnl_percent < 0
timeout_exits:          trades closed due to max_hold_days
trailing_stop_exits:    trades closed by trailing stop
atr_stop_exits:         trades closed by ATR stop
win_rate:               wins / total_trades
total_pnl_usd:          sum of all realized pnl
total_pnl_percent:      total_pnl_usd / starting_capital_this_week
max_drawdown_percent:   worst peak-to-trough from daily_pnl history
best_trade:             highest pnl_percent and which token
worst_trade:            lowest pnl_percent and which token
avg_hold_days:          average holding period
avg_win_size:           average pnl_percent of winning trades
avg_loss_size:          average pnl_percent of losing trades
profit_factor:          total_wins_usd / total_losses_usd (> 1.0 is good)
expectancy:             (win_rate × avg_win) - (loss_rate × avg_loss)
```

### 1c. Regime performance breakdown

Separate this week's trades by the regime they were entered in:
```
regime_risk_on:    { trades, win_rate, avg_pnl, profit_factor }
regime_ranging:    { trades, win_rate, avg_pnl, profit_factor }
regime_risk_off:   { trades, win_rate, avg_pnl, profit_factor }
```

This data feeds into regime-specific weight optimization in Phase 3.

### 1d. Compare with SM actual performance (~10 credits, optional)

If budget allows, check how the SM wallets you followed actually performed:
```bash
nansen research token pnl --chain solana --token <token_mint> --pretty
```

For the top 2-3 tokens you traded, check:
- Did SM wallets that bought also profit?
- Did you enter at a similar or better price than SM average?
- Did SM sell before you? (potential timing signal for next week)
- How much lag was there between SM entry and your entry?

This step is optional and costs credits. Skip it if budget is tight or if the week had few trades.

## Phase 2: Attribution — Which rules worked?

### 2a. Rule-by-rule analysis

Each closed trade in portfolio.json has a `triggered_rules` array and a `conviction_score`. For each unique rule:

```
For rule R:
  times_triggered:      how many trades had R in their triggered_rules
  win_rate:             what % of those trades were profitable
  avg_pnl_percent:      average PnL of trades that used this rule
  expected_value:       win_rate × avg_win - (1 - win_rate) × avg_loss
  contribution_usd:     sum of pnl_usd for trades that used this rule
```

### 2b. Exit strategy analysis

Separately analyze how each exit type performed:
```
For exit_type E (atr_stop, trailing_stop, take_profit, time_limit, sell_watchlist, regime_shift):
  times_triggered:      count
  avg_pnl_at_exit:      average PnL when this exit triggered
  missed_upside:        for stop/time exits — what was the price 3 days later? Did we sell too early?
  saved_downside:       for stop exits — what was the price 3 days later? Did the stop prevent worse loss?
```

This is crucial: if trailing stops are consistently selling at -6% from peak but the price recovers 80% of the time → the trail is too tight. If ATR stops trigger and the price drops further 70% of the time → the stop is well-calibrated.

### 2c. Update rule history with exponential decay

Load `~/.soulhunter/rule-history.json`. This file persists across weeks. Structure:

```json
{
  "updated_at": "2026-03-30",
  "decay_half_life_weeks": 4,
  "total_weeks_active": 6,
  "rules": {
    "flow_accelerating": {
      "weighted_triggers": 8.2,
      "weighted_wins": 6.1,
      "weighted_losses": 2.1,
      "win_rate_weighted": 0.744,
      "avg_pnl_weighted": 14.3,
      "expected_value_weighted": 11.8,
      "current_weight": 0.25,
      "trend": "improving",
      "regime_breakdown": {
        "risk_on": { "weighted_triggers": 5.0, "win_rate": 0.82, "avg_pnl": 18.1 },
        "ranging": { "weighted_triggers": 2.5, "win_rate": 0.68, "avg_pnl": 9.2 },
        "risk_off": { "weighted_triggers": 0.7, "win_rate": 0.50, "avg_pnl": 1.1 }
      }
    },
    "flow_fresh_strong": { "...same structure..." },
    "fund_flow_positive": { "...same structure..." },
    "holdings_value_high": { "...same structure..." },
    "consensus_bonus": { "...same structure..." },
    "oracle_buying": { "...same structure..." },
    "oracle_new_position": { "...same structure..." },
    "fundamentals_solid": { "...same structure..." },
    "accumulation_phase": { "...same structure..." },
    "screener_alignment": { "...same structure..." }
  },
  "exit_types": {
    "trailing_stop": {
      "triggers": 12,
      "avg_pnl": 14.2,
      "missed_upside_rate": 0.25,
      "saved_downside_rate": 0.75
    }
  },
  "conviction_calibration": {
    "high": { "predicted_win_rate": 0.70, "actual_win_rate": 0.65, "n": 20 },
    "medium": { "predicted_win_rate": 0.55, "actual_win_rate": 0.52, "n": 35 },
    "low": { "predicted_win_rate": 0.45, "actual_win_rate": 0.41, "n": 15 }
  }
}
```

**Exponential decay**: when adding this week's data, apply decay to all historical data first:
```
decay_factor = 0.5 ^ (1 / half_life_weeks)  # with half_life = 4 weeks

For each historical metric:
  weighted_value = old_weighted_value × decay_factor + this_week_value
```

This means data from 4 weeks ago has half the weight of this week's data. Recent performance matters more than ancient history — markets change.

### 2d. Identify patterns

Look for:
- **Strong rules**: weighted_win_rate > 65% AND expected_value > 5% with ≥15 weighted triggers → candidate for weight increase
- **Weak rules**: weighted_win_rate < 45% OR expected_value < 0 with ≥15 weighted triggers → candidate for weight decrease
- **Insufficient data**: < 15 weighted triggers → keep current weight, mark for observation
- **Regime-dependent rules**: rule performs well in one regime but poorly in another → candidate for regime-specific weighting
- **Synergy patterns**: are there combinations of rules that consistently win? (e.g., "dca_accumulation + sm_netflow_top5 + cex_accumulation" together = 85% win rate)
- **Anti-patterns**: combinations that consistently lose despite individual rules looking OK

### 2e. Thesis Quality Attribution

Thesis tracking is what separates SoulHunter from a dumb signal follower. Analyze how well theses performed:

**For each closed trade this week:**
```
thesis_status_at_exit: alive | weakening | dead
exit_reason: was it thesis_invalidation or something else?
thesis_accuracy: did the invalidation conditions correctly predict the outcome?
```

**Thesis quality metrics:**
- `thesis_invalidation_saves`: trades where thesis died early AND price continued to fall after exit -> the thesis saved us
- `thesis_invalidation_misses`: trades where thesis died but price recovered -> thesis was too sensitive
- `thesis_alive_wins`: trades that exited profitably with thesis still alive -> normal good trade
- `thesis_alive_losses`: trades that hit stop-loss with thesis alive -> thesis didn't capture the risk

**Key insight:** If `thesis_invalidation_saves` > `thesis_invalidation_misses`, the thesis system is adding value. Track this ratio over time.

**Improvement signals:**
- High `thesis_alive_losses` -> invalidation conditions are too lax, add more conditions
- High `thesis_invalidation_misses` -> conditions are too sensitive, remove or widen triggers
- Most exits are thesis_invalidation -> theses might be too fragile, review condition quality

### 2f. Layer 1 Anchor Accuracy Review

Load `~/.soulhunter/layer1-anchor.json` and compare its regime call against actual outcomes:

1. **Regime accuracy**: Was the Layer 1 classification correct for this week?
   - Did Risk-Off regime coincide with actual market decline?
   - Did Risk-On regime coincide with positive performance?
   - Calculate: `regime_correct_weeks / total_weeks`

2. **Pillar accuracy**: Which pillars were most predictive?
   - Trend pillar: did BTC/SOL price direction match the pillar call?
   - SM Consensus pillar: did oracle wallet direction predict token performance?
   - Ecosystem health pillar: did TVL/volume trends correlate with returns?

3. **Regime change trigger accuracy**: Did any triggers fire? Were they correct?
   - Trigger fired + regime actually changed = good trigger
   - Trigger fired + false alarm = trigger too sensitive
   - Regime changed + no trigger fired = missing trigger

**Adjustment:** If Layer 1 accuracy < 60% over 4 weeks, simplify the regime classification (fewer pillars, wider thresholds). If > 80%, consider adding granularity.

### 2g. News Impact Attribution

If BlockBeats news scanning was active this week:

1. **News-driven exits**: How many positions were closed due to news?
   - Was the exit correct? (price continued to fall after news-driven exit)
   - Was it premature? (price recovered within 48h)

2. **News-driven thesis deaths**: Did news correctly identify thesis invalidation?

3. **Missed news**: Were there significant events that the monitor DIDN'T catch?

Track: `news_value_added = news_saves - news_false_alarms`. If consistently negative, reduce news sensitivity. If consistently positive, this is a valuable edge.

### 2h. Signal D (DCA Intelligence) Attribution

For trades where Signal D contributed:
- Did SM DCA signals predict sustained price increases?
- Did multiple DCA orders on same token correlate with higher win rate?
- Track: DCA-signal trades vs non-DCA trades performance comparison

## Phase 3: Calibrate — Adjust parameters

### 3a. Update rule weights

Based on Phase 2 analysis, adjust conviction scoring weights.

**Constraints (prevent overfitting):**
- Single-week weight change: ±0.03 max (tightened from ±0.05 — smaller steps are safer)
- Minimum weight for any active rule: 0.03
- Maximum weight for any single rule: 0.30
- All weights must sum to ~1.0 (normalize after adjustment)
- Rules with < 15 weighted triggers: don't adjust weight
- Penalty weights (negative signals) can change by ±0.02 max

**Process:**
1. Start with current weights from rule-history.json
2. For each rule with ≥15 weighted triggers:
   - expected_value > 10% → increase weight by 0.03
   - expected_value 5-10% → increase weight by 0.015
   - expected_value 0-5% → no change
   - expected_value -5% to 0 → decrease weight by 0.015
   - expected_value < -5% → decrease weight by 0.03
3. **Regime modulation**: if a rule's win_rate differs by >15% between regimes, consider making the weight regime-dependent (store separate weights per regime in rule-history.json)
4. Normalize all weights to sum to 1.0
5. Save to rule-history.json

### 3b. Calibrate exit parameters

**ATR multiplier adjustment:**

| Evidence | Adjustment |
|----------|------------|
| ATR stops: saved_downside_rate > 70% | Good calibration, no change |
| ATR stops: saved_downside_rate < 50% | Stops too tight, widen by 0.2× ATR |
| ATR stops: saved_downside_rate > 85% | Could be too wide (missing recoveries), tighten by 0.1× ATR |

**Trailing stop adjustment:**

| Evidence | Adjustment |
|----------|------------|
| Trailing stops: missed_upside_rate > 40% | Trail too tight, widen by 1% |
| Trailing stops: missed_upside_rate < 20% | Trail working well, no change |
| Trailing stops: missed_upside_rate > 60% | Trail much too tight, widen by 2% |

**Trailing stop bounds**: never tighter than -4% from peak, never wider than -12% from peak.

**Take-profit vs trailing stop analysis:**
- If most take-profit exits show continued upside afterward → trailing stop is better, consider lowering the fixed take-profit to a higher activation threshold
- If most trailing-stop exits show continued downside → trailing stop is well-timed

### 3c. Adjust risk parameters

Risk is defined by stop-loss, not position size. Adjust the max-loss-at-stop percentages based on performance:

| This week's max drawdown | Adjustment |
|--------------------------|------------|
| < -3% (very safe) | Consider increasing max_loss_at_stop by 1% for high conviction (never exceed 8%) |
| -3% to -6% (normal) | No change |
| -6% to -8% (close to limit) | Tighten: max_loss_at_stop -= 1%, ATR multipliers += 0.2 |
| Hit -8% daily breaker | Tighten: max_loss_at_stop -= 2%, ATR multipliers += 0.3, reduce max_open_positions by 1 |
| Hit -12% weekly breaker | Emergency: max_loss_at_stop = 3% for all conviction levels, max_open_positions = 2, skip next week's new entries entirely |

**Hard ceilings that evolution can never breach:**
- max_single_trade_loss: 8% of capital
- max_total_portfolio_loss: 20% of capital
- daily_loss_limit: 10%
- weekly_loss_limit: 15%
- ATR multiplier: never below 1.2x (stops too tight = noise kills you)
- ATR multiplier: never above 3.5x (stops too wide = defeats the purpose)

### 3d. Signal-Level Weight Evolution

Beyond individual rule weights, evolve the allocation between Signal A / B / C:

**Track per-signal performance:**
```
For each trade, record which signals contributed:
  signal_a_contributed: true/false (was flow_acceleration or fund_flow a reason?)
  signal_b_contributed: true/false (was oracle_buying a reason?)
  signal_c_contributed: true/false (was fundamentals or accumulation a reason?)
  signal_d_contributed: true/false (was DCA intelligence or ecosystem data a reason?)
```

**Calculate per-signal win rates and expected value:**
```
signal_a_only_trades: trades where A contributed but not B/D
signal_b_only_trades: trades where B contributed but not A
signal_d_only_trades: trades where D contributed but not A/B
signal_a_and_b_trades: trades where both A and B contributed
signal_a_and_d_trades: trades where both A and D contributed (DCA + flow acceleration = strongest combo?)
```

**Adjustment logic (every 4 weeks, with >= 10 trades):**
- If Signal A trades outperform Signal B by > 10% in expected value -> shift 5% weight from B to A
- If Signal B begins outperforming (oracle wallets mature) -> shift 5% weight from A to B
- If Signal E (DCA+Yield) consistently identifies winners -> shift up to 5% from C to E
- Signal C should stay at 10-25% — it's confirmation + indicators, not a standalone driver
- Signal D (Trend gate) stays at 10% — it's a go/no-go filter, not scored for weight shifting
- Cold start: A starts at 50%, shifts toward 35% as B and E mature

**Bounds:**
- Signal A: 25%-55%
- Signal B: 10%-35%
- Signal C: 10%-25%
- Signal D: 10% (fixed — trend gate)
- Signal E: 10%-25%
- Always sums to 100%

This ensures the system naturally shifts from SM Consensus-driven (early weeks) to Oracle+DCA-driven (mature weeks) based on actual performance data, not assumptions.

### 3e. Oracle Wallet Validation (CRITICAL — this is the Alpha engine's maintenance)

Load `~/.soulhunter/oracle-wallets.json`.

**For each oracle wallet, check this week's signal accuracy:**

1. Last week's research identified tokens that oracle wallets were buying
2. Did those tokens actually go up this week?
3. Calculate per-wallet: `signal_accuracy = profitable_calls / total_calls`

**Status transitions:**
```
new_candidate (0.25 weight)
  → 2+ appearances + accuracy data → strong_candidate (0.50)
  → 3+ appearances + accuracy > 50% → verified_oracle (0.75)
  → 4+ appearances + accuracy > 60% → elite_oracle (1.0)

Demotion:
  → accuracy dropped below 40% for 2 consecutive weeks → demote one level
  → 3 weeks with no activity → remove from list
  → wallet started appearing on sell side → immediate removal
```

**Discover new oracle candidates** — this happens automatically during /research Step 1, but evolve validates the existing list and cleans up stale entries.

**Key metric to track**: `oracle_hit_rate` = percentage of oracle-sourced trades that were profitable. This is the single most important number for SoulHunter. If it drops below 55% for 3+ weeks, the oracle discovery process itself needs debugging — maybe the "who bought early" signal is being gamed, or the market regime changed.

### 3f. Position sizing evolution — Kelly-inspired

Fixed position sizing leaves money on the table when edge is strong and risks too much when edge is weak. Use a simplified Kelly approach to adapt position sizes.

**Calculate current edge per conviction level:**
```
For conviction level L:
  edge = (win_rate × avg_win_percent) - (loss_rate × avg_loss_percent)
  kelly_fraction = edge / avg_win_percent    (simplified Kelly)
  suggested_size = kelly_fraction × 0.5      (half-Kelly for safety)
```

**Adjustment rules:**
- If `suggested_size` > current `position.size_percent` by > 2% → increase by 1% (gradual)
- If `suggested_size` < current `position.size_percent` by > 2% → decrease by 1%
- Never exceed hard ceiling (12% max per position)
- Never go below 2% (minimum meaningful position)
- Require ≥ 8 trades at that conviction level before adjusting

**Example:**
```
very_high conviction: win_rate 75%, avg_win 28%, avg_loss 12%
  edge = (0.75 × 28) - (0.25 × 12) = 21 - 3 = 18%
  kelly = 18 / 28 = 0.64 → half-kelly = 0.32 (32%)
  capped at 12% → suggested: 12%
  current: 8% → increase to 9% (gradual step)

medium conviction: win_rate 55%, avg_win 18%, avg_loss 14%
  edge = (0.55 × 18) - (0.45 × 14) = 9.9 - 6.3 = 3.6%
  kelly = 3.6 / 18 = 0.20 → half-kelly = 0.10 (10%)
  current: 4% → no change (within 2% band)
```

This makes the agent automatically size up when winning and size down when losing — without manual intervention.

**Critical: losing weeks auto-shrink.** If weekly PnL < -3%, force all position sizes down by 1% for next week regardless of Kelly calculation. This prevents spiral drawdowns. Restore sizes only after a positive week.

### 3g. Entry threshold adjustment

The `entry_threshold` determines the minimum conviction score to consider a trade.

| This week's outcome | Adjustment |
|---------------------|------------|
| win_rate > 65% AND > 5 trades AND expectancy > 5% | Lower threshold by 0.03 (find more opportunities) |
| win_rate 50-65% | No change |
| win_rate < 50% AND > 5 trades | Raise threshold by 0.03 (be more selective) |
| < 5 trades total | Don't adjust (insufficient data) |
| Expectancy < 0 (losing money per trade on average) | Raise threshold by 0.05 |

**Bounds: entry_threshold must stay within 0.45 - 0.80**

### 3g. Conviction calibration

Compare predicted vs actual win rates for each conviction level:

```
For conviction level L (high/medium/low):
  calibration_error = predicted_win_rate - actual_win_rate
  if |calibration_error| > 0.10 AND n ≥ 10:
    adjust conviction thresholds to improve calibration
```

If high conviction trades are winning at only 55% (predicted 70%), the conviction scoring is overconfident. Response:
- Raise the score threshold for "high" from 0.70 → 0.75
- Or: reduce position size for "high" conviction to match actual win rate

If low conviction trades are winning at 55% (predicted 45%), the scoring is underconfident:
- Lower the "low" threshold from 0.35 → 0.30 to capture more of these

### 3i. Learn when NOT to trade

The most profitable adaptation is often knowing when to sit out. Track conditions where trading consistently loses:

**Build a "no-trade" pattern library:**
```
For each losing trade, record the conditions at entry:
  regime, conviction_level, signal_types, market_volatility (BTC 7d range),
  days_since_research, number_of_concurrent_positions
```

After 4+ weeks, look for patterns:
- "Trades entered when BTC 7d volatility > 20% have 30% win rate" → add as no-trade filter
- "Trades entered > 5 days after research have 35% win rate" → add signal freshness decay
- "4th and 5th concurrent positions have 25% win rate" → reduce max_positions from 5 to 3
- "Medium conviction trades in Ranging regime have negative expectancy" → raise threshold for Ranging

**Cash is a position.** If the no-trade patterns suggest the current environment is unfavorable, the evolved strategy should recommend: "Conditions unfavorable. Staying in cash until [specific condition changes]." This is not failure — it's discipline.

Save no-trade patterns to rule-history.json under `no_trade_filters`.

## Phase 4: Mutate — Controlled Experimentation

This is what makes SoulHunter truly adaptive. Every 3 weeks, introduce one small mutation to test a hypothesis.

### 4a. Generate mutation candidates

Based on the data from Phases 1-3, identify one area where a controlled experiment could improve the strategy. Examples:

- "ACCELERATING tokens (7d/30d ratio > 4) have 80% win rate but we're using ratio > 4 as threshold — what if ratio > 8 gets an extra conviction bonus?"
- "Fund flow signal has 25% higher win rate than general SM consensus — what if fund_flow_positive weight goes from 0.20 to 0.30?"
- "Tokens entered within 48h of research have better returns — what if we add a 'signal freshness' bonus that decays each day?"
- "trader_count ≥ 5 tokens have never lost — what if we auto-upgrade conviction for consensus > 5?"
- "Oracle wallets discovered via profiler pnl-summary with avg_roi > 1.0 outperform those with avg_roi 0.3-1.0 — raise the oracle discovery threshold?"

### 4b. Design the experiment

For the chosen mutation:
- **Hypothesis**: clear statement of what you expect to happen
- **Mechanism**: exactly what parameter or rule changes
- **Scope**: apply to max 20% of trades (1-2 positions per week) — don't risk the whole portfolio on an experiment
- **Duration**: 3 weeks minimum for meaningful data
- **Success metric**: specific, measurable (e.g., "mutation trades have >60% win rate and >8% expectancy")
- **Kill criteria**: if mutation trades lose >15% aggregate in any single week → abort

### 4c. Track mutation separately

In rule-history.json, tag mutation trades:
```json
{
  "active_mutation": {
    "id": "mutation_003",
    "hypothesis": "Fund-only signals with doubled weight improve win rate",
    "started_week": "2026-W14",
    "expires_week": "2026-W17",
    "trades": [],
    "current_pnl_percent": 5.2,
    "status": "active"
  },
  "mutation_history": [
    {
      "id": "mutation_001",
      "hypothesis": "...",
      "result": "adopted",
      "impact": "+3.2% win rate improvement"
    }
  ]
}
```

### 4d. Evaluate and decide

After 3 weeks:
- **Adopt**: mutation trades outperformed non-mutation trades → incorporate into main strategy
- **Reject**: mutation trades underperformed → revert and record why
- **Extend**: inconclusive (too few trades) → run 2 more weeks

## Phase 5: Self-Diagnosis — Am I making systematic errors?

This is the meta-evolution layer. Every 4 weeks, run a deeper self-assessment.

### 5a. Behavioral audit

Check for common trading agent failure modes:

| Pattern | Detection | Response |
|---------|-----------|----------|
| **Overtrading** | >8 trades/week average over 4 weeks | Raise entry threshold by 0.05 |
| **Undertrading** | <2 trades/week average over 4 weeks | Lower entry threshold by 0.03 (but verify this isn't good discipline) |
| **Chasing recovery** | Position sizes increasing after losing weeks | Force position sizes back to base; add "no size increase after loss week" rule |
| **Regime blindness** | Similar loss rate in Risk-On and Risk-Off | Tighten Risk-Off filters (they should have fewer but better trades) |
| **Signal decay** | win_rate trending down over 4 weeks | Full research reset — clear watchlist and rebuild from scratch |
| **Concentration** | >50% of PnL from one token | Diversification is working poorly — reduce max position, increase min tokens |
| **Early exit** | >40% of exits are trailing stops that "missed" >20% further upside | Trailing stops too tight — widen across the board |
| **Late exit** | >40% of exits are time limits with losses | Entry signals may be wrong, not just timing — focus evolution on entry rules |

### 5b. Regime transition analysis

The most dangerous moment for any strategy is a regime shift. Track how the system performs during transitions:

```
For each regime change detected this month:
  regime_from: "Risk-On"
  regime_to: "Risk-Off"
  detection_delay_days: 2    (how long before the system recognized the shift)
  pnl_during_transition: -3.2%   (losses between shift start and detection)
  response_quality: "adequate" | "too_slow" | "false_alarm"
```

**Adaptation targets:**
- Detection delay should be ≤ 2 days (use the BTC/SOL price check in execute.md)
- Transition losses should be ≤ half of the weekly loss limit
- False alarm rate (detected shift that reversed within 48h) should be < 30%

If detection is too slow: consider tightening the regime override thresholds in execute.md (e.g., BTC -8% → -6%).
If too many false alarms: loosen the thresholds (e.g., BTC -8% → -10%).

This is where most automated strategies die — they adapt to yesterday's regime while the market has already moved on.

### 5c. Comparative performance

Calculate rolling metrics:
```
sharpe_4w:          (avg_weekly_return - risk_free) / std(weekly_returns) over last 4 weeks
sortino_4w:         (avg_weekly_return - risk_free) / std(negative_weekly_returns)
max_drawdown_4w:    worst peak-to-trough over 4 weeks
calmar_ratio:       annualized_return / max_drawdown
```

Track these over time. If Sharpe drops below 0.5 for 3 consecutive weeks → trigger a full strategy review (more aggressive research next cycle, consider pausing trading for one week).

### 5d. Generate diagnosis report

Append to the weekly evolution report:
```markdown
## Self-Diagnosis (Week N, Monthly Check)
- Overtrading risk: LOW (4.2 trades/week avg)
- Signal decay: NONE (win rate stable at 62%)
- Behavioral bias detected: MILD concentration risk (JTO = 38% of PnL)
- Sharpe (4w): 1.24 | Sortino (4w): 1.85
- Action items: Reduce JTO max position to 4%, diversify entry signals
```

## Phase 6: Generate Evolution Report

Write a markdown report to `~/.soulhunter/reports/week-YYYY-WW.md`:

```markdown
# SoulHunter Evolution Report — Week [N]

## Market Regime
- Classification: [Risk-On / Ranging / Risk-Off / BTC-Divergent]
- BTC: [+/-X]% (7d) | SOL: [+/-X]% (7d)
- Regime overrides triggered: [count]

## Performance
- Trades: [N] total | [W] wins | [L] losses | [T] timeouts
- Win rate: [X]% (weighted: [X]%)
- Profit factor: [X]
- Expectancy: [+/-X]% per trade
- Total PnL: [+/-$X] ([+/-X]%)
- Max drawdown: [-X]%
- Best: [TOKEN] [+X]% | Worst: [TOKEN] [-X]%

## Exit Analysis
- ATR stops: [N] (saved downside [X]% of the time)
- Trailing stops: [N] (missed upside [X]% of the time)
- Take profits: [N]
- Time limits: [N]
- Sell watchlist alerts: [N]
- **Thesis invalidation: [N] (saved [X]% further downside on average)**

## Thesis Quality
- Thesis invalidation saves (correct early exits): [N]
- Thesis invalidation misses (premature exits): [N]
- Thesis save ratio: [X]% (target > 60%)
- Thesis alive losses (stop hit, thesis intact): [N] -> review invalidation conditions

## Layer 1 Anchor Accuracy
- Regime call this week: [classification] (confidence [X])
- Was it correct? [yes/no — evidence]
- Pillar accuracy: Trend [X]%, SM Consensus [X]%, Ecosystem [X]%
- Regime change triggers fired: [N], correct: [N]

## News Impact (BlockBeats)
- Headlines scanned: [N]
- News-driven actions: [N]
- News saves (correct action): [N]
- News false alarms: [N]
- Net news value: [positive/negative/neutral]

## What Worked
- [Rule with highest positive contribution and why]
- [Specific trade example with entry → peak → exit path]

## What Didn't Work
- [Rule with worst performance and analysis]
- [Specific trade example and what went wrong]

## Learnings
- [Key insight — e.g., "Fund wallet signals outperform Smart Trader signals by 18% in Risk-On regime"]
- [Pattern — e.g., "CEX accumulation score > 0.5 had 78% win rate across 12 trades"]

## Strategy Adjustments
- [Weight changes with before → after and reasoning]
- [Exit parameter changes with evidence]
- [Risk parameter changes with evidence]
- [Entry threshold change]

## Active Mutation
- [Current experiment status, if any]
- [Mutation trade performance vs baseline]

## Self-Diagnosis (every 4 weeks)
- [Behavioral audit results]
- [Rolling Sharpe/Sortino]
- [Action items]

## Missed Opportunity Review
- Tokens screened but not traded: [N]
- Missed winners (would have been profitable): [list with % gain]
- Validated blacklist (avoided losers): [list with % loss avoided]
- Lesson: [what to adjust]

## Edge Benchmark
- Pure beta return (equal-weight SM netflow): [X]%
- SoulHunter return: [X]%
- Alpha generated: [+/-X]% (should be positive)
- Alpha trend (4-week): [improving/stable/declining]

## Position Sizing Evolution
- Kelly-suggested sizes vs current sizes per conviction level
- Adjustment made: [if any]

## Regime Transition (if applicable)
- Transitions detected: [N]
- Average detection delay: [N] days
- PnL during transitions: [X]%

## Cumulative Performance (since inception)
- Total return: [X]%
- Max drawdown: [-X]%
- Overall win rate: [X]%
- Profit factor: [X]
- Weeks active: [N]
- Sharpe estimate (annualized): [X]
- Total trades: [N]
- Current regime track record: [regime → win_rate for each]
- Cumulative alpha vs pure beta: [X]%
```

## Phase 7: Print summary and diary entry

Show the evolution report to the user.

Write a diary entry summarizing the evolution:
```bash
soulpass diary write --content '{
  "type": "evolution_report",
  "week": "2026-W13",
  "layer1_regime": "Ranging",
  "layer1_accuracy": 0.75,
  "regime": "risk_on",
  "pnl_percent": 6.84,
  "win_rate": 0.625,
  "profit_factor": 1.82,
  "expectancy": 4.3,
  "sharpe_4w": 1.24,
  "thesis_save_ratio": 0.72,
  "thesis_invalidation_exits": 2,
  "news_value_added": 1,
  "key_learning": "ACCELERATING signals (flow ratio > 4) hit 78% win rate; thesis invalidation saved 2 positions from further -12% decline",
  "strategy_changes": [
    "weight flow_accelerating: 0.22 -> 0.25",
    "weight sm_dca_active: 0.08 -> 0.10 (DCA signals outperformed)",
    "trailing_stop widened: -6% -> -7% (missed_upside_rate was 45%)"
  ],
  "active_mutation": "mutation_003: fund-only doubled weight (week 2/3)",
  "cumulative_return_percent": 18.3,
  "self_diagnosis": "healthy, mild concentration risk flagged"
}'
```

Then tell the user: "Evolution complete. Run `/soulhunter research` now to generate next week's strategy with updated parameters."

## First-Time Evolution

If this is the first week (no rule-history.json exists), the evolution is simpler:
1. Review trades and calculate performance
2. Create rule-history.json with default weights and this week's data
3. Create initial conviction_calibration baselines
4. Generate the report
5. Note: "Insufficient data for meaningful weight adjustment. Using defaults for another week. Real evolution begins week 3 when we have enough weighted triggers."

No mutations until week 4 (need baseline data first).

## Evolution Schedule Summary

| Week | Actions |
|------|---------|
| 1 | Baseline only — log data, use defaults |
| 2 | Begin exponential decay tracking, first weight review (probably no changes yet) |
| 3 | First possible weight adjustments (if any rule hits 15 weighted triggers) |
| 4 | First mutation experiment begins + first self-diagnosis |
| 7 | Mutation evaluation + second self-diagnosis |
| 8+ | Full cycle: evolve → mutate → diagnose every 4 weeks |
