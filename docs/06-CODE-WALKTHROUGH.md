# 06 — Code Walkthrough

> File-by-file tour of the codebase. Answers "what does this file do?" and "which lines do the important things?" Click any link to jump to that location in your editor.

---

## Project layout

```
hyperliquid-trading-agent/
├── CLAUDE.md                      ← collaboration guide (project memory)
├── README.md                      ← quickstart (but defaults are wrong — see doc 04)
├── pyproject.toml                 ← dependencies
├── .env.example                   ← template for your .env
├── Dockerfile                     ← Docker image (optional path to run)
├── src/
│   ├── main.py                    ← entry point + trading loop + HTTP API
│   ├── config_loader.py           ← reads .env into CONFIG dict
│   ├── risk_manager.py            ← all safety guards
│   ├── agent/
│   │   └── decision_maker.py      ← Claude API integration
│   ├── indicators/
│   │   ├── local_indicators.py    ← hand-rolled EMA/RSI/MACD/etc.
│   │   └── taapi_client.py        ← legacy/unused, keep for reference
│   ├── trading/
│   │   └── hyperliquid_api.py     ← Hyperliquid SDK wrapper
│   └── utils/
│       ├── formatting.py          ← number rounding helpers
│       └── prompt_utils.py        ← JSON serialization helpers
└── docs/                          ← all documentation
```

---

## `src/config_loader.py` — Configuration

**Lines**: 1–112 | [Open file](../src/config_loader.py)

**What it does**: loads environment variables (from `.env` or the shell) into a single `CONFIG` dict that every other module imports. All defaults live here.

**Key sections**:

