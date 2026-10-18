# Pro Trading Capabilities, Competitive Landscape, and Build vs Reuse

*Planning doc 02 · Scope: how professional firms optimize trading, what Ubik can realistically copy, the competitor and open-source landscape, table stakes vs differentiators, and build-vs-reuse. Facts, prices and versions were checked on 2026-10-18. They change often, so re-check them before making a decision that depends on them.*

## Summary

- **Ubik can copy the professional *process* but not the professional *structural advantages*.** An individual can match pro-grade research discipline, statistical validation, risk limits, cost analysis, monitoring and strategy governance, because these are mostly software and cheap compute. An individual cannot match pros on speed (servers placed next to the exchange), trading costs (VIP fee tiers and rebates) or scale. Ubik should go all-in on the first group and clearly decline the second: no high-frequency trading (HFT), no latency arbitrage, and no live market making at retail fee tiers.
- **For an individual, the biggest gain comes from not losing money through avoidable mistakes:** overfit backtests, fee drag, oversized positions, operational errors and stolen API keys. Ubik's core promise should be a "trading-desk process in a box" that reports honest numbers.
- **There is a clear gap in the market.** Hosted retail bots (3Commas, Cryptohopper, Bitsgap, Coinrule, Pionex, exchange-native grid/DCA bots) are easy to use. But their backtests are shallow and run on candles only, they show no overfitting statistics, and they require handing API keys to a third party. Open-source engines (Freqtrade, NautilusTrader, Hummingbot, LEAN) are powerful but built for developers. No product combines rigor, honesty, beginner UX and keeping your own keys. That combination is Ubik's opening.
- **Differentiators worth investing in:**
  - A strategy lifecycle with promotion gates and kill criteria declared up front.
  - An "honest backtest report card": out-of-sample results, trial count, Deflated Sharpe Ratio, cost drag and a benchmark.
  - Measured backtest-vs-live drift.
  - A plain-language explanation for every decision.
  - Self-hosted keys and full data export.
- **Build vs reuse: build Ubik's own thin core and reuse permissively licensed libraries.**
  - **Build ourselves:** the domain model, strategy API, event-driven engine, simulated execution and risk layer.
  - **Reuse:** CCXT (MIT) for exchanges, TradingView Lightweight Charts (Apache-2.0) for charts, and BSD/MIT/Apache analytics libraries.
  - **Avoid:** forking Freqtrade (GPL-3.0, one bot per strategy, fills simulated from candles only) and building on vectorbt (its Commons Clause forbids selling a product built on it).
  - **Re-evaluate NautilusTrader (LGPL-3.0) as the engine once its v2 API stabilizes.** It has the best architecture of the candidates, but it is mid-rewrite: 2.0.0rc7 was released on 15 Oct 2026. A 2-week spike at the start of P1 should confirm the choice.

---

## 1. How professional trading firms optimize trading

### 1.1 Who "the pros" are

Professional firms fall into three archetypes, and they optimize for very different things:

| Archetype | Typical holding period | Where their edge comes from | Crypto examples |
|---|---|---|---|
| **Quant / systematic funds** | Minutes to months | Research quality, portfolio construction, risk discipline | Systematic crypto funds, CTAs (trend-following funds) trading BTC futures |
| **Prop shops / HFT / market makers** | Microseconds to minutes | Speed, fee rebates, inventory management, exchange relationships | Jump, Wintermute, GSR-style liquidity providers |
| **Crypto trading desks** | Hours to months | Execution quality, basis/funding trades, managing exchange (counterparty) risk | Fund and treasury desks, OTC desks |

Ubik's natural model is the **first and third archetypes**: slower holding periods, a disciplined process and close attention to risk. The second archetype wins on infrastructure that an individual cannot buy.

The professional trade lifecycle, which the rest of this section walks through:

```
Data ─► Research (hypothesis → backtest → validation) ─► Signal combination ─► Portfolio construction & sizing
   ─► Pre-trade risk ─► Execution ─► Post-trade analytics (TCA, attribution) ─► Monitoring & ops
   ─► Governance (review, kill, retire) ─► back to Research
```

### 1.2 The lifecycle, capability by capability

Each capability is laid out the same way: **what pros do**, what **Ubik can** realistically copy (and in which phase), and what **Ubik should not** attempt.

#### C1 — Data infrastructure
- **What pros do:**
  - Capture every trade and the full order book from many exchanges. (The order book is the live list of buy and sell orders waiting at each price. "L2" means aggregated by price level; "L3" means every individual order.)
  - Timestamp each event at the moment of capture and normalize symbols across exchanges.
  - Store data "point-in-time": exactly what was knowable at that moment, including coins that were later delisted.
  - Run automated quality checks and keep redundant feeds.
  - Add alternative data such as on-chain flows, funding rates, open interest and sentiment.
  - Spend six to seven figures a year on all of this.
- **What Ubik can do:**
  - **P1:** 1-minute candles (OHLCV = open, high, low, close, volume) and trades for a handful of liquid pairs from free exchange APIs. Periodic order-book snapshots to estimate the spread (the gap between the best buy and best sell price). Automated checks for gaps and outliers. Versioned datasets, so every backtest can be reproduced.
  - **P2:** free derivatives and on-chain data (see 03 and 06).
- **What Ubik should not do:** build a full-depth, multi-exchange tick archive in P1. It is expensive to store and run, and strategies that trade on minute-to-day horizons don't need it. Paid data feeds should be added only when a validated strategy demonstrably needs them.

#### C2 — Research and alpha generation
"Alpha" means return that isn't explained by simply holding the market.
- **What pros do:**
  - **Start from a hypothesis.** Every strategy needs an economic reason the edge should exist, such as forced sellers, a risk premium or a behavioral bias. A curve that merely looks good is not enough.
  - **Then backtest, then validate in ways designed to defeat self-deception:**
    - Held-out data the strategy was never tuned on.
    - Walk-forward testing: repeatedly fit on a past window, then test on the next window the strategy hasn't seen.
    - Corrections for how many variants were tried, such as the Deflated Sharpe Ratio ([Bailey & López de Prado](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2460551)) and the Probability of Backtest Overfitting ([Bailey et al.](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2326253)). Academic finance now demands much higher significance thresholds because so many ideas get tested (Harvey, Liu & Zhu, 2016).
  - **Then check capacity and get peer review.** Research code is the same code that runs in production.
