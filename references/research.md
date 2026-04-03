# /soulhunter research — Strategy Generation

This document defines the complete flow for `/soulhunter research`. The goal: update the Layer 1 trend anchor, discover oracle wallets, scan SM trading activity, and generate a strategy with conviction-scored watchlist entries backed by thesis + R:R for each candidate.

## Pre-flight

1. Check `nansen account` — confirm sufficient credits (need ~20 for full research)
2. If `~/.soulhunter/strategy.json` exists, note when it was generated — skip research if < 2 days old unless user forces it
3. Load `~/.soulhunter/oracle-wallets.json` if it exists (carry forward discovered oracles)
4. **Load `~/.soulhunter/layer1-anchor.json`** — this is your compass. Read the regime classification and `allowed_bias` BEFORE any signal analysis. All subsequent decisions must be consistent with Layer 1 unless overwhelming evidence warrants a regime change.

## Architecture: Five Signals, Hierarchical Decision

Based on empirical Solana on-chain data analysis, our edge comes from combining five signals in a hierarchical architecture (Foundation → Validation → Catalyst):

1. **Signal A — SM Consensus + Fund Flow Acceleration** (PRIMARY, 35% weight)
   SM netflow with trader_count analysis + 7d vs 30d momentum detection, systematic accumulation pattern recognition.
   This is our highest-conviction signal because: Fund labels are rare on Solana, so when multiple Fund/SM wallets converge on one token, it's a strong institutional signal.
   Apply label-based weighting: Fund → 1.5x, Smart Trader → 1.0x, High Balance/High Activity → 0.5x, Unlabeled → 0.3x (only if trade_value > $10K).

2. **Signal B — Oracle Wallet Tracking + Direction Bias** (SECONDARY, 20% weight)
   Discovered profitable wallets tracked over time. Their new positions inform our watchlist.
   Discovery path: profiler pnl-summary on wallets from SM dex-trades (filtered for trade_value > $1K).
   **Direction Bias**: aggregate oracle wallet activity direction — if most oracles are accumulating the same sector/tokens, that's a macro signal that feeds into Layer 1. Compute direction_bias and bias_strength like ApexHunter's leaderboard bias.

3. **Signal C — Token-Level Confirmation** (TERTIARY, 20% weight)
   who-bought-sold concentration, token info fundamentals, screener alignment, Nansen indicators (risk + reward scores).
   Used as confirmation/rejection, never standalone. Nansen indicators moved here from Signal D for tighter integration with token-level analysis.

4. **Signal D — Trend Confirmation** (FOUNDATION GATE, 10% weight)
   SOL price structure (Weinstein stage), BTC/SOL relative strength, ecosystem health (stablecoin flows, DEX volume breadth, number of tokens with positive SM netflow).
   This is the go/no-go gate — if trend says Risk-Off, all conviction scores are capped regardless of SM signals.

5. **Signal E — DCA Intelligence + Yield Signals** (CATALYST, 15% weight)
   Jupiter DCA detection (institutional accumulation patterns), DeFi yield opportunities, token flow intelligence. DCA orders from SM wallets represent sustained conviction — not a one-time trade, but a planned accumulation strategy.

Why this ordering? On Solana:
- SM dex-trades are structurally small ($1K-$6K range, not $100K+ like ETH)
- Most pump early-buyers are unlabeled wallets (retail/insider driven)
- Fund labels are scarce → each Fund flow is a high-information event
- trader_count >= 3 on same token = genuine independent consensus
- Jupiter DCA orders from SM wallets signal sustained conviction over time
- Trend structure is the go/no-go gate — SM data validates, never overrides trend

---

## Step 0: Layer 1 Anchor — Trend-First Macro Regime (~1 credit)

The Layer 1 Anchor is the foundation of all trading decisions. Trend structure is the ground truth — SM data and news are validation, not primary. Update `~/.soulhunter/layer1-anchor.json` every research cycle and on any regime change detection.

**See `references/trend-analysis.md` for the complete multi-timeframe methodology** — Weinstein staging, EMA alignment, RSI regime filter, Keltner Channel entry zones, Wyckoff triggers, and regime change detection. This section implements that methodology.

### 0a. Pillar 1: Trend Structure (PRIMARY — the boss)

```bash
# SOL daily candles for Weinstein staging + EMA alignment (~1 credit)
nansen research token ohlcv --chain solana \
  --token So11111111111111111111111111111111111111112 \
  --timeframe 24h --pretty

# Current prices (free)
soulpass price BTC SOL
```

From the daily candle data, compute:

**1. Weinstein Stage (weekly regime):**
Group daily candles into weekly bars. Compute 30-period SMA on weekly closes.
- **Stage 2** (price > rising 30w SMA) = RISK-ON, full allocation allowed
- **Stage 4** (price < falling 30w SMA) = RISK-OFF, max 15% exposure
- **Stage 1/3** (flat SMA, price oscillating) = RANGING, normal parameters

**2. Daily EMA Alignment (trend health):**
Compute EMA21, EMA50, SMA200 on daily closes.
- EMA21 > EMA50 > SMA200 = strong bull → full allocation
- EMA21 > EMA50, both < SMA200 = bull within bear → cautious, smaller positions
- EMA21 < EMA50 < SMA200 = strong bear → CASH mode, max 15% exposure
- EMA21 < EMA50, both > SMA200 = pullback within bull → potential buy zone IF HLs intact

