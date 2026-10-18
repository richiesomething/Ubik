# Risk Management, Portfolio Construction & Execution

*Ubik planning doc 05 · Draft v0.1 · 2026-10-18 · ID prefixes: **RK** (risk management), **PF** (portfolio & capital allocation), **EX** (order management, execution, path to live)*

## Summary

- **One mandatory pipeline in every mode.** Every order goes Strategy → Position sizer → **Risk engine** → **Order management system (OMS)** → Venue adapter. Strategies never touch an exchange connection. Only the venue adapter at the end changes between backtest, paper and live. Build this pipeline fully in P1, even though the money is fake: that is what makes P3 a configuration change instead of a rewrite.
- **Layered, explainable limits with conservative defaults.** Limits apply per order, per strategy, per asset or correlated cluster, per account and per exchange. There are three trading states (ACTIVE / REDUCE_ONLY / HALTED) and a kill switch that still works when the web UI is down. Every blocked or downsized order gets a plain-language reason.
- **Beginner default sizing: risk 1% of the strategy's capital per trade, with the stop 2×ATR away, and no position larger than 20% of that capital.** No leverage before P4. Kelly sizing is offered only as a capped, fractional option in pro mode.
- **Treat the exchange and the stablecoin as risks, not as "the bank" and "cash."** Value everything in USD (or the owner's fiat), cap the capital held on each exchange, and monitor stablecoin pegs. Precedents: USDC fell to about $0.87 in March 2023, USDe fell to about $0.65 on Binance on 10 Oct 2025, and Bitget paused withdrawals after a hack in September 2026.
- **Professional-grade order handling from day one.** That means an event-sourced order state machine, deterministic client order IDs, an explicit "UNKNOWN" order state, continuous reconciliation, exact-decimal double-entry accounting, and the exchanges' real precision and fee rules applied in every mode.
- **Close the loop with transaction cost analysis (TCA), and go live in stages.** Measure the real cost of every paper and live fill and feed it back into backtest assumptions. Go live through paper → exchange demo → shadow → pilot with tiny capital → scaled, with written promotion and demotion rules.

---

## 1. Scope and context

This doc covers everything between "a strategy wants to trade" and "the trade is booked and reconciled." That includes risk limits, position sizing, protective exits, multi-strategy capital allocation, the OMS, execution tactics, cost measurement, and the staged path to real money. Strategy logic, the backtest engine and its fill model, and performance metrics are in **04**. Indicator math (ATR, volatility, correlation, regimes) is in **03**. Exchange connectivity, storage and process layout are in **06**. Screens and wording are in **07**. Key security, authentication and tax exports are in **08**.

**How professional firms handle this, in brief.** US broker-dealers must run automated *pre-trade* risk checks before any order reaches a market: credit and capital thresholds, plus "erroneous order" checks. This is SEC Rule 15c3-5, the "Market Access Rule." The cautionary tale is Knight Capital. On 1 Aug 2012 a botched software deployment sent millions of unintended orders in about 45 minutes and lost about $440M. The SEC's [order](https://www.sec.gov/litigation/admin/2013/34-70694.pdf) faulted missing automated controls and missing kill-switch procedures ([background](https://www.cio.com/article/2393212/software-testing-lessons-learned-from-knight-capital-fiasco.html)). Professional firms also keep the risk function independent of the traders, monitor P&L and exposure in real time, reconcile against brokers every day, and measure execution quality. Ubik copies these practices at individual scale. Open-source engines show the same patterns. [NautilusTrader's RiskEngine](https://nautilustrader.io/docs/latest/api_reference/risk) has ACTIVE/REDUCING/HALTED trading states, per-instrument maximum order value and order-rate limits, and an `OrderDenied` event with a reason. [Freqtrade's "protections"](https://www.freqtrade.io/en/stable/plugins/) include StoplossGuard, MaxDrawdown, CooldownPeriod and LowProfitPairs. Doc 02 decides build-vs-reuse. My input: own the risk engine, OMS and ledger, because they are the safety surface, and reuse exchange plumbing such as CCXT or native SDKs.

**Terms used below.**
- **Notional:** the money value of an order or position (price × quantity).
- **bps (basis point):** 0.01%, so 50 bps = 0.5%.
- **Drawdown:** the drop from the highest equity reached so far.
- **ATR (Average True Range):** the average size of a bar's price range, typically over 14 bars. It is a simple volatility gauge.
- **Maker / taker:** a maker order rests in the order book and adds liquidity, usually at a lower fee. A taker order executes immediately against resting orders.
- **Slippage:** the gap between the price you expected and the price you got.

---

## 2. Risk architecture

### 2.1 The pipeline

```
 market data ──► Strategy ──(intent: "long BTC, stop 57,600")──► Position sizer
                                                                     │ order request (qty, price)
                                                                     ▼
        ┌───────────────────────────── RISK ENGINE ─────────────────────────────┐
        │ trading-state gate → instrument rules → per-order → per-strategy →    │
        │ per-asset/cluster → per-account → per-exchange → data/clock health    │
        │                    result: APPROVE / REDUCE / DENY + reason           │
        └───────────────────────────────────┬──────────────────────────────────┘
                                            ▼ risk-approved order (carries decision ID)
                              Order Management System (OMS)
                                            ▼
             Venue adapter:  BacktestVenue · PaperVenue · DemoVenue · LiveVenue
                                            ▼ acks, fills, balances
        Ledger & positions ──► post-trade monitors (loss limits, drawdown, depeg,
                               staleness, reconciliation) ──► trading state / kill switch
```

### 2.2 Design rules

1. **Bypass is impossible by construction.** A strategy receives a context object that can emit *intents* or *order requests* and read data. It has no handle to the OMS submit function, the venue adapter or credentials. The OMS accepts only orders stamped with a risk-decision ID. Tests try to inject an unstamped order and expect it to be refused.
2. **Three outcomes.** The engine can APPROVE, REDUCE (downsize to fit a limit, only if the request allows partial sizing, never upsize) or DENY. Every decision is stored with a snapshot of its inputs: prices, current exposures and the limit values. That snapshot drives the explanations and post-mortems.
3. **Trading states.** The states are **ACTIVE**, **REDUCE_ONLY** (only orders that shrink an existing position) and **HALTED** (nothing except cancels). Each state can apply globally, per exchange, per strategy or per symbol, and the most restrictive applicable state wins.
4. **Same engine, same limits, every mode.** The backtester runs the same risk engine in-process on simulated time, and so does the paper trader. A backtest that ignores the daily loss limit tells a beginner a story that live trading will not repeat.
5. **Cancels always pass.** Cancels reduce risk, so every state allows them. A "cancel-replace" (amend) that increases size or aggressiveness is checked like a new order.
6. **Defense in depth for live (P3).** The LiveVenue adapter separately enforces a short list of hard "last-line" caps: maximum order value, a price band and a maximum order rate. Their config is independent of the risk engine, so one bug cannot cause a catastrophe on its own.
7. **Soft limits and hard ceilings.** Soft limits are adjustable in the UI within a range. Hard ceilings, such as leverage = 1× or an absolute maximum order value, live only in the server config file and need a restart to change. Tightening any limit takes effect at once. Loosening is deliberate: typed confirmation, plus a cool-off period while live strategies run.
8. **Fast enough that nobody wants to bypass it.** Target under 5 ms per check at the 99th percentile, in-process.

### 2.3 Limit layers and beginner defaults ("Safe profile")

| Layer | Check | Beginner default | Notes |
|---|---|---|---|
| Instrument | Exchange filters: price tick, quantity step, min/max quantity, minimum order value ("min notional"), price bands | Always on | Same rules in backtest, paper and live (EX-010) |
| Per order | Maximum order value | 25% of the strategy's capital, plus an absolute cap (P3 pilot: $100) | REDUCE if allowed, else DENY |
| Per order | "Fat-finger" price band: how far a limit order may cross the current mid-price | 1% for BTC/ETH, 2% for others | A buy limit far *below* market is harmless; one far *above* is what fills at a terrible price |
| Per order | Expected slippage of a market order, estimated from the live order book | ≤ 0.5% | Otherwise DENY, or send as a protected IOC (EX-015) |
| Per order | Duplicate guard | Same symbol, side and size within 10 s → DENY | Catches loops and double clicks |
| Per strategy | Capital "sleeve" (PF-001): cannot spend more than allocated | Owner sets | |
| Per strategy | Max open positions | 3 | |
| Per strategy | Order rate / daily turnover | 10 orders/min, 200/day; turnover ≤ 5× sleeve/day | Runaway-strategy guard |
| Per strategy | Daily loss | 5% of sleeve → strategy REDUCE_ONLY until 00:00 UTC | Realized plus unrealized |
| Per strategy | Drawdown breaker | Smaller of 20% or 1.5× backtest max drawdown → strategy HALTED | Manual re-arm |
| Per strategy | Cooldown | No re-entry on a symbol for 3 bars after a stop-out; 24 h pause after 3 losses in a row | |
| Per asset | Max share of account equity in one coin | BTC/ETH 50%, others 25% | Stablecoins have their own cap (RK-033) |
| Cluster | Coins that move together count as one bet | "Crypto majors" cluster ≤ 75% of equity | Static in P1, correlation-driven in P2 |
| Per account | Daily / weekly loss | 3% / 7% of equity → account REDUCE_ONLY until the period resets | |
| Per account | Drawdown breaker | 15% → REDUCE_ONLY; 20% → HALTED, review required | |
| Per account | Leverage / negative balances | Not allowed (spot only) | Hard ceiling until P4 |
| Per exchange | Capital held on the exchange | Owner-set USD cap (live) | Counterparty risk (§5) |
| System | Stale data / clock drift / stablecoin peg | 10 s stale symbol; 1,000 ms clock offset; 1% depeg for 5 min | §4.4 and §5 |

Crypto trades 24/7, so "daily" resets at 00:00 UTC by default. The owner can pick a local timezone instead.

### 2.4 Requirements: risk architecture

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-001 | Ubik shall route every order-creating or order-amending command, in every mode (backtest, paper, demo, live), through the risk engine. The OMS shall refuse any order that lacks a valid risk-decision ID. | Must | P1 | Core invariant. Test: inject an unstamped order and expect refusal. |
| RK-002 | Strategy code shall have no access to venue adapters, credentials or OMS submit functions. It may only emit intents or order requests through the strategy context. | Must | P1 | Bypass-proof by construction. |
| RK-003 | The risk engine shall return APPROVE, REDUCE (downsize only) or DENY, with a reason code and a plain-language message. Each decision shall be persisted with an input snapshot. | Must | P1 | Explainability and audit. |
| RK-004 | Ubik shall support trading states ACTIVE, REDUCE_ONLY and HALTED at global, exchange, strategy and symbol scope. The most restrictive applicable state shall win, and cancels shall be allowed in all states. | Must | P1 | Mirrors NautilusTrader's TradingState. |
| RK-005 | Ubik shall enforce the per-order, per-strategy, per-asset, per-cluster, per-account and per-exchange limits in §2.3, configured in a versioned risk profile with the listed defaults. | Must | P1 | Static clusters in P1 (dynamic: PF-008). |
| RK-006 | The backtester and paper trader shall apply the same risk profile as live. Backtest reports shall list every order that was reduced or denied, by reason. | Must | P1 | Honesty principle; coordinate with 04. |
| RK-007 | Each risk-profile change shall be logged (time, old value, new value, source). Every backtest, paper run and live session shall record the risk-profile version it used. | Must | P1 | Reproducibility. |
| RK-008 | Hard ceilings (max leverage, absolute max order value, max risk per trade) shall be changeable only in the server config file followed by a restart, never via the UI or strategy code. | Should | P1 | Prevents in-the-moment loosening. |
| RK-009 | Loosening a limit while any live strategy is running shall require typed confirmation and take effect after a cool-off (default 24 h). Tightening shall apply immediately. | Must | P3 | Behavioural safeguard against tilt (emotional, loss-chasing decisions). |
| RK-010 | The LiveVenue adapter shall independently enforce last-line caps (max order value, price band, max orders per minute) from a config separate from the risk engine's. | Must | P3 | Defense in depth (cf. Rule 15c3-5). |
| RK-011 | A pre-trade check shall complete in under 5 ms at the 99th percentile for a single order. | Should | P1 | Measured in CI benchmarks. |

---

## 3. Position sizing

Position sizing answers "how much?", and it matters more than "when?". The same entry signal at 5× the size is a different and much riskier strategy.

| Method | Plain-language explanation | Formula (simplified) | Good for | Watch out |
|---|---|---|---|---|
| **Fixed amount** | "Always buy $500 worth." | qty = amount / price | First experiments | Ignores account size and volatility |
| **Fixed fraction of capital** | "Always use 10% of the strategy's money." | qty = f × capital / price | Simple growth with the account | Ignores how far away the stop is |
| **Fixed-fractional risk (stop-based)** | "If my stop is hit, I lose at most 1% of the strategy's money." | qty = (r × capital) / (entry − stop) | **Recommended default** | Needs a stop on every trade |
| **ATR-based** | Fixed-fractional risk with the stop placed a multiple of ATR away, so the stop is wider in wild markets and the position is automatically smaller | stop = entry − k × ATR; qty as above | **Recommended default** (k = 2) | ATR lags sudden volatility spikes |
| **Volatility targeting** | Size each position so its expected daily swing is a target, such as "20% a year." BTC's realized volatility often runs 40–70% a year, so a 20% target means holding about a third to a half of the sleeve in BTC. | weight = target vol / asset vol | Steadier equity curves; portfolio building (P2) | Volatility estimates jump in crises; cap the weight |
| **Fractional Kelly** | Kelly is the mathematically "growth-optimal" bet size given a known edge. Real edges are uncertain, and full Kelly can lose 50%+ along the way. | f\* = W − (1 − W)/R, then use ≤ 0.25 × f\* | Pro mode, with a long track record | Overestimated edge leads to over-betting. Not for beginners. |

(W = win rate, R = average win / average loss.)

**Worked example (beginner default).** Paper account: $10,000, with the BTC strategy's sleeve at $5,000. BTC is $60,000 and daily ATR(14) is $1,800. The stop goes 2 × ATR = $3,600 below entry, at $56,400. Risk = 1% × $5,000 = $50, so qty = $50 / $3,600 = 0.0139 BTC, which is a notional of about $833 (17% of the sleeve, under the 20% cap). Ubik tells the user: *"If BTC falls to $56,400, this trade will close at a loss of about $50 plus fees (1% of this strategy's money)."* The quantity is rounded **down** to the exchange's step size. If the result falls below the exchange's minimum order value, the trade is skipped with an explanation rather than rounded *up* (RK-016).

