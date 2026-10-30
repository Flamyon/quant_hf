# UBUNTU LAUNCH SEQUENCE & OPERATIONAL CHECKLIST (DEPLOYMENT PROMPT)

## 1. ROLE & OBJECTIVE
Track progress through the 4 sequential deployment gates below. Do not advance to the next phase until every item in the current phase is marked `[x]`.

## 2. PHASE 1: UBUNTU OS HARDENING & ENVIRONMENT SETUP
- [ ] [os_1] Disable Ubuntu lid-close suspend in `/etc/systemd/logind.conf`:
  `sudo sed -i 's/#HandleLidSwitch=suspend/HandleLidSwitch=ignore/' /etc/systemd/logind.conf && sudo sed -i 's/#HandleLidSwitchExternalPower=suspend/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf && sudo systemctl restart systemd-logind`
- [ ] [os_2] Install and verify `chrony` NTP clock sync (clock drift under 5ms):
  `sudo apt update && sudo apt install -y chrony && sudo systemctl enable --now chrony && chronyc tracking`
- [ ] [os_3] Tune Linux kernel TCP stack for low-latency WebSockets (`tcp_fastopen=3`, `tcp_tw_reuse=1`, `rmem_max=16777216`) via `scripts/harden_ubuntu.sh` and `sudo sysctl -p`.
- [ ] [os_4] Install `uv` and initialize an isolated Python 3.12 workspace in `~/quant_hf`.

## 3. PHASE 2: CODE AGENT BUILD & ADVERSARIAL RED-TEAM AUDIT
- [ ] [code_1] Run the Builder Agent (`head.md`) in `~/quant_hf` to scaffold and implement all modules, agents, DuckDB schemas, and unit tests.
- [ ] [code_2] Open a fresh agent session and run the Auditor Agent (`auditor.md`); generate the `RED-TEAM VULNERABILITY REPORT`, patch all identified flaws, and verify all 8 torture tests in `tests/test_adversarial_audit.py` pass (`pytest -v` 100% green).
- [ ] [code_3] Run the Dashboard Agent (`dashboard.md`) and verify the TUI with `uv run python -m ui.dashboard --demo`.
- [ ] [code_4] Run the walk-forward backtest under hostile stress assumptions (2x slippage, 50% Maker-to-Taker fallback):
  `uv run python -m backtest.walk_forward --days 14 --capital 300 --stress`
  Gate criteria: Net Annualized Sharpe > 1.5, Max Drawdown < 8%, Fee-to-Gross-PnL Ratio < 40%.

## 4. PHASE 3: 7-DAY PAPER LIVE STRESS TEST (REAL-TIME WEBSOCKETS)
- [ ] [paper_1] Configure `.env` with `MODE=PAPER`, `CAPITAL_USDC=300`, and Telegram Bot credentials; install and start the `systemd` service:
  `sudo cp scripts/quant-hf.service /etc/systemd/system/ && sudo systemctl daemon-reload && sudo systemctl enable --now quant-hf.service`
- [ ] [paper_2] Run continuously for 7 full days on the Ubuntu laptop with lid closed; monitor logs via `journalctl -u quant-hf -f`.
- [ ] [paper_3] Network chaos test: unplug Ethernet / disable Wi-Fi for 60 seconds during active Paper trading; verify `data_agent` reconnects cleanly and `reconciler_agent` restores state without crashing.
- [ ] [paper_4] Test the remote Telegram `/panic` command from a phone; verify the service flattens simulated positions, persists the lock in DuckDB, and halts cleanly.

## 5. PHASE 4: MICRO-LIVE CAPITAL DEPLOYMENT (100€ -> 500€ SCALING)
- [ ] [live_1] Generate an isolated Hyperliquid API Agent Wallet at `app.hyperliquid.xyz/API` (trade/cancel only; zero withdrawal permissions) and add `HL_ACCOUNT_ADDRESS` and `HL_API_PRIVATE_KEY` to `.env` (`chmod 600 .env`).
- [ ] [live_2] Deploy the initial 100€ ($110 USDC) in `MODE=LIVE`, capped at 2 concurrent pairs (`MAX_ACTIVE_PAIRS=2`).
- [ ] [live_3] Audit the first 50 live round-trip trades in DuckDB: verify realized Maker/Taker fee ratios, zero unhedged orphan legs, and live slippage within 15% of Paper mode assumptions before scaling capital to 500€ ($550 USDC).