**3. Structure (HH/HL vs LH/LL):**
Identify the last 3-4 swing highs and lows on daily chart.
- HH + HL = uptrend confirmed
- LH + LL = downtrend confirmed
- Structure break (swing low violated in uptrend) = regime change warning

**4. RSI(14) Regime Filter:**
Compute RSI(14) on daily closes.
- RSI > 50 = bull regime
- RSI < 50 = bear regime
- RSI divergence (price new high, RSI lower high) = early warning of regime change, flag for "regime change watch"

**5. BTC/SOL Relative Strength:**
```
SOL_rs = (SOL_now / SOL_30d_ago) / (BTC_now / BTC_30d_ago)
```
- RS > 1.05 = SOL outperforming, bullish for ecosystem
- RS < 0.95 = SOL underperforming, reduce exposure

### 0b. Pillar 2: SM Consensus Direction (VALIDATION — confirms or conflicts)

Look at the aggregate direction of oracle wallets and SM flows from previous research cycle:
- How many oracle wallets are actively accumulating vs distributing?
- What's the net SM flow direction across all tracked tokens?
- Are Fund-labeled wallets adding or reducing Solana exposure?

**Key rule:** If SM conflicts with trend structure → WAIT. Trend wins until SM volume overwhelms structure. If trend says bearish but SM is accumulating, this could be early accumulation OR SM is wrong. Wait for trend confirmation.

This pillar feeds directly from Signal B data — you'll update it after Step 2. On the first run, mark as "insufficient_data".

### 0c. Pillar 3: Ecosystem Health + News (CATALYST — timing)

Assess Solana ecosystem health using available signals:
- Stablecoin flows into Solana (from SM netflow data — look for USDC/USDT flows)
- Number of tokens with positive SM netflow (breadth indicator)
- General meme coin sentiment: are SM wallets exploring new tokens or concentrating in established ones?

BlockBeats macro news:
```bash
curl -s -H "api-key: $BLOCKBEATS_API_KEY" \
  "https://api-pro.theblockbeats.info/v1/newsflash" \
  -G --data-urlencode "size=10" --data-urlencode "lang=en"
```

### 0d. Regime Classification

Combine Weinstein Stage + EMA alignment + RSI + BTC/SOL RS + SM consensus into a single regime call:

| SOL Trend | EMA/RSI | BTC | SM Consensus | Regime | Effect |
|-----------|---------|-----|-------------|--------|--------|
| Stage 2 | 21>50>200, RSI>50 | Bull | Accumulating | **Risk-On** | Full allocation, wider take-profits |
| Stage 2 | 21>50 | Bear/Chop | Any | **BTC-Divergent** | 50% exposure cap, tighter stops |
| Stage 2 | 21<50 (bearish cross) | Any | Any | **Regime Watch** | Reduce by 30%, monitor for Stage 3 transition |
| Stage 4 | Any | Any | Any | **Risk-Off** | 15% max exposure, only highest conviction |
| Stage 1/3 | Flat | Chop | Neutral | **Ranging** | Normal parameters |
| Stage 1/3 | Flat | Chop | Accumulating | **SOL-Outperform** | Normal, slight bias to SOL ecosystem |
| Any | RSI divergence | Any | Any | **Regime Watch** | Early warning — reduce by 30%, flag for next cycle |
| Any | Any | Any | Distributing | **Cautious** | Reduce exposure by 30%, tighten stops |

### 0e. Circuit Breaker
BTC dropped >10% in 7 days → **PAUSE**. No new strategy. Only run /execute for stop-losses.
SOL dropped >15% in 7 days → **PAUSE**. Same as above.

### 0f. Define Regime Change Triggers
Set specific, measurable conditions that would cause a regime reassessment (see `references/trend-analysis.md` → "Regime Change Detection" for the full early warning sequence):
```json
{
  "regime_change_triggers": [
    "SOL breaks above $X (above 30w SMA, Stage 2 confirmed) -> reassess to Risk-On",
    "SOL breaks below $X (below 30w SMA, Stage 4 confirmed) -> reassess to Risk-Off",
    "Daily EMA21 crosses below EMA50 -> reassess, potential regime downgrade",
    "Daily RSI divergence detected (price new high, RSI lower high) -> flag regime watch",
    "SOL daily structure break (last swing low violated in uptrend) -> confirm regime change",
    "3+ oracle wallets start distributing -> reassess SM consensus",
    "BTC/SOL RS drops below 0.95 -> reassess ecosystem health"
  ]
}
```

### 0g. Update Layer 1 Anchor File
Write the updated analysis to `~/.soulhunter/layer1-anchor.json` with:
- Classification, confidence score (0-1), pillar details
- Allowed bias (e.g., "full_allocation", "cautious_longs_only", "preservation_mode")
- Regime change triggers (specific price levels and conditions)
- History entry with date, regime, and event description

---

## Step 1: Signal A — SM Consensus + Fund Flow Acceleration (~5 credits)

This is the primary signal. We're looking for tokens where institutional money is converging.

### 1a. SM Netflow — Full Picture (1 call)
```bash
nansen research smart-money netflow --chain solana --sort net_flow_7d_usd:desc --limit 30 --pretty
```

From the results, extract TWO key metrics for each token:
- `net_flow_7d_usd` — recent direction
- `net_flow_30d_usd` — longer-term trend
- `trader_count` — how many independent SM wallets are involved

**Build the Flow Acceleration Table:**

