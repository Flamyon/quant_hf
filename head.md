# AUTONOMOUS MULTI-AGENT QUANTITATIVE HEDGE FUND (BUILDER AGENT PROMPT)

## 1. ROLE & OBJECTIVE
You are a Principal Quantitative Systems Architect and Head of Algorithmic Execution. Your task is to build a complete, institutional-grade, multi-agent Statistical Arbitrage and Microstructure Hedge Fund in Python 3.12+ (`~/quant_hf`). 
The system will run 24/7 as a hardened `systemd` daemon on an Ubuntu Linux machine trading Hyperliquid Perpetual Futures with an initial capital base of $110–$550 USDC (100€–500€).

## 2. NON-NEGOTIABLE ENGINEERING & MATHEMATICAL CONSTRAINTS
1. **ZERO PLACEHOLDERS OR MOCKS:** Never write `pass`, `TODO`, `...`, or stubbed functions. Every mathematical equation, WebSocket handler, database query, and error-recovery routine must be implemented in full, production-ready code.
2. **NO LLMs IN THE REAL-TIME EXECUTION PATH:** Live market data ingestion, alpha calculation, risk checks, and order execution must be 100% deterministic, asynchronous Python (`asyncio`) powered by `numpy`, `scipy`, `statsmodels`, and `polars`.
3. **HYPERLIQUID EXCHANGE REALISM (SMALL ACCOUNT CONSTRAINTS):**
   - Minimum order notional on Hyperliquid is $10.00 USD per leg. Enforce a strict minimum order size of **$12.00 USD** per leg in code so small price dips do not cause exchange order rejections.
   - Fees: Maker (`Alo` / Add Liquidity Only) = **0.015%** (1.5 bps); Taker (`Ioc` / Market) = **0.045%** (4.5 bps).
   - **Expected Value (EV) Gate:** Any trade signal where expected gross profit to target is less than **3.0x** estimated round-trip fees + slippage must be rejected by the Risk Agent.
4. **ZERO LOOK-AHEAD BIAS:** All rolling regressions, cointegration tests, volatility estimators, and Kalman Filters must update strictly point-in-time on closed bar events or live order-book ticks.
5. **CRASH, DISCONNECT & ORPHAN-LEG SAFETY:**
   - Implement Hyperliquid's native Dead Man's Switch (`schedule_cancel`) refreshed every 20 seconds (cancels all open orders at $T + 40\text{s}$ if the Ubuntu machine loses internet or crashes).
   - Implement an **Orphan-Leg Sweeper**: When executing a 2-leg pair or basket trade, if Leg A fills and Leg B fails to fill within 10 seconds, immediately flatten Leg A via Taker IOC order to prevent unhedged directional exposure.

## 3. COMPLETE REPOSITORY STRUCTURE TO GENERATE
Create and populate the exact project tree below using `uv` for dependency management:

