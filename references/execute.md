# /soulhunter execute & /soulhunter monitor — Trade Execution

This document defines the complete flow for both `/soulhunter execute` (full signal evaluation + trade execution, ~10 credits) and `/soulhunter monitor` (price-only check + stop/TP enforcement + news scan + thesis check, 0 Nansen credits).

**Before ANY action, load `~/.soulhunter/layer1-anchor.json`** and read `allowed_bias`. This is your compass. Every decision this cycle must be consistent with the anchor unless you detect a regime change trigger.

## Mandatory Decision Protocol (MDP)

Every trading decision — buy, sell, add, reduce, hold-through-volatility — must pass through this protocol. Monitor and execute differ only in data depth (free vs paid), not in decision rigor. Skipping any step produces a blind spot that costs money.

### MDP Step 1: Collect All Three Data Pillars

You cannot make a decision without all three. A decision based on two out of three is a guess.

| Pillar | Monitor (free) | Execute (paid) |
|--------|---------------|----------------|
| **News** | BlockBeats newsflash API (free, mandatory) | Same |
| **SM Intelligence** | Last known oracle positions from portfolio.json + strategy.json | Fresh Nansen SM trades (~5 credits) + oracle position refresh (~5 credits) |
| **Technicals** | soulpass price (free) + price history from portfolio.json | Same + token screener data |

**News is never optional.** The BlockBeats newsflash API is free and takes one curl call. Trading blind is how you get killed.

```bash
# MANDATORY every cycle — both monitor and execute
curl -s -H "api-key: $BLOCKBEATS_API_KEY" \
  "https://api-pro.theblockbeats.info/v1/newsflash" \
  -G --data-urlencode "size=10" --data-urlencode "lang=en"
```

If `BLOCKBEATS_API_KEY` is not set, check with the user. If truly unavailable, you may proceed but MUST note "NEWS BLIND" in all decision outputs as a risk flag.

### MDP Step 2: Agent Debate

After collecting all three pillars, spawn two parallel subagents with opposite mandates for every actionable decision (entry, exit, size change, hold-vs-close).

```
Buy/Hold/Add Agent (model: sonnet):
  "Given this data, argue aggressively for [BUY/HOLD/ADD]. 
   Data: [prices, news, SM activity, trend, position details, trade history].
   Be SPECIFIC. No hedging. Max 150 words."

Sell/Close/Reduce Agent (model: sonnet):
  "Given this data, argue aggressively for [SELL/CLOSE/REDUCE].
   Data: [identical data as Buy agent].
   Be SPECIFIC. No hedging. Max 150 words."
```

Both agents run in parallel. Each gets identical raw data including:
- Current prices and price history
- News headlines from this cycle
- SM trade activity / oracle positions
- Current portfolio state and trade history
- Any relevant lessons from memory (past mistakes)

### MDP Step 3: Synthesize and Decide

Read both arguments. Decide using these filters in order:

