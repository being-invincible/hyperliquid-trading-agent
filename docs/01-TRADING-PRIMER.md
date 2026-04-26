# 01 — Trading Primer (For This Bot)

> Skip this if you already trade perpetual futures and know what funding, leverage, liquidation, TP, and SL are. Otherwise read it once — every other doc assumes these terms.

---

## The asset: a perpetual future ("perp")

When you trade BTC on Hyperliquid, **you are not buying actual Bitcoin**. You are entering a **perpetual futures contract** — an agreement whose value tracks Bitcoin's price.

Two things matter:

1. **Direction**: you can go **long** (bet price goes up) or **short** (bet price goes down). Owning real BTC only lets you bet up. Perps let you bet down too.
2. **Leverage**: you don't need to put up the full notional value. You post a small amount of **collateral** (USDC), and the exchange treats your position as if it were larger.

When you're done, you "close" the position. Profit/loss is paid in USDC. You never touch real BTC.

---

## Leverage — the part that hurts beginners

Leverage is just a multiplier on your position size relative to your collateral.

| Collateral | Leverage | Position size | 1% adverse move |
|------------|----------|---------------|-----------------|
| $100       | 1x       | $100          | -$1 (-1%)       |
| $100       | 5x       | $500          | -$5 (-5%)       |
| $100       | 10x      | $1,000        | -$10 (-10%)     |
| $100       | 20x      | $2,000        | -$20 (-20%)     |

Notice: with 10x leverage, a **10% move against you wipes out your collateral**. Crypto routinely moves 5–10% in a single day. This is why this bot has a **`MAX_LEVERAGE=10`** cap by default and a **mandatory stop-loss**.

> **What "wiped out" means**: when your collateral can no longer cover the loss, the exchange **liquidates** the position — closes it forcibly and keeps your collateral. With 10x, liquidation happens around -10% (slightly worse, due to fees and a maintenance margin buffer). With 20x, around -5%.

The bot's `allocation_usd` field is the **notional** position size (what you control), not the collateral. So an `allocation_usd: 500` on a $100 account is effectively 5x leverage.

---

## Long vs. Short

- **Long (buy)**: you profit if the price goes up. Lose if it goes down.
- **Short (sell)**: you profit if the price goes down. Lose if it goes up.

In this bot, the LLM picks `action: "buy"` (open long) or `action: "sell"` (open short). If you already have an opposite position, the order partially or fully closes it instead of opening a new one — Hyperliquid handles that automatically.

---

## Funding — the silent cost

Perpetual contracts have no expiry, but their price needs to stay close to the real ("spot") price. The mechanism that does this is called **funding**.

Every hour:

- If perp price > spot (lots of long demand): **longs pay shorts**.
- If perp price < spot (lots of short demand): **shorts pay longs**.

The rate is small (typically 0.001%–0.05% per hour) but it accrues continuously. If you hold a 10x long during a high-funding period for 24 hours at 0.02% per hour, you pay:

```
0.02% × 24 hours × 10 (leverage) = 4.8% on your collateral, in 24 hours, just for holding
```

**Why this matters for this bot**: the LLM is told to consider funding as a "tilt, not a trigger." It might not weigh funding heavily. If you trade in markets with extreme funding (often during squeezes), funding can quietly bleed you. Watch the `funding_annualized_pct` value the bot logs — anything above 50% annualized is high.

---

## Take-profit (TP) and stop-loss (SL) orders

Two safety/automation orders the bot places **after** opening a position:

- **Take-profit (TP)**: an order that closes the position when price reaches a *favorable* level. "Take my profit when BTC hits $76,000."
- **Stop-loss (SL)**: an order that closes the position when price reaches an *unfavorable* level. "Cut my losses if BTC falls to $69,000."

Both are **trigger orders** on Hyperliquid — they sit on the exchange and fire automatically. You don't need the bot running for them to execute.

Rules the bot enforces:

- BUY (long): `tp_price > current_price`, `sl_price < current_price`
- SELL (short): `tp_price < current_price`, `sl_price > current_price`