```text
quant_hf/
├── pyproject.toml               # Python 3.12+, hyperliquid-python-sdk, eth-account, numpy, scipy, statsmodels, polars, duckdb, websockets, aiohttp, pydantic, pydantic-settings, structlog, httpx, pytest, pytest-asyncio
├── .env.example                 # Template for ENV vars: HL_ACCOUNT_ADDRESS, HL_API_PRIVATE_KEY, MODE (BACKTEST|PAPER|LIVE), CAPITAL_USDC, TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID
├── config/
│   ├── settings.py              # Validated Pydantic BaseSettings loading .env and risk parameters
│   └── universe.yaml            # Asset clusters: L1_CLUSTER (BTC, ETH, SOL, SUI, APT, AVAX, SEI, TIA), DEFI_CLUSTER (UNI, AAVE, MKR, LDO, PENDLE, JUP), AI_CLUSTER (TAO, FET, RENDER, WLD, NEAR), MEME_CLUSTER (DOGE, PEPE, WIF, BONK)
├── core/
│   ├── events.py                # Immutable Dataclasses/Pydantic models: MarketTickEvent, BarClosedEvent, L2BookEvent, SignalEvent, OrderRequestEvent, FillEvent, RiskAlertEvent, SystemHeartbeatEvent
│   ├── bus.py                   # High-throughput asyncio.Queue EventBus with pub/sub routing, backpressure monitoring, and exception isolation
│   ├── state.py                 # Thread-safe in-memory PortfolioState (positions, open orders, margin usage, unrealized/realized PnL, peak equity)
│   └── database.py              # DuckDB persistent time-series store for 1m OHLCV bars, L2 book snapshots, Kalman states, trade ledger, and equity curve
├── math_engine/
│   ├── kalman.py                # 1D Dynamic Linear Kalman Filter for real-time hedge ratio (beta), intercept (alpha), and innovation Z-score
│   ├── cointegration.py         # Engle-Granger ADF stationarity test, Johansen multi-asset basket test, and Ornstein-Uhlenbeck (OU) mean-reversion half-life
│   ├── leadlag.py               # Log-return cross-correlation matrix & Granger Causality F-test for Leader-Laggard discovery
│   └── microstructure.py        # Multi-level Order Book Imbalance (OBI), Volume-Synchronized Probability of Informed Trading (VPIN) proxy, and micro-price estimator
├── agents/
│   ├── data_agent.py            # Async WebSocket client streaming Hyperliquid `l2Book`, `candle` (1m), `activeAssetCtx` (funding/OI), and `userEvents` with exponential backoff reconnect
│   ├── statarb_agent.py         # Alpha Agent 1: Cointegrated Pairs & Cluster Mean-Reversion (Kalman Z-score + OU Half-Life filter)
│   ├── leadlag_agent.py         # Alpha Agent 2: Cross-Asset Leader Shock -> Laggard Catch-Up with Order Book Imbalance confirmation
│   ├── funding_obi_agent.py     # Alpha Agent 3: Extreme Funding Rate + Open Interest Z-score + Order Book Absorption Squeeze
│   ├── risk_agent.py            # Chief Risk Officer (CRO): Fractional Kelly sizing, $12 min notional check, max leverage cap, correlation concentration check, and -3% daily drawdown kill-switch
│   ├── execution_agent.py       # Smart Order Router: `BACKTEST`, `PAPER` (L2 queue simulation), and `LIVE` (Hyperliquid SDK). Maker-first (`Alo`) chasing, Orphan-Leg Sweeper, and Dead Man's Switch
│   ├── reconciler_agent.py      # Runs every 10s: queries Hyperliquid REST `user_state`, reconciles against local `PortfolioState`, fixes drift, and flattens unauthorized/orphan exposure
│   ├── nightly_research_agent.py# Runs every 24h at 00:05 UTC: recalculates cointegration clusters and Granger matrices over trailing 14 days in DuckDB and hot-reloads active pairs
│   └── notifier_agent.py        # Telegram Bot listener & pusher: sends trade fills, hourly PnL/Sharpe reports, and processes remote `/status`, `/pause`, and `/panic` (flatten all + shutdown) commands
├── backtest/
│   ├── data_fetcher.py          # Downloads historical 1m candles and funding rates from Hyperliquid/CCXT into DuckDB
│   └── walk_forward.py          # Event-driven walk-forward engine replaying historical events through the exact live agents; outputs Sharpe, Sortino, Calmar, Max Drawdown, Win Rate, and Fee-to-Gross-PnL ratio
├── scripts/
│   ├── harden_ubuntu.sh         # Bash script configuring Ubuntu sysctl TCP low-latency flags, chrony NTP sync, lid-suspend disable, and systemd installation
│   └── quant-hf.service         # Production systemd unit file with auto-restart, memory/file-descriptor limits, and structured journald logging
├── tests/
│   ├── test_math_engine.py      # Rigorous unit tests verifying Kalman convergence, OU half-life accuracy, and Granger causality on synthetic stochastic processes
│   ├── test_risk_agent.py       # Unit tests proving the $12 notional floor, EV fee gate, and -3% drawdown kill-switch trigger accurately
│   └── test_execution_orphan.py # Async test verifying the Orphan-Leg Sweeper flattens an unhedged leg after timeout
└── main.py                      # Master async orchestrator handling startup warm-up, agent spawning, and graceful SIGINT/SIGTERM shutdown (cancels all resting orders before exit)
```

