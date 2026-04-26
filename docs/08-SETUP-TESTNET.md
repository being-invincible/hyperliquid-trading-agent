# 08 — Setup Guide: Testnet First

> Step-by-step from nothing to a running bot on Hyperliquid testnet. No real money at risk. Do this for 1–3 days before going live.

---

## What you'll have at the end

A Python process running on your machine that:
- Connects to Hyperliquid **testnet** (fake money, real mechanics)
- Calls Claude's API (real API calls, ~$1–5 for a few days of testing)
- Makes trade decisions every 15 minutes
- Writes logs you can inspect

---

## Prerequisites checklist

- [ ] A Mac, Linux, or Windows machine (Mac/Linux easier)
- [ ] Python 3.12 or newer — check with `python3 --version`
- [ ] An Anthropic account with an API key
- [ ] A MetaMask wallet (or any Ethereum-compatible wallet) — you'll create two wallet addresses

---

## Part 1 — Get your Anthropic API key

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Sign up / log in
3. Navigate to **API Keys** → **Create Key**
4. Copy the key (starts with `sk-ant-api03-...`) — you'll only see it once
5. **Set a spend limit**: in the console go to **Billing** → **Usage Limits** → set a monthly limit. For testing, $10/month is plenty.

Save this key — you'll paste it into `.env` shortly.

---

## Part 2 — Set up your Hyperliquid testnet wallets

You need **two** wallet addresses: a main wallet (holds the funds) and an agent wallet (signs trades). On testnet, both use fake funds.

### Step 2a — Get the testnet app

