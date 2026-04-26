# Stability, Performance & Alerting — Design Spec

**Date:** 2026-04-27  
**Status:** Approved  
**Scope:** Three sequential milestones — M1 (stability), M2 (performance), M3 (email alerting)  
**Out of scope:** Intelligence improvements, full dashboard, architecture refactor — deferred to future specs

---

## Context

The agent has 6 active bugs that cause crashes on market orders and during tool calling, and a retired model string that will break all API calls on June 15 2026. Performance is bottlenecked by sequential candle fetches. There is no alerting when the agent loses money or force-closes a position.

This spec covers the minimal set of changes to make the agent production-stable, meaningfully faster, and observable for loss events — without touching trading logic or architecture.

---

## Milestone 1 — Stability (Bug Fixes)

**Workflow per fix:** commit → close the corresponding GitHub issue on `irfaad15-sys/hyperliquid-trading-agent` → note the issue URL.

### Fix 1 — `limit_price` NameError on market orders

**File:** `src/main.py` ~line 459  
**Issue:** #1

`limit_price` is only assigned inside `if order_type == "limit":` but referenced unconditionally in the diary write below. Every market order crashes with `NameError`.

```python
# Before (crashes on market orders)
if order_type == "limit":
    limit_price = output.get("limit_price")

# After
limit_price = None                          # initialize before the block
if order_type == "limit":
    limit_price = output.get("limit_price")
```

---

### Fix 2 — Retired model string

**File:** `src/config_loader.py` line 81  
**Issue:** #2

`claude-sonnet-4-20250514` retires June 15 2026. Any deployment without `LLM_MODEL` set in `.env` will fail all API calls after that date.

```python
# Before
"llm_model": _get_env("LLM_MODEL", "claude-sonnet-4-20250514"),

# After
"llm_model": _get_env("LLM_MODEL", "claude-sonnet-4-6"),
```

---

### Fix 3 — `asyncio.run()` inside running event loop

**File:** `src/agent/decision_maker.py` ~line 173  
**Issue:** #3

`_handle_tool_call()` calls `asyncio.run()` while already inside the running event loop. This raises `RuntimeError` and breaks tool calling entirely when `ENABLE_TOOL_CALLING=true`.

```python
# After — run in a fresh thread that owns its own event loop
import concurrent.futures

def _handle_tool_call(self, tool_name, tool_input):
    try:
        asyncio.get_running_loop()
        # We are inside a running loop — offload to a thread
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as executor:
            future = executor.submit(
                asyncio.run, self._execute_tool(tool_name, tool_input)
            )
            return future.result()
    except RuntimeError:
        # No running loop — safe to call directly
        return asyncio.run(self._execute_tool(tool_name, tool_input))
```

---

### Fix 4 — `state['balance']` KeyError

**File:** `src/main.py` ~line 110  
**Issue:** #4

Direct key access on external API data crashes if the key is ever absent.

```python
# Before
state['balance']

# After
balance = state.get('balance', 0.0)
if balance == 0.0:
    logging.warning("Account balance is 0 — API response may be incomplete: %s", state)
```

---

### Fix 5 — Duplicate `action` assignment

**File:** `src/main.py` ~line 424  
**Issue:** #6

The second `action = output["action"]` overwrites the safe `.get()` call and raises `KeyError` when the LLM omits the field.

```python
# Before
action = output.get("action")
...
action = output["action"]   # remove this line

# After
action = output.get("action", "hold")   # single safe assignment
```

---

### Fix 6 — Fills detection false positives

**File:** `src/main.py` ~line 474  
**Issue:** #5

The fills check matches by coin symbol only — any fill for the same asset is attributed to the current order. Add a 30-second timestamp window to narrow matches without requiring order ID tracking.

```python
import time
cutoff_ms = (time.time() - 30) * 1000   # 30 seconds ago in ms

order_filled = any(
    f.get("coin") == asset
    and (f.get("time") or f.get("timestamp") or 0) > cutoff_ms
    for f in recent_fills_struct
)
```

---

## Milestone 2 — Performance

### Change 1 — Concurrent candle fetches

**File:** `src/main.py` ~line 282  
**Issue:** #7

Replace sequential per-asset candle fetches with a two-level `asyncio.gather()`: both timeframes concurrently per asset, all assets concurrently.