1. **Data vs Emotion** — which argument cites specific data points vs vague feelings?
2. **Layer 1 Alignment** — which argument is consistent with the regime anchor?
3. **Regret Minimization** — if wrong, which action would you regret MORE?
4. **Default to Less** — when arguments are 50/50, do less (don't over-manage)

### When to run MDP

| Situation | Run full MDP? |
|-----------|--------------|
| Buying a new token | YES — full 3-pillar + debate |
| Selling a position early (before stop/TP) | YES — full 3-pillar + debate |
| Stop-loss triggered mechanically | NO — execute immediately, debrief after |
| TP1 hit | NO — execute the pre-planned partial sell |
| Major price move (>15% in 4h on meme coin) | YES — escalate to full data pull + debate |
| Routine monitor with no signals | NO — just log "no action, thesis intact" |
| Holding through volatility (position losing >10%) | YES — debate hold vs close |

The point: mechanical rules (stops, TPs) execute without debate. Judgment calls (new entries, early exits, sizing changes) require the full protocol.

### MDP Output Format

Every decision must be logged with this structure:

```
DECISION: [ACTION] [TOKEN]
├── News Pillar: [summary of relevant news, or "no relevant news"]
├── SM Pillar: [SM activity summary]
├── Technical Pillar: [price, trend, support/resistance]
├── Buy Case: [agent's argument, 1-2 sentences]
├── Sell Case: [agent's argument, 1-2 sentences]
├── Synthesis: [why you chose this action]
├── Layer 1 Alignment: [yes/no + explanation]
└── R:R Ratio: [X:1]
```

---

## Important: Stop-losses require agent execution

Stop-losses and trailing stops are NOT on-chain limit orders — they are stored in portfolio.json and evaluated every time `/soulhunter monitor` or `/soulhunter execute` runs. This means:

- With 4-hour monitoring, a stop-loss can be up to 4 hours late
- In a flash crash, the actual exit price may be worse than the stop price
- This is acceptable for the strategy's timeframe (days to weeks, not minutes)
- The thesis invalidation check catches many bad situations before the stop is even needed

---

## Monitor Loop (every 4 hours, 0 credits)

The monitor is the most critical safety mechanism. It runs every 4 hours and uses ONLY free data (soulpass price + BlockBeats news). It is a decision-making trader, not just a safety net.

### Step M0: Collect Three Data Pillars (MDP Step 1)

Before any position checks, gather all three pillars. This is the foundation.

**Pillar 1: News** (mandatory, free)
```bash
curl -s -H "api-key: $BLOCKBEATS_API_KEY" \
  "https://api-pro.theblockbeats.info/v1/newsflash" \
  -G --data-urlencode "size=10" --data-urlencode "lang=en"
```
If API key missing, warn user and flag "NEWS BLIND" on all decisions.

**Pillar 2: Technicals** (free)
```bash
soulpass price <MINT1> <MINT2> ... SOL
```
Get prices for all open positions + SOL + watchlist tokens. Use mint addresses for non-standard tokens.

**Pillar 3: SM Intelligence**
- Monitor: use last known data from strategy.json + oracle-wallets.json (0 credits)
- Execute: pull fresh SM trades + oracle positions (~10 credits)

Run news and price fetches in parallel — they're independent.

### Step M1: Parse News Headlines

Read the newsflash results from M0. Assess each headline:
1. Does this change the macro regime?
2. Does this impact any held position or watchlist token?
3. Does this create a new opportunity?

If HIGH IMPACT news is found that threatens an open position, flag it — the thesis check in M5 will decide whether to close using full MDP (debate included).

### Step M1.5: Full Market Scan (NEW — from ApexHunter)

Check ALL watchlist tokens from strategy.json, not just held positions. The best trade might be in a token you're NOT holding yet.

```bash
soulpass price <ALL_WATCHLIST_TOKENS> SOL
```

For each watchlist token:
- Has price moved to a favorable entry zone since last check?
- Has a setup formed (pullback to support, dip on no news)?
- Any relative strength divergence vs SOL?

If a clear opportunity is spotted → flag for MDP evaluation after position management (Step M6.5).

### Step M2: Stop-Loss Check (HIGHEST PRIORITY)

For each open position, check if the stop-loss has been hit:
- If `current_price <= stop_loss_price` -> **SELL immediately** via soulpass swap
- If ATR data isn't available: use the fixed -15% fallback stop

**Do not hesitate. Do not wait. Stop means stop.**

**Small capital stop adjustment (≤ $1K):** Use structural stops (below key support levels) instead of tight ATR-based stops. Rationale: with small positions on volatile meme coins, tight stops generate more losing trades from normal volatility. A structural stop below the last significant low is more meaningful than 1.5× ATR on a 20% daily range token.

### Step M3: Take-Profit + Trailing Stop Check

**Asset tier determines TP system.** See Step 5c (full execute flow) for the complete tier-specific TP rules: Tier S (SOL) uses trend-based exits, Tier A (DeFi) uses two-stage TP, Tier M (Meme) uses aggressive 4-stage TP ladder. The symmetric TP below is the **Tier A default** for small capital:

**Tier A — Small Capital TP Tiers (≤ $1,000) — tighter, faster profit capture:**

TP targets must be SYMMETRIC with stop distance. If stop is 12% away, first TP triggers at 12% profit.
**Note: Tier M meme coins use the aggressive TP ladder (+50%/+100%/+300%/+500%) instead — see Step 5c.**

**Tier 1: price moves 1× stop distance in your favor**
- Example: stop is 12% below entry → TP1 triggers at +12%
- Sell 30% of position
- Do NOT move stop yet — let remaining 70% breathe
- Log with exit_reason: "take_profit_tier_1"

**Tier 2: price moves 2× stop distance in your favor**
- Sell another 30% (40% remains)
- NOW move stop to breakeven on remaining position
- Activate trailing stop
- Log with exit_reason: "take_profit_tier_2"

**Tier 3: Trailing stop on remaining 40%**
- Trail at 8% from peak (meme coins need wider trail than perps)
- Update peak_price_since_entry each cycle
- If triggered: close remaining position
- Log with exit_reason: "trailing_stop"

**Why this is better:** Old system had TP1 at +30% but stop at 12%. Most meme coin trades got stopped or time-limited before reaching TP1. New system makes TP1 reachable — same distance as stop. This captures the 1× R:R quickly, then lets remaining position run for asymmetric upside.

**Standard Capital TP (> $1,000):** Keep original staged system (50% at +30%, trail on rest).

### Step M4: Winner Management (Druckenmiller + Minervini — learned from ApexHunter)

**Lock in gains — but don't strangle the trade:**

- Move stop to breakeven ONLY when price has moved ≥ 2× ATR from entry (not just +10%)
- This ensures stop is outside normal noise range. A 3% move on a meme coin with 15% daily ATR is noise — don't lock BE on noise.
- Once BE is justified (2× ATR move confirmed):
  - Move stop to +2% profit lock (not just breakeven)

**Scale into winners (Druckenmiller concentration):**
- Position profitable AND thesis strengthening (SM still accumulating per last research) → add 50% more size
- Only scale in if R:R from CURRENT price to stop/target still ≥ 3:1
- Max one scale-in per position
- Recalculate average entry and adjust stop accordingly

**Why this matters:** ApexHunter learned that a BE stop at +0.6% move got triggered by normal volatility. The thesis was right, but a tight BE killed the trade. Wait for 2× ATR confirmation before locking gains.

### Step M5: Thesis Health Check (CRITICAL — from Day 1)

**Every position has a thesis. Every monitor cycle, ask: "Is my thesis still alive?"**

Load each position's `thesis.invalidation_conditions` from portfolio.json. For each condition:
- Can you check it with free data? (price levels, time elapsed, regime status)
- Has the condition been triggered?

**Thesis status progression:**
- `alive` -> thesis intact, all good
- `weakening` -> one invalidation condition partially triggered (e.g., SM flow slowing but not reversed)
- `dead` -> invalidation condition fully triggered

**Thesis dead -> CLOSE NOW.** Don't wait for the stop. Don't wait for the next execute cycle. A dead thesis means the entire reason for the trade is gone. The stop-loss is for when you're wrong about direction — thesis death means you're wrong about EVERYTHING.

Update `thesis.status` and `thesis.events[]` in portfolio.json:
```json
{
  "events": [
    {"time": "2026-04-01T14:00:00Z", "status": "weakening", "reason": "SM 7d flow ratio dropped from 15x to 3x"},
    {"time": "2026-04-01T18:00:00Z", "status": "dead", "reason": "SM 7d netflow turned negative — flow reversal confirmed"}
  ]
}
```

### Step M6: Breaking News Scan (BlockBeats API)

Use the blockbeats-skill to scan for breaking news that could impact positions:

**Read the headlines yourself** (use your LLM judgment, not keyword matching). For each headline, judge:
1. Does this change the macro regime? (BTC crash, regulatory action, hack)
2. Does this directly impact any held token? (project hack, team exit, listing/delisting)
3. Does this create a new opportunity? (major partnership, ecosystem grant)

**HIGH IMPACT + counter to position thesis -> close immediately:**
- Solana network outage while holding SOL ecosystem tokens
- Token project rug/hack/exploit
- Major regulatory action against held token's sector

**MEDIUM IMPACT -> flag for next execute cycle:**
- General market sentiment shift
- Competitor token news

**Deduplicate news** using `last_newsflash_id` in `~/.soulhunter/funding-tracker.json` to avoid processing the same news twice.

### Step M7: Time Limit Check

If any position has exceeded its `exit_deadline`:
- If profitable -> exit (take the win)
- If losing < -5% AND regime favorable -> extend 3 days (one extension only)
- If already extended -> exit regardless

### Step M8: Circuit Breaker

- Daily loss > 8% -> `freeze_new_entries = true` (exits only)
- Weekly loss > 12% -> `full_freeze = true` (stops only, pause all activity)

### Step M9: Update Portfolio

- Log any closed trades with exit_reason (including `thesis_invalidation`)
- Update price_history, peak_price_since_entry
- Update daily_pnl
- Write diary entry if any actions were taken

### Step M10: Print Monitor Summary

```
SoulHunter Monitor | [timestamp] | Regime: [X]
Positions: [N] open | Thesis: [N] alive, [N] weakening
[TOKEN1]: $X.XX (PnL +Y%) | thesis: alive | stop: $X.XX (Z% away)
[TOKEN2]: $X.XX (PnL -Y%) | thesis: weakening | trailing active at $X.XX
News: [N] items scanned, [N] relevant
Next monitor: in 4 hours | Next execute: [time]
```

---

## Execute Flow (every 12 hours, 0 Nansen credits for position management, ~5 credits optional for SM refresh)

**IMPORTANT: Run the full Monitor Loop (M1-M9) FIRST before evaluating new entries.** Process all exits before opening new positions. Free capital from dead positions before committing to new ones.

Execute does everything Monitor does, PLUS:
- Refreshes SM flow data for existing positions (thesis validation with real data, not just price)
- Evaluates new entry signals from strategy.json watchlist
- Runs scale-in checks for staged entries
- Applies six-gate entry filter system

## Pre-flight Checks

Before doing anything, verify:

1. **Strategy file exists and is valid**
```bash
cat ~/.soulhunter/strategy.json
```
If `valid_until` has passed, tell the user: "Strategy expired. Run `/soulhunter research` first to generate a new one."

2. **Portfolio state exists**
```bash
cat ~/.soulhunter/portfolio.json
```
If it doesn't exist, create it from `templates/portfolio-template.json`. Ask the user for starting capital.

3. **Wallet has funds**
```bash
soulpass balance --token USDC
```
Verify USDC balance matches (approximately) what portfolio.json says for available capital.

4. **Regime still valid**
```bash
soulpass price BTC SOL
```
Compare current BTC/SOL prices against `regime.btc_price_at_generation` and `regime.sol_price_at_generation` in strategy.json.

**Regime override rules** (concrete thresholds):
- BTC dropped > 8% from `btc_price_at_generation` → **override to Risk-Off**: max exposure 15%, only conviction ≥ 0.75 entries, all position sizes halved
- BTC dropped > 10% → **FULL PAUSE**: no new entries, only process stop-losses
- SOL dropped > 15% → **FULL PAUSE** same as above
- BTC rose > 8% from generation → regime may have shifted to Risk-On. Warn user to re-run `/soulhunter research`.

**Risk-Off override parameter adjustments** (applied on top of strategy.json values):
```
position.size_percent = min(strategy_value, 0.03)
position.stop_loss_atr_multiplier = min(strategy_value, 1.5)
position.take_profit_percent = min(strategy_value, 0.15)
position.max_hold_days = min(strategy_value, 7)
risk_limits.max_portfolio_exposure_percent = 0.15
entry_rules.min_conviction = 0.75
```

## Execution Loop

### Step 1: Get current prices for all relevant tokens

```bash
soulpass price TOKEN1 TOKEN2 TOKEN3 ... BTC SOL
```

Use the token symbols from strategy.json watchlist + sell_watchlist + all open positions. Always include BTC and SOL for regime tracking. Record all prices.

### Step 2: Update ATR and volatility data

For each token with price history in portfolio.json:
- Append today's price
- Recalculate ATR: `avg(|close_n - close_n-1|)` over the last 14 data points
- If ATR has changed significantly from strategy.json value, use the **updated** ATR for stop-loss calculations

Also update the **peak price since entry** for each open position (used by trailing stop).

### Step 3: Check daily loss circuit breaker

Calculate today's unrealized + realized PnL from portfolio.json.
- If daily loss > 8% of total capital -> **STOP**. Only process stop-losses and trailing stops (Step 5), do not open any new positions. Tell the user why.
- If weekly loss > 12% of total capital -> **FULL STOP**. Process stop-losses, then freeze all activity. Recommend `/soulhunter evolve` to diagnose what went wrong.

### Step 4: Check sell watchlist alerts

Cross-reference the `sell_watchlist` from strategy.json against open positions:
- If any held token appears on the sell watchlist → **ALERT** the user
- If held token has both negative SM netflow AND distribution DCA signals noted in strategy → **recommend immediate exit** (execute in Step 6)
- If held token appears on sell watchlist but signals are mild → flag for monitoring, tighten trailing stop by 2%

### Step 5: Check exit signals for open positions

For each position in portfolio.json, evaluate exits in this priority order:

**a-0) Reflexivity Cycle Check (Tier M meme coins only)**

