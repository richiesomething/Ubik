# Ubik — Requirements Overview

> Status: **Draft v0.3** (2026-10-18; updated after the cross-doc consistency review). This is the index to Ubik's requirements. The requirements themselves live in the domain docs (02–08). This doc shows how they fit together: what P1 must deliver, the rules every feature obeys, the non-functional targets, and how each part of the owner's brief is covered.

## Summary

- **About 500 requirements across 17 areas**, each with an ID, a priority (Must / Should / Could) and a phase (P1–P4). Roughly half are P1, which makes P1 a substantial MVP. [09 Roadmap](09-roadmap.md) splits it into five usable slices and a first-cut list.
- **Ten cross-cutting rules** (§4) matter more than any single feature. Examples: every order passes the risk engine; one strategy codebase and one matching engine serve backtest, paper and live; no look-ahead; exact-decimal money; persist before send; paper and live never mix; every number is explained.
- **The owner's five asks are all covered** (§6): simulated money first and real money later (staged), the professional toolkit (mapped capability by capability in §3), market and technical analysis, beginner friendliness, and honest backtesting.
- **P1 is done** when goals G1–G5 from the vision doc are met and every P1 "Must" has a passing test (§7).

## 1. How requirements are organized

| Field | Meaning |
|---|---|
| **ID** | `PREFIX-NNN`. The prefix identifies the area (table below). IDs are never reused; a dropped requirement is struck through, not deleted |
| **Priority** | **Must**: the phase isn't done without it. **Should**: expected, but can slip one phase with a logged reason. **Could**: nice to have |
| **Phase** | P1 "Paper Trader" · P2 "Analyst" · P3 "Live" · P4 "Advanced" ([00 §6](00-vision-and-scope.md)) |
| **Wording** | "Ubik shall …", written so a test can check it |

| Prefix | Area | Doc |
|---|---|---|
| CAP | Professional capabilities (cross-cutting) | [02](02-landscape-and-pro-capabilities.md) |
| TA, MA, SIG | Technical analysis & charting; market analysis; signals, scanner & alerts | [03](03-market-and-technical-analysis.md) |
| ST, BT, PT | Strategy model & builder; backtesting & optimization; paper trading | [04](04-strategies-backtesting-paper-trading.md) |
| RK, PF, EX | Risk; portfolio & allocation; order management, execution & path to live | [05](05-risk-portfolio-execution.md) |
| DA, AR, NFR | Data; architecture & tech stack; non-functional | [06](06-data-and-architecture.md) |
| UX, ED | Product experience & UI; education & AI assistant | [07](07-ux-and-beginner-experience.md) |
| SEC, CMP | Security; legal, compliance, tax & disclaimers | [08](08-security-legal-compliance.md) |
| HL | Hyperliquid integration (primary venue) | [11](11-hyperliquid-integration.md) |

**When docs overlap**, the doc that owns the topic wins (e.g. 05 owns risk limits, 04 owns fill rules, 08 owns key handling), and [10 Decisions](10-decisions-and-open-questions.md) records any conflict the planning lead resolved.

## 2. Requirement counts

<!-- COUNTS:START -->
| Area | Prefix | P1 (of which Must) | P2 | P3 | P4 | Total |
|---|---|---|---|---|---|---|
| Pro capabilities | CAP | 27 (24) | 10 | 6 | 1 | 44 |
| Technical analysis & charting | TA | 21 (16) | 7 | 0 | 1 | 29 |
| Market analysis | MA | 9 (4) | 17 | 0 | 1 | 27 |
| Signals, scanner & alerts | SIG | 15 (14) | 2 | 0 | 0 | 17 |
| Strategy model & builder | ST | 16 (16) | 6 | 0 | 1 | 23 |
| Backtesting & optimization | BT | 26 (22) | 9 | 0 | 2 | 37 |
| Paper trading | PT | 13 (11) | 1 | 2 | 0 | 16 |
| Risk | RK | 29 (25) | 4 | 7 | 1 | 41 |
| Portfolio & allocation | PF | 7 (5) | 7 | 1 | 0 | 15 |
| Orders, execution & path to live | EX | 15 (15) | 3 | 9 | 1 | 28 |
| Data | DA | 22 (16) | 3 | 0 | 0 | 25 |
| Architecture | AR | 32 (30) | 2 | 3 | 0 | 37 |
| Non-functional | NFR | 18 (14) | 2 | 0 | 0 | 20 |
| Product experience | UX | 49 (45) | 4 | 2 | 1 | 56 |
| Education & AI assistant | ED | 14 (11) | 13 | 1 | 0 | 28 |
| Security | SEC | 16 (16) | 7 | 19 | 0 | 42 |
| Legal, compliance & tax | CMP | 14 (14) | 4 | 8 | 3 | 29 |
| **Total** | | **343 (298)** | **101** | **58** | **12** | **514** |

