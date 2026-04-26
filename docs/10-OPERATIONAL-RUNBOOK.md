# 10 — Operational Runbook

> How to monitor, restart, troubleshoot, and stop the bot. Covers daily operations once it's running.

---

## Starting the bot

```bash
# From the project root, with your .env configured:
python3 src/main.py

# Or explicitly:
python3 -m src.main

# Or override assets/interval at the command line:
python3 src/main.py --assets "BTC ETH SOL" --interval 15m
```

CLI args (`--assets`, `--interval`) take priority over `.env` values.

---

## Stopping the bot

```bash
Ctrl+C
```

The Python process exits cleanly. **Orders on the exchange are NOT cancelled.** Existing positions stay open. TP/SL triggers on Hyperliquid keep working without the bot.

To also cancel all open orders after stopping:
1. Go to [app.hyperliquid.xyz](https://app.hyperliquid.xyz) (or testnet equivalent)
2. Positions → Open Orders → Cancel All

---

## Running it in the background (so it keeps going when you close the terminal)

**Option A — screen** (simplest on Mac/Linux):

```bash
# Start a detached session named "bot"
screen -S bot
python3 src/main.py
# Detach: press Ctrl+A then D

# Reattach later:
screen -r bot

# List sessions:
screen -ls
```

**Option B — nohup**:

```bash
nohup python3 src/main.py > bot.log 2>&1 &
echo $! > bot.pid   # save the process ID

# Check if running:
cat bot.pid
ps aux | grep src.main

# Stop it:
kill $(cat bot.pid)
```

**Option C — Docker** (if you have Docker installed):

```bash
# Build the image
docker build -t hl-trading-bot .

# Run with your .env
docker run --env-file .env -p 3000:3000 hl-trading-bot

# Run in the background:
docker run -d --name hl-bot --env-file .env -p 3000:3000 hl-trading-bot

# View logs:
docker logs -f hl-bot

# Stop:
docker stop hl-bot
```

Note: the Dockerfile installs base dependencies but not `web3` or `rich` — you may need to update it. The `pip install` command in the readme form is safer for now.

---

## Monitoring the bot

### Live terminal output

The bot logs to stdout using Python's `logging` module. Every important event appears:

```
2025-10-19 12:00:01 - INFO - Combined prompt length: 4823 chars for 3 assets
2025-10-19 12:00:14 - INFO - Claude response: stop_reason=end_turn, usage=...
2025-10-19 12:00:14 - INFO - LLM reasoning summary: BTC in uptrend...
2025-10-19 12:00:14 - INFO - Decision rationale for BTC: 4h EMA20 above EMA50...
2025-10-19 12:00:14 - INFO - BUY BTC amount 0.000140 at ~71234.0
2025-10-19 12:00:14 - INFO - TP placed BTC at 73000.0
2025-10-19 12:00:14 - INFO - SL placed BTC at 69000.0
2025-10-19 12:15:14 - INFO - Hold ETH: Consolidating at resistance, awaiting breakout
2025-10-19 12:15:14 - INFO - Hold SOL: No confluence on 4h, staying flat
```

### Diary — what the bot decided

```bash
# Show last 10 entries, pretty-printed
tail -10 diary.jsonl | while read line; do echo "$line" | python3 -m json.tool; echo "---"; done

# Just show actions and assets
tail -20 diary.jsonl | python3 -c "
import sys, json
for line in sys.stdin:
    d = json.loads(line)
    print(f\"{d.get('timestamp','?')[:19]} | {d.get('asset','?'):6} | {d.get('action','?'):15} | {d.get('rationale','')[:60]}\")
"
```

### Decisions log — per-cycle summary

```bash
tail -5 decisions.jsonl | python3 -m json.tool
```

Shows: timestamp, cycle number, reasoning text, decision list, account_value.

### LLM request log — token usage and timing

```bash
tail -50 llm_requests.log
```

Shows model, message count, last message excerpt, stop_reason, token usage.

### HTTP API — while bot is running

```bash
# Last 20 diary entries as JSON
curl http://localhost:3000/diary?limit=20

# LLM request log (last 2000 chars)
curl http://localhost:3000/logs

# Full LLM log
curl "http://localhost:3000/logs?limit=all"

# Download diary for offline analysis
curl "http://localhost:3000/diary?download=1" -o diary.jsonl
```

### Hyperliquid app — what actually happened on-exchange

Always use the Hyperliquid web app to verify what the bot did:
- **Positions tab**: current open positions with P&L
- **Open Orders tab**: pending TP/SL trigger orders
- **Fills tab**: trade history (what actually filled and at what price)

The diary reflects what the bot *intended*; the exchange reflects what *actually happened*.

---

## Common issues and fixes

### "RISK BLOCKED: Balance ... below minimum reserve"

**Cause**: The `check_balance_reserve` bug (C2). The check uses `balance` (withdrawable margin) not `account_value`. When collateral is locked in positions, `balance` can be low even with a healthy account.

**Fix**: Lower `MIN_BALANCE_RESERVE_PCT` in your `.env` to `5` or `2`:
```bash
MIN_BALANCE_RESERVE_PCT=5
```
Then restart the bot.

---

### "RISK BLOCKED: Daily loss circuit breaker is active"

**Cause**: Account dropped more than `DAILY_LOSS_CIRCUIT_BREAKER_PCT` from today's peak.

**This is working as intended.** The bot won't place new trades until UTC midnight resets the circuit breaker.

**What to do**:
1. Check your positions. Are they OK or still bleeding?
2. If positions are in bad shape, close them manually on the Hyperliquid app.
3. Decide whether to adjust `DAILY_LOSS_CIRCUIT_BREAKER_PCT` (only raise it if you understand the implications).
4. The bot will resume at the next UTC midnight automatically.

---

### "Agent error: ... / Traceback" in the log

**Cause**: Claude API threw an error (usually network/timeout, occasionally model overload).

**The bot automatically retries once** per cycle. If both attempts fail, it skips that cycle and tries again at the next interval.

**If it persists**: check [status.anthropic.com](https://status.anthropic.com) for API outages.

---

### Bot placed a trade but I don't see a position on Hyperliquid

**Possible causes**:
1. The market order filled and the TP or SL immediately fired (closing it before you looked)
2. The order was rejected by the exchange (check `diary.jsonl` — `"order_result"` field may say "rejected")
3. Testnet issue — try refreshing the Hyperliquid app

**How to diagnose**:
```bash
# Check the diary entry for this trade
grep "BTC" diary.jsonl | tail -5 | python3 -m json.tool
# Look at the "order_result" and "filled" fields
```

---

### Bot is trading every cycle (excessive churn)

**Cause**: The LLM is ignoring the cooldown instruction (M7 — no machine-side enforcement). This wastes on API fees and slippage.

**Options**:
- Increase `INTERVAL` to `1h` or `4h` — fewer cycles = fewer decisions
- Reduce `ASSETS` to 1–2 assets — fewer decisions per cycle
- Phase 2 fix: add machine-side cooldown tracking

---

### "ZeroDivisionError" in traceback

**Cause**: `amount = alloc_usd / current_price` where `current_price` is 0 (a HIP-3 asset whose price lookup failed).

**Fix**: remove HIP-3 assets (`xyz:...`) from `ASSETS` for now and stick to native crypto perps.

---

### Log files growing very large

`prompts.log` and `llm_requests.log` are never rotated. After a week of 5-min intervals they can reach hundreds of MB.

**Manually trim**:
```bash
# Keep only last 10000 lines
tail -10000 prompts.log > prompts.log.tmp && mv prompts.log.tmp prompts.log
tail -10000 llm_requests.log > llm_requests.log.tmp && mv llm_requests.log.tmp llm_requests.log
```

---

### How to check if the bot is actually running

```bash
# Look for a running python process with src.main
ps aux | grep src.main

# Or if you saved the PID:
kill -0 $(cat bot.pid) 2>/dev/null && echo "Running" || echo "Not running"
```

---

## Emergency: manual position close

If the bot is misbehaving and you need to close all positions immediately:

1. **Stop the bot**: `Ctrl+C`
2. Go to [app.hyperliquid.xyz](https://app.hyperliquid.xyz)
3. Positions tab → **Close All** button (or close each individually)
4. Open Orders tab → **Cancel All** to remove pending TP/SL orders
5. Confirm all positions show $0 in the Positions tab

Do NOT restart the bot until you understand what went wrong.

---

## Useful log analysis commands

```bash
# How many trades today?
grep $(date +%Y-%m-%d) diary.jsonl | python3 -c "
import sys, json, collections
counts = collections.Counter()
for line in sys.stdin:
    d = json.loads(line)
    counts[d.get('action','')] += 1
print(dict(counts))
"

# List all force-closes
grep 'risk_force_close' diary.jsonl | python3 -m json.tool

# List all risk blocks
grep 'risk_blocked' diary.jsonl | python3 -c "
import sys, json
for line in sys.stdin:
    d = json.loads(line)
    print(f\"{d.get('timestamp','?')[:19]} | {d.get('asset','?')} | {d.get('reason','?')}\")
"

# P&L per trade (rough)
python3 -c "
import json
with open('diary.jsonl') as f:
    for line in f:
        d = json.loads(line)
        if d.get('action') in ('buy','sell'):
            alloc = d.get('allocation_usd', 0)
            tp = d.get('tp_price')
            sl = d.get('sl_price')
            entry = d.get('entry_price')
            if tp and sl and entry:
                tp_pct = abs(tp - entry) / entry * 100
                sl_pct = abs(sl - entry) / entry * 100
                print(f\"{d.get('asset')} {d.get('action')} | entry={entry} tp={tp}(+{tp_pct:.1f}%) sl={sl}(-{sl_pct:.1f}%) alloc=\${alloc}\")
"
```

---

## Keeping the bot healthy long-term

1. **Watch the diary daily.** Even a 2-minute scan catches issues early.
2. **Check Hyperliquid positions at least once a day.** The diary is bot intent; exchange is truth.
3. **Check Anthropic API costs weekly.** Set a spend limit in the console.
4. **Keep log files trimmed.** Rotate weekly.
5. **Don't leave it running unattended for more than 48 hours initially.** Graduate to longer unattended runs as you gain confidence.
6. **Back up your `.env`.** If you lose it, you lose your credentials.