| Token | 7d Flow | 30d Flow | 7d/30d Ratio | trader_count | Signal |
|-------|---------|----------|--------------|-------------|--------|
| PUMP  | $728K   | $48K     | 15.2×        | 3           | 🔥 ACCELERATING (new conviction) |
| MPLX  | $297K   | $297K    | 1.0×         | 2           | ⚡ FRESH (all buying this week) |
| JUP   | $415    | $674K    | 0.001×       | 2           | 📉 DECELERATING (buying slowed) |
| NEET  | -$1.7K  | $27K     | negative     | 4           | ⚠️ REVERSING (was buying, now selling) |

**Interpretation rules:**
- `7d/30d ratio > 4` AND `trader_count ≥ 2` → **ACCELERATING** — highest conviction. SM is piling in faster than before.
- `7d ≈ 30d` (ratio 0.8-1.2) AND both positive → **FRESH** — all buying happened this week. Strong but could be one-time.
- `7d/30d ratio < 0.25` AND `30d > 0` → **DECELERATING** — the buying impulse is fading. Lower conviction for new entries.
- `7d < 0` AND `30d > 0` → **REVERSING** — SM was buying but is now selling. Exit signal for held positions.
- `7d > 0` AND `30d < 0` → **CONTRARIAN** — recent short-term buying against longer downtrend. Needs extra validation.
- `trader_count ≥ 4` on any signal → **CONSENSUS BONUS** — multiple independent SM wallets agree. Upgrade conviction.

### 1a-bis. Reflexivity Cycle Positioner — The Big Win Engine

**Relationship to Signal A scoring:** The Cycle Stage and Signal A are two views of the SAME 7d/30d ratio data. Signal A scores conviction (35% weight) for entry decisions. The Cycle Stage determines position management rules in execute.md (when to add, hold, or exit). They complement — Signal A says "how confident are we?", Cycle Stage says "where are we in the reflexive loop?" Both are computed from the same ratio; they never conflict because they answer different questions.

The flow acceleration ratio isn't just a signal — it's a **cycle positioner**. Soros's reflexivity theory applies perfectly to Solana tokens: SM buying → price rises → retail notices → more buying → price accelerates → narrative forms → FOMO → exhaustion → collapse. The 7d/30d ratio tells you WHERE you are in this cycle, and that determines everything: whether to enter, how much to size, and when to exit.

**Five Cycle Stages:**

| Stage | 7d/30d Ratio | Price Action | SM Behavior | Your Action |
|-------|-------------|-------------|-------------|-------------|
| **Seeding** | 0-2 | Flat or slight uptick | SM quietly accumulating, low trader_count (2-3) | **ENTER** — probe size (3-4% capital risk). This is where big wins START. Most people miss this because "nothing is happening." |
| **Sprouting** | 2-5 | Rising, volume picking up | SM continues buying, trader_count growing | **ADD** — scale to 6-8% capital risk. Price confirms SM was right. |
| **Eruption** | 5-15 | Accelerating, CT starts buzzing | SM still holding, retail entering | **HOLD** — set trailing stop (1.5× ATR from peak). Do NOT add. Ride the reflexive loop. |
| **Frenzy** | >15 OR SM starts selling | Parabolic, "everyone" talking | SM beginning to distribute | **EXIT 50%** — sell half immediately. Trail remaining 50% with tight 1.0× ATR stop. |
| **Collapse** | Ratio declining + 7d negative | Dumping | SM fully exited or shorting | **EXIT 100%** — all out. Do NOT buy the dip. Wait for new Seeding stage. |

**Why this is the big win mechanism:**
- Entry at Seeding with 3-4% risk
- Price moves +200% to Eruption stage
- Your probe is now +200%, trailing stop locks in +150% minimum
- Even with 50% exit at Frenzy, remaining 50% might ride to +500%
- **Total return: 20-40% of capital from a 3-4% risk trade = 5:1 to 10:1 R:R**
- You need this to work 2-3 times per quarter. The other 7-8 trades can all be stop-loss exits and you still win on net.