Phase is the phase in which a requirement first applies (rows marked "P1 (Must P2)" count as P1). Generated from the requirement tables in docs 02–08 on 2026-10-18; regenerate after edits.
<!-- COUNTS:END -->

## 3. "Everything pro trading firms do": capability coverage

[02 §1](02-landscape-and-pro-capabilities.md) walks the professional trade lifecycle. This table shows where each capability lands in Ubik. **Ubik copies the professional *process*; it doesn't attempt the professional *infrastructure*** (co-location, VIP fee tiers, HFT).

| Pro capability | What Ubik does | Phase | Main requirements |
|---|---|---|---|
| **C1 Data infrastructure** | Live and historical data with quality checks, explicit gaps, point-in-time storage, provenance on every dataset, self-recorded data | P1 (more sources P2) | DA-001–DA-022, TA-026, CAP-001 |
| **C2 Research & alpha generation** | Event-driven backtester with realistic costs; trial ledger; out-of-sample vault; benchmarks; reproducibility. Walk-forward, optimization, overfitting statistics and Monte Carlo come in P2 | P1 → P2 | BT-001–BT-037, CAP-002–CAP-011 |
| **Market & technical analysis** | ~20 core indicators, multi-timeframe charts, regime and volatility labels, Market Health Summary in P1. Derivatives, on-chain, sentiment, macro, correlations and scanner in P2. Every output carries an evidence badge | P1 → P2 | TA-*, MA-*, SIG-* |
| **C3 Signal combination** | Rule builder with ALL/ANY logic in P1; weighted/voted combinations with each signal's contribution in P2; ML in P4 | P1 → P4 | ST-015, CAP-012 |
| **C4 Portfolio construction & capital allocation** | Strategy sleeves with manual or equal-weight allocation and a cash reserve in P1; risk parity, volatility scaling, correlation clusters, VaR/ES and multi-strategy backtests in P2 | P1 → P2 | PF-001–PF-013, CAP-013, CAP-014 |
| **C5 Pre-trade & real-time risk** | Risk engine no strategy can bypass; layered limits (Careful risk profile by default); position sizing; loss limits and drawdown breakers; three-level kill switch; stale-data handling (no new entries) and clock-drift checks | P1 | RK-001–RK-040, CAP-015–CAP-017 |
| **C6 Execution & transaction cost analysis (TCA)** | Real fee schedules and exchange rules; protected market orders; per-order TCA (implementation shortfall) from paper fills onward; limit-chase, TWAP and iceberg later; smart order routing in P4 | P1 → P4 | EX-009–EX-021, CAP-019–CAP-023 |
| **C7 Post-trade analytics & attribution** | Honest reports, journal with a reason for every action, backtest-vs-paper re-simulation; P&L attribution (market, strategy, fees, slippage) in P2 | P1 → P2 | BT-034–BT-036, UX-016, CAP-024, CAP-025 |
| **C8 Monitoring, operations & incident response** | Health checks, freshness monitor, external heartbeat monitor (P1), dead-man's switch (P3), incident cards, backups; runbooks and two-channel live alerting in P3 | P1 → P3 | AR-032, CAP-026, UX-007, EX-026, EX-027 |
| **C9 Research governance** | Versioned strategies; lifecycle with objective promotion gates and pre-declared kill criteria; automatic demotion; graduation scorecard | P1 → P3 | CAP-031–CAP-035, EX-022, PT-015 |
| **C10 Treasury & counterparty management** | Reference-currency valuation, stablecoin peg monitoring, per-exchange capital caps, withdrawal-free keys, dedicated sub-accounts | P1 → P3 | RK-028–RK-031, CAP-018, CAP-036, SEC-008–SEC-016 |