- **What Ubik can do: nearly all of it.** Validation is math and cheap compute. From P1/P2, Ubik can offer:
  - trial counting and a locked out-of-sample period;
  - walk-forward testing and DSR/PBO scores;
  - lookahead-bias checks;
  - cost-inclusive backtests against a benchmark;
  - replays of real crash scenarios.

  **This is the area where an individual can genuinely match a pro firm.** Algorithms are in 04.
- **What Ubik should not do:** offer beginners large-scale automated "alpha mining", meaning searching thousands of indicator combinations or ML features. With about 5 pairs and a few years of data, it mostly produces overfit strategies. If Ubik ever offers it (P4), it must sit behind the same validation gates.

#### C3 — Signal combination
- **What pros do:**
  - Blend many weak, weakly correlated signals into one forecast. They weight the signals by statistical methods, such as shrinkage regression or the information coefficient (the correlation between a signal and the returns that follow it).
  - Neutralize exposures they don't want, such as exposure to the overall market.
  - Penalize turnover (how often positions change).
- **What Ubik can do:**
  - **P1:** rules combined with AND/OR.
  - **P2:** weighted scores or votes, plus a "does this signal add anything?" report on each signal's marginal contribution.
  - Running a few uncorrelated strategies side by side is the realistic retail version of signal combination.
- **What Ubik should not do:** build factor-neutral cross-sectional models. Ubik will trade 3–10 coins that largely move with BTC, and their correlations approach 1 in crashes. That universe is too narrow for these models to work.

#### C4 — Portfolio construction and capital allocation
- **What pros do:**
  - Size positions by risk, not by conviction. Volatility targeting scales each position so its expected volatility stays constant.
  - Set a risk budget for each strategy.
  - Use covariance-aware optimizers with shrinkage.
  - Use fractional Kelly sizing. The Kelly formula gives an optimal bet size for a known edge; pros bet only a fraction of it because the edge itself is uncertain.
  - Respect capacity limits and use rebalancing that accounts for costs.
