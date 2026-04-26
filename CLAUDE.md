# CLAUDE.md — Project Context & Collaboration Guide

## Who is the user

Harish — medium-beginner trader, new to Hyperliquid and perpetual futures. Planning to start with $100, testnet first, then real money. Goal is to understand every aspect of this bot before running it with real funds.

Skills context: intermediate Python reader (can follow code), not a quant. Needs trading concepts explained, not assumed.

## Final goal

**Get this Hyperliquid AI trading bot running profitably with real money, in a controlled and understood way.**

The path to get there:
1. Fully understand the codebase (documentation phase — in progress)
2. Fix critical bugs before any real money (see [docs/11-KNOWN-GAPS-AND-CAVEATS.md](docs/11-KNOWN-GAPS-AND-CAVEATS.md))
3. Run on testnet for 1–3 days to validate the plumbing works
4. Run on mainnet with $100 for 1–2 weeks at conservative risk settings
5. Evaluate results, tune, scale

## Current phase

**Phase 1 — Documentation.** Writing comprehensive docs in `docs/`. See the active plan below.

## Active plan

See [.claude/PLAN-DOCUMENTATION.md](.claude/PLAN-DOCUMENTATION.md) for the full documentation plan with progress tracking.

## Key things to remember when helping

- Explain trading concepts; never assume Harish already knows them
- Always link to specific files/lines when referencing code
- Use superpower skills (`superpowers:brainstorming`) when altering existing code
- Use CodeRabbit for code review when major changes are made
- The README and code defaults disagree (see [docs/04-RISK-MANAGEMENT.md](docs/04-RISK-MANAGEMENT.md)) — never trust README-stated defaults; read `config_loader.py`
- For setup/configuration: always note testnet vs. mainnet differences
- Before going live: C3, C5, C6, C2 bugs in coderabbit review must be fixed first

## Collaboration preferences

- Terse responses, no trailing summaries
- Explain the "why" when flagging risks
- Don't add features or refactor beyond what's requested
- When code changes are needed, plan first, code second
- Commit only when explicitly asked

## Project constraints

- Python 3.12+
- Dependencies managed via `pyproject.toml` (poetry) or direct pip install
- No test suite exists — validate by running on testnet
- Single-file entry point: `src/main.py`
- Config is 100% via environment variables (`.env` file)

## References

- [Hyperliquid testnet](https://app.hyperliquid-testnet.xyz)
- [Hyperliquid mainnet](https://app.hyperliquid.xyz)
- [Anthropic console](https://console.anthropic.com) — for API key + spend limits
- [CodeRabbit review findings](.claude/CODERABBIT-REVIEW.md)