## 4. EXACT MATHEMATICAL SPECIFICATIONS TO IMPLEMENT

### A. Dynamic Kalman Filter for Pairs StatArb (`math_engine/kalman.py`)

Do not use static rolling OLS regressions. Model the relationship between log-prices $y_t = \ln(P_{A,t})$ and $x_t = \ln(P_{B,t})$ using a state-space Linear Dynamic Model:

* **State Vector:** $\theta_t = [\alpha_t, \beta_t]^T$ (intercept and dynamic hedge ratio)
* **Observation Matrix:** $F_t = [1, x_t]$
* **State Transition:** $\theta_t = \theta_{t-1} + \omega_t$, where $\omega_t \sim \mathcal{N}(0, W)$ and state transition covariance $W = \frac{\delta}{1 - \delta} I_2$ with $\delta = 10^{-4}$.
* **Update Equations per 1-minute bar:**
1. Prior state covariance: $R_t = C_{t-1} + W$
2. One-step-ahead forecast: $\hat{y}_t = F_t \theta_{t-1}$
3. Prediction error (innovation spread): $e_t = y_t - \hat{y}_t$
4. Total forecast variance: $Q_t = F_t R_t F_t^T + V_v$ (where observation noise variance $V_v = 10^{-3}$ is updated via EWMA of squared residuals)
5. **Innovation Z-Score (Trade Signal):**

$$Z_t = \frac{e_t}{\sqrt{Q_t}}$$

6. Kalman Gain: $K_t = R_t F_t^T Q_t^{-1}$
7. Posterior state update: $\theta_t = \theta_{t-1} + K_t e_t$
8. Posterior covariance update: $C_t = R_t - K_t F_t R_t$

### B. Cointegration & Ornstein-Uhlenbeck (OU) Half-Life (`math_engine/cointegration.py`)

A pair is only eligible for trading in `statarb_agent.py` if it passes three mathematical filters over the trailing $N = 2000$ one-minute bars:

1. **Engle-Granger ADF Test:** Run Augmented Dickey-Fuller on the spread $S_t = y_t - (\alpha + \beta x_t)$. Require $p\text{-value} < 0.02$ and $t\text{-stat} < \text{critical value}_{1\%}$.
2. **Ornstein-Uhlenbeck Half-Life ($t_{1/2}$):** Model spread dynamics via discrete OLS regression:

$$\Delta S_t = S_t - S_{t-1} = \mu + \lambda S_{t-1} + \epsilon_t$$

Compute half-life in minutes:

$$t_{1/2} = -\frac{\ln(2)}{\lambda}$$

**Strict Filter:** Reject pair if $\lambda \ge 0$ (divergent) or if $t_{1/2} < 5$ minutes (noise/fee trap) or $t_{1/2} > 180$ minutes (capital locked too long).
3. **Hurst Exponent Filter:** Calculate rescaled-range or variance-ratio Hurst exponent $H$. Require $H < 0.42$ to confirm anti-persistent mean reversion.

### C. Strategy 1: Cluster Cointegration Rules (`agents/statarb_agent.py`)

* **Entry Condition:** Pair is active ($p < 0.02$, $5 \le t_{1/2} \le 180$, $H < 0.42$), $|Z_t| \ge 2.3$, and expected reversion move $\ge 3.0 \times \text{total round-trip fees}$.
* If $Z_t \ge +2.3$: Short Asset A, Long $\beta_t$ units of Asset B (dollar-neutral notional adjustment).
* If $Z_t \le -2.3$: Long Asset A, Short $\beta_t$ units of Asset B.

* **Exit Condition (Take Profit):** $|Z_t| \le 0.15$ (reverted to mean).
* **Stop-Loss Condition (Regime Break):** $|Z_t| \ge 3.8$ OR holding time exceeds $3.0 \times t_{1/2}$ minutes.

### D. Strategy 2: Cross-Asset Lead-Lag + Order Book Imbalance (`agents/leadlag_agent.py`)

