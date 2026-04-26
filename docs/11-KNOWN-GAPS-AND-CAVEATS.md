# 11 — Known Gaps and Caveats

> Read this before going live. Honest. Complete. No sugarcoating.

Code reviewed on 2026-04-26. Findings categorized as **Critical**, **High**, **Medium**, **Low**.

---

## The bottom line

**The 4 bugs you must fix before real money:**

| ID | Summary | Risk if unfixed |
|----|---------|-----------------|
| **C3** | Force-close opens opposite-side position | Losing position becomes new opposite trade at market, doubling exposure in worst case |
| **C5** | TP/SL not validated against current price direction | Wrong-side TP fires immediately at market on entry → instant loss |
| **C6** | Stale price used for sizing + SL after LLM latency | Trade sized on a price 30–120s old; auto-SL miscalculated |
| **C2** | Balance reserve check uses withdrawable balance, not account_value | Bot may silently block ALL new trades when collateral is in use |

These are real bugs in code that handles real money. Everything else is important but can wait for Phase 2.

---

## Structural risk: this bot has no proven edge

Before even getting to bugs — the fundamental truth about this bot:

**Claude is not a trading oracle.** It has no special insight into future prices. When it says "BTC 4h EMA20 above EMA50, positive MACD, going long" — that reasoning is coherent, plausible, and statistically meaningless as a predictor of BTC's next move.

Markets are extremely hard to predict. Professional hedge funds with teams of PhDs and proprietary data still frequently underperform. An LLM making decisions from public indicator data has no structural edge over a coin flip on any given trade.

**What this bot does well:**
- Applies consistent position sizing and stop-losses (when the risk manager is bug-free)
- Doesn't let emotion override rules
- Logs everything for analysis
- Acts as an excellent learning tool for understanding trading mechanics

**What it doesn't do:**
- Guarantee profit
- Backtest
- Adapt to market regime changes
- Access news, order book depth, on-chain data, sentiment

Run it for what it is — a learning experiment with automated risk management — not as an income stream.

---

## Critical bugs

### C3 — Force-close is not reduce-only (can flip to opposite position)