**Cycle Stage overrides conviction scoring:**
- Seeding/Sprouting → use conviction score normally for entry decision
- Eruption → do NOT enter new positions (you're late), only hold existing
- Frenzy → exit signal overrides ALL buy signals, even if conviction is 0.95
- Collapse → immediate exit regardless of thesis

Store cycle stage for each watchlist token in strategy.json as `cycle_stage`.

### 1b. Fund-Only Netflow (institutional money specifically)
```bash
nansen research smart-money netflow --chain solana --labels "Fund" --sort net_flow_7d_usd:desc --limit 20 --pretty
```

Fund labels are rare on Solana (~3-5 tokens per week). Every positive Fund flow is a high-information event.

Compare with 1a:
- Token has BOTH SM netflow + Fund netflow positive → **INSTITUTIONAL CONSENSUS** (strongest signal)
- Token has SM netflow positive but no Fund flow → normal SM conviction
- Token has Fund flow only (not in general SM list) → investigate deeper (Fund may be early)

### 1c. SM Holdings — Position Size Validation
```bash
nansen research smart-money holdings --chain solana --sort balance_24h_percent_change:desc --limit 20 --pretty
```

Cross-reference with netflow data. Key fields:
- `value_usd` — how much SM has in this position (filter: > $10K to matter)
- `holders_count` — independent SM wallets holding (filter: ≥ 2)
- `balance_24h_percent_change` — are they still adding?
- `share_of_holdings_percent` — what fraction of total SM portfolio this represents

**Why this matters**: netflow shows direction, holdings shows SIZE. A $225K SM position (like WOJAK with 5 holders) is a much stronger signal than a $1K position (like PIGEON with 1 holder).

### 1d. Sell Pressure Detection
```bash
nansen research smart-money netflow --chain solana --sort net_flow_30d_usd:asc --limit 15 --pretty
```

Tokens with the most negative 30d SM netflow = SM is distributing. If ANY token you currently hold appears here → **flag for exit review**.

Also watch for the PUNCH pattern: `7d positive but 30d negative` = SM is divided. High trader_count with divergent flows = controversy. Avoid or reduce position.

---

## Step 2: Signal B — Oracle Wallet Discovery & Tracking (~8 credits)

Oracle wallets are SM wallets with verified track records of profitable trading. Discovery is a multi-week process.

### 2a. SM Dex-Trades — Filter for Meaningful Trades
```bash
nansen research smart-money dex-trades --chain solana --limit 50 --sort trade_value_usd:desc --pretty
```

**Critical filter**: On Solana, SM dex-trades are full of $9-$100 micro-cap sniping from bots. Only consider trades where `trade_value_usd > $1,000`. Everything below is noise.

From filtered results, record unique `trader_address` values. These are wallets worth profiling.

### 2b. Profile Promising Wallets
For each unique wallet from 2a with meaningful trade sizes:
```bash
nansen research profiler pnl-summary --address <wallet> --chain solana --pretty
```

Note: `profiler labels` requires paid plan. `pnl-summary` works on free plan and tells us:
- `top5_tokens` — what they've traded and their realized PnL/ROI
- Consistent positive ROI across multiple tokens = potential oracle wallet

**Oracle wallet criteria:**
1. Profitable on 3+ of top 5 tokens (consistent, not lucky)
2. Average realized_roi > 0.3 (30%+ returns)
3. Active in the last 30 days (still trading)
4. Trades across different token types (not just one niche)

### 2c. Track Known Oracle Wallets
If `~/.soulhunter/oracle-wallets.json` exists, check what oracle wallets are doing now:
```bash
nansen research profiler balance --address <oracle_wallet> --chain solana --pretty
```

Compare current holdings with last week's snapshot:
- New positions = potential entry signals
- Reduced positions = exit signals
- Same positions = continued conviction

### 2d. Update Oracle Wallet List
Update `~/.soulhunter/oracle-wallets.json`:
```json
{
  "updated_at": "2026-03-26",
  "wallets": {
    "5fkA...": {
      "source": "sm_dex_trades",
      "first_seen": "2026-03-26",
      "status": "new_candidate",
      "pnl_data": {
        "top_tokens": ["我的刀盾 +48%", "LIFE +159%", "SHAPE +150%"],
        "profitable_count": 4,
        "avg_roi": 0.93
      },
      "weight": 0.25,
      "last_verified": "2026-03-26"
    }
  }
}
```

Weight by track record:
- `elite_oracle` (≥3 weeks, >60% accuracy): 1.0
- `verified_oracle` (2+ weeks, consistent PnL): 0.75
- `strong_candidate` (1 week, strong PnL data): 0.5
- `new_candidate` (just discovered): 0.25

---

## Step 3: Signal C — Token-Level Confirmation (~5 credits)

For each candidate token from Steps 1-2, do deeper analysis.

### 3a. Free Pre-filter with Token Info (0 credits)
```bash
nansen research token info --chain solana --token <mint> --pretty
```

Hard filters (reject if ANY fails):
- `market_cap_usd` < $1M → too small, rug risk
- `total_holders` < 500 → insufficient distribution
- `liquidity_usd` < $100K → can't exit positions
- `token_age_days` < 7 → too new, no track record

### 3b. Who-Bought-Sold Concentration (top 2-3 tokens only)
```bash
nansen research token who-bought-sold --chain solana --token <mint> --pretty
```

Look at the buyer distribution:
- **Concentrated buying** (top 3 buyers = 80%+ of volume) → whale-driven, risky
- **Distributed buying** (top 10 buyers each < 15% of volume) → organic demand, healthier
- **Labeled buyers** present (Fund, Smart Trader, Token Millionaire) → institutional interest confirmation
- **Net accumulator ratio**: `bought_volume_usd / (bought_volume_usd + sold_volume_usd)` across all buyers. > 0.7 = strong accumulation phase.

### 3c. Screener Cross-Reference
```bash
nansen research token screener --chain solana --timeframe 7d --smart-money --sort netflow:desc --limit 20 --pretty
```

Check if our Signal A tokens also appear in the SM-filtered screener. Alignment = confirmation.

Also note `nof_traders` in screener results — another consensus measure.

### 3d. Nansen Indicators (for top 2-3 candidates)
```bash
nansen research token indicators --chain solana --token <mint> --pretty
```

Use the Nansen Score components:
- **Risk score**: high risk → lower position size, tighter stops. > 70th percentile → -0.05 penalty.
- **Reward score**: high reward → wider take-profits. > 70th percentile → +0.05 bonus.

---

## Step 3.5: Signal D — Trend Confirmation (~1 credit for token OHLCV)

The trend gate — this data is collected in Step 0 (Layer 1 Anchor) and feeds into conviction scoring as a go/no-go gate. Also provides entry zone identification for watchlist tokens.

**For each top-3 watchlist token, pull 1H candles for entry zone analysis:**
```bash
nansen research token ohlcv --chain solana --token <mint> --timeframe 1h --pretty
```

**Keltner Channel on 4H** (group 1H candles into 4H bars):
```
middle = EMA(close, 20)  on 4H bars
ATR14 = ATR(14) on 4H bars
upper = middle + 2 × ATR14
lower = middle - 2 × ATR14
```

Record the current price position within the Keltner Channel for each token:
- Above upper band → strong trend, don't chase, wait for retrace to middle
- Near middle band → **PRIMARY ENTRY ZONE** — buy here with the trend
- Below lower band → oversold / panic dump → look for Spring pattern (see `references/trend-analysis.md` → "1H Triggers")

Also compute for each token:
- **Daily RSI(14)** from the 1H data (aggregate to daily)
- **EMA21/EMA50 alignment** on 4H — trend direction on execution timeframe
- **Swing structure** — is this a pullback to entry zone or a structure break?

See `references/trend-analysis.md` for the complete methodology on each indicator.

**Signal D Scoring (max 0.10):**
- SOL in Weinstein Stage 2 (bullish): +0.05
- SOL outperforming BTC (positive relative strength): +0.03
- Ecosystem breadth positive (multiple tokens with positive SM netflow): +0.02
- Token at Keltner middle band (ideal entry zone): +0.02 bonus
- Token daily RSI > 50 (bullish momentum): +0.01
- SOL in Stage 4 (bearish): -0.20 penalty (caps all scores, Risk-Off gate)
- SOL underperforming BTC significantly: -0.05 penalty
- Token at Keltner upper band (overbought/chasing): -0.03 penalty

This signal is low weight but high impact — the penalty for Risk-Off regime effectively caps all conviction scores, ensuring no trades happen against the macro trend. The Keltner Channel position determines WHERE to enter, not WHETHER to enter.

---

## Step 3.7: Signal E — DCA Intelligence + Yield Signals (~3 credits)

Jupiter DCA orders from SM wallets are a uniquely Solana signal. Unlike one-time swaps, a DCA order represents sustained, planned accumulation — the wallet is committing to buy over time. This is institutional behavior.

### 3.5a. SM DCA Detection
```bash
nansen research smart-money dcas --chain solana --pretty
```

Look for:
- SM wallets with active DCA orders on tokens from our watchlist -> **STRONG confirmation**
- DCA order size relative to daily volume -> larger = more conviction
- Multiple SM wallets running DCAs on the same token -> **INSTITUTIONAL ACCUMULATION**

### 3.5b. Token-Level DCA Orders (for top candidates only)
```bash
nansen research token jup-dca --token <mint> --pretty
```

Check for concentration of DCA buying:
- Large total DCA volume relative to market cap -> bullish accumulation
- Multiple independent DCA orders -> distributed conviction (not one whale)

### 3.5c. Nansen Indicators (for top 2-3 candidates)
```bash
nansen research token indicators --chain solana --token <mint> --pretty
```

Use the Nansen Score components:
- **Risk score**: high risk -> lower position size, tighter stops
- **Reward score**: high reward -> wider take-profits

### 3.5d. Flow Intelligence (optional, for high-conviction tokens)
```bash
nansen research token flow-intelligence --chain solana --token <mint> --days 30 --pretty
```

30-day flow pattern reveals:
- Steady inflow -> sustained accumulation (bullish)
- Spike then flat -> one-time event (neutral)
- Declining inflow -> distribution beginning (bearish)

**Signal E Scoring (max 0.15):**
- SM DCA active on token: +0.08
- Multiple SM DCA orders: +0.05
- Flow intelligence shows steady accumulation: +0.03
- DeFi yield opportunity on token: +0.02

---

## Step 4: Generate Strategy File

### 4a. Conviction scoring (revised: five-signal, hierarchical)

**How conviction is calculated:** Each signal produces a raw score (0 to signal_max). The raw score is then normalized to 0-1 range and multiplied by the signal weight. Final conviction = sum of all weighted signal scores + penalties.

```
# Step 1: Compute raw score for each signal (sum of triggered sub-conditions)
# Step 2: Normalize: signal_normalized = min(raw_score / signal_max, 1.0)
# Step 3: Weight: signal_weighted = signal_normalized × signal_weight
# Step 4: conviction = sum(all signal_weighted) + sum(penalties)
# Step 5: conviction = clamp(conviction, 0.0, 1.0)

# Signal A: SM Consensus (weight: 0.35, max raw: 0.35)
# Sub-conditions are cumulative — multiple can fire. Raw score capped at 0.35.
  flow_accelerating:       0.18  (7d/30d ratio > 4 AND trader_count >= 2)
  flow_fresh_strong:       0.10  (7d ~ 30d, both > $10K, trader_count >= 2)
  fund_flow_positive:      0.07  (Fund-label netflow positive — rare and valuable)
  holdings_value_high:     0.04  (SM holdings value > $50K on this token)
  consensus_bonus:         0.04  (trader_count >= 4 in netflow OR holders)
  systematic_accumulation: 0.03  (same SM wallet makes >5 buys in same token over 7d)
  # Sub-total max: 0.46, but CAPPED at 0.35. Exceeding sub-scores = maximum confidence.

  # Signal B: Oracle Wallet + Direction Bias (weight: 0.20, max raw: 0.20)
  oracle_buying:           0.08  (verified+ oracle wallet in current holders)
  oracle_new_position:     0.05  (oracle wallet opened NEW position this week)
  oracle_dca:              0.03  (oracle wallet running accumulation DCA)
  oracle_direction_bias:   0.04  (majority of oracles accumulating in same sector)
  oracle_bias_strong:      0.04  (bias_strength > 0.6 — strong consensus among top wallets)
  # Sub-total max: 0.24, CAPPED at 0.20.

  # Signal C: Token Confirmation + Indicators (weight: 0.20, max raw: 0.20)
  fundamentals_solid:      0.08  (mcap > $5M, holders > 2K, liquidity > $200K)
  accumulation_phase:      0.04  (who-bought-sold ratio > 0.7)
  screener_alignment:      0.03  (appears in SM screener with positive netflow)
  nansen_reward_high:      0.05  (Nansen reward score > 70th percentile)
  # Sub-total max: 0.20, equals cap.

  # Signal D: Trend Confirmation (weight: 0.10, max raw: 0.10 — go/no-go gate)
  sol_stage2_bullish:      0.05  (SOL in Weinstein Stage 2)
  sol_outperforming_btc:   0.03  (SOL/BTC relative strength positive)
  ecosystem_breadth:       0.02  (multiple tokens with positive SM netflow)
  # Sub-total max: 0.10, equals cap.

  # Signal E: DCA Intelligence + Yield (weight: 0.15, max raw: 0.15)
  sm_dca_active:           0.07  (SM wallet running Jupiter DCA on this token)
  multiple_sm_dcas:        0.04  (2+ independent SM DCA orders)
  flow_intel_accumulating: 0.02  (30d flow intelligence shows steady inflow)
  defi_yield_opportunity:  0.02  (token has meaningful DeFi yield opportunity)
  # Sub-total max: 0.15, equals cap.
)

# Example calculation:
# Token PUMP: flow_accelerating(0.18) + fund_flow_positive(0.07) = 0.25 raw Signal A
#   Signal A normalized = min(0.25/0.35, 1.0) = 0.714
#   Signal A weighted = 0.714 × 0.35 = 0.250
# Oracle buying(0.08) = 0.08 raw Signal B
#   Signal B normalized = min(0.08/0.20, 1.0) = 0.400
#   Signal B weighted = 0.400 × 0.20 = 0.080
# fundamentals_solid(0.08) + screener(0.03) = 0.11 raw Signal C
#   Signal C weighted = min(0.11/0.20, 1.0) × 0.20 = 0.110
# sol_stage2(0.05) + rs(0.03) = 0.08 raw Signal D
#   Signal D weighted = min(0.08/0.10, 1.0) × 0.10 = 0.080
# sm_dca(0.07) = 0.07 raw Signal E
#   Signal E weighted = min(0.07/0.15, 1.0) × 0.15 = 0.070
# conviction = 0.250 + 0.080 + 0.110 + 0.080 + 0.070 = 0.590
# After penalties (e.g., nansen_risk_high -0.05): conviction = 0.540

penalties:
  flow_decelerating:      -0.15  (7d/30d ratio < 0.25 — buying fading)
  flow_reversing:         -0.25  (7d negative, 30d positive — SM exiting)
  sm_selling_30d:         -0.20  (appears in bottom of 30d netflow)
  concentrated_buying:    -0.10  (top 3 buyers > 80% of volume)
  low_liquidity:          -0.15  (liquidity < $100K — exit risk)
  nansen_risk_high:       -0.05  (Nansen risk score > 70th percentile)
  trend_risk_off:         -0.20  (Layer 1 regime is Risk-Off — caps all scores)
```

### 4b. Position Sizing — STOP-LOSS BASED (not capital-percentage-based)

Size positions by how much you're willing to lose if the stop triggers, NOT by arbitrary capital percentages. This is the most important change from v3 to v4.

**Formula:** `max_position = (capital x max_loss_percent) / stop_distance_percent`

| Conviction | Max Loss at Stop (% of capital) | ATR Multiplier | Take-Profit |
|------------|--------------------------------|----------------|-------------|
| >= 0.85 (very high) | 8% (max — never exceed) | 2.5x ATR | Staged: 50% at +30%, 50% trailing |
| 0.70-0.84 (high) | 6% | 2.0x ATR | Staged: 50% at +30%, 50% trailing |
| 0.50-0.69 (med) | 4% | 1.8x ATR | Full exit at +25% |
| 0.35-0.49 (low) | 3% | 1.5x ATR | Full exit at +20% |
| < 0.35 | skip | — | — |

**Example (small capital):** Capital $500, PUMP token, entry $0.036, ATR = $0.004, conviction 0.72 (high)
- Stop distance: ATR x 2.0 = $0.008 (22.2% of entry price)
- Max loss: $500 x 6% = $30
- Max position: $30 / 0.222 = **$135** (27% of capital in one trade)
- TP1 at +30% = $0.047: profit on 50% = $20.25
- If remaining rides to +60%: profit = $20.25 + $40.50 = **$60.75 (+12.2% of capital)**

**Example (micro capital):** Capital $300, high conviction (0.85), stop 15% away
- Max loss: $300 x 8% = $24
- Max position: $24 / 0.15 = **$160** (53% of capital)
- This is correct and intentional. The stop IS the risk control, not the position size.

**Compare with old approach:** Old v3 would cap this at 10% = $30 position. Even at +30% TP, profit = $9. Not worth the operating cost. The new approach sizes correctly for the actual risk.

**Regime adjustment (Risk-Off):**
In Risk-Off regime, reduce max loss per trade by 40%:
- Very high conviction: 5% max loss (instead of 8%)
- High conviction: 4% (instead of 6%)
- Only trade conviction >= 0.70 in Risk-Off

**Take-profit strategy by conviction:**
- Very high / High conviction: **staged exit** — sell 50% at TP1 (+30%), move stop to breakeven, let remaining 50% ride with trailing stop. This captures big moves.
- Medium / Low conviction: **full exit** at TP (+20-25%). Don't ride these — they have lower probability of running.

**Flow acceleration auto-upgrade:** if `flow_ratio_7d_30d > 10` AND `fund_flow_positive`, auto-set conviction to minimum 0.85 (very_high). This is a rare, extremely strong institutional signal — don't underscore it.

### Small Capital Conviction Thresholds (<= $1,000)

With small capital, the fixed costs of running the agent demand that you actually trade. Lower thresholds by ~0.10:

| Regime | Standard Threshold | Small Capital Threshold |
|--------|-------------------|------------------------|
| Risk-On | >= 0.35 | >= 0.25 |
| Ranging | >= 0.50 | >= 0.40 |
| Risk-Off | >= 0.70 | >= 0.60 |

**Why:** A strategy that filters out every signal and sits in $300 cash is a guaranteed loss due to overhead costs. The risk of a bad trade ($15-20 loss) is smaller than the cost of inaction. Lower thresholds = more trades = more data for evolution = faster learning.

### 4c. R:R Calculation (PTJ Rule — MANDATORY)

For EVERY watchlist entry, calculate the reward-to-risk ratio BEFORE including it:

```
stop_distance = entry_price - stop_loss_price  (ATR × multiplier)
tp_distance = take_profit_price - entry_price  (TP1 for staged, full TP for simple)
rr_ratio = tp_distance / stop_distance
```

**Hard filter: R:R < 3:1 -> REJECT from watchlist.** This is Iron Law #7.

For staged take-profit positions, use the blended expected return:
```
expected_return = (0.50 × TP1_distance) + (0.50 × estimated_trailing_exit)
rr_ratio = expected_return / stop_distance
```

If R:R is borderline (3:1 to 4:1), note it — the position gets minimum size.
If R:R is excellent (>5:1), note it — candidate for larger position.

### 4d. Thesis Generation (MANDATORY)

For EVERY watchlist entry, write a thesis. This is not optional — it's what separates SoulHunter from a dumb signal follower.

**Thesis structure:**
```json
{
  "thesis": {
    "summary": "Fund wallets accumulating PUMP with 15.2x flow acceleration, 3 independent SM wallets converging, oracle 5fkA also buying",
    "invalidation_conditions": [
      "SM 7d netflow turns negative (flow reversal)",
      "Oracle 5fkA sells PUMP position",
      "Fund-labeled wallet exits",
      "Price drops below 14d low without SM re-accumulation"
    ],
    "rr_ratio": 4.2,
    "rr_calculation": {
      "target_price": 0.048,
      "stop_price": 0.032,
      "potential_profit_percent": 31.5,
      "potential_loss_percent": 7.5
    }
  }
}
```

The thesis invalidation conditions are checked every /monitor and /execute cycle. When ANY condition triggers, the thesis status changes to "dead" and the position is closed immediately — don't wait for the stop loss.

### 4f. Entry rules

```json
{
  "entry_rules": {
    "min_rr_ratio": 3.0,
    "price_pullback_from_7d_high": 0.05,
    "price_change_24h_max": 0.08,
    "price_change_24h_min": -0.15,
    "high_conviction_override_24h_max": 0.12
  }
}
```

### 4g. Sell watchlist

From Step 1d:
- Tokens with 30d negative SM netflow that we currently hold
- Tokens showing REVERSING pattern (7d negative, 30d positive)
- Tokens where oracle wallets reduced positions
- Tokens with PUNCH pattern (SM divided, high trader_count but mixed flows)

### 4h. Save strategy

Write to `~/.soulhunter/strategy.json`. Include:
- `layer1_anchor_ref` — reference to current Layer 1 anchor classification and confidence
- `regime` and parameters
- `signal_a` section (SM consensus data, flow acceleration table, label weights)
- `signal_b` section (oracle wallet activity + direction bias + bias_strength)
- `signal_c` section (token-level confirmation + Nansen indicators)
- `signal_d` section (trend confirmation — SOL stage, BTC/SOL RS, ecosystem breadth)
- `signal_e` section (DCA intelligence + yield signals)
- `watchlist` with conviction scores, R:R ratios, thesis, and position parameters
- `sell_watchlist` for exit signals
- `blacklist` with reasons

Print summary:
- **Layer 1 Anchor**: regime classification, confidence, pillar summary
- **Signal A highlights**: which tokens have accelerating SM flows, Fund flows (with label weights)
- **Signal B**: oracle wallet new positions + aggregate direction bias + bias_strength
- **Signal D**: trend gate status — SOL stage, BTC/SOL RS, ecosystem breadth
- **Signal E**: any SM DCA orders detected (institutional accumulation), yield opportunities
- **Top trades**: tokens with highest combined conviction, R:R ratio, and one-line thesis
- **Sell warnings**: REVERSING or DECELERATING tokens in current portfolio
- **Credits used**

## Step 5: Dynamic Research Frequency

Based on market conditions, set the next research timing:

| Condition | Next research |
|-----------|--------------|
| BTC 7d volatility > 15% | In 3 days (mid-week check) |
| Any ACCELERATING token found with conviction > 0.7 | In 3 days (track momentum) |
| Normal conditions | Next Sunday |
| Risk-Off regime | Every 3 days (monitor closely) |

Save `next_research_date` to strategy.json. /execute should remind the user when it's time.

## Credit Budget (Target: ≤ 18 credits per weekly cycle)

Run `nansen account` before starting.

| Step | Credits | Purpose |
|------|---------|---------|
| Step 0: Regime | 0 | soulpass price (free) |
| Step 1a: SM Netflow (all) | ~5 | smart-money netflow |
| Step 1b: Fund Netflow | ~5 | smart-money netflow (Fund label) |
| Step 1c: SM Holdings | ~5 | smart-money holdings |
| Step 1d: Sell detection | 0 | reuse 1a data (sort differently) |
| Step 2a: SM Dex-Trades | ~5 | smart-money dex-trades |
| Step 2b: Wallet Profiling | ~5 | profiler pnl-summary (×2 wallets) |
| Step 2c: Oracle Tracking | ~5 | profiler balance (×1-2 wallets) |
| Step 3a: Token Info | 0 | token info (free) |
| Step 3b: Who-Bought-Sold | ~5 | token who-bought-sold (×1-2 top candidates) |
| Step 3c: SM Screener | ~1 | token screener (SM filter) |
| **Total** | **~36** | |

With 10,000 credits, this gives us ~277 research cycles. More than enough to iterate and optimize.

**Credit optimization tips:**
- Step 1d reuses Step 1a data — just sort/filter differently in code
- Step 3a (token info) is free — use it liberally for pre-filtering
- Steps 2b-2c (profiling) can be skipped on weeks with no new interesting SM traders
- Step 3b is the most optional — skip if Signal A alone gives high conviction

## First-Time Research (Cold Start)

On the very first run, there are no oracle wallets yet. The first cycle is special:

1. Run Step 0 — Layer 1 Anchor (works from day one with BTC/SOL prices)
2. Run Step 1 fully — SM Consensus works from day one
3. Run Step 2a-2b to START building the oracle wallet list (all wallets start as `new_candidate`)
4. Step 2c is skipped (no historical oracle data yet)
5. Run Step 3 for confirmation on top Signal A candidates
6. Run Step 3.5 for DCA intelligence — this also works from day one
7. First week strategy leans heavily on Signal A (SM Consensus)

**Cold start conviction rebalancing:**
When `oracle_wallets_active = 0` (no verified/elite oracles yet):
- Signal A gets +15% (total 50%)
- Signal B gets 5% minimum — **NOT zero.** If profiler pnl-summary reveals wallets with >60% win rate and >$20K realized PnL, that data is real.
- Signal C gets +5% (total 25%)
- Signal D stays at 10% (trend data is available from day one)
- Signal E stays at 10% (DCA data is available immediately)
- Once oracle wallets reach verified status (week 3+), shift to normal 35/20/20/10/15 weights

**Why not zero B on cold start?** Because the with-skill eval found wallet 3fup with 77% win rate and $59K PnL on its first run. Ignoring that is leaving money on the table. A wallet with proven PnL is a signal regardless of how many weeks we've tracked it.

Concretely, cold-start scoring:
```
Signal A (60%):
  flow_accelerating:       0.28
  flow_fresh_strong:       0.12
  fund_flow_positive:      0.15
  holdings_value_high:     0.10
  consensus_bonus:         0.10

Signal B (10% — cold start minimum):
  oracle_pnl_verified:     0.10  (wallet has >60% win rate AND >$20K realized PnL)
  oracle_buying:           0.00  (need tracking history, can't use yet)

Signal C (20%):
  fundamentals_solid:      0.10
  accumulation_phase:      0.05
  screener_alignment:      0.05

Signal D (10%):
  sm_dca_active:           0.06
  nansen_reward_high:      0.04
```

**Flow acceleration auto-upgrade:** if `flow_ratio_7d_30d > 10` AND `fund_flow_positive` AND `trader_count >= 2`, automatically set conviction floor to 0.85. Don't let the formula underscore a screaming institutional signal.

Tell the user: "First week already uses real data — SM consensus is primary, but newly discovered oracle wallets with strong PnL and DCA intelligence contribute immediately."

## Why This Strategy Beats Pure Beta

Most Solana SM tools just show "what SM is buying" (beta). We add alpha through:

1. **Flow acceleration detection** — not just "is SM buying?" but "is SM buying FASTER than last month?"
2. **Fund scarcity signal** — on Solana, Fund flows are rare events. Each one is high-information.
3. **Consensus counting** — trader_count >= 4 means independent SM wallets agree. Not following one whale.
4. **Oracle wallet layer** — verified profitable wallets tracked over time adds a personal edge.
5. **Multi-timeframe analysis** — 7d vs 30d catches momentum shifts before they show in price.
6. **DCA intelligence** — Jupiter DCA orders from SM wallets signal sustained institutional conviction, not one-time trades.
7. **Thesis discipline** — every position has a thesis with invalidation conditions. When the thesis dies, the position dies.
8. **Layer 1 anchor** — macro regime awareness prevents overtrading in bear markets and undertrading in bull markets.
