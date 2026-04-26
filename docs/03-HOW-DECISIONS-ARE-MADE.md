# 03 — How Decisions Are Made

> The single most important doc. Walks through one loop iteration, end-to-end, with line references. If you only understand one thing about this bot, understand this.

---

## The short version

Every interval (5 minutes by default), the bot:

1. Reads exchange state.
2. Closes any position bleeding more than 20%.
3. Pulls market data and computes indicators.
4. **Asks Claude what to do**, with a long system prompt.
5. **Validates** Claude's answer against hard-coded risk rules.
6. If validated: places the trade + take-profit + stop-loss.

The "decision" is whatever Claude returns. There is no rule-based fallback. There is no statistical backtest. The "trading strategy" is whatever Claude reasons about in that moment based on the market data the bot puts in front of it.

---

## Step 1 — Account state ([main.py:108–131](../src/main.py#L108-L131))

```python
state = await hyperliquid.get_user_state()
total_value = state.get('total_value') or state['balance'] + sum(p.get('pnl', 0) for p in state['positions'])
```

Pulls:
- **Balance** — withdrawable USDC sitting as collateral.
- **Total value** — balance + unrealized PnL of all positions.
- **Positions** — list of open positions with size, entry price, leverage, liquidation price.

These come from Hyperliquid's `info.user_state` API. Then for each position, it queries the current price to compute the latest mid-based PnL.

> **Caveat**: there's a known issue here. `get_user_state()` in [hyperliquid_api.py:395–397](../src/trading/hyperliquid_api.py#L395-L397) has a fallback that does `sum(max(p.get("pnl", 0.0), 0.0) for p in enriched_positions)` — note the `max(..., 0)`. **It drops negative PnL.** On a losing day, total_value reads higher than reality. Flagged in [11-KNOWN-GAPS-AND-CAVEATS.md](11-KNOWN-GAPS-AND-CAVEATS.md) as **H1**.

---

## Step 2 — Force-close losers ([main.py:134–162](../src/main.py#L134-L162))

```python
positions_to_close = risk_mgr.check_losing_positions(state['positions'])
for ptc in positions_to_close:
    if is_long: await hyperliquid.place_sell_order(coin, size)
    else:       await hyperliquid.place_buy_order(coin, size)
    await hyperliquid.cancel_all_orders(coin)
```