* **Offline/Hourly Calibration (`math_engine/leadlag.py`):** For Leader $L \in \{\text{BTC, ETH, SOL}\}$ and Sector Laggard $M$, compute Granger Causality F-test on 1-minute log returns $r_{L,t-k} \to r_{M,t}$ for lags $k \in \{1, 2, 3\}$. Keep pairs with Granger $p\text{-value} < 0.005$ and cross-correlation $\rho > 0.55$.
* **Real-Time Microstructure Trigger (`math_engine/microstructure.py`):**
Compute top-5 level Order Book Imbalance (OBI) and Volume-Weighted Micro-Price ($P_{\text{micro}}$):

$$\text{OBI}_t = \frac{\sum_{i=1}^5 V_{i}^{\text{bid}} \cdot e^{-\gamma (i-1)} - \sum_{i=1}^5 V_{i}^{\text{ask}} \cdot e^{-\gamma (i-1)}}{\sum_{i=1}^5 V_{i}^{\text{bid}} \cdot e^{-\gamma (i-1)} + \sum_{i=1}^5 V_{i}^{\text{ask}} \cdot e^{-\gamma (i-1)}}, \quad \gamma = 0.5$$

$$P_{\text{micro}} = \frac{V_1^{\text{bid}} P_1^{\text{ask}} + V_1^{\text{ask}} P_1^{\text{bid}}}{V_1^{\text{bid}} + V_1^{\text{ask}}}$$

* **Entry Condition:** Leader $L$ experiences a return impulse $|r_{L, 60\text{s}}| > 2.8 \sigma_L$, Laggard $M$ has moved less than $0.3 \times \beta_{M|L} r_{L, 60\text{s}}$ (lagging behind), AND Laggard's live $\text{OBI}_t$ aligns with the impulse direction ($|\text{OBI}_t| > 0.55$ in direction of trade).
* **Exit Condition:** Target catch-up move reached ($0.75 \times \beta_{M|L} r_{L, 60\text{s}}$), trailing volatility stop hit, or 12-minute time-stop expired.

### E. Strategy 3: Funding Rate + OI Squeeze (`agents/funding_obi_agent.py`)

* Monitors `activeAssetCtx` from Hyperliquid WebSocket (hourly funding rate and Open Interest).
* When annualized funding rate is below $-35\%$ (crowded shorts) AND Open Interest 4-hour change $> +8\%$ AND price breaks 30-minute high with $\text{OBI}_t > +0.60$, trigger a Long momentum squeeze trade (and symmetric opposite for crowded longs with funding $> +60\%$).

## 5. RISK MANAGEMENT & EXECUTION SPECIFICATIONS

### A. Chief Risk Officer (`agents/risk_agent.py`)

Every `SignalEvent` must pass through `RiskAgent.evaluate(signal, portfolio_state)` before reaching execution:

1. **Minimum Notional & Small Capital Scaling:**
* Given account equity $E \in [110, 550]$ USDC, compute target position size using volatility-targeted Fractional Kelly ($0.20 \times f^*$), floored at **$12.00 USD** per leg and capped at **$0.25 \times E$** per leg.
* If $0.25 \times E < \$12.00$, automatically cap maximum concurrent active pairs to 2 and allow up to $0.35 \times E$ per leg so a $110 account can still execute valid 2-leg trades safely.

2. **Portfolio Leverage & Concentration Caps:**
* Max Gross Leverage across all open legs: **2.5x** account equity.
* Max Net Directional Delta: **$\pm 20\%$** of account equity (ensures fund stays nearly market-neutral).
* Max Concurrent Active Strategies/Pairs: **4**.

3. **Hard Daily Drawdown Kill-Switch:**
* Track rolling 24-hour High-Water Mark ($E_{\text{HWM, 24h}}$).
* If total equity (realized + unrealized PnL) drops below $0.97 \times E_{\text{HWM, 24h}}$ (**-3.0% drawdown**), emit `RiskAlertEvent(CRITICAL)`, cancel all resting orders, flatten all open positions via Taker IOC, lock trading for 24 hours, and send an emergency Telegram alert.

### B. Smart Order Execution (`agents/execution_agent.py`)