For meme coin positions, check the cycle stage from strategy.json FIRST — it overrides all other exit logic:

```
Load cycle_stage for this token from strategy.json
If cycle_stage changed since last check:
  - Seeding → Sprouting: normal, consider adding (scale up to 6-8% risk if conviction holds)
  - Sprouting → Eruption: HOLD, activate trailing stop (1.5× ATR), do NOT add
  - Eruption → Frenzy: SELL 50% immediately, tighten trailing to 1.0× ATR on remainder
  - Frenzy → Collapse: SELL 100% immediately, no debate needed
  - Any stage → 7d flow turns negative: SELL 100% immediately
```

If cycle_stage data is stale (>24h old for meme coins), use current price action as proxy:
- Price up >30% since last check + volume spike → likely Eruption/Frenzy territory → set trailing stop
- Price down >15% since last check + volume spike → likely Collapse → exit

Cycle-based exits are MECHANICAL (like stop-losses) — no MDP debate needed. The cycle positioner already embodies the analysis.

**a) ATR-based stop-loss hit?**
```
current_price ≤ entry_price - (atr × atr_multiplier)
```
The `atr_multiplier` comes from strategy.json (set per-token based on conviction and regime).

Example: entry at $3.42, ATR = $0.25, multiplier = 2.0 → stop at $3.42 - $0.50 = $2.92

