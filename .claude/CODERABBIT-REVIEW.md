# CodeRabbit Review Findings

Reviewed: all source files, 2026-04-26.
Focus: correctness, race conditions, risk-management bypasses, LLM output trust, money-handling bugs.

---

## Critical — Fix before any real-money run

### C1 — README/code defaults disagree
README states 10% max position, 50% total exposure, 10% circuit breaker, 20% balance reserve.
Code defaults: 20%, 80%, 25%, 10% respectively.
**Fix:** Update README to match code, or set tighter values in `.env`.

### C2 — Balance reserve check uses withdrawable balance, not account_value
`risk_manager.py:229` checks `balance` (withdrawable USDC). With open positions consuming collateral, `balance` can be near zero while `account_value` is fine — silently disabling new trades.
**Fix:** Compare `account_value` against the reserve threshold, not withdrawable `balance`.

### C3 — Force-close uses market_open (not reduce-only) — can flip to opposite position
`main.py:142-146` calls `place_sell_order`/`place_buy_order` which call `market_open`. If TP/SL has partially filled since the snapshot, the close order is larger than the remaining position and opens a new opposite-side position.
**Fix:** Use reduce-only order for force-closes. Cancel TP/SL before closing. Re-fetch position size before placing the close.

### C4 — Implausibly large LLM allocations silently capped, no sanity reject
Any allocation from the LLM (even $10,000 on a $100 account) is silently capped at the `MAX_POSITION_PCT` limit. A stuck/broken prompt ships trades every cycle.
**Fix:** Reject (not just cap) allocations > 5× account_value, and log loudly.

### C5 — TP/SL direction not validated against current price
`validate_trade()` never checks that `tp_price` is on the correct side. A long with `tp_price < current_price` fires immediately at market, locking in a loss. Auto-SL falls back to `entry_price = 1.0` if current_price is 0 (HIP-3 lookup miss) — SL is then meaningless.
**Fix:** In `validate_trade`, reject `current_price <= 0`. Validate TP/SL sign relative to current_price; nullify and re-compute via `enforce_stop_loss` if wrong.

### C6 — Stale price used for sizing after LLM latency
`asset_prices[asset]` is cached before the LLM call. Sonnet-4 takes 30–120s. The cached price is then used for `amount = alloc_usd / current_price` and auto-SL computation. On volatile assets this is meaningfully stale.
**Fix:** Re-fetch `current_price` immediately before sizing and SL computation. Reject if stale or 0.

### C7 — Assets that failed data-gather can still pass the execution gate
`main.py:422` checks `asset in args.assets` but not `asset in asset_prices`. If data gather failed (price = missing), the LLM may still decide on that asset and the trade executor fails noisily.
**Fix:** Add `asset in asset_prices and asset_prices[asset] > 0` to the execution gate.

---

## High — Fix before scaling beyond testnet

### H1 — `get_user_state` drops negative PnL in fallback
`hyperliquid_api.py:396` uses `max(p.get("pnl", 0.0), 0.0)` — negative PnL is ignored. On a losing day, `account_value` reads higher than reality. Circuit breaker fires later than it should.
**Fix:** Remove the `max(..., 0)` — use signed PnL.

### H2 — Fill confirmation logic is not order-specific
After placing an order, bot checks recent fills for any fill on that coin in the last 1 second. Includes previous TP/SL fills. `filled: true` in diary is misleading.
**Fix:** Match fills by OID or by timestamp > order placement time.

### H3 — Force-close path lacks reduce-only (see C3)
TP/SL placed after entry correctly use `reduce_only=True` (6th arg to `Exchange.order`). Force-close path does not.

### H4 — Force-close recurs every loop for the same position
`check_losing_positions` returns any position over threshold every cycle. If the close order is slow to fill, a second close fires next cycle → double close → net short.
**Fix:** Track `force_close_in_progress` per asset for at least one cycle.

### H5 — Diary entry not written if TP/SL placement fails mid-execution
If the entry order succeeds but TP or SL placement throws, the outer `except Exception` catches it and no diary entry is written. Position on exchange, no local record.
**Fix:** Write diary entry immediately after entry order, before TP/SL placement. Append TP/SL OIDs separately.

### H6 — Tool-call branch uses deprecated `get_event_loop()` in async context
`decision_maker.py:173` — on Python 3.12+, `asyncio.get_event_loop()` in a non-main thread raises DeprecationWarning or fails.
**Fix:** Use `asyncio.new_event_loop()` explicitly in the ThreadPoolExecutor branch.

### H7 — Duplicate force-close orders per cycle
Related to H4. Each cycle that sees a losing position issues a new close. With no reduce-only, compounding closes.

---

## Medium

### M1 — `extract_oids` silently ignores error statuses
If TP order returns `{"error": "Insufficient margin"}`, `extract_oids` returns `[]`. Bot logs "TP placed" but it wasn't.
**Fix:** Check for `error` keys in statuses and log/raise.

### M2 — ADX series is one bar late (off-by-one)
`local_indicators.py:296` uses `[None] * (period * 2)` leading Nones; should be `period * 2 - 1`. Direction correct, timing one bar late vs TA-Lib.

### M3 — StochRSI padding calculation is fragile
`local_indicators.py:233-237` strip-and-re-pad may be off by 1-2 bars in edge cases.

### M4 — Cooldown is prompt-only, no machine enforcement
Bot tells Claude to self-impose 3-bar cooldown. Claude may ignore this on any cycle.
**Fix:** Track `last_action_time[asset]`; block non-hold within N intervals.

### M5 — `calculate_sharpe` always returns 0
`trade_log` entries don't have a `pnl` key; `calculate_sharpe` reads `r.get('pnl', 0)` → always 0.

### M6 — Log files never rotated
`prompts.log` and `llm_requests.log` are append-only. At 5-min intervals they'll grow to many MB/day.

### M7 — `MAX_POSITION_PCT` env var with `%` suffix crashes at startup
`float("10%")` raises. Document to omit the `%`, or strip non-numeric chars.

---

## Low

### L1 — README force-close wording misleading
"Force Close -20%" sounds like 20% of portfolio. Actually 20% of position notional. Clarify.

### L2 — `clear_terminal()` uses `os.system('clear')` — bad in Docker/log contexts
Replace with a print statement or remove entirely.

### L3 — Re-import of CONFIG inside `main_async` is redundant
`main.py:609` re-imports `CONFIG as CFG` when it's already available.

---

## Verdict

**Do not run with real money until C2, C3, C5, C6 are fixed.** The risk-manager wiring is conscientious (validate_trade runs before every entry), but the force-close path bypasses it and lacks reduce-only — meaning the guard intended to stop a losing position from bleeding can instead *open a new opposite-side position*. Combined with stale-price sizing and unvalidated TP/SL directions, a single bad cycle can lose 5–20% of the account before any human notices. Fix C2/C3/C5/C6, tighten `.env` to `MAX_POSITION_PCT=10`, `MAX_TOTAL_EXPOSURE_PCT=30`, then run testnet for 1–3 days.
