# Documentation Index

Complete documentation for the Hyperliquid AI Trading Bot.

---

## Start here

| Doc | What it covers | Read time |
|-----|---------------|-----------|
| [00 — Overview](00-OVERVIEW.md) | What this bot is, what it costs, honest expectations | 10 min |
| [01 — Trading Primer](01-TRADING-PRIMER.md) | Perp futures, leverage, funding, TP/SL — for beginners | 15 min |
| [11 — Known Gaps & Caveats](11-KNOWN-GAPS-AND-CAVEATS.md) | Bugs and risks before going live | 10 min |

## How it works

| Doc | What it covers |
|-----|---------------|
| [02 — Architecture](02-ARCHITECTURE.md) | Subsystems, data flow, lifecycle |
| [03 — How Decisions Are Made](03-HOW-DECISIONS-ARE-MADE.md) | One full loop iteration, step by step |
| [04 — Risk Management](04-RISK-MANAGEMENT.md) | Every safety guard, README vs. code defaults, recommended settings |
| [05 — Indicators Explained](05-INDICATORS-EXPLAINED.md) | EMA, RSI, MACD, ATR, BBands, ADX, OBV, VWAP in trader terms |
| [06 — Code Walkthrough](06-CODE-WALKTHROUGH.md) | File-by-file tour with line references |

## Setup and operations

| Doc | What it covers |
|-----|---------------|
| [07 — Configuration Reference](07-CONFIGURATION-REFERENCE.md) | Every env var, valid values, recommended first-run settings |
| [08 — Setup: Testnet](08-SETUP-TESTNET.md) | Wallet setup → testnet → first run (start here) |
| [09 — Going Live](09-GOING-LIVE.md) | Mainnet $100 ramp, conservative settings, what to watch |
| [10 — Operational Runbook](10-OPERATIONAL-RUNBOOK.md) | Monitor, restart, troubleshoot, stop, emergency procedures |

---

## The recommended reading path

**If you're new**: 00 → 01 → 03 → 04 → 08 → 11 → 09

**If you want to understand the code**: 02 → 03 → 05 → 06

**If you just want to set it up**: 07 → 08 → 09

**Before any real money**: you must read 04 and 11.

---

## Project files (outside docs)

- [CLAUDE.md](../CLAUDE.md) — collaboration guide and project memory
- [.claude/PLAN-DOCUMENTATION.md](../.claude/PLAN-DOCUMENTATION.md) — documentation plan and progress
- [.claude/CODERABBIT-REVIEW.md](../.claude/CODERABBIT-REVIEW.md) — full code review findings
- [README.md](../README.md) — quickstart (note: defaults in README don't match code — see doc 04)
