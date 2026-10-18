# Market & Technical Analysis

*Ubik planning doc 03. Scope: technical analysis and charting (TA), market analysis (MA), and the scanner, signals and alerts (SIG). Sources checked 2026-10-18. Updated 2026-10-18 for **Hyperliquid as the primary venue** (D-45; details in [doc 11](11-hyperliquid-integration.md), cited here by HL-xxx IDs rather than repeated).*

## Summary

- **Hyperliquid (HL) is the primary venue; Binance is secondary** (D-45). Ubik analyses HL spot (UBTC, UETH, USOL, HYPE against USDC) and HL perps (BTC, ETH, SOL). Because a **1× isolated perp is a P1 trading vehicle** (D-46 as amended by the lead), the perp's **mark price, oracle price, hourly funding and open interest are P1 data**, not P2 analysis extras. Binance (USDⓈ-M perps) and Coinbase (USD spot) supply the long *proxy* history, always labelled with a fidelity grade (D-52, HL-035).
- **The owner's horizon is 5m–1h, starting at 1h** (Q-04). Analysis defaults to 1h bars. Every timeframe shows how big a typical bar move is next to the round-trip cost (MA-032), because at 5m a round trip eats most of a typical move.

- **Build one indicator engine first, and use it everywhere.** The chart, scanner, alerts, strategies, backtests and paper/live trading all call the same code. It works on closed bars only and updates incrementally (one new bar at a time, without recomputing history). Tests check that its streaming output matches a batch reference (TA-Lib). This engine is what makes "sim-live parity" real.
- **Keep "describing the market" separate from "predicting it", and label every output with how strong the evidence is.** The research is reasonably consistent:
  - Volatility (the size of moves) is forecastable; direction mostly is not.
  - Trend and time-series momentum have the best (but fragile) support in crypto.
  - Candlestick and chart patterns have weak support.
  - On-chain "cycle" metrics rest on 3–4 market cycles.

  Every card, alert and scan carries a badge: Descriptive, Supported, Mixed, Weak or Untested.
- **Keep P1 small and honest.** P1 ships:
  - about 20 core indicators;
  - a multi-pane chart that shows trades from backtests and paper trading;
  - rule-based labels for trend and volatility;
  - price, indicator and regime alerts;
  - a **Market Health Summary** built from descriptive items only.

  There is no "buy score".
- **Store all data point-in-time.** Each value records when it happened and when it could first have been known. Revisable data (on-chain, sentiment, macro, ETF flows) is stored append-only. Backtests that use revised history are flagged as optimistic.
- **Start recording derivatives and market-wide data in P1.** HL perp data (mark, oracle, funding, OI) is *used* from P1; most other derivatives and market-wide analysis ships in P2. Free history is short or missing:
  - Hyperliquid's public API keeps only the last **5,000 candles** per interval (about 3.5 days of 1m), and real HL perp trading starts on **2023-02-26**. Ubik records HL from day one (HL-032).
  - Binance's API serves only 30 days of open interest and long/short ratios.
  - Historical BTC dominance from CoinGecko is paid-only.
- **Use free data first and track licenses.** Exchange APIs, Deribit, alternative.me, DefiLlama, FRED/ALFRED and Coin Metrics Community cover most of P2. Several of these are non-commercial or require attribution, which matters if Ubik ever goes multi-user (P4).

---

## 1. Principles and evidence

### 1.1 Design principles

1. **Compute once, consume everywhere.** An indicator is a pure function of point-in-time inputs.
2. **Closed bars by default.** A *bar* (candle) is one period's open/high/low/close/volume (OHLCV). The still-forming bar is drawn on the chart but never feeds a decision unless a rule explicitly asks for intrabar data.
3. **Describe first, predict carefully.** Volatility, spread and drawdown describe *now* accurately. They are not forecasts of direction.
4. **Show the evidence.** Every output says what it means, why it matters and how reliable it is.
5. **Swappable sources.** Every series sits behind a source adapter (doc 06), so upgrading to paid data is a configuration change.

### 1.2 Evidence labels

| Label | Meaning |
|---|---|
| **Descriptive** | Accurately measures the present; makes no forecast (spread, realized volatility, dominance). |
| **Supported** | Several peer-reviewed crypto studies found the effect out-of-sample (on data not used to build it) and after costs. It may still fade. |
| **Mixed** | Studies disagree, or samples are short or old. |
| **Weak** | Rigorous tests mostly find no reliable edge. Shown for education. |
| **Untested** | No evidence either way. This is the default for user-defined indicators. |

A user's own Signal Lab result (MA-022) is shown *next to* the label and never replaces it.

The evidence label rates what published research says about an indicator or signal. It is separate from the backtest trust check (doc 07), which rates one backtest from Low to Higher.

### 1.3 What has, and hasn't, shown predictive value

