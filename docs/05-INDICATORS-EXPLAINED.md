# 05 — Indicators Explained

> Every indicator the bot computes, in trader-friendly terms, with what each is good and bad at. If you've never heard of EMA or RSI, start here.

---

## What is a "technical indicator"?

A formula applied to price (and sometimes volume) data that produces a number. The number is supposed to summarize some aspect of market behavior — trend, momentum, volatility, etc. Indicators are not magic; they are statistical lenses on the same OHLCV bars you can see on a chart.

The bot computes these from the **last 100 candles** at two timeframes:
- **5m** — short-term, "what's happening right now"
- **4h** — long-term, "what's the bigger trend"

All computation lives in [src/indicators/local_indicators.py](../src/indicators/local_indicators.py).

---

## EMA — Exponential Moving Average

**File**: [local_indicators.py:43–57](../src/indicators/local_indicators.py#L43-L57). Computed for periods **20** and **50**.

**What it is**: a moving average that weights recent prices more than older ones. Smoother than just looking at recent closes; less laggy than a simple moving average.

**How traders use it**:
- **Trend direction** — price above EMA20 = short-term uptrend; price above EMA50 = bigger uptrend.
- **EMA cross** — when EMA20 crosses *above* EMA50, often called a "golden cross" (bullish). Below = "death cross" (bearish).
- **Dynamic support/resistance** — price often pulls back to the EMA before continuing.

**How the bot uses it**: the system prompt specifically references "4h EMA20 vs EMA50" as a structure check. Claude tends to require alignment of EMAs across timeframes before flipping direction.

**Limitations**:
- Lags reality. By the time EMAs cross, the move is often half-done.
- Useless in choppy markets — gives constant whipsaws.
- Period is arbitrary. EMA20 vs EMA50 is convention, not gospel.

---

## RSI — Relative Strength Index

**File**: [local_indicators.py:64–95](../src/indicators/local_indicators.py#L64-L95). Computed for periods **7** and **14**.

**What it is**: a 0–100 oscillator that measures the *speed* of recent price changes. Internally: ratio of average gains to average losses over N periods, normalized.

**Conventional reads**:
- **RSI > 70**: "overbought" — price has risen fast, may pull back.
- **RSI < 30**: "oversold" — price has fallen fast, may bounce.
- **RSI = 50**: neutral.

**Common mistake the bot's prompt explicitly warns against**: treating RSI extremes as automatic reversal signals. The system prompt says:

> Overbought/oversold ≠ reversal by itself: Treat RSI extremes as risk-of-pullback. You need structure + momentum confirmation to bet against trend.

In a strong trend, RSI can sit at 80+ for days. Shorting just because RSI > 70 is a classic loser.

**Better uses**:
- **Divergence** — price makes a higher high while RSI makes a lower high → momentum is fading even though price isn't.
- **Slope** — RSI rising fast, regardless of level, = momentum building.
- **Failure swing** — RSI fails to break above 70 on a price rally → suggests trend exhaustion.

**RSI(7) vs RSI(14)**:
- 14 is the default. 7 is faster, noisier, gives earlier (and more frequent false) signals.

---

## MACD — Moving Average Convergence Divergence

**File**: [local_indicators.py:102–134](../src/indicators/local_indicators.py#L102-L134). Computed with default 12/26/9.

**What it is**: difference between EMA(12) and EMA(26), with an EMA(9) "signal line" smoothing.

Three series:
- **MACD line** = EMA(12) - EMA(26)
- **Signal line** = EMA(9) of MACD line
- **Histogram** = MACD - Signal

**Reads**:
- **MACD > 0**: shorter EMA above longer EMA → uptrend.
- **MACD crosses Signal upward**: momentum shifting bullish.
- **Histogram growing**: trend accelerating.
- **Histogram shrinking**: trend slowing.

**How the bot uses it**: the system prompt mentions "MACD regime" — the question of whether MACD is positive (bullish regime) or negative (bearish regime). The LLM is told to require both 4h EMA structure AND MACD regime alignment before flipping direction.

**Limitations**: Same as EMAs — lagging, choppy in ranges. The histogram is the most useful component for spotting momentum changes.

---

## ATR — Average True Range

**File**: [local_indicators.py:141–165](../src/indicators/local_indicators.py#L141-L165). Computed for periods **3** and **14**.

**What it is**: a volatility measure. Average of the "true range" (max of: high-low, |high - prev close|, |low - prev close|) over N bars. Output is in **price units**, not percent.

**Reads**:
- **High ATR**: market is volatile; expect bigger swings.
- **Low ATR**: market is calm; expect smaller swings.

**How the bot uses it**:
- **Position sizing** — the system prompt mentions "decisive break beyond ~0.5×ATR" as a confirmation threshold. ATR-based stops are a standard technique: SL = entry - 2×ATR.
- **Vol regime** — the prompt says "in high volatility (elevated ATR), reduce or avoid leverage."

**ATR(3) vs ATR(14)**:
- 3 = recent volatility burst.
- 14 = baseline volatility.

**Why it's important for risk**: a 1% adverse move means very different things on a low-vol vs high-vol day. ATR converts "how much is normal" into a number.

---

## Bollinger Bands

**File**: [local_indicators.py:172–195](../src/indicators/local_indicators.py#L172-L195). Default: period **20**, std dev **2.0**.

**What it is**: SMA(20) plus and minus 2 standard deviations of the last 20 closes. Three lines: upper, middle (SMA), lower.

**Reads**:
- **Price near upper band**: stretched up; may revert OR may be breaking out.
- **Price near lower band**: stretched down.
- **Bands narrow** ("squeeze"): low volatility, often precedes a big move.
- **Bands wide**: high volatility.

**How traders use it**: mean-reversion in ranges (sell at upper, buy at lower) and breakout signals when price punches through a band on volume.

**Limitation**: in trending markets, "stretched" can stay stretched for a long time.

---

## ADX — Average Directional Index

**File**: [local_indicators.py:246–307](../src/indicators/local_indicators.py#L246-L307). Default period **14**.

**What it is**: a 0–100 measure of **trend strength** (not direction). Built on top of +DI and -DI (positive/negative directional indicators).

**Reads**:
- **ADX > 25**: meaningful trend exists.
- **ADX > 40**: strong trend.
- **ADX < 20**: ranging, no trend.

**Important**: ADX says **whether** there is a trend, not which way. Combine with EMAs or MACD to get direction.

**How the bot uses it**: not heavily emphasized in the prompt, but available in the indicator dump. The LLM may use it as a "is this a trend or a chop?" check before applying trend-following logic.

> **Implementation note**: there's a flagged off-by-one issue (M2) in the bot's ADX calculation — the result series is shifted one bar late versus TA-Lib convention. Direction is correct; timing is one bar off. Not material for 5-min decisions but worth knowing.

---

## OBV — On-Balance Volume

**File**: [local_indicators.py:314–326](../src/indicators/local_indicators.py#L314-L326).

**What it is**: a cumulative sum where each bar's volume is added (close up) or subtracted (close down) from a running total.

**Reads**:
- **OBV rising while price flat or falling** → "smart money" accumulating; bullish divergence.
- **OBV falling while price flat or rising** → distribution; bearish divergence.

**Limitation**: the absolute number is meaningless (depends on history). Only the **slope and divergence** matter.

---

## VWAP — Volume Weighted Average Price

**File**: [local_indicators.py:333–346](../src/indicators/local_indicators.py#L333-L346).

**What it is**: cumulative average price weighted by volume from the start of the data window.

**Reads**:
- **Price above VWAP**: buyers in control on average.
- **Price below VWAP**: sellers in control.

VWAP is a standard institutional reference price ("are we filling above or below VWAP?"). Useful as a magnet — price often reverts to it.

**Bot's implementation note**: it's cumulative across the whole 100-bar window, not session-anchored. So this is "volume-weighted average since 8 hours ago" for 5m data, which is a slightly different metric than the daily VWAP traders quote on charts.

---

## Stochastic RSI

**File**: [local_indicators.py:202–239](../src/indicators/local_indicators.py#L202-L239). Default: rsi_period 14, stoch_period 14, k_smooth 3, d_smooth 3.

**What it is**: a stochastic oscillator applied **to the RSI series** instead of price. Two lines: %K (faster), %D (slower).

**Reads**:
- Same overbought/oversold logic as RSI but more sensitive.
- %K crossing %D up from oversold = bullish.
- %K crossing %D down from overbought = bearish.

**Use it as**: a noisy early warning. Often gives signals before RSI does, but with more false positives.

> **Implementation note**: padding logic is "tangled" (M3). Values may be 1–2 bars off in edge cases. Direction OK, exact timing approximate.

---

## What the bot puts in the prompt vs. what it computes

The bot computes everything in `compute_all()` but only sends a subset to Claude:

| Indicator | In 5m intraday section? | In 4h long-term section? |
|-----------|------------------------|--------------------------|
| EMA20     | latest + last 10 series | latest only             |
| EMA50     | —                       | latest only             |
| RSI7      | latest + series         | —                       |
| RSI14     | latest + series         | series                  |
| MACD      | latest + series         | series                  |
| ATR3      | —                       | latest                  |
| ATR14     | —                       | latest                  |
| Open Interest, Funding, Recent mid prices | yes (per asset, top-level) | — |

The full set (including BBands, ADX, OBV, Stoch RSI, VWAP) is **only available via tool calling** — Claude has to ask for it via `fetch_indicator`.

---

## What none of these indicators tell you

- **News**. A war breaks out, BTC drops 8%. No indicator predicted it.
- **Liquidations**. A whale's leveraged position gets liquidated; price moves 3% in 30 seconds. Indicators don't see the order book.
- **Funding squeeze setups**. Indicators don't know that funding is about to flip and force a deleveraging cascade.
- **Macro shifts**. Fed announcement, CPI print — these dominate price action and aren't in the indicator data.

This is one of several reasons indicator-based bots are **not magic profit machines**. They give the model a vocabulary; they do not give it foresight.

---

## How to think about the bot's reads

When you read the bot's diary entry and it says:

> "Decision rationale for BTC: 4h EMA20 above EMA50 with positive MACD, intraday RSI not extreme, ATR contained — long bias confirmed."

That's Claude doing pattern-matching against the indicator dump. It's plausible reasoning. **Plausible ≠ correct.** The market doesn't care that EMAs aligned — it cares about flows, news, and positioning, of which the bot can see almost nothing.

This is the honest limit of what's possible here. Trade accordingly.