- [Lines 10–68](../src/config_loader.py#L10-L68) — helper parsers: `_get_env`, `_get_bool`, `_get_int`, `_get_json`, `_get_list`. These convert raw strings to typed values.
- [Lines 71–111](../src/config_loader.py#L71-L111) — the `CONFIG` dict. Every key maps to one env var. If the env var is absent, a default string is returned.

**Key gotcha**: `_get_env("MAX_POSITION_PCT", "20")` returns the string `"20"`, not the integer 20. `RiskManager` receives this string and does `float(CONFIG.get(...))` — so it works, but if you put `MAX_POSITION_PCT=10%` (with percent sign) it crashes at float conversion.

**Nothing to change here normally.** Everything is driven by your `.env` file.

---

## `src/risk_manager.py` — Safety Guards

**Lines**: 1–289 | [Open file](../src/risk_manager.py)

**What it does**: enforces every hard-coded risk limit. The LLM cannot override these.

**Key methods**:

| Method | Lines | What it does |
|--------|-------|-------------|
| `__init__` | [18–31](../src/risk_manager.py#L18-L31) | Reads thresholds from CONFIG, initializes daily tracking |
| `_reset_daily_if_needed` | [33–42](../src/risk_manager.py#L33-L42) | Resets high-watermark at UTC midnight |
| `check_position_size` | [48–58](../src/risk_manager.py#L48-L58) | Single position > max% → return False |
| `check_total_exposure` | [60–75](../src/risk_manager.py#L60-L75) | All positions + new alloc > max% → return False |
| `check_leverage` | [77–86](../src/risk_manager.py#L77-L86) | alloc/balance > max_leverage → return False |
| `check_daily_drawdown` | [88–102](../src/risk_manager.py#L88-L102) | Account dropped > X% from today's high → block |
| `check_concurrent_positions` | [104–110](../src/risk_manager.py#L104-L110) | Count of open positions >= max → block |
| `check_balance_reserve` | [112–122](../src/risk_manager.py#L112-L122) | Balance < X% of initial → block (**C2 bug here**) |
| `enforce_stop_loss` | [128–138](../src/risk_manager.py#L128-L138) | Auto-set SL if missing |
| `check_losing_positions` | [144–183](../src/risk_manager.py#L144-L183) | Returns positions to force-close |
| `validate_trade` | [189–274](../src/risk_manager.py#L189-L274) | Runs all checks before entry order |
| `get_risk_summary` | [276–288](../src/risk_manager.py#L276-L288) | Returns thresholds as dict → goes into LLM prompt |

**Key flow in `validate_trade`** ([lines 189–274](../src/risk_manager.py#L189-L274)):

```python
if action == "hold": return True  # skip all checks
if alloc <= 0: return False
if alloc < 11: bump to 11         # HL minimum
check_daily_drawdown → reject if tripped
check_balance_reserve → reject if tripped    ← C2 bug
check_position_size → CAP (not reject) if over
check_total_exposure → reject if over
check_leverage → reject if over
check_concurrent_positions → reject if over
enforce_stop_loss → auto-set if missing
return True
```

**Note the asymmetry**: position-size excess → capped and continued. Everything else → rejected. This was an intentional design choice (to allow trades to proceed at smaller size) but means an LLM sending $10,000 allocations will silently trade at the cap every cycle.

---

## `src/trading/hyperliquid_api.py` — Exchange Client

**Lines**: 1–543 | [Open file](../src/trading/hyperliquid_api.py)

**What it does**: wraps the Hyperliquid Python SDK. All exchange communication goes through here.

**Initialization** ([lines 42–77](../src/trading/hyperliquid_api.py#L42-L77)):
- Loads private key or mnemonic from CONFIG
- Chooses testnet vs. mainnet base URL
- Sets `query_address` = vault address (main wallet) if present, else signer address
- Creates `Info` client (read) and `Exchange` client (write)

**Retry logic** ([lines 87–131](../src/trading/hyperliquid_api.py#L87-L131)):
`_retry(fn, *args, max_attempts=3, backoff_base=0.5)` — wraps any sync call in exponential backoff. Network errors cause a retry after 0.5s, 1s, 2s. Non-network errors get one retry then re-raise.

**Key trading methods**:

| Method | Lines | SDK call |
|--------|-------|----------|
| `place_buy_order` | [164–177](../src/trading/hyperliquid_api.py#L164-L177) | `exchange.market_open(asset, True, amount, None, slippage)` |
| `place_sell_order` | [178–191](../src/trading/hyperliquid_api.py#L178-L191) | `exchange.market_open(asset, False, amount, None, slippage)` |
| `place_limit_buy` | [192–207](../src/trading/hyperliquid_api.py#L192-L207) | `exchange.order(asset, True, amount, limit_price, {"limit": {"tif": "Gtc"}})` |
| `place_limit_sell` | [208–223](../src/trading/hyperliquid_api.py#L208-L223) | `exchange.order(asset, False, ...)` |
| `place_take_profit` | [225–240](../src/trading/hyperliquid_api.py#L225-L240) | `exchange.order(..., {"trigger": {..., "tpsl": "tp"}}, True)` ← reduce_only=True |
| `place_stop_loss` | [242–257](../src/trading/hyperliquid_api.py#L242-L257) | `exchange.order(..., {"trigger": {..., "tpsl": "sl"}}, True)` ← reduce_only=True |

**Important**: TP/SL both pass `True` as the 6th positional argument to `exchange.order` = `reduce_only=True`. Good. The force-close path in `main.py` does NOT — it uses `market_open` without reduce_only. This is **C3**.

**Data methods**:

| Method | Lines | Returns |
|--------|-------|---------|
| `get_user_state` | [353–397](../src/trading/hyperliquid_api.py#L353-L397) | balance, total_value, enriched positions |
| `get_current_price` | [399–421](../src/trading/hyperliquid_api.py#L399-L421) | mid price (float), 0.0 on miss |
| `get_meta_and_ctxs` | [422–444](../src/trading/hyperliquid_api.py#L422-L444) | cached market metadata |
| `get_open_interest` | [446–468](../src/trading/hyperliquid_api.py#L446-L468) | OI float or None |
| `get_candles` | [470–518](../src/trading/hyperliquid_api.py#L470-L518) | list of {t, open, high, low, close, volume} |
| `get_funding_rate` | [520–543](../src/trading/hyperliquid_api.py#L520-L543) | funding float or None |

**HIP-3 handling**: assets with `:` in the name (e.g. `xyz:GOLD`) use different API endpoints. `get_candles` branches at [line 494](../src/trading/hyperliquid_api.py#L494-L507) to use the raw `/info` post endpoint instead of `candles_snapshot`. `get_current_price` similarly branches at [line 413](../src/trading/hyperliquid_api.py#L413-L420).

**`round_size`** ([lines 133–162](../src/trading/hyperliquid_api.py#L133-L162)): looks up an asset's `szDecimals` from the cached meta and rounds the order amount to that precision. Without this, Hyperliquid rejects orders with too many decimals. This is why `get_meta_and_ctxs()` is pre-loaded at startup.

---

## `src/indicators/local_indicators.py` — Technical Indicators

**Lines**: 1–409 | [Open file](../src/indicators/local_indicators.py)

**What it does**: computes all technical indicators from raw OHLCV candle lists. Pure Python, no external dependencies.

**Individual functions**:

| Function | Lines | Indicator |
|----------|-------|-----------|
| `sma` | [32–39](../src/indicators/local_indicators.py#L32-L39) | Simple Moving Average |
| `ema` | [43–57](../src/indicators/local_indicators.py#L43-L57) | Exponential Moving Average |
| `rsi` | [64–95](../src/indicators/local_indicators.py#L64-L95) | RSI (Wilder smoothing) |
| `macd` | [102–134](../src/indicators/local_indicators.py#L102-L134) | MACD + signal + histogram |
| `atr` | [141–165](../src/indicators/local_indicators.py#L141-L165) | Average True Range |
| `bbands` | [172–195](../src/indicators/local_indicators.py#L172-L195) | Bollinger Bands |
| `stoch_rsi` | [202–239](../src/indicators/local_indicators.py#L202-L239) | Stochastic RSI (%K and %D) |
| `adx` | [246–307](../src/indicators/local_indicators.py#L246-L307) | ADX (trend strength) |
| `obv` | [314–326](../src/indicators/local_indicators.py#L314-L326) | On-Balance Volume |
| `vwap` | [333–346](../src/indicators/local_indicators.py#L333-L346) | Volume Weighted Average Price |
| `compute_all` | [353–394](../src/indicators/local_indicators.py#L353-L394) | Computes all above from one candle list |
| `last_n` | [397–400](../src/indicators/local_indicators.py#L397-L400) | Last N non-None values from a series |
| `latest` | [403–408](../src/indicators/local_indicators.py#L403-L408) | Last non-None value from a series |

**Common pattern**: each indicator function takes either a list of candle dicts or a list of floats, returns a list of the same length as input. Leading values that can't be computed (need more history) are `None`.

**`compute_all` returns a dict** with these keys:
`ema20`, `ema50`, `rsi7`, `rsi14`, `macd`, `macd_signal`, `macd_histogram`, `atr3`, `atr14`, `bbands_upper`, `bbands_middle`, `bbands_lower`, `adx`, `obv`, `vwap`.

**`last_n(series, n)` and `latest(series)`** strip out `None` values before operating — so you always get the last N actual values, not the last N including trailing Nones.

---

## `src/agent/decision_maker.py` — LLM Integration

**Lines**: 1–384 | [Open file](../src/agent/decision_maker.py)

**What it does**: calls Claude, handles tool calls, parses output, sanitizes if needed.

**Class `TradingAgent`** ([lines 15–384](../src/agent/decision_maker.py#L15-L384)):

- **`__init__`** ([18–23](../src/agent/decision_maker.py#L18-L23)) — creates `anthropic.Anthropic` client, sets model, sanitize_model, max_tokens.
- **`decide_trade(assets, context)`** ([25–27](../src/agent/decision_maker.py#L25-L27)) — public entry point, delegates to `_decide`.
- **`_decide(context, assets)`** ([29–383](../src/agent/decision_maker.py#L29-L383)) — main logic.

**Inside `_decide`**:

1. **System prompt** ([31–85](../src/agent/decision_maker.py#L31-L85)) — defines the trading persona, output format, and output contract. Full text in the prompt string.

2. **Tool definition** ([87–118](../src/agent/decision_maker.py#L87-L118)) — `fetch_indicator` schema. Only active when `ENABLE_TOOL_CALLING=True`.

3. **`_call_claude(msgs, use_tools=True)`** ([135–160](../src/agent/decision_maker.py#L135-L160)) — wraps `client.messages.create`. Logs to `llm_requests.log`. If `THINKING_ENABLED=True`, adds the thinking block and bumps max_tokens to 16000.

4. **`_handle_tool_call(tool_name, tool_input)`** ([162–228](../src/agent/decision_maker.py#L162-L228)) — executes `fetch_indicator`. Fetches candles from Hyperliquid, computes indicators, returns JSON string back to Claude. Note the threading workaround ([173–182](../src/agent/decision_maker.py#L173-L182)) — tool calls run in a sync context but need async API calls, so it uses a ThreadPoolExecutor.

5. **`_sanitize_output(raw_content, assets_list)`** ([230–258](../src/agent/decision_maker.py#L230-L258)) — fallback when Claude returns malformed output. Sends to `claude-haiku` which is cheaper and simpler.

6. **Main loop** ([261–383](../src/agent/decision_maker.py#L261-L383)) — iterates up to 6 times handling tool call / final text response. When Claude stops with `stop_reason == "end_turn"` (text response), parses JSON, normalizes with `setdefault` for missing keys, returns.

**What the loop does on parse failure**:
- Tries `_sanitize_output` → if successful, returns that.
- Returns `hold` with `rationale="Parse error"` for all assets.

**Important**: `_decide` is **synchronous** even though everything around it is async. It uses `asyncio.to_thread` inside `_handle_tool_call` to bridge. This is why `decide_trade` can be called with `agent.decide_trade(...)` (no await) from the async loop in `main.py`.

---

## `src/main.py` — Entry Point + Trading Loop

**Lines**: 1–659 | [Open file](../src/main.py)

This is the largest and most important file. It wires everything together.

**Top-level structure**:

| Lines | What |
|-------|------|
| [1–26](../src/main.py#L1-L26) | Imports, logging setup |
| [28–42](../src/main.py#L28-L42) | `clear_terminal()`, `get_interval_seconds()` |
| [44–654](../src/main.py#L44-L654) | `main()` — everything else |

**Inside `main()`**:

| Lines | What |
|-------|------|
| [44–71](../src/main.py#L44-L71) | Parse CLI args, instantiate HyperliquidAPI + TradingAgent + RiskManager |
| [72–88](../src/main.py#L72-L88) | State init: start_time, invocation_count, active_trades, diary_path, price_history |
| [89–553](../src/main.py#L89-L553) | `run_loop()` — the main loop |
| [555–598](../src/main.py#L555-L598) | HTTP endpoint handlers: `handle_diary`, `handle_logs` |
| [600–614](../src/main.py#L600-L614) | `start_api()`, `main_async()` — aiohttp setup |
| [616–652](../src/main.py#L616-L652) | Utility functions: `calculate_total_return`, `calculate_sharpe`, `check_exit_condition` |
| [654–658](../src/main.py#L654-L658) | `asyncio.run(main_async())` — entry |

**Key blocks inside `run_loop()`** ([89–553](../src/main.py#L89-L553)):

| Lines | Block |
|-------|-------|
| [93–101](../src/main.py#L93-L101) | Pre-load meta and HIP-3 meta caches |
| [104–116](../src/main.py#L104-L116) | Fetch account state, compute total_value, sharpe |
| [118–131](../src/main.py#L118-L131) | Enrich positions with current prices |
| [133–162](../src/main.py#L133-L162) | Force-close losing positions (C3 bug here) |
| [163–170](../src/main.py#L163-L170) | Load last 10 diary entries for context |
| [172–188](../src/main.py#L172-L188) | Fetch open orders, normalize trigger prices |
| [190–214](../src/main.py#L190-L214) | Reconcile active_trades vs. exchange truth |
| [215–241](../src/main.py#L215-L241) | Fetch recent fills, normalize timestamps |
| [243–266](../src/main.py#L243-L266) | Build dashboard dict for prompt |
| [268–321](../src/main.py#L268-L321) | Gather market data + indicators per asset |
| [323–341](../src/main.py#L323-L341) | Build context_payload JSON, log to prompts.log |
| [360–388](../src/main.py#L360-L388) | Call agent.decide_trade; retry once if failed |
| [390–416](../src/main.py#L390-L416) | Log reasoning, write decisions.jsonl |
| [418–552](../src/main.py#L418-L552) | Execute trades: risk check → order → TP/SL → diary |
| [553](../src/main.py#L553) | Sleep INTERVAL seconds |

**`active_trades` list**: in-memory only. Lost on restart. Each entry is a dict with:
```python
{
  "asset": "BTC",
  "is_long": True,
  "amount": 0.0007,
  "entry_price": 71234.0,
  "tp_oid": 12345,
  "sl_oid": 12346,
  "exit_plan": "Close if 4h close below EMA50",
  "opened_at": "2025-10-19T12:00:00"
}
```

This is the bot's memory of what it's done. It's reconciled against exchange state at the top of each loop. If the bot restarts, this list is empty — reconciliation re-discovers stale trades over the next cycle.

---

## `src/utils/formatting.py` and `prompt_utils.py`

**Tiny utility modules**. Nothing tricky.

- **`formatting.py`** ([full file](../src/utils/formatting.py)): `format_number(value, decimals=2)` and `format_size(value)`. Just wrappers around `round(float(value), n)` with exception safety.
- **`prompt_utils.py`** ([full file](../src/utils/prompt_utils.py)):
  - `json_default(obj)` — custom JSON serializer for datetime/set (used in `json.dumps(..., default=json_default)` when building the prompt).
  - `safe_float(value)` — cast to float or None.
  - `round_or_none(value, decimals)` — round or None.
  - `round_series(series, decimals)` — round each item in a list.

---

## What's not in the codebase

- `src/indicators/taapi_client.py` — the original TAAPI.io API client. **Unused.** Kept as reference. Safe to ignore.
- `dashboard/` — mentioned in README but **not in this repo**. The `/diary` and `/logs` API endpoints are the hook it would use.
- Tests — **there are none**. Testing is done by running on testnet.
- Database — **none**. Everything is flat `.jsonl` files and in-memory Python lists.
