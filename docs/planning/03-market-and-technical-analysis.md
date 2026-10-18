# Market & Technical Analysis

*Ubik planning doc 03. Scope: technical analysis (TA) and charting, market analysis (MA), and the scanner, signals and alerts (SIG). Last checked against sources: 2026-10-18.*

## Summary

- **Build one indicator engine that everything else uses.** Charts, the scanner, alerts, strategies, backtests and paper and live trading all call the same code. It uses closed bars only, updates one bar at a time, and is tested so that its streaming results match a batch reference (TA-Lib) exactly. This is the main thing that makes "sim-live parity" possible, and it is the first thing to build.
- **Separate "describing the market" from "predicting it", and label every output with how strong the evidence is.** The research is fairly consistent on a few points. Volatility can be forecast; direction mostly cannot. Trend and time-series momentum have the best support for direction in crypto, but that support is fragile and fades over time. Candlestick and chart patterns have weak support. On-chain "cycle" metrics rest on only 3–4 market cycles. Every card, alert and scan result in Ubik carries an evidence badge (Descriptive / Supported / Mixed / Weak / Untested) and a one-line reason.
- **Keep P1 small and honest.** About 20 core indicators, a multi-pane chart with trade markers from backtests and paper trades, simple rule-based labels for regime and volatility, price/indicator/regime alerts, and a **Market Health Summary** built only from descriptive items (trend, volatility, liquidity, upcoming events). It has no "buy score".
- **Store all data "point-in-time" from day one.** Every value records when it happened (`event_time`) and when Ubik could first have known it (`available_at`). On-chain, sentiment, macro and ETF data get revised after the fact, so they are stored append-only. Backtests that use revised history are flagged as optimistic.
- **Start recording derivatives and market-wide data in P1, even though their analysis ships in P2.** Free history is short or missing. Binance's API, for example, only serves the last 30 days of open interest and long/short ratios, and CoinGecko's historical global market-cap chart is paid-only. Recording is cheap. History that was never recorded cannot be recovered.
- **Use free data first and flag the license terms.** Exchange APIs, Deribit (DVOL), alternative.me, DefiLlama, FRED/ALFRED and Coin Metrics Community cover most of P2. Several free sources are non-commercial or require attribution, which matters if Ubik ever goes multi-user (P4).

---

## 1. Principles and evidence base

### 1.1 Design principles

1. **Compute once, consume everywhere.** An indicator is a pure function of point-in-time inputs. It has the same code path in the chart, scanner, strategy, backtest and live engine.
2. **Closed bars by default.** A "bar" or "candle" is one time period's open/high/low/close/volume (OHLCV). Values from the bar that is still forming are drawn on the chart but are never passed to decisions unless a rule explicitly asks for intrabar data.
3. **Describe first, predict carefully.** Many market measures, such as volatility, spread and drawdown, are accurate *descriptions* of right now but weak *forecasts*. Ubik keeps those two uses apart.
4. **Show the evidence, not hype.** Every analysis says what it means, why it matters and how reliable it is.
5. **Free data first, swappable sources.** Each data series sits behind a source adapter (owned by doc 06), so moving to a paid feed is a configuration change.

### 1.2 Evidence labels (used throughout)

| Label | Meaning | Example |
|---|---|---|
| **Descriptive** | Measures the present accurately. Makes no claim about the future. | Spread, current realized volatility, BTC dominance |
| **Supported** | Several peer-reviewed studies, including out-of-sample tests (tested on data the model never saw) and tests after trading costs, found an effect in crypto. The effect may still fade. | Volatility persistence; trend/time-series momentum |
| **Mixed** | Some studies find an effect and others do not, or the samples are short or old. | Fear & Greed, funding as a directional signal, exchange flows |
| **Weak** | Rigorous tests mostly find no reliable edge. Shown for education and communication. | Candlestick patterns, Fibonacci levels |
| **Untested** | No evidence either way. This is the default for user-defined indicators. | Custom indicator |

The user's own Signal Lab results (MA-026) appear *next to* the label and never replace it.

### 1.3 What the research says (decision-relevant summary)