**Deliberately not attempted:** co-located HFT, latency arbitrage, live market making at retail fee tiers, MEV, and autonomous AI trading ([02 §1.4](02-landscape-and-pro-capabilities.md)).

## 4. Cross-cutting rules

Every feature, in every phase, obeys these. They are the first thing to check in a design or code review.

| # | Rule | Requirements |
|---|---|---|
| 1 | **One pipeline, no bypass.** Every order, whether from a strategy, the manual UI or an AI proposal, goes Strategy → Sizer → Risk engine → OMS → Venue adapter, in all modes. Strategies never touch exchange APIs or credentials. | RK-001, RK-002, SEC-002, CAP-015, AR-005 |
| 2 | **Sim-live parity.** One strategy interface, one indicator engine, one matching engine (`SimVenue`) and one risk profile across backtest, paper and live. Only the clock, data feed and venue adapter change. | ST-003, TA-001, BT-001, PT-001, AR-011, AR-012, RK-006 |
| 3 | **No look-ahead.** Strategies, alerts and scans see only closed bars and data whose `available_at` ≤ decision time. Repainting constructs only in confirmed form. Truncation tests in CI. | TA-002, TA-010, TA-026, TA-028, ST-004, DA-010 |
| 4 | **Exact money.** Decimal arithmetic everywhere money or quantities appear; double-entry ledger; invariants checked after every event. | BT-015, EX-009, EX-012, DA-018, NFR-007 |
| 5 | **Persist before send, never duplicate.** Orders are saved with a deterministic client ID before any network call; ambiguous results are queried, never blindly resubmitted; continuous reconciliation. | EX-002–EX-007, AR-015, NFR-006 |
| 6 | **Reproducible.** Every run stores a manifest (data snapshot, strategy version, config, seed, engine version) and re-runs bit-identically. | BT-016, DA-021, NFR-010, CAP-002 |
| 7 | **Honest by default.** Costs always on; benchmark and out-of-sample result on every report; provenance labels; honesty warnings above the numbers; hypothetical-performance disclaimers; no hype words. | BT-030, BT-035, BT-036, CAP-039, CMP-018, ED-004, ED-006 |
| 8 | **Explain everything.** Every metric, indicator, setting, warning and blocked trade has a plain-language explanation tied to the glossary. | ED-001, RK-038, SIG-011, UX-008, ED-025 |
| 9 | **Paper and live never mix.** Separate ledgers and schemas, persistent PAPER/LIVE labels, never-combined totals, live-only tax exports. | PT-013, UX-031, CMP-012, SEC-004 |
| 10 | **Friction proportional to risk; secrets nowhere.** Risk-reducing actions are instant, risk-increasing ones need confirmation (and cooling-off when live). No secret ever reaches logs, the browser or the AI. | UX-030, SEC-014, SEC-007 |

## 5. Non-functional targets (P1 unless noted)

Owned by [06 §11](06-data-and-architecture.md) and the docs named in the last column.

| Quality | Target | Requirement |
|---|---|---|
| Decision latency | Finalized candle → strategy decision ≤ 250 ms p95 for ≤ 20 instruments | NFR-001 |
| Order latency | Approved intent → order leaves Ubik ≤ 300 ms p95 (excluding network) | NFR-002 |
| Availability | Engine ≥ 99.5% per month in P1; ≥ 99.9% for live on a VPS | NFR-004 |
| Recovery | Reconnect ≤ 5 s p95, backfill ≤ 2 min; zero duplicate orders and zero lost fills after any crash | NFR-005, NFR-006 |
| Data durability | P1: nightly off-site backups, lose at most 24 h (RPO), restore within 4 h (RTO), one tested restore before P1 exit. P3: RPO ≤ 5 min, RTO ≤ 1 h, quarterly restore drills | NFR-012, AR-031 |
| Backtest speed (target) | 1 instrument × 1 year of 1m candles in ≤ 60 s on 4 vCPU / 8 GB | NFR-008, BT-018 |
| Reproducibility | Byte-identical trades and metrics for the same manifest | NFR-010 |
| Time | Host clock offset ≤ 100 ms; venue offset warning > 500 ms, new orders denied > 1,000 ms | NFR-013, RK-027 |
| Footprint | 20 instruments + 10 paper bots on 4 vCPU / 8 GB / 100 GB SSD | NFR-014 |
| Portability | Linux, macOS, Windows (Docker); amd64 and arm64; no cloud-vendor lock-in | NFR-015 |
| Alerting | Owner alerted within 2 min of engine death, stale data while bots are running or positions are open, ledger mismatch or disk > 85% (live: always; paper: outside quiet hours) | NFR-017 |
| Rate limits | Use at most 70% of every venue's published limits; zero IP bans | NFR-018, EX-006, AR-008 |
| Accessibility | WCAG 2.2 AA; text and table alternatives for charts | UX-045, UX-046 |
| Security baseline | Login + MFA even single-user; private-by-default network; hash-locked dependencies; append-only hash-chained audit log | SEC-017–SEC-031 |
| Privacy | No telemetry by default; data inventory; opt-in AI data sharing | CMP-024, ED-009–ED-015 |