| Family | Evidence | Label |
|---|---|---|
| Volatility forecasting | Volatility clusters. HAR models (a regression on daily/weekly/monthly realized volatility) forecast Bitcoin volatility out-of-sample ([Reading WP](https://reading-clone.eprints-hosting.org/87932/1/R%26R_Forecasting%20the%20Volatility%20of%20Bitcoin_complete.pdf), [Bouri et al. 2021](https://repository.up.ac.za/handle/2263/76318)). | Supported for *size* of moves, not direction |
| Trend / time-series momentum | Price-to-moving-average ratios predicted daily BTC returns in and out of sample ([Detzel et al. 2021](https://profiles.wustl.edu/en/publications/learning-and-predictability-via-technical-analysis-evidence-from-/)). Time-series momentum appears at daily and weekly horizons ([Liu & Tsyvinski 2021](https://nber.org/papers/w24877)). But across ~15,000 rules there was *no out-of-sample predictability for Bitcoin* ([Hudson & Urquhart 2021](https://ideas.repec.org/a/spr/annopr/v297y2021i1d10.1007_s10479-019-03357-1.html)). Momentum weakened after BTC futures launched ([abstract](https://utilities.tinbergen.nl/?p=66235)). | Supported, fragile |
| Oscillator thresholds (e.g., RSI < 30) | Heavily data-mined. Thresholds are unstable, and "oversold" persists in downtrends. | Weak–Mixed |
| Candlestick patterns | 68 patterns across 23 coins were of little use ([IEEE 2021](https://ieeexplore.ieee.org/document/9671826)). Isolated exceptions don't generalize ([Emerald 2024](https://emerald.com/insight/content/doi/10.1108/IJQRM-12-2022-0363/full/html)). | Weak |
| Chart patterns, Fibonacci, Elliott | Little rigorous evidence. Identification is subjective and happens after the fact. | Weak |
| Cross-sectional market/size/momentum | A three-factor model explains crypto returns across coins ([Liu, Tsyvinski & Wu 2022](https://papers.ssrn.com/abstract=3379131)). Much of the effect sits in small coins, and costs bite ([Han, Kang & Ryu](https://acfr.aut.ac.nz/__data/assets/pdf_file/0009/918729/Time_Series_and_Cross_Sectional_Momentum_in_the_Cryptocurrency_Market_with_IA.pdf)). | Supported (broad universe only) |
| Carry (funding/basis) | Average futures–spot carry was ~7%/yr in 2019–2024, with spikes above 40% ([BIS WP 1087](https://www.bis.org/publ/work1087.htm)). Earning it requires derivatives. | Supported as a return source; Mixed as a directional signal |
| Attention (Google searches) | Predicted returns 1–several weeks ahead, but the sample ends in 2018 ([summary](https://www.cxoadvisory.com/currency-trading/crypto-asset-risks-and-returns/)). | Mixed |
| Fear & Greed | Short-sample studies find some 1-week to 1-month predictive content ([Cavalheiro et al. 2024](https://ideas.repec.org/a/eme/rbfpps/rbf-08-2023-0224.html)). About half the index is price-derived, so it double-counts TA. | Mixed |
| On-chain valuation (MVRV, SOPR) | Mostly vendor claims calibrated on 3–4 cycles, with inconsistent thresholds. Glassnode shows revised data inflating backtests ([Glassnode](https://research.glassnode.com/why-use-point-in-time-data/)). | Weak |
| Exchange flows | One preprint finds intraday predictability, mainly for ETH ([arXiv 2411.06327](https://arxiv.org/pdf/2411.06327)). | Mixed |
| Order-flow imbalance (OFI) | Explains price changes over the *same* seconds-to-minutes interval ([Cont et al. 2014](https://arxiv.org/pdf/1011.6402)). | Descriptive; not actionable at Ubik's latency |
| LLM news sentiment | Predicted next-day stock returns, mostly small caps ([Lopez-Lira & Tang](https://arxiv.org/abs/2304.07619v5)). LLM training data leaks the future into backtests ([Glasserman & Lin](https://arxiv.org/abs/2309.17322)). | Mixed; high leakage risk |
| ML return prediction | Even on large stock panels, out-of-sample R² is a fraction of a percent per month ([Gu, Kelly & Xiu 2020](https://www.nber.org/papers/w25398.pdf)). Most crypto ML studies are in-sample ([IJIM-DI 2024](https://doaj.org/article/38d750a5f9fd4328924b125548276b93)). | Untested → P4 research |
| Volatility-managed scaling | Failed out-of-sample in equities ([Cederburg et al. 2020](https://www.lehigh.edu/~xuy219/research/COWY.pdf)). | Use for risk control (doc 05), not alpha |

**Meta-evidence.** Most published finance "discoveries" are likely false positives; [Harvey, Liu & Zhu](https://papers.ssrn.com/abstract=2513152) argue for a t-statistic hurdle above 3. The more variants you try, the likelier the best one is luck ([PBO](https://papers.ssrn.com/abstract=2326253), [Deflated Sharpe](https://papers.ssrn.com/abstract=2460551)). This is why Ubik counts every variant a user tests in its trial ledger (MA-023, doc 04 BT-021).

---

## 2. Technical analysis

### 2.1 Indicator catalog

**Bold = P1.** Everything else is P2.

| Family | Indicators | Beginner one-liner |
|---|---|---|
| Trend | **SMA, EMA, MACD, ADX/±DI, Donchian channel, Supertrend**; WMA, Hull MA, KAMA, Ichimoku, Parabolic SAR, Aroon, regression slope, Kaufman efficiency ratio | "Is price heading up, down or sideways, and how strongly?" |
| Momentum | **RSI, Stochastic, Rate of Change**; Stoch RSI, Williams %R, CCI, TSI | "How fast price moved. Stretched readings can snap back, or stay stretched in strong trends." |
| Volatility | **ATR, Bollinger Bands (+%B, bandwidth), realized volatility, volatility percentile**; Keltner, Parkinson/Garman-Klass/Yang-Zhang, squeeze, volatility cones | "How big swings are now versus normal." |
| Volume | **Volume + MA, relative volume (RVOL), OBV, VWAP (UTC day)**; anchored VWAP, MFI, Chaikin Money Flow, cumulative volume delta (CVD), volume profile | "Is real activity behind this move?" |
| Support/resistance | **Prior day/week high-low-close, classic pivots, confirmed swing highs/lows**; auto S/R zones, round numbers | "Where buying or selling clustered before. These are zones, not walls." |
| Market structure | **Higher-highs/lows trend state, regime label (MA-001)**; break of structure, confirmed ZigZag, Choppiness index | "Is the market stepping up or stepping down?" |

Abbreviations, once:

| Term | Meaning |
|---|---|
| SMA / EMA | Simple / exponential moving average |
| MACD | Difference of two EMAs |
| ADX | Trend strength, 0–100 |
| ATR | Average true range: typical bar range |
| VWAP | Volume-weighted average price |
| OBV | Running total of volume signed by bar direction |
| Volume profile | Volume traded at each price |

**Patterns.**
- **Candlestick patterns** (P2, labelled Weak): engulfing, hammer/shooting star, doji, inside bar, morning/evening star. Detected on closed bars only.
- **Chart patterns** (P4, Could, labelled Weak): double tops, triangles, flags, head-and-shoulders. Detected only once *confirmed* (the breakout bar has closed). Each detection is stamped with the bar at which it became knowable, because pattern detectors "repaint" (they redraw history as new bars arrive).

**Multi-timeframe.**
- Timeframes: 1m, 5m, 15m, 1h, 4h, 1d, 1w. All are built from a stored 1-minute base series with UTC boundaries.
- A 1h strategy may read the daily trend, but it sees only the *last closed* daily bar.
- **Default timeframe: 1h** (the owner's starting horizon, Q-04). 15m is the next step once a 1h strategy has survived paper trading; 5m is labelled *Experimental*. 4h and 1d remain available as trend filters.
- **On Hyperliquid the 1m base comes from Ubik's own recorder** (HL-032), because the public API returns only the last 5,000 candles per interval. Older 1m bars come from basis-aligned proxy history (Binance USDⓈ-M perps for HL perps, Coinbase USD for HL spot), each bar carrying `source_venue` and a fidelity grade A–D (HL-034, HL-035). Charts show the grade so a proxy wick is never mistaken for an HL print.

**Libraries** (doc 02 owns build-vs-reuse):
- **[TA-Lib](https://pypi.org/project/TA_Lib/) 0.8.2** (pinned; BSD license) as the batch reference. Its wheels have bundled the C library since version 0.6.5, and it supports NumPy 2.
- **An incremental engine:** our own, or based on [talipp](https://github.com/nardew/talipp), which is young; evaluate it first.
- **Avoid pandas-ta.** Its GitHub repo is gone and its last release was a beta in Sep 2025 (doc 06 §8.2; AR-025 bars unmaintained libraries).
- **Charting: [TradingView Lightweight Charts 5.2](https://www.tradingview.com/blog/en/tradingview-lightweight-charts-version-5-50837/)** (pinned per doc 06 §8.2; multi-pane, plugin "primitives"). Its license is Apache-2.0 **plus attribution**: a NOTICE and a visible link to tradingview.com, which its `attributionLogo` option provides. Drawing tools are not included and must be built as plugins. It draws on a canvas, so Ubik builds its own accessible layer (text summary, data table, keyboard crosshair; 07 UX-046, 06 AR-027) and verifies it in the P1 spike.

**Charting stance.** The chart must show what the strategy *saw and did*:
- trade markers carrying the rule that fired;
- a synced equity pane;
- a bar-replay mode that hides the future. This is a strong learning tool with no look-ahead.

**Drawings and backtests.** A line drawn today over old candles contains hindsight. Drawings can trigger alerts, but in backtests they apply only from their creation time.

### 2.2 TA requirements

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| TA-001 | Ubik shall implement each indicator once, in one library used by chart, scanner, alerts, strategies, backtester, paper and live engines. | Must | P1 | Sim-live parity. |
| TA-002 | Strategies, alerts and scans shall receive closed-bar values only by default. The chart shall draw forming-bar values in a visually distinct style. | Must | P1 | Main source of look-ahead/repaint bugs. |
| TA-003 | Streaming and batch computation of every indicator shall match (relative tolerance ≤1e-9) on a shared dataset. Batch output shall match TA-Lib where TA-Lib has the indicator. | Must | P1 | CI parity suite. |
| TA-004 | Each indicator shall declare: inputs, parameter defaults and ranges, warm-up length, outputs, repaint flag, evidence label, plain-language text and glossary link. | Must | P1 | Drives UI and explanations. |
| TA-005 | Ubik shall provide the P1 (bold) indicators in §2.1. | Must | P1 | |
| TA-006 | Ubik shall provide the P2 indicators in §2.1, including volume profile and CVD (which needs trade data). | Should | P2 | |
| TA-007 | Indicators shall output null until warm-up is met. Backtests shall auto-load pre-start history and shall not act before warm-up. | Must | P1 | No fake early signals. |
| TA-008 | Ubik shall support 1m, 5m, 15m, 1h, 4h, 1d and 1w bars aggregated from a 1-minute base with UTC boundaries. | Must | P1 | Native candles are kept for cross-checks only. |
| TA-009 | Strategies, scans and alerts shall be able to reference higher timeframes. Higher-timeframe values shall update only when that bar closes. | Must | P1 | |
| TA-010 | Repainting constructs (swing points, fractals, ZigZag, pivots, patterns, Ichimoku lagging span) shall be exposed to strategies only in confirmed form, with the lag stated. | Must | P1 | |
| TA-011 | Ubik shall detect candlestick patterns on closed bars, labelled Weak. | Could | P2 | |
| TA-012 | Ubik shall detect confirmed chart patterns, stamped with the bar at which each became knowable. | Could | P4 | |
| TA-013 | The chart shall offer candlestick, OHLC, line and area styles and linear/log scales. Heikin-Ashi (smoothed synthetic candles) is P2, with a permanent note that fills never occur at those prices. | Must | P1 | Common beginner backtest trap. |
| TA-014 | The chart shall offer overlays, ≥3 sub-panes, a synced crosshair and a data window showing all values at the cursor. | Must | P1 | |
| TA-015 | Backtest and paper/live trades shall appear as markers. Each tooltip shall show time, side, price, size, fees, P&L and the rule that fired. | Must | P1 | |
| TA-016 | The chart shall draw position, stop-loss, take-profit and resting-order lines. | Must | P1 | Doc 05. |
| TA-017 | Backtest views shall show equity and drawdown panes time-synced to price. | Should | P1 | Doc 04. |
| TA-018 | The chart shall visibly mark data gaps and outages and shall never interpolate OHLC. | Must | P1 | |
| TA-019 | The forming candle shall update within 1 s of new trade data. The chart shall pan and zoom 20,000 bars at ≥30 fps on a mid-range laptop. | Should | P1 | |
| TA-020 | The chart shall display times in the user's local time zone while all computation uses UTC. Tooltips shall show bar open and close times. | Should | P1 | |
| TA-021 | Ubik shall offer a bar-replay mode that hides future bars. | Should | P2 | Learning tool. |
| TA-022 | Drawing tools shall include: horizontal line, trend line, zone, Fibonacci (labelled Weak) and text (P1); then ray, channel, measure, risk/reward tool and anchored VWAP (P2). | Should | P1/P2 | |
| TA-023 | Drawings shall persist per user and symbol and carry a creation time. They shall be usable as alert triggers. In backtests they shall apply only from creation time. | Should | P2 | Prevents hindsight. |
| TA-024 | Users shall be able to save layouts and indicator templates. At phone widths (≥ 320 px, WCAG 1.4.10 reflow) the chart shall show a single pane with touch gestures and the summary sentence on top; sub-panes shall be available from tablet width up. | Should | P1 | Doc 07 owns mobile layout (§9, UX-051). |
| TA-025 | Users shall be able to define custom indicators (rule language, later Python plugins) that work in every TA-001 consumer. They are labelled Untested. Python indicator plugins shall run only in the 08 SEC-035 sandbox. | Should | P2 | Doc 04. P1 loads first-party indicators only (08 SEC-034). Python plugins (P2+) run in the 06 AR-037 sandbox process, must declare TA-004 metadata and must pass TA-003/TA-028. |

---

## 3. Market analysis

### 3.1 Regimes

A **regime** is a persistent market state, such as trending vs ranging, or calm vs stormy.

**P1 uses transparent rules:**
- **Trend state** (Up / Down / Sideways), from:
  - price vs the 200-day SMA,
  - the slope of the 50-day SMA,
  - whether ADX is above about 20–25.
- **Volatility state** (Calm / Normal / Stormy): 30-day realized volatility ranked against its trailing 1-year values (bottom third / middle / top third).
- **Hysteresis:** a state must hold for N bars before the label flips.

**P2 adds statistical methods:**
- Hidden Markov models (which infer unobserved states from returns and volatility).
- Change-point detection.
- Variance-ratio and Hurst tests.

All P2 models are fitted on *expanding windows only*. A model fitted on all history "knows" where the regimes later fell. Regime labels are Descriptive. Using them as strategy filters is sensible, but it is still a hypothesis to test.

### 3.2 Volatility

| Measure | Phase | Detail |
|---|---|---|
| Realized volatility (what price actually did) | P1: close-to-close | P2 adds range-based Parkinson, Garman-Klass and Yang-Zhang, and 5-minute intraday estimates. Annualized over 365 days. |
| Implied volatility (the options market's expectation) | P2 | Deribit's **DVOL**, a 30-day BTC/ETH index similar to the VIX, via the free [`get_volatility_index_data`](https://docs.deribit.com/api-reference/market-data/public-get_volatility_index_data.md) endpoint. Deribit is now under Coinbase, which [mirrors the docs](https://docs.cdp.coinbase.com/api-reference/market-data/public-get_volatility_index_data.md). |
| Volatility risk premium (VRP) | P2 | Implied minus realized. |
| Forecasts | P2 | HAR-RV and GARCH. |

Uses:
- position sizing and stop distances (doc 05);
- a beginner "typical daily move" ("BTC moves about ±2.1% on a typical day right now");
- a **typical bar move per timeframe** set against the round-trip cost (MA-032). As a rough guide at recent volatility, BTC's median 1h move is a few tens of basis points, so an HL perp round trip at tier-0 taker fees (about 9 bp) costs a noticeable fraction of it; at 5m the typical move is only around 10–15 bp, so the same round trip eats most of it. Ubik computes the live figures rather than relying on these examples.

### 3.3 Correlation and cointegration

**Correlation:**
- Rolling correlations on daily returns (30/90 days).
- Beta of altcoins to BTC.
- Correlation to equities and the dollar (P2).

Beginners must be shown that correlations jump toward 1 in sell-offs, so holding BTC, ETH and SOL together diversifies little.

**Cointegration** means two prices that wander but stay tied together; it is the basis of pairs trading. P2 offers Engle-Granger and Johansen tests and the mean-reversion half-life, always with rolling stability. Crypto pair relationships shift across sub-periods ([Fil & Kristoufek 2020](https://doi.org/10.1109/ACCESS.2020.3024619)). Pairs *trading* needs shorting, so it waits for P4.

### 3.4 Dominance and breadth

**Dominance:**
- BTC dominance (BTC market cap ÷ total crypto market cap);
- ETH/BTC;
- stablecoin dominance.

**Breadth** (whether a move is broad or narrow):
- % of the top-N coins above their 50- and 200-day MAs;
- advancers vs decliners;
- new 30-day highs vs lows.

CoinGecko's free `/global` endpoint gives *current* dominance only; its history endpoint is [paid](https://docs.coingecko.com/reference/global-market-cap-chart.md). Ubik should snapshot it daily from P1.

Breadth needs a data universe (top 50–100 coins) wider than the traded pairs. That universe must use **point-in-time membership**, to avoid survivorship bias: testing only coins that still exist today.

### 3.5 Liquidity and microstructure

Measures:
- **Spread**, in basis points (1 bp = 0.01%).
- **Depth** within ±0.5%, 1% and 2% of the mid-price.
- **Slippage estimate** for the user's order size, from walking the order book.
- **Flow measures:** order-book imbalance, CVD, OFI.
- **Volume profile.**
- **Amihud illiquidity:** price impact per dollar traded.

These feed **execution and risk**: doc 05's slippage model and the scanner's liquidity filters. They are not alpha. OFI works over seconds ([Cont et al.](https://arxiv.org/pdf/1011.6402)), and a self-hosted bot paying taker fees cannot compete at that horizon.

Store per-minute depth bands (kept forever) plus raw top-20 snapshots, retained per doc 06 NFR-011. Shorter raw windows are a configurable disk-saving option, shown in the NFR-011 disk forecast. Full order-book diffs are heavy.

**Hyperliquid's order book arrives as full snapshots, not deltas.** The `l2Book` channel sends the whole visible book each time: 20 levels per side, or 5 levels with the `fast` option. Officially a push happens on a block at least 0.5 s after the previous push; on 2026-10-18 we measured about every 5.4 s for 20 levels and about every 0.5 s for `fast` (measured, not guaranteed; doc 11 §4.5). So on HL Ubik:
- records **both** the fast 5-level book (~0.5 s) and the 20-level book (~5.4 s), plus a coarser-price variant for wider depth bands (HL-032);
- treats each snapshot as complete (no sequence numbers to stitch), de-duplicates identical snapshots, and flags a gap when snapshots stop;
- computes spread and top-of-book from `bbo` and the fast book, and depth bands from the 20-level book. Depth more than about 20 levels from the touch is not visible in real time.

HL spot books are much thinner than HL perp books (BTC perp roughly 58× the volume of UBTC spot; doc 11 §3.1), so liquidity cards show the two side by side for the vehicle comparison (HL-006).

### 3.6 Derivatives

Perpetual futures ("perps": futures with no expiry, kept near spot by periodic **funding** payments between longs and shorts) dominate price discovery. On Hyperliquid they are also a **P1 trading vehicle**: a strategy may hold a 1× isolated perp instead of spot (D-46; paper shorts at ≤ 1× are allowed from P1 per the lead amendment, ST-010 in doc 04). So the perp metrics a position depends on (mark price, oracle price, funding, open interest) move from "P2 analysis" to **P1 data needed to trade** (MA-028). Market-wide crowding analysis built on them stays P2.

| Metric | Meaning | Caveat |
|---|---|---|
| Funding (per venue, OI-weighted, annualized) | Positive = longs pay shorts, so the crowd is leveraged long | Intervals vary (1h/4h/8h); store predicted *and* settled values |
| HL funding (P1) | Paid **every hour**; payment = position size × **oracle** price × rate. Annualized = hourly rate × 8,760 (24 × 365). A running cost for a long perp, income for a short | Capped at 4% per hour. HL charged 8-hourly until about mid-2023; normalize early history (doc 06). Measured BTC averages: about 24%/yr in 2024, 10.6% in 2025, 5.2% so far in 2026 (doc 11 §2) |
| Predicted funding, HL vs CEX (P1 capture, P2 display) | HL's free `predictedFundings` gives the next rate on HL, Binance and Bybit side by side | Shows when HL's crowd differs from the CEX crowd (HL-054) |
| Mark price (P1) | The price HL uses for stops, liquidations and unrealized P&L: a median of oracle-plus-premium, HL's own book and CEX perp prices | Not the last trade. In fast markets mark and mid can differ; see "mark vs mid" below |
| Oracle price (P1) | A weighted median of CEX spot prices (Binance weight 3; OKX, Bybit 2 each; Kraken, KuCoin, Gate, MEXC and HL itself 1 each), published by validators about every 3 s | The funding basis and the reference for HL's "too far from oracle" order rejection |
| Open interest (OI) | Total open contracts | The Binance API keeps only 30 days ([docs](https://developers.binance.com/docs/derivatives/coin-margined-futures/market-data/Top-Trader-Long-Short-Ratio)); daily archives exist on data.binance.vision ([issue](https://github.com/binance/binance-public-data/issues/211)). Venues count OI differently: one side of each contract, or both sides. HL's convention is **unverified**, so every OI series carries a convention label (MA-030) |
| Basis | Futures premium over spot, annualized | Quarterly futures roll over |
| Liquidations | Forced closures that amplify moves | Binance pushes at most one per symbol per second, so it **undercounts** ([docs](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Liquidation-Order-Streams)). Use as a relative gauge only. On HL every fill is on-chain, so liquidations and auto-deleveraging (ADL) can be counted from public fills (P2, MA-031) |
| Long/short ratios | Share of *accounts* net long | Definitions differ by venue. Every contract has both a long and a short side |
| Options (P4, Could): DVOL term structure, 25-delta skew, put/call | Demand for downside vs upside protection | "Max pain" is Weak |

Ubik presents funding and OI mainly as **leverage and crowding-risk gauges**. As a contrarian *direction* signal they are Mixed. For an open HL perp position, funding is also shown as a **cost (or income) in the position's own terms**: accrued funding and its annual rate, beside the liquidation price (HL-005).

Free aggregated history: [Coinalyze](https://dlthub.com/context/source/coinalyze) (free key, ~40 calls/min). [CoinGlass](https://coinglass.com/pricing) has no free API; it starts at $29/month for personal use. For HL itself, full hourly funding history is free from the public API (from 2023-05-12), and mark/oracle history comes from Ubik's recorder plus the optional official archive (HL-033, HL-039).

**Why the oracle matters to a beginner.** HL does not invent its own reference price. The oracle is essentially "what the big exchanges' spot markets say", with Binance weighted most. So HL perps follow CEX spot closely in calm markets (closes agree within a few basis points, doc 11 §6.2), and Binance history is a reasonable proxy for HL closes. Crash *wicks* are different: HL's own book can wick deeper (on 10 Oct 2025 HL spot UBTC fell 6–8% further than Coinbase BTC), which is why proxy history carries a wick warning (HL-035).

**Mark vs mid: what drives stops.** HL triggers stops and liquidations on the **mark** price, not the last trade or mid (HL-037). Indicators and conditions that place or move a stop on an HL perp (ATR stops, Donchian exits, trailing stops) are therefore computed on the **mark-price series** where it exists. Where a rule must use last or mid price, Ubik models and shows the **mark–mid gap** so the owner knows the stop may fire at a different moment than the chart suggests (MA-029).

**Hyperliquid symbols.** HL perps are addressed by their index in the venue's metadata (BTC = 0); spot pairs by `10000 + index`, displayed as `@142`-style names. HL spot "BTC" is really **UBTC**, a bridged token (doc 11 §2). **Testnet asset ids differ from mainnet.** Ubik keys every series on (network, asset id) from effective-dated registry snapshots (HL-040) and shows friendly aliases such as "BTC perp" and "BTC spot (UBTC, bridged)". Analysis never joins testnet and mainnet data.

**Builder-deployed (HIP-3) markets.** HIP-3 equity, commodity and index perps may appear in P2 as read-only, labelled "24/7 macro context" cards in Pro mode, hidden in Guided mode, and never tradable before a separate review (HL-009, D-54).

**Jurisdiction flag:** Binance.com rejects requests from US IP addresses (HTTP 451, "restricted location"), and several derivatives venues restrict some countries. Hyperliquid's Terms prohibit "Restricted Persons" (US or Ontario residence or location, sanctioned territories); live trading on HL cannot be armed for them (HL-045, D-53). Source selection depends on the owner's country (docs 06/08). Ubik must never suggest bypassing geo-blocks.

### 3.7 On-chain (P2; BTC and ETH first)

| Metric | Meaning | Source |
|---|---|---|
| Hash rate, transactions, active addresses | Network usage and security | Coin Metrics Community, mempool.space |
| MVRV (market value ÷ realized value) | Price relative to holders' average cost basis | Coin Metrics Community may include it (verify); otherwise paid |
| Stablecoin supply | Cash that could flow into crypto | [DefiLlama](https://stablecoins.llama.fi/stablecoins), free |
| Exchange flows/reserves, SOPR, NUPL | Possible selling pressure and profit-taking | Paid (Glassnode API is Professional-tier; CryptoQuant uses credits). Depends on address-labelling heuristics that get revised |

Coin Metrics Community needs no key (10 requests per 6 s). Its license is **CC BY-NC 4.0, non-commercial** ([docs](https://docs.coinmetrics.io/packages/coin-metrics-community-data)). That is fine for personal use and a blocker for a commercial SaaS.

### 3.8 Sentiment and news (P2)

**Fear & Greed** ([alternative.me](https://alternative.me/crypto/fear-and-greed-index/)):
- **Components:** volatility 25%, momentum/volume 25%, social 15%, surveys 15% (currently paused), dominance 10%, search trends 10%.
- **Access:** a free daily API. Attribution must appear next to the data.
- **Presentation:** a crowd-mood summary, not an independent signal.

**Google Trends:** the API is an application-only alpha ([Google](https://developers.google.com/search/blog/2025/07/trends-api)). Website values are re-scaled on every query, so history must be captured point-in-time. P4, Could.

**Social media data** (X, Reddit) is costly or restricted. P4 (Could), via a paid aggregator.

**News pipeline:**
1. Pull headlines from RSS feeds, an aggregator (CryptoPanic; verify its terms) and, most importantly for a trading bot, **exchange announcements** (listings, delistings, maintenance).
2. Deduplicate and tag each item by asset and event type.
3. Store headline, URL and timestamps only, never full text (copyright; each source's terms are recorded in the 08 CMP-007 data-source registry).
4. An LLM (large language model) writes cited digests with no price predictions, and the model version is logged. Headlines and announcements reach the LLM labelled as untrusted input (08 SEC-039).
5. LLM-derived features enter backtests only for dates after the model's training cutoff, or with anonymized inputs ([Glasserman & Lin](https://arxiv.org/abs/2309.17322)).

### 3.9 Macro and event calendar

**Macro series (P2), daily:**

| Series | Source |
|---|---|
| Dollar | FRED broad dollar index `DTWEXBGS`, a free proxy for the licensed ICE DXY |
| Treasury yields | FRED `DGS2`, `DGS10` |
| Equity volatility | FRED `VIXCLS` |
| Equities, gold | Free daily sources (FRED's S&P 500 history is limited) |
| US spot BTC/ETH ETF net flows | Farside, SoSoValue. Usually final only next morning, so treated as T+1 |

**ALFRED**, FRED's archive of past data vintages, supplies point-in-time macro values. Weekend values are carried forward with a staleness flag.

**Event calendar.** Every event stores `scheduled_at`, `announced_at`, importance, source and the assets it affects.

| Phase | Events |
|---|---|
| P1 | FOMC: the [Fed calendar](https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm) lists 2027 decisions on Jan 27, Mar 17, Apr 28, Jun 9, Jul 28, Sep 15, Oct 27 and Dec 8, at 2 pm ET. US CPI and payrolls (BLS schedules). The next Bitcoin halving: block 1,050,000, about April 2028, computed live from block height. Manual events. |
| P2 | Token unlocks ([Tokenomist](https://docs.tokenomist.ai/tier-overviews); web free, API paid or trial), network upgrades, exchange notices, Deribit and CME expiries. |

Uses:
- alerts;
- doc 05's optional event guardrail (RK-041, P2), which sets a strategy to REDUCE_ONLY or scales down its new entries around high-importance events;
- event studies, such as BTC's behavior around the last N FOMC meetings, with sample size and uncertainty.

### 3.10 MA requirements

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| MA-001 | Ubik shall compute rule-based trend state and volatility state per asset on 4h and 1d, with hysteresis. Both shall be usable as strategy conditions. | Must | P1 | |
| MA-002 | Ubik shall offer HMM and change-point regime models, fitted on expanding windows only, reporting how often labels flip. | Should | P2 | No full-sample look-ahead. |
| MA-003 | Ubik shall compute annualized (365-day) close-to-close realized volatility, its 1-year percentile, and a plain-language "typical daily move". | Must | P1 | Sizing (doc 05); beginner intuition. |
| MA-004 | Ubik shall provide range-based volatility estimators, DVOL (BTC, ETH), the VRP, and HAR/GARCH forecasts with their out-of-sample error shown. | Should | P2 | |
| MA-005 | Ubik shall compute rolling correlations among the traded pairs. | Should | P1 | Doc 05 concentration risk. |
| MA-006 | Ubik shall compute beta-to-BTC, correlation to macro series, and cointegration tests with rolling stability. | Could | P2 | |
| MA-007 | Ubik shall snapshot BTC dominance, total market cap and stablecoin market cap daily from P1, and chart them with ETH/BTC from P2. | Should | P1/P2 | History is not free. |
| MA-008 | Ubik shall compute breadth over a configurable top-N universe using point-in-time membership. | Should | P2 | |
| MA-009 | Ubik shall monitor spread and top-of-book depth per traded pair and flag deterioration. | Should | P1 | |
| MA-010 | Ubik shall compute depth bands and an order-size slippage estimate, plus CVD and OFI labelled "Descriptive / short-horizon". | Should | P2 | Shared with doc 05 TCA. |
| MA-011 | Ubik shall record funding (predicted and settled), OI and long/short ratios for BTC/ETH/SOL perps on at least one venue, starting in P1. | Should | P1 | Free history is short. On HL this is part of the Must recorder (HL-032, MA-028); Binance is the CEX venue (HL-053). |
| MA-012 | Ubik shall display funding, OI, basis and liquidations, each with its caveat text. Options data follows in P4 (Could). | Should | P2 | Includes HL vs Binance basis and funding (HL-054). |
| MA-013 | Ubik shall ingest the free on-chain set (§3.7) for BTC/ETH, recording each source's license and attribution. Paid metrics shall be pluggable by configuration, preferring point-in-time variants. | Should | P2 | |
| MA-014 | Ubik shall ingest Fear & Greed daily and display its attribution next to the value. | Should | P2 | |
| MA-015 | Ubik shall ingest headlines and exchange announcements, deduplicate them, tag asset and event type, and store `published_at` and `ingested_at`. | Should | P2 | |
| MA-016 | Ubik shall produce LLM news digests that cite sources, contain no price predictions, and log the model version. | Should | P2 | Doc 07 AI assistant. Headlines and announcements are untrusted input, labelled per 08 SEC-039; no assistant-memory writes. |
| MA-017 | LLM-derived features shall be used in backtests only for dates after the model's training cutoff or with anonymized inputs. Otherwise the backtest shall be flagged. | Must | P2 | |
| MA-018 | Ubik shall ingest macro series with vintage-aware storage, carrying weekend values forward with staleness flags. | Should | P2 | |
| MA-019 | Ubik shall maintain an event calendar with `scheduled_at` and `announced_at`: the P1 set in P1 and the full set in P2. | Should | P1/P2 | |
| MA-020 | Every analysis output shall show its source, as-of time and freshness. Stale values shall be greyed out and excluded from signals. | Must | P1 | |
| MA-021 | When an optional source is unavailable (geo-blocked, rate-limited, discontinued), the affected card shall say why and shall never present stale values as live. | Must | P1 | |
| MA-022 | A **Signal Lab** shall evaluate any signal's forward returns at 1h/4h/1d/7d. It shall show hit rate, mean, information coefficient (rank correlation with future returns), bootstrap confidence intervals, net-of-cost results and walk-forward splits. | Should | P2 | Research before strategy. Doc 04 owns strategy metrics. |
| MA-023 | The trial ledger (signal entries, doc 04 BT-021) shall count every signal or parameter variant a user evaluates, show the count, and apply deflated statistics to the best result. | Should | P2 | With doc 04 DSR/PBO. Same ledger as BT-021; signal entries count toward the DSR trial count of any strategy built from that signal family. |
| MA-024 | Ubik shall provide an event-study view and statistical tests: ADF/KPSS stationarity, variance ratio, Ljung-Box, and Granger causality labelled "association, not cause". | Could | P2 | statsmodels/arch. |

---

## 4. Quant, statistical and ML analysis

### 4.1 Feature engineering

A *feature* is any number fed to a rule or model. Ubik's conventions:
- Use returns rather than price levels. Prices are non-stationary: their statistics drift over time.
- Normalize by volatility: return ÷ ATR, or distance from a moving average in σ units.
- Compute z-scores and percentiles on **rolling** windows only.
- Use cross-sectional ranks and calendar fields (hour, weekday, time to funding).
- Use regime labels as conditioning variables.

All features are versioned and computed by the TA-001 engine.

### 4.2 Factor ideas

| Factor | Use in Ubik | Evidence | Phase |
|---|---|---|---|
| Time-series momentum / trend | Strategy templates (doc 04), regime filter | Supported, fragile | P1 |
| Cross-sectional momentum (rotate into the strongest coins) | Scanner rank; long-only rotation | Supported; costs bite | P2 |
| Volatility (targeting, low-volatility tilt) | Sizing (doc 05) | Risk-useful; as alpha, Mixed | P1/P2 |
| Liquidity (Amihud, volume shocks) | Universe filter | Mixed | P2 |
| Carry (funding/basis) | Positioning gauge now; trade it later | Supported | P2 view / P4 trade |
| Value-like on-chain (MVRV, NVT) | Context card only | Weak | P2 |
| Attention / sentiment, seasonality | Signal Lab only | Mixed / Weak | P2 (Signal Lab); further sources P4 |

### 4.3 Why ML overfits here (ML is mostly P4)

- **Tiny signal.** Little of the variation in returns is predictable.
- **Few independent samples.** There is one BTC price path and about 4 halving cycles.
- **Structural breaks.** Futures arrived in 2017 and spot ETFs in 2024.
- **Leakage.** Overlapping labels, full-sample normalization, random k-fold cross-validation on time series, survivorship-biased universes.
- **Hidden multiple testing.** Hyperparameter search is multiple testing in disguise.

Guardrails:
- purged, embargoed walk-forward cross-validation (gaps between training and test data so overlapping labels cannot leak);
- a locked holdout period;
- comparison with simple baselines (buy-and-hold, MA crossover, random entries);
- PBO and Deflated Sharpe reporting;
- a feature-stability check;
- a paper-trading incubation period.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| MA-025 | Ubik shall provide a versioned point-in-time feature store computed by the shared indicator engine. | Should | P2 | Basis for Signal Lab and ML. |
| MA-026 | Ubik shall compute cross-sectional factor scores (momentum, volatility, liquidity) over the data universe. | Could | P2 | Scanner ranks. |
| MA-027 | ML models shall be off by default. Before a model can drive paper trading, it shall pass purged/embargoed walk-forward CV, a locked holdout, baseline comparison and PBO reporting. | Should | P4 | |

---

## 5. Scanner, signals and alerts

### 5.1 Signals

Every signal records:
- an expression in the **same condition language as the strategy builder** (doc 04);
- its timeframe;
- `evaluated_at`;
- its inputs, with each input's `available_at`;
- its evidence label;
- its explanation.

This makes scan → alert → backtest → strategy one click in each direction.

### 5.2 Scanner

- **P1:** a watchlist with indicator columns; enough for a handful of pairs.
- **P2:** a full scanner:
  - **Universe filters:** venue, quote currency, minimum 24h volume, maximum spread, minimum history.
  - **Conditions** on any timeframe, with sorting.
  - **Timing:** runs at bar close and stores timestamped snapshots.
  - **Beginner presets**, each with an explanation and evidence badge: "Strong uptrend", "Pullback in uptrend", "Unusual volume", "Volatility squeeze", "Funding extreme", "New 30-day high".

### 5.3 Alerts

**Types:**
- **P1:** price level, % move in a window, indicator condition, regime change, unusual volume, volatility spike, data/system health.
- **P2:** drawing crossed, funding/OI extremes, liquidity deterioration, correlation break, event reminders, asset news.

"Unusual volume" (RVOL) uses a time-of-day baseline, because crypto volume has strong intraday seasonality.

**Noise controls:** cooldowns, re-arm with hysteresis, once / once-per-bar / every-time modes, expiry, quiet hours, per-channel rate limits, deduplication.

**Channels:**
- **P1:** in-app, plus at least one external (out-of-app) channel the owner chooses: email, Telegram (the de-facto crypto channel) or ntfy. Further P1 channels are Should.
- **P2:** web/PWA push, Discord, webhook.
- **P3:** SMS (Could).

The owner adds, tests and removes channels in Settings → Notifications (07 UX-054). Channel credentials are secrets under 08 SEC-014.

### 5.4 SIG requirements

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SIG-001 | Every signal shall be a typed object (expression, timeframe, `evaluated_at`, inputs with `available_at`, evidence label, explanation). | Must | P1 | Auditability. |
| SIG-002 | Scan and alert conditions shall use the strategy builder's condition language and be convertible to and from backtestable strategies. | Must | P1 | Doc 04. |
| SIG-003 | Ubik shall support the P1 alert types (§5.3), and the P2 types in P2. | Must | P1/P2 | |
| SIG-004 | Alerts shall evaluate at bar close by default. Only price-level alerts may evaluate intrabar, and they shall be marked as such. Each firing shall record its evaluation time and input values. | Must | P1 | Same timing as backtests. |
| SIG-005 | Alerts shall support cooldown, re-arm/hysteresis, trigger modes, expiry, quiet hours, per-channel rate limits and deduplication. | Must | P1 | Alert fatigue. Critical alerts bypass quiet hours (07 UX-038). |
| SIG-006 | Ubik shall deliver alerts in-app and via at least one owner-chosen external channel (email, Telegram or ntfy) in P1 (Must), via additional channels in P1 (Should), and by web push, Discord and webhook in P2. Delivery shall be at-least-once with a dedupe ID, with failures retried and visible. Median delivery shall be ≤10 s after bar close. | Must | P1/P2 | An external channel is needed for 06 NFR-017 and AR-021. Channel setup and testing: 07 UX-054. |
| SIG-007 | Alert messages shall state what happened, the value, its plain meaning, the evidence label and a chart link. They shall never use imperative trade language ("buy now"). | Must | P1 | Honesty. |
| SIG-008 | Users shall be able to backtest an alert: how often it would have fired, and the forward-return distribution after each firing. | Should | P2 | Uses MA-022. |
| SIG-009 | Ubik shall provide a watchlist with indicator columns (P1) and a full scanner with filters, presets and timestamped snapshots (P2). | Should | P1/P2 | |
| SIG-010 | Ubik shall raise data-health alerts for stale feeds, disconnects and gaps. | Must | P1 | Paper and live safety. Stale thresholds, trading effects and alert timing per doc 05 §4.4 (Warning in paper, Critical in live). |

---

## 6. Testability and data-timing rules

Every indicator and signal must work in strategies and backtests **without look-ahead**, meaning without using information that wasn't yet available at decision time.

```
observation(series_id, asset, period_start, period_end, event_time,
            value, available_at, first_seen_at, source, revision_no)
```

- `available_at` = the earliest moment a live system could have known the value: publisher time plus a documented lag.
- `first_seen_at` = when Ubik itself stored it.

**Rules:**
1. **Decision clock.** A decision at time *t* uses only observations with `available_at ≤ t`. For bars, *t* is the bar close. Fills occur no earlier than the next tradable price (fill model in doc 04).
2. **Closed bars only.** For explicitly intrabar rules, the backtester must use sub-bar data (1m or trades), never the bar's final OHLC.
3. **Higher timeframes** step at their own close.
4. **Append-only storage for revisable data.** This covers on-chain, sentiment, macro, ETF flows, market cap/supply, search trends and universe membership. A revision is a new row, and backtests read "as known at *t*".
5. **Pre-capture history** (backfilled before Ubik began recording) gets a conservative `available_at` (table below) and a "revised data — likely optimistic" badge.
6. **No full-sample fitting.** Thresholds, z-scores, percentiles, HMMs and ML use expanding or rolling windows only.
7. **Repaint-free.** Constructs that repaint are exposed only in confirmed form (TA-010).
8. **Warm-up** must complete before any signal (TA-007).
9. **Universe as of date.** This includes coins later delisted.
10. **Gaps stay gaps.** Values are forward-filled only within a per-series staleness limit, and filled values are flagged.
11. **News uses `ingested_at`.** Publishers edit and backdate timestamps.
12. **LLM leakage rule** (MA-017).
13. **Reproducibility.** Backtests are reproducible from stored inputs. Live LLM outputs are stored, never regenerated.

**Conservative `available_at` defaults for backfilled data** (final values set in doc 06):

| Data | `available_at` |
|---|---|
| OHLCV | Bar close + 2 s |
| Settled funding | Funding timestamp (predicted funding is a separate series) |
| OI, long/short ratios | Period end + 1 min |
| DVOL | Candle close |
| Fear & Greed | End of labelled UTC day + 1 h |
| Daily on-chain, DefiLlama | Day end + 24 h |
| FRED / ALFRED | Vintage date |
| ETF flows | Next US business day, 12:00 UTC |
| Events | `announced_at` |

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| TA-026 | All stored series shall carry `event_time`, `available_at` and `first_seen_at`. The backtester shall enforce `available_at ≤ decision time` for every input. | Must | P1 | Core guarantee. |
| TA-027 | Revisable series shall be stored append-only with revision numbers (`revision_no`) and read as-of the decision time. | Must | P2 | Doc 06 adopts these field names. |
| TA-028 | A look-ahead harness shall run every indicator and signal on truncated data and fail if any historical output changes when future bars are appended. | Must | P1 | Catches repaint and leakage. |
| TA-029 | Backtests using pre-capture revisable data shall display a "revised data" badge. | Must | P2 | |

---

## 7. Beginner presentation

Every analysis card, alert and scan result answers four questions:
1. **What this means:** the current value in plain words.
2. **Why it matters:** how traders use it.
3. **How reliable:** the evidence badge and one line of reason, linked to sources.
4. **Learn more:** a glossary link (doc 07).

Pro mode adds raw values, parameters and statistics.

| Family | "What it means" example | "How reliable" line |
|---|---|---|
| Trend | "BTC has been in a daily uptrend for 23 days." | "Supported but fragile: trend-following has worked historically, with long losing streaks." |
| RSI | "RSI 28: price fell fast recently." | "Weak as a buy signal alone: in downtrends, 'oversold' often gets more oversold." |
| Volatility | "Stormy: BTC moves ~±3.4% on a typical day now vs ~±2% normally." | "Descriptive. Future volatility is fairly forecastable; direction isn't." |
| Funding (P2) | "Traders pay 35%/yr to stay long: crowded longs." | "Mixed: crowding raises crash risk but can persist for weeks." |
| Fear & Greed (P2) | "82, Extreme Greed." | "Mixed: partly built from price itself; short research record." |
| MVRV (P2) | "Price is ~2.9× holders' average cost." | "Weak: based on only a few cycles; thresholds vary by source." |

**Market Health Summary.** One screen per asset plus a whole-market view.
- **P1 cards:** Trend, Volatility, Liquidity, Momentum, Upcoming events, Data freshness.
- **P2 adds:** Leverage/positioning, Sentiment, Macro, On-chain.

Cards are colored by *condition* (calm/stormy, thin/deep), never by buy/sell. The screen deliberately has **no composite buy/sell score**, because one confident number would hide weak components.

**Language rules:**
- Use probabilistic phrasing.
- Give every base rate with its sample size and a range ("in 31 similar cases since 2018, BTC was higher 7 days later 55% of the time; plausible range 37–71%").
- Below 30 cases, say "too few past cases to judge".
- Lint copy and AI-assistant output against the banned-phrase list in 07 ED-006 (e.g. "guaranteed", "can't lose", "will pump").
- Contrarian indicators always show both readings.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SIG-011 | Every analysis card, alert and scan result shall follow the What / Why / How-reliable / Learn-more template. | Must | P1 | Doc 07 (ED-001, ED-028). |
| SIG-012 | Ubik shall maintain a versioned evidence registry (label, one-line reason, citations) for every indicator and signal, shown as a badge. User-defined items default to Untested. | Must | P1 | Shown as the How-reliable layer (07 ED-028); distinct from the backtest trust check. |
| SIG-013 | Ubik shall provide the Market Health Summary with the P1 cards, extended in P2. | Must | P1 | |
| SIG-014 | Guided mode shall show no composite buy/sell score. Any composite score shall display its components, weights and out-of-sample record. | Must | P1 | Avoids false certainty. |
| SIG-015 | Every displayed historical statistic shall include sample size, period and an uncertainty range. With n < 30, Ubik shall show "insufficient history" instead. | Must | P1 | |
| SIG-016 | UI copy templates and AI-assistant market text shall pass an automated check against the 07 ED-006 banned-phrase list. | Must | P1 | Doc 07 ED-006 owns the list. AI-assistant text is checked from P2, when the assistant ships. |
| SIG-017 | Contrarian or ambiguous indicators shall present both interpretations. | Should | P2 | |

---

## 8. Data needs handoff (for doc 06)

| Analysis item | Data | Granularity | History (minimum needed; retention per 06 NFR-011) | Likely source (free → paid) | Phase |
|---|---|---|---|---|---|
| All TA, charts, regimes | Spot OHLCV + trade count, taker-buy volume | 1m base | Full available | Exchange REST, bulk archives (data.binance.vision; access terms per doc 08 §8.5), WebSocket | P1 |
| CVD, volume profile, fill realism | Trades | Tick | 30–90 days raw (06 keeps 2 years); aggregates kept | Exchange WS / aggTrades archives | P1 collect, P2 use |
| Spread, depth, slippage, OFI | L2 order book | Top-20 every 1 s (≤ 5 instruments in P1, 06 DA-003); depth bands per minute | 30 days raw (06 keeps 1 year); bands forever | Exchange WS | P1 basic, P2 |
| Watchlist, universe | Tickers, symbol rules, listings/delistings | 1 min; events | Point-in-time from P1 | Exchange REST | P1 |
| Breadth, dominance, factors | Daily OHLCV + market cap, top 50–100 | Daily | 5+ years | CoinGecko Demo (attribution), CoinMarketCap Basic, exchange klines | P2 (snapshot from P1) |
| Funding | Settled + predicted | Per event; 1 min | Full | Binance/Bybit/OKX public APIs; Coinalyze | P1 capture, P2 |
| OI, long/short | OI (contracts, USD), ratios | 5 min | API 30 days → self-capture; daily archives | Binance + data.binance.vision; Coinalyze → CoinGlass ($29+/mo) | P1 capture, P2 |
| Liquidations | Events (sampled) | Event | From capture | Exchange WS; Coinalyze → CoinGlass | P2 |
| Basis | Futures + index prices | 1 min–1 h | Full | Exchange APIs | P2 |
| Implied volatility | DVOL; later option chains | 1 min–1 h | DVOL since 2021 | Deribit public API | P2; option chains P4 (Could) |
| On-chain basic | Hash rate, transactions, active addresses, MVRV | Daily | Full | Coin Metrics Community (CC BY-NC), mempool.space | P2 |
| On-chain advanced | Exchange flows, SOPR, NUPL | Daily/hourly | Full, point-in-time | Glassnode Professional API, CryptoQuant | P2 optional paid |
| Stablecoins | Circulating supply | Daily | Full | DefiLlama | P2 |
| Fear & Greed | Index | Daily | Feb 2018+ | alternative.me (attribution) | P2 |
| Attention | Search interest | Daily/weekly | 5 years | Google Trends API alpha | P4 Could |
| News | Headline, URL, tags, timestamps | Event | Rolling | RSS, CryptoPanic (verify), exchange announcements | P2 |
| Social sentiment | Mentions, tone | Hourly | 1+ year | LunarCrush/Santiment (paid) | P4 Could |
| Macro | Dollar, yields, VIX, equities, gold | Daily | 10+ years, with vintages | FRED/ALFRED (free key) | P2 |
| ETF flows | Net flow per fund | Daily (T+1) | Jan 2024+ | Farside/SoSoValue → paid APIs | P2 Could |
| Event calendar | Events with `announced_at` | Event | Forward + 5 years back | federalreserve.gov, BLS, block height, Tokenomist, manual | P1/P2 |
| LLM digests | Prompts, outputs, model version | Event | Forever | LLM provider (docs 06/07) | P2 |

Cross-cutting needs for doc 06:
- the observation schema in §6;
- per-source license and attribution fields;
- per-series staleness thresholds;
- swappable source adapters.

---

## Open questions for the owner

1. **Which country are you in?** It determines which exchange APIs work for you. Binance.com blocks US IPs, and derivatives venues vary by country. It also determines which data sources you can legitimately use.
2. **Which trading horizon interests you?** Minutes, hours or days? I recommend 1h–1d bars for P1: costs are smaller relative to moves, and the evidence is better.
3. **What paid-data budget, if any?** For example, $0 now and $30–100/month in P2.
4. **Preferred alert channel?** P1 needs at least one outside the app: Telegram, email or ntfy. Web push arrives in P2.
5. **Is LLM news summarization worth an ongoing API cost in P2?**
6. **Could Ubik ever become commercial or multi-user?** If so, avoid non-commercial and personal-use-only data from the start.
7. **Do you want candlestick and chart-pattern tools for learning,** knowing they'll be labelled Weak?

## Risks

| Risk | Mitigation |
|---|---|
| Beginners read indicators as instructions ("oversold → buy") | Evidence badges, the four-question template, no composite score, banned-phrase checks |
| Look-ahead, repaint or full-sample-fitting bugs inflate backtests | Parity tests (TA-003), truncation harness (TA-028), point-in-time storage, confirmed-only repainting constructs |
| Data revisions and survivorship bias | Append-only storage, point-in-time universe, "revised data" badge |
| Short free history for derivatives, dominance and sentiment | Capture from P1 (MA-007, MA-011) |
| Free sources change terms, rate limits or geo-access | Source adapters, freshness checks, graceful degradation (MA-020/021) |
| License limits (CC BY-NC, personal-use plans) block a future SaaS | Record license per source; review before P4 (doc 08) |
| Mass testing in the scanner / Signal Lab "discovers" luck | Trial ledger (doc 04 BT-021, MA-023), deflated statistics, walk-forward (MA-022) |
| Edges from 2014–2021 fade after ETFs and institutional entry | Recency-weighted evaluation, paper incubation, registry reviewed each release |
| LLM hallucination or leakage | Cited digests, no predictions, cutoff rule, stored outputs |
| "Everything pro firms do" scope creep (indicator bloat, high-frequency microstructure) | Phased catalog; microstructure positioned as execution context, not alpha |
| Alert fatigue; order-book storage costs | Noise controls (SIG-005); per-minute depth bands; raw retention per 06 NFR-011, with P1 L2 recording capped at ≤ 5 instruments and shorter raw windows as a configurable disk-saving option |