If ATR data isn't available (new token, < 5 price points): fall back to a fixed -15% stop-loss.

**b) Trailing stop hit? (high/medium conviction positions only)**

Trailing stop activates once the position reaches a certain profit threshold, then follows the price up:
- **Activation**: when unrealized profit > 10%
- **Trail distance**: set per-token in strategy.json (typically -5% to -8% from peak)

```
peak_price = max(all prices since entry)
trailing_stop_price = peak_price × (1 + trailing_stop_percent)
if current_price ≤ trailing_stop_price AND trailing stop is activated → SELL
```

Example: entry at $3.42, price peaked at $4.50 (activated at +10%), trail = -6% → sell if price drops to $4.23.

Update `peak_price_since_entry` in portfolio.json every execution.

**c) Staged take-profit hit?**

Top traders don't sell everything at one target — they scale out to let winners run.

**Tier-specific take-profit systems:**

**Tier S (SOL) — hold for trend, not for targets:**
```
No fixed TP — SOL is a trend position, not a trade.
Use trailing stop only (2× ATR from peak).
Exit on Weinstein Stage transition (Stage 2 → Stage 3) or daily EMA crossover (EMA21 < EMA50).
```

**Tier A (DeFi Blue-Chips) — two-stage TP:**
```
Stage 1: current_price ≥ entry_price × 1.30 (default: +30%)
  → Sell 50% of position
  → Move stop-loss to breakeven (entry_price)
  → Let remaining 50% ride with trailing stop
  → Record: partial_exit = true, remaining_amount, breakeven_stop_active = true

Stage 2: trailing stop on remaining 50%
  → Already handled by step 5b
  → With breakeven stop as floor, worst case is 0% loss on remaining half
```

