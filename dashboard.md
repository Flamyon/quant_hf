# REAL-TIME QUANT TELEMETRY & SSH TERMINAL DASHBOARD (TUI AGENT PROMPT)

## 1. ROLE & OBJECTIVE
You are a Lead Quantitative Telemetry & Low-Latency UI Engineer. Build a standalone, zero-interference Terminal User Interface (TUI) in Python 3.12 (`ui/dashboard.py`) using `textual` and `rich` (install via `uv add textual rich`).
The dashboard must allow the fund manager to monitor the Ubuntu daemon locally or remotely via SSH in real time without ever blocking, locking, or slowing down the live trading `asyncio` loop or DuckDB writes.

## 2. STRICT ARCHITECTURAL & CONCURRENCY CONSTRAINTS
1. **ZERO LOCK CONTENTION ON DUCKDB:** DuckDB allows only one single read-write process at a time. Because the live trading daemon (`main.py`) holds the write lock on `quant_hf.duckdb`, the TUI dashboard MUST NEVER open `quant_hf.duckdb` in read-write mode or block the trading engine.
2. **LOCK-FREE TELEMETRY SNAPSHOTS + READ-ONLY QUERIES:**
   - Have `main.py` / `state.py` write an atomic, memory-mapped JSON/MsgPack snapshot every 500ms to `/dev/shm/quant_hf_telemetry.json` (Ubuntu RAM disk—zero SSD wear, microsecond latency) containing live WebSocket tick timestamps, active Kalman Z-scores, L2 Order Book Imbalances (OBI), open positions, and `OrderLifecycleState`.
   - For historical PnL, trade ledger, and fee ratios, have the TUI query DuckDB strictly using `duckdb.connect("quant_hf.duckdb", read_only=True)` or read periodic Parquet snapshots exported by the daemon.
3. **EMERGENCY IPC KILL-SWITCH:** Pressing `P` (Panic) inside the TUI must require a `Y/N` confirmation modal and, if confirmed, send an immediate local UNIX signal (`SIGUSR1`) or write a trigger file `/dev/shm/quant_hf_panic.trigger` that `risk_agent.py` checks every 100ms to cancel all orders, flatten all legs with `reduce_only=True`, and halt trading.

## 3. DASHBOARD LAYOUT & WIDGET SPECIFICATIONS (`ui/dashboard.py`)
Build a full-screen, Bloomberg-style dark-mode terminal grid refreshing at 4 Hz (every 250ms) with the following 5 panels:

### Panel 1: Top System & CRO Risk Header (`HeaderBar`)
- **Mode Badge:** Color-coded indicator (`BACKTEST` [Blue], `PAPER` [Yellow], `LIVE` [Red/Blinking]).
- **Account Equity & Drawdown Gauge:** Live Total Equity (Wallet + Mark-to-Mid Unrealized PnL), 24h High-Water Mark ($E_{\text{HWM, 24h}}$), current Daily Drawdown % (progress bar toward the `-3.0%` hard kill-switch), and Gross Leverage (`0.00x / 2.50x`).
- **System Health:** WebSocket Tick Age (`ms` since last packet—turns Red if `> 1000ms`), Dead Man's Switch countdown (`schedule_cancel` TTL), and Ubuntu NTP clock drift (`chronyc` status).

### Panel 2: Live Alpha Radar — Cointegration & Kalman Z-Scores (`StatArbTable`)
- DataTable sorted by $|Z_t|$ descending across all active cointegrated pairs:
  - Columns: `Pair (A/B)` | `Hedge Beta (β_t)` | `OU Half-Life (m)` | `ADF p-val` | `Live Z-Score (Z_t)` | `Visual Bar [-4.0 to +4.0]` | `EV / Fee Ratio` | `Status`
  - Color coding: Highlight rows in green/amber when $|Z_t| \ge 2.0$ (approaching $\pm 2.3$ entry threshold) and red if $|Z_t| \ge 3.5$ (approaching $\pm 3.8$ regime-break stop).

### Panel 3: Microstructure & Lead-Lag Scanner (`MicrostructurePanel`)
- Displays the 3 Market Leaders (`BTC`, `ETH`, `SOL`) 60-second return shocks ($\sigma$ units) alongside the top 5 Granger-causal Laggards:
  - Columns: `Leader -> Laggard` | `Leader Shock (σ)` | `Laggard Lag Gap` | `L2 OBI [-1 to +1]` | `Micro-Price Spread (bps)` | `Signal State`

### Panel 4: Active Positions, Maker Chaser & Orphan-Leg Monitor (`PositionsTable`)
- Displays every open leg and resting order with millisecond-level lifecycle visibility:
  - Columns: `Pair/Strategy` | `Leg A (Side/Sz/Px)` | `Leg B (Side/Sz/Px)` | `Lifecycle State` (`PENDING_PAIR`, `HEDGED`, `FLATTENING_ORPHAN`, `CLOSING`) | `Maker Timer (10s)` | `Net Unrealized PnL ($ / %)` | `Time Held / Max Hold`

### Panel 5: Execution Ledger & Fee-Drag Auditor (`FooterLog & Metrics`)
- **Left Split:** Institutional performance summary (`Net Sharpe`, `Sortino`, `Win Rate %`, `Gross PnL`, `Total Fees Paid`, and `Fee-to-Gross-PnL %`—turns Red if `> 35%`).
- **Right Split:** Scrolling real-time structured log tail showing `Alo` Post-Only Maker fills, cancel/replaces, and `reconciler_agent` audit checks.

## 4. KEYBOARD SHORTCUTS (HOTKEYS)
- `q`: Quit TUI cleanly (leaves background `systemd` trading daemon running untouched).
- `r`: Force immediate state reconciliation (`reconciler_agent` REST sync).
- `f`: Filter Alpha Radar between All Pairs vs. Active Signals Only.
- `p`: **EMERGENCY PANIC FLATTEN** (opens confirmation modal -> triggers `risk_agent` immediate `reduce_only=True` market flatten of all positions and halts trading).

## 5. DELIVERABLES & EXECUTION PLAN
1. Create `core/telemetry.py` to publish lock-free atomic JSON state snapshots to `/dev/shm/quant_hf_telemetry.json` from `main.py`.
2. Create `ui/dashboard.py` implementing the complete `Textual` application with CSS styling and a `--demo` flag (`uv run python -m ui.dashboard --demo`) that simulates live fluctuating Kalman Z-scores, L2 OBI shifts, and a multi-leg trade lifecycle so the layout can be inspected immediately before live trading.
3. Add a bash alias script `scripts/watch_hf.sh` (`uv run python -m ui.dashboard`) so the user can launch the dashboard anytime over SSH with a single command.
