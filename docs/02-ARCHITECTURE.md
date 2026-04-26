# 02 — Architecture

> Subsystems, data flow, and what calls what. If you understand this, you understand the bot.

---

## Subsystems at a glance

```
.------------------.    .---------------------.    .---------------------.
|   Config Loader  |--->|   Trading Loop      |--->|   HTTP API Server   |
|  (.env -> dict)  |    |   (src/main.py)     |    |   :3000 /diary /logs|
'------------------'    '---------------------'    '---------------------'
                                  |
        .-------------------------+--------------------------.
        v                         v                          v
.-------------.     .------------------------.     .------------------.
| Hyperliquid |     |  Indicator Engine      |     |  Decision Engine |
|  API Client |     |  (local computation    |     |  (Claude API)    |
|             |     |   from candles)        |     |                  |
| - candles   |     |                        |     | - system prompt  |
| - state     |     | - EMA, RSI, MACD       |     | - tool calling   |
| - mids      |     | - ATR, BBands, ADX     |     | - JSON parsing   |
| - orders    |     | - OBV, VWAP            |     | - sanitizer LLM  |
| - fills     |     |                        |     |                  |
'-------------'     '------------------------'     '------------------'
        ^                                                   |
        |                                                   v
        |                                          .------------------.
        |                                          |  Risk Manager    |
        |                                          |  (hard-coded     |
        |                                          |   safety rules)  |
        |                                          '------------------'
        |                                                   |
        '---------------------------------------------------'
                         (validated trades placed)
```

Six logical pieces. Code-wise:

| Subsystem        | Files                                          |
|------------------|------------------------------------------------|
| Config Loader    | `src/config_loader.py`                         |
| Trading Loop     | `src/main.py`                                  |
| HTTP API Server  | `src/main.py` (the `handle_*` functions)       |
| Hyperliquid API  | `src/trading/hyperliquid_api.py`               |
| Indicator Engine | `src/indicators/local_indicators.py`           |
| Decision Engine  | `src/agent/decision_maker.py`                  |
| Risk Manager     | `src/risk_manager.py`                          |
| Utils            | `src/utils/formatting.py`, `src/utils/prompt_utils.py` |

---

## Data flow during one loop iteration

This is the most important diagram in these docs. Every 5 minutes, this happens:

```
                       [Loop tick]
                            |
                            v
     +------------------------------------------------+
     | 1. Fetch account state from Hyperliquid        |
     |    (balance, positions, unrealized PnL)        |
     +------------------------------------------------+
                            |
                            v
     +------------------------------------------------+
     | 2. RiskManager.check_losing_positions()        |
     |    -> close anything at >= 20% loss            |
     +------------------------------------------------+
                            |
                            v
     +------------------------------------------------+
     | 3. Fetch open orders + recent fills            |
     |    Reconcile local active_trades list with     |
     |    exchange truth (drop stale entries)         |
     +------------------------------------------------+
                            |
                            v
     +------------------------------------------------+
     | 4. For each asset:                             |
     |     - get_current_price(asset) (mid)           |
     |     - get_open_interest(asset)                 |
     |     - get_funding_rate(asset)                  |
     |     - get_candles(5m, 100)                     |
     |     - get_candles(4h, 100)                     |
     |     - compute_all(candles_5m)                  |
     |     - compute_all(candles_4h)                  |
     +------------------------------------------------+
                            |
                            v
     +------------------------------------------------+
     | 5. Build context_payload (one big JSON):       |
     |     - invocation metadata                      |
     |     - account dashboard                        |
     |     - risk_limits                              |
     |     - market_data (one entry per asset)        |
     |     - instructions                             |
     +------------------------------------------------+
                            |
                            v
     +------------------------------------------------+
     | 6. agent.decide_trade(assets, context)         |
     |     -> Claude API call                         |
     |     -> may call fetch_indicator tool (loop)    |
     |     -> JSON parse / sanitize fallback          |
     |     -> { reasoning, trade_decisions[] }        |
     +------------------------------------------------+
                            |
                            v
     +------------------------------------------------+
     | 7. (Optional) retry once if output invalid     |
     +------------------------------------------------+
                            |
                            v
     +------------------------------------------------+
     | 8. For each trade_decision:                    |
     |     if action in ("buy", "sell"):              |
     |       a. risk_mgr.validate_trade()             |
     |       b. if approved: place order              |
     |       c. place TP order (if tp_price)          |
     |       d. place SL order (if sl_price)          |
     |       e. append to active_trades               |
     |       f. write diary entry                     |
     |     else: log "hold"                           |
     +------------------------------------------------+
                            |
                            v
                  [sleep INTERVAL seconds]
                            |
                            v
                       [next tick]
```