**Tier M (Meme Coins) — aggressive TP ladder (capture reflexive pump):**
```
TP1: current_price ≥ entry_price × 1.50 (+50%)
  → Sell 30% → recover most of cost basis
  
TP2: current_price ≥ entry_price × 2.00 (+100%)
  → Sell 30% → locked profit, remaining 40% is house money
  → Tighten trailing to 1.5× ATR
  
TP3: current_price ≥ entry_price × 4.00 (+300%)
  → Tighten trailing to 1.0× ATR (protect the big win)
  
TP4: current_price ≥ entry_price × 6.00 (+500%)
  → Sell 20%, trail final 20% at 0.75× ATR
  → This is the "let it go parabolic" portion — house money riding a rocket
```

This gives asymmetric payoff: if a meme coin goes +30% with old system, you capture only +20% on a full exit. With the ladder, a +200% move captures +50% on 30%, +100% on 30%, and +170% (trailing from peak) on remaining 40%. Weighted average: +107%.

**For low conviction positions**: skip staged take-profit, use fixed exit at `take_profit_percent` (sell 100%). Low conviction doesn't warrant riding the position.

**d) Regime-aware time limit?**

Instead of a hard timeout, check both time AND trend:
```
now > entry_time + max_hold_days
```
If time limit hit:
- If position is profitable → exit (take the win)
- If position is losing < -5% AND regime is still favorable → extend by 3 days (one extension only, mark `extended: true`)
- If position is losing AND regime shifted to unfavorable → exit immediately
- If already extended → exit regardless

**Priority: stop-loss > sell-watchlist-alert > trailing stop > take-profit > time-limit.**
Process ALL exits before entries.

### Step 5.5: Deep Thesis Check with SM Data (Execute only — ~5 credits optional)

**This is what makes Execute different from Monitor.** Monitor checks thesis with free data (prices, news). Execute can spend Nansen credits to verify thesis with actual SM flow data.