### Requirements: position sizing

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-012 | Ubik shall provide fixed-amount, fixed-fraction, fixed-fractional-risk and ATR-stop sizing methods, selectable per strategy and identical in all modes. | Must | P1 | Core methods. |
| RK-013 | New strategies in beginner mode shall default to 1% risk of sleeve per trade, a stop at 2×ATR(14), and a maximum position of 20% of the sleeve. Beginner mode shall allow 0.25–2% risk per trade; pro mode is hard-capped at 5%. | Must | P1 | Conservative default. |
| RK-014 | Before a strategy is enabled, Ubik shall show the computed size, the money at risk per trade in currency and as a percentage, and the loss after 10 consecutive stop-outs. | Must | P1 | Makes risk tangible (coordinate with 07). |
| RK-015 | Ubik shall provide volatility-targeting sizing with a configurable target, a lookback, and a per-asset maximum weight. | Should | P2 | Needed for portfolio allocation. |
| RK-016 | Sizing shall round quantity down to the step size. A trade below the venue's minimum quantity or minimum order value shall be skipped and explained, never rounded up. | Must | P1 | Rounding up silently increases risk. |
| RK-017 | Fractional Kelly shall be available only in pro mode, capped at 0.25×, and only after ≥ 100 closed paper or live trades, with a warning about estimation error. | Could | P2 | Backtest-only edges are unreliable. |

---

## 4. Protective mechanisms

### 4.1 Stops: exchange-native vs bot-managed

A **stop-loss** exits when price moves against you by a set amount. A **take-profit** exits at a target. A **trailing stop** follows the price up (for a long) and exits on a pullback of a set size.

