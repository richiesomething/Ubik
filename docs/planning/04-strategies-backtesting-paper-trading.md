# Strategies, Backtesting & Paper Trading

*Planning doc 04 for Ubik. Owner-readable; jargon is explained on first use. Research checked 2026-10-18.*

## Summary

- **Every strategy runs on one interface in all three modes.** A strategy asks for *target positions* (or, for grids, order intents) and sees the world only through an injected **clock**, **data feed** and **venue** (the exchange it trades on). It never calls an exchange itself. Backtest, paper and live differ only in which clock, feed and venue the run's configuration plugs in. The **matching engine**, the code that decides whether and at what price a simulated order fills, is literally the same code in the backtester and in paper trading.
- **The event-driven backtester is the "official" result.** It replays history one event at a time, as live trading would. A fast *vectorized* screener (which computes whole price series at once) is added in P2 for big parameter sweeps, but every number shown as "the backtest" comes from the event-driven engine. Fills are conservative by default: never at the close of the bar that produced the signal, limit orders fill only when price trades *through* the limit, and stops are gap-aware. Fees come from versioned per-exchange schedules, exchange order rules are enforced, and money is computed with exact decimal arithmetic.
- **Honesty is a product feature.** Ubik keeps a **trial ledger** (every strategy variant ever tried) and a locked **out-of-sample "vault"** (recent data never used for tuning). In P2 it adds overfitting statistics: the Deflated Sharpe Ratio and the Probability of Backtest Overfitting. Every report compares the strategy with buy-and-hold BTC and with cash, and plain-language warnings sit *above* the headline numbers.
- **P1 offers seven explainable templates** (DCA, grid, MA crossover, Donchian breakout, RSI and Bollinger mean reversion, rebalancing) and a **no-code rule builder**, both compiling to the same engine. P1 runs no user-written code (decision D1). From P2, Python strategies run only in the SEC-035 sandbox and get the same automatic look-ahead and warm-up checks. P1 is spot-only, so strategies are long or flat. True long/short pairs trading and funding carry wait for derivatives in P4.
- **Paper trading is the backtester's matching engine fed by live market data**, not an exchange testnet. Ubik records the live data it trades on and re-runs a daily **parity replay** over it to measure divergence. A strategy **leaves paper** (stage S1 of doc 05's staged path to live) only after a published checklist: same version and venue throughout, ≥ 30 days and ≥ 20 round trips (one entry plus its matching exit), ≥ 95 % signal match, cost model confirmed, results inside the backtest's expectation band, and more. The checklist proves faithful *implementation*, not a real *edge*, and the UI says so.

Related docs: indicators in [03](03-market-and-technical-analysis.md); risk, sizing, the order management system (OMS) and execution in [05](05-risk-portfolio-execution.md); data and architecture in [06](06-data-and-architecture.md); screens and the Guided and Pro modes in [07](07-ux-and-beginner-experience.md); sandboxing, AI safety and disclaimers in [08](08-security-legal-compliance.md).

---

## 1. Strategy model

### 1.1 Lifecycle

1. **Initialize** (`on_start`): declare subscriptions (symbols × timeframes), compute warm-up needs and restore persisted state.
2. **React to events.** The default trigger is a **closed bar**. A *bar* (or *candle*) is the open/high/low/close/volume (OHLCV) summary of one time interval, and it is "closed" once that interval has ended. Timers handle scheduled actions such as DCA and rebalancing. Trade and order-book events arrive in P2 for faster strategies.
3. **Produce intents.** The default is a **target position** ("hold 40 % of the account in BTC"). Strategies that need explicit orders, such as a grid, emit **order intents** ("limit buy 0.01 BTC at 61,200").
4. **Risk and order layer** (doc 05): turns intents into orders, applying limits, sizing, exchange rounding and kill switches. It can shrink or veto any intent and cannot be bypassed.
5. **Feedback:** order updates and fills flow back to the strategy.

```python
# Pseudocode — illustrative only
class TrendBTC(Strategy):
    params = ParamSchema(
        fast=Int(default=20, min=5, max=100, beginner_range=(10, 50)),
        slow=Int(default=50, min=20, max=300, step=5),
        stop_pct=Decimal(default="0.05", min="0.01", max="0.25"))
    def on_start(self, ctx): ctx.subscribe("BTC/USDT", ["4h", "1d"])
    def on_bar(self, ctx, bar) -> list[Intent]:
        h = ctx.data.history("BTC/USDT", "4h", n=self.p.slow)  # only CLOSED bars ≤ ctx.now
        if crossover(sma(h, self.p.fast), sma(h, self.p.slow)):
            # the risk layer sizes this down to the risk profile (doc 05; Careful default: 1% risk per trade)
            return [TargetWeight("BTC/USDT", 1.0, stop_pct=self.p.stop_pct)]
        if crossunder(...): return [TargetWeight("BTC/USDT", 0.0)]
        return []
```

### 1.2 Sim-live parity: what gets abstracted

| Port (swappable component) | Backtest | Paper (P1) | Live (P3) |
|---|---|---|---|
| **Clock** (`Clock`) | Simulated; jumps to the next event | Wall clock (exchange timestamps) | Wall clock |
| **Data feed** (`MarketDataFeed`) | Replay of a pinned historical snapshot | Live exchange websockets + REST backfill | Same as paper |
| **Venue** (`ExecutionVenue`) | `SimVenue` (matching + fee/latency/slippage models) | **Same `SimVenue`**, fed by live data | `LiveVenue`, which forwards risk-approved intents to the execution gateway (docs 05/06) |
| **Ledger** | Decimal ledger | Persisted decimal ledger | Exchange balances + internal ledger + reconciliation (doc 05) |
| **Indicators** | Shared library (doc 03) | Shared | Shared |