**Only spend credits if any of these conditions are true:**
- Position has been open > 3 days (time for SM flows to change)
- Position is losing > -5% (thesis may be dying)
- Previous monitor flagged thesis as "weakening"
- Strategy is > 4 days old (data may be stale)

**If spending credits (max 5 per execute cycle):**
```bash
# Check if SM is still buying our held tokens
nansen research token info --chain solana --token <held_token_mint> --pretty  # 0 credits
# If thesis involves SM flow direction:
nansen research smart-money netflow --chain solana --sort net_flow_7d_usd:desc --limit 10 --pretty  # ~5 credits
```

**For each open position, answer ONE question: "Would I open this trade RIGHT NOW with today's data?"**

Check against the position's `thesis.invalidation_conditions`:
- Has SM flow reversed? (7d netflow turned negative)
- Has the oracle wallet exited?
- Has the fund-labeled wallet sold?
- Has the narrative/catalyst been invalidated?

If ANY invalidation condition is triggered -> **thesis is dead -> CLOSE NOW** (Iron Law #6).
Don't wait for the stop. The stop is for being wrong on direction. Thesis death means the reason is gone.

**Think like the big traders:** When oracle CH8Agh6 dumps BUTTCOIN, that's not noise — that's your thesis dying. The stop-loss is for price direction errors. Thesis death means the REASON is gone. Close immediately, don't wait for mechanical stop.

If thesis is weakening (partial triggers) -> tighten trailing stop by 2%, flag in portfolio.

This step runs BEFORE evaluating new entries because freeing capital from dead positions enables better new trades. Holding a position whose thesis is dead costs both unrealized loss AND opportunity cost.

### Step 6: Evaluate entry signals for watchlist tokens

For each token in strategy.json watchlist that you don't already hold:

**Pre-check: Minimum trade amount**
Calculate the trade amount: `capital.available_usd × position.size_percent × staged_first_percent`
- If amount < $5 → **SKIP** (below dust threshold)
- Solana gas is ~$0.001 and Jupiter swap fee is ~0.3%, so any trade above $5 is practical

**Check entry rules (ALL must pass):**

a) **Conviction meets regime threshold?**
   Standard thresholds (capital > $1K):
   - Risk-Off (or override): only enter if conviction >= 0.70
   - BTC-Divergent: only enter if conviction >= 0.60
   - Ranging: only enter if conviction >= 0.50
   - Risk-On / SOL-Outperform: only enter if conviction >= 0.35
   
   Small capital thresholds (capital <= $1K):
   - Risk-Off: >= 0.60
   - BTC-Divergent: >= 0.50
   - Ranging: >= 0.40
   - Risk-On / SOL-Outperform: >= 0.25
   - Read `conviction_score` directly from watchlist entry — it was pre-computed by /research

b) **24h price change in acceptable range?**
   - Get current price: `soulpass price <TOKEN>`
   - Compare vs price_history in portfolio.json (nearest 24h-ago data point)
   - If no price history yet (cold start): compare current price vs strategy generation data

   **IMPORTANT — this rule prevents CHASING PUMPS, not buying dips:**
   - If price went UP > `price_change_24h_max` (+8%) → SKIP, don't chase a pump
   - If price went DOWN > `price_change_24h_min` (-15%) → SKIP, something may be wrong (crash/rug)
   - If price is DOWN between 0% and -15% → **PASS — this is a potential dip-buy opportunity**

   A 10% dip on a high-conviction token with strong SM flows is an IDEAL entry, not a reason to skip.

   **High conviction dip bonus**: conviction ≥ 0.70 AND price dropped 5-15% → flag as "dip buy opportunity" in the decision output. This is where the best entries happen.
   **High conviction pump override**: conviction ≥ 0.70 AND regime is Risk-On → `price_change_24h_max` relaxed to +12%

c) **Pullback from recent high?**
   - Calculate 7-day high from price_history in portfolio.json
   - If current price < 7d_high × (1 - price_pullback_from_7d_high) → pass
   - If not enough history (< 3 data points) → **skip this rule** (first few executions)

d) **Position limit and exposure check**
   - Count open positions. If ≥ `risk_limits.max_open_positions` → don't open more
   - Calculate total exposure: sum of all position costs / total capital
   - If exposure + new_position_size > regime max → don't open more

e) **Not on sell watchlist or blacklist?**
   - If token appears on `sell_watchlist` or `blacklist` -> do not enter