Go to [app.hyperliquid-testnet.xyz](https://app.hyperliquid-testnet.xyz) in your browser.

### Step 2b — Connect your main wallet

1. Click **Connect Wallet** in the top right
2. Connect MetaMask (or any EVM wallet)
3. This wallet is your **main wallet** — note its address (the `0x...` string from MetaMask)

> If you don't have MetaMask: [metamask.io](https://metamask.io) → Install extension → Create wallet → Back up the seed phrase securely.

### Step 2c — Get testnet USDC

1. On the testnet app, look for a **Faucet** button or link (usually top of page or sidebar)
2. Click it to receive testnet USDC (fake money)
3. Confirm the transaction in MetaMask
4. You should see a USDC balance appear after a moment

### Step 2d — Create an agent (API) wallet

This is a separate wallet that the bot will use to sign trades. Hyperliquid calls these "API Wallets."

Option A — Use MetaMask to create a second account:
1. In MetaMask, click your account icon → **+ Create Account**
2. Name it "Hyperliquid Agent"
3. **Export its private key**: click the three dots → **Account Details** → **Show Private Key**
4. Copy the `0x...` private key — **store it securely, never share it**

Option B — Let Hyperliquid generate one:
1. On the testnet app: Settings (gear icon) → **API → Create API Key**
2. The testnet will generate a wallet and show you its private key
3. Copy and save the private key

### Step 2e — Authorize the agent wallet

1. Still on [app.hyperliquid-testnet.xyz](https://app.hyperliquid-testnet.xyz)
2. Go to **Settings** → **API** (or **Wallets** depending on UI version)
3. Click **Authorize Agent** (or **Add API Wallet**)
4. Paste your agent wallet's **public address** (the `0x...` address, not the private key)
5. Confirm the transaction in MetaMask with your **main wallet**

Now your agent wallet is authorized to trade on behalf of your main wallet. The bot will use the agent wallet's private key to sign, and trades will affect the main wallet's balance.

---

## Part 3 — Install Python dependencies

```bash
# Check Python version (must be 3.12+)
python3 --version

# Navigate to the project
cd /path/to/hyperliquid-trading-agent

# Install dependencies directly (no poetry needed)
pip install \
  hyperliquid-python-sdk \
  anthropic \
  python-dotenv \
  aiohttp \
  requests \
  web3 \
  rich
```

If you get permission errors, try `pip install --user ...` or use a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate   # Mac/Linux
# or on Windows: .venv\Scripts\activate
pip install hyperliquid-python-sdk anthropic python-dotenv aiohttp requests web3 rich
```

---

## Part 4 — Configure your `.env` file

```bash
# From the project root:
cp .env.example .env
```

Open `.env` in any text editor and fill in:

```bash
# === REQUIRED ===
ANTHROPIC_API_KEY=sk-ant-api03-YOUR-KEY-HERE
HYPERLIQUID_PRIVATE_KEY=0xYOUR-AGENT-WALLET-PRIVATE-KEY
HYPERLIQUID_VAULT_ADDRESS=0xYOUR-MAIN-WALLET-ADDRESS

# === TRADING ===
ASSETS="BTC ETH SOL"
INTERVAL=15m
LLM_MODEL=claude-sonnet-4-6

# === TESTNET ===
HYPERLIQUID_NETWORK=testnet

# === CONSERVATIVE RISK ===
MAX_POSITION_PCT=10
MAX_LOSS_PER_POSITION_PCT=15
MAX_LEVERAGE=3
MAX_TOTAL_EXPOSURE_PCT=30
DAILY_LOSS_CIRCUIT_BREAKER_PCT=10
MANDATORY_SL_PCT=3
MAX_CONCURRENT_POSITIONS=3
MIN_BALANCE_RESERVE_PCT=5
```

**Triple-check**:
- `HYPERLIQUID_PRIVATE_KEY` = agent wallet's **private key** (the secret one)
- `HYPERLIQUID_VAULT_ADDRESS` = main wallet's **public address** (the visible one)
- `HYPERLIQUID_NETWORK=testnet` — this is critical for testnet. Wrong here = real money.

---

## Part 5 — First run

```bash
# From the project root (with venv active if you used one):
python3 src/main.py
```

Or using the module form:

```bash
python3 -m src.main
```

You should see output like:

```
Starting trading agent for assets: ['BTC', 'ETH', 'SOL'] at interval: 15m
2025-10-19 12:00:00 - INFO - Loaded HIP-3 meta for dex: xyz   (only if you have xyz: assets)
2025-10-19 12:00:01 - INFO - Combined prompt length: 4823 chars for 3 assets
2025-10-19 12:00:15 - INFO - Claude response: stop_reason=end_turn, usage=...
2025-10-19 12:00:15 - INFO - LLM reasoning summary: BTC shows uptrend alignment...
2025-10-19 12:00:15 - INFO - Decision rationale for BTC: ...
2025-10-19 12:00:15 - INFO - BUY BTC amount 0.000140 at ~71234.0
2025-10-19 12:00:15 - INFO - TP placed BTC at 73000.0
2025-10-19 12:00:15 - INFO - SL placed BTC at 69000.0
```

If you see errors:
- `Missing required environment variable: ANTHROPIC_API_KEY` → check your `.env` file
- `ValueError: Either HYPERLIQUID_PRIVATE_KEY...` → check the private key line in `.env`
- `RuntimeError: Hyperliquid retry: ...` → network issue or wrong key
- `ModuleNotFoundError` → run `pip install` again

---

## Part 6 — Verify it's working

### Check the Hyperliquid testnet UI

1. Go to [app.hyperliquid-testnet.xyz](https://app.hyperliquid-testnet.xyz)
2. Connect your **main** wallet
3. Check **Positions** tab — you should see positions appearing after each trade cycle
4. Check **Open Orders** — TP and SL orders should appear

### Check the local diary

```bash
# Watch trades in real time (run in a separate terminal)
tail -f diary.jsonl | python3 -c "import sys, json; [print(json.dumps(json.loads(l), indent=2)) for l in sys.stdin]"

# Or just view the last 5 entries
tail -5 diary.jsonl | python3 -m json.tool
```

### Check the HTTP API

While the bot is running, open a browser or terminal:

```bash
curl http://localhost:3000/diary | python3 -m json.tool
```

You should see recent trade diary entries.

### Check the LLM log

```bash
tail -100 llm_requests.log
```

Shows Claude API call metadata, token usage, and response stop_reason.

---

## Part 7 — What to observe for 1–3 days

Keep it running on testnet and watch for:

1. **Does it trade at all?** Check `diary.jsonl` for any `"action": "buy"` or `"action": "sell"` entries. If everything is `hold`, the risk manager might be blocking trades (check the log for "RISK BLOCKED").

2. **Does the risk manager fire?** Look in `diary.jsonl` for entries with `"action": "risk_blocked"` or `"action": "risk_force_close"`. These tell you which limits are triggering.

3. **Are TP/SL orders appearing on the exchange?** Go to Hyperliquid testnet → Open Orders. You should see trigger orders for each open position.

4. **Does the balance reserve bug (C2) block all trades?** If you see frequent `RISK BLOCKED: Balance...below minimum reserve` in the logs but your testnet account has funds, the bug is triggering. Fix: reduce `MIN_BALANCE_RESERVE_PCT` to 2 or 3.

5. **Does the LLM give sensible rationale?** Check `decisions.jsonl` — the `reasoning` field should describe market conditions coherently, not just say "parse error" or "tool loop cap."

6. **What does it cost?** After a day, check [console.anthropic.com](https://console.anthropic.com) → Usage. Multiply by 7 to project weekly cost.

---

## Part 8 — Stopping the bot

```bash
# In the terminal where it's running:
Ctrl+C
```

The bot stops cleanly. Any orders already placed on the exchange (TP/SL triggers) **remain active**. To cancel them:
1. Go to Hyperliquid (testnet) → Open Orders
2. Cancel manually, or wait for them to expire/fill

To start again:
```bash
python3 src/main.py
```

The bot will reconcile with the exchange state on the first loop.

---

## Testnet limitations to be aware of

- **Testnet prices are the same as mainnet** — Hyperliquid testnet tracks real prices. So testnet performance is somewhat realistic in terms of price moves.
- **Testnet liquidity may be slightly different** — fills might behave marginally differently.
- **Testnet is maintained by Hyperliquid** — it can be reset or go down occasionally.
- **Testnet performance ≠ mainnet performance** — because the LLM is non-deterministic. Even the same market conditions will produce different decisions on different runs.

---

## What "success" on testnet looks like

After 1–3 days on testnet you want to confirm:
- [ ] Bot runs without crashing for hours
- [ ] Orders appear on the exchange correctly (including TP/SL)
- [ ] Risk manager fires when it should (check against your limits)
- [ ] Diary shows coherent rationale from Claude
- [ ] No "force-close opened opposite position" weirdness in the logs
- [ ] API cost is within your projections

If all boxes are checked, proceed to [09-GOING-LIVE.md](09-GOING-LIVE.md).

If there are issues, check [10-OPERATIONAL-RUNBOOK.md](10-OPERATIONAL-RUNBOOK.md) for troubleshooting.
