# quant_hf

Autonomous multi-agent statistical-arbitrage and microstructure fund for Hyperliquid perpetuals. It runs 24/7 as a `systemd` daemon on Ubuntu with a $110–$550 USDC account. Python 3.12, `asyncio`, DuckDB. No LLMs in the execution path.

## Strategies
- **StatArb:** Kalman-filter pairs trading, gated by ADF, OU half-life and Hurst filters.
- **Lead-Lag:** BTC/ETH/SOL shocks traded on laggards, confirmed by order book imbalance.
- **Funding/OI squeeze:** extreme funding plus open-interest growth and order book absorption.

## Safety
- $12 minimum notional per leg.
- Trades must clear a 3x fee EV gate.
- -3% daily drawdown kill-switch, persisted across restarts.
- Orphan-leg sweeper and exchange Dead Man's Switch.
- Maker-first (`Alo`) execution. All exits use `reduce_only`.

## Prompts
Run these in order, each in its own agent session:

| File | Purpose |
|------|---------|
| [head.md](head.md) | Builder: full system spec and implementation |
| [auditor.md](auditor.md) | Red-team audit, fixes and adversarial tests |
| [dashboard.md](dashboard.md) | Textual TUI for monitoring over SSH |
| [launch.md](launch.md) | Four-gate deployment checklist (OS, audit, paper, live) |

## Quick start
```bash
uv sync
uv run pytest -v
uv run python -m backtest.walk_forward --days 14 --capital 300 --stress
uv run python -m ui.dashboard --demo
```

Set `MODE` to `BACKTEST`, `PAPER` or `LIVE` in `.env` (see `.env.example`). Don't go live before completing every gate in [launch.md](launch.md).
