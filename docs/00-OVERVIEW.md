# 00 — Overview: What Is This Bot?

> **Read this first.** Plain English. No code. By the end you should know what this software does, what it does not do, and roughly what risks you are taking.

---

## What this is, in one sentence

It is a Python program that, every few minutes, asks **Claude (an AI from Anthropic)** to look at the current crypto market and decide whether to **buy, sell, or hold** a list of assets on **Hyperliquid** — a decentralized exchange that lets you trade leveraged futures contracts. If Claude says "buy" or "sell," and a hard-coded set of safety checks agree, the program places that trade automatically using your money.

That's it. There is no secret algorithm. The "intelligence" is Claude reading market numbers and writing back a JSON object that says what to do.

---

## What it actually does, step by step

Every loop iteration (every 5 minutes by default):

1. **Looks at your account** — balance, what positions are open, what unrealized P&L (profit/loss) those positions have.
2. **Force-closes any losing position** that has lost more than 20% of its size. This is a safety net that runs *before* asking Claude anything.
3. **Pulls market data** — for each asset you told it to trade, it fetches recent candle data (price/volume bars) at two timeframes: 5-minute and 4-hour.
4. **Computes technical indicators** — EMA, RSI, MACD, ATR, Bollinger Bands, ADX, OBV, VWAP. (See [05-INDICATORS-EXPLAINED.md](05-INDICATORS-EXPLAINED.md) for what these mean.)
5. **Builds one big JSON message** that contains: your account state, your open trades, your active orders, recent fills, the indicators, the risk limits, and the list of assets to consider.
6. **Sends that message to Claude** with a long system prompt that tells Claude to act as a "rigorous quantitative trader."
7. **Reads Claude's reply** — Claude returns a JSON object with one entry per asset saying `buy` / `sell` / `hold`, how much money to risk (`allocation_usd`), the take-profit and stop-loss prices, and a written rationale.
8. **Checks each decision against safety rules** — position size cap, leverage cap, exposure cap, daily-loss circuit breaker, balance reserve, max concurrent positions. If any rule fails, the trade is rejected or shrunk.
9. **Places the trade** — a market or limit order. Then places a separate take-profit order and a separate stop-loss order on the exchange.
10. **Logs everything** to local files (`diary.jsonl`, `prompts.log`, `llm_requests.log`, `decisions.jsonl`).
11. **Sleeps until the next interval** and repeats.

There is also a small built-in HTTP server on port 3000 that serves `/diary` and `/logs` endpoints so you can monitor what it's doing.

---

## What it does *not* do

These things **are missing** and you should know it:

- **No backtesting.** The strategy has never been validated against historical data. Nobody has shown it makes money on past markets.
- **No proven edge.** It's an LLM. The LLM is good at sounding confident — that is not the same as being right about price direction.
- **No spend cap on the Claude API.** Each loop costs a few cents of API spend. Running 24/7 at 5-minute intervals = ~288 calls/day. You will spend money on Anthropic API fees independent of trading P&L.
- **No kill switch beyond the daily circuit breaker.** If the bot misbehaves, you stop it by killing the process.
- **No multi-instance safety.** If you accidentally run two copies, they will both place orders.
- **No alerts.** It doesn't text you, email you, or page you. You have to actively check it.
- **No gradual rollout.** It will trade your full configured size on the first cycle.

---

## What money is involved

There are three separate money costs:

1. **Trading capital** — the USDC you deposit on Hyperliquid. This is what gets traded. Can go up or down.
2. **Anthropic API fees** — every loop iteration calls Claude. At Sonnet 4 prices, expect roughly **$3–10 per day** depending on how many assets you trade and how long the prompt gets. This bill exists whether you make trading profit or not.
3. **Hyperliquid trading fees + funding** — Hyperliquid charges per-trade fees (maker/taker), and perpetual contracts pay/receive **funding** every hour. Funding can quietly eat profit if you hold positions on the wrong side of crowded trades.