If the LLM doesn't set a stop-loss, the risk manager auto-sets one at **5% from entry** (configurable via `MANDATORY_SL_PCT`). There is no auto-TP — the LLM either sets one or doesn't.

> **Important caveat**: the bot's "force-close at 20% loss" guard (in `risk_manager.py`) is a *separate*, *additional* safety net. It uses a market order, which can fill significantly worse than the trigger price during fast markets.

---

## What "candle data" is

A "candle" or "OHLCV bar" is a summary of price activity over a time window. Each candle has:

- **O**pen — the price at the start of the window
- **H**igh — the highest price during the window
- **L**ow — the lowest price during the window
- **C**lose — the price at the end of the window
- **V**olume — how much was traded during the window

The bot pulls the last 100 candles at two timeframes:

- **5m candles** = each candle covers 5 minutes → 100 candles = ~8 hours of recent action.
- **4h candles** = each covers 4 hours → 100 candles = ~16 days of recent action.

These two timeframes feed the indicators. The 5m view is "what's happening now"; the 4h view is "what's the bigger trend."

---

## Order types

The bot can place two basic kinds:

- **Market order**: "buy at whatever the current price is." Fills immediately. The bot uses 1% slippage tolerance — if the price moves more than 1% between submitting and filling, the order is canceled.
- **Limit order**: "buy at $X or better." Sits on the exchange waiting. May never fill if price doesn't reach your limit.

The LLM chooses by setting `order_type: "market"` or `order_type: "limit"` (with `limit_price`). Default is market.

After the entry order, the bot **always** places TP and SL as **trigger orders** (if prices are provided) — these are reduce-only, meaning they can only close, not open, a position.

---

## What "exposure" and "notional" mean

- **Notional value**: the dollar value of what you control. A position of 0.01 BTC at BTC=$70,000 is **$700 notional**.
- **Exposure**: total notional across all your positions. The bot caps total exposure at 80% of your account value by default (`MAX_TOTAL_EXPOSURE_PCT=80`).

So with $100 account value, total open positions cannot exceed $80 in notional. With 10x leverage cap and ~$80 max notional, your effective leverage is bounded.

---

## What you can trade on Hyperliquid

Two categories:

1. **Native perps** — BTC, ETH, SOL, HYPE, AVAX, SUI, ARB, LINK, and 200+ more crypto perps. Symbol is just the ticker: `BTC`, `ETH`.
2. **HIP-3 perps** — traditional finance assets that trade through a separate "dex" namespace. Stocks (`xyz:TSLA`), commodities (`xyz:GOLD`, `xyz:OIL`), indices (`xyz:SP500`), forex (`xyz:EUR`). Note the `dex:asset` format with the colon.

The bot supports both. Just put the symbols in your `ASSETS` env var, e.g.:
```
ASSETS="BTC ETH SOL xyz:GOLD xyz:TSLA"
```

> Note: HIP-3 markets (especially stocks) are **closed when the underlying market is closed**. Trading TSLA at 3am is going to look weird and may have very thin liquidity. Stick to crypto for the first few weeks unless you know what you're doing.

---

## Where the money lives

When you deposit USDC on Hyperliquid, it sits in your **perp account** (or, in newer "unified" accounts, a USDC balance that perps can draw from). It is **not on your local machine**. The bot just sends signed messages to Hyperliquid telling it what to do.

If your machine loses internet, your positions stay open. The on-exchange TP/SL keep working. The bot just can't make new decisions until it reconnects.

---

## Summary cheat sheet

- **Long = buy = profit when price goes up.**
- **Short = sell = profit when price goes down.**
- **Leverage = position-size multiplier. 10x = 10% adverse move wipes out collateral.**
- **Funding = small recurring fee/payment between longs and shorts every hour.**
- **TP = order that closes profitably. SL = order that closes at a loss.**
- **Notional = dollar value of your position. Exposure = sum of notionals.**
- **Liquidation = exchange forcibly closes your position when collateral can't cover losses.**
- **Two wallets: agent (signs) and main (holds funds).**
