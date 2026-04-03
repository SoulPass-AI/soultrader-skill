# Trend Analysis — The Foundation of Every Decision

Trend structure is ground truth. SM data and news are catalysts and validation. This document defines the exact technical analysis process for each timeframe, adapted for Solana spot trading.

## Analysis Order: Top-Down, Always

```
Weekly (SOL + BTC, 30 seconds) → Daily (1 minute) → 4H (1 minute) → Entry trigger
```

Higher timeframes ALWAYS override lower ones. A 1H buy signal inside a daily downtrend is a trap — the trend resumes down. On Solana, you can only go long, so a bearish trend means WAIT, not SHORT.

**Solana-specific reality:** You're trading spot — no leverage, no shorts. This means the trend framework is even MORE important than for perps traders, because your ONLY option is to buy or stay in cash. Buying against a downtrend is the single most expensive mistake in spot trading.

## Data Sources

All technical analysis uses these data sources:

```bash
# SOL candles — daily timeframe (primary for weekly + daily analysis)
# Use 24h timeframe and compute weekly from daily closes
nansen research token ohlcv --chain solana \
  --token So11111111111111111111111111111111111111112 \
  --timeframe 24h --pretty

# Individual token candles — 1h timeframe (for 4H grouping + entry triggers)
nansen research token ohlcv --chain solana \
  --token <mint_address> \
  --timeframe 1h --pretty

# Current prices (free, unlimited)
soulpass price SOL BTC <TOKEN>
```

**Credit note:** Token OHLCV costs ~1 credit per call. Use it strategically — SOL candles for regime analysis, then only pull individual token candles for watchlist candidates that passed SM filters. Don't pull candles for 30 tokens; pull for the 3-5 that Signal A/B already flagged.

## Weekly: Regime Classification (Weinstein Stage)

SOL is the primary reference for all Solana token trading. BTC is the secondary market-wide reference. The Solana ecosystem rises and falls with SOL — meme coins, DeFi tokens, infrastructure tokens all correlate.

**Weinstein 4-Stage Classification:**