## 6. Traceability: the owner's brief → requirements

| The owner asked for… | Where it's covered | Key requirements |
|---|---|---|
| "Make the bot compatible with Hyperliquid … the primary way we're going to interact with crypto" | Hyperliquid as primary venue: native client, agent-key security, data recording and proxy history, ≤ 1× positions, jurisdiction gate, path to live | HL-001–HL-054, D-45–D-55 |
| "A crypto trading bot" | Strategy engine running 24/7 with templates, a rule builder and (P2) Python | ST-001–ST-022, AR-010 |
| "For now we won't trade real money … later we'll make it work with real money" | Paper trading on live data with no keys (P1); sim-live parity so live is configuration, not a rewrite; staged path to live with tiny-capital pilot (P3) | PT-001–PT-016, EX-022–EX-028, SEC-007, CAP-035 |
| "Everything trading companies do to optimize trading, e.g. market analysis and technical analysis" | Capability map §3: data, research rigor, analysis, signal combination, portfolio, risk, execution/TCA, attribution, operations, governance, treasury | CAP-*, TA-*, MA-*, SIG-*, RK-*, PF-*, EX-* |
| "Beginner friendly" | Guided/Pro modes, onboarding, plain-language explanations of everything, Learn hub, honest framing, no gamification, AI assistant (P2) | UX-*, ED-*, glossary |
| "Back test trading strategies to see how they would work" | Event-driven backtester with realistic fills and costs; verdict and trust check; P2 walk-forward, overfitting statistics, Monte Carlo and stress tests | BT-001–BT-037, CAP-002–CAP-011 |

## 7. P1 definition of done

P1 is complete when all of the following hold. The acceptance check for each goal is the test that proves it.

| Goal ([00 §4.1](00-vision-and-scope.md)) | Acceptance check | Main requirements |
|---|---|---|
| **G1** First meaningful backtest quickly | Moderated test: a first-time user goes from first launch to a template backtest with a plain-language summary in ≤ 10 minutes, unaided | UX-001, UX-004, ED-002 |
| **G2** Backtests are trustworthy | Golden tests pass in CI; re-running any saved run is byte-identical; every report shows costs, benchmark and out-of-sample result | BT-016, BT-030, BT-035, CAP-006, NFR-010 |
| **G3** Paper behaves like real trading would | 30-day unattended soak with ≥ 3 strategies; daily parity replay within tolerance (signal match ≥ 95%, mean fill difference ≤ 1.5× modeled slippage); chaos tests show no duplicate or lost orders | PT-009, PT-011, NFR-006 |
| **G4** Risk controls always hold | Automated tests: every limit-violating order is blocked in all modes; Pause all (L1) is one tap away and the kill switch (L2) halts and cancels all open orders within 10 s | RK-001–RK-024, SEC-005 |
| **G5** Everything is explained | Automated check: every metric, indicator and warning has a glossary entry and an explanation | ED-001, ED-025 |
| **All P1 Musts** | Each P1 "Must" in docs 02–08 has at least one linked test or review item | — |

G6 (live trading is a configuration change) is the P3 exit criterion.

## 8. Changing requirements

1. Edit the requirement in its owning domain doc. Never reuse an ID; strike through dropped ones.
2. If the change resolves a conflict or changes scope, add or update a decision in [10](10-decisions-and-open-questions.md).
3. If it changes a phase, update the slice tables in [09](09-roadmap.md).
4. Re-run the requirement count (§2).
