# Risk Management, Portfolio Construction & Execution

*Ubik planning doc 05 · Draft v0.1 · 2026-10-18 · ID prefixes: **RK** (risk), **PF** (portfolio & capital allocation), **EX** (order management, execution, path to live)*

## Summary

- **One mandatory pipeline in every mode.** Every order goes Strategy → Position sizer → **Risk engine** → **Order management system (OMS)** → Venue. Strategies never touch an exchange connection. Only the venue changes between backtest, paper and live: `SimVenue` for backtest and paper, `LiveVenue` for exchange demo and live (names per doc 06). Build this pipeline fully in P1, even with fake money: that is what makes P3 a configuration change instead of a rewrite.
- **Layered, explainable limits with conservative defaults.** Limits apply per order, per strategy, per asset or correlated cluster, per account and per exchange. Trading can be ACTIVE, REDUCE_ONLY or HALTED, and a kill switch still works when the web UI is down. Every blocked trade gets a plain-language reason.
- **Default sizing (Careful profile): risk 1% of the strategy's capital per trade, with the stop 2×ATR away, and no position larger than 20% of that capital.** The looser Balanced and Adventurous risk profiles allow more risk per trade, but the hard ceilings are the same in every profile. No leverage before P4. Kelly sizing is a Pro-mode option, fractional, capped and still inside the risk profile's limits.
- **The exchange and the stablecoin are risks, not "the bank" and "cash."** Value everything in USD (or the owner's fiat), cap the capital on each exchange, and watch stablecoin pegs. Precedents: USDC hit $0.87 in 2023, USDe hit about $0.65 on Binance in October 2025, and Bitget froze withdrawals after a hack in September 2026.
- **A professional-grade OMS from day one.** That means event-sourced orders, deterministic client order IDs, an explicit UNKNOWN order state, continuous reconciliation, exact-decimal double-entry accounting, and real exchange precision and fee rules in every mode.
- **Measure real trading costs and go live in stages.** Transaction cost analysis (TCA) on every paper and live fill shows assumed vs observed costs from P1 and, from P2, calibrates backtest cost assumptions. The route to real money is paper → exchange demo → shadow → tiny-capital pilot → scaled, with written promotion and demotion rules.

---

## 1. Scope and professional context

This doc covers everything between "a strategy wants to trade" and "the trade is booked and reconciled." Other docs cover the neighbouring pieces:
- **04:** strategy logic, the backtest fill model, and metrics.
- **03:** indicator math, such as ATR and correlation.
- **06:** exchange connectivity and storage.
- **07:** screens and wording.
- **08:** API-key security, authentication and tax exports.
- **02:** build-vs-reuse. My input there: own the risk engine, OMS and ledger, because they are the safety surface, and reuse exchange plumbing (CCXT or native SDKs).

**What professional firms do.** US brokers must run automated *pre-trade* checks before an order reaches a market: capital thresholds and "erroneous order" filters, under SEC Rule 15c3-5. The cautionary tale is Knight Capital. In 2012 a botched deployment sent more than 4 million unintended orders in 45 minutes, for a loss of about $460M (per the SEC). The SEC [order](https://www.sec.gov/litigation/admin/2013/34-70694.pdf) cites missing automated controls and kill-switch procedures. Firms also keep the risk function independent of the traders, monitor exposure in real time, reconcile with their brokers every day, and measure execution quality. Ubik copies these practices at individual scale. Open-source precedents include [NautilusTrader's RiskEngine](https://nautilustrader.io/docs/latest/api_reference/risk), with ACTIVE/REDUCING/HALTED states, per-instrument notional caps and an `OrderDenied` event, and [Freqtrade's protections](https://www.freqtrade.io/en/stable/plugins/), such as StoplossGuard, MaxDrawdown and CooldownPeriod.

**Terms used below.**
- **Notional:** the money value of an order or position (price × quantity).
- **bps (basis point):** 0.01%.
- **Drawdown:** the fall from the highest equity reached so far.
- **Reference drawdown:** the worst drawdown in the strategy version's full backtest (development set plus vault), net of costs, under its risk profile. Breakers, the S4 gate and demotion are set as multiples of it.
- **Round trip:** one entry plus its matching exit. Every trade count in this doc's gates and evidence counts round trips, not orders or fills.
- **Bot:** one strategy version deployed on one account. Each bot owns exactly one sleeve (§6).
- **ATR (Average True Range):** the average size of a bar's price range, a simple volatility gauge.
- **Maker / taker:** a maker order rests in the order book, usually at a cheaper fee. A taker order executes immediately.
- **Slippage:** the gap between the expected price and the price actually received.

---

## 2. Risk architecture

```
 market data ──► Strategy ──(intent: "long BTC, stop 56,400")──► Position sizer
                                                                     │ order request
                                                                     ▼
        ┌───────────────────────────── RISK ENGINE ─────────────────────────────┐
        │ trading state → instrument rules → order → strategy → asset/cluster → │
        │ account → exchange → data & clock health  ⇒  APPROVE / REDUCE / DENY  │
        └───────────────────────────────────┬──────────────────────────────────┘
                                            ▼ approved order (carries decision ID)
                              Order Management System (OMS)
                                            ▼
                         Venue (ExecutionVenue port, doc 06):
          SimVenue (backtest, paper) · LiveVenue → execution gateway (demo, live)
                                            ▼ acks, fills, balances
        Ledger & positions ──► post-trade monitors (loss limits, drawdown, depeg,
                               staleness, reconciliation) ──► trading state / kill switch
```

How the pipeline is enforced:
- **No bypass.** Strategies get a context object that can emit intents and read data. It has no access to the OMS, the venue adapter or credentials. The OMS refuses any order without a risk-decision ID.
- **Three outcomes.** The engine can APPROVE, REDUCE (downsize only, never upsize) or DENY. Each decision is stored with a snapshot of its inputs, which drives explanations and post-mortems.
- **Three trading states.** **ACTIVE**; **REDUCE_ONLY** (only orders that shrink a position); **HALTED** (cancels only). States apply globally, per exchange, per strategy or per symbol, and the most restrictive wins. Cancels always pass. Amendments that increase risk are checked like new orders.
- **Same rules in every mode.** The same engine and limits run in backtest and paper. A backtest that ignores the daily loss limit tells a beginner a story that live trading won't repeat.
- **Defense in depth (P3).** The execution gateway (doc 06), which alone talks to the exchange in live mode, separately enforces a few "last-line" checks from its own config: symbol allow-list, max order notional, price band, order rate and the S4 pilot caps. One bug can't cause a catastrophe alone.
- **Soft and hard limits.** Soft limits come from the risk profile (§2.1) and are adjustable in the UI within it. Hard ceilings, such as leverage = 1×, live only in a server config file. Tightening applies instantly. Loosening is deliberate.
- **Fast.** Target < 5 ms per check, so speed is never a reason to bypass it.

### 2.1 Limit layers, defaults and risk profiles

| Layer | Check | Default (Careful profile) |
|---|---|---|
| Instrument | Exchange filters: tick, step, min/max quantity, minimum order value, price bands | Always on, same in all modes |
| Order | Max order value | 25% of strategy capital, plus an absolute cap (P3 pilot: $100) |
| Order | "Fat-finger" band: how far a limit price may cross the mid-price | 1% BTC/ETH, 2% others |
| Order | Expected slippage of a market order, estimated from the live order book | ≤ 0.5%, else protected IOC (EX-015) or deny |
| Order | Duplicate guard | Same symbol, side and size within 10 s → deny |
| Strategy | Capital sleeve (PF-001) / max open positions | Owner-set / 3 |
| Strategy | Runaway guard | ≤ 10 orders/min, ≤ 200/day, turnover ≤ 5× sleeve/day |
| Strategy | Daily loss | 5% of sleeve → REDUCE_ONLY and cancel resting entry orders until 00:00 UTC |
| Strategy | Drawdown breaker | Smaller of 20% and 1.5 × reference drawdown → HALTED (manual re-arm with a note) |
| Strategy | Cooldown | 3 bars after a stop-out; 24 h pause after 3 consecutive losses |
| Asset | Share of equity in one coin | BTC/ETH ≤ 50%, others ≤ 25% |
| Cluster | Coins that move together count as one bet | "Crypto majors" ≤ 75% of equity |
| Account | Daily / weekly loss | 3% / 7% → REDUCE_ONLY and cancel resting entry orders until the period resets |
| Account | Drawdown breaker | 15% → REDUCE_ONLY and cancel resting entry orders; 20% → HALTED with review |
| Account | Leverage, negative balances | Not allowed (spot only, hard ceiling) |
| Exchange | Capital held on the exchange | Owner-set USD cap (live) |
| System | Stale data (§4.4) / clock drift / stablecoin peg | 10 s / 1,000 ms / 1% for 5 min |

Crypto trades 24/7, so "daily" resets at 00:00 UTC unless the owner picks a local timezone. Loss limits use **marked-to-market** equity, meaning open positions are valued at current prices, so a large open loss counts. Cancelling resting entry orders never touches protective stops and exits: they keep resting.

**Reference drawdown and the 20% cap.** The strategy drawdown breaker uses the reference drawdown (§1): the worst drawdown in the version's full backtest, development set plus vault, net of costs. The full backtest is used because a short vault alone gives a noisy, too-small reference. The owner's questionnaire answer (07 §2.2, Q5) and any per-strategy setting may only **lower** the 20% cap, never raise it. The same breaker is the default kill criterion that each strategy declares before S1 (02 CAP-033); the owner may declare stricter ones.

**Risk profiles.** Soft limits come from a versioned **risk profile** (RK-005). There are three. Every new install starts on **Careful** in every mode, including backtests and paper, and in both Guided and Pro mode. The UI mode changes presentation and defaults only, never limits or gates (07 UX-026).

| Profile | Risk per trade | Stops | Other soft limits | Gate overrides |
|---|---|---|---|---|
| **Careful** (default) | Default 1%, adjustable 0.25–1% | Mandatory | The table above | None |
| **Balanced** | Up to 2% | Mandatory | The table above | None |
| **Adventurous** | Up to the 5% hard ceiling (RK-007) | Strategies without stops allowed with an explicit, logged acknowledgement | The table above | Limited S1 overrides, per 04 PT-015 |

Hard ceilings are identical in all three profiles. Moving to a looser profile requires the one-time acknowledgement flow in 07 UX-053: knowledge checks reused from onboarding, the consequence shown in money, and typed confirmation; in live mode the SEC-018 cooling-off also applies (RK-008). Moving to a tighter profile is instant.

**Allocation-style templates (DCA and rebalancing).** These hold a planned share of the sleeve by design and have no stop. Their max position is the template's declared target weight, which replaces the 20%-of-sleeve cap but stays within the asset caps (BTC/ETH ≤ 50%, others ≤ 25%) and the cluster cap (≤ 75%). Initial allocation and rebalance trades are split into orders that each respect the max order value. Instead of a stop, they rely on the strategy drawdown breaker plus a mandatory worst-case display before enabling, as 04 ST-014 does for grids. Benchmark lines such as buy-and-hold and DCA in reports are computed outside the risk profile and labelled as reference lines.

### 2.2 Requirements: risk architecture

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-001 | Ubik shall route every order-creating or order-amending command, in all modes, through the risk engine. The OMS shall refuse orders lacking a valid risk-decision ID. | Must | P1 | Core invariant. Test by injecting an unstamped order. |
| RK-002 | Strategy code shall have no access to venue adapters, credentials or OMS submit functions. | Must | P1 | Bypass-proof by construction. |
| RK-003 | The risk engine shall return APPROVE, REDUCE (downsize only) or DENY with a reason code and plain-language message, and persist each decision with an input snapshot. | Must | P1 | Explainability, audit. |
| RK-004 | Ubik shall support ACTIVE / REDUCE_ONLY / HALTED states at global, exchange, strategy and symbol scope. The most restrictive state wins; cancels are always allowed. | Must | P1 | |
| RK-005 | Ubik shall enforce the layered limits of §2.1 from a versioned risk profile (Careful, Balanced or Adventurous; Careful by default). It shall log every profile change, and every backtest, paper run and live session shall record the profile version it used. | Must | P1 | Static clusters in P1; dynamic clusters in PF-008. |
| RK-006 | The backtester and paper trader shall apply the same risk profile as live. Backtest reports shall list orders reduced or denied, by reason. | Must | P1 | Honesty principle (04). |
| RK-007 | Hard ceilings (max leverage, absolute max order value, max risk per trade of 5%) shall be identical in every risk profile and changeable only in the server config file followed by a restart. | Should | P1 | |
| RK-008 | In live mode, loosening any limit, moving to a looser risk profile or raising an allocation shall require typed confirmation and take effect only after a cooling-off period (default 24 h, configurable 1–24 h, per 08 SEC-018). Shortening the cooling-off is itself a risk-increasing change and takes effect only after the current delay. Tightening applies immediately. | Must | P3 | Guards against "tilt" (emotional decisions). First arming of live trading has a fixed 24 h cooling-off (SEC-018). |
| RK-009 | The execution gateway (doc 06) shall independently enforce last-line checks from its own config, separate from strategy and risk-profile config: symbol allow-list, max order notional, price band, order rate and the S4 pilot caps (EX-025). | Must | P3 | Defense in depth. `LiveVenue` is only the gateway's engine-side client. |

---

## 3. Position sizing

Sizing answers "how much?", and it often matters more than "when?".

| Method | Plain language | Formula (simplified) | Verdict |
|---|---|---|---|
| Fixed amount | "Always buy $500 worth." | qty = amount / price | First experiments only; ignores volatility |
| Fixed fraction | "Always use 10% of the strategy's money." | qty = f × capital / price | Simple; ignores where the stop is |
| **Fixed-fractional risk** | "If my stop is hit, I lose at most 1% of the strategy's money." | qty = r × capital / (entry − stop) | **Default** |
| **ATR stop** | The above, with the stop k×ATR away. Wild markets get a wider stop and therefore a smaller position. | stop = entry − k × ATR | **Default** (k = 2) |
| Volatility targeting | Size so each position swings about a target amount, e.g. 20% a year. BTC often runs 40–70% a year, so the result is roughly a third to a half of the sleeve. | weight = target vol / asset vol | P2; cap weights, since volatility jumps in crises |
| Fractional Kelly | Kelly is the "growth-optimal" bet for a *known* edge. Real edges are uncertain, and full Kelly can halve an account. | f\* = W − (1−W)/R; use ≤ 0.25 f\* | Pro mode only, P2; still capped by the risk profile |

**Worked example.** The BTC strategy's sleeve holds $5,000. BTC is $60,000 and daily ATR(14) is $1,800. The stop goes 2×ATR below entry, at $56,400. Risk is 1% = $50, so qty = $50 / $3,600 = 0.0139 BTC, a notional of about $833 (17% of the sleeve, under the 20% cap). Ubik says: *"If BTC falls to $56,400 this trade loses about $50 plus fees, which is 1% of this strategy's money."* Quantity is rounded **down** to the exchange's step size. If that falls below the exchange minimum, the trade is skipped and explained, never rounded up.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-010 | Ubik shall provide fixed-amount, fixed-fraction, fixed-fractional-risk and ATR-stop sizing, per strategy, identical in all modes. | Must | P1 | |
| RK-011 | Ubik shall default to 1% risk per trade, a 2×ATR(14) stop and a max position of 20% of the sleeve. Risk per trade shall be adjustable only within the active risk profile's range: 0.25–1% under Careful, up to 2% under Balanced, and up to the 5% hard ceiling (RK-007) under Adventurous. | Must | P1 | Same in Guided and Pro mode. Allocation-style templates use their target weight instead of the 20% cap (§2.1). |
| RK-012 | Before enabling a strategy, Ubik shall show the position size, money at risk per trade, and the loss after 10 consecutive stop-outs. | Must | P1 | Makes risk tangible (07). |
| RK-013 | Ubik shall provide volatility-target sizing with a configurable target, a lookback, and per-asset max weights. | Should | P2 | |
| RK-014 | Sizing shall round quantities down to the step size and skip, with an explanation, any trade below the venue minimum. | Must | P1 | Rounding up silently adds risk. |
| RK-015 | Fractional Kelly (≤ 0.25×) shall be offered only in Pro mode, stay within the active risk profile's risk-per-trade limit, and require ≥ 100 closed paper/live round trips, with an estimation-error warning. | Could | P2 | |

---

## 4. Protective mechanisms

### 4.1 Stops: exchange-native vs bot-managed

A **stop-loss** exits when price moves against you by a set amount. A **take-profit** exits at a target. A **trailing stop** follows a rising price and exits on a pullback of set size.

| | Exchange-native (rests at the exchange) | Bot-managed (Ubik watches and sends the exit) |
|---|---|---|
| Works if Ubik or its internet is down | **Yes** | No |
| Flexibility (time, indicator or partial exits) | Limited to the venue's order types | **Full** |
| Side effects | Locks the balance and counts toward order caps (Binance `MAX_NUM_ALGO_ORDERS` [filter](https://developers.binance.com/docs/binance-spot-api-docs/filters)) | None until triggered |
| In a crash | Stop-*market* fills but can slip badly. Stop-*limit* may not fill at all. | Same, plus Ubik's reaction delay |

**Recommendation: hybrid.** Ubik manages the precise stop. In live mode it also rests a wider native **"catastrophe stop"** (about 1.5× the distance) so positions survive a Ubik outage. Backtest and paper simulate both. Prefer stop-market for protective exits. Where only stop-limit exists, as on Coinbase spot, set a wide limit offset, because a stop that doesn't fill protects nothing. Over $19B of leveraged positions were liquidated in about a day on [10 Oct 2025](https://www.coingecko.com/learn/october-10-crypto-crash-explained), and order books thinned dramatically.

### 4.2 Breakers and cooldowns

Loss limits, drawdown breakers, cooldowns, maximum positions and concentration caps use the defaults in §2.1. Breakers have **hysteresis** so they don't flap on and off around the threshold. A drawdown HALT stays until a human re-arms it with a note, for example "reviewed; reducing size." Daily and weekly REDUCE_ONLY states clear automatically when the period rolls over.

From P2 a strategy can also opt into an **event guardrail** (RK-041): around high-importance events in doc 03's event calendar (MA-019), such as a central-bank rate decision, it goes REDUCE_ONLY or scales down new entries.

### 4.3 Kill switch

| Level | Action | Access and timing | Typical trigger |
|---|---|---|---|
| **L1 Pause** | Global REDUCE_ONLY, and cancel resting entry (position-increasing) orders. Protective stops and exits keep working. | "Pause all" in the global bar: ≤ 1 click/tap, no confirmation | Manual ("Pause all") |
| **L2 Cancel** | L1, plus cancel all open orders on all venues, verified by re-querying (RK-024) | ≤ 2 clicks/taps, no confirmation. Completes within 10 s in paper and live. | Manual; runaway guard; suspected key compromise |
| **L3 Flatten** | L2, plus close all positions into quote or fiat using protected, sliced orders | ≤ 2 clicks/taps plus an explicit confirmation. Starts within 10 s and reports progress until flat. | Manual by default. Auto on account HALT only if the owner opts in. Available in paper from P1. |

L2 is what 08 SEC-005 and 02 CAP-017 describe. Loss limits and drawdown breakers set trading states (§2.1); they are not kill-switch levels. The kill switch is reachable from the web UI, the host CLI and an authenticated API (08), and it runs with priority over strategy evaluation. Re-arming is manual. In live mode (P3) a **dead-man's switch** cancels resting entry orders if the engine's heartbeat stops (RK-025). Kraken offers this natively as [`cancel_all_orders_after`](https://docs.kraken.com/api-reference/order-management/dead-mans-switch) (refresh every 15–30 s with a 60 s timeout). Where a venue has no native mechanism, a local **watchdog** process does the cancel. Both differ from the **external heartbeat monitor** (§9.2), an off-host service that only alerts the owner.

### 4.4 Stale data and clock drift

BTC/ETH data on major exchanges updates many times per second, so silence is a signal. **Stale** is a data-health flag on a symbol, venue or feed, not a trading state: there are still only three trading states (RK-004). This table is the single source for stale-data handling; docs 04, 06 and 07 refer to it.

| Condition | Trading effect | Owner alert |
|---|---|---|
| Symbol stale: > 10 s without trades or quotes, a gap in the order-book sequence, or a missing 1m bar | New entries denied. Cancels always pass. Exits only as protected orders priced from a fresh reference, so bot-managed exits in practice wait for fresh data; native catastrophe stops (P3) are unaffected. Strategies skip incomplete bars, and the paper venue produces no fills against stale prices. | The in-app freshness indicator updates immediately |
| Venue-wide staleness > 30 s | Venue REDUCE_ONLY | In-app, immediately |
| Staleness lasting 2 min while bots are running or positions are open | No further change | External alert (06 NFR-017): Warning in paper (held during quiet hours), Critical in live (bypasses quiet hours) |
| Candle-feed gap (06: candle age > 2 × timeframe + grace) | None by itself: an ingest-quality flag, not the trading rule | Per doc 06 |

Exchanges reject signed requests with skewed timestamps; Binance checks `recvWindow` again just before matching ([changelog](https://developers.binance.com/docs/binance-spot-api-docs/CHANGELOG)). Ubik therefore measures its clock offset against exchange server time and requires NTP time sync on the host.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-016 | Every entry shall carry a defined protective exit. The Careful and Balanced profiles shall forbid strategies without stops; the Adventurous profile shall allow them only with an explicit, logged acknowledgement. | Must | P1 | Same in Guided and Pro mode. Allocation-style templates (DCA, rebalancing) rely on the drawdown breaker plus a worst-case display instead (§2.1). |
| RK-017 | Ubik shall support bot-managed stop-loss, take-profit, trailing and time exits in all modes, evaluated on each price update (bars in backtest, per 04). | Must | P1 | |
| RK-018 | In live mode Ubik shall rest a native catastrophe stop (default 1.5× the bot-stop distance) where the venue supports one. Paper and backtest shall simulate it. | Should | P3 | Designed in P1. |
| RK-019 | Ubik shall enforce daily loss limits at strategy and account scope, and weekly loss limits at account scope, on marked-to-market equity. A breach shall set REDUCE_ONLY on that scope and cancel its resting entry orders until the period resets; protective stops and exits keep resting. | Must | P1 | §2.1 defaults. |
| RK-020 | Ubik shall enforce strategy and account drawdown breakers per §2.1: a strategy is HALTED at the smaller of 20% and 1.5 × its reference drawdown; an account goes REDUCE_ONLY at 15% and HALTED at 20%. The owner's questionnaire answer (07 §2.2, Q5) or a per-strategy setting may only lower the 20% cap. Clearing a HALT requires a manual re-arm with a note. | Must | P1 | The strategy breaker is the default kill criterion (02 CAP-033). |
| RK-021 | Ubik shall enforce re-entry cooldowns after stop-outs and pauses after consecutive losses (defaults §2.1). | Should | P1 | |
| RK-022 | Ubik shall enforce max open positions (3 per strategy, 6 per account) and the per-asset and cluster caps in §2.1. | Must | P1 | |
| RK-023 | Ubik shall provide kill-switch levels L1–L3 (§4.3), reachable from the web UI, CLI and authenticated API, and running with priority over strategy evaluation. L1 shall take ≤ 1 click/tap with no confirmation; L2 ≤ 2 clicks/taps with no confirmation, completing within 10 s; L3 ≤ 2 clicks/taps plus an explicit confirmation, starting within 10 s. | Must | P1 | Works in paper from day one, including L3. |
| RK-024 | After L2/L3, Ubik shall verify open orders = 0 and positions are within dust tolerance (leftovers below the minimum order size), report the result, and alert on failure (Critical in live, Warning in paper; 07 §8). | Must | P1 | |
| RK-025 | In live mode Ubik shall cancel resting entry orders when its heartbeat is missing for > 60 s, using native cancel-after where available, else a local watchdog process. | Should | P3 | Dead-man's switch. Not the external heartbeat monitor (06 AR-032), which only alerts. |
| RK-026 | Ubik shall handle stale data per the §4.4 table: on a stale symbol (> 10 s without trades or quotes, a book-sequence gap, or a missing 1m bar) deny new entries, always pass cancels, and allow exits only as protected orders priced from a fresh reference; set the venue REDUCE_ONLY after > 30 s of venue-wide staleness; and alert externally after 2 min of staleness while bots are running or positions are open. | Must | P1 | Thresholds configurable per pair. "Stale" is a data-health flag, not a trading state. |
| RK-027 | Ubik shall measure clock offset to each venue every 60 s, warn above 500 ms, and deny new orders above 1,000 ms. | Must | P1 | Host-vs-exchange offset. NTP required; the host-vs-NTP limit is 06 NFR-013. |
| RK-041 | Ubik shall offer an optional per-strategy event guardrail: within a configurable window around high-importance events in the MA-019 calendar (default 2 h before to 1 h after), it shall either set the strategy REDUCE_ONLY or scale new entries by a configurable factor (default 0.5). Backtests shall apply it point-in-time, counting an event only from its `announced_at` (doc 03). | Could | P2 | Off by default. Doc 03 §3.9 refers to it. |

---

## 5. Non-market risks

These can lose money even when the strategy is right.

| Risk | Real example | Ubik control |
|---|---|---|
| Exchange insolvency / fraud | FTX, Nov 2022: customer funds frozen, then tied up in bankruptcy | Per-exchange capital cap; only trading capital on exchanges; keys never allow withdrawals (08) |
| Hack / withdrawal freeze | Bybit, Feb 2025: about $1.5B stolen ([Crystal](https://crystalintelligence.com/investigations/the-10-biggest-crypto-hacks-in-history/)). Bitget, 24 Sep 2026: about $387.5M taken, withdrawals paused and reopened in phases ([BleepingComputer](https://www.bleepingcomputer.com/news/security/bitget-resumes-bitcoin-withdrawals-after-3875-million-crypto-heist/)) | Same as above; spread larger capital across two venues (P3) |
| Stablecoin depeg | USDC hit about $0.87 when $3.3B of reserves were stuck at SVB, Mar 2023 ([The Block](https://www.theblock.co/post/218993/usdc-and-dai-remain-at-about-0-90-following-circles-disclosure-of-funds-at-svb)). USDe hit about $0.65 *on Binance only* for about 90 min on 10 Oct 2025 ([Cointelegraph](https://cointelegraph.com/research/exclusively-obtained-orderbook-data-reveals-details-about-usde-crash)). Several small stablecoins failed in 2026. | Price stablecoins at a market reference, never assume 1.00; peg monitor using ≥ 2 sources; per-stablecoin cap |
| Liquidity | Thin books in crashes; wide spreads on altcoins | Minimum-volume universe filter; order size limited relative to visible depth; protected market orders |
| Outage / maintenance | Venues degrade exactly when stops are needed | Stale-data halt, native catastrophe stops, reconcile before resuming |
| Ambiguous API results | Binance: a timeout (`-1007`) or any 5xx means "execution status unknown," so the order *may* exist ([limits](https://developers.binance.com/docs/binance-spot-api-docs/rest-api/limits)) | UNKNOWN order state (§7) |
| Rate-limit bans | Binance answers repeated 429s with IP bans (HTTP 418) lasting "2 minutes to 3 days" | Client-side limiter; venue HALTED on a ban |
| API churn | Binance removed listenKey user streams (2026-02-20) and retired v1 endpoints (2026-03-25) ([changelog](https://developers.binance.com/docs/binance-spot-api-docs/CHANGELOG)) | Adapter contract tests and changelog watch (06) |
| Runaway software | Knight Capital, 2012 | Order-rate, turnover and duplicate guards; kill switch; staged rollout |

**Leverage (P4 only).** With leverage you borrow to hold a bigger position. If losses push your collateral below the "maintenance margin," the exchange **liquidates**: it force-closes the position at market, usually at the worst moment. Perpetual futures also charge or pay **funding** every few hours, and in extreme events exchanges may **auto-deleverage (ADL)** profitable traders. On 10 Oct 2025, exchange-internal collateral pricing amplified the liquidation cascade.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-028 | Ubik shall value balances, P&L and limits in the owner's reference currency (default USD), pricing stablecoins at a market reference rather than 1.00. | Must | P1 | Depegs must show up in equity. |
| RK-029 | Ubik shall monitor held stablecoins against ≥ 2 reference sources: warn at > 0.5% deviation; at > 1% for 5 min, set REDUCE_ONLY on pairs quoted in that coin and alert. | Should | P1 | Must by P3. |
| RK-030 | Ubik shall cap the share of equity in any single stablecoin (default 60%) and warn when it is exceeded. | Should | P2 | |
| RK-031 | Ubik shall limit the trading universe to pairs above a minimum 24h volume (default $50M), and deny or split orders larger than 10% of visible depth within 0.5% of mid. | Must | P1 | |
| RK-032 | Ubik shall track USD value per exchange against an owner-set cap and alert when it is exceeded, suggesting a manual withdrawal. Ubik never withdraws. | Must | P3 | The FTX and Bitget lesson. |
| RK-033 | Ubik shall set a venue to HALTED when its error/reject rate exceeds a threshold (default 20% over 5 min) or on an IP ban, and alert. | Must | P3 | Exercised in P1 via fault injection (EX-021). |
| RK-034 | Ubik shall consume venue status and maintenance notices and set the venue REDUCE_ONLY during announced maintenance. | Should | P3 | |
| RK-035 | Ubik shall deny orders that would make a spot balance negative or require margin, borrowing or leverage until P4 is explicitly enabled. | Must | P1 | Hard ceiling. |
| RK-036 | P4 derivatives shall enforce max leverage (default 2×), isolated margin, a liquidation distance ≥ 3× daily ATR, and funding accrual in P&L, and shall include liquidation and ADL in stress tests. | Must | P4 | |

---

## 6. Portfolio construction and capital allocation

**Sleeves.** Each **bot** (one strategy version deployed on one account) runs in exactly one **sleeve**: a virtual capital partition with its own capital, positions, fills and P&L. Several bots may share an account from P1, each in its own sleeve. Sleeves plus an unallocated reserve always add up exactly to the real account; the reserve, the Manual sleeve (PF-014) and the External sleeve (EX-013) have no bot. This gives each strategy an honest stand-alone record, supports per-strategy limits, and works identically in all modes. A sleeve can only sell coins it holds, so one strategy can never sell another's position. Internal netting of opposite orders across sleeves saves fees but complicates accounting, so it is deferred to P4. Manual orders placed from Ubik's UI go to the Manual sleeve (PF-014). Manual trading directly on the exchange, outside Ubik, breaks the invariant: Ubik detects it (EX-013), and a dedicated exchange account or exchange sub-account is recommended.

| Allocation method | Plain language | When |
|---|---|---|
| Manual fixed | "A gets $3,000, B gets $2,000" | P1 default |
| Equal weight | Same capital each | P1 baseline |
| Inverse volatility | Calmer strategies get more, so each contributes similar swings | P2 |
| Risk parity | Like inverse volatility, but strategies that move together share one risk budget | P2, 3+ strategies |
| Performance-based with decay | More capital to recent winners, with older results counting less | P2, Pro mode only. Recent performance is mostly noise, so shrink toward equal weight and cap each strategy. |

**Rebalancing** happens monthly, or when a sleeve drifts more than 25% (relative) from target. Moving cash between sleeves is a bookkeeping entry. Trades happen only if a sleeve must shrink a position, and they pass the risk engine.

**Correlation** measures how much two things move together, from −1 (opposite) to +1 (in lockstep). BTC, ETH and SOL are usually strongly correlated, and in crashes **correlations jump toward 1**, so "diversifying" across crypto majors protects less than it appears to. Ubik tracks rolling correlations of assets *and of strategy returns*.

**Value at Risk (VaR)** answers: "On a normal bad day, how much could I lose?" A 1-day 95% VaR of $300 means that on about 19 days out of 20 the loss should be under $300. It says nothing about the 20th day.

**Expected Shortfall (ES)** answers the follow-up: "On those worst 1-in-20 days, what's the *average* loss?" For example, $520. ES is the more honest number for crypto's "fat tails," where big moves happen far more often than a bell curve predicts.

Ubik computes both by **historical simulation**: it applies each of the last N days' real price moves to today's holdings. Calm-period VaR understates crash risk, so Ubik adds **stress tests**:
- **Historical replays:** Mar 2020, May 2021, Jun 2022, Nov 2022 (FTX), Mar 2023 (USDC), 10 Oct 2025.
- **Hypothetical shocks:** "−30% across crypto," "USDT −10%," "exchange X frozen."

**Exposure dashboard (P1).** Shows equity, cash vs crypto, exposure by asset, stablecoin, exchange and strategy, a "fuel gauge" for each limit, and current trading states. P2 adds VaR, ES, stress results and a correlation heatmap. Visual design is in 07.

**Account performance (P1).** Besides each strategy's own record, the owner sees how the whole account did over any period, including each sleeve's contribution (PF-015). P1 keeps allocation simple: manual or equal-weight across the bots on an account. P2 adds the other allocation methods, rebalancing, portfolio risk (VaR, ES, stress tests) and multi-strategy portfolio backtests.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| PF-001 | Each bot shall run in exactly one sleeve with its own capital, positions, fills and P&L in all modes. Sleeves plus the reserve shall equal account balances exactly, checked on every fill. | Must | P1 | Several bots may share an account. 06 §9 models SLEEVE. |
| PF-002 | A sleeve shall only sell assets and spend cash it holds; cross-sleeve borrowing is not allowed. | Must | P1 | |
| PF-003 | Ubik shall keep an unallocated cash reserve (default 10% of equity) that strategies cannot use. | Should | P1 | Fees and rebalancing buffer. |
| PF-004 | Ubik shall support manual-fixed and equal-weight allocation. | Must | P1 | |
| PF-005 | Ubik shall support inverse-volatility and risk-parity allocation with configurable lookback and min/max weights. | Should | P2 | |
| PF-006 | Performance-based allocation shall use decayed metrics (half-life 60 days), 50% shrinkage toward equal weight, a 40% cap per strategy, and ≥ 90 days or ≥ 50 round trips of paper/live record. Backtest-only records do not count. | Could | P2 | Anti noise-chasing. |
| PF-007 | Ubik shall rebalance sleeves on a calendar (default monthly) and on drift thresholds (default 25% relative). Trades required by rebalancing pass the risk engine. | Should | P2 | |
| PF-008 | Ubik shall compute rolling 30/90-day correlations of assets and strategy returns, derive dynamic clusters (default threshold 0.7) for RK-022, and alert when active strategies' average correlation exceeds 0.7. | Should | P2 | |
| PF-009 | Ubik shall compute 1-day 95%/99% VaR and ES by historical simulation over ≥ 1 year, each with a plain-language sentence. | Should | P2 | |
| PF-010 | Ubik shall run historical-replay and hypothetical stress tests against the current portfolio, on demand and weekly. | Should | P2 | |
| PF-011 | Ubik shall show the exposure dashboard with limit-utilization gauges and trading states. | Must | P1 | |
| PF-012 | Ubik shall backtest multi-strategy portfolios with sleeves, allocation, rebalancing and the shared risk engine. | Should | P2 | With 04. |
| PF-013 | Ubik shall optionally map sleeves to exchange sub-accounts in live mode. | Could | P3 | Where the venue allows. |
| PF-014 | Ubik shall book manual orders placed from the UI (04 PT-006; paper from P1, live from P3) to a per-account Manual sleeve funded by explicit allocation from the reserve. They shall pass the same risk engine, count toward account-level limits, and never change a strategy sleeve's positions. They shall be excluded from strategy metrics, trial counts and graduation evidence. | Should | P1 | Shown in the Journal with reason "manual". Closing a bot's position from the UI is booked to that bot's sleeve as a logged "manual intervention". |
| PF-015 | Ubik shall show account-level performance for any selected period: time-weighted return net of costs, max drawdown and recovery, volatility, fees and slippage, cash and reserve drag, and each sleeve's contribution (bots and Manual), compared with buy-and-hold BTC and with cash, and labelled paper ("PAPER · simulated money") or LIVE per 07 UX-031. | Must | P1 | On the Portfolio screen (07). Monthly statements and exports P2; 02 CAP-024 attribution stays P2. |

---

## 7. Order management system (OMS)

```
 PROPOSED ──deny──► DENIED                        (never left Ubik)
    │ approve
 PENDING_SUBMIT ──venue reject──► REJECTED
    │ ack           └─timeout/5xx/disconnect─► UNKNOWN ──resolved──► (actual state)
    ▼                                             └─not found after grace─► REJECTED
  OPEN (resting or untriggered stop) ──fill──► PARTIALLY_FILLED ──► FILLED
    ├─cancel─► CANCEL_PENDING ──► CANCELED      (filled qty kept if partial)
    └─time-in-force / venue expiry ──► EXPIRED
```

**Event sourcing.** Every transition is appended as an event: risk decision, sent, acknowledged, each fill, cancel requested, cancel confirmed. Current state is derived from the events, which gives a full audit trail and deterministic crash recovery. Each event is written **before** the network call (write-ahead), so after a crash Ubik knows which orders might be in flight.

**Client order IDs and idempotency.** Ubik creates the ID before sending, e.g. `ubk-L-trend01-000123`, within venue limits (≤ 36 characters on Binance). This makes resending **idempotent**, meaning twice has the same effect as once. Venues treat reused IDs differently. Coinbase Advanced returns the existing order ([docs](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/orders/create-order)). Binance rejects a duplicate only while the original is still open. So the OMS never relies on the venue: after any ambiguous result it **asks first, then acts**.

**Retries, unknown states, rate limits.**
- **Retries.** Reads and cancels retry with exponential backoff plus jitter (randomized waits). New orders are never blindly resent.
- **UNKNOWN orders.** A timeout, 5xx, Binance `-1007` or a mid-request disconnect marks the order UNKNOWN. New orders on that symbol and sleeve are blocked, and Ubik resolves the order via the user-data stream and a query by client ID. If it isn't resolved within 30 s, the symbol goes REDUCE_ONLY and an alert fires.
- **Rate limits.** Client-side token buckets mirror each venue's published limits at ≤ 70% usage. Examples are Binance's per-IP request weight and per-account unfilled-order counts, and Kraken's decaying per-key counter ([Kraken](https://docs.kraken.com/exchange/guides/rest/ratelimits)). The queue priority is **cancels > protective exits > entries > queries**, and `Retry-After` is honored.

**Reconciliation.** Ubik compares its ledger and open orders with what the venue reports. It runs at startup, on reconnect, every 60 s and after kill-switch actions. In live mode **the exchange is the source of truth**. A mismatch puts the affected scope in REDUCE_ONLY, and the correction is posted as an explicit adjustment entry, never a silent overwrite. Paper mode reconciles against the simulator's exchange-like account API, so the code path is proven before real money is involved.

**Exact accounting.**
- **Decimals, never floats.** Binary floats can't represent 0.1 exactly, and the errors accumulate.
- **Instrument rules.** Tick, step, min/max quantity and order value, price bands, max open orders, and order-type flags (Binance: `ocoAllowed`, `pegInstructionsAllowed`, `opoAllowed`) are fetched from each venue, versioned and applied identically in all modes. Since 2026 Binance also *expires* taker orders that would fill outside a band around a reference price, its "[Price Range Execution Rule](https://developers.binance.com/en/docs/products/spot/CHANGELOG)," and the paper simulator must model this.
- **Fees.** Each fill records the amount, the fee asset (base, quote or e.g. BNB), maker/taker, and the USD value. Fee schedules come from doc 06's venue registry (AR-002), the single source; the figures here are only illustrative. Entry-tier taker fees range from about 0.1% (Binance) to 0.8% (Kraken) and up to 1.2% (Coinbase Advanced, until the owner's account confirms its tier). An unverified entry tier defaults to the higher, conservative rate. Kraken restructured its tiers in July 2026 ([comparison](https://brokerchooser.com/news/coinbase-vs-kraken-fee-structures-and-trading-features-compared--bbeea493)). A 1.2% round-trip cost wipes out a strategy that averages +0.8% per round trip, so Ubik uses the account's real fee tier.
- **Double-entry ledger.** Each fill posts balanced entries, e.g. +0.01 BTC, −600 USDT, −0.6 USDT fee. **Tax lots** are kept so 08 can produce FIFO, specific-ID or average-cost reports.

Indicative tables (06 owns the schema): `orders(client_order_id, venue, sleeve_id, symbol, side, type, tif, qty, price, stop_price, state, risk_decision_id)`, `order_events(order_id, seq, type, payload, venue_ts, local_ts)`, `fills(venue_trade_id, order_id, qty, price, fee_amount, fee_asset, liquidity)`, `ledger_entries(txn_id, account, sleeve_id, asset, amount, kind, ref_id)`, `risk_decisions(outcome, reason_code, inputs_snapshot, profile_version)`. Every table also carries `workspace_id` (06). There is no free `mode` column: paper and live use identical table definitions in two Postgres schemas, `paper` and `live`, each with a mode CHECK constraint, so the code path is the same and the two can't mix. Backtest runs stay ephemeral. S3 shadow fills are booked to a shadow account in the `paper` schema, never to the live ledger.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| EX-001 | Ubik shall manage orders through the §7 state machine. Undefined transitions shall be rejected, logged and alerted. | Must | P1 | |
| EX-002 | Ubik shall store order and risk events append-only, derive state from them, and persist each event before the matching network send. | Must | P1 | Event sourcing, write-ahead. |
| EX-003 | Ubik shall assign each order a deterministic, account-unique client order ID before submission, within venue limits. | Must | P1 | |
| EX-004 | Ubik shall retry reads and cancels with exponential backoff and jitter. It shall never resubmit a new order after an ambiguous result without first querying by client ID. | Must | P1 | No duplicate orders. |
| EX-005 | Ubik shall handle UNKNOWN orders per §7: block the symbol and sleeve, resolve within 30 s, else REDUCE_ONLY and alert; "not found" after a grace window becomes REJECTED. | Must | P1 | Exercised in paper via fault injection. |
| EX-006 | Ubik shall rate-limit per venue at ≤ 70% of published limits, prioritizing cancels > exits > entries > queries, and honor Retry-After. | Must | P1 | Owns the ≤ 70% rule; 06 AR-008 and NFR-018 cite it. |
| EX-007 | Ubik shall reconcile orders, balances and fills at startup, on reconnect, every 60 s and after kill-switch actions. It shall set REDUCE_ONLY on mismatch and post corrections as adjustment entries. | Must | P1 | Against the simulator in paper. |
| EX-008 | Live trading shall start in REDUCE_ONLY until the first reconciliation passes. | Must | P3 | |
| EX-009 | Ubik shall use exact decimal arithmetic for all prices, quantities, balances and fees. Binary floats are prohibited in accounting and order paths. | Must | P1 | Lint plus property tests. |
| EX-010 | Ubik shall load, version and refresh each venue's instrument rules and order-type capabilities, and apply them identically in all modes. Quantities round down to step and prices round to tick in the conservative direction. | Must | P1 | Sim-live parity. |
| EX-011 | Ubik shall record per-fill fee amount, asset, rate, maker/taker and USD value, using the account's actual fee tier (or a chosen tier in paper) from doc 06's venue registry (AR-002). An unverified tier shall default to the higher, conservative rate. | Must | P1 | |
| EX-012 | Ubik shall maintain a double-entry ledger by sleeve and venue account, preserving tax lots. | Must | P1 | Tax exports in 08. |
| EX-013 | Ubik shall detect orders, fills or balance changes it did not originate, book them to an "External" sleeve, and alert (Critical in live; 07 §8). | Should | P3 | Manual trades or key compromise. |

---

## 8. Execution

### 8.1 Order types

Support was checked against public docs in Oct 2026: [Binance](https://developers.binance.com/docs/binance-spot-api-docs/CHANGELOG), [Coinbase Advanced](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/orders/create-order), [Kraken](https://docs.kraken.com/api-reference/trading/add-order). Adapters must read capability flags at runtime rather than hard-code this table.

| Type | What it does / when to use | Binance | Coinbase Adv. | Kraken |
|---|---|---|---|---|
| Market | Fill now at best prices. Urgent exits only; Ubik sends a *protected* version. | MARKET | market_market_ioc | market |
| Limit (GTC/GTD) | Rest at your price or better (good-till-canceled / good-till-date). Entries, take-profits. | LIMIT | limit_limit_gtc / gtd | limit |
| Post-only | A limit order canceled if it would trade immediately, which guarantees the maker fee. Patient entries. | LIMIT_MAKER | post_only flag | `post` flag |
| IOC | Fill what's available now and cancel the rest. Protected market orders, slices. | timeInForce IOC | sor_limit_ioc | IOC |
| FOK | All or nothing, immediately. Rarely needed. | timeInForce FOK | limit_limit_fok | Not on spot REST (emulate) |
| Stop-market | Becomes market at the trigger. Protective stops; can slip in gaps. | STOP_LOSS | Not on spot | stop-loss |
| Stop-limit | Becomes limit at the trigger. Has a price cap but may not fill in a crash. | STOP_LOSS_LIMIT | stop_limit_…_gtc / gtd | stop-loss-limit |
| Trailing stop | A stop that follows price. Locks in gains. | `trailingDelta` (bps) ([FAQ](https://developers.binance.com/docs/binance-spot-api-docs/faqs/trailing-stop-faq)) | Not native (bot-managed) | trailing-stop(-limit) |
| OCO / bracket | Linked orders: one fills, the other cancels (TP + SL). | OCO, OTO, OTOCO, OPO (Dec 2025) | trigger_bracket; attached TP/SL | Conditional close |
| Iceberg | Shows only part of the size. Larger orders in thin books. | icebergQty (≤ 100 parts) | None | iceberg (display ≥ 1/15) |
| TWAP / scaled / pegged | Exchange-side slicing, a price ladder, or a price that follows the best bid/ask. | Pegged (Aug 2025); TWAP via separate Algo API | twap_limit_gtd, scaled_limit_gtc | None |

**Defaults:**
- **Entries:** limit or post-only unless urgent.
- **Stop exits:** protected market.
- **Take-profits:** resting limit.
- **Missing types:** where a venue lacks a type, Ubik emulates it bot-side and labels it "emulated." An emulated stop does not protect you while Ubik is offline.

### 8.2 Execution algorithms

For BTC/ETH on top exchanges, an order of a few thousand dollars is a sliver of the visible book, so **market impact** (your own order moving the price) is negligible. **Fees and the bid-ask spread dominate**, and the best lever at personal scale is choosing maker over taker when there's no urgency. Algorithms start to matter for thin altcoins, or for orders above about 1% of the depth near the mid-price.

| Algorithm | Plain language | Phase |
|---|---|---|
| Protected market | IOC limit with a price cap; the unfilled remainder is reported. The default "urgent" order. | P1 |
| Limit-chase / peg | Post-only with timeout-then-cross: sit at the best bid, re-price every N seconds up to K times or X bps, then fall back to protected market. Saves fees. | P2 |
| TWAP / iceberg | Split into randomized time slices, or show only part of the size. For large orders, thin pairs and gradual buying. | P2 |
| VWAP / POV / smart order routing | Follow the volume curve, trade a % of market volume, or split across exchanges. Institutional-scale tools; little value for one person. | P4 |

### 8.3 Transaction cost analysis (TCA)

TCA measures what a trade *really* cost compared with the moment the strategy decided to trade. The core measure is **implementation shortfall**. Example:
- The strategy decides to buy 0.05 BTC with mid at $60,000. This is the **decision price**.
- The order leaves 400 ms later with mid at $60,010. This is the **arrival price**, and the gap is the delay cost: 1.7 bps.
- The average fill is $60,030: spread plus impact adds 3.3 bps.
- The taker fee is 10 bps.

The total is 15 bps, about $4.50 on $3,000. A backtest assuming 5 bps understated the cost 3×. Unfilled limit orders also carry **opportunity cost**: the move missed on the quantity that never filled.

**Feedback loop.** P1 computes TCA for every order and shows each strategy's assumed vs observed cost. From P2, TCA is aggregated by venue, pair, order type, size and volatility regime, and doc 04's backtest cost model is calibrated from it, using the **75th percentile** of observed slippage plus real fees rather than the average, so it leans conservative. Each strategy shows "backtest assumed X bps / reality Y bps." Paper fills, simulated against the live order book, are tagged *simulated*. They can't capture queue position or your own impact, so they are a floor on real costs, not an estimate.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| EX-014 | The internal order model shall support market, limit (GTC/GTD), post-only, IOC, FOK, stop-market, stop-limit, take-profit, trailing stop and OCO/bracket. Each maps to a native venue type or a labelled bot-side emulation. | Must | P1 | Native mappings exercised from P3 (S2 exchange demo). |
| EX-015 | Market orders shall by default be protected: sent as IOC limits capped at 0.5% through the best price, with any unfilled remainder reported. | Must | P1 | Fat-finger and crash safety. |
| EX-016 | Ubik shall provide a limit-chase algorithm (post-only with timeout-then-cross) with configurable re-price interval, max re-prices, max drift and fallback. | Should | P2 | Maker-fee savings. |
| EX-017 | Ubik shall provide TWAP and iceberg algorithms (bot-side with randomization; native where available). | Could | P2 | |
| EX-018 | Ubik shall provide VWAP, POV and cross-venue smart order routing. | Could | P4 | |
| EX-019 | Ubik shall compute per-order TCA: decision price, arrival mid, spread, average fill, fees, implementation shortfall, fill ratio, time to fill and opportunity cost, tagged simulated or real. It shall also show each strategy's assumed vs observed cost. | Must | P1 | |
| EX-020 | Ubik shall aggregate TCA and calibrate backtest cost assumptions from it (default 75th-percentile slippage). | Should | P2 | The per-strategy assumed vs observed view is P1 (EX-019). |
| EX-021 | The paper venue shall behave like an exchange: asynchronous acks with latency, partial fills, filter rejections, rate limits, price-band expiries and configurable fault injection (timeouts, disconnects, unknown states). | Must | P1 | The paper venue is `SimVenue` fed by live data (06). Exercises OMS error paths before P3; fill model owned by 04. |

---

## 9. Path to live (P3)

### 9.1 Stages

Stages are tracked per bot (one strategy version on one account) and are independent of the trading states. Draft, S0 and S1 are tracked from P1; S2–S5 unlock in P3.

| Stage | What runs | Money | Graduate when (defaults) |
|---|---|---|---|
| Draft | A strategy version being built or edited | None | A backtest of this version has completed |
| S0 Backtest | Historical data | None | Passes 04's out-of-sample checks (P1: the vault check; walk-forward, DSR and PBO join in P2); ≥ 100 round trips over the version's full backfilled history on the traded pair, not pooled across pairs (schedule-based strategies such as DCA and rebalancing: ≥ 24 completed cycles) |
| S1 Paper | Live data plus Ubik's simulator | Simulated | Same version throughout; ≥ 30 days and ≥ 20 round trips (schedule-based strategies: ≥ 8 cycles); within the expectation band; 0 unresolved reconciliation breaks; ≥ 99% uptime; the other 04 §8.5 criteria, including the parity replay (04 PT-011) |
| S2 Exchange demo | Real exchange API with demo funds, e.g. [Binance Spot Demo Mode](https://developers.binance.com/docs/binance-spot-api-docs/demo-mode/general-info) (since Jan 2026; same features as live, unlike a testnet). Testnets are not used for graduation. | Exchange play money | Adapter contract suite (06 AR-004) passes; 7 days with no unresolved UNKNOWN orders. For venues without a realistic demo, see below. |
| S3 Shadow | Live account through a **read-only** key, sized against real balances; orders priced against the live book, never sent. Simulated fills are booked to a shadow account in the paper schema, never to the live ledger. | Real balances, nothing sent | ≥ 14 days; shadow TCA within the cost model |
| S4 Pilot | Live under hard caps | ≤ $500 per strategy, ≤ $100 per order | ≥ 30 days and ≥ 20 round trips; slippage ≤ model + 10 bps; drawdown ≤ 1.25 × reference drawdown; no Critical incidents |
| S5 Scaled | Live | At most 2× step-ups per 30 days, up to the owner's cap | Ongoing; demotion rules apply |

A version that is withdrawn ends in **Retired**, a terminal state. Retired versions still count in the trial ledger (04 BT-021).

**Entering S1 (enforced from P1).** A bot may start paper trading when all of these hold:
- The blocking 04 ST-018 checks pass, including the truncation look-ahead test (02 CAP-009, 04 ST-019).
- Its kill criteria are declared (02 CAP-033; default: the RK-020 strategy drawdown breaker).
- If its backtest has fewer than 100 round trips, the owner has seen the RK-039 consequence preview and typed a confirmation.

The full S0 bar is re-checked at S1 exit (04 §8.5, "Backtest quality (S0)"), so the 100-round-trip floor stays binding for promotion beyond S1.

**S2 on venues without a realistic spot demo.** Coinbase's sandbox is mocked, and Kraken has no spot demo (06 §2). On such venues S2 is met by the 06 AR-004 contract suite, run against the mocked sandbox where one exists and then with minimum-size live orders as the first step of S2, plus fault-injection tests on the paper venue (EX-021). Those live orders already need a trade-only key and live arming (08).

**"Within the expectation band"** means per-round-trip returns are no worse than the 10th percentile of a bootstrap (resample) of the backtest's round trips, and drawdown has not exceeded the reference drawdown. The aim is to catch strategies that only worked in hindsight, not to demand that live beats the backtest.

**Promotion needs owner approval; demotion is automatic.** Gate overrides follow 04 PT-015: duration, round-trip or cycle count and same version are never overridable; other S1 criteria can be overridden only under the Adventurous profile, with a logged justification and a 24 h cooling-off. A new version always re-enters at S1. Demotion triggers:
- Drawdown > 1.5 × reference drawdown → back to paper (S1).
- Slippage > 2× the model for 2 weeks → back to pilot size.
- A reconciliation break or UNKNOWN order unresolved for > 15 min → HALTED and reviewed.
- Three Warning-level incidents in 30 days → down one stage.

**Pre-live checklist:**
1. Same strategy artifact (code hash, parameters, risk profile) as passed S1–S3.
2. Trade-only API key with withdrawals off and an IP allowlist (08).
3. Dedicated exchange account or exchange sub-account.
4. Exchange capital cap set, and only pilot capital deposited.
5. Kill switch L1–L3 tested on the exchange demo within the last 7 days (on a venue without one: in paper, plus L2 during the S2 minimum-size live orders).
6. Native catastrophe stops verified.
7. Two external notification channels set up and tested (07 UX-054), and a test Critical alert acknowledged.
8. Clock sync OK, with a plan for host restarts and outages.
9. The owner has acknowledged "worst case for this pilot: $X."
10. Jurisdiction and exchange ToS checks done (08).

### 9.2 Monitoring, alerting and runbooks

**What is monitored:**
- **Health:** engine heartbeat; data freshness per symbol; order acknowledgement latency.
- **Order flow:** reject and error rates; UNKNOWN count; reconciliation status.
- **Money and risk:** P&L against limits; exposure; capital per exchange.
- **Environment:** clock offset; stablecoin deviation; host and database health.

An **external heartbeat monitor** (06 AR-032, P1) alerts the owner if Ubik goes silent, since a dead process can't send its own alerts. The engine pings it every 30 s, and 90 s of silence raises the alert, within 06 NFR-017's 2 minutes: a Warning in paper (held during quiet hours), Critical in live.

Severity levels use doc 07's names (07 §8 owns notifications): **Critical / Warning / Info / Digest**. Critical is reserved for events that can lose real money (live, P3); the same events in paper are Warnings.
- **Critical, act now (live):** kill switch fired, live reconciliation break, unresolved UNKNOWN order, IP ban, external activity, API key rejected, or engine offline or stale data with live positions. Sent on all configured external channels, bypassing quiet hours, and repeated until acknowledged.
- **Warning, today:** loss limit hit, depeg warning or demotion; in paper, the Critical events above, plus engine offline or stale data (§4.4) while bots are running or positions are open.
- **Info / Digest:** daily summary and TCA drift.

**Runbooks (summaries; the full versions live in-app with one-click containment):**
- **Exchange down:** the venue goes REDUCE_ONLY automatically, and native stops remain. Don't market-sell into a recovering book. Reconcile, then re-enable in steps.
- **Runaway strategy:** the guard HALTs the strategy and cancels its orders (L2). Compare intents with orders in the logs, fix, re-backtest, and **demote to paper**: the fixed strategy is a new version, which re-enters at S1.
- **API key compromise:** external orders or fills appear, or the exchange sends security emails. Fire L2. The owner revokes the key on the exchange (Ubik shows the steps) and checks the withdrawal allowlist. Then issue a new key, reconcile, and resume at pilot (with 08).
- **Stablecoin depeg:** affected pairs go REDUCE_ONLY. The owner decides whether to hold or convert.
- **Reconciliation mismatch:** the affected scope goes REDUCE_ONLY. Find the cause (a missed fill event, a BNB fee, a manual trade) and post an adjustment entry.

### 9.3 What P1 must get right so P3 is configuration, not a rewrite

1. **One pipeline.** Strategies emit intents only. The pipeline is identical in all modes, and only the venue behind the `ExecutionVenue` port swaps: `SimVenue` for backtest and paper, `LiveVenue` (which forwards to the execution gateway) for exchange demo and live (06).
2. **Realistic paper venue.** It is asynchronous, can fail, and is rate-limited (EX-021).
3. **Real exchange rules.** Real instrument rules and fee schedules apply everywhere.
4. **Exact accounting.** Decimals, the double-entry ledger, sleeves and tax lots exist from day one.
5. **Robust order handling.** Event-sourced orders, deterministic client IDs and UNKNOWN handling, with reconciliation running in paper too.
6. **Safety controls.** Trading states, the kill switch and alerting exist in P1.
7. **Mode is configuration.** It is set per account and bot, and strategy logic cannot see it, so code can't behave differently live.
8. **Time.** All timestamps are UTC, with exchange and local times both recorded and a monotonic clock for timeouts.
9. **Secrets.** Secrets go through an abstraction (08) from day one. P1 stores no exchange keys at all (08 SEC-007), but notification channel credentials are already secrets (08 SEC-014).

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| EX-022 | Ubik shall track a rollout stage (Draft, S0–S5, Retired) per bot, with a graduation scorecard (criteria, current values, pass/fail). Promotion requires the checklist plus owner approval; demotion is automatic per §9.1. | Must | P3 | Draft/S0/S1 tracking and the §9.1 S1-entry checks are required from P1; 04 PT-014 shows S1 progress in P1. |
| EX-023 | Ubik shall support the exchange demo by pointing `LiveVenue` and the execution gateway at the venue's demo endpoints and keys, with no other code differences. | Must | P3 | Serves S2 (§9.1). Testnets are not used for graduation. Venues without a realistic demo use the §9.1 S2 variant. |
| EX-024 | Ubik shall support shadow mode: a read-only key on the live account, orders generated, sized against real balances and priced against the live book with TCA, nothing sent, and simulated fills booked to a shadow account in the paper schema, never the live ledger. | Should | P3 | |
| EX-025 | The pilot stage shall enforce hard caps (≤ $500 per strategy, ≤ $100 per order by default) in the execution gateway's last-line checks (RK-009). | Must | P3 | |
| EX-026 | Ubik shall emit the §9.2 metrics and deliver alerts by severity (07 §8). In live mode, Critical alerts shall go to ≥ 2 verified external channels and repeat until acknowledged. | Must | P3 | P1 already has in-app alerts, ≥ 1 owner-chosen external channel (07 UX-037, UX-054) and the external heartbeat monitor (06 AR-032, NFR-017). |
| EX-027 | Ubik shall include in-app runbooks for the §9.2 incidents, with one-click containment actions. | Should | P3 | |
| EX-028 | Switching a strategy to live shall be configuration only (same artifact hash), requiring trade-only credentials, typed confirmation and per-strategy opt-in. The strategy API shall not expose the trading mode to strategy logic. | Must | P3 | Sim-live parity. |

---

## 10. Beginner guardrails

The Careful profile (§2.1) is the default for every new install in **every mode, including backtests and paper**, so paper results are honest about what live trading would allow. Guided and Pro mode change presentation and defaults only, never limits or gates (07 UX-026). The Balanced and Adventurous profiles widen soft limits but never the hard ceilings. Moving to a looser profile goes through the one-time acknowledgement flow in 07 UX-053; moving to a tighter one is instant.

**Every block is explained** with five parts: what happened, which limit, current vs allowed, why the limit exists, and what to do. Example:

> **Ubik didn't place this BTC buy.** It would have raised Bitcoin to 62% of your account; your limit is 50%. *Why:* with most of your money in one coin, one bad day for that coin is a bad day for your whole account. *What you can do:* wait for an open BTC position to close, or lower this strategy's position size. [Learn more: concentration risk]

**Risky changes show consequences, not just "Are you sure?"** Example, under the Adventurous profile: *"At 3% risk per trade, 10 losing trades in a row (this happens) would cost about $2,630, or 26% of this strategy's money. Type RAISE RISK to confirm."*

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-037 | New installations shall default to the Careful profile in all trading modes (backtest, paper, live) and both UI modes. Moving to a looser risk profile shall require the one-time acknowledgement flow in 07 UX-053 and, in live mode, the SEC-018 cooling-off (RK-008); moving to a tighter profile shall take effect instantly. | Must | P1 | Each later loosening still needs RK-039's preview and confirmation. |
| RK-038 | Every risk denial or reduction shall show what was blocked, which limit, current vs allowed values, why the limit exists and next steps, linked to a glossary entry. | Must | P1 | Test: every reason code has message text. |
| RK-039 | Raising risk per trade above 2%, loosening loss or drawdown limits, disabling stops, or enabling a strategy with < 100 backtest round trips shall show a consequence preview and require typed confirmation. | Must | P1 | With < 100 round trips, the confirmation lets the bot start S1 Paper; the S0 floor stays binding for promotion beyond S1 (§9.1, 04 §8.5, PT-015). |
| RK-040 | The main dashboard shall show limit utilization, current trading states and a filterable timeline of recent risk decisions with reasons. | Should | P1 | Builds trust; shares PF-011. |

---

## 11. Jurisdiction flags (details in 08)

- **Venues and quote currency.** Binance.com does not serve US residents; Binance.US is a separate venue with fewer features. In the EEA, several MiCA-regulated venues restricted USDT trading for EEA users in 2025. The owner's country therefore determines the venues, the quote stablecoin (USDT, USDC or EUR) and the order types available, so the risk engine must not assume a quote currency.
- **Tax lots.** The tax lot method (FIFO, specific ID, average cost) varies by country. The ledger keeps lots so any method can be computed.
- **Exchange terms.** Exchange ToS on API trading and sub-accounts vary by country and account tier.

## Open questions for the owner

1. **Pain threshold:** with real money, what loss in a bad *month* would make you stop? Your answer can tighten, but never loosen, the default loss and drawdown limits.
2. **Capital and venues:** roughly how much might run live eventually, and would you split it across two exchanges to limit counterparty risk?
3. **Country and reference currency** (USD, EUR, other)? These determine the venues, the quote stablecoin and the tax lot method.
4. **Dedicated account:** will you trade manually on the exchange account Ubik uses? A dedicated account or sub-account is strongly recommended.
5. **At 3 a.m.:** if the account drawdown breaker fires, should Ubik only pause and cancel (L1/L2), or also flatten automatically (L3)?
6. **Alerts:** which two external channels reliably reach you (Telegram, email or ntfy in P1; web push from P2)?
7. **Native stops:** are you OK with exchange-side catastrophe stops that lock part of your balance?
8. **Holding period:** intraday, days, or longer? This changes how much fees, staleness thresholds and execution algorithms matter.

## Risks

| Risk | Mitigation |
|---|---|
| Paper results too rosy (no queue position or impact in the fill model) | Conservative fill model (04), TCA tagging, demo and shadow stages, pilot caps |
| Defaults feel restrictive, so the owner disables guardrails | Explain every block; Careful/Balanced/Adventurous profiles; hard ceilings stay regardless |
| False halts from noisy staleness or depeg detectors cause alert fatigue | Per-pair thresholds, hysteresis, multiple peg sources |
| P3-grade OMS and reconciliation in P1 slows the MVP | Keep algorithms, allocation methods and VaR out of P1; build only the §9.3 invariants |
| Exchange API changes break adapters | Contract tests against exchange demos; changelog watch; runtime capability flags |
| Correlations go to 1 in crashes | Cluster caps, stress replays, ES over VaR |
| Bot stops fail while offline; native stops slip or miss in flash crashes | Hybrid stops, sizing that assumes gaps, dead-man's switch |
| Exchange insolvency or hack freezes funds | Per-exchange caps and minimal on-exchange capital. Software can't fully mitigate this. |
| Self-hosted machine fails with positions open | Native catastrophe stops, external heartbeat monitor, cloud option (06) |
| Precision or accounting bugs | Decimals only, double-entry ledger, property tests, continuous reconciliation |
| The owner can always override (single user) | Cooling-off periods, typed confirmations, consequence previews, logged decisions |