From daily candle data, compute 30-period SMA on weekly closes (group 7 daily candles into one weekly bar, use the Friday close or the last day's close). Then classify:

| Stage | Price vs 30w SMA | SMA Direction | Action |
|-------|-----------------|---------------|--------|
| Stage 1 (Base) | Oscillating around flat SMA | Flat | WAIT — no trend, accumulation phase |
| Stage 2 (Uptrend) | Above rising SMA | Rising | BUY — full allocation allowed |
| Stage 3 (Top) | Oscillating around flat/rolling SMA | Flattening | REDUCE exposure, take profits |
| Stage 4 (Downtrend) | Below falling SMA | Falling | CASH — max 15% exposure, only highest conviction |

**This single filter eliminates most losing trades.** If SOL is in Stage 4, don't buy altcoins no matter what SM data says. Fund wallets accumulating during Stage 4 have multi-month time horizons and can absorb 50%+ drawdowns — you likely can't.

**SMA direction determination:** Compare current 30w SMA with 4 weeks ago:
- Rising by > 1% → Rising
- Falling by > 1% → Falling
- Within ±1% → Flat

### BTC/SOL Relative Strength

After classifying SOL's stage, compute relative strength:

```
SOL_rs = (SOL_price_now / SOL_price_30d_ago) / (BTC_price_now / BTC_price_30d_ago)
```

| SOL RS | Meaning | Effect |
|--------|---------|--------|
| > 1.05 | SOL outperforming BTC | Solana ecosystem health, bullish for altcoins |
| 0.95 - 1.05 | In-line with BTC | Neutral, follow SOL's own stage |
| < 0.95 | SOL underperforming BTC | Capital leaving Solana ecosystem, reduce exposure |

**When SOL underperforms BTC in a bull market:** This is a warning that the Solana ecosystem is losing momentum even while crypto broadly rises. Reduce altcoin exposure and concentrate in SOL itself if anything.

## Daily: Structure + EMA Alignment

From SOL daily candles (and individual token daily candles for watchlist assets):

**Three checks on daily:**

### 1. EMA Alignment (trend health)

Compute EMA21, EMA50, SMA200 on daily closes.

| Alignment | Meaning | Trading Bias |
|-----------|---------|-------------|
| EMA21 > EMA50 > SMA200 | Strong bull | Full allocation, buy pullbacks aggressively |
| EMA21 > EMA50, both < SMA200 | Bull within bear context | Cautious buys only, smaller positions, tighter stops |
| EMA21 < EMA50 < SMA200 | Strong bear | CASH. Do not buy. Wait for structure change. |
| EMA21 < EMA50, both > SMA200 | Pullback within bull | Potential buy zone IF structure holds (HLs intact) |

**For individual tokens (not SOL):** Many Solana tokens don't have 200 days of history. Use EMA21 vs EMA50 only. Meme coins that are < 30 days old: skip daily EMA analysis entirely — use 1H/4H only.

### 2. Structure: HH/HL vs LH/LL

Identify the last 3-4 swing highs and swing lows on daily candles.

- **Uptrend confirmed**: Each swing low is higher than the previous swing low AND each swing high is higher than previous
- **Downtrend confirmed**: Each swing high is lower AND each swing low is lower
- **Structure break warning**: Most recent swing low violated in uptrend = trend may be ending

**Pullback vs Reversal distinction:**
- Pullback: price retraces but HOLDS above the last swing low, volume declining during retrace → healthy, look for entry
- Reversal: price breaks below the last swing low, volume expanding on the break → trend change, exit positions

**Solana-specific:** Meme coins break structure frequently — a single whale dump can violate swing lows without invalidating the overall narrative. For meme coins, weight SM flow data more heavily than structure. For blue-chips (SOL, JUP, JTO, PYTH), structure analysis is reliable.

### 3. RSI(14) on Daily

Compute RSI(14) from daily closes:

```
RS = average_gain(14) / average_loss(14)
RSI = 100 - (100 / (1 + RS))
```

- RSI > 50: bull regime confirmed — buy signals are valid
- RSI < 50: bear regime confirmed — stay in cash or tight stops only
- RSI > 70: overbought — don't initiate NEW positions, but don't sell winners either
- RSI < 30: oversold — potential bounce coming, but in Stage 4 this is a TIMING signal, not a BUY signal

**RSI divergence — the most valuable early warning:**
- Price makes new high but RSI makes lower high → **bearish divergence** → take profits, tighten stops
- Price makes new low but RSI makes higher low → **bullish divergence** → watch for reversal entry, SM accumulation likely starting

RSI divergence on daily typically precedes a trend change by 1-2 weeks. When you see it, flag the asset as "regime change watch" and check SM flow data more frequently.

## 4H: Entry Zone Identification

From 1h candle data, group into 4H bars (4 consecutive 1h candles → one 4H bar: first open, max high, min low, last close, sum volume).

### Keltner Channel (EMA20 +/- 2x ATR)

Compute on 4H candles:
```
middle = EMA(close, 20)
ATR14 = ATR(14) on 4H bars
upper = middle + 2 × ATR14
lower = middle - 2 × ATR14
```

| Price Position | Meaning | Action |
|---------------|---------|--------|
| Above upper band | Strong trend breakout | Don't chase. Wait for retrace to middle band. |
| Near middle band | Pullback in uptrend | **PRIMARY ENTRY ZONE** — this is where you buy with the trend |
| Below lower band | Oversold / panic dump | Potential reversal zone. Look for Spring pattern (see 1H Triggers). |

**Why Keltner and not Bollinger:** Keltner uses ATR (true range) which captures gap moves and wicks better than standard deviation. On Solana where tokens can gap 20% on news, this matters.

### Key Levels That Matter

Not all support/resistance is equal. Prioritize these (in order of importance):

1. **Daily swing highs/lows** visible on the zoomed-out chart — structural levels where price reversed
2. **Breakout origin points** — the price level from which a strong pump started. Retests of breakout origins are high-probability entries.
3. **Round numbers** ($100 SOL, $1.00 JUP, $0.01 meme coin) — psychological levels with heavy limit orders
4. **EMA21/EMA50 on 4H** as dynamic support — trend-following entries
5. **Volume profile** — price levels where the most trading occurred = strongest S/R

### Volume Context (Solana-specific)

Unlike perps (which have OI/funding), Solana spot uses DEX volume as the primary context:

- **Price rising + volume rising** = genuine demand, trend healthy
- **Price rising + volume declining** = rally losing steam, fewer buyers at higher prices
- **Price falling + volume spike** = panic selling or whale dump, check SM data for context
- **Price falling + volume declining** = orderly selling, may be near exhaustion
- **Volume spike at support level** = potential accumulation (especially if SM dex-trades confirm)

## 1H: Entry Trigger Signals

The 1H timeframe ONLY answers "buy NOW or wait." It never determines direction — that's set by Weekly + Daily.

### High-Probability 1H Triggers (in order of reliability):

**1. Spring (Wyckoff) — highest win rate for spot**
- Price breaks below a key support level (from Layer 2) briefly — 1-3 candles
- Then SNAPS BACK above the support level
- Volume spikes on the break then subsides — weak hands were flushed
- This is a stop-hunt / shakeout. The "real" move is the snap-back up.
- Enter on the snap-back candle close. Stop just below the wick of the spring.
- **Solana context:** Meme coin springs happen frequently — a whale dumps, triggers cascading sells, then price recovers within minutes. SM dex-trades during the spring (an oracle buying the dip) = strongest confirmation.

**2. Reversal candle at key level**
- Bullish engulfing or hammer candle at EMA21/50 or structural support
- Volume spike on the reversal candle (buyers stepping in)
- Enter on close of reversal candle. Stop below the candle's wick.

**3. EMA21 bounce in uptrend**
- Price touches or slightly pierces 1H EMA21, then closes back above
- This works ONLY when 4H trend is aligned (EMA21 > EMA50)
- Enter on close. Stop = 2× ATR(14) on 1H below entry.

**4. RSI divergence + level confluence**
- 1H RSI divergence (price new low, RSI higher low) AT a key 4H support level
- Two independent signals pointing the same way = confluence
- Enter when 1H candle closes above the divergence trigger candle's high

**5. SM dex-trade at key level (Solana-specific)**
- An oracle wallet buys a token at or near a 4H support level
- This is unique to SoulHunter — real-time SM execution as a trigger
- Combine with any of triggers 1-4 for highest conviction

### What is NOT a trigger:
- A single news headline without price confirmation at a key level
- SM accumulating without price being at a relevant technical level
- "Token is down 50%, it must be cheap" — that's not a trigger, that's a falling knife
- Social media hype without SM flow confirmation

## ATR-Based Parameters (Adapted for Solana Spot)

For each asset, compute ATR(14) on 1H candles (or the timeframe you're trading):

| Use | Formula | Notes |
|-----|---------|-------|
| Stop-loss distance | 2.0 × ATR(14) below entry | For blue-chips (SOL, JUP, JTO) |
| Stop-loss distance (meme coins) | 2.5 × ATR(14) below entry | Meme coins have 15-20% daily ATR — need wider stops |
| BE stop trigger | Price must move ≥ 2.0 × ATR above entry | Don't move to BE too early — kills good trades |
| TP1 | 1.5× stop distance | Partial profit (sell 30-50%) |
| TP2 | 3× stop distance | Most of remaining position |
| TP3 (meme coins) | 5× stop distance or narrative exhaustion | Let winners run on meme coins |
| Trailing stop | 1.5 × ATR from peak | After TP1 hit, trail the rest |
| "Don't chase" filter | Price moved > 3× ATR in last 4H | If already pumped 3× ATR, wait for retrace |

**Meme coin ATR reality check:** A meme coin with $0.01 price and ATR of $0.002 (20% of price) needs a stop at $0.01 - 2.5×$0.002 = $0.005. That's a 50% stop — totally normal for meme coins. If 50% risk is too much for your position size, SIZE DOWN, don't tighten the stop. Tight stops on volatile assets = guaranteed stop-outs on noise.

## Regime Change Detection

The most valuable skill: identifying when SOL transitions between Weinstein stages.

**Early warnings (appear in order):**
1. **Momentum divergence** — RSI on daily makes lower high while SOL price makes higher high. This is the FIRST signal, often 1-2 weeks before price confirms.
2. **EMA convergence** — EMA21 and EMA50 gap narrowing after a long trend. The trend is losing steam.
3. **Failed new high** — SOL attempts new high but fails and closes back within range. Equal High or Lower High = distribution starting.
4. **Structure break** — the definitive confirmation. Last swing low broken in uptrend. After this, the trend has officially changed.
5. **EMA crossover** — the lagging confirmation. By the time EMA21 crosses below EMA50, the trend change is established. Use this to confirm, not to trigger.

**Ecosystem-level regime change signals:**
- Stablecoin outflows from Solana (check SM netflow for USDC/USDT)
- Number of tokens with positive SM netflow declining (breadth narrowing)
- SM wallets consolidating into SOL/stables (flight to quality within ecosystem)

**For the bot:** Check conditions 1-3 every research cycle. When any appear, flag as "regime change watch" and reduce position sizing by 30%. Condition 4 is the confirmation to shift to CASH/defensive mode.

## Trend Phase Assessment — Where Are We in the Cycle?

Knowing SOL is in Stage 2 (uptrend) is only half the picture. Knowing WHERE you are in Stage 2 determines HOW aggressively to buy.

**How to assess phase maturity:**

1. **Distance from the moving average** — how far has SOL risen above SMA30w?
   - < 10% above: early phase, trend building, ideal time to accumulate
   - 10-25% above: mid phase, trend established, buy pullbacks to EMA21/50
   - > 25% above: late phase, extended, reduce new entries, tighten stops on existing
   - > 40% above: euphoria zone, buying here is catching the top. Only hold existing winners with trailing stops.

2. **RSI context within the trend phase**
   - RSI 50-60 in Stage 2: healthy uptrend, plenty of room to run. Pullbacks to this RSI level are buy entries.
   - RSI 60-70 in Stage 2: trend is strong, momentum healthy. Buy pullbacks but with smaller size.
   - RSI > 70 in Stage 2: overbought. Don't buy new positions. Hold winners with trailing stops. Expect a pullback.

3. **Volume exhaustion** — is buying interest fading?
   - Volume expanding on up-moves = fresh buyers entering, trend has fuel
   - Volume declining on up-moves = fewer buyers at higher prices, fuel running low
   - Volume spiking on down-moves = profit-taking cascade, check if structure holds

**The critical question every cycle:** "Am I early enough to buy pullbacks, or am I late enough that I should only trail existing winners?"

An SM wallet that accumulated at $120 SOL can hold through a pullback to $150 because they're up 25%. You buying at $180 cannot absorb the same drawdown. Same trend, different phase, different strategy.

## Oversold in a Downtrend — WAIT, Don't Buy

The most common spot trading mistake: seeing SOL RSI < 30 and thinking "it's cheap, time to buy." In Stage 4, oversold means "the immediate drop is exhausted, expect a BOUNCE before more downside."

**How to think about oversold conditions in Stage 4:**

1. **Oversold is a TIMING input, not a BUY signal.** Direction is still set by the stage (Stage 4 = CASH). Oversold tells you a bounce is coming — that bounce is for sellers to exit, not for buyers to enter.

2. **The bounce targets** (where the dead-cat bounce runs out of steam):
   - 4H EMA21 (weak bounce, strong downtrend)
   - 4H EMA50 / Keltner middle (normal bounce)
   - Daily EMA21 (strong bounce in weakening downtrend)
   
   These are where any remaining longs should EXIT, not where new longs should ENTER.

3. **The exception — SM accumulation during Stage 4 oversold:** If you see Fund-labeled wallets using Jupiter DCA to accumulate a token during Stage 4 oversold conditions, they may be early for the Stage 1 (base) transition. This is a VERY long-term signal (weeks to months). You can start a tiny probe position (1-2% of capital) but understand you might sit in drawdown for weeks.

## Trading Style for Spot — Hold vs Swing

Since you can only buy (no shorts), the framework simplifies:

**Ask: "How much room is left in this uptrend, and can I survive a pullback?"**

- **Stage 2 early/mid + SM accumulating** → Trend hold. Buy pullbacks to EMA21/50, hold for weeks with trailing stops. Let winners compound.
- **Stage 2 late + overbought** → Swing/scalp. Quick entries at support, take profit at resistance. Don't hold overnight.
- **Stage 1/3 ranging** → Selective. Buy only the highest-conviction SM signals at range support. Size small.
- **Stage 4** → Cash. Period. The 15% max exposure rule exists for a reason.

**Capital size and phase interact:**
- $300 capital + Stage 2 early = accumulate aggressively (8-10% per position), can absorb pullbacks because positions are small
- $300 capital + Stage 2 late = small positions only, quick profits, protect capital
- $5K+ capital + Stage 2 early = full allocation across 3-5 high-conviction tokens

## Integration with SM Data

Trend analysis is the FRAMEWORK. SM data sits INSIDE this framework:

| Trend | SM Signal | Conviction | Action |
|-------|-----------|------------|--------|
| Stage 2 + SM accumulating | HIGH | BUY — full probe size, scale on confirmation |
| Stage 2 + SM silent | MEDIUM | BUY — reduced size, wider stops |
| Stage 2 + SM distributing | CAUTION | HOLD existing positions, no new entries |
| Stage 1/3 + SM accumulating | MEDIUM-LOW | Small probe only, wait for Stage 2 confirmation |
| Stage 1/3 + SM silent | LOW | WAIT — no edge |
| Stage 4 + SM accumulating | LOW | They can hold for months; you likely can't. Max 2% probe. |
| Stage 4 + SM silent/distributing | ZERO | CASH. No trades. |

The SM wallets ARE sophisticated trend traders themselves — their accumulation reflects their own analysis, expressed as capital allocation rather than chart markup. But they have more capital, longer horizons, and can absorb drawdowns you can't. Follow their DIRECTION, not their TIMING.
