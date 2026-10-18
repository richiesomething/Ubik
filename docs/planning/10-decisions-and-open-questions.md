# Ubik — Decisions & Open Questions

> Status: **Draft v0.1** (2026-10-18). This doc records (1) the decisions the planning lead made so the domain docs agree with each other, (2) the questions only the owner can answer, with the default Ubik will use until they do, and (3) the combined top risks. **Every decision here can be overridden by the owner**; when one changes, update this log first, then the affected docs.

## Summary

- **About 30 planning decisions are recorded below.** The most consequential: build a thin Python core of our own (D-02), make paper trading Ubik's own simulator on live data with no exchange keys (D-05), allow only declarative strategies in P1 (D-07), build the full strategy → risk → order-management pipeline already in P1 (D-09), and take a staged six-step path to real money (D-13).
- **Six questions block development** and should be answered in Phase 0: your country and tax residence, which exchanges you can use, your trading horizon, whether Ubik will ever be used by anyone else, where it will be hosted, and your budgets. Each has a safe default, so planning can continue without them.
- **Other questions can wait** until P2 or P3 (risk tolerance on real money, alert channels, AI data sharing, auto-flatten behavior, and so on).
- **The top risks are false confidence** (paper or backtest results that won't survive real trading), **scope creep** from "everything pro firms do", **jurisdiction and exchange changes**, and **key or account compromise** once real money is involved.

## 1. Decision log

Status values: **Decided** (planning lead, owner may override) · **Provisional** (decided, pending a spike or owner answer) · **Superseded**.

### 1.1 Product and scope

| ID | Decision | Status | Why | Source |
|---|---|---|---|---|
| D-01 | Ubik is a **single-owner, self-hosted** tool trading the owner's own funds on the owner's own accounts through P3. Multi-user, signals, copy trading or managing others' money need a legal review first. User-owned data still carries an owner/workspace ID from P1 so multi-user stays possible. | Decided | Personal use is the low-regulation zone; retrofitting tenancy later is expensive | 00, 06, 08 (CMP-001, SEC-020) |
| D-06 | **Spot only, long or flat, until P4.** No leverage, margin, borrowing or shorting. True long/short pairs trading and funding carry wait for P4. | Decided | Beginner safety; regulatory simplicity | 00, 04 (ST-010), 05 (RK-035) |
| D-10 | **Strategy bars of 1h–1d are the recommended P1 default**, simulated on a 1-minute base. Faster strategies are allowed in Pro mode, with a fee-drag warning. | Decided | Costs are small relative to moves; better evidence; retail can't win on speed | 02 §1.3, 03, 04 (BT-003) |
| D-14 | **No high-frequency trading, latency arbitrage, live market making at retail fee tiers, MEV, or autonomous AI trading.** | Decided | An individual can't compete there, or it harms others | 00 §4.2, 02 §1.4 |

### 1.2 Architecture and technology

| ID | Decision | Status | Why | Source |
|---|---|---|---|---|
| D-02 | **Build Ubik's own thin core** (strategy API, event-driven engine, simulated venue, risk, order management, ledger) in Python. **Reuse** permissively licensed libraries: ccxt (exchanges), TA-Lib (indicator reference), Lightweight Charts (charts), polars/DuckDB (data). **Don't fork** Freqtrade (GPL, one bot per process) or build on vectorbt (Commons Clause). | Provisional: the NautilusTrader spike may change it | Ubik's differentiators live in the core; licenses keep a future distribution path open | 02 §4, 06 §8 |
| D-03 | **NautilusTrader gets a ≤ 2-week spike in Phase 0** (AR-028). Adopt it only if Ubik's strategy API, ledger and explanations sit cleanly on top and v2 is stable; otherwise revisit for P4. | Decided | Strongest open-source sim-live engine, but it's mid-rewrite (v2 release candidate in Oct 2026) | 02, 06 |
| D-04 | **Modular monolith**: one Python codebase deployed as `engine`, `api` and `worker` processes, plus an execution `gateway` from P3 that alone holds exchange keys. Valkey Streams between processes; SSE to the browser; Docker Compose deploy on amd64/arm64. | Decided | Simple for one developer; the gateway isolation is the main security boundary | 06 §7, 08 §4 |
| D-08 | **Two storage roles**: PostgreSQL + TimescaleDB is the system of record (orders, fills, ledger, config, audit, recent candles). A Parquet + DuckDB lake holds history and immutable backtest snapshots. | Decided | Measured at about 17 MB per pair-year of 1m candles; reproducible snapshots | 06 §6 |
| D-15 | **Engine in Python 3.13 with polars (not pandas) and TA-Lib; avoid pandas-ta** (unmaintained). UI in TypeScript/React with Lightweight Charts (attribution required). | Decided | Maintenance status and performance | 06 §8.2 |
| D-16 | **One closed-bar, incremental indicator engine for every mode and feature**, tested against TA-Lib and a truncation (look-ahead) harness. | Decided | Indicator parity is the foundation of sim-live parity | 03 (TA-001, TA-003, TA-028), 06 (AR-026) |
| D-17 | **All data is stored point-in-time**: when it happened and when it could first be known. Revisable data is append-only. | Decided | Prevents look-ahead from revised data | 03 §6, 06 §5 |
| D-18 | **Record live data from P1**, including derivatives and market-wide data that only ships as analysis in P2. | Decided | Free history is short (e.g. Binance serves 30 days of open interest) | 03, 06 |
| D-19 | **Dependency license policy**: shipped code uses only permissive or LGPL-as-library dependencies; GPL/AGPL only as separate, unmodified services; enforced in CI. | Decided | Keeps the distribution option open at near-zero cost | 02 (CAP-044), 06 (NFR-020) |

### 1.3 Trading engine, backtesting and paper trading

| ID | Decision | Status | Why | Source |
|---|---|---|---|---|
| D-05 | **Paper trading uses Ubik's own simulator fed by live public market data**, not an exchange testnet. P1 stores **no** trade-capable exchange keys. | Decided | Testnet prices aren't real; no keys means no key risk in P1 | 04 §8, 08 (SEC-007) |
| D-07 | **P1 strategies are declarative only** (templates and the no-code rule builder). The Python strategy SDK arrives in **P2**, in a sandbox (no network, no secrets, resource limits). | Decided | Resolves 04 (ST-017, Should P1) vs 08 (SEC-034, no user code in P1): safety first, and the sandbox is P2 | 04, 08 |
| D-09 | **One mandatory pipeline in every mode: Strategy → Position sizer → Risk engine → Order management (OMS) → Venue adapter.** Strategies can't reach an exchange. The full OMS (event-sourced, persist-before-send, reconciliation) and double-entry ledger are built in **P1**. | Decided | This is what makes P3 a configuration change rather than a rewrite; it expands the vision doc's "basic risk controls" | 05 §2, §9.3 |
| D-11 | **The event-driven backtester is the only "official" result.** A fast vectorized screener (P2) is for sweeps only; finalists are confirmed event-driven. Fills are conservative by default. | Decided | Accuracy and parity over speed | 04 §4 |
| D-12 | **Honesty machinery from P1**: trial ledger (AI-drafted variants count), locked out-of-sample vault, benchmarks on every report, honesty warnings above the numbers, reproducibility manifests. Overfitting statistics (DSR, PBO), walk-forward and Monte Carlo arrive in P2. | Decided | Ubik's main differentiator | 02 §3, 04 §5–6 |
| D-13 | **Path to live**: S0 Backtest → S1 Paper (≥ 30 days and ≥ 20 round trips) → S2 Exchange demo (≥ 7 days) → S3 Shadow, read-only (≥ 14 days) → S4 Pilot (≤ $500 per strategy) → S5 Scaled. **Promotion needs owner approval; demotion is automatic.** | Decided (thresholds pending the owner's answer to Q-15) | Proves implementation before risking money; the honesty note says it doesn't prove an edge | 04 §8.5, 05 §9.1, 07 §2.4 |

### 1.4 Risk and safety

| ID | Decision | Status | Why | Source |
|---|---|---|---|---|
| D-20 | **The Safe profile is the default in every mode**, including backtests: 1% risk per trade with a 2×ATR(14) stop, max position 20% of a strategy's capital, account daily/weekly loss 3%/7% → reduce-only, account drawdown 15% → reduce-only and 20% → halted. Full table in [05 §2.1](05-risk-portfolio-execution.md). | Decided (limits revisited after Q-14) | Conservative defaults; backtests must show what the risk engine would really allow | 05 |
| D-21 | **Kill switch with three levels** (L1 pause new entries, L2 also cancel all orders, L3 also flatten), reachable from the web UI, the server command line and an authenticated API. One-tap "Pause all" maps to L1. Auto-flatten (L3) on an account halt only if the owner opts in. | Decided | Works even if the UI is down | 05 §4.3, 07 (UX-014), 08 (SEC-005) |
| D-22 | **Friction proportional to risk**: anything that lowers risk is instant; anything that raises it needs confirmation, plus a cooling-off period once real money is involved. Going live is a deliberate **arming** step (re-authentication, knowledge check, 24 h first-time cooling-off). Config files can only disarm. | Decided | Owner can always override, but never on impulse | 07 §7.3, 08 |
| D-23 | **Exchanges and stablecoins are risks, not "the bank" and "cash"**: value everything in the owner's reference currency, cap capital per exchange, monitor stablecoin pegs. | Decided | FTX, the USDC 2023 depeg, USDe in October 2025, the Bitget 2026 hack | 05 §5 |
| D-24 | **Exchange keys (P3)**: trade-only, withdrawal permission checked and refused, IP allow-listed, dedicated sub-account, decryptable only by the gateway, never logged or sent to the AI. | Decided | The 3Commas leak showed even trade-only keys get abused | 08 §5 |

### 1.5 Experience, AI and compliance

| ID | Decision | Status | Why | Source |
|---|---|---|---|---|
| D-25 | **Guided and Pro modes** share one app. Risk information is never hidden in either mode. Users level up by learning and time, never by trade count or profit. | Decided | Beginner friendly without dumbing down | 07 §4 |
| D-26 | **No gamification**: no confetti, streaks, points, leaderboards, "top movers" on Home, or default price-move notifications. | Decided | Regulator and academic evidence that these increase overtrading | 07 §7 |
| D-27 | **Every analysis output carries an evidence badge** (Descriptive / Supported / Mixed / Weak / Untested). No composite "buy score" in Guided mode. | Decided | Describing the market ≠ predicting it | 03 §1.2, 07 |
| D-28 | **The AI assistant arrives in P2** (Claude via the Anthropic API). It explains and drafts; read-only tools plus form prefill; cites sources; monthly spend cap. It never places orders, arms live trading, changes limits or gives personalized buy/sell advice. News and social text given to it is marked untrusted. | Decided | Usefulness without autonomy; prompt-injection risk | 06 (AR-029), 07 §6, 08 §6.6 |
| D-29 | **Paper and live are impossible to confuse**: persistent label, distinct visual treatment, separate ledgers and database schemas, never combined totals. Tax exports contain live trades only. | Decided | The most dangerous UX failure | 07 (UX-031), 08 (CMP-012) |
| D-30 | **Tax-grade records from P1** (append-only, cost basis, fees, fiat value at trade time). Exports in P3: generic CSV/JSON, Koinly and CoinTracker formats. | Decided | Exchanges now report to tax authorities (DAC8, CARF, Form 1099-DA) | 08 §9 |
| D-31 | **Venue defaults by jurisdiction profile**: outside the US, Binance + Coinbase (or Kraken); in the US, Coinbase Advanced + Kraken. Ubik never circumvents geo-blocks. Quote currency (USDT vs USDC vs fiat) follows jurisdiction, because USDT pairs were delisted for EEA users. | Provisional, pending Q-01/Q-02 | Availability and legality differ by country | 06 §2 (AR-001), 08 (CMP-004, CMP-006) |

## 2. Open questions for the owner

Each question has the **default Ubik plans around** until you answer. Answering the "Phase 0" group unblocks development.

### 2.1 Answer during Phase 0 (blocking)

| ID | Question | Why it matters | Default if unanswered |
|---|---|---|---|
| Q-01 | **Which country (and US state, if any) do you live in and pay tax in?** | Decides which exchanges and pairs are legal for you, the quote currency, tax rules and export formats | Non-US, non-EEA profile; USD reference currency; no live trading until answered |
| Q-02 | **Which exchanges do you have verified accounts on? Do they offer sub-accounts?** | Paper trading should use the venue you'll go live on (graduation requires it) | Binance public data for P1 paper trading, Coinbase as second venue |
| Q-03 | **Will anyone else ever use Ubik** (friends, open-source release, paid product, SaaS)? When? | Drives license policy, data-source choices (some are non-commercial), and legal exposure | Personal use only, but keep the permissive-license policy (D-19) |
| Q-04 | **What trading horizon interests you**: minutes, hours, days, or longer? | Fee drag, data needs, fill-model accuracy and which strategies make sense | 1h–1d bars (D-10) |
| Q-05 | **Home machine or VPS? Which country would the server be in? Can you get a static IP for P3?** | Uptime, legal location of the bot, IP allow-listing of keys | Home machine or small VPS for P1; VPS with static IP recommended for P3 |
| Q-06 | **Budgets**: monthly hosting, paid data, AI usage; a one-time legal/tax consult before P3? | Decides free-vs-paid data and the AI spend cap | ≤ $60/month hosting, $0 paid data in P1, AI cap ~$15/month in P2, consult recommended before P3 |

### 2.2 Answer before or during P2

| ID | Question | Default if unanswered |
|---|---|---|
| Q-07 | Do you want to write Python strategies yourself, or are templates and the rule builder enough? | Builder first; Python SDK in P2 (D-07) |
| Q-08 | Which alert channels reliably reach you (Telegram, email, ntfy push, Discord)? Quiet hours and time zone? | In-app plus Telegram; quiet hours 23:00–07:00 in your browser's time zone |
| Q-09 | Are you comfortable sending strategy and journal data (never keys) to Anthropic's API for the assistant? Or prefer a local model? | Opt-in, off until you enable it |
| Q-10 | Starting level: Guided or Pro? Home currency and language? Any color-vision or accessibility needs? Mainly phone or desktop? | Guided; USD; English; color-blind-safe palette available; installable web app (PWA) for phone monitoring |
| Q-11 | Do you want candlestick/chart-pattern tools for learning, labelled "Weak"? Scenario challenges? | Patterns in P2 labelled Weak; challenges in P2 |
| Q-12 | Grid bots in P1? | Yes, with the ST-014 safeguards (range, exit rule, inventory cap) |
| Q-13 | Use an existing tool (e.g. Freqtrade in dry-run mode) to learn while Ubik is built? | Optional; fine for personal use despite its GPL license |

### 2.3 Answer before P3 (real money)

| ID | Question | Default if unanswered |
|---|---|---|
| Q-14 | **What loss in a bad month would make you stop?** What drawdown could you tolerate on real money? | Safe profile limits (D-20); account drawdown halt at 20% |
| Q-15 | Are graduation gates (30 days paper, etc.) hard rules even for you, or overridable after a 24 h delay with a logged reason? | Overridable only in Pro mode after 24 h, never for duration, trade count or same-version rules (PT-015) |
| Q-16 | Roughly how much might run live eventually? Split across two exchanges? | Pilot ≤ $500 per strategy; per-exchange cap set by you before arming |
| Q-17 | If the account drawdown breaker fires at 3 a.m., should Ubik only pause and cancel (L1/L2), or also flatten positions (L3)? | Pause and cancel; no auto-flatten |
| Q-18 | After a server reboot, resume live trading automatically (key held in OS keyring or KMS) or wait for your passphrase? | Wait for passphrase (safer; trading stops until you unlock) |
| Q-19 | Are exchange-side "catastrophe" stops OK (they lock part of your balance)? Do you have passkeys or a hardware security key? Which tax tool or accountant do you use? | Native catastrophe stops on; passkey preferred, TOTP fallback; generic CSV export |

## 3. Top risks (combined)

Sourced from the risk sections of docs 02–08; each doc keeps its detailed list.

| # | Risk | Impact | Main mitigations | Docs |
|---|---|---|---|---|
| R1 | **False confidence**: backtests or paper results look better than reality (overfitting, optimistic fills, look-ahead, survivorship) | Real losses in P3; owner over-trusts Ubik | Conservative fills; trial ledger and OOS vault; DSR/PBO; benchmarks; truncation tests; shadow backtests; staged rollout with tiny pilot | 02, 03, 04, 05 |
| R2 | **Scope creep** from "everything trading firms do" | P1 never ships | Phase tags; explicit non-goals; vertical slices; first-cut list ([09 §7](09-roadmap.md)) | 02, 03, 06, 09 |
| R3 | **Sim-live parity drift** between backtest, paper and live | Graduation evidence invalid | Shared matching engine and indicator engine; parity and golden tests; daily shadow backtest | 04, 06 |
| R4 | **Jurisdiction or exchange changes** (geo-blocks, delistings, API changes) | Venue unusable; data gaps | Jurisdiction profiles; ≥ 2 venues; contract tests; self-recorded data | 06, 08 |
| R5 | **Key theft or account takeover** (P3) | Funds drained through trades | Gateway-only keys; withdrawal refusal; IP and symbol allow-lists; sub-account; alerts on trades Ubik didn't place; kill switch | 05, 08 |
| R6 | **Exchange insolvency, hack, or stablecoin depeg** | Funds frozen or devalued | Per-exchange capital caps; minimal balances; peg monitoring; fiat valuation | 05 |
| R7 | **Operational failure** (crash, home internet down, clock drift) with positions open | Unmanaged positions; duplicate orders | Persist-before-send; reconciliation; dead-man's switch; native catastrophe stops; VPS for live | 05, 06 |
| R8 | **Owner disables guardrails** because they feel restrictive | Safety erodes | Explain every block; friction proportional to risk; hard ceilings stay | 05, 07, 08 |
| R9 | **AI assistant invents numbers** or is prompt-injected | Bad decisions; data leak | Tool-grounded numbers; cited sources; read-only tools; untrusted-content labels; evaluation set | 06, 07, 08 |
| R10 | **Supply-chain compromise** (malicious PyPI/npm packages targeting crypto) | Key theft; rogue orders | Hash-locked dependencies; dependency review; scanning; sandboxed strategy code | 08 |
| R11 | **Regulatory drift** or accidental move from personal tool to regulated service | Legal exposure | Single-owner scope (D-01); legal gate before sharing; review at each phase gate | 08 |
| R12 | **Data source disappears or changes terms** | Analysis cards break; license issues | Source adapters; license registry; graceful degradation; self-recording | 03, 06 |