[risk_manager.py:144–183](../src/risk_manager.py#L144-L183) computes per-position loss as `abs(pnl / notional) * 100`. If that exceeds `MAX_LOSS_PER_POSITION_PCT` (default 20), the position goes on the close list.

This runs **before the LLM is even called**. It's the panic exit. Even if Claude is hung, broken, or returning garbage, big losers get closed.

> **Critical caveat**: the close uses `place_sell_order` / `place_buy_order`, which under the hood is `market_open` — **not reduce-only**. If a TP or SL has partially filled between the snapshot and this call, the close order is sized larger than the remaining position and **opens an opposite-side position**. Flagged as **C3** in doc #11. This is the most dangerous bug in the system today.

---

## Step 3 — Reconcile open orders & local state ([main.py:174–214](../src/main.py#L174-L214))

The bot keeps a local `active_trades` list with `{asset, is_long, amount, entry_price, tp_oid, sl_oid, exit_plan}`. The exchange has its own truth. They drift.

The reconciliation:
- Pull all open orders from the exchange.
- For each entry in `active_trades`: if the asset has **no position and no orders**, drop it as stale and write a `reconcile_close` diary entry.

This is how the bot recovers from its own state diverging — for example, after a TP fills overnight, or after a process restart.

---

## Step 4 — Gather market data per asset ([main.py:268–321](../src/main.py#L268-L321))

For each asset in your `ASSETS` env var:

| Datum | API call | Purpose |
|-------|----------|---------|
| Mid price | `get_current_price` | Current price for sizing & TP/SL |
| Open interest | `get_open_interest` | How crowded the market is |
| Funding rate | `get_funding_rate` | Cost/payout for holding |
| 5m candles (×100) | `get_candles(asset, "5m", 100)` | Short-term trend & momentum |
| 4h candles (×100) | `get_candles(asset, "4h", 100)` | Big-picture trend |
| Indicators (5m & 4h) | `compute_all(candles)` | EMA, RSI, MACD, ATR, BBands, ADX, OBV, VWAP |

The mid price from this step also gets cached into `asset_prices[asset]` and used much later — at order-placement time — for sizing and SL computation. **By that point it's likely 30+ seconds stale** because the LLM call takes time. See **C6** in doc #11.

---

## Step 5 — Build the prompt ([main.py:323–341](../src/main.py#L323-L341))

The bot constructs a single big JSON payload with:

```python
{
  "invocation": { ... how many cycles, what time ... },
  "account":    { ... balance, positions, active trades, recent fills, open orders, recent diary ... },
  "risk_limits": { ... all the risk thresholds ... },
  "market_data": [ { ...per-asset indicators... }, ... ],
  "instructions": { "assets": [...], "requirement": "Decide actions for all assets..." }
}
```

This payload is logged to `prompts.log` and then sent to Claude as the user message.

---

## Step 6 — The LLM call ([decision_maker.py:25–384](../src/agent/decision_maker.py#L25-L384))

### The system prompt

Lives in [decision_maker.py:31–85](../src/agent/decision_maker.py#L31-L85). It tells Claude:

- You are a "rigorous quantitative trader."
- Pursue setups where calculated risk is outweighed by expected edge.
- **Hysteresis**: only flip direction if both 4h structure AND 5m structure agree.
- **Cooldown**: after opening, adding, reducing, or flipping, impose a 3-bar self-cooldown.
- Funding is a tilt, not a trigger.
- RSI extremes ≠ reversal — need confirmation.
- Use `fetch_indicator` tool if you need more.
- **Output format**: strict JSON with `reasoning` (long string) and `trade_decisions` (array, one per asset, each with `asset`, `action`, `allocation_usd`, `order_type`, `limit_price`, `tp_price`, `sl_price`, `exit_plan`, `rationale`).

> Important: the cooldown is **only enforced by Claude itself**. There is no machine-side check that ensures Claude actually waits 3 bars. If Claude flip-flops every cycle, the bot flip-flops every cycle. Flagged as **M7** in doc #11.

### The Claude call

```python
response = self.client.messages.create(
    model="claude-sonnet-4-20250514",
    system=system_prompt,
    messages=[{"role": "user", "content": context}],
    max_tokens=4096,
    tools=[fetch_indicator]  # only if ENABLE_TOOL_CALLING=True
)
```

- Default model: `claude-sonnet-4-20250514` (configurable via `LLM_MODEL`).
- Max output: 4096 tokens (16000 if extended thinking is on).
- **Extended thinking**: off by default (`THINKING_ENABLED=false`). When on, Claude does internal reasoning before answering, costing more tokens but generally improving quality.

### Tool calling loop

If `ENABLE_TOOL_CALLING=True`, Claude can call `fetch_indicator(indicator, asset, interval, period?)`. The bot loops up to 6 times handling tool calls before giving up.

By default this is off — the prompt already includes a rich indicator snapshot, so tool calling is rarely needed and adds latency.

### Parsing the answer

The bot strips ```` ``` ```` fences if present, then `json.loads()`. If parsing fails, it calls a **sanitizer LLM** (default `claude-haiku-4-5-20251001`) to extract clean JSON from whatever Claude actually said. If that also fails, it returns "hold" for everything with rationale="Parse error."

### What Claude actually returns (example)

```json
{
  "reasoning": "BTC 4h EMA20 above EMA50, MACD positive, momentum aligned. ETH similar. SOL diverging — RSI extreme. Holding SOL pending confirmation.",
  "trade_decisions": [
    {
      "asset": "BTC",
      "action": "buy",
      "allocation_usd": 50,
      "order_type": "market",
      "limit_price": null,
      "tp_price": 73500,
      "sl_price": 70200,
      "exit_plan": "Close if 4h close below EMA50 or stop hits",
      "rationale": "Higher-timeframe trend aligned with intraday breakout."
    },
    { "asset": "ETH", "action": "hold", "allocation_usd": 0, ... },
    { "asset": "SOL", "action": "hold", "allocation_usd": 0, ... }
  ]
}
```

---

## Step 7 — Retry on bad output ([main.py:343–388](../src/main.py#L343-L388))

If the parse fails or every decision is a parse-error hold, the bot retries **once** with a stricter "Return ONLY the JSON array" instruction. If that also fails, the cycle ends with no trades.

---

## Step 8 — Risk validation ([main.py:419–454](../src/main.py#L419-L454) → [risk_manager.py:189–274](../src/risk_manager.py#L189-L274))

For every decision with `action ∈ {buy, sell}`:

```python
allowed, reason, output = risk_mgr.validate_trade(
    output, state, initial_account_value
)
if not allowed:
    add_event(f"RISK BLOCKED {asset}: {reason}")
    # write to diary, skip this trade
    continue
```

`validate_trade` runs seven checks in order:

1. **Bump tiny allocations to $11** — Hyperliquid's $10 minimum.
2. **Daily drawdown circuit breaker** — if account dropped > 25% from today's high, block.
3. **Balance reserve** — if balance < 10% of initial, block. *(Bug flagged as C2: this uses `balance` not `account_value`, which can give false trips when collateral is locked in positions.)*
4. **Position size** — if alloc > 20% of account, **cap** (don't reject) at 20%.
5. **Total exposure** — if sum of positions + new alloc > 80% of account, block.
6. **Leverage** — if alloc/balance > 10x, block.
7. **Concurrent positions** — if already at 10, block.
8. **Mandatory stop-loss** — if `sl_price` is None, auto-set at 5% from entry.

> **Important caveat (C5)**: `validate_trade` does **not** verify `tp_price` and `sl_price` are on the right side of `current_price`. If Claude returns a long with `tp_price < current_price`, the bot places that as a take-profit trigger that fires immediately at market, locking in a loss. Same for SL on the wrong side.

---

## Step 9 — Place the trade ([main.py:457–512](../src/main.py#L457-L512))

```python
amount = alloc_usd / current_price  # current_price is potentially stale!

if order_type == "limit" and limit_price:
    order = await hyperliquid.place_limit_buy/sell(asset, amount, limit_price)
else:
    order = await hyperliquid.place_buy_order/place_sell_order(asset, amount)

if output.get("tp_price"):
    tp_order = await hyperliquid.place_take_profit(asset, is_buy, amount, tp_price)
if output.get("sl_price"):
    sl_order = await hyperliquid.place_stop_loss(asset, is_buy, amount, sl_price)
```

After 1 second, the bot checks recent fills to set a `filled` flag for the diary.

> **Caveat (C6)**: `current_price` here is the value cached at step 4, **before** the LLM call (which may have taken 30+ seconds for sonnet-4 with thinking). On a fast-moving asset, sizing is off by 1–2%, and the auto-SL is computed off stale price.

---

## Step 10 — Log to diary ([main.py:516–547](../src/main.py#L516-L547))

A JSON line is appended to `diary.jsonl` with everything: action, asset, alloc, amount, entry, TP, SL, OIDs, rationale, exit plan.

If the entry order succeeded but the TP/SL placement raised, the broad `except Exception` swallows it and **the diary entry is never written**. The position exists on the exchange with no local record. Flagged as **H5**.

---

## Step 11 — Sleep ([main.py:553](../src/main.py#L553))

```python
await asyncio.sleep(get_interval_seconds(args.interval))
```

Then the loop runs again.

---

## What this means in practice

- **The bot's "intelligence" is Claude.** Indicators are inputs; risk manager is a guardrail. The decision itself comes from a non-deterministic LLM.
- **Every cycle is independent.** Claude has no memory of past cycles except what the bot explicitly puts in the prompt (active trades, recent diary, recent fills). It cannot "learn" from past mistakes within a run; only the prompt context can carry information forward.
- **The risk manager is your real safety net.** The LLM is told risk limits, but it cannot enforce them. The risk manager *can*. When the risk manager has a bug (see C2, C3, C5, C6), there is no other line of defense.
- **The bot is reactive.** It has no cron-like idea of "wait until Friday's close to decide." It runs on a fixed interval.
- **Decisions are slow.** A typical Sonnet-4 call returns in 5–30 seconds. If you set INTERVAL too short (under 1 minute), cycles will overlap and chaos ensues.

---

## What changes if you turn on extended thinking?

Setting `THINKING_ENABLED=true` enables Claude's "extended thinking" mode (controlled via `THINKING_BUDGET_TOKENS`, default 10000). Claude does internal reasoning before answering. Effect:

- Decisions take **2–5× longer** (30s–2min per cycle).
- Output quality generally improves; reasoning is more rigorous.
- Costs more (you pay for thinking tokens).
- The stale-price problem (**C6**) is amplified.

For a $100 first-run we recommend **leaving it off**. Speed and predictability matter more than reasoning depth at this size.