| Family | Evidence | Ubik label |
|---|---|---|
| **Volatility clustering / forecasting** | Volatility is persistent. HAR-type models (a regression on daily, weekly and monthly realized volatility) forecast Bitcoin realized volatility out-of-sample, and adding jump terms helps ([Bitcoin HAR studies](https://reading-clone.eprints-hosting.org/87932/1/R%26R_Forecasting%20the%20Volatility%20of%20Bitcoin_complete.pdf), [Bouri et al. 2021](https://repository.up.ac.za/handle/2263/76318)). | Supported for forecasting *size* of moves, not direction |
| **Trend / time-series momentum / price-vs-moving-average** | The ratio of price to its moving average predicted daily BTC returns in and out of sample ([Detzel et al. 2021, *Financial Management*](https://profiles.wustl.edu/en/publications/learning-and-predictability-via-technical-analysis-evidence-from-/)). Time-series momentum showed up at daily and weekly horizons ([Liu & Tsyvinski 2021, *RFS*](https://nber.org/papers/w24877)). Of ~15,000 technical rules tested, many were profitable, *but there was no out-of-sample predictability for Bitcoin* ([Hudson & Urquhart 2021](https://ideas.repec.org/a/spr/annopr/v297y2021i1d10.1007_s10479-019-03357-1.html)). Time-series momentum held up better than cross-sectional momentum after costs ([Han, Kang & Ryu](https://acfr.aut.ac.nz/__data/assets/pdf_file/0009/918729/Time_Series_and_Cross_Sectional_Momentum_in_the_Cryptocurrency_Market_with_IA.pdf)). One study found the effect weakened after Bitcoin futures launched ([seminar abstract](https://utilities.tinbergen.nl/?p=66235)). | Supported but fragile |
| **Oscillator thresholds (RSI < 30 "oversold")** | Included in large rule-sets, but individual thresholds are unstable and heavily data-mined. In strong trends, oversold markets can stay oversold. | Mixed→Weak |
| **Candlestick patterns** | A study of 68 patterns across 23 large coins found they were of little use and many had low accuracy ([IEEE 2021](https://ieeexplore.ieee.org/document/9671826)). Isolated exceptions, such as engulfing on BTC, were not consistent ([Emerald 2024](https://emerald.com/insight/content/doi/10.1108/IJQRM-12-2022-0363/full/html)). | Weak |
| **Chart patterns, Fibonacci, Elliott waves** | Little rigorous crypto evidence, and identifying them is subjective and done after the fact. | Weak |
| **Cross-sectional factors (market, size, momentum)** | Three factors explain the cross-section of crypto returns ([Liu, Tsyvinski & Wu 2022, *JF*](https://papers.ssrn.com/abstract=3379131)). Much of the effect is in small, illiquid coins, outside the P1 universe. | Supported (broad universe only) |
| **Carry / funding (as a return source)** | The average futures-spot carry was ~7% a year from 2019–2024 and sometimes exceeded 40% ([BIS WP 1087](https://www.bis.org/publ/work1087.htm)). Capturing it requires derivatives (P4). | Supported (existence); Mixed as a *directional* signal |
| **Attention (Google searches)** | Attention predicted returns 1–several weeks ahead in the 2011–2018 sample ([Liu & Tsyvinski 2021](https://www.cxoadvisory.com/currency-trading/crypto-asset-risks-and-returns/)). | Mixed (old sample) |
| **Fear & Greed index** | Short-sample studies find some predictive content 1 week to 1 month ahead ([Cavalheiro et al. 2024](https://ideas.repec.org/a/eme/rbfpps/rbf-08-2023-0224.html), [Charles Univ.](https://journal.fsv.cuni.cz/storage/1548_attachment.pdf)). About half the index is built from price volatility and momentum, so it partly double-counts TA. | Mixed |
| **On-chain valuation (MVRV, SOPR, NUPL)** | Mostly vendor claims calibrated on ~3–4 cycles, with thresholds that differ by source. Revisions inflate backtests: Glassnode's own example shows a signal performing worse on point-in-time data ([Glassnode PIT](https://research.glassnode.com/why-use-point-in-time-data/)). | Weak (few independent samples) |
| **Exchange flows** | One preprint finds intraday predictability, mostly for ETH ([arXiv 2411.06327](https://arxiv.org/pdf/2411.06327)). | Mixed |
| **Order-flow imbalance** | Order-flow imbalance explains price changes *over the same short interval* almost linearly ([Cont, Kukanov & Stoikov 2014](https://arxiv.org/pdf/1011.6402)). The horizon is seconds to minutes. | Descriptive; not actionable at Ubik's latency |
| **LLM news sentiment** | LLM-scored headlines predicted next-day stock returns, mostly through drift in small caps ([Lopez-Lira & Tang](https://arxiv.org/abs/2304.07619v5)). An LLM's training data can leak the future into backtests ([Glasserman & Lin 2023](https://arxiv.org/abs/2309.17322)). Evidence for crypto majors is thin. | Mixed; high leakage risk |
| **ML return prediction** | Even in large stock panels the best machine-learning models reach out-of-sample R² of a fraction of a percent per month ([Gu, Kelly & Xiu 2020](https://www.nber.org/papers/w25398.pdf)). Much of the crypto ML literature is in-sample ([IJIM-DI 2024](https://doaj.org/article/38d750a5f9fd4328924b125548276b93)). | Untested → treat as P4 research |
| **Volatility-managed scaling** | Strong in-sample. In equities it generally failed out-of-sample ([Cederburg et al. 2020, *JFE*](https://www.lehigh.edu/~xuy219/research/COWY.pdf)). | Use for risk control (doc 05), not as alpha |

**Meta-evidence.** Most published "discoveries" in finance are likely false positives. [Harvey, Liu & Zhu (2016)](https://papers.ssrn.com/abstract=2513152) argue for a t-statistic hurdle above 3, not 2. The more variants someone tries, the more likely the best one is luck; see the [Probability of Backtest Overfitting](https://papers.ssrn.com/abstract=2326253) and the [Deflated Sharpe Ratio](https://papers.ssrn.com/abstract=2460551). This is why the scanner and Signal Lab count how many variants the user has tried (MA-027).

---

## 2. Technical analysis

### 2.1 Indicator catalog

Bold = P1. Everything else is P2 unless noted.

| Family | Indicators | Plain-language one-liner |
|---|---|---|
| **Trend** | **SMA, EMA, MACD, ADX/±DI, Donchian channel, Supertrend**; WMA, Hull MA, KAMA, Ichimoku, Parabolic SAR, Aroon, linear-regression slope, Kaufman efficiency ratio | "Is price heading up, down or sideways, and how strongly?" |
| **Momentum** | **RSI, Stochastic, Rate of Change (n-bar return)**; Stoch RSI, Williams %R, CCI, TSI, Awesome Oscillator | "How fast price has moved. Stretched readings sometimes snap back, but in strong trends they can stay stretched." |
| **Volatility** | **ATR, Bollinger Bands (+%B, bandwidth), realized volatility (annualized), volatility percentile**; Keltner channels, Parkinson / Garman-Klass / Yang-Zhang estimators, BB-inside-Keltner "squeeze", volatility cones | "How big the typical swings are right now compared with normal." |
| **Volume** | **Volume + volume MA, relative volume (RVOL), OBV, VWAP (UTC-day)**; anchored VWAP, MFI, Chaikin Money Flow, A/D line, cumulative volume delta (CVD), volume profile (POC, value area) | "Is real trading activity behind this move?" |
| **Support / resistance** | **Prior day/week high-low-close, classic pivot points, confirmed swing highs/lows**; auto S/R zones (clustered swing points), round-number levels | "Price areas where buying or selling concentrated before. They are zones, not walls." |
| **Market structure** | **Trend-state label (higher highs/lows), regime label (MA-001)**; break of structure, confirmed ZigZag, Choppiness index | "Is the market making higher highs and higher lows, or the reverse?" |

Jargon, explained once:
- **SMA/EMA**: simple/exponential moving average.
- **MACD**: the difference between two EMAs.
- **ADX**: trend strength on a 0–100 scale.
- **ATR**: average true range, the typical bar-to-bar range.
- **VWAP**: volume-weighted average price.
- **OBV**: on-balance volume, a running total of volume signed by up or down bars.
- **POC**: point of control, the price with the most traded volume.
- **Value area**: the price band containing ~70% of volume.

**Candlestick patterns (P2, labelled Weak):** engulfing, hammer/shooting star (pin bar), doji, inside bar, morning/evening star. They are detected on closed bars only and are offered mainly for education and as optional filters.

**Chart patterns (P3, Could, labelled Weak):** double top/bottom, triangles, flags, head-and-shoulders. These are detected only once the pattern is *confirmed* (the breakout bar has closed). Each detection is stamped with the bar on which it would have been knowable, because pattern detectors notoriously "repaint" (they redraw history once later bars arrive).

**Multi-timeframe analysis (P1).** Supported timeframes are 1m, 5m, 15m, 1h, 4h, 1d and 1w. Every timeframe is built from the 1-minute base series with UTC boundaries, so it matches across exchanges and between backtest and live trading. A 1h strategy can read the 1d trend, but it sees the *last closed* daily bar, which steps once per day. A higher-timeframe value is never a still-forming bar.

**Library approach (reference only; doc 02 owns build-vs-reuse):**
- Use [TA-Lib](https://pypi.org/project/TA_Lib/) (BSD license) as the batch reference implementation. Since 0.6.5 its wheels bundle the C library, and it supports NumPy 2.
- For streaming, use an incremental engine: either our own, or modelled on [talipp](https://github.com/nardew/talipp), which is still young and should be evaluated first.
- Avoid pandas-ta as a core dependency. Its maintainer has said there will be no further releases without sponsorship ([sponsors page](https://sponsors.ecosyste.ms/accounts/twopirllc)).
- For charting, use [TradingView Lightweight Charts v5](https://www.tradingview.com/blog/en/tradingview-lightweight-charts-version-5-50837/): Apache-2.0, with multi-pane layouts and plugin "primitives" for drawings. Confirm the attribution requirement in its LICENSE/NOTICE. Drawing tools are *not* built in and must be implemented as plugins.

### 2.2 Charting and drawing tools

The chart is the main workspace. It must show what the strategy actually saw and did.
- **Trade markers:** from backtests and paper trades, with the rule that fired.
- **Equity pane:** synced to price.
- **Bar replay mode:** steps through history without showing the future. This is an excellent beginner learning tool and is look-ahead-free by design.

**Drawings and backtests.** A line drawn *today* over old candles contains hindsight. Drawings are therefore usable in alerts, but in backtests they apply only from their creation timestamp onward.

### 2.3 TA requirements

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| TA-001 | Ubik shall implement every indicator once, in a single indicator library used by the chart, scanner, alerts, strategies, backtester, paper and live engines. | Must | P1 | Sim-live parity; prevents "chart says X, bot did Y". |
| TA-002 | Ubik shall pass only closed-bar indicator values to strategies, alerts and scans by default. Forming-bar values are drawn visually distinct ("forming") on the chart. | Must | P1 | Most common source of look-ahead and repaint bugs. |
| TA-003 | For every indicator, streaming (bar-by-bar) and batch computation shall produce identical outputs (relative tolerance ≤1e-9) on a shared test dataset. Batch outputs shall match TA-Lib where TA-Lib has the indicator. | Must | P1 | Automated parity test suite in CI. |
| TA-004 | Each indicator shall declare metadata: inputs, parameters (defaults, valid ranges), warm-up length, outputs, repaint flag, evidence label, plain-language description and glossary link. | Must | P1 | Drives the UI, validation and beginner explanations. |
| TA-005 | Ubik shall provide the P1 indicator set in §2.1 (bold items). | Must | P1 | Covers trend, momentum, volatility, volume and S/R basics. |
| TA-006 | Ubik shall provide the P2 indicator set in §2.1, including volume profile and CVD. | Should | P2 | CVD needs trade (tick) data. |
| TA-007 | Indicators shall output null (not 0) until their warm-up period is met. Backtests shall not act on an indicator before warm-up and shall load enough pre-start history automatically. | Must | P1 | Avoids fake early signals. |
| TA-008 | Ubik shall support 1m, 5m, 15m, 1h, 4h, 1d and 1w bars, all built from a stored 1-minute base with UTC boundaries. | Must | P1 | Native exchange candles are kept only to cross-check the aggregation. |
| TA-009 | A strategy, scan or alert shall be able to reference a higher timeframe. The higher-timeframe value shall update only when that higher-timeframe bar closes. | Must | P1 | Multi-timeframe without look-ahead. |
| TA-010 | Repainting constructs (swing points, fractals, ZigZag, pivots, chart patterns, Ichimoku lagging span) shall be exposed to strategies only in their "confirmed" form, with the confirmation lag stated. | Must | P1 | Swing highs/lows are P1. Others follow when added. |
| TA-011 | Ubik shall detect the listed candlestick patterns on closed bars and label them "Weak evidence". | Could | P2 | Education value; optional strategy filter. |
| TA-012 | Ubik shall detect confirmed chart patterns, stamped with the bar at which each became knowable. | Could | P3 | Low evidence, high complexity. |
| TA-013 | The chart shall offer candlestick, OHLC-bar, line and area styles, with linear and log price scales. | Must | P1 | Log scale matters for long BTC histories. |
| TA-014 | The chart shall offer Heikin-Ashi candles with a persistent note that these are synthetic prices and that fills never happen at them. | Should | P2 | Heikin-Ashi = smoothed candles. A common beginner backtest trap. |
| TA-015 | The chart shall support price overlays and at least 3 indicator sub-panes, a synchronized crosshair, and a data window showing all values at the cursor. | Must | P1 | |
| TA-016 | The chart shall plot backtest and paper/live trades as entry/exit markers. A tooltip shall show time, side, price, size, fees, P&L and the rule that fired. | Must | P1 | Core to understanding strategy behavior. |
| TA-017 | The chart shall draw open position, stop-loss, take-profit and resting-order lines. | Must | P1 | Coordinate with doc 05. |
| TA-018 | The backtest view shall show equity and drawdown panes time-synced to the price chart. | Should | P1 | Coordinate with doc 04. |
| TA-019 | The chart shall visibly mark data gaps and exchange outages, and shall never interpolate OHLC values. | Must | P1 | Honesty; also flags bad backtest periods. |
| TA-020 | The chart shall update the forming candle within 1 s of receiving trade/ticker data, and shall pan and zoom 20,000 bars at ≥30 fps on a mid-range laptop. | Should | P1 | Performance budget. |
| TA-021 | The chart shall show times in the user's local time zone (configurable) while all computation uses UTC. Bar tooltips shall show open and close time. | Should | P1 | Avoids "daily candle" confusion. |
| TA-022 | Ubik shall offer a bar-replay mode that hides future bars and lets the user step forward or play. | Should | P2 | Strong learning tool. |
| TA-023 | Drawing tools (P1): horizontal line, trend line, rectangle zone, Fibonacci retracement (labelled Weak), text note. | Should | P1 | |
| TA-024 | Drawing tools (P2): ray, parallel channel, measure (price/%/bars/time), long/short risk-reward tool, anchored VWAP. | Should | P2 | |
| TA-025 | Drawings shall persist per user and symbol, carry a creation timestamp, be usable as alert triggers, and in backtests apply only from creation time forward. | Should | P2 | Prevents hindsight drawings in tests. |
| TA-026 | Users shall be able to save and load chart layouts and indicator templates. | Should | P1 | |
| TA-027 | The chart shall be usable at 375 px width with touch pan/zoom and up to 2 sub-panes. | Should | P1 | Mobile usability (doc 07). |
| TA-028 | Users shall be able to define custom indicators, via the strategy rule language and later Python plugins, that work in every consumer listed in TA-001. They are labelled "Untested" until evaluated. | Should | P2 | Coordinate with doc 04's strategy builder. |
| TA-029 | The chart shall offer a compare mode (normalized overlay of several assets). | Could | P2 | |

---

## 3. Market analysis

### 3.1 Regime detection

A **regime** is a persistent market state, such as trending vs ranging or calm vs stormy.

**P1 uses transparent rules:**
- **Trend state** (Up / Down / Sideways), from:
  - price vs the 200-day SMA,
  - the slope of the 50-day SMA,
  - ADX above or below ~20–25.
- **Volatility state** (Calm / Normal / Stormy): 30-day realized volatility as a percentile of its trailing 1-year distribution, split at the 33rd and 67th percentiles.
- **Hysteresis.** A state must persist N bars before the label flips, so labels don't flicker.

**P2 adds statistical methods:**
- **Hidden Markov models (HMMs):** a model that infers hidden states from returns and volatility.
- **Change-point detection:** finds dates where the statistical behavior shifts.
- **Variance-ratio / Hurst tests:** measure whether a series trends or mean-reverts.
- **Rules for these methods:** They must be fitted on expanding windows only. HMMs fitted on the full history "know" where future regimes lie. Each model reports how often its labels flip.

Regime labels are *Descriptive*. Using them as strategy filters is common and sensible, but it is still a hypothesis that must be tested.

### 3.2 Volatility

**Realized volatility** is how much price actually moved. Planned estimators:
- **P1:** close-to-close.
- **P2:** range-based Parkinson, Garman-Klass and Yang-Zhang, plus intraday 5-minute realized volatility.
- **Convention:** annualize with 365 days, because crypto trades every day.

**Implied volatility** is the market's expectation, read from option prices. Deribit's **DVOL** (a 30-day implied-volatility index for BTC and ETH, similar to the VIX) is free through the public `get_volatility_index_data` endpoint ([Deribit docs](https://docs.deribit.com/api-reference/market-data/public-get_volatility_index_data.md)). Deribit now sits under Coinbase, whose developer docs mirror the API ([Coinbase CDP](https://docs.cdp.coinbase.com/api-reference/market-data/public-get_volatility_index_data.md)).

**Volatility risk premium (VRP)** = implied minus realized volatility.

**Forecasts** (P2): HAR-RV and GARCH models, which use recent volatility to forecast future volatility.

**Uses:**
- position sizing and stop distances (doc 05);
- a beginner-friendly **"typical daily move"** ("BTC currently moves about ±2.1% on a typical day");
- regime labels.

### 3.3 Correlation and cointegration

**Correlation** measures whether assets move together. Planned outputs:
- rolling correlation matrices (30/90-day on daily returns; 7-day on hourly);
- beta of altcoins to BTC;
- correlation of BTC to equities and the dollar (P2).

**Show beginners that correlations jump toward 1 in sell-offs**, so "diversifying" across BTC/ETH/SOL protects little.

**Cointegration** means two prices that wander but stay tied together; it is the basis of pairs trading. Tools (P2): Engle-Granger and Johansen tests, and the half-life of mean reversion. Every result is reported with its rolling stability, because crypto pairs relationships change across sub-periods ([Fil & Kristoufek 2020](https://dspace.cuni.cz/handle/20.500.11956/197060)). Pairs *trading* needs short selling, so it is P4 (derivatives). Spot-only P1 can only use this for analysis.

### 3.4 Dominance and breadth

Planned measures:
- **Dominance:** BTC dominance (BTC market cap ÷ total crypto market cap), the ETH/BTC ratio, and stablecoin dominance.
- **Breadth** (how many coins join a move):
  - % of the top-N coins above their 50/200-day MA,
  - advancers vs decliners,
  - new 30-day highs vs lows.

CoinGecko's free `/global` endpoint gives *current* dominance. The historical `/global/market_cap_chart` endpoint is paid-only ([CoinGecko docs](https://docs.coingecko.com/reference/global-market-cap-chart.md)). **Ubik should snapshot dominance daily from P1.**

Breadth needs a *data* universe (top 50–100 coins) that is wider than the *traded* universe. It also needs **point-in-time universe membership** to avoid survivorship bias (testing only on coins that still exist today).

### 3.5 Liquidity and microstructure

Planned measures:
- **Order-book measures:**
  - bid-ask spread (in basis points; 1 bp = 0.01%),
  - depth within ±0.5/1/2% of mid,
  - estimated slippage for the user's order size, found by walking the book.
- **Order-flow measures:**
  - order-book imbalance,
  - trade-sign imbalance and CVD,
  - order-flow imbalance (OFI).
- **Volume and illiquidity:**
  - volume profile,
  - daily Amihud illiquidity (price impact per dollar traded).

**Recommendation:** in Ubik these serve **execution and risk**, as inputs to doc 05's slippage model and to the scanner's liquidity filters. They are not alpha. OFI explains price over *seconds to minutes* ([Cont et al.](https://arxiv.org/pdf/1011.6402)). A self-hosted bot on a home connection that pays taker fees cannot compete at that horizon.

**Storage:** full L2 order-book diffs are heavy. Store aggregated depth bands per minute plus short rolling raw windows (doc 06).

### 3.6 Derivatives data

Derivatives data matters even for spot traders, because perpetual futures ("perps": futures with no expiry, kept near spot by periodic **funding** payments between longs and shorts) dominate crypto price discovery.

| Metric | What it means | Caveats |
|---|---|---|
| Funding rate (per exchange, OI-weighted, annualized) | Positive = longs pay shorts: crowd is leveraged long | Intervals differ (1h/4h/8h) by venue and symbol; store both *predicted* and *settled* rates |
| Open interest (OI) | Total open contracts; rising OI + rising price = new leveraged longs | Binance API serves only the last 30 days ([Binance docs](https://developers.binance.com/docs/derivatives/coin-margined-futures/market-data/Top-Trader-Long-Short-Ratio)); daily archives exist on data.binance.vision ([issue](https://github.com/binance/binance-public-data/issues/211)) |
| Basis | Futures premium over spot, annualized | Quarterly futures roll; carry ~7%/yr average ([BIS](https://www.bis.org/publ/work1087.htm)) |
| Liquidations | Forced closures; cascades amplify moves | Binance's stream pushes at most one liquidation per symbol per second, so it **undercounts** ([Binance docs](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Liquidation-Order-Streams)). Use it as a relative gauge only |
| Long/short ratios | Share of *accounts* (or top traders) net long | Definitions vary by exchange; every contract has a long and a short, so this is about who holds them, not net demand |
| Options: DVOL, 25-delta skew, put/call OI (P3) | Fear of downside vs upside | "Max pain" is folklore (Weak) |

Evidence: carry exists. Funding as a *directional* contrarian signal is practitioner lore with conflicting reports (Mixed). Ubik presents funding and OI mainly as **leverage and crowding risk gauges**.

Free aggregated history is available from [Coinalyze](https://dlthub.com/context/source/coinalyze) (free key, ~40 calls/min). [CoinGlass](https://coinglass.com/pricing) has no free API; it starts at $29/month for personal use.

**Jurisdiction flag:** Binance.com's API refuses requests from US IP addresses (HTTP 451, "restricted location"). Several derivatives venues also restrict some countries. Source choice depends on the owner's country (docs 06/08). Ubik must not suggest bypassing geo-blocks.

### 3.7 On-chain metrics (P2, BTC and ETH first)

| Metric | Meaning | Source (free → paid) |
|---|---|---|
| Hash rate, transaction count, active addresses | Network usage and security | Coin Metrics Community, mempool.space |
| MVRV (market value ÷ realized value) | How far price sits above holders' average cost basis | Coin Metrics Community may expose it (verify); Glassnode/CryptoQuant paid |
| Stablecoin supply | "Dry powder" that could enter crypto | [DefiLlama](https://stablecoins.llama.fi/stablecoins) (free) |
| Exchange net flows / reserves | Coins moving to exchanges (possible selling) or away | Paid (Glassnode API is Professional-tier; CryptoQuant credits). Depends on address-labelling heuristics, which get revised |
| SOPR, NUPL, miner flows | Profit-taking / holder behavior | Paid |

Coin Metrics Community is free with no key needed (10 requests per 6 s). Its license is **CC BY-NC 4.0 (non-commercial)** ([docs](https://docs.coinmetrics.io/packages/coin-metrics-community-data)). That is fine for personal use and a blocker for a commercial SaaS.

On-chain data must be stored point-in-time (§6). Backtests over vendor history recorded before Ubik started capturing are flagged "revised data — likely optimistic".

### 3.8 Sentiment and news (P2)

**Fear & Greed (alternative.me).**
- **Components:** volatility 25%, momentum/volume 25%, social 15%, surveys 15% (paused), dominance 10%, trends 10%.
- **API:** free daily API, which requires prominent attribution next to the data ([alternative.me](https://alternative.me/crypto/fear-and-greed-index/)).
- **How Ubik presents it:** as a summary of crowd mood, not an independent signal. It overlaps with price-based TA.

**Attention (Google Trends).**
- The official API is still an application-only alpha ([Google](https://developers.google.com/search/blog/2025/07/trends-api)).
- Website values are rescaled on every query, so history must be captured point-in-time.
- Could, P3.

**Social (X/Reddit)** is expensive or restricted. It is Could, P3, via a paid aggregator.

**News pipeline:**
1. **Sources:**
   - RSS feeds from major outlets;
   - an aggregator (CryptoPanic — verify current terms);
   - **exchange announcements**: listings, delistings, maintenance. These are the most practically important for a trading bot.
2. **Storage:** headlines, URLs and timestamps only, not full text (copyright, doc 08).
3. **Processing:** deduplicate; tag by asset and event type (hack, ETF, regulation, listing, macro).
4. **LLM summary:** an LLM (large language model) writes a short digest that cites its sources.

**LLM rules:**
- Summaries never state price predictions.
- Model name and version are logged.
- LLM-derived features enter backtests only for dates *after* the model's training cutoff, or using anonymized headlines. Otherwise the model may "remember" what happened next ([Glasserman & Lin](https://arxiv.org/abs/2309.17322)).

### 3.9 Macro context and event calendar

**Macro series (P2), daily:**

| Series | Source / note |
|---|---|
| US dollar | FRED broad trade-weighted index `DTWEXBGS`, a free proxy (ICE's DXY itself is licensed) |
| Rates | 2y/10y Treasury yields (`DGS2`, `DGS10`), fed funds rate |
| Equities and fear | S&P 500 / Nasdaq (FRED's licensed history is limited), VIX (`VIXCLS`) |
| Other | Gold; US spot BTC/ETH ETF net flows (Farside, SoSoValue; usually complete only the next morning, so treat as T+1) |

**ALFRED** (FRED's archive of past data vintages) provides point-in-time macro values. Crypto trades 24/7 and macro markets don't, so weekend values are carried forward with a staleness flag, never invented.

**Event calendar** (P1 basic, P2 full). Each event stores `scheduled_at`, `announced_at`, importance, source and affected assets.
- **Basic set:**
  - **FOMC:** the [Fed calendar](https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm) lists 2027 meetings ending Jan 27, Mar 17, Apr 28, Jun 9, Jul 28, Sep 15, Oct 27 and Dec 8. The statement comes at 2 pm ET on day 2.
  - **US CPI and payrolls:** BLS schedules.
  - **Bitcoin halving:** next at block 1,050,000, about April 2028. Compute it live from block height.
- **Full set:**
  - token unlocks ([Tokenomist](https://docs.tokenomist.ai/tier-overviews): free web, API paid/trial);
  - network upgrades (curated);
  - exchange maintenance and delistings;
  - Deribit/CME expiries (rule-based).

**Uses:**
- alerts;
- doc 05's optional "reduce size before high-impact events" guardrail;
- **event studies**: how BTC behaved around the last N FOMC meetings, with sample size and uncertainty.

### 3.10 MA requirements

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| MA-001 | Ubik shall compute rule-based trend state (Up/Down/Sideways) and volatility state (Calm/Normal/Stormy) per asset on 4h and 1d, with configurable hysteresis. Both shall be usable as strategy conditions. | Must | P1 | Transparent, testable. |
| MA-002 | Ubik shall offer statistical regime models (HMM, change-point), fitted on expanding windows only, reporting label-flip frequency. | Should | P2 | Prevents full-sample look-ahead. |
| MA-003 | Ubik shall compute close-to-close realized volatility (annualized, 365-day) and its trailing 1-year percentile. | Must | P1 | Feeds sizing (doc 05) and labels. |
| MA-004 | Ubik shall show a plain-language "typical daily move" range derived from recent volatility. | Should | P1 | Beginner intuition for risk. |
| MA-005 | Ubik shall provide range-based volatility estimators and HAR-RV/GARCH volatility forecasts, with the forecasts' own out-of-sample error displayed. | Should | P2 | |
| MA-006 | Ubik shall ingest DVOL (BTC, ETH) and display implied vs realized volatility and the VRP. | Should | P2 | Free Deribit API. |
| MA-007 | Ubik shall compute rolling correlations among the traded pairs (30/90-day, daily returns). | Should | P1 | Needed by doc 05 concentration risk. |
| MA-008 | Ubik shall compute beta-to-BTC, correlation to macro series, and cointegration tests (Engle-Granger, Johansen, half-life) with rolling-stability display. | Could | P2 | |
| MA-009 | Ubik shall snapshot BTC dominance, total market cap and stablecoin market cap daily starting in P1, and chart them with ETH/BTC in P2. | Should | P1/P2 | Free history unavailable. |
| MA-010 | Ubik shall compute market breadth over a configurable top-N data universe using point-in-time universe membership. | Should | P2 | Avoids survivorship bias. |
| MA-011 | Ubik shall monitor spread and top-of-book depth for each traded pair and flag when either deteriorates past thresholds. | Should | P1 | Execution risk; scanner filter. |
| MA-012 | Ubik shall compute depth bands (±0.5/1/2%) and an order-size slippage estimate from order-book snapshots. | Should | P2 | Shared with doc 05 TCA. |
| MA-013 | Ubik shall compute CVD, trade imbalance and OFI, labelled "Descriptive / short-horizon". | Could | P2 | |
| MA-014 | Ubik shall begin recording funding rates (predicted and settled), open interest and long/short ratios for BTC/ETH/SOL perps on at least one venue in P1. | Should | P1 | Free API history is 30 days for some series. |
| MA-015 | Ubik shall display funding (per venue, OI-weighted, annualized), OI, OI change, basis and liquidations, each with its caveat text (e.g., "liquidation feed undercounts"). | Should | P2 | |
| MA-016 | Ubik shall display options data (DVOL term structure, 25-delta skew, put/call OI). | Could | P3 | |
| MA-017 | Ubik shall ingest the free on-chain set in §3.7 for BTC and ETH, recording each source's license and attribution terms. | Should | P2 | CC BY-NC terms for Coin Metrics. |
| MA-018 | Paid on-chain metrics (exchange flows, SOPR, NUPL) shall be pluggable via configuration. When available, the point-in-time variants shall be preferred. | Could | P2 | Glassnode PIT exists on paid tiers. |
| MA-019 | Ubik shall ingest Fear & Greed daily, with the attribution displayed next to the value. | Should | P2 | alternative.me terms. |
| MA-020 | Ubik shall ingest news headlines and exchange announcements, deduplicate them, tag asset and event type, and store `published_at` and `ingested_at`. | Should | P2 | |
| MA-021 | Ubik shall produce LLM news digests that cite source links. Digests shall contain no price predictions, and the model name/version shall be logged with each digest. | Should | P2 | Coordinate with doc 07's AI assistant. |
| MA-022 | LLM-derived features shall be usable in backtests only for dates after the model's training cutoff or with anonymized inputs. Otherwise the backtest shall be flagged. | Must | P2 | LLM look-ahead leakage. |
| MA-023 | Ubik shall ingest the macro series in §3.9 with vintage-aware storage. Weekend/holiday values are carried forward with a staleness flag. | Should | P2 | FRED/ALFRED. |
| MA-024 | Ubik shall maintain an event calendar with `scheduled_at`, `announced_at`, importance, source and affected assets. P1 covers FOMC, CPI, payrolls, halving and manual events. P2 adds unlocks, upgrades, exchange notices and expiries. | Should | P1/P2 | |
| MA-025 | Ubik shall provide an event-study view (asset behavior around past events of a type), always showing sample size and confidence intervals. | Could | P2 | |
| MA-026 | Ubik shall provide a **Signal Lab**. It shall evaluate any signal's forward returns at 1h/4h/1d/7d horizons, including: hit rate, mean, information coefficient (rank correlation with forward returns), bootstrap confidence intervals, net-of-cost results, and walk-forward splits. | Should | P2 | Research before strategy. Doc 04 owns strategy metrics. |
| MA-027 | Ubik shall keep a multiple-testing ledger counting every signal/parameter variant a user evaluates. It shall show the count and apply deflated statistics to the best result. | Should | P2 | Harvey-Liu-Zhu; DSR/PBO (doc 04). |
| MA-028 | Ubik shall provide statistical tests: ADF and KPSS (stationarity), variance ratio, Ljung-Box (autocorrelation), and Granger causality labelled "association, not cause". | Could | P2 | statsmodels / arch. |
| MA-029 | Every analysis output shall carry its source, `as_of` time and freshness. Values staler than a per-series threshold shall be greyed out and excluded from signals. | Must | P1 | Data-quality honesty. |
| MA-030 | Analyses shall degrade gracefully when an optional source is unavailable (geo-blocked, rate-limited, discontinued): the card says why, and no stale values are shown as live. | Must | P1 | Free sources change often. |

---

## 4. Quant, statistical and ML analysis

**Feature engineering principles** (a "feature" is any number fed into a model or rule):
- Use returns, not raw price levels (prices are non-stationary: their statistics drift over time).
- Normalize by volatility: return ÷ ATR, or distance from the MA in σ units.
- Use *rolling* z-scores and percentile ranks, never statistics fitted on the full sample.
- Use cross-sectional ranks across the universe.
- Add calendar features: hour, weekday, time-to-funding.
- Use regime labels as conditioning variables.
- Version every feature and compute it with the TA-001 engine. A P2 feature store makes features reusable across Signal Lab, strategies and later ML.

**Factor ideas for crypto:**

| Factor | Ubik use | Evidence | Phase |
|---|---|---|---|
| Time-series momentum / trend | Strategy templates (doc 04), regime filter | Supported, fragile | P1 |
| Cross-sectional momentum (rotate into strongest coins) | Scanner rank, long-only rotation | Supported in research; costs bite | P2 |
| Volatility (low-vol, vol-targeting) | Sizing (doc 05) | Risk-useful; alpha Mixed | P1/P2 |
| Liquidity (Amihud, volume shocks) | Universe filter; scanner | Mixed | P2 |
| Carry (funding / basis) | Positioning gauge now; trade later | Supported existence | P2 view / P4 trade |
| Value-like on-chain (MVRV, NVT) | Context card only | Weak | P2 |
| Attention / sentiment | Context; Signal Lab | Mixed | P2–P3 |
| Seasonality (weekday, hour) | Signal Lab only | Weak, unstable | P2 |

**Where ML overfits** (ML is mostly P4):
- **Too little signal.** Return predictability is tiny.
- **Too few independent samples.** BTC has one price path and ~4 halving cycles.
- **Non-stationarity.** Structure changed with futures in 2017, ETFs in 2024 and institutional flows.
- **Leakage.** Overlapping labels, full-sample normalization, random k-fold cross-validation on time series, and survivorship-biased universes.
- **Hyperparameter search is multiple testing in disguise.**

**Guardrails when ML arrives:**
- purged, embargoed walk-forward cross-validation (gaps between training and test data so overlapping labels can't leak);
- a locked final holdout period;
- mandatory comparison against simple baselines (buy-and-hold, MA crossover, random entries);
- PBO/DSR reporting;
- checks on feature-importance stability;
- a paper-trading incubation period before any live use.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| MA-031 | Ubik shall provide a versioned, point-in-time feature store, computed with the shared indicator engine. | Should | P2 | Basis for Signal Lab and ML. |
| MA-032 | Ubik shall compute cross-sectional factor scores (momentum, volatility, liquidity) over the data universe. | Could | P2 | Scanner ranks. |
| MA-033 | ML models shall be disabled by default. When enabled, they shall require purged/embargoed walk-forward CV, a locked holdout, baseline comparison and PBO reporting before they can drive paper trading. | Should | P4 | Overfitting control. |

---

## 5. Scanner, signals and alerts

**Signal object.** Every signal records:
- an expression in the **same condition language as the strategy builder** (doc 04);
- its timeframe;
- `evaluated_at` (the bar close);
- the inputs it used and their `available_at` times;
- its evidence label;
- an explanation template.

This makes "scan → alert → backtest → strategy" one click in each direction.

**Scanner (P2).**
- **Universe filters:** exchange, quote currency, minimum 24h volume, maximum spread, minimum history length.
- **Conditions and sorting:** any condition, on any timeframe, with sortable columns.
- **Beginner presets,** each with an explanation and evidence badge:
  - "Strong uptrend"
  - "Pullback in uptrend"
  - "Unusual volume"
  - "Volatility squeeze"
  - "Funding extreme"
  - "Breaking 30-day high"
- **Timing:** scans run at bar close. Results are timestamped snapshots, so "what did the scanner show last Tuesday" is answerable.

In P1, with only a handful of pairs, a watchlist table with indicator columns is enough.

**Alerts.**
- **P1 types:** price level crossed, % move within a window, indicator condition, regime change, unusual volume, volatility spike, data/system health.
- **P2 types:** drawing crossed, funding/OI extremes, liquidity deterioration, correlation break, event reminders, asset news.
- **Unusual-volume baseline:** RVOL is measured against a time-of-day baseline, because crypto volume has strong intraday seasonality.
- **Anti-noise controls:** cooldowns, hysteresis/re-arm, once / once-per-bar / every-time modes, expiry, per-channel rate limits, quiet hours, deduplication.

**Delivery channels:**

| Phase | Channels |
|---|---|
| P1 | In-app; email; Telegram bot (the de-facto crypto channel) |
| P2 | Browser/PWA push; Discord; generic webhook |
| P3 | SMS (Could) |

Channel credentials are handled per doc 08.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SIG-001 | Ubik shall represent every signal as a typed object (expression, timeframe, `evaluated_at`, inputs with `available_at`, evidence label, explanation). | Must | P1 | Auditability. |
| SIG-002 | Scan and alert conditions shall use the strategy builder's condition language. Any condition shall be convertible into a backtestable strategy and vice versa. | Must | P1 | Coordinate with doc 04. |
| SIG-003 | Ubik shall support the P1 alert types listed above. | Must | P1 | |
| SIG-004 | Ubik shall support the P2 alert types listed above. | Should | P2 | |
| SIG-005 | Alerts shall evaluate on bar close by default. Only price-level alerts may evaluate intrabar, and they shall be marked "intrabar". Each firing shall record its evaluation time and input values. | Must | P1 | Same timing semantics as backtests. |
| SIG-006 | Alerts shall support cooldown, re-arm/hysteresis, trigger mode (once / once per bar / every time), expiry, quiet hours, per-channel rate limits and deduplication. | Must | P1 | Prevents alert fatigue. |
| SIG-007 | Ubik shall deliver alerts in-app (Must), by email and Telegram (Should) in P1, and by web push, Discord and webhook in P2. | Must/Should | P1/P2 | |
| SIG-008 | Alert delivery shall be at-least-once with a dedupe ID. Failures shall be retried and visible. Median delivery shall be ≤10 s after bar close. | Should | P1 | |
| SIG-009 | Alert messages shall state what happened, the value, a plain-language meaning, the evidence label and a chart link. They shall never use imperative trade language ("buy now"). | Must | P1 | Honesty principle. |
| SIG-010 | Users shall be able to "backtest an alert": see how often it would have fired over a chosen period and the forward-return distribution afterwards. | Should | P2 | Uses Signal Lab (MA-026). |
| SIG-011 | Ubik shall provide a watchlist with configurable indicator columns (P1) and a full scanner with universe filters, conditions, sorting and presets (P2). | Should | P1/P2 | |
| SIG-012 | Scans shall run at bar close and store timestamped result snapshots. | Should | P2 | |
| SIG-013 | Ubik shall raise data-health alerts for stale feeds, exchange disconnects and data gaps. | Must | P1 | Paper and live safety. |

---

## 6. Testability and data-timing rules

Every indicator and signal must be usable in strategies and backtests **without look-ahead** (using information that was not yet available at decision time). Ubik's timing model has these fields:

```
observation(series_id, asset, period_start, period_end, event_time,
            value, available_at, first_seen_at, source, revision_no)
```

- `event_time` / `period_end`: when the fact happened.
- `available_at`: the earliest time a live system could have known it. This is the exchange or publisher time plus a documented lag.
- `first_seen_at`: when Ubik itself first stored it.

**Rules:**

1. **Decision clock.** A decision made at time *t* may use only observations with `available_at ≤ t`. For bar data, *t* is the bar's close. Fills happen no earlier than the next tradable price (fill model owned by doc 04).
2. **Closed bars only.** The forming bar is never an input unless a rule is explicitly intrabar. Then the backtester must use sub-bar data (1m or trades), not the bar's final OHLC.
3. **Higher timeframes** step at their own close (TA-009).
4. **Point-in-time storage.** Revisable series are append-only: on-chain, sentiment, macro, ETF flows, market cap/supply, Google Trends, universe membership. A revision becomes a new row and never overwrites the old one. Backtests read "as known at *t*".
5. **Pre-capture history.** History bought or backfilled before Ubik began capturing gets a conservative `available_at` (table below). Backtests that use it show a "revised data — results likely optimistic" badge.
6. **No full-sample fitting.** Anything fitted (z-score windows, HMMs, thresholds, percentiles, ML) uses only past data: expanding or rolling windows.
7. **Repaint-free.** Repainting constructs are exposed only in confirmed form with an explicit lag (TA-010).
8. **Warm-up.** No signal fires before warm-up (TA-007).
9. **Universe as of date.** Scans and backtests over many coins use only the coins listed and liquid at that date, including those later delisted.
10. **Gaps are gaps.** Missing data is never interpolated. Forward-fill is allowed only within a per-series staleness limit, and filled values are flagged.
11. **News timestamps.** Use `ingested_at` (or `published_at` only if Ubik captured the item live), because publishers edit and backdate.
12. **LLM leakage.** Follow MA-022.
13. **Determinism.** Backtests are reproducible from stored inputs. Live LLM outputs are stored, not regenerated.

**Default availability lags for backfilled history (P2 values to confirm in doc 06):**

| Data | Conservative `available_at` |
|---|---|
| Exchange OHLCV | Bar close + 2 s |
| Settled funding | Funding timestamp; predicted funding is a separate live series |
| OI, long/short ratios | Period end + 1 min |
| DVOL | Candle close |
| Fear & Greed | End of the labelled UTC day + 1 h |
| Daily on-chain | Day end + 24 h (vendor-specific) |
| DefiLlama stablecoin supply | Day end + 24 h |
| FRED/ALFRED macro | Vintage date |
| ETF flows | Next US business day, 12:00 UTC |
| Event calendar | `announced_at` (emergency events have no advance notice) |

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| TA-030 | All stored series shall carry `event_time`, `available_at` and `first_seen_at`. The backtester shall enforce `available_at ≤ decision time` for every input. | Must | P1 | Core no-look-ahead guarantee. |
| TA-031 | Revisable series shall be stored append-only with revision numbers, and backtests shall read them as-of the decision time. | Must | P2 | Applies when P2 sources arrive. |
| TA-032 | A look-ahead test harness shall run each indicator and signal on truncated data and confirm historical outputs never change when future bars are appended. | Must | P1 | Catches repaint and leakage bugs. |
| TA-033 | Backtests using pre-capture backfilled revisable data shall display a "revised data" warning badge. | Must | P2 | |

---

## 7. Beginner presentation

**Card template.** Every analysis card, alert and scan result has four parts:
1. **What this means:** one sentence with the current value in plain words.
2. **Why it matters:** one sentence on how traders use it.
3. **How reliable:** the evidence badge plus one line of reason, linked to sources.
4. **Learn more:** a link to the glossary (doc 07).

The Pro view adds raw values, parameters and statistics.

| Family | Example "what it means" | "How reliable" line |
|---|---|---|
| Trend | "BTC has been in an uptrend on the daily chart for 23 days." | "Supported but fragile: trend-following has worked in crypto historically, with long losing streaks and weaker results recently." |
| Momentum (RSI) | "RSI is 28: price fell fast recently." | "Weak as a buy signal on its own: in downtrends 'oversold' often gets more oversold." |
| Volatility | "Stormy: BTC moves about ±3.4% on a typical day now, vs ±2% normally." | "Descriptive, and future volatility is reasonably forecastable. It says nothing about direction." |
| Liquidity | "Spread 0.01%; a $1,000 order would move price <0.01%." | "Descriptive." |
| Funding / OI (P2) | "Traders are paying 35%/yr to stay long: crowded long positioning." | "Mixed: crowded positioning raises the risk of sharp drops, but it can persist for weeks." |
| Sentiment (P2) | "Fear & Greed 82 (Extreme Greed)." | "Mixed: partly built from price itself; short, inconsistent research record." |
| On-chain (P2) | "MVRV 2.9: price is ~2.9× holders' average cost." | "Weak: based on only a few market cycles; thresholds differ by source." |
| Patterns | "Bullish engulfing candle on 4h." | "Weak: studies find little reliable edge from candlestick patterns." |
| Events | "FOMC decision in 2 days (high impact)." | "Descriptive: volatility tends to rise around such events; direction is unpredictable." |

**Market Health Summary (P1).** One screen per asset plus a "whole market" view. Its cards are:
- **P1:** Trend, Volatility, Liquidity, Momentum, Upcoming events, Data freshness.
- **P2:** Leverage/positioning, Sentiment, Macro, On-chain.

Cards are colored by *condition* (calm/stormy, thin/deep), never by "buy/sell". The screen deliberately has **no single composite buy/sell score**. A composite would hide weak components behind one confident-looking number.

**Language rules:**
- Use probabilistic phrasing.
- Show base rates with sample size and uncertainty ("In 31 similar cases since 2018, BTC was higher 7 days later 55% of the time; range 37–71%").
- Below n = 30, say "too few past cases to judge".
- Copy and AI-assistant outputs are linted against a banned-phrase list ("guaranteed", "can't lose", "sure thing", "will pump").
- Contrarian indicators always show both readings ("high funding = bullish crowd, but also squeeze risk").

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SIG-014 | Every analysis card, alert and scan result shall follow the What / Why / How-reliable / Learn-more template. | Must | P1 | Coordinate with doc 07. |
| SIG-015 | Ubik shall maintain a versioned evidence registry (label, one-line reason, citations) for every indicator and signal, shown as a badge. User-defined items default to "Untested". | Must | P1 | Honesty principle. |
| SIG-016 | Ubik shall provide a Market Health Summary with the P1 cards listed above, extended with the P2 cards in P2. | Must | P1 | |
| SIG-017 | Ubik shall not show a composite buy/sell score in beginner mode. Any composite score shall display its components, weights and its own out-of-sample record. | Must | P1 | Avoids false certainty. |
| SIG-018 | Any displayed historical statistic shall include sample size, period and an uncertainty range. With n < 30, Ubik shall show "insufficient history" instead. | Must | P1 | |
| SIG-019 | UI copy templates and AI-assistant outputs about markets shall be checked against a banned-phrase list in automated tests. | Should | P1 | Coordinate with doc 07's AI assistant. |
| SIG-020 | Contrarian or ambiguous indicators shall present both interpretations. | Should | P2 | |

---

## 8. Data needs handoff (for doc 06)

| Analysis item | Data | Granularity | History depth | Likely source (free → paid) | Phase |
|---|---|---|---|---|---|
| All TA, charts, regimes | Spot OHLCV (+ trade count, taker-buy volume) | 1m base | Full available (BTC 2017+ on Binance) | Exchange REST + bulk archives (data.binance.vision), WebSocket live | P1 |
| VWAP precision, CVD, volume profile, fill realism | Trades (tick) | Per trade | 30–90 days raw, aggregates kept | Exchange WS / aggTrades archives | P1 collect, P2 use |
| Spread, depth, slippage, OFI | L2 order book | Top-20 snapshots every 1–10 s; depth bands per minute | 30 days raw, bands forever | Exchange WS | P1 basic, P2 |
| Watchlist / scanner universe | 24h tickers, symbol metadata, listing/delisting dates | 1 min; events | Point-in-time from P1 | Exchange REST (exchangeInfo) | P1 |
| Breadth, dominance, factor ranks | Daily OHLCV + market cap for top 50–100 coins | Daily | 5+ years | CoinGecko (Demo: 10k credits/mo, attribution), exchange klines; CoinMarketCap Basic | P2 (snapshot from P1) |
| BTC / stablecoin dominance | Global market cap | Daily snapshot | From P1 capture; history paid | CoinGecko `/global` (free) → paid chart | P1 capture |
| Funding | Settled + predicted funding | Per funding event; 1 min predicted | Full (free from exchanges) | Binance/Bybit/OKX public APIs, Coinalyze (free key) | P1 capture, P2 |
| OI, long/short ratios | OI (contracts, USD), ratios | 5 min | API: 30 days only → self-capture; daily archives | Binance API + data.binance.vision metrics, Coinalyze → CoinGlass ($29+/mo) | P1 capture, P2 |
| Liquidations | Liquidation events | Event (sampled) | From capture | Exchange WS (undercounts), Coinalyze → CoinGlass | P2 |
| Basis | Quarterly futures + index prices | 1 min–1 h | Full | Exchange APIs | P2 |
| Implied vol | DVOL; later option chains | 1 min–1 h | DVOL since 2021 | Deribit public API | P2 / P3 |
| On-chain basic | Hash rate, tx count, active addresses, MVRV | Daily | Full | Coin Metrics Community (CC BY-NC), mempool.space | P2 |
| On-chain advanced | Exchange flows/reserves, SOPR, NUPL | Daily/hourly | Full, point-in-time preferred | Glassnode (Professional API), CryptoQuant | P2 optional paid |
| Stablecoin supply | Circulating by coin/chain | Daily | Full | DefiLlama (free) | P2 |
| Fear & Greed | Index + label | Daily | Feb 2018+ | alternative.me (free, attribution) | P2 |
| Attention | Search interest | Daily/weekly | 5 years | Google Trends API alpha (application) | P3 |
| News / announcements | Headline, URL, tags, timestamps | Event | Rolling; metadata kept | RSS, CryptoPanic (verify terms), exchange announcement pages | P2 |
| Social sentiment | Mentions, sentiment | Hourly | 1+ year | Paid aggregators (LunarCrush, Santiment) | P3 Could |
| Macro | Dollar index, yields, VIX, equities, gold | Daily | 10+ years, vintages | FRED/ALFRED (free key) | P2 |
| ETF flows | Net flow by fund | Daily (T+1) | Jan 2024+ | Farside/SoSoValue → paid APIs | P2 Could |
| Event calendar | Scheduled events with `announced_at` | Event | Forward + past 5 years | federalreserve.gov, BLS, block height, Tokenomist (web/paid), manual | P1 basic, P2 |
| LLM digests | Prompts, outputs, model version | Event | Forever | LLM provider (doc 06/07) | P2 |

Cross-cutting needs for doc 06:
- the observation schema in §6;
- per-source license/attribution fields;
- per-series staleness thresholds;
- source adapters that can be swapped by configuration.

---

## Open questions for the owner

1. **Which country are you in?** It decides which exchange APIs you can reach: Binance.com returns "restricted location" to US IPs, and derivatives venues vary. It also decides which derivatives data sources are legitimate for you.
2. **What trading horizon interests you most:** minutes, hours or days? I recommend 1h–1d bars for P1, because costs are lower relative to moves and the evidence is stronger. Tick and order-book depth would then be mainly execution context.
3. **Paid data budget?** For example $0 now, and $30–100/month in P2 for CoinGlass or a Glassnode tier.
4. **Preferred alert channels?** Telegram, email, push?
5. **Is LLM news summarization worth an ongoing API cost in P2?** Do you have a provider preference?
6. **Could Ubik ever become commercial or multi-user?** If so, avoid non-commercial (CC BY-NC) and personal-use-only data sources from the start.
7. **Do you want candlestick and chart-pattern features for learning,** knowing they will be labelled "Weak evidence"?

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Beginners read indicators as instructions ("RSI oversold → buy") | Losses; loss of trust | Evidence badges, the What/Why/How-reliable template, no composite buy score, banned-phrase lint |
| Look-ahead, repaint or full-sample fitting bugs inflate backtests | False confidence before live money | TA-003 parity tests, TA-032 truncation harness, point-in-time storage, confirmed-only repainting constructs |
| Data revisions and survivorship bias | Optimistic backtests | Append-only storage, point-in-time universe, "revised data" badge |
| Short or missing free history for derivatives, dominance and sentiment | P2 signals can't be tested over long periods | Start capturing in P1 (MA-009, MA-014) |
| Free sources change terms, rate limits or geo-blocks, or shut down | Broken cards; silent staleness | Source adapters, freshness checks (MA-029/030), graceful degradation |
| License limits (CC BY-NC, personal-use plans, attribution) | Blocks a future SaaS | Record license per source; review before P4 (doc 08) |
| Mass testing in the scanner / Signal Lab finds "edges" that are luck | Overfit strategies | Multiple-testing ledger, deflated statistics, walk-forward (MA-026/027) |
| Edges documented in 2014–2021 data fade after ETFs and institutional flows | Strategies underperform | Recency-weighted evaluation; paper incubation; evidence registry reviewed each release |
| LLM hallucination or leakage in news features | Misinformation; inflated backtests | Cited digests, no predictions, cutoff rule (MA-022), stored outputs |
| Alert fatigue | Users ignore important alerts | Cooldowns, hysteresis, rate limits, quiet hours (SIG-006) |
| "Everything pro firms do" scope creep (indicator bloat, HFT microstructure) | Delays P1; false promise | Phased catalog; microstructure positioned as execution context, not alpha |
| Order-book storage cost | Disk and ops burden on a personal server | Depth bands + short raw retention (doc 06) |