- **What Ubik can do:**
  - **P1:** separate "what to trade" (the signal) from "how much" (the sizing policy), so any strategy can run under any sizing policy. Default to risk-based sizing with hard caps.
  - **P2:** inverse-volatility or equal-risk allocation across strategies.
  - **P2+:** optimizers from permissive libraries, e.g. [skfolio](https://pypi.org/project/skfolio/) and Riskfolio-Lib (BSD-3) or PyPortfolioOpt (MIT).
- **What Ubik should not do:**
  - Default to unconstrained mean-variance optimization. It amplifies estimation noise, which is why it is nicknamed "error maximization".
  - Offer leverage before P4.

#### C5 — Pre-trade and real-time risk
- **What pros do:**
  - Run every order through automated checks before it leaves: maximum size, a price band to catch "fat-finger" typos, credit, and position and exposure limits. US broker-dealers are legally required to have these checks (SEC Rule 15c3-5).
  - Watch P&L and exposure in real time.
  - Apply drawdown limits that cut risk automatically.
  - Run stress tests and compute VaR (value-at-risk, a statistical "typical worst day").
  - Keep an independent risk function that can overrule traders.
  - **Crypto desks also:** cap exposure to each exchange, watch for stablecoin depegs, monitor liquidation and auto-deleveraging (ADL) risk, and staff all of this 24/7.
- **What Ubik can do:**
  - **P1:** a risk engine that sits between every strategy and the exchange in every mode, and that strategies cannot bypass. A daily-loss limit, a max-drawdown halt and per-asset caps, all on by default. A kill switch.
  - **P2:** replays of historical crashes.
  - **P3:** a per-exchange and per-stablecoin exposure dashboard.
  - Details are in 05.
- **What Ubik should not do:** lead with VaR as the headline risk number for beginners. Crypto returns are fat-tailed (extreme moves happen far more often than a normal curve predicts), so VaR understates crash risk. Lead with drawdown and "what would have happened in crash X".

#### C6 — Execution and transaction cost analysis (TCA)
- **What pros do:**
  - **Execution algorithms:**
    - **TWAP** spreads an order evenly over time.
    - **VWAP** trades in step with market volume.
    - **Implementation-shortfall algorithms** (Almgren & Chriss, 2000) balance price impact against the risk of waiting.
  - **Smart order routing** across exchanges.
  - **Order-type and position choices:** maker vs taker decisions (resting a limit order vs crossing the spread) and modeling of queue position.
  - **Fee-tier management and maker rebates.**
  - **Latency engineering:** co-location (servers in the exchange's data center), kernel-bypass networking and FPGAs.
  - **TCA:** compare each fill with the **decision price** (when the signal fired) and the **arrival price** (when the order reached the market). The gap is called **implementation shortfall**.
- **What Ubik can do:** at retail size on BTC, ETH or SOL, market impact is negligible. Costs come almost entirely from **fees and the spread**, so Ubik should optimize those:
  - fee-aware order choice (limit or post-only orders when the trade isn't urgent);
  - throttling over-trading;
  - simple slicing when an order is large relative to the best bid/ask;
  - full TCA on every paper fill (P1) and live fill (P3), so slippage assumptions are calibrated from real fills.

  Details are in 05.
- **What Ubik should not do:**
  - Co-location, latency arbitrage, HFT or strategies that must react in under a second. A home or cloud connection has latency of tens to hundreds of milliseconds and loses every race against firms that measure in microseconds.
  - Smart routing across many exchanges, which is P4 at most.

#### C7 — Post-trade analytics and performance attribution
- **What pros do:** produce a daily "P&L explain" that splits results into:
  - market moves (beta);
  - strategy skill (alpha);
  - costs.

  They break this down by strategy, asset and factor. They also compare live results with backtest and paper results (the "implementation gap") and watch how capacity decays.
- **What Ubik can do:**
  - **P1:** standard metrics with plain-language explanations (04), per-strategy trade journals and a backtest-vs-live drift report.
  - **P2:** P&L split into BTC beta, strategy contribution, fees and slippage.
- **What Ubik should not do:** build multi-factor risk models (Barra-style). There are too few assets for them to work.

#### C8 — Monitoring, operations and incident response
- **What pros do:**
  - Run on-call rotations, heartbeats, alerting and automatic failover.
  - Reconcile their records with the exchange or prime broker.
  - Use change management, kill switches, runbooks and post-mortems.
  - Keep keys in hardware security modules.

  The classic cautionary tale is Knight Capital in 2012. A botched deployment reactivated dead code and sent more than 4 million orders in 45 minutes, losing roughly $440–460M. Before the market opened, 97 automated error emails had gone unread. It was the SEC's first enforcement action under its market-access rule ([summary](https://www.wilmerhale.com/en/insights/client-alerts/knight-capital-settles-rule-15c3-5-violations-with-sec-agrees-to-pay-12-million)).
- **What Ubik can do:**
  - **P1:** a watchdog and heartbeats; auto-pause when data goes stale; alerts to the owner's phone; an append-only audit log that allows deterministic replay; versioned deployments.
  - **P3:** reconciliation with the exchange.
  - Throughout: "fail safe" design. If something breaks, Ubik stops placing new orders rather than carrying on blind. A single server with backups is enough.
- **What Ubik should not do:** run multi-region high-availability clusters for one user. At this scale, the extra complexity causes more incidents than it prevents. Aim to fail into a paused, flat state rather than to never fail.

#### C9 — Research governance
- **What pros do:**
  - No strategy trades until it passes a review (an investment committee). The review covers a written hypothesis, the evidence, risk limits, capacity and **kill criteria agreed in advance**.
  - Capital ramps up in stages.
  - Live results are monitored for decay. Alpha fades as other traders discover the same edge.
  - Retired strategies are documented, not deleted.
- **What Ubik can do: all of it, from P1.** This is the most valuable pro habit for an individual, because the biggest risk is usually the owner's own enthusiasm. In practice:
  - lifecycle states with promotion gates;
  - kill criteria registered in advance, with automatic pause when they are breached;
  - decay alerts;
  - a "strategy graveyard" of retired strategies, which also feeds the trial count.
- **What Ubik should not do:** nothing here needs avoiding. Just keep the gates understandable. A user may override a gate only with an explicit acknowledgment that gets logged.

#### C10 — Treasury and counterparty management (crypto-desk specific)
- **What pros do:**
  - Keep minimal balances on exchanges and spread capital across venues.
  - Sweep funds to custody.
  - Watch exchange health and diversify across stablecoins.

  Recent lessons:
  - FTX collapsed in 2022.
  - Bybit lost about $1.5B in a February 2025 hack, which the FBI attributed to North Korea's Lazarus group ([BleepingComputer](https://www.bleepingcomputer.com/news/security/lazarus-hacked-bybit-via-a-breached-safe-wallet-developer-machine/)).
  - During the 10 Oct 2025 crash, the USDe stablecoin briefly traded at about $0.65 on Binance ([Galaxy](https://www.galaxy.com/insights/research/cryptos-flash-crash-liquidation-binance-adl-auto-deleveraging)).
- **What Ubik can do:**
  - **P3:** show capital per exchange and per stablecoin, with caps and warnings.
  - Never request withdrawal permissions.
  - Explain that paper money is safe, but live funds held on an exchange carry counterparty risk.
- **What Ubik should not do:** move money between exchanges automatically. That would require API keys with withdrawal rights, which is an unacceptable risk.

### 1.3 Where an individual can and cannot compete

| Source of edge | Pro firms | Individual using Ubik | Verdict |
|---|---|---|---|
| Speed / latency | Co-located, microseconds | Home or cloud VM, ~50–300 ms | **Cannot compete.** Avoid any strategy that depends on speed. |
| Trading costs | Top VIP tiers, near-zero fees or maker rebates | Base tiers, e.g. ~0.1% per side on large exchanges; 0.40% maker / 0.60% taker at Coinbase Advanced's entry tier ([NerdWallet](https://www.nerdwallet.com/investing/reviews/coinbase)) | **Cannot compete.** Trade less often; make cost drag visible. |
| Information | Paid tick data, alternative data, research teams | Free public data, which is unusually rich in crypto | **Partly.** Data access is decent, but there is no information edge. |
| Size / capacity | Large positions that move prices | Small positions with negligible impact | **Advantage.** But "small coins with no liquidity" is a trap, not an edge. |
| Constraints | Mandates, quarterly reporting, career risk | None. Can hold for a long time or sit in cash. | **Advantage.** Patience is a real retail edge. |
| Process discipline | Committees, risk teams | Ubik's gates and limits | **Can match.** This is Ubik's core value. |
| Statistical rigor | PhD teams | Automated in Ubik | **Can match**, because compute is cheap. |
| Operations / security | Dedicated teams | Ubik guardrails, no one on call | **Partly.** Ubik must fail safe. |

**Fee arithmetic every beginner should see.** Take a strategy that is fully invested and turns over once a day, paying 0.1% on each side of the trade:
- That is about 0.2% per day.
- Compounded over a year, (1 − 0.002)^365 ≈ 0.48.
- So **fees alone consume about half the capital in a year unless the strategy has a real edge.**
- At a 0.6% taker fee, the same strategy is hopeless.

Pros at the top fee tiers pay close to nothing. This is why "fee drag" must be a first-class number in Ubik (CAP-022).

### 1.4 Explicit non-goals

| Non-goal | Why | Revisit? |
|---|---|---|
| Co-located HFT, latency arbitrage | Infrastructure and fee disadvantages make losses near-certain | Never |
| Live market making on CEXs at retail fee tiers | Needs maker rebates, fast inventory management and queue priority | P4: simulation and education only (CAP-043) |
| Cross-exchange arbitrage | Thin spreads, transfer delays, and keys with withdrawal rights | P4, simulation first |
| MEV / front-running on DEXs (profiting by jumping ahead of other users' transactions) | Harms other users; legally risky in some jurisdictions | Never |
| Managing other people's money, copy-trading, selling signals | Regulated activity (see 08) | Only with legal review |
| Leverage, derivatives | Beginner safety | P4 |
| Autonomous AI trading without human confirmation | LLM traders have been erratic (§2.2) | Not by default |

---

## 2. Competitive landscape (status as of October 2026)

### 2.1 Retail platforms and analysis tools

Ratings are qualitative: L = low, M = medium, H = high. "Backtest quality" covers realism (fees and slippage, fill model), bias protection and statistical validation.

| Product | What it is | Analysis | Backtest quality | Paper trading | Beginner-friendly | Live trading | Pricing (Oct 2026) | Open source |
|---|---|---|---|---|---|---|---|---|
| **3Commas** | Hosted DCA, grid and signal bots, plus a SmartTrade terminal | L–M; TradingView signals | L–M; candle-based bot backtests | Yes, demo account in all plans | M | Yes, many CEXs via API keys | Starter $15–20, Pro $38–50, Expert $105–140/mo (annual vs monthly billing), trial only ([official](https://3commas.io/pricing)) | No |
| **Cryptohopper** | Hosted bots, strategy designer, marketplace | M | L–M | Yes (one paper bot per subscription) | M | Yes, 16+ CEXs | Explorer $29, Adventurer $69, Hero $129/mo (17% off annually); 3-day trial; no free tier listed ([official](https://www.cryptohopper.com/pricing)) | No |
| **Bitsgap** | Hosted grid, DCA and combo bots | L–M | L; backtest lookback capped at 30/180/365 days by tier | Yes (free plan includes 20 demo bots) | M–H | Yes | Free; Basic $29, Advanced $69, Pro $149/mo ($23/$55/$119 billed annually) ([official](https://bitsgap.com/pricing)) | No |
| **Coinrule** | Hosted IF-THEN rule builder, 350+ templates | L–M | L | Yes (demo) | **H** | Yes, 20+ venues; stocks via Alpaca | Free tier; about $29.99, $59.99, $449.99/mo (third-party figures, [review](https://www.softwaretestinghelp.com/coinrule-review/)) | No |
| **TradeSanta** | Hosted grid/DCA long and short templates | L | L | Limited | H | Yes | About $15–25/mo ([HackerNoon roundup](https://hackernoon.com/lang/bg/top-10-best-ai-bot-trading-in-2026-features-and-pricing)) | No |
| **Pionex** | An exchange with 16+ built-in bots | L | L (parameter suggestions from recent history) | Limited | H | Yes, but funds are held by Pionex | Bots free; about 0.05% spot fee (unverified, [review](https://www.daytrading.com/pionex-vs-luno)) | No |
| **Exchange-native bots** (Binance, Bybit, OKX) | Grid, DCA, rebalancing, TWAP and funding-arbitrage bots; OKX signal bot fed by TradingView ([overview](https://forklog.com/en/grid-dca-and-the-signals-marketplace-an-overview-of-okxs-trading-bots/)) | Exchange charts | L (historical APY shown) | Mostly none | H | Yes, single exchange, runs on the exchange's servers, no API key needed | Free; you pay trading fees. Leverage is one click away. | No |
| **Gainium** | Hosted DCA, grid and combo bots | L–M | M (simulates liquidations) | Yes, free | M | Yes | Free backtesting and paper trading; paid credits for live bots ([site](https://gainium.io/backtesting)) | No |
| **TradingView** | Charting, the Pine Script language and a Strategy Tester | **H** | M (bar-based; scripts can "repaint", i.e. change past signals) | Yes | M–H | Via brokers and webhooks | Free Basic; Essential ~$13–15, Plus ~$30–35, Premium ~$60–70, Ultimate ~$200–240/mo ([comparison](https://friendofthetrend.com/tradingview/plan-comparison/)) | No (its [Lightweight Charts](https://www.npmjs.com/package/lightweight-charts) library is Apache-2.0) |

### 2.2 Open-source frameworks and libraries

Versions are from PyPI on 2026-10-18.

| Framework | License | Language | Status | Backtest quality | Paper / live | Beginner | Notes |
|---|---|---|---|---|---|---|---|
| **[Freqtrade](https://github.com/freqtrade/freqtrade)** | GPL-3.0 | Python | Very active; 2026.9 (29 Sep 2026); ~55k GitHub stars | M. Candle-based; orders fill at the requested price with no slippage while inside the candle's range ([docs](https://www.freqtrade.io/en/stable/backtesting/)). Strong bias tooling ([lookahead-analysis](https://www.freqtrade.io/en/stable/lookahead-analysis/)). | Dry-run / yes, spot and futures via CCXT | L (CLI, config, Python) | Hyperopt, FreqAI (ML), FreqUI web UI, Telegram |
| **[Hummingbot](https://hummingbot.org/)** | Apache-2.0 | Python/Cython | Active; release 20260922 | L–M (candle-based dashboard backtests) | Paper connectors / yes, CEX and DEX ("318 connectors") | L | Focused on market making and arbitrage; governed by a foundation; Condor AI harness |
| **Jesse** | MIT core; live plugin needs a paid license ([docs](https://docs.jesse.trade/docs/livetrade)) | Python | Active; 3.4.0 (16 Oct 2026) | M–H (clean multi-timeframe design, avoids lookahead) | Via the licensed plugin | M | Elegant strategy API |
| **OctoBot** | GPL-3.0 | Python | 2.1.1 (Mar 2026) | M | Yes / yes, self-hosted or cloud | M–H | Free and paid cloud tiers; grid, DCA, TradingView and AI modes ([review](https://gainium.io/review/octobot)) |
| **[NautilusTrader](https://github.com/nautechsystems/nautilus_trader)** | LGPL-3.0 | Rust core + Python | Very active; 1.231.0 stable (Aug 2026); v2 rewrite at 2.0.0rc7 (15 Oct 2026); warns of breaking changes | **H** (event-driven, deterministic, order-book level, nanosecond timestamps) | Sandbox / yes: Binance, Bybit, OKX, Coinbase, Kraken, Deribit, Hyperliquid and more | Very L | Backtest-live parity is its central design goal |
| **vectorbt** | Apache-2.0 + [Commons Clause](https://github.com/polakowo/vectorbt/blob/master/LICENSE.md) | Python (Numba, Rust) | Active; 1.1.2 (Oct 2026); paid PRO edition | Research-grade: very fast parameter sweeps, simplified fills | No / no | L | Great for exploring ideas; the Commons Clause bars selling a product built on it |
| **backtrader** | GPL-3.0+ | Python | Last release Apr 2023, effectively unmaintained | M | Legacy | L | Avoid for new work |
| **backtesting.py** | AGPL-3.0 | Python | 0.6.6 (Jul 2026) | L–M (single asset, simple) | No / no | M | AGPL is hazardous for SaaS |
| **LEAN / QuantConnect** | Apache-2.0 engine | C# + Python | Active. Cloud plans: Free (backtests, no live trading); Researcher $84/mo adds live trading on Coinbase, Binance, Kraken and Bybit ([official](https://www.quantconnect.com/pricing)) | H (event-driven, multi-asset, good data) | Yes / yes | L–M | Heavy .NET stack with an equities heritage |
| **CCXT** | MIT | Python, TypeScript, others | 4.5.88 (Oct 2026) | n/a | Connects to 100+ exchanges (REST and WebSocket) | n/a | The de-facto standard exchange-API layer |
| **hftbacktest** | MIT | Rust + Python | 2.4.4 | Very H for HFT and market making (models queue position and latency) | n/a | Very L | P4 reference for market-making simulation |

### 2.3 Incidents and complaints, and what Ubik should learn

| Incident | What happened | Lesson for Ubik |
|---|---|---|
| **3Commas API-key leak (Oct–Dec 2022)** | Keys stored by 3Commas were used for unauthorized trades, first on FTX ([The Block](https://www.theblock.co/post/179237/ftx-api-keys-3commas-exploited)). The company first blamed phishing, then confirmed in December 2022 that the leaked data was genuine. Binance told users to disable any key ever entered into 3Commas ([The Block](https://www.theblock.co/post/198295/binance-warns-about-3commas-api-leak-says-users-should-disable-keys), [Cointelegraph](https://cointelegraph.com/news/3commas-ceo-confirms-api-key-leak-following-warning-from-cz)). Reported losses ran from millions to tens of millions of dollars. | Keys never leave the owner's own infrastructure. Use trade-only keys with no withdrawal permission and an IP allowlist. Alert on any trade Ubik didn't place (CAP-028, CAP-036; detail in 08). |
| **Platform shutdowns and plan changes** | Trality wound down its bot product in 2023 ([HaasOnline blog](https://www.haasonline.com/the-fall-of-more-trading-bot-companies-and-projects/), a competitor's account). Shrimpy discontinued its consumer offering ([Cryptohopper blog](https://cryptohopper.com/de/blog/shrimpy-stops-consumer-offering-introducing-cryptohopper-as-the-perfect-alternative-9465), also a competitor). Cryptohopper's pricing page now lists only a 3-day trial. | Self-host, offer full data export and avoid lock-in (CAP-037). |
| **Malware disguised as trading bots** | GitHub repositories with inflated stars stole keys and wallets, e.g. a "solana-pumpfun-bot" ([Cointelegraph](https://cointelegraph.com/news/solana-trading-bot-github-malware-scam)). In 2026, a campaign of fake Polymarket trading-bot repositories used typosquatted npm packages (packages named to look like real ones) ([summary of SlowMist/StepSecurity reports](https://threatcluster.io/cluster/malware-spread-via-fake-polymarket-trading-bot-targets-defi--efb72af6)). | Pin and lock dependencies; scan licenses and vulnerabilities; never run untrusted community strategy code outside a sandbox (CAP-044; detail in 08). |
| **Exchange failures** | FTX collapsed in 2022. Bybit was hacked for about $1.5B in February 2025. | Cap and display exposure per exchange (CAP-018). |
| **Market-structure stress (10 Oct 2025)** | The largest liquidation cascade on record, with about $19B liquidated ([CoinGecko](https://www.coingecko.com/learn/october-10-crypto-crash-explained)). USDe was mispriced on Binance because Binance valued it from its own order book. Binance later reimbursed about $283M ([RockawayX](https://rockawayx.com/insights/crypto-market-update-october-2025)). | Backtests built on calm data understate tail risk. Stop orders can fill far below their trigger. Stablecoins carry risk. Run stress replays (CAP-010). |
| **Knight Capital (2012)** | A deployment error caused a $440–460M loss in 45 minutes (§1.2 C8). | Deployment controls, alerts that actually get read, and a kill switch (CAP-017, CAP-029). |
| **AI-trading hype** | In the "Alpha Arena" contests (late 2025), LLMs traded crypto perpetuals with real money. Results varied wildly, were sensitive to prompt wording, and several models lost heavily ([Forklog](https://forklog.com/en/ai-model-grok-4-2-triumphs-in-trading-tournament/amp)). | The AI explains and assists but does not trade on its own; AI-drafted strategies pass the same gates as any other (CAP-041). |
| **Recurring user complaints across retail bot reviews** | Backtests that look great but fail live. DCA and martingale bots (bots that buy more as the price falls) that "work until a long downtrend". Grid bots that lose in trending markets. Leverage one click away. Slow support. | Every template ships with a "when this fails" section and a demonstration (CAP-038). Every number is labeled with where it came from (CAP-039). |

---

## 3. Table stakes vs differentiators

**Table stakes.** Without these, Ubik won't be credible next to the free and cheap alternatives:
- live market data and charting with common indicators (03);
- template strategies (DCA, grid, trend, mean-reversion) and a no-code builder (04, 07);
- backtests that include fees;
- paper trading with simulated balances;
- stop-loss / take-profit and position limits (05);
- a P&L dashboard;
- notifications;
- a UI that works on mobile;
- secure key handling, from P3 (08);
- support for at least one or two major exchanges (06).

**Differentiators**, where Ubik can genuinely stand out:

| Differentiator | Why it's defensible | Who does it today |
|---|---|---|
| **Honest backtest report card**: out-of-sample results, walk-forward, trial count, Deflated Sharpe Ratio / PBO, benchmark, fee drag and stress replays in one beginner-readable verdict | Retail tools are paid for exciting numbers, so they lack the incentive. Open-source tools have the pieces but no one assembles them for beginners. | Nobody, end to end. Freqtrade has bias tools; QuantConnect has rigor but isn't aimed at beginners. |
| **Strategy lifecycle with gates and pre-declared kill criteria** ("investment committee in a box") | Turns pro governance into a guided workflow | Nobody at retail level |
| **Measured backtest-vs-live parity**: re-simulate the paper or live period and explain every difference | Requires an engine designed for it from day one | NautilusTrader's architecture enables it, but it has no UX for it |
| **Explain-everything traces**: each signal, risk decision and order carries a plain-language reason, and the AI assistant answers from these traces | Requires owning the domain model and event log | Nobody |
| **Self-hosted keys and full data export** | Avoids 3Commas-style leaks and platform shutdowns | Open-source tools (but not beginner-friendly) |
| **Pro-style TCA and attribution at retail scale** | Fees and slippage are the retail edge-killers | Rare outside institutional tools |

---

## 4. Build vs reuse

### 4.1 What Ubik's core must deliver
The core must deliver eight things, which serve as the criteria for evaluating the options in 4.2:
1. **Parity:** the same strategy code runs unchanged in backtest, paper and live.
2. **Explainable decisions:** every decision is recorded and can be narrated in plain language.
3. **Portfolio-level risk** across multiple strategies.
4. **A no-code builder** that compiles to the same strategy model as coded strategies.
5. **A validation pipeline:** trial counting, walk-forward, DSR/PBO.
6. **License freedom**, to keep the P4 multi-user, SaaS or commercial path open.
7. **Maintainable by a very small team** in Python.
8. **Spot markets at minute-to-daily horizons first**; order-book simulation is P4.

### 4.2 Options evaluated

| Option | Strengths | Weaknesses | Fit |
|---|---|---|---|
| **A. Fork / extend Freqtrade** | Mature and battle-tested; dry-run mode; CCXT; web UI; hyperopt; bias tools; big community | GPL-3.0 makes everything combined with it GPL if distributed. Built around one bot per strategy, so a portfolio-level risk layer would fight the framework. Fills are candle-only with no slippage inside the candle. Its DataFrame-style strategy API is awkward for explainability traces. The beginner UX would need a rebuild anyway. | 2/5 |
| **B. Build on NautilusTrader** | Best architecture: event-driven, deterministic, parity by design, risk engine, reconciliation, many crypto adapters. LGPL allows proprietary code that uses it as a library. | Mid-rewrite: v2 is at release candidate 7 and breaking changes are expected. Steep learning curve, with a Rust toolchain needed for any core changes. Far more than P1 needs. Everything for beginners would still have to be built on top. | 3.5/5 now; 4/5 once v2 is stable |
| **C. Build on Hummingbot** | Apache-2.0; huge connector set; strong for market making | Its center of gravity is market making and arbitrage (P4 non-goals for now); weak backtesting | 2/5 |
| **D. Build on LEAN** | Apache-2.0; rigorous; multi-asset | C#/.NET core alongside a Python stack; heavy; equities-centric; QuantConnect's cloud is the polished path | 2/5 |
| **E. vectorbt as the backtester** | Extremely fast parameter sweeps | Commons Clause blocks selling Ubik; vectorized design breaks parity (live trading is event-by-event); no live trading | 1.5/5 for the core; fine for the owner's private research notebooks |
| **F. Own thin core plus permissive libraries** | Full control of the domain model and explainability. One event-driven engine for all three modes. License freedom. P1 scope (a few spot pairs, bars and trades) is a few thousand lines. | We own the simulator and order-management correctness, so bugs could create false confidence. Exchange quirks arrive in P3 (CCXT absorbs many of them). | **4/5 (recommended)** |

### 4.3 License implications
*This is not legal advice; doc 08 should confirm.*

| License | Examples | Private self-hosted use | Distributing Ubik as a download | Running Ubik as SaaS |
|---|---|---|---|---|
| MIT / BSD / Apache-2.0 | CCXT, Hummingbot, Jesse core, LEAN, skfolio, Lightweight Charts | OK | OK (keep the notices; Apache also grants patent rights) | OK |
| LGPL-3.0 | NautilusTrader | OK | OK if used as a library. Changes to Nautilus itself must be published under LGPL, and users must be able to swap the library. | OK |
| GPL-3.0 | Freqtrade, OctoBot, backtrader | OK | The whole combined program must be GPL-3.0 | Technically OK (no distribution happens), but it closes off a later downloadable or closed-source release |
| AGPL-3.0 | backtesting.py | OK | Must be AGPL | Must offer source code to network users |
| Apache-2.0 + Commons Clause | vectorbt | OK | Can't sell a product whose value derives substantially from it | Same restriction |
| Proprietary add-on | Jesse live-trading plugin | License fee | Depends on its terms | Depends on its terms |

### 4.4 Recommendation
1. **Build Ubik's own core in Python:**
   - domain model (instruments, orders, fills, positions, accounts);
   - strategy API that the no-code builder compiles to;
   - a single event-driven engine with pluggable clocks: historical replay for backtests, live feed for paper and live;
   - simulated execution with fee, spread and slippage models;
   - risk engine;
   - append-only event log.

   Design ideas to borrow openly: NautilusTrader's message-bus and adapter architecture and its reconciliation approach; Freqtrade's lookahead and recursive bias checks and its dry-run mode.
2. **Reuse permissive components:**
   - CCXT (MIT) for exchange REST and WebSocket access;
   - Lightweight Charts (Apache-2.0) for the UI (03 and 07 decide);
   - BSD/MIT/Apache libraries for indicators, statistics and portfolio optimization;
   - hftbacktest (MIT) as the P4 reference.
3. **Use other frameworks as test oracles, not dependencies.** Run reference strategies through Ubik and, privately, through Freqtrade, NautilusTrader and vectorbt, then compare the trades they produce. This differential testing catches simulator bugs. Using code privately for testing doesn't distribute it, so their licenses don't affect Ubik.
4. **Spike before committing (2 weeks, start of P1).** Implement one reference strategy (an SMA crossover with volatility-targeted sizing on BTC/USDT, 1-minute data aggregated to 1-hour) twice:
   - (a) on a minimal Ubik engine prototype;
   - (b) on NautilusTrader v2.

   Score both on:
   - lines of glue code;
   - ability to emit plain-language decision traces;
   - paper trading on live data;
   - time to add a custom portfolio risk rule;
   - API breakage during the spike.

   If Nautilus wins clearly, adopt it as the engine *behind Ubik's own strategy API* (LGPL permits this).
5. **Revisit trigger:** re-evaluate NautilusTrader when 2.0 final has shipped and gone three months without breaking changes, or when P4 order-book or market-making work begins.

---

## 5. Requirements (CAP)

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| CAP-001 | For every dataset used by a backtest or paper session, Ubik shall record the source exchange, symbol, timeframe, time range, retrieval time and data-quality results (gaps, duplicates, outliers), and shall show detected gaps before a backtest runs. | Must | P1 | Pros treat data quality as step zero. Storage is covered in 06. |
| CAP-002 | Re-running a saved backtest with the same strategy version, configuration and dataset snapshot shall produce identical trades and metrics. | Must | P1 | Determinism underpins trust, debugging and parity checks. |
| CAP-003 | When a backtest universe is defined by a rule (e.g. "top 10 by volume"), Ubik shall include pairs that were delisted or renamed during the period. | Should | P2 | Avoids survivorship bias (only counting the survivors). P1 uses a fixed list of major pairs. |
| CAP-004 | Before a strategy can be promoted past Backtested, Ubik shall require a plain-language hypothesis for it: why it should work, and when it should fail. | Should | P1 | Pro research starts from a reason, not a curve. Prompts are designed in 07. |
| CAP-005 | For each strategy, Ubik shall count and display every backtest and parameter variant ever run, including those on retired versions (the "trial count"). | Must | P1 | Input to the Deflated Sharpe Ratio; also a strong honesty signal. |
| CAP-006 | By default, Ubik shall reserve the most recent part of history as out-of-sample data, report in-sample and out-of-sample results separately, and warn when the out-of-sample period has been evaluated more than a set number of times. | Must | P1 | Pro "held-out data" discipline. Defaults are set in 04. |
| CAP-007 | Ubik shall provide walk-forward validation and report the Deflated Sharpe Ratio and Probability of Backtest Overfitting, each with a plain-language verdict. | Must | P2 | Core differentiator. Algorithms are in 04. |
| CAP-008 | Every backtest, paper and live report shall show, for the same period and net of costs, the result of buy-and-hold on the traded asset(s) and of holding cash. | Must | P1 | Many "profitable" bots underperform simply holding BTC. |
| CAP-009 | Ubik shall automatically detect lookahead bias by recomputing signals on truncated data and comparing the results, and shall block promotion when bias is found. | Must | P1 | The most common silent backtest bug. Freqtrade shows the check is feasible. |
| CAP-010 | Ubik shall run every candidate strategy through historical stress scenarios (at minimum: Mar 2020, May 2021, Nov 2022 FTX, 10 Oct 2025) and report the drawdown and recovery for each. | Should | P2 | Beginners otherwise judge strategies on calm periods. Depends on data availability (06). |
| CAP-011 | Ubik shall report Monte-Carlo or bootstrap ranges (e.g. 5th–95th percentile) for return and max drawdown alongside point estimates. | Should | P2 | Communicates uncertainty honestly. |
| CAP-012 | A strategy shall be able to combine multiple signals (AND/OR rules in P1, weighted scores or votes from P2), and Ubik shall report each signal's standalone and marginal contribution. | Should | P2 | The retail-scale version of signal combination. |
| CAP-013 | Signal generation shall be separate from position sizing, so the same strategy version can run under different sizing policies (fixed amount, % of equity, volatility-targeted) without code changes. | Must | P1 | Pro practice; enables risk-based defaults (05). |
| CAP-014 | Ubik shall allocate capital across concurrently running strategies using a selectable policy (equal capital, or equal risk / inverse volatility) with per-strategy caps, and shall show the correlation between strategies' returns. | Should | P2 | Multi-strategy portfolios arrive in P2. |
| CAP-015 | In every mode, every order shall pass through a risk engine that strategies cannot bypass. It shall enforce at least: max order value, max position per asset, max total exposure, a price-band (fat-finger) check and a max order rate. Rejected orders shall be logged with the reason. | Must | P1 | Mirrors pro pre-trade controls. Parity requires the engine in backtests too. Limits are set in 05. |
| CAP-016 | Ubik shall enforce account-level daily-loss and max-drawdown limits, on by default, that pause all strategies and cancel open orders when breached. | Must | P1 | Safety by default. |
| CAP-017 | Ubik shall provide a kill switch, reachable within two clicks or taps from any screen, that halts all strategies and cancels all open orders. In live mode it shall offer an optional "flatten positions" step. | Must | P1 | Present in paper mode too, so the habit and the code path are proven before P3. |
| CAP-018 | Ubik shall show capital per exchange and per stablecoin / quote asset, and shall warn when either exceeds a user-set cap. | Should | P3 | Crypto-desk counterparty practice (FTX, the Bybit hack, the USDe depeg). |
| CAP-019 | Simulated execution shall apply the exchange's actual fee schedule for the user's tier and model spread and slippage with conservative defaults. Any zero-cost simulation shall be visibly labeled as unrealistic. | Must | P1 | Fees and spread dominate retail costs. Models are in 04 and 05. |
| CAP-020 | For every order, Ubik shall record the decision price, submission time and price, fill prices, fees and latency, and shall compute per-trade implementation shortfall and an aggregate TCA report. | Must | P1 | Paper in P1, live in P3. Feeds slippage calibration. |
| CAP-021 | Ubik shall alert when realized paper or live slippage exceeds the backtest model by more than a configurable threshold, and shall offer to recalibrate the model. | Should | P2 | Closes the TCA feedback loop. |
| CAP-022 | For each strategy, Ubik shall show annualized fee drag (fees plus slippage as a % of capital, given the strategy's turnover), and shall warn when the average gross edge per trade is smaller than the estimated round-trip cost. | Must | P1 | The most important honesty number for retail traders (§1.3). |
| CAP-023 | Ubik shall support execution styles selectable per strategy: at least market, limit, post-only with timeout-then-cross, and time-sliced (TWAP). | Should | P3 | Order mechanics are in 05. |
| CAP-024 | Ubik shall split each strategy's P&L into market (beta to BTC or the traded asset), strategy contribution, fees and slippage, per strategy and per asset. | Should | P2 | Performance attribution at retail scale. |
| CAP-025 | For any strategy running in paper or live mode, Ubik shall re-simulate the same period on the recorded market data and show trade-by-trade and P&L differences with their attributed causes (fills, latency, data, fees). | Must | P1 | Measures backtest-live parity instead of assuming it. A key differentiator. |
| CAP-026 | Ubik shall monitor data freshness, engine heartbeat, exchange connectivity and clock drift. On a breach, it shall pause the affected strategies and alert the user in-app and via at least one push channel. | Must | P1 | Fail safe. Notification channels are covered in 06 and 07. |
| CAP-027 | Ubik shall keep an append-only audit log of data checkpoints, signals, risk decisions, orders, fills, configuration changes and user actions that is sufficient to replay any trading day deterministically. | Must | P1 | The basis for explanations, incident review and tax records (08). |
| CAP-028 | In live mode, at startup and at a configurable interval, Ubik shall reconcile its orders, positions and balances with the exchange; halt on any unexplained mismatch; and alert on any order or trade it did not originate. | Must | P3 | Would quickly surface 3Commas-style key abuse. |
| CAP-029 | Any code or parameter change to a live strategy shall create a new version that must pass through Paper, or a logged explicit override, before it trades real money. | Must | P3 | The Knight Capital lesson. |
| CAP-030 | Ubik shall support incident records (impact, root cause, follow-ups) linked to the affected strategies and log entries, and shall prompt for one after any automatic live halt. | Could | P3 | A lightweight post-mortem habit. |
| CAP-031 | Ubik shall version every strategy (code, parameters, sizing and risk configuration) and tie every backtest, paper and live result to exactly one version. | Must | P1 | Prerequisite for governance and reproducibility. |
| CAP-032 | Ubik shall implement the lifecycle Idea → Backtested → Validated → Paper → Live-Pilot → Live → Retired with objective, configurable promotion gates (e.g. minimum out-of-sample length and trade count, overfitting threshold, minimum paper duration and parity) and safe defaults. Live states unlock in P3. | Must | P1 | "Investment committee in a box". Gate defaults are set in 04 and 05. |
| CAP-033 | Each strategy shall declare kill criteria before entering Paper (default example: drawdown greater than 1.5× its worst out-of-sample drawdown). A breach shall auto-pause the strategy and require an explicit review to resume. | Must | P1 | Pre-commitment beats rationalizing in the moment. |
| CAP-034 | Ubik shall compare live and paper return distributions with backtest expectations and flag statistically significant decay. | Should | P2 | Monitoring for model decay. |
| CAP-035 | Real-money capital for a newly promoted strategy shall start at a small pilot amount set by the user, and shall increase only through explicit, logged steps after the pilot criteria are met. | Must | P3 | Staged capital ramp, as pros do. Detail in 05. |
| CAP-036 | Ubik shall never request, store or use API keys with withdrawal or transfer permission, and shall store trade keys only on the owner's infrastructure, encrypted at rest. | Must | P3 | The 3Commas lesson. Detail in 08. |
| CAP-037 | Ubik shall export all user data (strategies, versions, results, orders, fills, audit logs) on demand in open formats (CSV, JSON, Parquet). | Must | P1 | The platform-shutdown lesson; also needed for tax (08). |
| CAP-038 | Ubik shall ship templates for DCA, grid, trend-following, mean-reversion and breakout strategies. Each shall include a plain-language "when this fails" section and a stress scenario that demonstrates the failure. | Must | P1 | Competitors hide these failure modes. Template list is in 04. |
| CAP-039 | Every performance figure shall be labeled with its provenance (in-sample, out-of-sample, walk-forward, paper, live) and cost basis (gross or net). In-sample-only results shall never be shown unlabeled. | Must | P1 | Makes the honesty principle testable. |
| CAP-040 | Ubik shall accept external signals via authenticated webhook (e.g. TradingView alerts). These signals shall still pass through the risk engine and lifecycle gates. | Could | P2 | Common competitor feature. |
| CAP-041 | Strategies or parameter changes generated by the AI assistant shall be treated as new versions subject to the same gates. The assistant shall not place, modify or cancel orders without explicit user confirmation. | Must | P2 | LLM traders have proven erratic. Assistant UX is in 07. |
| CAP-042 | Ubik shall flag strategies whose design needs sub-second reaction (e.g. holding periods under 1 minute) and explain why retail infrastructure can't compete there. | Should | P1 | Encodes the non-goals in §1.4. |
| CAP-043 | Ubik shall offer market making and cross-exchange arbitrage only as simulations that model fee tier, latency, queue position and transfer delay, and shall not run them live without a separate review. | Could | P4 | Teaches why these strategies are pro-only. |
| CAP-044 | Shipped code shall use only permissive (MIT, BSD, Apache-2.0) or LGPL-as-library dependencies, enforced by an automated license and vulnerability check, unless the owner explicitly accepts an exception. | Must | P1 | Keeps the P4 commercial / multi-user path open and reduces supply-chain risk. |

---

## Open questions for the owner

1. **Will Ubik ever be distributed or sold** (open-source release, paid download, SaaS)? This decides how strict the license policy needs to be (CAP-044). If the answer is "never, personal only", GPL tools like Freqtrade become acceptable to embed.
2. **Which exchanges do you actually use, and where are you resident?** Jurisdiction decides which venues are available (for example, US residents can't use Binance.com or Bybit) and which fee schedules apply. That in turn drives the fee-drag math in §1.3.
3. **Will you write Python strategies yourself, or only use the no-code builder?** This affects how much weight the code-first strategy API gets compared with the builder.
4. **What holding period and capital do you have in mind for P3** (minutes, hours or days; hundreds or tens of thousands of dollars)? This decides whether execution optimization beyond "minimize fees" matters at all.
5. **Would you accept an interim tool?** For example, running Freqtrade in dry-run to learn while Ubik is built. This is fine for personal use regardless of its GPL license.
6. **How much autonomy do you want the AI assistant to have?** The recommendation here is "propose and explain, never act without confirmation".
7. **Is there a budget for paid data** (e.g. tick or order-book history) if a validated strategy needs it?

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **Scope creep from "everything trading firms do"** | Ubik never ships | Phase-tagged capability map; explicit non-goals (§1.4); copy pro *process* first |
| **Bugs in our own simulator create false confidence** | Bad strategies get promoted | Differential testing against Freqtrade, Nautilus and vectorbt; golden-file tests; mandatory parity reports (CAP-025) |
| **Wrong engine choice** (own core vs NautilusTrader) | Rework in P3 or P4 | 2-week spike; clean adapter boundaries; defined revisit trigger (§4.4) |
| **"Pro-grade tools" read as "pro-grade returns"** | Users over-trust Ubik and size up | Honest labeling (CAP-039), benchmark (CAP-008), fee drag (CAP-022), education (07) |
| **Free exchange-native bots out-compete Ubik on convenience** | Low perceived value | Compete on honesty, learning and control, not convenience; integrate rather than replicate where sensible |
| **A copyleft or Commons-Clause dependency slips in** | Blocks a later commercial path | Automated license gate in CI (CAP-044) |
| **Supply-chain malware in trading-related packages** | Stolen keys or funds | Lockfiles, dependency review and sandboxing (08) |
| **Regulatory drift** (signals, copy-trading, AI advice, SaaS hosting other users' funds) | Legal exposure | Kept out of scope until legal review (08) |
| **Competitor facts go stale** (prices, shutdowns) | Wrong decisions | Re-verify at each phase gate; several prices here come only from third-party sources and are marked as such |