Parity is *architectural*: the engine kernel, event ordering and accounting are shared, and only the ports change. [NautilusTrader](https://nautilustrader.io/blog/why-nautilustrader-exists/) is built the same way ("parity is built into the architecture rather than relying on convention"). It is open source ([LGPL-3.0](https://github.com/nautechsystems/nautilus_trader)) with a Rust core and Python strategies, but its v2 rewrite was still a release candidate in October 2026. Ubik therefore builds its **own thin event-driven core** (decision D2, doc 02). A ≤ 2-week trial build (a "spike") with NautilusTrader v2 at the start of P1 records design ideas to borrow and a revisit trigger (doc 06 AR-028); it does not choose the engine. Other frameworks serve as test oracles. The requirements below are written for Ubik's own core. **Code parity does not guarantee result parity**, which is why §8.4 adds divergence monitoring.

### 1.3 State, parameters, timeframes, assets, versioning

- **State** must be serializable and is saved after every event. Strategies may not use global variables, the wall clock, file or network access, or unseeded randomness; a seeded `ctx.rng` is provided.
- **Parameters** are declared in a typed schema (int, decimal, enum, bool, timeframe, symbol) with default, min/max/step, unit, description and an optional *beginner-safe range*. The same schema drives the builder sliders, the validator and the optimizer's search space.
- **Multiple timeframes:** a 4h strategy can read daily bars, but only bars that have closed by `ctx.now`. The 18 Oct daily bar becomes visible at 00:00 UTC on 19 Oct. This removes the most common way look-ahead sneaks in.
- **Multiple assets:** subscriptions are declared up front. Events with the same timestamp are delivered in a fixed instrument order, so runs are deterministic.
- **Versioning:** a strategy version is the content hash of {code or rule JSON, parameters, universe, timeframes, engine API version}. Versions are immutable and every run references one. The owner sees friendly names such as "Trend BTC v3", and can browse, branch from and compare versions (doc 07 UX-055). The engine API version changes only when the strategy API breaks, so routine Ubik upgrades don't reset graduation (doc 06 AR-035).

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| ST-001 | Ubik shall define one strategy interface with hooks `on_start`, `on_bar` (closed bar), `on_timer`, `on_order_update`/`on_fill`, `on_stop`, and `on_trade`/`on_book` for tick-level strategies. | Must | P1 (tick hooks P2) | A single contract makes parity possible. |
| ST-002 | Strategies shall express decisions only as intents (target position/weight or order intent). They shall have no access to exchange APIs, and every intent shall pass through the risk & order layer (doc 05). | Must | P1 | Risk controls can't be bypassed; going live doesn't touch strategy code. |
| ST-003 | Strategies shall reach time, data, account and execution only through an injected context. The same strategy version shall run unmodified in backtest, paper and (P3) live, with the mode chosen solely by run configuration. | Must | P1 | Sim-live parity. Test: one version hash produces runs in every mode. |
| ST-004 | The data API shall return only records whose availability time (`available_at`, doc 03) is ≤ `ctx.now`. Bars become available at their close, and higher-timeframe bars only after they close. | Must | P1 | Prevents look-ahead by construction. |
| ST-005 | Parameters shall be declared in a typed schema (default, min/max/step, unit, description, optional beginner-safe range), and out-of-range values shall be rejected. | Must | P1 | Feeds the builder, validator and optimizer. |
| ST-006 | Strategy state shall be persisted after each event and restored on restart. A strategy stopped at a random event and resumed shall make the same later decisions as an uninterrupted run. | Must | P1 | Automated restart test. |
| ST-007 | Strategies shall be able to subscribe to multiple symbols and timeframes, with deterministic ordering of same-timestamp events. | Must | P1 | Multi-timeframe filters; P2 rotation. |
| ST-008 | Strategy versions shall be immutable and content-hashed, and every run shall reference exactly one version. | Must | P1 | Reproducibility; any change resets graduation (§8.5). |
| ST-009 | Strategy code shall be deterministic: no wall-clock reads, unseeded randomness, or network/file I/O. | Must | P1 | Same inputs give the same outputs. |
| ST-010 | While only spot trading is enabled, Ubik shall reject intents that would create a short or leveraged position. | Must | P1 | P1–P3 are spot. *Long/flat* means the strategy either holds the coin or holds cash. |

---

## 2. Strategy types to support

The *regime* is the broad state of the market: trending, ranging (moving sideways in a band) or turbulent. Most strategies win in one regime and lose in another, so every template card shows both columns.

| Type | Plain-language description | Tends to work when… | Tends to fail when… | Phase |
|---|---|---|---|---|
| **DCA** (dollar-cost averaging) | Buy a fixed amount on a schedule regardless of price. | Long-term uptrends; it also removes the stress of timing. | Long bear markets. It is a *discipline*, not an edge, and Ubik must not claim it "beats" a lump-sum purchase. | P1 |
| **Grid** | Ladders of buy orders below the price and sell orders above it, profiting from back-and-forth movement. | Choppy ranges with volatility that is large relative to fees. | Trends. A downtrend leaves a growing losing pile of coins (*inventory risk*); an uptrend sells everything early. Needs a range exit. | P1 |
| **MA crossover** (trend) | Buy when a fast moving average crosses above a slow one; exit on the reverse. | Long, persistent trends, which crypto has had historically. | Sideways markets: *whipsaws* (many small losses) and heavy fees. | P1 |
| **Breakout / Donchian ("Turtle")** | Buy at an N-day high; exit at an M-day low. | Big directional moves. | False breakouts. Win rate is typically only 30–40 %, and beginners must expect that. | P1 |
| **Mean reversion: RSI / Bollinger** | Buy when price is stretched down (RSI < 30, or below the lower Bollinger band) and sell on the bounce. | Ranges and over-reactions in liquid pairs. | Crashes, where "oversold" stays oversold. Without a stop, a few big losses erase many small wins. | P1 |
| **Rebalancing** | Hold fixed weights (e.g. 50 % BTC / 50 % USDT) and trade back to them on a schedule or band breach. | Volatile but mean-reverting relative prices; it also controls risk. | One asset trends relentlessly, and the strategy keeps selling the winner. | P1 |
| **Momentum / rotation** | Hold the top-k coins by recent return or strength vs BTC; rotate periodically. | Dispersion, with leaders continuing to lead. | Sharp reversals. Severe survivorship bias unless the universe is point-in-time (§5). | P2 |
| **Volatility breakout** | Enter when price moves more than k × recent volatility (e.g. ATR, the average true range) from a reference level. | Quiet periods ending in expansions. | News-spike fakeouts. | P2 |
| **Pairs / statistical arbitrage** | Trade the spread between two related coins when it is unusually wide, expecting convergence. | Stable (*cointegrated*) relationships. | The relationship breaks, which is common in crypto. **On spot only an "ETH↔BTC switching" variant is possible (P2); true long/short pairs need margin or perps (P4).** | P2 / P4 |
| **Funding-rate carry** | Hold spot and short the matching perpetual to collect *funding* (periodic payments between longs and shorts). | Persistently positive funding. | Funding flips, the venue fails, or the short is liquidated in a spike (e.g. 10 Oct 2025). | P4 |
| **Market making** | Continuously quote buy and sell prices and earn the spread. | Low or rebated fees, fast infrastructure, tight inventory control. | *Adverse selection* (better-informed traders pick you off) and trends. Ubik should say plainly that individuals rarely beat professional market makers. | P4 |
| **ML-driven** | A machine-learning model predicts returns, and rules turn its predictions into positions. | A robust signal plus disciplined validation (BT-032). | The easiest type to overfit; regime changes silently break models. | P4 |

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| ST-011 | Ubik shall ship templates for DCA, grid, MA crossover, Donchian breakout, RSI mean reversion, Bollinger reversion and rebalancing. Each shall have defaults, a plain-language description, a "works when / fails when" card and its known risks. | Must | P1 | Beginners start from explainable patterns. |
| ST-012 | Ubik shall add momentum/rotation, volatility-breakout and spot pair-switching templates. | Should | P2 | Needs multi-asset, point-in-time data. |
| ST-013 | Funding carry, market making, true long/short pairs and ML strategies shall be enabled only once derivatives, L2 data and BT-032 validation exist. | Could | P4 | Prevents premature high-risk features. |
| ST-014 | The grid template shall require a price range, a range-exit rule and an inventory cap, and shall show the worst-case inventory if price falls through the whole grid. | Must | P1 | Grid bots are popular, and their main risk is hidden. A ladder larger than doc 05's runaway guard (≤ 10 orders/min) is placed in batches. |

---

## 3. Strategy authoring

### 3.1 Three tiers, one engine

1. **Templates:** pick one, adjust a few sliders within beginner-safe ranges, and backtest.
2. **No-code rule builder:** "WHEN [conditions] THEN [action]".
   - Conditions: indicator comparisons (RSI(14) < 30), crossovers (EMA 20 crosses above EMA 50), price vs a level or band, time or schedule ("every Monday 09:00 UTC"), position state ("held > 10 bars") and portfolio state ("drawdown > 10 %"), combined with ALL/ANY.
   - Actions: enter or exit to a target weight, add/reduce, set a stop-loss, take-profit or trailing stop, place a limit order at an offset.
   - Every entry rule needs an exit path. Rules are stored as versioned **rule JSON**, a small domain-specific language (DSL), and run by a `RuleStrategy` interpreter through the same interface as code. They are diffable and, from P2, can be *ejected* to Python.
3. **(P2) Python** for advanced users: an SDK with typed helpers, a local test harness and docs. Code runs only in the SEC-035 sandbox (doc 08).

### 3.2 LLM assistant: drafting and explaining safely

The assistant can translate an idea into rule JSON (or, from P2, a code diff in Pro mode), explain any strategy or result in plain words, and suggest robustness checks. Guardrails:

- Drafts pass the same validator, are shown as a diff and need explicit confirmation. The assistant can never start a paper or live bot.
- **AI-drafted variants are counted in the trial ledger** (§5). Fifty suggested variants are 50 trials, so the assistant must not become a data-snooping machine.
- It never predicts returns. Any statement about performance quotes the stored run together with its warnings.
- UX details are in doc 07, safety in doc 08.

### 3.3 Validation before running

| Check | Kind | Result |
|---|---|---|
| Look-ahead in code (e.g. `shift(-n)`, indexing past `now`, filling at the signal bar's own close) | Static | Error |
| **Truncation test:** recompute each signal using only data up to its timestamp; any change means look-ahead (as in Freqtrade's [lookahead-analysis](https://www.freqtrade.io/en/stable/lookahead-analysis/)) | Dynamic | Error, listing the affected signals |
| **Warm-up test:** compare indicator values (e.g. EMA) across different history lengths and auto-set the warm-up period (cf. [recursive-analysis](https://www.freqtrade.io/en/stable/recursive-analysis/)) | Dynamic | Warning + auto-fix |
| No stop or exit rule | Static | Error under the Careful and Balanced risk profiles; warning under Adventurous (doc 05 RK-016). DCA and rebalancing templates are exempt: as allocation-style strategies they rely on the drawdown breaker and a worst-case display instead (doc 05 §2.1) |
| Short or leverage on spot; order type unsupported by the venue | Static | Error |
| Order size below the exchange's minimum notional (minimum order value) for this account size | Static | Error |
| Non-determinism (clock, unseeded random, I/O) | Static | Error |
| Round-trip fees × round trips per year > 50 % of the gross edge | Post-run | Warning: "fees eat this strategy" |
| Data range missing or gappy | Static | Warning or error (BT-017) |

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| ST-015 | Ubik shall provide a no-code rule builder (conditions, ALL/ANY logic and actions as in §3.1) whose output is versioned rule JSON executed through the standard strategy interface. | Must | P1 | Beginner-friendly, same engine. |
| ST-016 | Every rule-builder entry rule shall have an exit path (exit rule, stop or time exit). | Must | P1 | No accidental "buy and pray". |
| ST-017 | Ubik shall provide a Python strategy SDK. User-written and LLM-generated strategy code shall run only inside the SEC-035 sandbox (doc 08; the `sandbox` process role, doc 06 AR-037), with market data in and intents out. | Must | P2 | Per D1; SEC-034 allows no user-supplied code in P1. |
| ST-018 | Ubik shall run the §3.3 checks before every backtest and every paper or live start, blocking on errors and showing warnings. | Must | P1 | Catches common mistakes. |
| ST-019 | Ubik shall run the truncation look-ahead test on every strategy version (templates and rule strategies in P1; sandboxed code strategies from P2) and report any signal that changes. | Must | P1 | Rule strategies are safe by construction but tested too. Blocking for entry to S1 (§8.5, CAP-009). |
| ST-020 | Ubik shall eject a rule strategy to Python and prove equivalence (identical trades on the same data). | Could | P2 | Path from beginner to pro. |
| ST-021 | The AI assistant shall draft strategies only as validated rule JSON or code diffs requiring user confirmation. It shall not start runs, and its variants shall be recorded in the trial ledger. | Must | P2 | Safety and multiple-testing honesty. |
| ST-022 | The AI assistant shall not state or imply expected future returns. Performance statements shall cite stored runs together with their warnings. | Must | P2 | Honesty principle; doc 08. |
| ST-023 | Ubik shall export a strategy version as a portable file (rule JSON or code, parameter schema and values, sizing policy, risk profile, hypothesis, kill criteria and content hash). It shall import such a file as a new strategy at the Draft stage, with "imported" provenance and a fresh trial ledger, flagged untrusted per SEC-036; imported code shall run only under SEC-035. | Should | P2 | Personal backup and moving strategies between installs. No in-app sharing or marketplace (CMP-001). |

---

## 4. Backtesting engine

### 4.1 Approach

The **event-driven** engine is the source of truth for every reported result, for paper trading and for graduation. P1 needs nothing else, because its optimization grids are small. In P2 a **vectorized screener** handles big sweeps: it is approximate but runs thousands of combinations per minute, and its top-N candidates are re-run event-driven. Ubik builds the screener itself on NumPy/Numba-style arrays. vectorbt is excluded as a dependency because its [Commons Clause](https://vectorbt.dev/terms) forbids selling a product built on it (doc 02), but it remains fine for private research and as a test oracle. A consistency test runs reference strategies in both engines and flags disagreement.

### 4.2 Data granularity

| Data | Use | Phase |
|---|---|---|
| **1-minute OHLCV** | Default *simulation* granularity. A 4h or 1d strategy still gets fills simulated on 1m sub-bars | P1 |
| 1-second bars / trade prints | Finer fills, queue modeling, short-timeframe strategies | P2 |
| L2 order book (several price levels of bids and asks) | Book-walk slippage, market making. Historical L2 is mostly paid, so **record it from the live feed starting in P1** (up to 5 instruments in P1; retention per doc 06 NFR-011) | P2+ (recording P1) |
| Exchange rules and fee schedules over time | Rounding, rejections, fees | P1 |
| Funding rates, mark prices | Perps | P4 |

Data sourcing is covered in doc 06.

### 4.3 Fill modeling

**Timing.** A decision made on bar *t*'s close reaches the venue at *t* + latency and fills at the **next available price**. Filling at bar *t*'s own close is a classic optimistic bias, so it is off by default and turning it on adds a warning. NautilusTrader's [bar-execution docs](https://nautilustrader.io/docs/nightly/concepts/backtesting/bar-execution/) apply the same rule and note that reading the current bar's open in `on_bar` is look-ahead.

**Intrabar path.** OHLC bars don't record whether the high or the low came first. When a stop and a target sit inside the same bar, Ubik resolves the order with 1m (or finer) sub-bars. If finer data isn't available, it assumes the **adverse level was hit first**.

| Order | Ubik default model | Configurable |
|---|---|---|
| **Market** | Reference price (next trade or open) ± half the bid-ask spread ± slippage; *taker* fee (the higher rate for removing liquidity) | Slippage model; L2 book walk (P2) |
| **Limit** | Fills only if price trades **through** the limit by ≥ 1 tick; a mere touch fills with probability *p* (default 0, since a resting order waits in a queue); *maker* fee (the lower rate for adding liquidity) | *p*; queue model (P2) |
| **Partial fills** | Fill per bar capped at a **participation rate** of that bar's volume (default 10 %); the rest keeps resting | Rate |
| **Queue position** (P2) | Back-of-queue: fills only after the volume ahead at that price has traded | On/off |
| **Stop** | Triggers when price crosses the stop and fills at the **worse of the stop price and the first post-trigger price**, plus slippage. (Freqtrade fills stops exactly at the stop price "even if low was lower" ([docs](https://www.freqtrade.io/en/stable/backtesting/)), which is optimistic.) | Slippage |
| **Rejects / expiries** | Below minimum notional, wrong tick or step, insufficient balance, post-only order that would cross: rejected exactly as the exchange would. Orders that would execute outside a venue's price band expire, as under Binance's [Price Range Execution Rule](https://developers.binance.com/en/docs/products/spot/CHANGELOG) (introduced March 2026) | none |

**Slippage** (expected price vs actual fill) has four models:

1. **Fixed basis points** (1 bp = 0.01 %).
2. **Spread-based:** half the recorded or estimated spread, plus a floor.
3. **Square-root impact:** extra cost ≈ *k*·σ·√(order size ÷ volume), the empirical law that penalizes large orders.
4. **L2 book walk** (P2): consume the recorded book levels.

The P1 default is max(spread-based, a conservative floor) together with the participation cap. **Latency** is fixed plus seeded random jitter, a few hundred milliseconds by default; doc 05/06 will measure the real value.

### 4.4 Fees, exchange rules, funding, accounting

- **Fees are data with effective dates**: maker/taker rate by tier, fee asset and discounts (e.g. Binance's BNB discount). The schedules themselves live in doc 06's venue registry (AR-002), the single source that the cost model reads; this doc doesn't repeat them. Schedules change (Kraken [raised its entry-tier fees in July 2026](https://pricingsaas.com/companies/kraken/diffs/2026W29)) and are expensive at retail tiers, where Coinbase Advanced and Kraken charge several times Binance's rate. Until the owner's account confirms its tier, the cost model uses the registry's higher, conservative pair.
  - Even at 0.6 % per side, 100 round trips a year cost about 120 % of the strategy's capital (if each round trip uses the full capital). Ubik must make this visible before the owner runs anything.
- **Exchange order rules.** Binance publishes these as [symbol filters](https://developers.binance.com/docs/binance-spot-api-docs/filters): tick size (PRICE_FILTER), quantity step (LOT_SIZE, MARKET_LOT_SIZE), minimum and maximum order value (NOTIONAL/MIN_NOTIONAL), price bands (PERCENT_PRICE) and order count (MAX_NUM_ORDERS). Ubik rounds quantity *down* to the step and price to the tick, then rejects as the exchange would. Historical rules usually aren't published, and Freqtrade [admits](https://www.freqtrade.io/en/stable/backtesting/) it applies today's rules to the past, so **Ubik snapshots rules daily from P1 onwards**.
- **Funding (P4):** perpetual funding is applied at each venue's actual timestamps and rates.
- **Accounting:** exact **decimal** arithmetic for every price, quantity, balance and fee; floating point only inside indicators. Each asset has a *free* balance and a *locked* balance (reserved for open orders). The ledger is double-entry, with the fee as its own entry. Invariants are checked after every event (no negative balances; value conserved apart from fees), and any violation aborts the run.

### 4.5 Reproducibility

Each run writes a **manifest**: data snapshot ID and hash, strategy version hash, parameters, engine version, fee/slippage/latency/rule model versions, seed and full config. Re-running the manifest must give **bit-identical** trades and metrics. **Golden tests** (reference strategies with stored expected results) run in CI and guard every engine change. They are complemented by *differential tests*: the same reference strategies run privately through Freqtrade, NautilusTrader and vectorbt, and Ubik explains every difference in trades (doc 02).

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| BT-001 | Ubik shall provide an event-driven backtester whose matching engine (`SimVenue`), fee/slippage/latency models and ledger are the same code used by the paper venue. | Must | P1 | Makes parity real. |
| BT-002 | Ubik shall provide a vectorized screener for parameter sweeps, labeled "approximate". Candidates shall be re-run event-driven before being reported or promoted to paper. | Should | P2 | Speed without trading away honesty. |
| BT-003 | The backtester shall simulate fills on 1-minute bars by default, regardless of the strategy's timeframe, and shall support trade-level data. | Must | P1 (trades P2) | Intrabar realism. |
| BT-004 | Orders generated on a bar close shall fill no earlier than the next available price after latency. Same-bar-close fills shall be off by default and shall add a report warning when enabled. | Must | P1 | Prevents look-ahead and optimistic fills. |
| BT-005 | When several orders could trigger in one bar, the engine shall resolve the order with finer data if available, else assume the adverse level was hit first. | Must | P1 | Conservative default. |
| BT-006 | Market orders shall fill at reference price ± half-spread ± modeled slippage, with taker fees. | Must | P1 | |
| BT-007 | Limit orders shall fill only when price trades through the limit (touch-fill probability configurable, default 0), with fill size capped at a configurable participation rate of traded volume (default 10 %). | Must | P1 | Conservative queue and liquidity model. |
| BT-008 | Stop orders shall fill at the worse of the stop price and the first post-trigger price, plus slippage. | Must | P1 | Crypto gaps and wicks are large. |
| BT-009 | Ubik shall offer fixed-bps and spread-based slippage models in P1, and square-root impact and L2 book-walk models in P2. | Must | P1/P2 | In P2, defaults are calibrated from measured paper/live costs (doc 05 EX-020, 75th percentile). |
| BT-010 | With trade data available, Ubik shall model limit-order queue position (back-of-queue). | Should | P2 | |
| BT-011 | Ubik shall model order latency as fixed delay plus seeded jitter. | Must | P1 | |
| BT-012 | Ubik shall apply fees per fill from versioned, effective-dated venue schedules (maker/taker, tier, fee asset, discounts) read from doc 06's venue registry (AR-002). Unverified entry tiers shall use the registry's higher, conservative rates. | Must | P1 | Fees change, as Kraken did in July 2026. |
| BT-013 | Ubik shall enforce venue order rules (tick, step, min/max quantity and notional, max open orders, price-band expiries) using the rule snapshot valid at the simulated time, falling back to the earliest snapshot with a warning. | Must | P1 | Daily snapshots from P1; the rule set is shared with doc 05's EX-010. |
| BT-014 | Ubik shall apply perpetual funding payments at venue funding timestamps. | Must | P4 | |
| BT-015 | Ubik shall use exact decimal arithmetic for all money, keep a double-entry ledger with free/locked balances, and check invariants after every event, aborting on violation. | Must | P1 | Precondition for P3 reconciliation. |
| BT-016 | Every run shall store a reproducibility manifest. Re-running it shall produce bit-identical trades and metrics, enforced by golden tests in CI. | Must | P1 | After upgrades, old manifests stay re-runnable with the engine version they name (doc 06 AR-035). |
| BT-017 | Before a run, Ubik shall check the data for gaps, duplicates, out-of-order timestamps, zero-volume bars and outliers, and apply a configurable policy (fail / skip / fill with warning). | Must | P1 | Bad data creates fake profits. |
| BT-018 | A one-year, single-symbol event-driven backtest on 1m bars (≈ 525,600 bars) shall finish in ≤ 60 s on doc 06's reference machine. | Should | P1 | Fast iteration; target to confirm. |
| BT-019 | Ubik shall backtest several strategies sharing one account, using doc 05's capital allocation. | Should | P2 | Several bots may share a paper account from P1 (doc 05 PF-001, PF-004); backtesting them together arrives here. |

---

## 5. Biases and safeguards

| Bias | Plain meaning | Ubik's prevention / detection |
|---|---|---|
| **Look-ahead** | Using information not yet available, e.g. trading at a close you used to decide. | Availability-time data API (ST-004), next-price fills (BT-004), static checks and the truncation test (ST-019). |
| **Survivorship / delisting** | Testing only coins that still exist, which silently drops the dead ones. One study of 3,904 coins (2014–2021) put the bias at [0.93 %/yr for value-weighted and 62 %/yr for equal-weighted portfolios](https://alexandria.unisg.ch/handle/20.500.14171/108037) (Ammann et al., 2022). | P1 trades only long-lived majors and shows a notice. Point-in-time universes including delisted coins (BT-020) must exist before rotation strategies ship. |
| **Data-snooping / multiple testing** | Try enough variants and one will look great by luck. After only **7** tries, expect a 2-year backtest with annual Sharpe > 1 even if the true Sharpe is zero. With 5 years of daily data, try no more than about **45** independent variants ([Bailey, Borwein, López de Prado & Zhu 2014](https://scholarworks.wmich.edu/math_pubs/40/); [slides](https://www.carmamaths.org/meetings/mabef/talks/mabef-David_Bailey-1.pdf)). Academic finance raised its bar for a "discovery" to t > 3 for the same reason ([Harvey, Liu & Zhu](https://www.nber.org/papers/w20592)). | **Trial ledger** (BT-021) feeds DSR (BT-028) and PBO (BT-029). Reports show "N variants tried". |
| **Selection bias** | Showing only the best period or coin. | The default window is the most recent 4 years of the development set, ending where the out-of-sample vault begins, or the whole development set if it is shorter (doc 07 §2.3). In 2026 it covers the 2022 bear phase; the vault covers the most recent period, including the 2025–26 bear phase, and its result is shown in a separate panel (BT-022). The full development set is one click away; shorter custom windows are labeled; all trials are visible. |
| **Unrealistic fills / ignored costs** | Always getting your price, at any size, for free. | §4.3 conservative fills, participation cap, costs always on and itemized, cost ×2 re-run (BT-033). |
| **Warm-up bias** | Indicators computed on too little history behave differently live. | Automatic warm-up detection (§3.3). |
| **Regime bias** | Tested only in a bull market. | Warning if the window lacks a ≥ 30 % BTC drawdown; stress library (§6.4). |
| **Counterparty blind spot** | A backtest assumes the exchange never fails, but FTX did. | Stress note; venue limits in doc 05. |

---

## 6. Validation and optimization

### 6.1 Default research protocol

```
Full history ─┬─ Development set (≈75 %) ── build, optimize, walk-forward
              └─ Vault (most recent ≈25 %, ≥ 6 months) ── opened deliberately, once per version; every opening logged
```

- **In-sample / out-of-sample.** *In-sample (IS)* data is used for building and tuning; *out-of-sample (OOS)* data is unseen. The vault is the final OOS check, and in P1 it is the only one. Its result is shown in a separate panel, and it can be opened once per strategy version; every opening is counted (BT-022). Opening it again after a change (which makes a new version) counts as a new trial.
- **Default backtest window:** the most recent 4 years of the development set, ending where the vault begins (or the whole development set if it is shorter).
- **Walk-forward analysis (P2).** Optimize on one window, test on the next unseen window, roll forward and repeat. Windows are *rolling* (fixed length) or *anchored* (expanding). The **stitched OOS equity curve** is the honest result. *Walk-forward efficiency* (OOS annualized return ÷ IS annualized return) well below ~0.5 hints at overfitting.
- **Optimization methods.** P1: grid search, capped at about 200 combinations. P2: random search and Bayesian search (e.g. [Optuna](https://pypi.org/project/optuna)'s TPE sampler, MIT-licensed) with trial budgets. The default objective is risk-adjusted (Sharpe or Calmar) with a minimum trade count, never raw return.
- **Robustness.** Parameter-sensitivity **heatmaps**: prefer *plateaus* (neighbouring settings also work) over sharp *peaks*. Also check start-date sensitivity and other liquid pairs (does it work on ETH and SOL, not just BTC?).

### 6.2 Overfitting statistics

| Statistic | Question it answers | Ubik use |
|---|---|---|
| **Probabilistic Sharpe Ratio (PSR)** and **Minimum Track Record Length (MinTRL)** ([Bailey & López de Prado 2012](https://www.risk.net/journal-risk/2223785/sharpe-ratio-efficient-frontier)) | "How sure are we the true Sharpe is above zero, given track length, skew and fat tails? How long must the record be?" | Every report (P2); graduation note (§8.5) |
| **Deflated Sharpe Ratio (DSR)** ([Bailey & López de Prado 2014](https://papers.ssrn.com/abstract=2460551)) | PSR corrected for the number of trials (from the trial ledger) and non-normal returns | Warn if DSR < 0.95 |
| **Probability of Backtest Overfitting (PBO)** via combinatorially symmetric cross-validation, CSCV ([Bailey et al. 2017](https://papers.ssrn.com/abstract=2326253)) | "How often does the in-sample winner land below the median out-of-sample?" | Optimizations with ≥ 10 configurations; strong warning if PBO > 0.5 |
| White's Reality Check / Hansen SPA | "Does the best variant beat the benchmark after accounting for all variants?" | Could, P2 |
| Combinatorial purged cross-validation, CPCV ([López de Prado 2018](https://en.wikipedia.org/wiki/Purged_cross-validation)) | Many leak-free OOS paths instead of one | Required for ML (P4) |

### 6.3 Monte Carlo and bootstrap (P2)

- **Trade reshuffle:** gives the distribution of maximum drawdown and losing streaks.
- **Trade bootstrap** (resample trades with replacement): gives confidence bands on return and expectancy.
- **Stationary block bootstrap** of daily returns (Politis & Romano 1994): keeps volatility clustering intact.
- Results are shown as 5th/50th/95th percentile bands plus a sentence such as "In 1 of 5 simulations you would have faced a drawdown worse than −38 %."

### 6.4 Stress-test library

Every strategy is run through each event window that falls inside its history. The windows are stored as data. Historical windows run from P1 (a Must for shipped templates, a Should for user strategies); the synthetic shocks below arrive in P2 (BT-027).

| Window | Event | Tests |
|---|---|---|
| 12–13 Mar 2020 | COVID "Black Thursday": BTC roughly −40–50 % in about a day; exchanges overloaded | Gaps, stops, outages |
| 19 May 2021 | China crackdown and leverage flush, ~$10 B liquidated (the record until 2025) ([MoneySense](https://www.moneysense.ca/?p=368849)) | Intraday crash |
| 7–13 May 2022 | Terra UST de-peg and LUNA collapse; UST ≈ $0.36 by 12 May ([source](https://www.spark.money/glossary/ust-collapse)) | Asset → ~0; stablecoin risk |
| 12–18 Jun 2022 | Celsius freezes withdrawals; 3AC contagion; BTC drops below $20k | Credit contagion |
| 6–11 Nov 2022 | FTX collapse; Chapter 11 filed 11 Nov | Exchange failure |
| 10–13 Mar 2023 | SVB failure; USDC de-pegs to ≈ $0.87–0.88 | Quote-stablecoin risk |
| 5 Aug 2024 | Yen carry-trade unwind; global risk-off | Correlation spike |
| 21 Feb 2025 | Bybit hack, about $1.5 B stolen; ETH −7 % within hours ([SCMP](https://www.scmp.com/tech/blockchain/article/3300087/bybits-us15-billion-loss-largest-crypto-hack-ever-stokes-bitcoin-ether-sell)) | Venue security event |
| 10–11 Oct 2025 | Largest liquidation event on record, ~$19 B reported (CoinGlass estimates more), after the 100 % China-tariff announcement; BTC fell from ~$122 k to roughly $102–105 k intraday, about 17–19 % below its ~$126 k high of 6 Oct; some altcoins briefly near zero on some venues ([summary](https://tr.tradingview.com/news/newsbtc%3Aa4b7d8789094b%3A0-crypto-crash-19-5-billion-wiped-out-in-record-breaking-liquidation-event)) | Wicks, venue/oracle failures |
| 5–6 Feb 2026 | BTC −13 % in a session, touching ~$60 k on 6 Feb, the lowest since Sep 2024 ([Forklog](https://forklog.com/en/news/bitcoin-tests-60000-support-triggering-billion%e2%80%91dollar-liquidations)) | Bear capitulation |
| Jun 2026 | Slide to ~$58–60 k on ETF outflows, > 50 % below the 2025 high ([Bitcoin.com](https://news.bitcoin.com/bitcoin-hits-59018-after-a-5-drop-forcing-237m-in-long-liquidations/)) | Long drawdown |
| Oct 2025 → 2026 | The whole bear phase | Strategies tuned only on the 2023–25 bull run |

Synthetic shocks: an instant −30 % gap; spread ×10 for an hour; a 6-hour venue outage with open orders; the quote stablecoin at $0.90.

### 6.5 Benchmarks

Every report shows **buy-and-hold BTC**, **buy-and-hold of the traded assets** (or an equal-weight basket), **cash** (0 % by default; an optional yield, flagged as not risk-free on exchanges) and, for DCA-like strategies, **plain DCA**. The verdict says whether the strategy beat buy-and-hold on a risk-adjusted basis, *after costs*. Benchmarks are reference lines: they are computed outside the risk profile (only the strategy goes through doc 05's limits) and labelled as references.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| BT-020 | Ubik shall build point-in-time universes, including later-delisted symbols, for any multi-asset selection strategy. | Must | P2 | Survivorship bias. |
| BT-021 | Ubik shall record every backtest and optimization trial of a strategy family, including AI-drafted variants, in a trial ledger and show the count in reports. | Must | P1 | Input for DSR and PBO; cheap to start early. This is the single trial ledger: it also satisfies CAP-005 and MA-023. Signal Lab and scanner evaluations are stored as signal entries and count toward the trials of strategies built from that signal family. Retired versions still count. |
| BT-022 | Ubik shall reserve an out-of-sample vault (default: the most recent 25 % of history, min 6 months), keep it out of the default backtest window, show its result in a separate panel, allow it to be opened once per strategy version, and log and display every opening. | Must | P1 | IS/OOS discipline from day one; the P1 out-of-sample check (G2). |
| BT-023 | Ubik shall offer grid optimization (≤ 200 combinations) in P1, and random and Bayesian optimization with trial budgets in P2. The default objective shall be risk-adjusted with a minimum trade count. | Must | P1/P2 | |
| BT-024 | Ubik shall provide rolling and anchored walk-forward analysis with a stitched OOS equity curve and walk-forward efficiency. | Must | P2 | |
| BT-025 | Ubik shall render parameter-sensitivity heatmaps for any two optimized parameters, with a plateau score (neighbourhood performance relative to the peak). | Should | P1 (Must P2) | Robust strategies sit on plateaus. |
| BT-026 | Ubik shall run trade-reshuffle and trade-bootstrap Monte Carlo and stationary block bootstrap, reporting percentile bands for return and drawdown. | Must | P2 | Matches CAP-011 and ED-005. |
| BT-027 | Ubik shall maintain the §6.4 stress library as editable data, report performance in each covered historical window automatically, and run the synthetic shocks. | Must | P1 (historical windows for shipped templates); P2 (synthetic shocks) | Historical windows are Should P1 for user strategies. Gives every template its failure-mode demonstration (CAP-038, CAP-010). |
| BT-028 | Ubik shall compute PSR and DSR (with trials taken from the ledger) and warn when DSR < 0.95. | Must | P2 | |
| BT-029 | Ubik shall compute PBO via CSCV for optimizations with ≥ 10 configurations and warn strongly when PBO > 0.5. | Must | P2 | Matches CAP-007. |
| BT-030 | Every run shall be compared with buy-and-hold BTC, buy-and-hold of the traded assets, and cash, with a plain-language verdict on whether the strategy beat them risk-adjusted after costs. | Must | P1 | |
| BT-031 | Ubik shall offer White's Reality Check or Hansen's SPA test for multi-strategy searches. | Could | P2 | |
| BT-032 | ML strategies shall pass combinatorial purged cross-validation (with purge and embargo) before paper trading. | Must | P4 | |
| BT-033 | Every report shall include an automatic re-run with fees and slippage ×2, showing net return and Sharpe beside the base case. | Should | P1 | Fragility check. |

---

## 7. Performance metrics and the backtest report

Conventions: crypto trades every day of the year, so annualization uses **365 days** (√365 for volatility). The risk-free rate defaults to 0. All figures are **net of fees and slippage** unless labeled "gross".

| Metric | Plain meaning | Good value / warning sign |
|---|---|---|
| **Total return / CAGR** | Overall gain; CAGR is the equivalent steady yearly growth rate | Meaningless without the benchmark and risk figures |
| **Volatility** (annualized) | How much returns swing | Compare with the asset; BTC has often run at 40–80 %/yr |
| **Sharpe** | Return per unit of total volatility | OOS after costs > 1 is good. **A backtested Sharpe > 3 on a daily-or-slower crypto strategy is a red flag for overfitting or a bug.** |
| **Sortino** | Like Sharpe, but penalizes only downside swings | Higher than Sharpe if the strategy cuts losses |
| **Calmar** | CAGR ÷ max drawdown | > 1 is good; < 0.3 means painful for the reward |
| **Max drawdown** | Largest peak-to-trough loss | "Could I live through this?" Beyond −25 % most beginners quit. |
| **Drawdown duration** | Time spent below the previous peak | > 1 year underwater is psychologically hard |
| **Win rate** | Share of profitable trades | Trend strategies win only ~30–45 %. **A high win rate with a small payoff ratio (grids, no stops) hides tail risk.** |
| **Payoff ratio** | Average win ÷ average loss | Read together with win rate |
| **Profit factor** | Gross profit ÷ gross loss | 1.2–2 is plausible. **> 3 on < 100 round trips is suspicious.** |
| **Expectancy** | Average profit per trade, in money and in *R* (multiples of the initial risk) | Must clearly exceed round-trip costs |
| **Number of trades** (round trips) | Sample size. One trade is one **round trip**: an entry plus its matching exit. Every trade count and gate in Ubik counts round trips | **< 30: anecdotes, not evidence** |
| **Exposure time** (time in market) | Share of time in the market | Compare with the benchmark's 100 % |
| **Turnover** | Capital traded per year | Multiplies the impact of fees |
| **Fees and slippage paid** | Costs in money and as % of gross profit | **> 50 % of gross profit = fragile** |
| **Average holding time** | How long trades last | Context for cost and latency sensitivity |
| **Top-5 trade share** | Share of profit from the five best trades | > 50 % means the result depends on luck |
| **Beta / correlation to BTC** | How much of the result is just "the market" | Beta ≈ 1 with lower return than BTC = no value added |
| **Skew, worst day** | Shape of the losses | Strongly negative skew = small steady gains, rare big losses |
| **PSR / DSR** (P2) | Confidence the edge is real | DSR < 0.95 triggers a warning |

**Report contents** (layout in doc 07):

1. **Verdict and trust check**, shown first; the honesty warnings below are its inputs (presentation in doc 07). Example: "Beat buy-and-hold BTC risk-adjusted: No · 23 round trips (too few) · no bear market in the test window."
2. Metrics vs benchmarks.
3. Equity curve vs benchmarks (log-scale toggle). The vault (out-of-sample) result sits in a separate panel; walk-forward OOS segments are shaded from P2.
4. Underwater (drawdown) chart.
5. Trades on the price chart, with entry and exit reasons in words.
6. Monthly returns heatmap; distribution of trade returns.
7. Rolling 90-day return and Sharpe.
8. Gross vs net cost breakdown, plus every order the risk engine reduced or denied, with the reason (doc 05 RK-006).
9. Stress-test table (historical windows from P1); Monte Carlo bands (P2).
10. Parameter heatmap when optimized (P1); PBO and DSR (P2).
11. **Assumptions** (fees, slippage, latency, fill rules) and the **reproducibility manifest**.

**Automatic honesty warnings** fire for:

- < 30 round trips;
- < 2 years of history, or no ≥ 30 % BTC drawdown in the window;
- DSR < 0.95 or PBO > 0.5 (P2);
- top-5 trades > 50 % of profit;
- costs > 50 % of gross profit;
- OOS Sharpe < half of IS Sharpe;
- optimal parameters at the edge of their range;
- participation cap hit frequently;
- same-bar fills enabled;
- universe not point-in-time;
- data-quality issues.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| BT-034 | Ubik shall compute the §7 metrics for every run using the stated conventions (365-day annualization, net of costs, risk-free 0 by default), with a plain-language tooltip and good/warning guidance for each. | Must | P1 | Glossary in doc 07. |
| BT-035 | Every backtest report shall contain the §7 items available in its phase, with the verdict and trust check placed above all performance numbers. | Must | P1 | Honesty principle. Items tagged P2 in §7 appear from P2. |
| BT-036 | Ubik shall evaluate the honesty-warning rules on every run and show triggered warnings in reports, comparisons and exports. | Must | P1 | |
| BT-037 | Strategy comparisons shall use the same period, costs and benchmarks, and shall label any comparison of runs with mismatched assumptions. | Should | P1 | No apples-to-oranges cherry-picking. The side-by-side comparison view is doc 07 UX-056. |

---

## 8. Paper trading

### 8.1 Design: Ubik's own simulator on live data

The paper venue is the backtester's `SimVenue`, fed by **live public market data** (trades, best bid/ask and L2 depth where available) from the exchange the owner intends to trade on live. It needs **no API keys**.

Exchange testnets are not suitable for judging performance. Binance's [docs](https://developers.binance.com/docs/binance-spot-api-docs/demo-mode/general-info) say the Spot Testnet's prices and order books are independent of the live exchange. Its newer Demo Mode tracks live prices more closely, but Binance itself warns that "realistic market data is not equal to real market data". Demo Mode also needs keys, and its fill logic differs from the backtester's. An **exchange demo** (a venue's demo environment that mirrors live, such as Binance Spot Demo Mode) is therefore used only from P3, as stage S2, to test the **live adapter's order lifecycle** (doc 05 EX-023), never to judge strategy performance (PT-016). Testnets are not used for graduation. Where a venue has no realistic spot demo (Coinbase's sandbox is mocked; Kraken has none), S2 is met instead by doc 06's adapter contract suite plus fault-injection tests on the paper venue (doc 05 §9.1, EX-021).

Paper fills have no market impact, which flatters large orders. That is why the participation cap and slippage models from §4.3 apply unchanged.

### 8.2 Features

- **Multiple paper accounts**, each with its own base currency, starting capital, venue, fee tier and bots (for side-by-side A/B tests). A **bot** is one strategy version deployed on one account. Each bot owns exactly one capital **sleeve** (doc 05 §6), so several bots can share an account, with manual or equal-weight allocation in P1.
- **Reset** archives the history; nothing is deleted.
- **Top-ups** are recorded as deposits, and returns are **time-weighted**, so adding money never looks like profit.
- **Manual paper orders** (beginner practice) go through the same risk layer. They are booked to the account's Manual sleeve, funded from the reserve (doc 05 PF-014), and never touch a bot's sleeve or its graduation evidence.
- Everything is labeled **"PAPER · simulated money"** (doc 07 UX-031), exports carry doc 08's paper watermark (CMP-012), and paper is never added to live totals.

### 8.3 Outages, gaps, restarts

| Situation | Behaviour |
|---|---|
| Symbol data stale: > 10 s without trades or quotes, a book-sequence gap, or a missing 1m bar (thresholds owned by doc 05 §4.4) | Data is flagged **stale**, a data-health flag rather than a trading state. New entries are denied, cancels always pass, and exits go out only as protected orders priced from a fresh reference; no fills against stale prices. Venue-wide staleness > 30 s sets the venue REDUCE_ONLY. The in-app freshness indicator updates at once; after 2 min of staleness while bots are running or positions are open, an external Warning goes out (held during quiet hours in paper) |
| Reconnect | Missed data is backfilled via REST. Strategies process missed bars in **catch-up mode**: state updates, but no orders for bars older than a threshold (default 2 bars) |
| Resting orders across a gap | Stops trigger at the first post-gap price. Limit orders fill only if backfilled trades confirm the price traded through; such fills are flagged "gap fill" |
| Exchange maintenance | Same as stale data; the incident is logged on the bot's timeline |
| Ubik crash or restart | State is rebuilt from the event-sourced ledger and strategy snapshot; open orders are restored without duplicates |

All live data consumed by paper bots is **recorded**: trades and top of book always, and L2 depth for up to 5 instruments in P1 (retention per doc 06 NFR-011). This enables the parity replay, builds a free archive of high-fidelity tick and L2 data, and feeds doc 06's storage plan.

### 8.4 Forward test vs backtest: detecting divergence

1. **Daily parity replay:** re-run the same strategy version through the backtester on the *recorded* paper-period data and compare trade by trade:
   - **Signal match rate:** a miss means a parity bug, a data difference or a timing problem.
   - **Fill difference** in basis points: the cost-model error, in the spirit of *implementation shortfall* (doc 05's transaction-cost analysis, TCA).
   - **PnL tracking error** between the two equity curves.
   - When several bots share a paper account, the replay runs one bot at a time (backtests of several bots together arrive in P2, BT-019). Where an account-level limit reduced or denied one of the bot's orders, the replay uses the stored risk decision (doc 05 RK-003) and labels the difference as an account limit, not a parity miss.
2. **Expectation band**, two uses:
   - *Gate:* doc 05's definition. Paper per-trade returns must be no worse than the 10th percentile of a bootstrap of the backtest's trades, and drawdown must not exceed the **reference drawdown** (the worst drawdown of the strategy version's full backtest, development set plus vault, net of costs and under its risk profile; doc 05 RK-020).
   - *Alert:* results outside the two-sided 5th–95th percentile Monte Carlo band (§6.3). Results that are suspiciously *good* are also investigated, because they often signal a bug.
3. **Liquidity sanity:** alert if the participation cap binds often.

### 8.5 Graduation criteria: leaving paper (stage S1 of doc 05's path to live)

Doc 05 §9.1 owns the full staged path: Draft → S0 Backtest → **S1 Paper** → S2 Exchange demo → S3 Shadow (read-only) → S4 Pilot (tiny capital) → S5 Scaled, with automatic demotion and the terminal state Retired. The stage is tracked per bot and is separate from the trading state (ACTIVE / REDUCE_ONLY / HALTED). S2–S5 unlock in P3. This section defines in detail what S1 requires, so that the gate tests *parity* and not just profit.

**Entering S1 (enforced from P1).** P1 tracks Draft, S0 and S1 per bot. A bot may start paper trading only when:
- the blocking §3.3 checks pass (ST-018), including the truncation look-ahead test (ST-019, CAP-009);
- kill criteria are declared (CAP-033). The default is the strategy drawdown breaker (doc 05 RK-020); the owner may declare stricter ones;
- if the backtest has fewer than 100 round trips, the owner has seen doc 05's RK-039 consequence preview and typed a confirmation.

Ubik also prompts for a one-line hypothesis (CAP-004), but this does not block. In P1 the out-of-sample check is the vault (BT-022); walk-forward, DSR and PBO join the S0 bar in P2. The table below is the S1 *exit* checklist: PT-014 shows progress against it from P1, and PT-015 enforces it from P3.

| Criterion | Default (can't be lowered except under the Adventurous profile, per PT-015) | Why |
|---|---|---|
| Same strategy version | Only one version hash for the whole period; any change resets the clock | Evidence must be about the exact code going live |
| Same venue | Paper on the intended live exchange, pair and fee tier | Liquidity, rules and availability differ by venue and jurisdiction |
| Duration and round trips | ≥ 30 calendar days **and** ≥ 20 round trips (doc 05 §9.1 S1). Schedule-based strategies (DCA, rebalancing) need ≥ 8 completed cycles instead | Minimal sample. Later stages S3 and S4 add another 44+ days before scaling |
| Signal match with the parity replay | ≥ 95 %, with every mismatch explained | Proves parity |
| Cost-model accuracy | Mean fill difference ≤ 1.5 × modeled slippage | Backtest cost assumptions hold |
| Consistency | Doc 05's expectation-band gate (§8.4) | Catches strategies that only worked in hindsight |
| Max drawdown | ≤ the strategy drawdown breaker (RK-020) | Safety |
| Backtest quality (S0) | ≥ 100 backtest round trips over the version's full backfilled history on the traded pair, with no pooling across pairs (schedule-based strategies: ≥ 24 completed cycles; doc 05 §9.1 S0); out-of-sample net Sharpe > 0 (the vault in P1, plus walk-forward from P2); matches or beats buy-and-hold risk-adjusted (or the owner acknowledges it doesn't); DSR ≥ 0.95 and PBO ≤ 0.5 if optimized (P2+); history includes a bear phase | Paper can't rescue a weak backtest |
| Operations | No unresolved incidents or reconciliation breaks in the last 14 days; ≥ 99 % uptime (doc 05); a restart recovery has succeeded; stale-data handling has been exercised | Live is unforgiving |
| Consent | Explicit owner approval (doc 07); promotion is never automatic | Informed decision |

**Honesty note shown at graduation:** passing paper proves faithful *implementation*, not *profitability*. By the MinTRL formula, with normal returns, showing at 95 % confidence that a true annual Sharpe of 1.0 is above zero takes about **2.7 years** of daily returns (≈ (1.645 ÷ (1/√365))² ≈ 990 days), and longer with crypto's fat tails. P3 therefore starts with tiny capital.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| PT-001 | The paper venue shall use the backtester's matching engine (`SimVenue`), ledger and fee/slippage/latency/rule models, fed by live public market data from the selected exchange. | Must | P1 | Parity with backtest. |
| PT-002 | Paper limit orders shall fill only when live trades print through the limit (touch probability configurable). Market orders shall fill from the live book (L2 walk if available, else half-spread plus slippage). Fill size shall be capped by participation in live volume. | Must | P1 | Realism without market impact. |
| PT-003 | Paper trading shall run without exchange API keys, and no live-order code path shall be reachable from a paper bot. | Must | P1 | Mode separation; doc 08. |
| PT-004 | Ubik shall support multiple paper accounts, each with its own base currency, capital, venue, fee tier and one or more bots, each bot in its own sleeve (doc 05 PF-001). | Must | P1 | A/B tests; practice. Matches doc 07 UX-021. |
| PT-005 | Paper accounts shall support reset (history archived) and top-up (recorded as a deposit), with time-weighted returns. | Must | P1 | Top-ups must not inflate performance. |
| PT-006 | Ubik shall allow manual paper orders from the UI through the same risk layer, booked to the account's Manual sleeve (doc 05 PF-014). | Should | P1 | Beginner practice. Manual orders never change a bot's sleeve or graduation evidence. |
| PT-007 | When market data is stale (thresholds in doc 05 §4.4), the paper venue shall deny new entries, produce no fills against stale prices, alert the owner per doc 05 §4.4 (a Warning in paper), and log the gap. Cancels shall always pass. | Must | P1 | Never fill on stale prices. "Stale" is a data-health flag, not a trading state (RK-004). |
| PT-008 | After reconnecting, Ubik shall backfill missed data, process missed bars in catch-up mode (no orders for bars older than a threshold), and apply gap-aware handling to resting orders, flagging gap fills. | Must | P1 | |
| PT-009 | Paper state (ledger, orders, strategy state) shall be event-sourced and crash-safe. Paper bots shall resume after a restart without duplicate or lost orders. | Must | P1 | Rehearses P3 recovery. |
| PT-010 | Ubik shall record the live market data consumed by paper bots: trades and top of book, and L2 where available. | Must (L2: Should) | P1 | Parity replays; future high-fidelity backtests. P1 L2 recording is capped at 5 instruments (doc 06 NFR-011). |
| PT-011 | Ubik shall run a daily parity replay per paper bot, reporting signal match rate, fill difference (bps) and PnL tracking error in the bot's parity report (doc 07 UX-011). | Must | P1 | Measures real parity. Delivers CAP-025; its §8.5 thresholds (signal match ≥ 95 %, mean fill difference ≤ 1.5 × modeled slippage) are the G3 measure. |
| PT-012 | Ubik shall alert when paper performance leaves the backtest's two-sided 5–95 % Monte Carlo band for the elapsed horizon, and shall evaluate doc 05's one-sided expectation-band gate. | Should | P2 | Early divergence warning; "too good" is investigated too. |
| PT-013 | All paper results shall be labeled "PAPER · simulated money" in the UI and reports (doc 07 UX-031), exports shall carry the paper watermark (doc 08 CMP-012), and paper results shall never be aggregated with live results. | Must | P1 | Honesty; doc 08. |
| PT-014 | Ubik shall display each paper bot's progress against the §8.5 criteria. Criteria that need P2 features shall show "available from P2". | Should | P1 | Feeds doc 05's graduation scorecard (EX-022); the full S0–S5 UI is doc 07 UX-010 (P3). |
| PT-015 | Ubik shall block promotion beyond S1 for any bot whose strategy version hasn't met §8.5. The duration, round-trip (or cycle) count and same-version criteria shall never be overridable, and a new version always re-enters at S1. Other §8.5 criteria may be overridden only under the Adventurous risk profile (doc 05), with a logged justification and a 24 h cooling-off. | Must | P3 | Core path-to-live safety. The owner's choice is tracked in doc 10 Q-15. |
| PT-016 | Exchange demo environments shall be used only to verify the live adapter's order lifecycle (stage S2), never as evidence of strategy performance. Testnets shall not be used for graduation. | Should | P3 | Testnet prices aren't real. Venues without a realistic demo meet S2 per doc 05 §9.1. |

---

## Open questions for the owner

1. **Which exchange will you trade on live, and from which country?** Graduation requires paper trading on the intended venue, and availability differs by jurisdiction (e.g. Binance vs Binance.US). Your fee tier also determines which strategies are viable at all.
2. **Decided (D1, doc 10 D-07):** P1 offers templates and the rule builder only; the Python SDK (ST-017) arrives in P2, running in the SEC-035 sandbox.
3. **What drawdown could you tolerate on real money** (−10 %, −15 %, −20 %)? Your answer can only tighten the strategy drawdown breaker (doc 05 RK-020, at most 20 %) used in warnings and graduation; it can never loosen it.
4. **Is the paper floor long enough?** The defaults are 30 days and 20 round trips, followed by exchange demo, read-only shadow and a tiny-capital pilot (doc 05), about 80+ days before scaling. A beginner may prefer a 90-day paper period before *any* real money. (DCA and rebalancing keep their schedule-based track of ≥ 8 cycles, doc 10 D-13. Tracked as doc 10 Q-15.)
5. **Grid bots in P1?** Recommended, with the ST-014 safeguards.
6. **Will strategies or results ever be shown to other people** (friends, a community, P4 SaaS)? Publishing hypothetical performance can be regulated, so doc 08 needs to know.
7. **Pay for historical tick or L2 data later (P2)**, or rely on Ubik's own recordings from P1?

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **False confidence**: a pretty equity curve is read as a promise | Real losses in P3; lost trust | Trust check first; DSR/PBO; vault; MinTRL note; tiny-capital start |
| **Parity drift** between backtest and paper/live (timing, data differences) | Graduation evidence invalid | Shared engine (BT-001); parity replays (PT-011); golden tests |
| **Fill-model error**: bar data hides queue position and intrabar order | Profit misestimated, especially for grids and limit-order strategies | Conservative defaults; cost ×2; recorded tick/L2 data; calibration from paper fills |
| **Stale fees and rules** (e.g. Kraken's July 2026 change) | Wrong cost assumptions | Effective-dated data in one venue registry (doc 06 AR-002); daily rule snapshots; warning when a schedule is > 90 days old |
| **Scope creep** ("everything pro firms do") | P1 slips | Ubik's own thin core (D2) plus proven libraries (ccxt, TA-Lib, Lightweight Charts); NautilusTrader limited to the time-boxed design spike (doc 06 AR-028); P1 limited to 1m simulation and 7 templates |
| **Own engine core**: Ubik's own thin core (D2) risks simulator bugs | False confidence, or rework | Golden and differential tests (§4.5); the doc 02 spike informs the design; NautilusTrader is re-evaluated only on doc 02 §4.4's revisit trigger |
| **AI assistant as an overfitting machine** | Inflated results | Trial ledger counts AI variants; DSR; assistant can't start runs |
| **Regime dependence**: tuned on the 2023–25 bull, deployed into the 2025–26 bear | Drawdowns beyond expectations | Stress library; regime warnings; walk-forward |
| **Exchange failure** (FTX-type), invisible to any backtest | Loss of funds held on the venue | Venue limits (doc 05); security (doc 08) |