f) **R:R ratio >= 3:1? (PTJ Rule — Iron Law #7)**
   - With symmetric TP system: TP1 is at 1× stop distance (R:R = 1:1 for first 30%)
   - But remaining 70% has unlimited upside via trailing stop
   - Blended R:R = `(0.30 × 1.0 + 0.30 × 2.0 + 0.40 × estimated_trail_exit) / 1.0`
   - For R:R calculation, use TP2 (2× stop distance) as the conservative target
   - R:R < 3:1 on TP2 basis -> **REJECT**, no matter how high the conviction
   - R:R 3:1 to 4:1 -> acceptable, minimum position size
   - R:R >= 5:1 -> excellent, candidate for larger size
   - **Log the R:R ratio for every trade** — this is the single most important number

g) **Cost-benefit check (small capital only, ≤ $1K) — learned from ApexHunter**
   - Estimate expected PnL: `position_size × stop_distance × 2 × win_probability`
   - If expected PnL < $1.00 → **SKIP** — trade can't cover operating costs
   - A $5 position with 12% stop risks $0.60. Even at 3:1 R:R with 50% win rate, expected value ≈ $0.30. Not worth the compute + API cost.
   - This gate prevents "correct but meaningless" trades that waste cycles

**Decision output format** (for transparency — EVERY token gets this, even skips):
```
Token: PUMP | Conviction: 0.85 (high)
├── Signal A (SM flows):    0.28/0.57 — flow accelerating 15x, 3 traders, Fund buying (1.5x weight)
├── Signal B (Oracle):      0.10/0.30 — oracle CH8Agh6 holding, bias_strength 0.7
├── Signal C (Token+Ind):   0.15/0.20 — MCap $8M, 5K holders, $300K liq, reward score high
├── Signal D (Trend gate):  0.08/0.10 — SOL Stage 2, outperforming BTC
├── Signal E (DCA+Yield):   0.08/0.15 — SM DCA active
├── Final Conviction:       0.85
├── Gate 1 (Conviction):    ✅ 0.85 ≥ 0.40 (Ranging small-cap)
├── Gate 2 (Not held):      ✅ not in portfolio
├── Gate 3 (Price action):  ❌ 24h change +12% > 8% max — PUMP CHASE BLOCKED
├── DECISION: ⏸️ WAIT — price needs to cool down

Token: RENDER | Conviction: 0.70 (high)
├── Signal A:               0.20/0.57 — fresh signal, 3 traders
├── Signal B:               0.10/0.30 — oracle 5fkA accumulating
├── Signal C:               0.12/0.20 — solid fundamentals, reward score moderate
├── Signal D:               0.08/0.10 — SOL Stage 2
├── Signal E:               0.05/0.15 — DCA detected
├── Final Conviction:       0.70
├── Gate 1 (Conviction):    ✅ 0.70 ≥ 0.40
├── Gate 2 (Not held):      ✅
├── Gate 3 (Price action):  ✅ -2.3% (dip buy opportunity! +0.05 bonus)
├── Gate 4 (Capacity):      ✅ 1/3 positions, 35% exposure
├── Gate 5 (Min size):      ✅ $15 > $5
├── Gate 6 (R:R):           ✅ 4.2:1 (TP2 at 2× stop / stop 7.1%) ≥ 3:1
├── Gate 7 (Cost-benefit):  ✅ Expected PnL $2.10 > $1.00
├── Thesis: "Fund flow acceleration 8.2x + oracle accumulating + SM DCA active"
├── Invalidation: SM flow reversal, oracle exit, price below $X
└── DECISION: ✅ ENTER — buy $15 via soulpass swap | stop $X | TP1 $X | TP2 $X
```

This tree format makes every decision auditable — the user (and the agent on re-read) can see exactly which gate blocked and why. Every watchlist token gets a full gate evaluation, even skips.

### Step 7: Execute trades

**For exits (sell):**
```bash
soulpass swap --from <TOKEN> --to USDC --amount <full_position_amount> --slippage 100
```

After execution:
- Check tx status: `soulpass tx <hash>`
- Move position from `positions` to `closed_trades` in portfolio.json
- Record: exit_price, exit_time, pnl_usd, pnl_percent, exit_reason, peak_price_during_hold, atr_at_exit
- Tag exit with the specific trigger: "atr_stop", "trailing_stop", "take_profit", "time_limit", "sell_watchlist_alert", "regime_shift"

**For entries (buy):**

Use staged entry for high conviction tokens:
- First buy: 60% of target position size
- Set a "scale_in" flag in the position

For medium conviction: buy full position at once.
For low conviction: buy full position at once (position is already small).

```bash
soulpass swap --from USDC --to <TOKEN> --amount <usd_amount> --slippage 100
```