```python
async def _fetch_asset_candles(hyperliquid, asset):
    candles_5m, candles_4h = await asyncio.gather(
        hyperliquid.get_candles(asset, "5m", 100),
        hyperliquid.get_candles(asset, "4h", 100),
    )
    return asset, candles_5m, candles_4h

# In the loop — replaces the for-loop sequential fetches
candle_results = await asyncio.gather(
    *[_fetch_asset_candles(hyperliquid, asset) for asset in args.assets]
)
candles_by_asset = {asset: (c5m, c4h) for asset, c5m, c4h in candle_results}
```

**Expected improvement:** ~4s → ~0.4s for 10 assets (10× speedup on data-fetch phase).

---

### Change 2 — `sort_keys=True` in context JSON

**File:** `src/main.py` ~line 338  
**Issue:** #9

Non-deterministic key ordering silently invalidates the prompt cache on every call. This is a prerequisite for Change 3.

```python
# Before
json.dumps(context_payload, default=json_default)

# After
json.dumps(context_payload, sort_keys=True, default=json_default)
```

---

### Change 3 — Claude prompt caching

**File:** `src/agent/decision_maker.py` ~line 32  
**Issue:** #10

Add `cache_control: {"type": "ephemeral"}` to the last static content block of the system prompt. The dynamic per-asset context is appended after the breakpoint and is never cached.

```python
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": system_prompt,
                "cache_control": {"type": "ephemeral"},   # cache this prefix
            }
        ],
    },
    {
        "role": "user",
        "content": dynamic_context_json,   # changes every loop — not cached
    },
]
```

**Requires:** `anthropic>=0.27` (already in `pyproject.toml`). No beta header needed for current SDK versions.

**Expected saving:** ~$200/year at 100 calls/day.

---

## Milestone 3 — Email Alerting

### New file: `src/notifications/emailer.py`

Self-contained module. Never imported by trading-critical paths. All calls are fire-and-forget wrapped in `try/except` — a failed email never halts the agent.

**Configuration** (add to `.env` and `config_loader.py`):

```
ALERT_EMAIL_FROM=you@gmail.com
ALERT_EMAIL_TO=you@gmail.com
ALERT_EMAIL_PASSWORD=xxxx-xxxx-xxxx-xxxx   # Gmail App Password
ALERT_SMTP_HOST=smtp.gmail.com             # optional, default smtp.gmail.com
ALERT_SMTP_PORT=587                        # optional, default 587
```

If any required variable (`FROM`, `TO`, `PASSWORD`) is missing, the emailer logs a warning at startup and all subsequent calls are no-ops. The agent never crashes due to email misconfiguration.

**Module interface:**

```python
class Emailer:
    def send_alert(self, subject: str, body: str) -> None:
        """Send an immediate [ALERT] email. Non-blocking, swallows all errors."""

    def maybe_send_digest(self, stats: dict) -> None:
        """Send daily [DAILY] digest if not yet sent today (UTC). No-op otherwise."""
```

**Immediate alert triggers** (added to `main.py`):

| Location | Event | Subject |
|---|---|---|
| Force-close block | Position force-closed by risk manager | `[ALERT] Force-close: {coin} -{loss_pct}%` |
| Circuit breaker check | Daily drawdown limit hit | `[ALERT] Circuit breaker active — no new trades` |
| Top-level loop `except` | Unhandled exception in trading loop | `[ALERT] Trading loop error: {error}` |

Alert body contains: asset, event type, PnL or loss %, account balance, UTC timestamp. Fits in a phone notification preview.

**Daily digest trigger:**

Called at the top of each loop iteration via `emailer.maybe_send_digest(stats)`. The emailer tracks `_last_digest_date`; if `datetime.now(UTC).date() != _last_digest_date`, it sends and updates the date. No cron, no scheduler.

Digest body:
```
Balance: $X,XXX.XX
Daily return: +/-X.XX%
Open positions: N
Trades today: N
Risk events today: none / [list]
```

**No new dependencies.** Uses `smtplib` and `email.mime` from stdlib only.

---

## Implementation Workflow

1. Each bug fix is a single commit referencing the issue number.
2. After committing, close the corresponding GitHub issue and note its URL.
3. M2 changes can be a single commit (`perf: concurrent fetches + prompt caching + sort_keys`).
4. M3 is a single commit (`feat: email alerter for loss events and daily digest`).

## Files Changed

| File | M1 | M2 | M3 |
|---|---|---|---|
| `src/main.py` | Fixes 1, 4, 5, 6 | Changes 1, 2 | Alert call sites |
| `src/config_loader.py` | Fix 2 | — | Email config vars |
| `src/agent/decision_maker.py` | Fix 3 | Change 3 | — |
| `src/notifications/emailer.py` | — | — | New file |

Total: ~130 lines changed or added across 4 files.
