# Documentation Plan — Hyperliquid Trading Bot

Referenced from [CLAUDE.md](../CLAUDE.md).

## Goal

Produce a complete docs/ folder that lets Harish understand every aspect of the bot at both high-level and granular level, then follow a tested step-by-step path to run it safely with real money.

## Progress

| # | File | Status | Notes |
|---|------|--------|-------|
| 00 | [00-OVERVIEW.md](../docs/00-OVERVIEW.md) | ✅ Done | Plain-English intro, costs, honest expectations |
| 01 | [01-TRADING-PRIMER.md](../docs/01-TRADING-PRIMER.md) | ✅ Done | Perp futures, leverage, funding, TP/SL for beginners |
| 02 | [02-ARCHITECTURE.md](../docs/02-ARCHITECTURE.md) | ✅ Done | Subsystems, data flow diagrams, lifecycle |
| 03 | [03-HOW-DECISIONS-ARE-MADE.md](../docs/03-HOW-DECISIONS-ARE-MADE.md) | ✅ Done | Loop step-by-step with line references |
| 04 | [04-RISK-MANAGEMENT.md](../docs/04-RISK-MANAGEMENT.md) | ✅ Done | Every guard, README/code drift table, recommended settings |
| 05 | [05-INDICATORS-EXPLAINED.md](../docs/05-INDICATORS-EXPLAINED.md) | ✅ Done | EMA/RSI/MACD/ATR/BBands/ADX/OBV/VWAP in trader terms |
| 06 | [06-CODE-WALKTHROUGH.md](../docs/06-CODE-WALKTHROUGH.md) | ✅ Done | File-by-file, key lines, what each piece does |
| 07 | [07-CONFIGURATION-REFERENCE.md](../docs/07-CONFIGURATION-REFERENCE.md) | ✅ Done | Every env var, valid values, recommended first-run settings |
| 08 | [08-SETUP-TESTNET.md](../docs/08-SETUP-TESTNET.md) | ✅ Done | Wallet setup → testnet → first run |
| 09 | [09-GOING-LIVE.md](../docs/09-GOING-LIVE.md) | ✅ Done | Mainnet $100 ramp with conservative settings |
| 10 | [10-OPERATIONAL-RUNBOOK.md](../docs/10-OPERATIONAL-RUNBOOK.md) | ✅ Done | Monitor, restart, troubleshoot, stop |
| 11 | [11-KNOWN-GAPS-AND-CAVEATS.md](../docs/11-KNOWN-GAPS-AND-CAVEATS.md) | ✅ Done | Coderabbit findings, bugs, honest risk disclosure |
| — | [docs/README.md](../docs/README.md) | ✅ Done | Table of contents |

## CodeRabbit Review

Findings already captured in [CODERABBIT-REVIEW.md](CODERABBIT-REVIEW.md).

Critical bugs that must be fixed before mainnet:
- **C3** — force-close opens opposite-side position (not reduce-only)
- **C5** — TP/SL not validated against current price direction
- **C6** — stale price used for sizing after LLM call latency
- **C2** — balance-reserve check uses withdrawable not account_value

## Phases after documentation

Phase 2: Fix bugs C2, C3, C5, C6
Phase 3: Testnet run (1–3 days)
Phase 4: Mainnet $100 run (conservative settings, 1–2 weeks)
Phase 5: Review, tune, scale