After execution:
- Check tx status: `soulpass tx <hash>`
- Add to `positions` in portfolio.json
- Record: entry_price, entry_time, amount, cost_usd, atr_at_entry, atr_multiplier, stop_loss_price, trailing_stop_percent, take_profit_percent, exit_deadline, triggered_rules (copy from strategy watchlist conviction_breakdown), conviction_score, conviction_level, regime_at_entry, signal_a_contributed (bool), signal_b_contributed (bool), signal_c_contributed (bool), signal_d_contributed (bool), signal_e_contributed (bool)
- Record thesis: copy `thesis` object from strategy watchlist entry (summary, invalidation_conditions, rr_ratio, rr_calculation, status="alive", events=[])
- Initialize: peak_price_since_entry = entry_price, trailing_stop_active = false

### Step 8: Scale-in check (for staged entries)

For positions with `scale_in: true` that were opened in a previous /execute:
- If price dropped 3-8% from entry (within 1× ATR) → buy the remaining 40% (this is the dip you were waiting for)
- If price went up > 5% from entry → buy remaining 40% (momentum confirmed)
- If price moved sideways (within ±3%) → wait one more cycle
- After 3 cycles of waiting → buy remaining 40% regardless (don't let the capital sit idle)
- Update position in portfolio.json: new average entry price, total amount, total cost

### Step 9: Update price history and metrics

- Append current prices to the price history section of portfolio.json
- Keep the last 30 data points per token (roughly 30 days if running daily)
- Update daily_pnl entry for today
- Update peak_price_since_entry for all open positions
- Recalculate and store current ATR for each held token

### Step 10: Write diary entry

```bash
soulpass diary write --content '{
  "type": "execution_report",
  "timestamp": "<now>",
  "layer1_regime": "Ranging",
  "regime": "risk_on",
  "regime_override": false,
  "actions": [
    {"action": "buy", "token": "JTO", "price": 3.42, "amount": 219, "usd": 750, "reason": "sm_netflow_top3 + dca_accumulation", "conviction": 0.78, "rr_ratio": 4.2, "thesis": "Fund flow acceleration 8.2x on JTO with oracle confirmation"},
    {"action": "sell", "token": "JUP", "price": 1.34, "amount": 670, "usd": 897, "reason": "thesis_invalidation — SM flow reversed", "pnl_percent": -3.2, "thesis_status": "dead"}
  ],
  "thesis_check": {"alive": 2, "weakening": 1, "dead": 0},
  "news_scanned": 5,
  "news_relevant": 1,
  "alerts": ["BONK on sell watchlist — monitoring"],
  "portfolio_value_usd": 5342,
  "open_positions": 3,
  "daily_pnl_percent": 2.1,
  "weekly_pnl_percent": 4.8,
  "total_exposure_percent": 18.5
}'
```

### Step 11: Print summary

Show the user:
- **Layer 1 Anchor**: current regime classification + confidence
- **Regime status**: current regime + any override applied
- **Actions taken**: buys/sells with prices, reasons, conviction scores, and R:R ratios
- **Thesis status**: for each entry, show the one-line thesis; for each exit, show thesis status at exit
- **Exit details**: for each sell, show entry->peak->exit path and which trigger fired (including thesis_invalidation)
- **Current open positions**: with unrealized PnL, distance to stop-loss, trailing stop status, thesis status
- **Sell watchlist alerts**: any held tokens flagged by SM selling
- **News impact**: any relevant BlockBeats news and actions taken
- **Available capital**: amount and % of total
- **Signals that didn't trigger**: and why (which rule blocked entry, including R:R filter)
- **Next recommended execution time**

## When Nothing Happens

Most executions will result in no trades. That's fine and expected. The strategy is patient — it waits for the right conditions. When no trades trigger, just report:

"No signals triggered. Prices checked for X tokens.
- [Token A] closest to entry: needs 3% more pullback from 7d high
- [Token B] blocked by regime (Risk-Off, conviction 0.62 < 0.75 threshold)
- [N] open positions, all within parameters
- Trailing stop active on [Token C] (peak $X.XX, stop at $X.XX)
- Next check recommended in [time]."

## Error Handling

- If `soulpass swap` fails → do NOT retry automatically. Report the error and let the user decide.
- If `soulpass price` fails for a specific token → skip that token this cycle, note the error.
- If portfolio.json is corrupted → stop everything, report the issue, suggest manual review.
- Always verify tx status after a swap before updating portfolio state.
- If BTC price check fails → assume regime unchanged, proceed with caution, warn user.