| | Exchange-native (order rests at the exchange) | Bot-managed (Ubik watches the price and sends the exit) |
|---|---|---|
| Works if Ubik or its internet connection is down | **Yes** | No |
| Flexibility (time exits, indicator exits, partial exits) | Limited to the exchange's order types | **Full** |
| Balance effects | Locks the coin balance, and counts toward per-symbol order caps (e.g. Binance's `MAX_NUM_ALGO_ORDERS` [filter](https://developers.binance.com/docs/binance-spot-api-docs/filters)) | None until triggered |
| Support varies by exchange | Yes (see §8.1). Coinbase spot has no stop-market or native trailing stop. | Uniform |
| Crash behaviour | Stop-*market* fills but may slip badly. Stop-*limit* may not fill at all. | Same as native, plus Ubik's own reaction delay |

**Recommendation: a hybrid approach.** Ubik manages the precise stop itself. In live mode it also places a **native "catastrophe stop"** at about 1.5× the bot stop's distance, so the position is protected if Ubik dies. Paper and backtest simulate both, so results don't depend on which one fires. For protective exits, prefer stop-market where the venue supports it. Otherwise use stop-limit with a wide limit offset, because a stop-limit that doesn't fill during a crash protects nothing. 10 Oct 2025 showed this: [over $19B of leveraged positions were liquidated in about a day](https://www.coingecko.com/learn/october-10-crypto-crash-explained), and books on some venues thinned out dramatically.

### 4.2 Loss limits, breakers, cooldowns, concentration

These are the defaults in §2.3. Two design details matter:
- **Loss limits use marked-to-market equity.** "Marked-to-market" means open positions are valued at current prices, so the figure includes unrealized losses as well as closed trades. Otherwise a strategy could sit on a large open loss and never hit the limit.
- **Breakers have hysteresis**, meaning they don't flap on and off around the threshold. A drawdown HALT stays until a human re-arms it with a written note, such as "reviewed, market regime changed, reducing size." Daily and weekly REDUCE_ONLY states clear automatically when the period rolls over.

### 4.3 Kill switch

| Level | Action | Typical trigger |
|---|---|---|
| **L1 Pause** | Global REDUCE_ONLY: no new entries; exits and stops keep working | Manual; account daily-loss limit |
| **L2 Cancel** | L1, plus cancel every open order on every venue, verified by re-querying | Manual; runaway-strategy guard; suspected key compromise |
| **L3 Flatten** | L2, plus close all positions into the account's quote asset or fiat using protected orders (IOC limit with a price cap, retried in slices) | Manual only by default. Optional auto-trigger on the account HALT breaker (owner decision). |

Rules:
- The kill switch is reachable from the web UI (one button plus a confirmation), from the host CLI, and from an authenticated API endpoint. Doc 08 owns the authentication design.
- It runs in the engine's control loop with priority over strategy evaluation, so a hung strategy cannot block it.
- Re-arming is manual.
- In live mode a **dead-man's switch** cancels resting entry orders if the engine stops heart-beating. Some exchanges offer this natively, for example Kraken's [`cancel_all_orders_after`](https://docs.kraken.com/api-reference/order-management/dead-mans-switch) (refresh every 15–30 s with a 60 s timeout). Elsewhere a separate watchdog process does it.

### 4.4 Stale data and system health

A bot trading on frozen prices is a bot trading blind. For BTC/ETH on major exchanges, market data updates arrive many times per second, so silence is a signal.
- **Symbol stale:** no update for more than 10 s (configurable per pair), or a sequence gap in the order-book feed. New orders for that symbol are denied, and strategies don't evaluate on a bar that is missing or incomplete.
- **Venue stale:** more than 30 s → venue REDUCE_ONLY plus an alert.
- **Clock drift:** exchanges reject signed requests whose timestamps fall outside a window. On Binance this is `recvWindow`, which is now checked again just before matching, per the [changelog](https://developers.binance.com/docs/binance-spot-api-docs/CHANGELOG). Ubik measures the offset from exchange server time every 60 s and requires NTP (network time sync) on the host.

### Requirements: protective mechanisms

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-018 | Every strategy entry shall carry a defined protective exit. Beginner mode shall not allow strategies without a stop; pro mode requires an explicit acknowledgement. | Must | P1 | Without a stop, the risk per trade is undefined. |
| RK-019 | Ubik shall support bot-managed stop-loss, take-profit, trailing-stop and time-based exits in all modes, evaluated on every price update (live/paper) or on bar data (backtest), per 04's fill model. | Must | P1 | |
| RK-020 | In live mode Ubik shall place an exchange-native catastrophe stop (default 1.5× the bot-stop distance) wherever the venue supports one. Paper and backtest shall simulate it. | Should | P3 | Protection while Ubik is offline; designed in P1. |
| RK-021 | Ubik shall enforce daily and weekly loss limits on marked-to-market USD equity at strategy and account scope. A breach sets REDUCE_ONLY until the period resets. | Must | P1 | Defaults in §2.3. |
| RK-022 | Ubik shall enforce drawdown breakers (strategy: smaller of 20% or 1.5× backtest max drawdown → HALTED; account: 15% → REDUCE_ONLY, 20% → HALTED). Clearing a HALT shall require manual re-arm with a note. | Must | P1 | |
| RK-023 | Ubik shall enforce cooldowns: no re-entry on a symbol for N bars after a stop-out (default 3), and a strategy pause after K consecutive losses (default 3 losses → 24 h). | Should | P1 | Similar to Freqtrade's CooldownPeriod and StoplossGuard. |
| RK-024 | Ubik shall enforce maximum open positions per strategy (default 3) and per account (default 6). | Must | P1 | |
| RK-025 | Ubik shall enforce per-asset and cluster concentration caps (§2.3) on USD-valued equity. | Must | P1 | |
| RK-026 | Ubik shall provide kill-switch levels L1/L2/L3 (§4.3), reachable from the web UI, the host CLI and an authenticated API. It shall run with priority over strategy evaluation. | Must | P1 | Works in paper from day one. |
| RK-027 | After L2/L3, Ubik shall verify against the venue or simulator that open orders = 0 and positions are within dust tolerance, report the result, and raise a critical alert on failure. | Must | P1 | "Dust" = leftover amounts below the minimum order size. |
| RK-028 | In live mode, Ubik shall cancel resting entry orders when the engine heartbeat is missing for > 60 s, using the venue's native cancel-after where available, else a watchdog process. | Should | P3 | Dead-man's switch. |
| RK-029 | Ubik shall deny new orders on a symbol whose market data is stale (> 10 s default, or a book-sequence gap), and set venue REDUCE_ONLY after > 30 s of venue-wide staleness. | Must | P1 | Thresholds are configurable per pair. |
| RK-030 | Ubik shall measure clock offset to each venue every 60 s, warn above 500 ms, and deny new orders above 1,000 ms. | Must | P1 | Requires host NTP (06). |

---

## 5. Non-market risks

Market risk is "price moves against me." The risks below can lose money even when the strategy is right.

| Risk | Real example | Ubik control |
|---|---|---|
| **Exchange insolvency / fraud** | FTX (Nov 2022): customer funds were frozen and then tied up in bankruptcy | A per-exchange capital cap; hold only *trading* capital on exchanges; spread larger capital across 2 venues (P3); API keys never carry withdrawal permission (08) |
| **Exchange hack / withdrawal freeze** | Bybit (Feb 2025): about $1.4B stolen ([Crystal](https://crystalintelligence.com/investigations/the-10-biggest-crypto-hacks-in-history/)). Bitget (24 Sep 2026): about $387.5M from hot/warm wallets, withdrawals paused and resumed in phases over the following days ([BleepingComputer](https://www.bleepingcomputer.com/news/security/bitget-resumes-bitcoin-withdrawals-after-3875-million-crypto-heist/), [EPI](https://www.electronicpaymentsinternational.com/newsletters/bitget-to-resume-withdrawals-after-crypto-hack/)) | Same as above, plus a runbook (§9.5) |
| **Stablecoin depeg** | USDC fell to about $0.87 when $3.3B of reserves were stuck at SVB (Mar 2023) ([The Block](https://www.theblock.co/post/218993/usdc-and-dai-remain-at-about-0-90-following-circles-disclosure-of-funds-at-svb)). USDe hit about $0.65 *on Binance only* for about 90 minutes on 10 Oct 2025 while holding near $1 elsewhere ([Cointelegraph](https://cointelegraph.com/research/exclusively-obtained-orderbook-data-reveals-details-about-usde-crash), [Decrypt](https://decrypt.co/344026/binance-reimburses-283m-after-market-crash-and-asset-depegging-issues)). Smaller stablecoins have collapsed in 2026, e.g. Resolv USR. | Value stablecoins at a market reference price, never assume 1.00; peg monitor; per-stablecoin cap; prefer major fiat-backed stables as quote currency |
| **Liquidity** | Thin books during crashes. Altcoin pairs can have wide spreads even in calm markets. | Liquidity filter on the trading universe (minimum 24h volume); order size limited relative to visible depth; protected market orders |
| **Exchange outage / maintenance** | Exchanges degrade or halt during peak volatility, which is exactly when stops are needed | Stale-data halt; native catastrophe stops; reconcile before resuming; consume the venue's status/maintenance notices (P3) |
| **API errors / ambiguous results** | Binance documents that a timeout (`-1007`) or any HTTP 5xx means "execution status unknown": the order **may have been placed** ([limits](https://developers.binance.com/docs/binance-spot-api-docs/rest-api/limits), [errors](https://developers.binance.com/docs/binance-spot-api-docs/errors)) | UNKNOWN order state and resolution (§7.3) |
| **Rate limits / bans** | Binance returns 429 for over-limit requests. Ignoring these escalates to an IP ban (HTTP 418) lasting "from 2 minutes to 3 days." | Client-side rate limiting with headroom; on a ban, the venue goes HALTED |
| **API churn** | Binance removed its listenKey user-data streams on 2026-02-20 and retired legacy v1 endpoints on 2026-03-25 ([changelog](https://developers.binance.com/docs/binance-spot-api-docs/CHANGELOG)) | Adapter contract tests in CI and changelog monitoring (06) |
| **Clock drift** | Signed requests rejected; candles misaligned | §4.4 |
| **Software bug / runaway strategy** | Knight Capital, 2012 | Order-rate, turnover and duplicate guards; kill switch; staged rollout |
| **Leverage / liquidation (P4)** | 10 Oct 2025: more than $19B liquidated in about 24 h. Exchange-internal collateral pricing amplified the cascade. | No leverage before P4 (RK-038). P4 controls are in RK-039. |

**Leverage primer for P4.** With leverage you borrow to hold a bigger position. If the position loses enough that your collateral falls below the "maintenance margin," the exchange **liquidates** it: it force-closes at market, often at the worst moment. Perpetual futures also charge or pay a **funding rate** every few hours. In extreme events, exchanges may **auto-deleverage (ADL)**, closing *profitable* opposing positions. P4 must model all of these before any leveraged order is allowed.

### Requirements: non-market risks

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-031 | Ubik shall value all balances, P&L and limits in the owner's reference currency (default USD), pricing each stablecoin at a market reference price rather than 1.00. | Must | P1 | Depeg must show up in equity. |
| RK-032 | Ubik shall monitor the peg of each held stablecoin: warn at > 0.5% deviation; at > 1% for 5 min, set REDUCE_ONLY on pairs quoted in it and alert. | Should | P1 | Must in P3. Uses ≥ 2 reference sources, so one exchange's glitch (the USDe case) doesn't trigger it alone. |
| RK-033 | Ubik shall cap the share of equity held in any single stablecoin (default 60%) and warn when exceeded. | Should | P2 | Diversify the "cash" leg. |
| RK-034 | Ubik shall restrict the trading universe to pairs above a minimum 24h volume (default $50M), and deny or split orders larger than 10% of the visible book depth within 0.5% of mid. | Must | P1 | Liquidity risk. |
| RK-035 | Ubik shall track the USD value held on each exchange against an owner-set cap and alert when it is exceeded, suggesting a manual withdrawal of the excess. | Must | P3 | Counterparty cap (the FTX/Bitget lesson). Ubik never withdraws. |
| RK-036 | Ubik shall set a venue to HALTED when its error/reject rate exceeds a threshold (default > 20% of requests over 5 min) or on any IP ban (HTTP 418), and alert. | Must | P3 | Exercised in P1 via paper fault injection (EX-021). |
| RK-037 | Ubik shall consume venue status and scheduled-maintenance information and pre-emptively set the venue to REDUCE_ONLY during announced maintenance. | Should | P3 | |
| RK-038 | Ubik shall deny any order that would make a spot balance negative, or that requires margin, borrowing or leverage, until P4 features are explicitly enabled. | Must | P1 | Hard ceiling. |
| RK-039 | In P4, Ubik shall enforce max leverage (default 2×), isolated margin by default, a minimum liquidation distance (default ≥ 3× daily ATR), funding-cost accrual in P&L, and liquidation and ADL scenarios in stress tests. | Must | P4 | Derivatives. |

---

## 6. Portfolio construction and capital allocation

### 6.1 Sleeves: many strategies, one account

Each strategy runs in a **sleeve**: a virtual sub-account with its own allocated capital, balances, positions, fills and P&L. The sleeves plus an unallocated reserve must always add up exactly to the real account. That gives every strategy an honest stand-alone track record, lets limits apply per strategy, and behaves identically in backtest, paper and live. A sleeve can only sell coins it holds and spend cash it holds, so on spot one strategy can never accidentally sell another's position. Exchange sub-accounts can be mapped to sleeves later (P3, optional). Internal netting of opposite orders from different sleeves saves fees but complicates accounting, so it is deferred to P4.

**Manual trading on the same exchange account** breaks this invariant. Ubik detects activity it did not originate, puts it in an "External" sleeve, and alerts (EX-030). The recommendation is a dedicated exchange account or sub-account for Ubik.

### 6.2 Allocation methods

| Method | Plain language | When to use |
|---|---|---|
| **Manual fixed** | "Strategy A gets $3,000, B gets $2,000." | P1 default |
| **Equal weight** | Every strategy gets the same capital | Simple, robust baseline |
| **Inverse volatility (volatility-scaled)** | Calmer strategies get more capital, so each contributes similar day-to-day swings | P2: when strategies differ a lot in volatility |
| **Risk parity (equal risk contribution)** | Like inverse volatility, but also accounts for strategies that move together, so two strategies that are really the same bet don't double the risk | P2: 3+ strategies |
| **Performance-based with decay** | Give more to strategies doing well recently, with older results counting less (exponential decay) | P2, pro mode only. Recent performance is mostly noise for most retail strategies, so shrink heavily toward equal weight and cap each strategy. |

**Rebalancing.** Rebalance on a calendar (default monthly), or when a sleeve drifts more than 25% (relative) from its target, whichever comes first. Moving *cash* between sleeves is a bookkeeping entry, not a trade. Trades happen only if a sleeve must shrink a position to fit its new allocation, and those trades go through the risk engine like any other.

### 6.3 Correlation, VaR and Expected Shortfall

**Correlation** measures how much two things move together, from −1 (opposite) through 0 (unrelated) to +1 (in lockstep). BTC, ETH and SOL are usually strongly positively correlated, and in crashes **correlations jump toward 1**. Diversifying across crypto majors gives less protection than it appears to. Ubik shows rolling 30- and 90-day correlations of assets *and of strategy returns*, and warns when active strategies are effectively the same bet.

**Value at Risk (VaR)** answers: "On a normal bad day, how much could I lose?" A 1-day 95% VaR of $300 on a $10,000 portfolio means that on about 19 days out of 20 the loss should be smaller than $300. It says nothing about the 20th day.

**Expected Shortfall (ES, also called CVaR)** answers the follow-up: "On those worst 1-in-20 days, how much do I lose *on average*?" For example, $520. ES is the more honest number for crypto, which has "fat tails": big moves happen far more often than a bell curve predicts.

Ubik computes both by **historical simulation**: it takes each of the last N days' actual price moves, applies them to today's holdings, and reads the loss distribution. Formula-based ("parametric") VaR assumes a bell curve and underestimates crypto risk, so it is not used. Because VaR built from calm periods understates crash risk, Ubik adds **stress tests** that replay named crises against the current portfolio: 12 Mar 2020, May 2021, June 2022 (Celsius/3AC), Nov 2022 (FTX), Mar 2023 (USDC), 10 Oct 2025. It also applies hypothetical shocks, such as "−30% across crypto," "USDT −10%," and "exchange X frozen: 100% of balance unavailable."

### 6.4 Exposure dashboard

P1 shows, at a glance:
- **Totals:** equity in USD, cash vs crypto, and gross exposure (sum of all positions).
- **Breakdowns:** exposure by asset, by stablecoin, by exchange and by strategy.
- **Limit usage:** a "fuel gauge" for each limit, showing the percentage used, colored green/amber/red.
- **Status:** the current trading state at every scope, and the latest risk decisions.

P2 adds VaR, ES, stress-test results, and the correlation heatmap. Doc 07 owns the visual design.

### Requirements: portfolio and capital allocation

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| PF-001 | Each strategy shall run in a sleeve with its own capital, balances, positions, fills and P&L in every mode. Sleeves plus the unallocated reserve shall equal account balances exactly, checked on every fill. | Must | P1 | Invariant tested continuously. |
| PF-002 | A sleeve shall only sell assets it holds and spend cash it holds; cross-sleeve borrowing is not allowed. | Must | P1 | |
| PF-003 | Ubik shall keep an unallocated cash reserve (default 10% of equity) that strategies cannot use. | Should | P1 | Buffer for fees and rebalancing. |
| PF-004 | Ubik shall support manual-fixed and equal-weight allocation. | Must | P1 | |
| PF-005 | Ubik shall support inverse-volatility and risk-parity allocation with configurable lookback and per-strategy min/max weights. | Should | P2 | |
| PF-006 | Ubik shall support performance-based allocation using exponentially decayed metrics (default half-life 60 days), 50% shrinkage toward equal weight, a 40% per-strategy cap, and a minimum paper/live track record of 90 days or 50 trades. Backtest-only results do not count. | Could | P2 | Guards against chasing noise. |
| PF-007 | Ubik shall rebalance sleeves on a calendar (default monthly) and on drift thresholds (default 25% relative). Sleeve cash moves shall be ledger transfers; any trades required shall pass the risk engine. | Should | P2 | |
| PF-008 | Ubik shall compute rolling 30/90-day correlations of assets and strategy returns, derive correlation clusters (threshold default 0.7) for RK-025, and alert when active strategies' average pairwise correlation exceeds 0.7. | Should | P2 | Replaces static clusters. |
| PF-009 | Ubik shall compute 1-day 95% and 99% VaR and ES by historical simulation over ≥ 1 year of data, with a plain-language sentence for each figure. | Should | P2 | |
| PF-010 | Ubik shall run historical-replay and hypothetical stress tests (§6.3) against the current portfolio, on demand and weekly. | Should | P2 | |
| PF-011 | Ubik shall provide the exposure dashboard in §6.4, with limit-utilization gauges. | Must | P1 | Basic version in P1. |
| PF-012 | Ubik shall backtest multi-strategy portfolios with sleeves, allocation, rebalancing and the shared risk engine. | Should | P2 | Coordinate with 04. |
| PF-013 | Ubik shall optionally map sleeves to exchange sub-accounts in live mode. | Could | P3 | Where the exchange and account tier allow it. |

---

## 7. Order management system (OMS)

### 7.1 Order lifecycle

```
 PROPOSED ──risk deny──► DENIED                     (never left Ubik)
    │ risk approve
    ▼
 PENDING_SUBMIT ──venue reject──► REJECTED
    │ ack                │ timeout / 5xx / disconnect
    ▼                    ▼
  OPEN ◄──resolved──── UNKNOWN ──not found after grace──► REJECTED (NOT_FOUND)
  (resting, or untriggered stop)
    │ fill        │ cancel request          │ time-in-force / venue expiry
    ▼             ▼                         ▼
 PARTIALLY_FILLED ──► FILLED      CANCEL_PENDING ──► CANCELED       EXPIRED
         └──── cancel / expire with partial fill → CANCELED / EXPIRED (filled qty kept)
```

Every transition is an appended event: order created, risk decision, sent, acknowledged, fill(s), cancel requested, cancel confirmed, and so on. Current state is derived from these events (**event sourcing**), which gives a complete audit trail and makes crash recovery deterministic. Each event is written to the database *before* the network call (a **write-ahead** log), so after a crash Ubik knows which orders might be in flight.

### 7.2 Client order IDs and idempotency

Ubik generates a **client order ID** before sending, for example `ubk-L-trend01-000123` (mode, strategy, sequence). It must fit the venue's limits, such as 36 characters on Binance. This ID is what makes the system **idempotent**, meaning "doing it twice has the same effect as once." Venues behave differently:
- Coinbase Advanced returns the existing order if the same `client_order_id` is reused ([create order](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/orders/create-order)).
- Binance rejects a duplicate ID only while the original order is still open.

The OMS therefore never relies on the venue to dedupe. After any ambiguous result it **asks first, then acts**.

### 7.3 Retries, rate limits, unknown states

- **Retries.** Reads and cancels retry with exponential backoff plus jitter (randomized waits, e.g. 0.5 s doubling to 8 s, at most 5 tries). A **new order is never blindly resent** after an ambiguous result.
- **UNKNOWN orders.** On timeout, 5xx, Binance `-1007`, or a disconnect mid-request, the order is marked UNKNOWN. New orders for that symbol and sleeve are blocked, and the OMS listens on the user-data stream while querying by client order ID. If the order isn't resolved within 30 s, the symbol goes REDUCE_ONLY and an alert fires. If the venue reports "not found" after a grace window, the order becomes REJECTED (NOT_FOUND).
- **Rate limits.** Ubik keeps client-side token buckets per venue that mirror the published limits, using at most 70% of each. Binance publishes its limits in `exchangeInfo`: e.g. a REQUEST_WEIGHT budget per minute per IP and an unfilled-order count per account ([limits](https://developers.binance.com/docs/binance-spot-api-docs/rest-api/limits)). Kraken uses a per-key decaying counter, with a separate limiter for trading calls ([rate limits](https://docs.kraken.com/exchange/guides/rest/ratelimits)). The send queue is prioritized: **cancels > protective exits > entries > data queries**. Ubik honors `Retry-After` on 429 and halts the venue on 418.

### 7.4 Reconciliation

Reconciliation compares what Ubik *thinks* (its ledger and open orders) with what the venue *says* (balances, open orders, recent fills). It runs:
- at startup;
- after every reconnect;
- every 60 s;
- after every kill-switch action.

In live mode the **exchange is the source of truth**. A mismatch beyond dust tolerance puts the affected scope in REDUCE_ONLY and raises an alert, and the fix is recorded as an explicit adjustment entry, never a silent overwrite. In paper mode the same code reconciles against the simulator's own exchange-like account API. That keeps the code path exercised long before real money. Live startup begins in REDUCE_ONLY until the first reconciliation passes.

### 7.5 Exact accounting

- **Decimals, never floats.** Binary floating point cannot represent 0.1 exactly, and the errors accumulate. All prices, quantities, balances and fees use Python `Decimal` or integer minor units, enforced by lint rules and property-based tests.
- **Instrument rules.** These include tick size, step size, min/max quantity, min/max order value, percent-price bands, max open orders, and order-type capability flags (Binance exposes e.g. `ocoAllowed`, `pegInstructionsAllowed`, `opoAllowed`). They are fetched from each venue, versioned, refreshed daily and after rule-related rejections, and **applied identically in backtest, paper and live**. Binance's [filters](https://developers.binance.com/docs/binance-spot-api-docs/filters) are the reference set. Since 2026, Binance also expires taker orders that would execute outside a dynamic band around a reference price, the "[Price Range Execution Rule](https://developers.binance.com/en/docs/products/spot/CHANGELOG)," so the paper simulator must model expiries for that reason too.
- **Rounding.** Quantity is rounded down to the step size. Limit prices are rounded to tick in the conservative direction (buy down, sell up).
- **Fees.** Each fill records the fee amount, the fee asset (base, quote, or a third token such as BNB), the rate, maker vs taker, and the USD value at fill time. The account's actual fee tier is fetched from the venue. Fee levels differ a lot: entry-tier spot taker fees range from about 0.1% on Binance to about 0.6% on Coinbase Advanced, and Kraken restructured its tiers in July 2026 ([comparison](https://brokerchooser.com/news/coinbase-vs-kraken-fee-structures-and-trading-features-compared--bbeea493)). A round trip at 0.6% + 0.6% = 1.2% wipes out a strategy that averages +0.8% per trade.
- **Double-entry ledger.** Every fill, fee, transfer between sleeves and adjustment is posted as balanced entries. For example, buying 0.01 BTC posts +0.01 BTC, −600 USDT and −0.6 USDT fee, against the sleeve and the venue account. **Tax lots** (acquisition time and cost) are kept, so doc 08 can produce FIFO, specific-ID or average-cost reports per jurisdiction.

Indicative table sketch (doc 06 owns the final schema):

```
orders(order_id, client_order_id, venue, account, sleeve_id, symbol, side, type, tif,
       qty, limit_price, stop_price, state, risk_decision_id, mode, created_at)
order_events(order_id, seq, event_type, payload_json, venue_ts, local_ts)
fills(fill_id, order_id, venue_trade_id, qty, price, fee_amount, fee_asset, liquidity[maker/taker], ts)
ledger_entries(entry_id, txn_id, account, sleeve_id, asset, amount_decimal, kind, ref_id, ts)
risk_decisions(decision_id, request_json, outcome, reason_code, inputs_snapshot_json, profile_version, ts)
```

### Requirements: OMS

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| EX-001 | Ubik shall manage orders through the state machine in §7.1. Undefined transitions shall be rejected, logged and alerted. | Must | P1 | |
| EX-002 | Ubik shall store every order and risk event append-only, derive current state from events, and persist each event before the corresponding network send. | Must | P1 | Event sourcing plus write-ahead. |
| EX-003 | Ubik shall assign each order a deterministic, account-unique client order ID before submission, within each venue's length and charset limits. | Must | P1 | Idempotency key. |
| EX-004 | Ubik shall retry reads and cancels with exponential backoff and jitter. It shall never resubmit a new order after an ambiguous result without first querying by client order ID. | Must | P1 | Prevents duplicate orders. |
| EX-005 | Ubik shall handle UNKNOWN orders per §7.3: block the symbol/sleeve, resolve within 30 s via stream plus query, else REDUCE_ONLY and alert; NOT_FOUND after a grace window becomes REJECTED. | Must | P1 | Exercised in paper via fault injection. |
| EX-006 | Ubik shall enforce client-side per-venue rate limits at ≤ 70% of published limits, with the priority order cancels > protective exits > entries > queries, honoring Retry-After. | Must | P1 | Modeled in paper; real in P3. |
| EX-007 | Ubik shall reconcile open orders, balances and recent fills at startup, on reconnect, every 60 s and after kill-switch actions. It shall set REDUCE_ONLY on mismatches and record corrections as adjustment entries. | Must | P1 | Against the simulator in paper. |
| EX-008 | Live mode shall start in REDUCE_ONLY until the first reconciliation passes. | Must | P3 | |
| EX-009 | Ubik shall use exact decimal arithmetic for every monetary and quantity value. Binary floats are prohibited in accounting and order paths. | Must | P1 | Enforced by lint and property tests. |
| EX-010 | Ubik shall load, version and refresh each venue's instrument rules and order-type capabilities, and apply them identically in backtest, paper and live. | Must | P1 | Sim-live parity. |
| EX-011 | Ubik shall round quantities down to the step size and round limit prices to tick in the conservative direction. | Must | P1 | |
| EX-012 | Ubik shall record per-fill fee amount, asset, rate, maker/taker and USD value, and use the account's actual fee tier (or an owner-chosen tier in paper). | Must | P1 | |
| EX-013 | Ubik shall maintain a double-entry ledger, by sleeve and venue account, preserving tax lots. | Must | P1 | Tax exports in 08. |

---

## 8. Execution

### 8.1 Order types: what, when, and who supports them

Support was checked against public API docs in Oct 2026: [Binance spot](https://developers.binance.com/docs/binance-spot-api-docs/CHANGELOG), [Coinbase Advanced Trade](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/orders/create-order), [Kraken AddOrder](https://docs.kraken.com/api-reference/trading/add-order). Adapters must read capability flags at runtime rather than hard-coding this table.

| Type | What it does | Use for | Watch out | Binance | Coinbase Adv. | Kraken |
|---|---|---|---|---|---|---|
| Market | Fill now at the best available prices | Urgent exits | Slippage. Ubik sends a *protected* version (EX-015). | MARKET | market_market_ioc | market |
| Limit (GTC/GTD) | Rest at your price or better (good-till-canceled / good-till-date) | Entries, take-profits | May never fill | LIMIT | limit_limit_gtc / gtd | limit |
| Post-only | Limit that is canceled if it would trade immediately, which guarantees the maker fee | Fee-sensitive, patient entries | Rejected in fast markets | LIMIT_MAKER | post_only flag | `post` flag |
| IOC | Fill what you can now, cancel the rest | Protected market orders, slices | Partial fills | timeInForce=IOC | sor_limit_ioc | IOC |
| FOK | All or nothing, immediately | Exact-size needs | Often no fill | timeInForce=FOK | limit_limit_fok | Not offered on spot REST (emulate or avoid) |
| Stop-market | Becomes a market order at the trigger | Protective stops | Gap slippage | STOP_LOSS | Not on spot (use stop-limit) | stop-loss |
| Stop-limit | Becomes a limit order at the trigger | Stops with a price cap | May not fill in a crash | STOP_LOSS_LIMIT | stop_limit_stop_limit_gtc / gtd | stop-loss-limit |
| Trailing stop | Stop that follows price | Locking in gains | Whipsaws | `trailingDelta` in bps on stop/TP types ([FAQ](https://developers.binance.com/docs/binance-spot-api-docs/faqs/trailing-stop-faq)) | Not native (bot-managed) | trailing-stop(-limit) |
| OCO / bracket | Linked orders: when one fills, the other cancels (e.g. TP + SL) | Exit pairs | Count toward order caps | OCO, OTO, OTOCO; OPO since Dec 2025 | trigger_bracket_gtc / gtd; attached TP/SL | Conditional close on the entry order |
| Iceberg | Shows only part of the size | Larger orders in thin books | | icebergQty (up to 100 parts in 2026) | None | iceberg (display ≥ 1/15) |
| Native TWAP / scaled | Exchange slices the order over time or across prices | Accumulation | | Separate Algo API (not evaluated) | twap_limit_gtd, scaled_limit_gtc | None |
| Pegged | Price follows best bid/ask | Passive execution | | Pegged orders (since Aug 2025) | None | None |

**Defaults.**
- **Entries:** limit or post-only when there is no urgency.
- **Exits on stop:** protected market.
- **Take-profit:** resting limit.
- **Never send raw market orders** into a book Ubik hasn't checked.

Where a venue lacks a type, Ubik emulates it bot-side and labels it "emulated" in the UI. An emulated stop does not protect you while Ubik is offline.

### 8.2 Execution algorithms: personal scale vs institutional

For BTC and ETH on top exchanges, an order of a few thousand dollars is a tiny fraction of the visible book, so **market impact** (your own order pushing the price) is negligible. The costs that dominate are **fees** and **the bid-ask spread**. At personal scale the best execution lever is choosing maker over taker where urgency allows. Algorithms start to matter for thin altcoins, or when an order exceeds about 1% of the depth near mid.

| Algorithm | Plain language | Matters for Ubik when… | Phase |
|---|---|---|---|
| Protected market | IOC limit with a price cap; unfilled remainder reported | Always: the default "urgent" order | P1 |
| Limit-chase / peg | Sit at the best bid (or ask) and re-price every N seconds up to K times or X bps, then fall back to protected market | Saving fees on non-urgent entries | P2 |
| TWAP | Split into equal slices over a period, with randomized timing and size | Order large relative to depth; thin pairs; DCA-style accumulation | P2 |
| Iceberg | Show only a slice of the order | Large resting orders | P2 |
| VWAP | Trade in proportion to the typical volume curve | Institutional benchmark; little value at personal size, and 24/7 crypto volume curves are noisy | P4 |
| POV (percentage of volume) | Be X% of market volume | Large orders | P4 |
| Smart order routing | Split across exchanges for the best price | Several funded venues; arbitrage | P4 |

### 8.3 Transaction cost analysis (TCA)

TCA measures what a trade *really* cost compared with the moment the strategy decided to trade. The core measure is **implementation shortfall**. Worked example: the strategy decides to buy 0.05 BTC when mid = $60,000 (the **decision price**). The order leaves 400 ms later with mid at $60,010 (**arrival price**; delay cost 1.7 bps). The average fill is $60,030 (spread and impact, another 3.3 bps). The taker fee is 0.10% (10 bps). Total shortfall is 15 bps, about $4.50 on $3,000. A backtest that assumed 5 bps total cost understated reality by 3×. For limit orders that never fill, **opportunity cost** counts too: the move you missed on the unfilled quantity.

**Feedback loop.** Ubik aggregates TCA by venue, pair, order type, size bucket and volatility regime. Doc 04's backtest cost model is then calibrated from it, defaulting to the **75th percentile** of observed slippage plus actual fees rather than the average, so the backtest leans conservative. The UI shows "backtest assumed X bps / reality Y bps" for each strategy. Paper fills are simulated against the live order book and are tagged *simulated*. They cannot capture queue position or your own market impact, so they are a floor on real costs, not an estimate of them.

### Requirements: execution and TCA

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| EX-014 | Ubik's internal order model shall support market, limit (GTC/GTD), post-only, IOC, FOK, stop-market, stop-limit, take-profit, trailing stop and OCO/bracket. Adapters map each to a native type or a labelled bot-side emulation. | Must | P1 | Native mapping exercised in P2 (demo) and P3. |
| EX-015 | Market orders shall by default be "protected": sent as IOC limit orders capped at 0.5% through the best price, with any unfilled remainder reported. | Must | P1 | Fat-finger and crash protection. |
| EX-016 | Ubik shall provide a limit-chase (peg) algorithm with configurable re-price interval, max re-prices, max drift and fallback. | Should | P2 | Maker-fee savings. |
| EX-017 | Ubik shall provide TWAP and iceberg algorithms (bot-side with randomization; native where available). | Could | P2 | |
| EX-018 | Ubik shall provide VWAP, POV and cross-venue smart order routing. | Could | P4 | Institutional-scale features. |
| EX-019 | Ubik shall compute per-order TCA: decision price, arrival mid, spread at arrival, average fill, fees, implementation shortfall (bps and currency), fill ratio, time to fill and opportunity cost, tagged simulated or real. | Must | P1 | |
| EX-020 | Ubik shall aggregate TCA and calibrate backtest cost assumptions from it (default 75th-percentile slippage), showing assumed vs observed costs per strategy. | Should | P2 | Comparison view in P1; automatic calibration in P2. |
| EX-021 | The paper venue shall behave like an exchange: asynchronous acknowledgements with latency, partial fills, filter rejections, rate limits, price-band expiries, and configurable fault injection (timeouts, disconnects, unknown states). | Must | P1 | Exercises OMS error paths before P3; fill model owned by 04. |

---

## 9. Path to live (P3)

### 9.1 Staged rollout

| Stage | What runs | Money | Purpose | Minimum to graduate (defaults) |
|---|---|---|---|---|
| **S0 Backtest** | Historical data | None | Is there an edge? | Passes 04's robustness checks (out-of-sample, walk-forward); ≥ 100 trades |
| **S1 Paper** | Live data plus Ubik's simulator | Simulated | Behaviour on unseen data; ops stability | ≥ 30 days and ≥ 20 trades; results within the expectation band; 0 unresolved reconciliation breaks; uptime ≥ 99% |
| **S2 Exchange demo** | Real exchange API, demo funds, e.g. [Binance Spot Demo Mode](https://developers.binance.com/docs/binance-spot-api-docs/demo-mode/general-info) (since Jan 2026; same features as live, unlike testnet) | Exchange play money | Prove the *real adapter*: order types, filters, errors, streams | Adapter test suite passes; 7 days with no unresolved UNKNOWN orders |
| **S3 Shadow** | Live account connected **read-only**; would-be orders priced against the live book, not sent | Real balances, nothing sent | Real fee tier, rules and balances; decision-to-book latency | ≥ 14 days; shadow TCA within the cost model |
| **S4 Pilot** | Live trading under hard caps | Tiny (default ≤ $500 per strategy, ≤ $100 per order) | Real fills, real slippage, real operations | ≥ 30 days and ≥ 20 trades; slippage ≤ model + 10 bps; drawdown ≤ 1.25× backtest max; 0 severity-1 incidents |
| **S5 Scaled** | Live | At most 2× step-ups per 30 days, up to the owner's cap | Grow carefully | Continuous; demotion rules apply |

**"Within the expectation band"** means live/paper per-trade returns are no worse than the 10th percentile of a bootstrap (resampling) of the backtest's trades, and drawdown has not exceeded the backtest's worst. The aim is to catch a strategy that only worked in hindsight. It does not require live to beat the backtest.

**Demotion is automatic; promotion needs the owner's approval.** Demotion triggers:
- Drawdown > 1.5× backtest max → back to S1 (paper).
- Slippage > 2× the model for 2 weeks → size cut back to pilot.
- Any reconciliation break or UNKNOWN order unresolved for > 15 min → strategy HALTED and reviewed.
- Three severity-2 incidents in 30 days → down one stage.

### 9.2 Pre-live checklist (per strategy and account)

1. The strategy artifact (code hash, parameters, risk profile) is identical to the one that passed S1–S3.
2. API key is trade-only, with withdrawals disabled, an IP allowlist and secure storage (08).
3. A dedicated exchange account or sub-account is used, with no manual trading on it.
4. Per-exchange capital cap is set, and only pilot capital is deposited.
5. Kill switch L1–L3 tested on the demo venue within the last 7 days, and the result is recorded.
6. Native catastrophe stops are verified on the venue.
7. Alerts reach the owner on two independent channels; a test alert has been acknowledged.
8. Clock sync is OK, and the host has auto-restart and an uninterruptible power/network plan (or runs in the cloud).
9. Owner has read the runbooks and acknowledged the max-loss figure: "worst case for this pilot: $X."
10. Jurisdiction and exchange ToS checks are done (08).

### 9.3 Monitoring and alerting

**Monitored metrics:**
- **Health:** engine heartbeat; market-data freshness per symbol; order acknowledgement latency.
- **Order flow:** reject and error rates; UNKNOWN order count; reconciliation status.
- **Money and risk:** P&L against each limit; exposures; capital per exchange.
- **Environment:** clock offset; stablecoin deviation; host CPU, disk and DB health.

An **external heartbeat check** (a cheap outside service pinged by Ubik) alerts if Ubik itself goes silent, because a dead process can't send alerts.

| Severity | Examples | Delivery |
|---|---|---|
| Sev-1 (act now) | Kill switch fired; live reconciliation break; unresolved UNKNOWN order; IP ban; foreign activity on the account; account HALTED | Push plus a second channel, repeated until acknowledged |
| Sev-2 (act today) | Loss limit hit; data stale > 5 min; depeg warning; strategy demoted | Push |
| Sev-3 (info) | Daily summary; TCA drift; limit utilization > 80% | Email/in-app |

### 9.4 Incident runbooks (summaries; full versions in-app)

| Incident | Detect | Contain (automatic → manual) | Recover | Review |
|---|---|---|---|---|
| **Exchange down/degraded** | Stale data, 5xx burst, status page | Venue REDUCE_ONLY. Native catastrophe stops remain. Don't market-sell into a recovering book. | Reconcile; verify open orders; re-enable the venue in steps | Did the stops behave? |
| **Runaway strategy** | Order-rate, turnover or duplicate guard; P&L spike | Strategy HALTED, L2 for its orders | Compare intents vs orders in the logs; fix; re-backtest; **demote to paper** | Root cause and new test |
| **API key compromise** | Foreign orders or fills in reconciliation, exchange security emails, unexpected balance changes | L2 kill switch; owner revokes the key on the exchange (Ubik shows the steps); check that withdrawal allowlists are intact | New key; reconcile; resume at pilot | With 08 |
| **Stablecoin depeg** | Peg monitor | REDUCE_ONLY on affected pairs | Owner decides: hold, or convert to fiat or another stable | Update caps |
| **Reconciliation mismatch** | Recon loop | Scope REDUCE_ONLY | Find the cause (missed fill event, BNB fee, manual trade); post an adjustment entry | Fix the gap |

### 9.5 What P1 must get right so P3 is a configuration change

1. Strategies emit intents only. The pipeline is identical in all modes, and only the venue adapter swaps (`BacktestVenue`, `PaperVenue`, `DemoVenue`, `LiveVenue`), all implementing one interface.
2. The paper venue behaves like an exchange: asynchronous, fallible and rate-limited (EX-021).
3. Real instrument rules and fee schedules are used everywhere (EX-010, EX-012).
4. Decimal accounting, a double-entry ledger, sleeves and tax lots exist from day one.
5. Event-sourced orders with deterministic client IDs, plus UNKNOWN-state handling.
6. Reconciliation runs in paper too.
7. Trading states, the kill switch and alerting exist in P1.
8. Mode is configuration per account and per strategy. The strategy API does not expose the mode to strategy logic, so code cannot behave differently live (EX-029).
9. All timestamps are UTC with exchange time and local time both recorded, using a monotonic clock for timeouts.
10. Secrets go through an abstraction (08), even if paper mode needs none.

### Requirements: path to live

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| EX-022 | Ubik shall track a rollout stage (S0–S5) per strategy × account. Stage changes shall be logged; promotion requires the checklist plus owner approval, and demotion is automatic per §9.1. | Must | P3 | Stage tracking for S0–S1 from P1. |
| EX-023 | Ubik shall support an exchange demo/testnet venue adapter using the same code as the live adapter, differing only in endpoints and keys. | Should | P2 | Proves the adapter before real money. |
| EX-024 | Ubik shall support shadow mode: live balances and rules read via a read-only key, orders generated and priced against the live book with TCA, nothing sent. | Should | P3 | |
| EX-025 | Pilot stage shall enforce hard caps (defaults ≤ $500 per strategy, ≤ $100 per order) in the LiveVenue last-line checks. | Must | P3 | |
| EX-026 | Ubik shall show a graduation scorecard per strategy (criteria in §9.1, current values, pass/fail). | Must | P3 | |
| EX-027 | Ubik shall emit the metrics in §9.3, deliver alerts by severity over ≥ 2 independent channels with acknowledgement for Sev-1, and support an external heartbeat check. | Must | P3 | Basic in-app alerts in P1. |
| EX-028 | Ubik shall include in-app runbooks for the incidents in §9.4, with one-click containment actions where applicable. | Should | P3 | |
| EX-029 | Switching a strategy to live shall be configuration only (same artifact hash), and shall require trade-only credentials, typed confirmation and per-strategy opt-in. The strategy API shall not expose the trading mode to strategy logic. | Must | P3 | Sim-live parity. |
| EX-030 | Ubik shall detect orders, fills and balance changes it did not originate, attribute them to an "External" sleeve, and raise a Sev-1 alert in live mode. | Should | P3 | Manual trades or a compromised key. |

---

## 10. Beginner guardrails

**Conservative defaults.** The "Safe profile" in §2.3 is the default in every mode, and paper trading uses it too, so paper results are honest about what live would allow. Pro mode unlocks wider soft limits but never the hard ceilings. Doc 07 owns how pro mode is unlocked (e.g. a short quiz).

**Explain every block.** Each denial or reduction produces a message with five parts: *what* happened, *which limit*, *current vs allowed*, *why the limit exists*, and *what you can do*. Example:

> **Ubik didn't place this BTC buy.** It would have raised Bitcoin to 62% of your account; your limit is 50%. *Why this limit exists:* if most of your money is in one coin, one bad day for that coin becomes a bad day for your whole account. *What you can do:* wait for an existing BTC position to close, or lower this strategy's position size. [Learn more: concentration risk]

Reason codes (e.g. `RISK_ASSET_CAP`, `RISK_DAILY_LOSS`, `RISK_PRICE_BAND`, `RISK_STALE_DATA`, `RISK_CLOCK_DRIFT`, `RISK_DEPEG`, `RISK_LIQUIDITY`, `RISK_ORDER_RATE`, `RISK_DUPLICATE`, `RISK_STATE_HALTED`, `VENUE_MIN_NOTIONAL`) each map to one glossary entry.

**Confirm risky changes with consequences, not just "Are you sure?"** Example: *"At 3% risk per trade, 10 losing trades in a row (which happens) would cost about $2,630, or 26% of this strategy's money. Type RAISE RISK to confirm."*

### Requirements: beginner guardrails

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| RK-040 | New installations shall default to the Safe profile (§2.3) in all modes. Switching to pro mode shall require an explicit acknowledgement flow defined in 07. | Must | P1 | |
| RK-041 | Every risk denial or reduction shall display a plain-language message containing what, which limit, current vs allowed values, why the limit exists, and next steps, linked to a glossary entry. | Must | P1 | Test: every reason code has message text. |
| RK-042 | Raising risk per trade above 2%, loosening any loss or drawdown limit, disabling stops, or enabling a strategy with < 100 backtest trades shall show a consequence preview and require typed confirmation. | Must | P1 | |
| RK-043 | Ubik shall show a risk-utilization panel (each limit's percentage used) and the current trading state on the main dashboard. | Should | P1 | Shares the PF-011 dashboard. |
| RK-044 | Ubik shall show a timeline of recent risk decisions (approved, reduced, denied) with reasons, filterable by strategy. | Should | P1 | Builds trust and helps debugging. |

---

## 11. Jurisdiction flags (details in 08)

- **Exchange availability.** Binance.com does not serve US residents, and Binance.US is a separate venue with its own feature set. EEA users trade on MiCA-licensed venues, several of which restricted USDT trading for EEA users in 2025. The owner's country therefore determines the venues, the quote stablecoin (USDT vs USDC vs EUR) and the available order types. The risk engine must be quote-agnostic.
- **Tax lot method** (FIFO, specific ID, average cost) varies by country. The ledger keeps lots so any method can be computed.
- **Exchange ToS** on automated/API trading and on sub-accounts vary by country and account tier.

## 12. Dependencies on other docs

- **04:** strategy API (intents), backtest fill model, robustness checks used for graduation, metrics, multi-strategy backtests.
- **03:** ATR, volatility, correlation and regime computations used by sizing and clusters.
- **06:** venue adapters, user-data streams, persistence, process layout (kill switch and watchdog), time sync, schema.
- **07:** risk message UX, kill-switch placement, Safe vs Pro mode, dashboards.
- **08:** API-key scope and storage, authentication for kill switch and alerts, tax exports, disclaimers.
- **02:** reuse decisions (NautilusTrader/Freqtrade/Hummingbot/CCXT).

## Open questions for the owner

1. **Pain threshold:** once real money is involved, what loss in a bad *month* would make you stop? This sets the default daily, weekly and drawdown limits.
2. **Capital and venues:** roughly how much do you expect to run live eventually, and are you willing to split it across two exchanges to limit counterparty risk?
3. **Country and reference currency:** where do you live, and do you think in USD, EUR or another currency? This affects venues, the quote stablecoin and tax lots.
4. **Dedicated account:** will you also trade manually on the exchange account Ubik uses? A dedicated account or sub-account is strongly recommended.
5. **When you're asleep:** if the account hits its drawdown breaker at 3 a.m., should Ubik only pause and cancel (L1/L2), or also flatten positions automatically (L3)?
6. **Alerts:** which two channels reliably reach you (phone push, Telegram, SMS, email)?
7. **Native stops:** are you comfortable with exchange-side catastrophe stops that lock part of your balance and are visible to the exchange?
8. **Holding period:** intraday, swing (days) or longer? This changes how much fees, stale-data thresholds and execution algorithms matter.

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Paper results are too rosy (fill model ignores queue position and impact) | Disappointment, or losses at P3 | Conservative fill model (04), TCA tagging, demo and shadow stages, pilot caps |
| Over-restrictive defaults frustrate the owner, who then disables guardrails | Guardrails bypassed | Explain every block; tiered profiles; hard ceilings stay regardless |
| False halts from noisy stale-data or depeg detectors | Missed trades, alert fatigue | Per-pair thresholds, hysteresis, multi-source peg reference |
| Building P3-grade OMS/reconciliation in P1 slows the MVP | Schedule | Keep algorithms, allocation methods and VaR out of P1; invest only in the invariants (§9.5) |
| Exchange API changes break adapters | Halts, unknown states | Adapter contract tests against demo venues; changelog watch; capability flags at runtime |
| Correlations go to 1 in crashes | Diversification illusion | Cluster caps, stress replays, ES rather than VaR |
| Bot-managed stops fail while Ubik is offline; native stops slip or fail to fill in flash crashes | Large single loss | Hybrid stops, sizing that assumes a gap beyond the stop, dead-man's switch |
| Exchange insolvency or hack freezes funds | Total loss of on-exchange capital | Per-exchange caps; only trading capital on exchange; no software fix beyond that |
| Single self-hosted machine fails with positions open | Unmanaged positions | Native catastrophe stops, external heartbeat, documented restart; cloud option (06) |
| Accounting or precision bugs | Wrong P&L, reconciliation drift | Decimal-only, double-entry, property tests, continuous reconciliation |
| Single-user override: the owner can always change config | Self-inflicted risk | Cool-offs, typed confirmations, consequence previews; decisions are logged |