**Where**: [main.py:142–146](../src/main.py#L142-L146)

```python
if is_long:
    await hyperliquid.place_sell_order(coin, size)  # ← market_open, not reduce-only!
else:
    await hyperliquid.place_buy_order(coin, size)
```

`place_sell_order` calls `exchange.market_open(asset, False, amount, ...)`. This opens a new short position, not reduces an existing long.

**Scenario that breaks**: You have a 0.01 BTC long. A TP fires, reducing it to 0.005 BTC. The same cycle, the force-close fires using the original size (0.01 BTC). Result: you close 0.005 (remaining) + open 0.005 short = you're now short BTC into a rising market.

**Additional issue**: SL/TP cancellation happens *after* the close order, leaving a race window.

**Fix needed**: Use reduce-only close. SDK's `exchange.order` with `reduce_only=True`, or use `market_close` if available. Cancel TP/SL *before* placing the close. Re-fetch position size right before closing.

---

### C5 — TP/SL direction not validated; auto-SL uses stale/zero price

**Where**: [risk_manager.py:189–274](../src/risk_manager.py#L189-L274) (missing check) and [risk_manager.py:265–266](../src/risk_manager.py#L265-L266) (zero fallback)

The system prompt tells Claude:
> BUY: tp_price > current_price, sl_price < current_price

But `validate_trade` never checks this. If Claude hallucinates and returns a long with `tp_price = 65000` while current_price = `70000`, the bot places a trigger order that fires **immediately at market** when opened — locking in a guaranteed loss.

Same issue: if `current_price` is 0 (HIP-3 lookup miss → `asset_prices.get(asset, 0)` returns 0), then:
```python
entry_price = current_price if current_price > 0 else 1.0  # ← falls back to $1
sl_price = round(1.0 - (0.05 * 1.0), 2)  # = 0.95
```
For BTC, an SL at $0.95 will never fire. Your position has effectively no stop.

**Fix needed**: In `validate_trade`: reject if `current_price <= 0`. Then validate TP/SL signs (buy: tp > price and sl < price; sell: tp < price and sl > price). Nullify and re-compute via `enforce_stop_loss` if wrong.

---

### C6 — Price is stale by the time it's used for sizing

**Where**: [main.py:273](../src/main.py#L273) (price cached) → [main.py:455](../src/main.py#L455) (used 30–120s later)

```python
# Step 4 — gather phase, before LLM call:
current_price = await hyperliquid.get_current_price(asset)
asset_prices[asset] = current_price  # cached here

# ... LLM call (30–120 seconds later) ...

# Step 8 — execution:
current_price = asset_prices.get(asset, 0)  # stale!
amount = alloc_usd / current_price           # sized on stale price
```

With `claude-sonnet-4` taking 30s average and potentially 120s with thinking, during high volatility:
- A 1% BTC move ($700 on a $70k price) means your $10 allocation buys wrong amount
- The auto-SL is computed off stale price — could be placed on the wrong side (see C5)

**Fix needed**: Re-fetch `current_price` immediately before the sizing calculation and SL setup. Reject trade if price is 0 or fetch fails.

---

### C2 — Balance reserve check uses withdrawable balance, not account value

**Where**: [risk_manager.py:112–122](../src/risk_manager.py#L112-L122)

```python
def check_balance_reserve(self, balance: float, initial_balance: float):
    min_balance = initial_balance * (self.min_balance_reserve_pct / 100.0)
    if balance < min_balance:   # 'balance' is withdrawable margin
        return False, "..."
```

With `MIN_BALANCE_RESERVE_PCT=10` and `initial_balance=$100`, the reserve floor is $10 of **withdrawable** margin. Once you have any positions open, your collateral is locked and `withdrawable` drops. On a $100 account with $30 in positions, `withdrawable` could be $5–15 depending on leverage — tripping this check and blocking all new trades even though your account is healthy.

**Fix needed**: Either compare `account_value` (total portfolio value) against the reserve, or document this as "withdrawable margin floor" and set it to a low value like 2–5%.

**Workaround until fixed**: Set `MIN_BALANCE_RESERVE_PCT=5` or lower in your `.env`.

---

## High severity

### H1 — Total value drops positive-PnL only in fallback

**Where**: [hyperliquid_api.py:395–397](../src/trading/hyperliquid_api.py#L395-L397)

```python
total_value = balance + sum(max(p.get("pnl", 0.0), 0.0) for p in enriched_positions)
#                              ^^^— negative PnL stripped out
```

On a losing day, `account_value` reads higher than reality, so the daily drawdown circuit breaker fires later than intended. **Only the fallback path is affected** — if Hyperliquid returns a proper `accountValue` in the state response, this code doesn't run. But in unified account setups or specific SDK versions, it might.

**Fix**: Remove `max(..., 0.0)` → use signed PnL.

### H4 — Force-close repeats every cycle for the same position

If the force-close order is slow to fill (thin liquidity, HIP-3 market), `check_losing_positions` fires again next cycle and places a second close order. Combined with C3 (not reduce-only), this could result in a flipped position growing with each cycle.

**Fix**: Track `force_close_in_progress` per asset; skip if a close was attempted last cycle for the same asset.

### H5 — Diary entry not written if TP/SL placement fails

If the entry order succeeds but TP placement throws (e.g., network blip), the outer `except Exception` at [main.py:548](../src/main.py#L548) catches everything and no diary entry is written. The position exists on the exchange with no bot memory of it.

**Fix**: Write the diary entry immediately after the entry order succeeds, before TP/SL placement. Append TP/SL OIDs as a follow-up update.

---

## Medium severity

### M4 — Cooldown not machine-enforced (LLM can churn every cycle)

The system prompt tells Claude to self-impose a 3-bar cooldown after direction changes. There is no code that enforces this. A jittery market produces a jittery LLM which produces excessive trading — burning API fees and slippage.

**Real cost**: at 5-min intervals, a bot flipping BTC buy/sell every cycle pays ~0.05% per trade in fees × 288 trades/day = 14.4% daily fee churn. Your $100 is effectively losing ~$14/day to fees before any market loss.

**Fix**: Add `last_action_time[asset]` tracking in `main.py`; block direction changes within 3 intervals unless force-close triggers.

### M1 — TP/SL order errors silently ignored

If `exchange.order()` returns an error status for a TP placement, `extract_oids` returns `[]`, `tp_oid = None`, and the log says "TP placed BTC at 73000.0" — but the TP was never actually placed.

**Fix**: Check for `error` keys in the order response statuses and log/raise.

### M6 — Log files grow without bound

`prompts.log` and `llm_requests.log` are never rotated. A week of 5-min intervals with full context dumps = 500MB–1GB. Running out of disk space crashes the bot.

**Workaround**: trim manually (see [10-OPERATIONAL-RUNBOOK.md](10-OPERATIONAL-RUNBOOK.md)). Long-term fix: add `logging.handlers.RotatingFileHandler`.

---

## Low severity

### L1 — README defaults don't match code

The README is the first thing a user reads. It lists safety defaults that are significantly looser than what it states. See [04-RISK-MANAGEMENT.md](04-RISK-MANAGEMENT.md) for the full table.

**Fix**: update README to match `.env.example` and code defaults, or note that `.env.example` is the authoritative default source.

### L7 — `clear_terminal()` runs `os.system('clear')` on startup

Fine locally. In Docker or when collecting output with `nohup`, this creates a garbage escape code in the logs.

---

## Missing features (not bugs, but worth knowing)

1. **No notification/alert system** — the bot doesn't tell you when something important happens. You have to actively check.

2. **No kill switch beyond Ctrl+C** — there's no "stop trading, close all positions" command. You have to stop the bot and manually close on the exchange.

3. **No multi-instance protection** — running two copies of the bot simultaneously results in duplicate orders.

4. **No backtest** — there's no way to test a new strategy configuration against historical data before running it live.

5. **No Anthropic spend limit integration** — the bot doesn't know or care how much you're spending on Claude API. Set limits in the Anthropic console separately.

6. **No persistent state across restarts** — `active_trades` is in-memory only. A restart loses the bot's knowledge of what it intended. It recovers via reconciliation but won't know the original exit plan for pre-restart positions.

7. **No position size update** — if you want to add to an existing position or reduce it partially, the bot currently opens new positions or closes full ones.

---

## Should you run this with $100?

**After fixing C2, C3, C5, C6 — yes, as a learning experiment.**

The risk manager is architecturally sound; the bugs are fixable. With the conservative settings from [04-RISK-MANAGEMENT.md](04-RISK-MANAGEMENT.md) and after a clean testnet run, a $100 live run is an acceptable learning investment.

**Without fixing those 4 bugs — no.** The force-close bug alone can flip a losing position into a new opposite position, which can flip again next cycle, cascading losses. This is the scenario most likely to turn $100 into $10 in a single day.

The order of operations:
1. Run testnet for 1–3 days
2. Fix C3, C5, C6 (C2 can be worked around with low `MIN_BALANCE_RESERVE_PCT`)
3. Run another testnet day to verify fixes
4. Then go live with $100 at conservative settings
5. Watch it daily for 2 weeks
6. Evaluate and decide next steps