So even if the bot is exactly break-even on trades, **you still lose money** to API fees and exchange fees. To break even *overall*, the bot has to outperform those costs.

---

## The two-wallet design

This trips up beginners. Hyperliquid trading uses **two wallets**:

- **Main wallet** — the wallet that *holds your USDC*. This is the one you deposit funds into. Its address goes in `HYPERLIQUID_VAULT_ADDRESS`.
- **Agent wallet** (also called API wallet) — a separate wallet whose private key the bot uses to *sign* trades on behalf of the main wallet. This is what goes in `HYPERLIQUID_PRIVATE_KEY`.

Why? Because if the agent wallet's private key leaks (e.g., a server gets hacked), the attacker can place trades but **cannot withdraw your funds**. Withdrawal still requires the main wallet's signature. This is a real, meaningful security boundary.

You set this up once on Hyperliquid's web UI. Walked through in [08-SETUP-TESTNET.md](08-SETUP-TESTNET.md).

---

## What "perpetual futures" means (very short)

Skip this section if you already know. Long version in [01-TRADING-PRIMER.md](01-TRADING-PRIMER.md).

A **perpetual futures contract** ("perp") lets you bet on an asset's price going up (`long`) or down (`short`) without actually owning the asset. You post **collateral** (USDC) and the exchange lets you control a position bigger than your collateral — that multiplier is **leverage**.

- **10x leverage** = $100 collateral controls a $1,000 position. A 1% move in your favor = +$10 (10% on your money). A 1% move against you = -$10.
- A **10% adverse move** with 10x leverage = your position is wiped out (**liquidated**). Hyperliquid takes your collateral.

That is why this bot's leverage cap and stop-loss matter. They are not optional.

---

## Who wrote this and is it audited?

The repo's `pyproject.toml` lists Gajesh Naik. The README explicitly says: *"Use at your own risk. No guarantee of returns. This code has not been audited."*

We have done a code review (see [11-KNOWN-GAPS-AND-CAVEATS.md](11-KNOWN-GAPS-AND-CAVEATS.md)). It is functional but has rough edges.

---

## The honest expectations setting

If you put $100 in and run this for a week:

- **Most likely outcome**: somewhere between -50% and +30% on the trading P&L, plus you'll spend ~$20–70 on API fees. Net result is unpredictable.
- **Best realistic outcome**: small profit. A profitable LLM trading bot is possible but not common and not proven for *this* bot.
- **Worst realistic outcome**: trading P&L hits the daily 25% drawdown circuit breaker on a bad day, then API fees keep ticking, then you stop the bot and walk away with $50–70.
- **Catastrophic outcome (low probability but real)**: a bug, a botched stop-loss, a leverage misconfiguration, or a network issue causes liquidation of multiple positions. With current default settings (max 80% total exposure, 10x leverage cap), this is contained but not impossible.

This is **not a savings account**. Treat the $100 as money you are willing to lose to learn how this works.

---

## How to read the rest of these docs

- **If you're new to trading**: read [01-TRADING-PRIMER.md](01-TRADING-PRIMER.md) next.
- **If you want to understand how it works under the hood**: read [02-ARCHITECTURE.md](02-ARCHITECTURE.md), [03-HOW-DECISIONS-ARE-MADE.md](03-HOW-DECISIONS-ARE-MADE.md), and [04-RISK-MANAGEMENT.md](04-RISK-MANAGEMENT.md).
- **If you just want to run it**: jump to [08-SETUP-TESTNET.md](08-SETUP-TESTNET.md), then [09-GOING-LIVE.md](09-GOING-LIVE.md).
- **Before going live**: you must read [11-KNOWN-GAPS-AND-CAVEATS.md](11-KNOWN-GAPS-AND-CAVEATS.md). That doc lists the bugs and design issues we found. Skipping it is the riskiest thing you can do.