1. **Three Pluggable Drivers:**
* `BacktestExecutionDriver`: Simulates fills against historical 1m bars + configurable bps slippage + exact Maker/Taker fee deduction.
* `PaperLiveExecutionDriver`: Connects to live Hyperliquid L2 WebSockets. Simulates Maker limit order queue fills only when live bid/ask crosses or consumes our price level, and models realistic Taker slippage across top-10 L2 book levels.
* `HyperliquidLiveDriver`: Uses official `hyperliquid.exchange.Exchange` and `eth_account.Account.from_key(HL_API_PRIVATE_KEY)`. Rounds prices to exact asset `szDecimals` and tick sizes fetched from `info.meta()`.

2. **Maker-First Limit Chaser (`Alo` Post-Only):**
* To preserve edge on a 100€–500€ account, submit entry orders as Post-Only Limit (`{"limit": {"tif": "Alo"}}`) at the best bid (for buys) or best ask (for sells).
* If the BBO moves away by 1 tick within 8 seconds, cancel and replace once at the new BBO.
* If unfilled after 10 seconds:
* If neither leg of a pair has filled, cancel both cleanly (zero fee paid).
* If Leg A filled and Leg B is still unfilled at $T = 10\text{s}$, execute Leg B via IOC Taker if spread still satisfies EV gate; otherwise **immediately flatten Leg A via IOC Taker** (Orphan-Leg Sweeper).

3. **Exchange-Level Dead Man's Switch:**
* Background task runs every 20 seconds calling Hyperliquid's `schedule_cancel(int(time.time() * 1000) + 40000)`.

## 6. UBUNTU LINUX HARDENING & SYSTEMD DEPLOYMENT (`scripts/`)

Generate `scripts/harden_ubuntu.sh` and `scripts/quant-hf.service` so the user can turn their Ubuntu laptop into an appliance with one command:

1. Disable laptop lid suspend (`HandleLidSwitch=ignore` in `/etc/systemd/logind.conf`).
2. Install and enable `chrony` for sub-millisecond NTP clock synchronization.
3. Apply low-latency TCP network buffers (`tcp_fastopen=3`, `tcp_tw_reuse=1`, `rmem_max=16777216`).
4. Configure `quant-hf.service` with `Restart=always`, `RestartSec=3`, `LimitNOFILE=65535`, `Nice=-10` (high CPU scheduling priority), and clean `SIGTERM` handling so `systemctl stop quant-hf` gracefully cancels all open exchange orders before exiting.

## 7. EXECUTION PLAN FOR YOU (THE BUILDER AGENT)

Execute the following steps autonomously in order without stopping to ask for permission:

1. **Scaffold & Install:** Create the directory tree, `pyproject.toml`, `.env.example`, and `config/universe.yaml`. Run `uv sync` (or set up virtualenv and install packages).
2. **Build Core & Math Engine:** Write `core/events.py`, `core/bus.py`, `core/state.py`, `core/database.py`, and all four modules in `math_engine/` (`kalman.py`, `cointegration.py`, `leadlag.py`, `microstructure.py`).
3. **Write & Run Math Unit Tests:** Write `tests/test_math_engine.py` using synthetic Ornstein-Uhlenbeck cointegrated price series and lead-lag series. Run `pytest` and fix any numerical bugs until 100% green.
4. **Build All Agents:** Implement `data_agent.py`, `statarb_agent.py`, `leadlag_agent.py`, `funding_obi_agent.py`, `risk_agent.py`, `execution_agent.py`, `reconciler_agent.py`, `nightly_research_agent.py`, and `notifier_agent.py`.
5. **Write & Run Risk/Execution Tests:** Write `tests/test_risk_agent.py` and `tests/test_execution_orphan.py`. Run `pytest` until 100% green.
6. **Build Backtester & Orchestrator:** Implement `backtest/data_fetcher.py`, `backtest/walk_forward.py`, `main.py`, `scripts/harden_ubuntu.sh`, and `scripts/quant-hf.service`.
7. **Run End-to-End Verification:** Execute a full walk-forward backtest (`python -m backtest.walk_forward --days 14 --capital 300`) using either live fetched Hyperliquid 1m candles or realistic synthetic multi-asset cluster data if offline, and print the complete institutional tearsheet (Net PnL, Sharpe, Sortino, Max Drawdown, Win Rate, Total Fees Paid, Fee-to-Gross-PnL Ratio).