---

## Key decisions about responsibilities

**Why are indicators computed locally instead of using TAAPI or another service?**
Originally the bot used TAAPI (a paid technical indicator API). It was replaced with local computation in `local_indicators.py` because:
1. No external dependency or API rate limit.
2. Works on every Hyperliquid market, including HIP-3 (stocks/commodities), not just crypto.
3. Free.

`taapi_client.py` is left in the repo but unused — it's marked as legacy in the README.

**Why does Claude (the LLM) only call tools optionally?**
The default is `ENABLE_TOOL_CALLING=False`. The reasoning: tool calling makes runs slower and more expensive, and the prompt already includes a rich indicator snapshot. Tools become useful only if Claude wants to drill into a specific timeframe or indicator the prompt didn't include.

**Why is there a "sanitizer" LLM?**
Sometimes Claude returns output wrapped in markdown fences or with extra prose. The sanitizer (`claude-haiku-4-5`, configurable) is a cheap, fast model that gets called *only* if the primary parse fails. Its job is to pull the JSON out of whatever Claude actually returned.

**Why is there a separate `active_trades` local list AND exchange queries?**
The exchange is the source of truth, but some local context (like which TP/SL OIDs go with which entry, and the LLM-supplied `exit_plan` text) only exists locally. The reconciliation logic (`main.py:200–214`) drops stale local entries when the exchange shows no position and no orders — but local state can still drift if you crash mid-loop.

**Why is force-close at 20% done before everything else?**
It's a panic exit. It runs without consulting the LLM so that even if the LLM is broken or hung, big losers still get closed. It's the bot's bottom-most safety net.

---

## Background subsystems

### HTTP API Server

`aiohttp` runs alongside the trading loop on port 3000 (configurable via `API_PORT`). Two endpoints:

- `GET /diary` — last N trade diary entries (default 200) as JSON. Add `?raw=1` for newline-delimited JSON, `?download=1` for download attachment.
- `GET /logs?path=<file>` — tail of a log file. Default `llm_requests.log`. Add `?limit=all` or `?download=1` for full content.

The dashboard mentioned in README (`dashboard/` directory) is **not in this repo**. It's described as a separate Next.js app. You can ignore it for now.

### Files written to disk

| File | Written by | Purpose |
|------|------------|---------|
| `diary.jsonl` | main loop | Every trade decision (and force-closes, blocks, holds). Append-only. |
| `prompts.log` | main loop | Full LLM prompt JSON, each cycle. Append-only. Grows large. |
| `llm_requests.log` | decision_maker | Truncated request/response metadata + token usage. |
| `decisions.jsonl` | main loop | Per-cycle decisions summary for dashboard consumption. |

These are written to **the working directory** where you run the bot, not a fixed path. If you `cd` somewhere else and run, the files end up there.

---

## What is *not* in this architecture

- No database. Everything is `.jsonl` files + in-memory state.
- No queue or job system. One loop, one process, one async event loop.
- No caching of LLM responses (each cycle is a fresh call).
- No rate limiting on the bot's side. Hyperliquid SDK retries with exponential backoff via `_retry()`.
- No process supervisor. If the bot crashes, it stays crashed.
- No notification/alerting system.
- No web UI for control. You start/stop with the shell.

---

## Lifecycle

```
$ python -m src.main --assets "BTC ETH" --interval 5m
   |
   |- load_dotenv()
   |- parse args, fall back to ASSETS / INTERVAL env
   |- HyperliquidAPI() loaded with private key
   |- TradingAgent() loaded with Claude client
   |- RiskManager() loaded with thresholds
   |
   |- asyncio.run(main_async)
   |    |- start aiohttp server on :3000
   |    |- run_loop()
   |        |- preload meta caches
   |        |- while True:
   |            |- fetch state, indicators, etc.
   |            |- LLM call
   |            |- execute trades
   |            |- sleep INTERVAL
```

When you `Ctrl-C`, asyncio cancels everything and the process exits. **In-flight orders that were already submitted to Hyperliquid stay on the exchange.** Only the *bot's* future decisions stop.
