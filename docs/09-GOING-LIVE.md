# 09 — Going Live: $100 on Mainnet

> Only read this after completing [08-SETUP-TESTNET.md](08-SETUP-TESTNET.md) and observing the bot run cleanly on testnet for 1–3 days.

---

## Are you ready?

Before touching mainnet, check every item:

- [ ] Bot ran on testnet for at least 24 hours without crashing
- [ ] Orders appeared correctly on testnet exchange (positions + TP/SL)
- [ ] You read [11-KNOWN-GAPS-AND-CAVEATS.md](11-KNOWN-GAPS-AND-CAVEATS.md) and understand the open bugs
- [ ] You have decided to accept: the bot may lose the entire $100
- [ ] You have set an Anthropic spend cap (API fees are real, win or lose)
- [ ] You know how to stop the bot immediately (`Ctrl+C`) and cancel open orders manually

**Not ready?** That's fine — run testnet longer. There is no rush.

---

## Part 1 — Fund your mainnet wallet

### Step 1 — Get a mainnet account on Hyperliquid

1. Go to [app.hyperliquid.xyz](https://app.hyperliquid.xyz) (note: not testnet)
2. Connect your MetaMask wallet (this becomes your main wallet)

### Step 2 — Deposit USDC

USDC needs to be on the **Arbitrum One** network to deposit into Hyperliquid.

**If you have USDC on Arbitrum already:**
1. On the Hyperliquid app, click **Deposit**
2. Enter amount ($100), confirm the MetaMask transaction
3. Wait ~30 seconds for it to appear as available balance

**If you don't have USDC on Arbitrum:**
You'll need to bridge from another chain. Options:
- Buy USDC directly on Arbitrum via Coinbase, Kraken, etc. and withdraw to your MetaMask address on Arbitrum network
- Bridge from Ethereum mainnet using the [Arbitrum Bridge](https://bridge.arbitrum.io/) — takes ~20 min and costs ETH gas
- Use a cross-chain service (Circle CCTP, Stargate, etc.)

> Start with exactly $100 or slightly less ($90–95). Keep $5–10 in your wallet for gas fees on future transactions.

### Step 3 — Confirm deposit visible on Hyperliquid

Log in to [app.hyperliquid.xyz](https://app.hyperliquid.xyz), connect your wallet. You should see your USDC balance in the portfolio section.

---

## Part 2 — Set up your mainnet agent wallet

You need a **separate** agent wallet for mainnet (don't reuse your testnet one — keep credentials separate).

**Option A — Create in MetaMask** (recommended):
1. MetaMask → account icon → **+ Create Account** → name it "HL Mainnet Agent"
2. **Export its private key**: three dots → Account Details → Show Private Key
3. Store securely — use a password manager

**Option B — Create via Hyperliquid app**:
1. App → Settings → API → Create API Wallet
2. Save the generated private key

**Authorize the agent wallet**:
1. Go to [app.hyperliquid.xyz](https://app.hyperliquid.xyz) → Settings → API
2. Click **Authorize Agent** / **Add API Wallet**
3. Enter your agent wallet's **public address**
4. Confirm with your main wallet in MetaMask

---

## Part 3 — Update `.env` for mainnet

Create a **separate** `.env` file or backup your testnet one first:

```bash
cp .env .env.testnet.backup
```

Then update `.env`:

```diff
- HYPERLIQUID_NETWORK=testnet
+ HYPERLIQUID_NETWORK=mainnet

- HYPERLIQUID_PRIVATE_KEY=0xYOUR-TESTNET-AGENT-KEY
+ HYPERLIQUID_PRIVATE_KEY=0xYOUR-MAINNET-AGENT-KEY

- HYPERLIQUID_VAULT_ADDRESS=0xYOUR-TESTNET-MAIN-WALLET
+ HYPERLIQUID_VAULT_ADDRESS=0xYOUR-MAINNET-MAIN-WALLET
```

Everything else stays the same. **Double-check `HYPERLIQUID_NETWORK=mainnet`** — this is the critical line.

---

## Part 4 — Conservative settings for first real-money run

These settings are designed for a $100 account where preserving capital matters more than maximizing profit:

```bash
# Trading
ASSETS="BTC ETH"              # only 2 assets — simplest to monitor
INTERVAL=15m                   # 15 min — moderate activity, ~$1.50-4/day API cost

# Conservative risk (do not loosen until you've run 1 week)
MAX_POSITION_PCT=10            # max $10 per trade
MAX_LOSS_PER_POSITION_PCT=15   # force-close at 15% position loss
MAX_LEVERAGE=3                 # max 3x leverage
MAX_TOTAL_EXPOSURE_PCT=30      # max $30 total notional
DAILY_LOSS_CIRCUIT_BREAKER_PCT=10   # stop after $10 daily loss
MANDATORY_SL_PCT=3             # auto-SL at 3% from entry
MAX_CONCURRENT_POSITIONS=2     # max 2 positions at once
MIN_BALANCE_RESERVE_PCT=5      # low (due to C2 bug)
```

**Why BTC and ETH only?**
- Most liquid on Hyperliquid — tighter spreads, more reliable fills
- More predictable (relatively) than altcoins
- Easier to monitor manually
- If one goes wrong, you're only tracking two things

---

## Part 5 — First mainnet run

```bash
# With venv active:
python3 src/main.py
```

**The first 30 minutes**:
- Watch the terminal output closely
- After the first cycle, go to [app.hyperliquid.xyz](https://app.hyperliquid.xyz) and check your account
- You should see your USDC balance unchanged if Claude returned `hold` for everything, or slightly different if a trade was placed

**The first 24 hours**:
- Check the Hyperliquid app every few hours
- Check `diary.jsonl` before bed
- Check your Anthropic API usage

---

## Part 6 — What to watch in the first week

### Daily routine

1. **Morning** — Open `diary.jsonl` or run `curl localhost:3000/diary`. How many trades yesterday? Any force-closes? Any risk blocks?

2. **Evening** — Check Hyperliquid positions. Are open positions still in healthy territory (not near liquidation)? How's the P&L?

3. **Weekly** — Check Anthropic API cost. Is it within your budget?

### Warning signs to act on immediately

| Sign | What to do |
|------|------------|
| Position P&L past -15% | The force-close should fire. If it doesn't, manually close. |
| Multiple concurrent positions with large losses | Stop the bot (`Ctrl+C`), cancel all orders on Hyperliquid app, close positions. |
| Bot output: "Traceback" errors repeating | Check the error. Often a network issue. Restart if it keeps happening. |
| Diary shows "risk_force_close" repeatedly | Risk limits are right at the edge — review your config. |
| Diary shows "RISK BLOCKED" on every cycle | Circuit breaker is active or balance reserve bug. Check the reason. |
| Anthropic API costs > projected | Either more assets, thinking mode accidentally on, or unusually long prompts. Check `llm_requests.log`. |

### When to stop the bot

Stop it (`Ctrl+C`) immediately if:
- Account value is down more than 30% (from $100 to $70)
- You see positions going wildly wrong (liquidation approaching)
- The bot is placing trades that don't make sense (wrong assets, wrong direction, huge sizes)
- You're not able to monitor it for the next 24+ hours

Always manually check Hyperliquid open orders after stopping — cancel any resting TP/SL if you're pausing for a while.

---

## Part 7 — Scaling up (after 2+ weeks)

Only after you've run $100 for 2 weeks and:
- Have a reasonable understanding of how it behaves
- Haven't lost more than 30–40% of the capital
- Have fixed the critical bugs (C3, C5, C6 at minimum)
- Trust the system enough to want to put more in

**Scaling approach**:
1. Add $100 more (total $200)
2. Optionally add one more asset (e.g., SOL)
3. Watch for another week
4. Scale again if comfortable

**Never** put in money you can't afford to lose. The expected value of this bot is unknown — it may make money, it may not.

---

## Reverting to testnet

If you want to stop real trading and go back to testnet:

```bash
cp .env .env.mainnet.backup
cp .env.testnet.backup .env   # if you saved one
# or just change HYPERLIQUID_NETWORK=testnet in .env
```

Your positions on mainnet stay open until they hit TP/SL or you close them manually on the Hyperliquid app. **The bot being stopped does not close positions.**

---

## What "profit" actually looks like

With $100 and these conservative settings:

- Max position: $10 at 3x = $30 notional exposure
- On a good run: BTC moves 3%, bot caught it → +$0.90 (9% on the $10 position)
- On a bad run: BTC drops 3%, SL fires at -3% → -$0.30 (3% on the $10 position)

You can see why $100 is a learning exercise, not an income stream. The percentage returns can be significant; the absolute dollars are small. The real value of running at $100 is:
1. Seeing how the bot actually behaves with real stakes
2. Learning what to watch, what to tune
3. Getting comfortable before putting more in

Don't judge the bot on profit/loss from the first week. Judge it on behavior correctness and risk adherence. Profit takes time and tuning.
