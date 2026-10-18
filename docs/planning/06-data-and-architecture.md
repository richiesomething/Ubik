# 06 — Data & Architecture

*Ubik planning document · draft 2026-10-18 · ID prefixes: `DA` (data), `AR` (architecture & tech stack), `NFR` (non-functional requirements).*

## Summary

- **Pick exchanges by the owner's jurisdiction profile. In P1 this is mostly a choice of data source.** Outside the US and the EU/EEA: **Binance** (best free history, Demo Mode, 0.10% fees) plus **Coinbase Advanced or Kraken** as a regulated second venue. In the EU/EEA: venues on ESMA's register of authorised crypto-asset service providers (CASPs) for the owner's country, quoted in USDC or EUR. In the US: **Coinbase Advanced + Kraken**. Binance.com blocks US IPs: we got HTTP 451 from a US address during research. Paper trading uses Ubik's own simulator. Exchange demo accounts are only used to test adapters before going live (stage S2, P3).
- **Ubik owns a thin trading kernel built on ports and adapters** (doc 10 D-02). The Clock, the data feed and the execution venue are swappable parts, so backtest, paper and live are combinations of the same code. One fill simulator (`SimVenue`) serves both backtest and paper. Exchange access goes through Ubik's `MarketDataFeed`/`ExecutionVenue` ports, implemented with **ccxt** (MIT; WebSocket support now included free). NautilusTrader gets a time-boxed design spike, not an adoption decision.
- **Use two storage roles.** **PostgreSQL (+TimescaleDB Community)** is the system of record (orders, fills, ledger, config, audit) and holds recent candles. A **Parquet + DuckDB data lake** of files holds history and immutable backtest datasets. Measured: 1-minute candles cost about 17 MB per pair-year. ClickHouse or QuestDB only comes in if we later record full order books at scale.
- **Run a modular monolith with three process roles from one Python codebase**: `engine` (always on), `api` and `worker`. From P2 a `sandbox` role runs any user-supplied code, and in P3 the execution `gateway` becomes the only holder of exchange keys (doc 08). Processes talk over Valkey Streams, and the browser gets live updates over SSE. Everything runs under Docker Compose on a 4 vCPU / 8 GB box for **~$0–60/month** in P1.
- **The proposed stack is confirmed, with adjustments.** Engine: Python 3.13, asyncio, FastAPI, **polars 2.x** (not pandas) and **TA-Lib 0.8.2** (pinned; its wheels have bundled the C library since 0.6.5). Avoid **pandas-ta**: its repo is gone and its last release was a beta in Sep 2025. UI: TypeScript/React with **TradingView Lightweight Charts 5** (Apache-2.0, attribution required).
- **Correctness comes before speed**: Decimal ledger, orders persisted *before* sending, UTC microsecond timestamps, explicit data gaps, point-in-time data, and golden/parity/replay tests in CI.

---

## 1. Scope and position on working assumptions

Scope here: exchanges, connectivity, data, storage, architecture, stack, data model, operations, NFRs. Related topics live elsewhere:

- Backtester and fill-model semantics: [04](04-strategies-backtesting-paper-trading.md)
- Risk, OMS and ledger rules: [05](05-risk-portfolio-execution.md)
- Screens and assistant UX: [07](07-ux-and-beginner-experience.md)
- Key storage, auth and legal: [08](08-security-legal-compliance.md)

Refinements to the lead's assumptions:

1. **"Postgres + time-series extension *or* columnar store"**: I recommend **both**, in separate roles. Postgres handles state and hot data; Parquet/DuckDB handles history and reproducible backtests.
2. **Python: yes. pandas as the core dataframe: no.** Use polars and numpy, and drop pandas-ta (§8).
3. **"Live data in paper mode": agreed. Ubik should also *record* it from day one.** For Coinbase and Kraken there is no free bulk order-book or candle history, so recording is the cheap way to get venue-specific data for realistic fills.
4. **Single-user: agreed, but put a `workspace_id` on user-owned tables now.** It costs nothing today. Retrofitting for P4 multi-user would be expensive.
5. **Jurisdiction is an architecture input.** It decides which endpoints are reachable and where a server may legally run.

---

## 2. Exchange selection

Terms used below:

- **REST**: request/response HTTP calls.
- **WebSocket (WS)**: a persistent connection where the exchange pushes updates.
- **Rate limit**: how many calls the exchange allows per time window.
- **Exchange demo**: an exchange-hosted fake-money environment that mirrors the live market (e.g. Binance Spot Demo Mode). Ubik uses it only for stage S2 (P3).
- **Testnet**: a separate practice market with its own prices and liquidity. Fine for testing API calls, but never used to judge a strategy or for graduation. Some exchanges call their test environment a "sandbox"; that is unrelated to Ubik's code sandbox (AR-037).
- **Maker / taker fee**: the fee for an order that rests on the book vs. one that fills immediately.

"Verify" marks figures not confirmed from a primary source.

| Exchange | API | Testnet / demo | Historical data | Spot fee, entry tier (maker/taker) | Rate limits | Availability |
|---|---|---|---|---|---|---|
| **Binance** | Excellent: REST, WS, WS trading API, FIX | Spot Testnet (separate prices/liquidity, monthly reset). **[Demo Mode](https://developers.binance.com/docs/binance-spot-api-docs/demo-mode/general-info)** (`demo-api.binance.com`, same features as live) | **Best free source.** [data.binance.vision](https://data.binance.vision/) bulk klines down to 1s, trades and aggTrades since 2017, futures funding/bookDepth/metrics. REST: 1,000 klines per call | 0.10% / 0.10% (0.075% with BNB) ([schedule](https://www.binance.com/en/fee/schedule)) | Per IP: 6,000 weight/min; orders 100 per 10 s and 200,000/day (read from `exchangeInfo`, 2026-10-18) | **Not for US persons.** HTTP 451 from a US IP during research. Restricted in other places too (e.g. AMF delisting in France, Jul 2026, per press). Verify per country |
| **Binance.US** | Separate company with a smaller API | Verify | Limited | Verify | Verify | [Some US states only](https://support.binance.us/en/articles/9842798-list-of-supported-states-and-regions) (e.g. no NY or TX) |
| **Coinbase Advanced** | Good: REST and WS. Public REST is [cached for 1 s](https://docs.cdp.coinbase.com/coinbase-app/advanced-trade-apis/rest-api) | [Sandbox](https://docs.cdp.coinbase.com/coinbase-app/advanced-trade-apis/sandbox) is **static and mocked** (accounts/orders only) | REST only, [350 candles per call](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/products/get-product-candles). No bulk dumps | Entry tier unverified: **0.60% / 1.20%** (under $1K per 30 days, per Bankrate and NerdWallet) or ~0.40% / 0.60% (other sources; lowered [Sep 2026](https://cryptodaily.co.uk/2026/09/coinbase-advanced-fees-usdc-vip-tiers)). Ubik assumes 0.60% / 1.20% until the owner's account confirms (AR-002) | Verify | US (incl. NY) and many countries. Publicly listed and regulated |
| **Kraken Pro** | Good: REST, WS v2, FIX | **No spot sandbox** (futures demo only) | OHLC returns only the **last 720 candles**. Trades can be paged through full history. [No bulk dump](https://docs.kraken.com/exchange/guides/general/historical-data) | **0.40% / 0.80%**, falling to 0.22% / 0.38% from $10K ([fees](https://www.kraken.com/features/fee-schedule)) | Per IP, ~1 req/s for history | US (not NY/ME per third-party lists), EU, UK, CA, AU |
| **Bybit** | Good (v5) | Testnet and [Demo](https://bybit-exchange.github.io/docs/v5/demo) (`api-demo.bybit.com`) | REST klines | ~0.10% / 0.10% (verify) | [600 req per 5 s per IP](https://bybit-exchange.github.io/docs/v5/rate-limit) | **No US/Canada.** EEA via Bybit EU (MiCA). Third-party reports say UK spot returned Dec 2025 |
| **OKX** | Good (v5) | [Demo](https://www.okx.com/docs-v5/en/) via header `x-simulated-trading: 1` | REST candles | ~0.08% / 0.10% (verify) | Per endpoint, per 2 s | Regional hosts `us.okx.com` and `eea.okx.com`. [OKX US](https://www.theblock.co/post/355107/okx-rebuilding-super-app-in-the-us-ceo-roshan-robert) is spot-only and excludes NY and TX |
| **Bitstamp** | Mature. [Owned by Robinhood since Jun 2025](https://robinhood.com/us/en/newsroom/robinhood-completes-acquisition-of-bitstamp/) | Verify | Long REST history | Higher (verify) | Verify | EU, UK, US |
| **Newer** (Hyperliquid, Bitget, MEXC) | Varies | Varies | Varies | Low | Varies | Derivatives focus or weaker regulatory standing. **Not P1.** Hyperliquid is a P4 candidate |

**Takeaways**

- **In P1 the exchange is a data dependency.** Fills are simulated, so what matters is free history plus a reliable WS feed *for the venue we'd later trade on*. Backtesting on Binance prices and then trading on Coinbase adds a cross-venue price gap ("basis") the backtest never saw.
- **Exchange sandboxes are uneven.** Binance, Bybit and OKX demos are realistic. Coinbase's sandbox is mocked, and Kraken has no spot sandbox. **So paper mode must be Ubik's own simulator** (AR-012). Exchange demos are only used for adapter contract tests and stage S2 (P3), never as evidence of strategy performance.
- **Entry-tier taker fees range 8–12×** (0.10% to 0.80%, or 1.20% if Coinbase Advanced's lowest tier applies). The cost model must be per venue and per tier, or backtests will mislead. The venue registry (AR-002) is the single source of fee schedules.

**Recommendation**

| Owner's jurisdiction | Primary | Secondary | Note |
|---|---|---|---|
| EU/EEA | A venue on ESMA's CASP register for the owner's country (Coinbase and Kraken report MiCA authorisations; verify Binance's status per country) | Another venue on the register | Quote in USDC or EUR: USDT pairs are generally unavailable to EEA users (doc 08 §8.2) |
| UK, most of APAC/LatAm where Binance is permitted (non-EU, non-US) | **Binance** | **Coinbase** or **Kraken** | Check country-level Binance status first |
| United States | **Coinbase Advanced** | **Kraken** | Backfill Coinbase via REST (1 year of 1m ≈ 1,500 calls). `data-api.binance.vision` was reachable from a US IP, but until doc 08 §8.5 confirms who may use it (⚖), the US profile backfills only from its own venues (Coinbase REST, Kraken trades) |
| Canada / other restricted | Decide after doc 08 | Kraken / Coinbase | — |

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| AR-001 | Ubik shall select venues through the jurisdiction profile (one entity, the same as doc 08 CMP-004), without code changes. Default profiles (EU/EEA: venues on ESMA's CASP register for the owner's country, quoted in USDC/EUR; other non-US: Binance + Coinbase or Kraken; US: Coinbase Advanced + Kraken). P1 implements the chosen profile's two venues: the intended live venue is Must, the second venue Should. | Must | P1 | Jurisdiction unknown. Each deployment needs at most two venues, matching doc 00 §6's "1–2 exchanges". |
| AR-002 | Ubik shall keep a venue registry of each venue's technical fields: order types, fee tiers, rate limits, demo/regional hosts, key-permission endpoint and field mapping (doc 08 SEC-008/SEC-010), and quirks. Adapters, the cost model and the UI shall all read from it. | Must | P1 | The single source of fee schedules: other docs cite it instead of hard-coding numbers. Each fee entry records its source and date. An unverified entry tier defaults to the higher, conservative rate until the owner's account confirms it (Coinbase Advanced 0.60% / 1.20%; Kraken 0.40% / 0.80%). Doc 08 CMP-003 adds the compliance fields to this same registry. |
| AR-003 | Ubik shall treat geo-block responses (HTTP 451/403 with region messages) as configuration errors: stop retrying, explain to the user, and never route around them. | Must | P1 | Legal safety (doc 08). |
| AR-004 | Before live trading on a venue, its adapter shall pass a contract-test suite against the venue's exchange demo (stage S2, doc 05 §9.1). Where the venue has no realistic spot demo (Coinbase's sandbox is mocked; Kraken has none), the suite runs against the mocked sandbox where one exists and then with minimum-size live orders as the first step of S2, together with fault-injection tests on the paper venue (doc 05 EX-021). | Must | P3 | Proves parity for each venue. This is doc 05 §9.1's S2 variant. Testnets are not used for graduation. |

---

## 3. Connectivity layer

| Option | Status (Oct 2026) | Pros | Cons |
|---|---|---|---|
| **ccxt** | v4.5.88 (14 Oct 2026), **MIT**. ~100 exchanges. **WebSocket "Pro" is bundled free** ([docs](https://docs.ccxt.com/docs/pro)). `setSandboxMode` for testnets, `enableDemoTrading` for Binance Demo | One unified REST+WS API in Python/JS. Handles signing, precision and market metadata | Lowest-common-denominator abstraction, so order types, errors and partial-fill semantics leak through. Several releases a week means regression risk |
| Native SDKs | Exchange-maintained | Full feature coverage | One integration per venue. Uneven quality |
| Framework adapters (NautilusTrader, Hummingbot) | §8 | Battle-tested order-state handling | Ties us to that framework's domain model |

**Design.** Ubik defines **ports** (interfaces the core depends on). **Adapters** implement them. Strategies never see ccxt.

```text
MarketDataFeed:  subscribe(instruments, channels) -> stream[Bar | Trade | BookTop | BookDelta]
                 history(instrument, timeframe, start, end) -> Frame
ExecutionVenue:  submit(OrderIntent{client_order_id}) -> Ack ; cancel(id) ; amend(...)
                 order_status(client_order_id) ; open_orders() ; fills(since) ; balances()
                 stream_account() -> stream[OrderEvent | Fill | BalanceEvent]
Clock:           now() -> UTC µs ; schedule(at, callback)
```

Implementations:

- **`CcxtFeed`**: live public market data via ccxt.
- **`SimVenue`**: the one matching engine (fill simulator) for backtest (history) and paper (live book and trades). The "paper venue" is `SimVenue` fed by the live feed.
- **`HistoricalFeed`**: replays lake snapshots.
- **`LiveVenue`** (P3): the engine-side `ExecutionVenue` that forwards risk-approved intents to the execution `gateway` (§7.1). Pointed at demo endpoints and keys, it also serves the exchange demo (stage S2, doc 05 EX-023).
- **`CcxtVenue`** (P3): exchange access inside the gateway, via ccxt.
- **`NativeVenue`**: a native-SDK adapter inside the gateway, only where ccxt falls short (P3+).

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| AR-005 | Strategy, risk and OMS code shall reach exchanges only through the `MarketDataFeed`/`ExecutionVenue` ports, never by importing ccxt or SDKs (CI-enforced). Native-SDK adapters may be added behind the same ports. | Must | P1 | Parity; swappable library. |
| AR-006 | Adapters shall pin an exact ccxt version. Upgrades shall merge only after contract tests and recorded-session replay tests pass. Each enabled venue's API changelog shall be watched, with an alert on announced breaking changes. | Must | P1 | Release churn; venue API churn (doc 05 §5). |
| AR-007 | Adapters shall normalize symbols (`BASE/QUOTE@VENUE`), numbers (Decimal parsed from strings), timestamps (UTC µs), errors (retryable / rate-limited / rejected / insufficient-funds / **unknown-state**) and order states (doc 05). | Must | P1 | "Unknown-state" triggers reconciliation. |
| AR-008 | Ubik shall rate-limit itself per venue and per IP, staying ≤ 70% of published limits (doc 05 EX-006, which owns the rule), and shall back off on 429/418/Retry-After. | Must | P1 | The headroom keeps capacity for cancels and protective exits. Binance bans IPs that keep retrying. |
| AR-009 | WS connections shall be supervised: heartbeats, reconnect with backoff and jitter, resubscription, order-book sequence-gap detection (forcing a fresh snapshot), and REST backfill of missed candles. | Must | P1 | Feeds drop regularly. |

---

## 4. Data sources and costs

Terms used below:

- **OHLCV**: open-high-low-close-volume candles.
- **Trades**: individual executions.
- **L2 order book**: resting orders aggregated by price level.
- **Funding rate**: periodic payments between perpetual-futures longs and shorts.
- **Open interest (OI)**: the number of open futures contracts.

| Data class | Free (use first) | Paid upgrade (2026 list prices) | License notes |
|---|---|---|---|
| OHLCV | Exchange REST/WS. Binance bulk dumps. Kraken trades→bars. Coinbase REST | [CoinAPI](https://www.coinapi.io/market-data-api/pricing) Startup $79/mo (pay-as-you-go with $25 free credit). Kaiko/Amberdata (enterprise quotes) | Personal use. Check exchange ToS before republishing |
| Trades | Binance bulk `trades`/`aggTrades`. Self-recorded WS. Kraken trades | [Tardis.dev](https://tardis.dev/#pricing) Spot Solo $900/mo, Academic $450/mo. **First day of every month is free without a key** (we tested this on 2026-10-18) | Tardis: no redistribution |
| L2 order book | **Self-record** top-20 snapshots. Binance futures `bookDepth`. Tardis free samples | Tardis, Kaiko, CoinAPI | Same |
| Funding / OI / liquidations | Exchange REST (Binance, Bybit, OKX public APIs, where reachable). Binance futures dumps. Coinalyze (free tier, verify) | [CoinGlass](https://www.coinglass.com/pricing) Hobbyist $29/mo (personal, 30 req/min), Standard $299 (commercial) | Personal plans: non-commercial |
| Market cap / metadata | [CoinGecko Demo](https://www.coingecko.com/en/api/pricing): free, 100 calls/min, 10K credits/month. Its `/global` endpoint gives current dominance only, so Ubik snapshots it daily (DA-024) | Basic $35/mo, Analyst $129/mo | **Attribution required**. No redistribution. CoinDesk Data (ex-CryptoCompare) [ended its free tier on 21 May 2026](https://coinmarketcap.com/api/resources/coindesk-api-alternatives-7-crypto-data-feeds-compared-in-2026) |
| On-chain | DefiLlama API (verify terms). Coin Metrics Community (CC BY-NC). mempool.space | Glassnode (API mostly on top tiers or quote; ~$49–$999/mo per trackers, verify). CryptoQuant (~$29–$109/mo, Premium ~$799 per [third-party](https://costbench.com/software/onchain-analytics/cryptoquant/), verify) | Metrics get **revised**, so store point-in-time (DA-017). Coin Metrics Community is non-commercial |
| Implied volatility | Deribit public API (DVOL) | Option chains: P4 | Verify terms |
| Sentiment | [alternative.me Fear & Greed](https://alternative.me/crypto/fear-and-greed-index/) (free, daily). Google Trends (P4) | Santiment, LunarCrush (verify) | Attribution must sit next to the data |
| News | Publisher RSS (check terms). GDELT. Exchange announcements | [CryptoPanic API](https://cryptopanic.com/developers/api/) (pricing unverified) | Store headlines and links only |
| Macro | FRED (free key; ALFRED keeps revision history). ETF flows: Farside, SoSoValue | — | Use those revision snapshots for point-in-time. Verify ETF-flow terms |
| Event calendar | P1: Fed and BLS schedules, Bitcoin block height, manual entries. P2: Tokenomist web | Tokenomist API (paid or trial) | Store `scheduled_at` and `announced_at` (DA-025) |

**Budget:** P1 is **$0**. P2 extras are optional, ~$30–150/mo. Tardis is a P3/P4 decision.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| DA-001 | Ubik shall ingest live 1m candles and trades (or the best available ticker) over WS for configured instruments, with REST polling as fallback. | Must | P1 | Live data even in paper. |
| DA-002 | Ubik shall backfill ≥ 5 years of 1m candles where available for the enabled venues of the jurisdiction profile (AR-001), using each venue's loader: Binance bulk files with checksum verification, Coinbase REST paging, or Kraken trades→bars. | Must | P1 | Backtest history. In P1, loaders are built only for the enabled venues. |
| DA-003 | From first start, Ubik shall record its own venue data: finalized candles, plus trades and top of book for instruments with active bots (doc 04 PT-010); and top-20 L2 snapshots every second plus per-minute depth bands (±0.5/1/2% around mid) for those instruments, capped at 5 instruments in P1. | Must (L2 and depth bands: Should) | P1 | Builds venue-specific slippage data that isn't free, and feeds parity replays. The cap keeps NFR-014's 100 GB (§6.1). Retention per NFR-011. |
| DA-004 | Ubik shall snapshot instrument rules (tick, lot, minimum order value, status) daily and keep their history. | Must | P1 | Backtests use the rules in force at the time. |
| DA-005 | Ubik shall record stablecoin→USD rates and label "USD" figures that are actually USDT/USDC. | Should | P1 | Depeg honesty. |
| DA-006 | From first start, Ubik shall record funding (predicted and settled), OI and long/short ratios for BTC/ETH/SOL perpetuals from free exchange sources, on at least one venue that the jurisdiction profile can reach (AR-003). | Should | P1 | Doc 03 MA-011: free history is short (e.g. 30 days of OI), so it must be recorded early. Display stays P2 (MA-012). |
| DA-007 | Each external source shall be a `DataSource` plugin with caching, quota tracking and an off switch. Paid sources are off by default and show estimated cost. If a source fails, dependent features show "unavailable since …" instead of stale values. | Should | P2 | Free tiers vanish. |
| DA-008 | Every data source shall have an entry in the data-source registry (doc 08 CMP-007, which owns it): attribution, commercial use, redistribution and retention. The UI shall show required attributions, and exports shall exclude data that may not be redistributed. | Must | P1 | CoinGecko, alternative.me and TradingView require attribution; Coin Metrics Community is non-commercial. One registry: CMP-007 absorbs this requirement. |
| DA-023 | At first launch Ubik shall start backfilling a quick-start dataset in the background (the primary venue's BTC pair, full available 1m history) from bulk files or a pre-seeded snapshot, so that it is ready by the end of onboarding. | Must | P1 | Lets the first template backtest finish within 10 minutes of first launch (doc 07 UX-004) without waiting for the full DA-002 backfill. REST-only venues (Coinbase, US profile) need the pre-seeded snapshot, because ~1,500 REST calls per year of 1m is too slow; the snapshot must respect the source's licence (DA-008). |
| DA-024 | Ubik shall snapshot CoinGecko `/global` (BTC dominance, total market cap, stablecoin market cap) daily from first start, with its attribution recorded in the data-source registry (DA-008). | Should | P1 | Doc 03 MA-007: the history endpoint is paid, so Ubik records its own. |
| DA-025 | Ubik shall store calendar events with `scheduled_at`, `announced_at`, importance, source and affected assets, from doc 03 §3.9's P1 sources (Fed and BLS schedules, Bitcoin block height, manual entries). | Should | P1 | Doc 03 MA-019; P2 sources follow. Backtests see an event only from its `announced_at`. |

---

## 5. Data quality and time

Bad data is the most common reason a backtest flatters a strategy. This section is the data contract for every other module.

**Time and candles**

- All timestamps are **UTC with µs precision**. The UI converts to local time for display only.
- Candles are labelled by **open time** and cover [open, open + timeframe).
- Higher timeframes are built from 1m in UTC. Days close at 00:00 UTC; weeks start Monday 00:00 UTC.
- A candle is **final** only when the venue flags it closed (Binance WS kline `x: true`), or at close + 2 s grace when polling.
- **Unit traps are real.** Binance bulk files switched from **milliseconds to microseconds in January 2025** (we checked the Dec 2024 and Jan 2025 files). A naive loader would be wrong by 1000×.

**Validation at ingest**

| Check | Rule | On failure |
|---|---|---|
| Parse | All fields present. Prices > 0 | Reject and log |
| OHLC invariants | low ≤ min(open, close) ≤ max(open, close) ≤ high; volume ≥ 0 | Quarantine |
| Alignment | open_time is a multiple of the timeframe. Not more than 5 s in the future | Reject |
| Duplicates | Unique (instrument, timeframe, open_time) | Upsert and log the revision |
| Continuity | Missing candle → `DATA_GAP` record | Backfill. **Never forward-fill silently** |
| Outliers | Return beyond k × rolling MAD, or > X% from the cross-venue median | Flag as `suspect` and keep the row |
| Reconciliation | REST vs. WS vs. bulk vs. rebuilt-from-trades | Diff report. One canonical source per venue |
| Candle-feed gap | Candle age > 2 × timeframe + grace (threshold configurable per series) | Ingest-quality flag only. The trading rule for stale data (> 10 s without trades or quotes, a book-sequence gap or a missing 1m bar → new entries denied) is doc 05 §4.4 / RK-026 |

**Survivorship and identity.** "Survivorship bias" means testing only on coins that still exist, which flatters results. Ubik keeps delisted instruments (LUNA/UST and FTT in 2022, BUSD pairs in 2023–24). It builds universes point-in-time: "top 10 by volume" is computed *as of each date*. Rename and migration aliases (e.g. MATIC→POL, 2024) keep history chains intact.

**Point-in-time storage.** Every stored series uses doc 03 §6's field names (TA-026): `event_time` (the moment the value describes), `available_at` (the earliest moment a live system could have known it), `first_seen_at` (when Ubik itself stored it) and, for revisable data, `revision_no`. Some data is revised after publication: on-chain metrics, sentiment, macro, rankings, corrected candles. A revision is a new row with the next `revision_no`, never an edit. News keeps `published_at` but uses `ingested_at` as its `available_at`, because publishers edit and backdate timestamps (doc 03 MA-015). Backtests read only `available_at ≤ decision time`. Anything else is lookahead: using information that didn't exist yet.

History backfilled before Ubik began recording gets these conservative `available_at` defaults (the final values doc 03 §6 asks doc 06 to set):

| Data | `available_at` |
|---|---|
| OHLCV | Bar close + 2 s (the same grace as finalized candles) |
| Settled funding | Funding timestamp (predicted funding is a separate series) |
| OI, long/short ratios | Period end + 1 min |
| DVOL | Candle close |
| Fear & Greed | End of labelled UTC day + 1 h |
| Daily on-chain, DefiLlama | Day end + 24 h |
| FRED / ALFRED | Vintage date |
| ETF flows | Next US business day, 12:00 UTC |
| Calendar events | `announced_at` |

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| DA-009 | Ubik shall store timestamps as UTC µs, label candles by open time over [open, open+tf), and build higher timeframes from 1m in UTC. Every stored series shall carry `event_time`, `available_at` and `first_seen_at`. | Must | P1 | Consistent semantics. Field names and the `available_at ≤ decision time` rule per doc 03 TA-026. |
| DA-010 | Strategies shall receive only finalized candles unless they explicitly opt in to intrabar data, which shall be labelled in their reports. | Must | P1 | No lookahead (doc 04). |
| DA-011 | Ubik shall run the §5 checks at ingest and store the results as queryable quality flags. It shall never silently correct data. | Must | P1 | Honesty. |
| DA-012 | Gaps shall be explicit records. They shall be auto-backfilled after reconnects, and any that remain shall be listed in backtest reports and shaded on charts. | Must | P1 | Beginners must see data holes. |
| DA-013 | A freshness monitor shall flag stale symbols, venues and feeds (a data-health flag, not a trading state) using per-series thresholds, and drive doc 05's stale-data handling (§4.4, RK-026: new entries denied; venue REDUCE_ONLY after 30 s) and the owner alerts of NFR-017. | Must | P1 | Classic bot failure. Doc 05 §4.4 owns the thresholds. |
| DA-014 | Ubik shall retain delisted instruments, support rename aliases, and build universes point-in-time. | Should | P1 | Must by P2 (scanner). |
| DA-015 | Bulk loaders shall verify checksums and detect timestamp-unit changes before writing. | Must | P1 | A verified trap. |
| DA-016 | Ubik shall record venue outages and maintenance windows, and the backtester shall treat them as non-tradable. | Should | P2 | Realism. |
| DA-017 | Revisable data shall be stored append-only, each revision a new row with the next `revision_no`, and backtests shall query it as of the decision time (`available_at ≤ decision time`). | Must | P2 | No lookahead from revisions. Matches doc 03 TA-027. |

---

## 6. Storage

### 6.1 Volume estimates

Measured on Binance BTCUSDT data with DuckDB, Parquet + zstd compression, on a 4-vCPU box. Other figures are estimates.

| Data | Rate | Size | P1 footprint |
|---|---|---|---|
| 1m candles | 525,600 per pair-year | **~33 B/row in Parquet → ~17 MB per pair-year** (Sep 2026: 43,200 rows = 1.42 MB). Postgres: ~150 B/row incl. index | 6 pairs × 2 venues × 6 years ≈ **1.3 GB Parquet**. 90 days hot in Postgres ≈ 0.25 GB |
| 1s candles | 31.5M per pair-year | ~0.8 GB per pair-year (zipped) | Optional |
| Trades (BTCUSDT) | **~3.0M per day** (1 Oct 2026) | **~4.3 B/row → ~13 MB/day → ~4.7 GB per pair-year** | 2–3 pairs: ~10 GB/yr. One day re-aggregated into 1m bars in **0.16 s** |
| L2 top-20 snapshots @ 1 s | 86,400 per pair-day | Est. 10–20 MB/day/pair | 5 pairs: ~20–35 GB/yr |
| Full incremental L2 (every book change) | 10+ updates/s | Est. 0.2–1 GB/day/pair (verify with Tardis samples) | Not P1 |
| Orders, fills, ledger, audit | Small | < 100 MB/yr | Postgres |

**Conclusion:** P1–P2 fits in a **100–250 GB SSD**. Only full order-book capture at scale needs a dedicated tick database.

### 6.2 Options

| Option | License | Strengths | Weaknesses | Ubik role |
|---|---|---|---|---|
| **PostgreSQL 17/18** | PostgreSQL | ACID (safe transactions), exact `NUMERIC`, mature | Row storage is bulky for ticks | **System of record** |
| **+ TimescaleDB Community** | Apache-2.0 core. Compression, continuous aggregates and retention under the [Timescale License](https://www.tigerdata.com/docs/about/latest/timescaledb-editions) (free to self-host; you can't resell the *database* as a service) | "Hypertables" (auto-partitioned time tables), ~10×+ compression, auto-rolled 5m/1h/1d views, all still SQL | "-oss" images lack these features | **Hot market data**, optional |
| **Parquet + DuckDB / polars** | Apache-2.0 format, MIT engines | Zero-ops columnar files (each column stored together, so fast scans and small size). Immutable snapshots. Portable | No concurrent multi-writer transactions. Needs a compaction job | **History and backtest datasets** |
| ClickHouse | Apache-2.0 | Very fast columnar analytics, ASOF joins | Another server. Not for transactional state | Tick/L2 store, P3+ |
| QuestDB | Apache-2.0 (HA is Enterprise) | Very high ingest (vendor claims ~1M rows/s, [benchmark](https://questdb.io/time-series-benchmark-suite)), ASOF joins | Smaller ecosystem | ClickHouse alternative |

### 6.3 P1 design and growth path

- **Postgres (+Timescale)** holds accounts, sleeves, orders, fills, ledger, bots, strategies, runs, alerts, audit, config and jobs. Paper and live trading tables sit in separate `paper` and `live` schemas (§9). It also keeps **90 days of 1m candles** plus rollups, for charts and live indicator warm-up.
- **The Parquet lake** uses layout `lake/{venue}/{type}/{symbol}/{tf}/year=YYYY/month=MM/`. A nightly job moves finished days out of Postgres. Backtests read only **dataset snapshots**: manifests of files, content hashes and a quality summary.
- **Growth**: past ~50 GB/month of tick/L2 capture, add ClickHouse or QuestDB behind the same `MarketDataStore` interface.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| DA-018 | PostgreSQL shall be the system of record for transactional and config data. All money amounts shall be `NUMERIC`/Decimal, never binary floats. | Must | P1 | Exact accounting. |
| DA-019 | Hot market data (≥ 90 days of 1m plus 5m/1h/1d rollups) shall be served from Postgres/TimescaleDB, and the schema shall remain valid on plain Postgres. | Should | P1 | Fast charts; no lock-in. |
| DA-020 | Historical market data shall be stored as zstd Parquet partitioned by venue/type/symbol/timeframe/month and queried with DuckDB/polars. A dedicated tick store may be added later behind the `MarketDataStore` interface. | Must | P1 | Measured ~17 MB per pair-year. |
| DA-021 | Every backtest shall reference an immutable dataset snapshot. Re-running the same snapshot, strategy version and config shall give identical results. | Must | P1 | Reproducibility. |
| DA-022 | The owner shall be able to export all user data at any time as CSV, JSON or Parquet: strategies, versions, backtest and paper runs and results, journal, orders, fills, ledger, balances, logs, audit and configuration. Paper exports shall carry the doc 08 CMP-012 watermark. | Must | P1 | Implements doc 02 CAP-037, the export requirement. Data ownership; tax-specific formats in P3 (doc 08 CMP-014). |

---

## 7. System architecture

### 7.1 A modular monolith with three process roles

A **modular monolith** is one codebase with enforced module boundaries, deployed as a few processes instead of many microservices. That gives most of the isolation at a fraction of the operational cost.

- **`engine`** is always on (asyncio). It runs ingestion, strategies, risk, OMS, venues, ledger and scheduler.
- **`api`** (FastAPI) serves REST and SSE, the AI-assistant gateway and notifications.
- **`worker`** processes run backtests, optimizations, backfills and reports. These are CPU-heavy, so they run in separate processes to avoid Python's single-thread lock (the GIL) stalling the engine.
- **`sandbox`** (P2) runs all user-supplied, LLM-generated and third-party code covered by doc 08 SEC-035, in every mode. The engine and workers talk to it only through the strategy port: market data and context in, intents out (AR-037).
- **`gateway`** (P3; interface designed in P1) is the execution gateway: the **only** process that decrypts exchange keys and talks to private exchange endpoints, as doc 08 SEC-001 requires. It accepts only risk-approved intents from the engine's `LiveVenue` client over an authenticated local channel. From its own config file, independent of strategy and risk-profile config, it enforces the last-line checks: symbol allow-list, maximum order notional, price band, order rate and the S4 pilot caps (doc 05 RK-009, EX-025).

```mermaid
flowchart LR
  subgraph EXT[External]
    EX[(Exchanges<br/>REST + WS)]
    DSRC[(Data sources)]
    LLM[(Claude API)]
    CHAN[(External channels<br/>Email / Telegram / ntfy)]
  end
  subgraph ENG[engine process]
    ING[Ingestion + validator]
    IND[Indicators & analysis]
    STR[Strategy runner]
    RISK[Risk engine]
    OMS[OMS / execution]
    SIM[SimVenue]
    LIVE[LiveVenue client]
    LED[Portfolio & ledger]
    SCH[Scheduler]
  end
  subgraph GWP[gateway process - P3]
    GW[Execution gateway<br/>only holder of keys,<br/>last-line checks]
  end
  subgraph SBXP[sandbox process - P2]
    SBX[User / third-party code<br/>no network, no secrets]
  end
  subgraph WRK[worker processes]
    BT[Backtester & optimizer<br/>same strategy/risk/OMS/SimVenue code]
    BF[Backfill & compaction]
    REP[Reports & scanner]
  end
  subgraph APIP[api process]
    API[REST + SSE]
    AST[AI assistant gateway]
    NOTI[Notifications]
  end
  UI[Web UI<br/>React + Lightweight Charts]
  PG[(PostgreSQL + TimescaleDB)]
  LAKE[(Parquet lake + DuckDB)]
  BUS[[Valkey Streams]]

  EX <--> ING
  LIVE <--> GW
  GW <--> EX
  DSRC --> BF
  ING --> IND --> STR --> RISK --> OMS
  OMS --> SIM --> LED
  OMS --> LIVE --> LED
  ING --> PG
  LED --> PG
  BF --> LAKE
  BT --> LAKE
  BT --> PG
  ENG <--> BUS
  WRK <--> BUS
  APIP <--> BUS
  API <--> PG
  UI <-->|REST / SSE| API
  AST --> LLM
  NOTI --> CHAN
  STR <-->|strategy port| SBX
  SCH --> BUS
```

### 7.2 How modes swap the clock, feed and venue

| Mode | Clock | Feed | Venue | Account |
|---|---|---|---|---|
| **Backtest** | `SimClock` (jumps event to event) | `HistoricalFeed` (snapshot) | `SimVenue` | Ephemeral |
| **Paper** | `WallClock` | `LiveFeed` | `SimVenue` (same code, fills against the live book) | Paper account (`paper` schema) |
| **Shadow** (P3, stage S3) | `WallClock` | `LiveFeed` | `SimVenue` for fills, plus `LiveVenue` **read-only** to compare simulated vs. achievable fills | Live account through a read-only key, for balances and state; simulated fills booked to a shadow account in the `paper` schema, never the live ledger |
| **Live** (P3) | `WallClock` | `LiveFeed` | `LiveVenue` client → execution `gateway` (holds the keys; can't be bypassed) | Live account (`live` schema), with reconciliation (doc 05) |

A single **composition root** wires the adapters together from `mode` in config. Strategy code is identical in all modes. It may use only the injected `Clock`, events and order-intent API. It may not call `datetime.now()`, open network connections, or import adapters.

### 7.3 Order flow with a write-ahead intent

**Idempotent** means doing something twice has the same effect as doing it once. The **write-ahead (outbox) pattern** means persisting an action *before* performing it. Together they mean a crash can never produce a duplicate order.

```mermaid
sequenceDiagram
  participant F as LiveFeed
  participant S as Strategy
  participant R as Risk engine
  participant O as OMS
  participant DB as Postgres
  participant V as Venue (Sim or Live)
  participant L as Ledger
  F->>S: BarClosed(BTC/USDT 1m, t)
  S->>R: OrderIntent(buy 0.01 BTC, limit)
  R-->>S: Rejected(reason) if a limit is breached
  R->>O: ApprovedIntent
  O->>DB: INSERT order PENDING_SUBMIT (deterministic client_order_id)
  O->>V: submit(client_order_id)
  V-->>O: Ack(venue_order_id)
  O->>DB: append OrderEvent(OPEN)
  V-->>O: Fill(qty, price, fee)
  O->>L: post fill
  L->>DB: one transaction: Fill + OrderEvent + double-entry ledger rows
  Note over DB,L: Event published on the bus, UI updated via SSE
```

On restart, the engine loads every order in `PENDING_SUBMIT` or `OPEN`, queries the venue by `client_order_id`, and adopts, cancels or marks each one. Only then do strategies resume. Paper mode uses the identical path, so the logic is battle-tested before real money.

### 7.4 Messaging, UI updates, config, scheduling, notifications

- **Messaging.**
  - Inside the engine: a typed in-process async bus.
  - Between processes: **Valkey Streams** behind an `EventBus` interface. Valkey is the BSD-licensed Redis fork; Redis 8 is now [AGPL/SSPL/RSAL tri-licensed](https://www.theregister.com/2025/08/14/redis_redux/). NATS JetStream can replace it in P4.
  - Durable jobs sit in a Postgres table claimed with `FOR UPDATE SKIP LOCKED`, so they survive restarts.
  - Rejected for P1: LISTEN/NOTIFY alone (no replay, 8 KB payload limit) and Kafka (too heavy).
- **Real-time UI.** **Server-Sent Events (SSE)** carry prices, bot status, job progress and assistant streaming. SSE is a one-way HTTP push stream with automatic reconnect. Commands go over REST, with TypeScript types generated from FastAPI's OpenAPI schema. WebSockets only if a truly two-way need appears.
- **Configuration.** Layered and Pydantic-validated: code defaults → `ubik.yaml` → environment variables (secrets) → DB runtime settings (UI-editable, every change audited).
- **Scheduler and notifications.** Schedules are persisted, with a missed-run policy. Notifications go in-app plus at least one external channel the owner chooses (email, Telegram or ntfy), with deduplication, rate limits and quiet hours. Critical alerts (live, P3) bypass quiet hours; paper Warnings are held (doc 07 §8).

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| AR-010 | Ubik shall be one repository and Python package of modules with declared interfaces, deployed as `engine`, `api` and `worker` processes. Cross-module imports shall be CI-checked. | Must | P1 | Isolation without microservices. |
| AR-011 | Each mode shall be assembled from the `Clock`/`MarketDataFeed`/`ExecutionVenue` ports at one composition root. Strategy code shall have no mode branches and no access to wall-clock time, network or adapters. | Must | P1 | Sim-live parity. |
| AR-012 | Backtest and paper shall use the same `SimVenue` fill-simulation code. | Must | P1 | One fill model to validate (doc 04). |
| AR-013 | Events shall flow over a typed in-process bus inside the engine and over a schema-versioned `EventBus` (Valkey Streams) between processes. | Must | P1 | Swappable later. |
| AR-014 | Backtests, optimizations, backfills and reports shall run as durable jobs in workers, with progress, cancellation, timeouts and CPU/memory limits. | Must | P1 | The engine never stalls. |
| AR-015 | The OMS shall persist each order with a deterministic `client_order_id` before submitting it. After a restart it shall reconcile every non-final order with the venue before strategies resume. This applies in paper mode too. | Must | P1 | No duplicates (NFR-006). |
| AR-016 | Order events, fills and ledger entries shall be append-only. Positions and balances shall be derived, snapshotted and rebuildable. | Must | P1 | Auditability (doc 05). |
| AR-017 | The UI shall receive real-time updates via SSE and send commands via REST, using an API client generated from OpenAPI. | Must | P1 | Simple and robust. |
| AR-018 | Configuration shall be layered (defaults → file → env → DB), validated at startup, and every runtime change shall be audited. | Must | P1 | Safe changes. |
| AR-019 | Live execution shall run in a separate `gateway` process, the only component that decrypts exchange credentials. It shall start only if `mode: live` is set; credentials are verified as trade-only with no withdrawals (doc 08 SEC-008) and IP-restricted (SEC-011); the egress-IP check passes (SEC-023); the owner has armed live trading per SEC-003 (SEC-016 step-up, the CMP-022 flow and the SEC-018 cooling-off); and a risk profile is active. Otherwise it shall refuse and explain why. It shall enforce the §7.1 last-line checks from its own config. | Must | P3 | Designed in P1 so that going live is a config change, not a rewrite (docs 05; 08 SEC-001/003/009). |
| AR-020 | Ubik shall provide a shadow mode comparing simulated fills with the live book and with read-only venue state. | Should | P3 | Staged rollout (stage S3, doc 05 §9.1). |
| AR-021 | A persisted scheduler (with missed-run policy) and a notification service (in-app plus ≥ 1 owner-chosen external channel: Telegram, email or ntfy; with dedup, rate limits and quiet hours that Critical alerts bypass) shall be provided. | Must | P1 | Unattended always-on bot. "Push" means web push (P2) only. Channel setup and testing: doc 07 UX-054; channel credentials are secrets under doc 08 SEC-014. |
| AR-022 | Every user-owned table shall carry `workspace_id`, and all data access shall go through a deny-by-default authorization layer. Automated cross-workspace access tests are Should in P1 and Must in P4. | Must | P1 | Cheap path to P4 multi-user: a workspace maps 1:1 to a P4 tenant. Same column as doc 08 SEC-020. |
| AR-023 | Indicators, strategies, venues, data sources and notification channels shall be loadable plugins (entry points) behind versioned interfaces. In P1 only first-party modules shipped with Ubik shall be loaded (doc 08 SEC-034). | Should | P1 | Must in P2 for third-party plugins. From P2, user and third-party code plugins (strategies, indicators) run only in the SEC-035 sandbox (AR-037). Venue, data-source and notification plugins need network access or credentials, so they stay first-party until a separate security review. |
| AR-037 | Ubik shall run all code covered by doc 08 SEC-035 (user-supplied, LLM-generated and third-party code, such as Python strategies, Python indicators per doc 03 TA-025, and third-party plugins) in a separate `sandbox` process role, in every mode: backtests and optimizations started by workers, and paper and live started by the engine. The engine and workers shall talk to it only through the strategy port (market data and context in, intents out). | Must | P2 | Doc 10 D-07 and D-44: no user code in P1 (SEC-034). In-process execution is limited to first-party code and declarative rule strategies. |

---

## 8. Tech stack

### 8.1 Engine core: build or adopt?

Doc 02 owns the overall build-vs-reuse decision. This is the architecture view.

| Option | License / status | Assessment |
|---|---|---|
| **Custom thin kernel** (Python asyncio + polars/numpy) | Ours | **Decided (doc 10 D-02).** We control event semantics, fill and ledger explanations, and beginner safety, which is where Ubik differs. One-minute bars on ≤ 20 pairs means tens of events per minute, trivial for Python. We own fill and accounting correctness, mitigated by the §10 tests. |
| **[NautilusTrader](https://github.com/nautechsystems/nautilus_trader)** | LGPL-3.0. Stable 1.231 (Aug 2026). **2.0 RC7 with a Rust-native runtime (Oct 2026)**. Python 3.12–3.14 | Strongest open-source sim-live engine, with stable Binance/Bybit/OKX/Coinbase/Kraken adapters. Against it: a steep learning curve, churn during the v2 transition, and a domain model that would leak into the beginner UX. **Spike for ≤ 2 weeks** (AR-028) to record design ideas to borrow; the spike is not an adoption gate. Re-evaluate only on doc 02 §4.4's revisit trigger (2.0 final stable for three months, or the start of P4 order-book or market-making work). |
| Freqtrade 2026.9 | GPL-3.0 | Mature dry-run, backtesting, hyperopt and UI. But it is built around a candle loop with one strategy per process, and GPL limits later distribution. Use it as a reference design. |
| Hummingbot (Apache-2.0), Jesse 3.4 (MIT), vectorbt 1.1 (Apache-2.0 + Commons Clause, verify) | — | Hummingbot is a P4 market-making reference. Copy vectorbt's vectorized-sweep idea for fast screening. |

### 8.2 Stack

| Layer | Choice (Oct 2026) | License | Why / alternatives |
|---|---|---|---|
| Engine language | **Python 3.13** (3.12 minimum), asyncio, `uv` lockfiles | PSF | Best quant ecosystem. Rust only as an escape hatch. Node has weak numerics; Go has no quant libraries |
| Dataframes | **polars 2.0** (6 Oct 2026), numpy, numba if profiling shows a need | MIT/BSD | Multithreaded, Arrow-native. pandas 3.0 only at library boundaries |
| Indicators | **TA-Lib 0.8.2**, pinned (Oct 2026; binary wheels for x86 and ARM on all three OSes, bundling the C library since 0.6.5, so no C compile) plus Ubik implementations with reference tests | BSD | **Avoid pandas-ta**: GitHub repo returns 404, domain gone, last release `0.4.71b0` (beta, Sep 2025). `ta` was last released 2023 |
| SQL on files | **DuckDB 1.5** | MIT | Parquet queries |
| API | **FastAPI 0.14x**, Pydantic v2, Uvicorn | MIT | Async, OpenAPI, SSE |
| DB access | SQLAlchemy 2 Core, psycopg 3 (LGPL, used unmodified), Alembic | MIT/LGPL | Migrations |
| DB / bus | PostgreSQL 17/18 + TimescaleDB; **Valkey 9** | PostgreSQL / Apache+TSL; BSD-3 | §6, §7.4 |
| Connectivity | **ccxt 4.5.x** | MIT | §3 |
| Frontend | **TypeScript, React 19, Vite, TanStack Query** | MIT | Mainstream, with lots of developer and AI help available |
| Price charts | **[Lightweight Charts 5.2](https://github.com/tradingview/lightweight-charts)** | Apache-2.0 **plus attribution** (NOTICE plus a visible link to tradingview.com; the `attributionLogo` option satisfies it) | Small and fast. Alternatives: KLineCharts 10, or ECharts 6 for non-price charts. TradingView's full "Charting Library" needs a separate license |
| Ops / testing | structlog, Prometheus, Grafana (AGPL, run unmodified), Uptime Kuma; pytest, Hypothesis, Playwright | — | §10 |

**Performance plan.**

- Precompute indicators in vectorized form before the event loop, so the loop only decides and simulates fills.
- Parallelize parameter sweeps across workers.
- Offer a vectorized "screening" backtest for wide sweeps, then confirm finalists with the event-driven engine (doc 04).
- **Indicator parity:** doc 03 specifies one closed-bar, incremental indicator engine used in every mode, tested against TA-Lib batch output. I support that design. As a fallback for any indicator without an incremental version, recompute it with the batch function over a warm-up window at each bar close. That costs microseconds for 1m bars and avoids two implementations drifting apart.

### 8.3 AI assistant integration (UX in doc 07)

- **Placement.** A gateway in the `api` process. It uses the official Anthropic Python SDK and streams answers to the UI over SSE.
- **Models (configurable per route).**

  | Model | ID | Price (input / output per M tokens) | Use |
  |---|---|---|---|
  | **Claude Opus 5.5** (default) | `claude-opus-5-5` | $4 / $20 | Analysis and explanations |
  | **Claude Sonnet 5.5** | `claude-sonnet-5-5` | $2 / $10 | Lighter chat |
  | **Claude Haiku 5.5** | `claude-haiku-5-5` | $0.10 / $0.50 | Bulk news/sentiment tagging through the Batch API at 50% off |

  The stable system prompt and glossary use prompt caching.
- **Tools, not memory.**
  - It answers through read-only tools: market snapshot, backtest report, metric explanations, positions, glossary.
  - It can create **drafts** (strategy config, alert) that the user confirms.
  - It **cannot** place or cancel orders, change risk limits or switch modes.
  - Any number it states comes from a tool result with an as-of timestamp.
- **Cost.** A monthly cap with a usage meter, and the feature is opt-in. Estimate: 50 chats/month at ~30K input and 2K output tokens on Opus 5.5 ≈ **$8/mo**. Tagging 3,000 headlines/day on Haiku 5.5 ≈ **$3/mo**.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| AR-024 | The engine, analysis and backtester shall be written in Python 3.13 (minimum 3.12) with locked dependencies, polars/numpy as the dataframe stack, and strict static typing in core modules (OMS, ledger, risk, adapters). | Must | P1 | Correctness, performance. |
| AR-025 | Indicators shall come from TA-Lib or Ubik implementations validated against reference values. Unmaintained libraries (e.g. pandas-ta) shall not be used. | Must | P1 | Supply-chain risk. |
| AR-026 | All modes and features (chart, scanner, strategies, backtest, paper, live) shall use the same indicator engine (doc 03). Any indicator without an incremental implementation shall be recomputed over a warm-up window with the batch function. A CI parity test shall assert streaming = batch (relative tolerance ≤ 1e-9, per doc 03 TA-003) on recorded data. | Must | P1 | Sim-live parity (doc 03 TA-003). |
| AR-027 | The UI shall be TypeScript/React. Price charts shall use Lightweight Charts with the attribution its license requires. | Must | P1 | License compliance. Lightweight Charts draws on a canvas, so Ubik builds doc 07 UX-046's text summary, data table and keyboard crosshair as its own accessible layer. Verify this in the P1 spike. |
| AR-028 | Before P1 build, a ≤ 2-week NautilusTrader spike shall be run against doc 02 §4.4's scoring criteria, and its outcome recorded as an architecture decision record (ADR): design ideas to borrow and the revisit trigger. | Should | P1 | Borrow proven designs; not an adoption gate (doc 10 D-02). Revisit only on the doc 02 §4.4 trigger. |
| AR-029 | The assistant shall use the Anthropic API with per-route models (default `claude-opus-5-5`; `claude-sonnet-5-5`, `claude-haiku-5-5`). It shall have read-only tools and draft-only actions, shall never place orders or change risk or mode settings, shall enforce a monthly spend cap, and shall state only numbers taken from tool results, with timestamps. | Must | P2 | Safety; no made-up ("hallucinated") figures. |

---

## 9. High-level data model

```mermaid
erDiagram
  WORKSPACE ||--o{ ACCOUNT : owns
  WORKSPACE ||--o{ STRATEGY : owns
  WORKSPACE ||--o{ ALERT_RULE : defines
  WORKSPACE ||--o{ AUDIT_EVENT : records
  VENUE ||--o{ INSTRUMENT : lists
  ASSET ||--o{ INSTRUMENT : "base / quote"
  INSTRUMENT ||--o{ INSTRUMENT_RULES : "versioned rules"
  INSTRUMENT ||--o{ CANDLE : has
  INSTRUMENT ||--o{ DATA_GAP : flags
  SERIES ||--o{ OBSERVATION : "point-in-time values"
  CALENDAR_EVENT }o--o{ ASSET : affects
  STRATEGY ||--o{ STRATEGY_VERSION : "immutable versions"
  STRATEGY ||--o{ TRIAL_LEDGER_ENTRY : "trials counted"
  STRATEGY_VERSION ||--o{ BACKTEST_RUN : "tested in"
  STRATEGY_VERSION ||--o{ VAULT_EVALUATION : "vault opened"
  DATASET_SNAPSHOT ||--o{ BACKTEST_RUN : feeds
  BACKTEST_RUN ||--o{ RUN_METRIC : produces
  STRATEGY_VERSION ||--o{ BOT : "deployed as"
  ACCOUNT ||--o{ SLEEVE : "split into"
  SLEEVE ||--o| BOT : funds
  RISK_PROFILE ||--o{ BOT : constrains
  RISK_PROFILE ||--o{ RISK_DECISION : "version applied"
  BOT ||--o{ STAGE_EVENT : "rollout stage"
  BOT ||--o{ DECISION_LOG : "signals and reasons"
  SLEEVE ||--o{ ORDER : books
  RISK_DECISION ||--o| ORDER : approves
  ORDER ||--o{ ORDER_EVENT : "state changes"
  ORDER ||--o{ FILL : "filled by"
  FILL ||--|{ LEDGER_ENTRY : posts
  ACCOUNT ||--o{ LEDGER_ENTRY : journal
  ACCOUNT ||--o{ TAX_LOT : "cost basis"
  ACCOUNT ||--o{ POSITION : derived
  ACCOUNT ||--o{ BALANCE : derived
  ALERT_RULE ||--o{ ALERT_EVENT : fires
```

Key design points:

- **Instrument**: canonical `BASE/QUOTE@VENUE`, with listing status and dates. Delisted instruments are kept. `INSTRUMENT_RULES` (tick, lot, minimum order value) has `valid_from`/`valid_to`. Renames use alias rows.
- **Candle**: `(instrument, timeframe, open_time UTC µs)` plus OHLCV, trade count, source, `is_final` and quality flags. Hot rows in Postgres, history in Parquet. Trades, book snapshots, depth bands, funding and OI live mostly in the lake. Every series carries `event_time`, `available_at` and `first_seen_at`, plus `revision_no` where revisable (§5).
- **Observation**: doc 03 §6's point-in-time record for non-candle series (funding, OI, dominance, on-chain, sentiment, macro): series, asset, period, `event_time`, value, `available_at`, `first_seen_at`, source, `revision_no`. **CalendarEvent** stores `scheduled_at`, `announced_at`, importance, source and affected assets (DA-025).
- **Account**: kind is `backtest`, `paper` or `live`. Paper and live use identical table definitions instantiated in two Postgres schemas, `paper` and `live`, each with a mode CHECK constraint so rows can't mix (doc 08 CMP-012). Going live changes only the schema binding and the venue, not code. Backtest accounts stay ephemeral. S3 shadow fills are booked to a shadow account in the `paper` schema, never to the live ledger.
- **Sleeve**: a capital partition of an account, with its own capital, positions, fills and P&L (doc 05 PF-001). An account has many sleeves. The reserve, the Manual sleeve (doc 05 PF-014) and the External sleeve (doc 05 EX-013) have no bot. Orders and ledger entries carry `sleeve_id`, and sleeves plus the reserve equal account balances exactly.
- **StrategyVersion** never changes after creation: code hash, parameter schema, parameters. A **Bot** is one strategy version deployed on one account, with instruments and a versioned risk profile (Careful, Balanced or Adventurous; doc 05 RK-005). It owns exactly one sleeve. Its rollout stage (Draft, S0–S5, Retired) and graduation scorecard are tracked per Bot as stage events (doc 05 EX-022, §9.1).
- **DecisionLog**: every typed signal (doc 03 SIG-001) and strategy decision, with input values and rule values at decision time. It feeds the Journal (doc 07 UX-016) and "why did my bot do that?" answers (ED-010). **RiskDecision**: APPROVE, REDUCE or DENY with a reason code, input snapshot and risk-profile version (doc 05 RK-003); `orders.risk_decision_id` points to it.
- **TrialLedgerEntry**: the single trial ledger (doc 04 BT-021), with strategy and signal entries; retired versions still count. It satisfies doc 02 CAP-005 and doc 03 MA-023. **VaultEvaluation** logs each opening of the out-of-sample vault, allowed once per strategy version (doc 04 BT-022).
- **TaxLot** preserves cost basis per acquisition (doc 05 EX-012, doc 08 CMP-013).
- **Order** has a deterministic `client_order_id`, a venue order ID, and an append-only `ORDER_EVENT` history. **Fill** is unique on (venue, venue trade ID) and records fee, fee asset and maker/taker.
- **LedgerEntry** is double-entry: every transaction sums to zero per asset, with signed `NUMERIC` amounts and typed entries (trade/fee/funding/deposit/adjustment). Positions and balances are derived from it (doc 05).
- **BacktestRun** records the snapshot, strategy version, config, cost model, seed and engine version, which together are enough to reproduce it.
- **AuditEvent**: actor, action, before/after values. Append-only and never deleted.
- Supporting tables: **Job**; **AssistantMessage** (prompt, output, model version, tokens and cost; live LLM outputs are stored, never regenerated, per doc 03 §6 rule 13 and MA-016); **JournalNote** (doc 07 UX-016); **Incident** (doc 07 UX-007, doc 02 CAP-030); **Acknowledgement** (stored consents and typed confirmations: doc 08 CMP-017 and CMP-022, doc 05 RK-039); **EvidenceRegistry** (doc 03 SIG-012); and **ApiCredential**, which stores only a reference to an encrypted secret (doc 08; no exchange keys in P1, SEC-007).

---

## 10. Deployment and operations

**Deployment.** One `docker compose up`, with images built for **amd64 and arm64** (PC, Apple-silicon Mac, Raspberry Pi 5, ARM VPS). Services:

- `postgres` (TimescaleDB image) and `valkey`
- `engine`, `api` (also serves the web app) and `worker` (scalable); `sandbox` from P2; `gateway` from P3
- `caddy` for TLS
- `backup`
- optional `observability` profile: Prometheus, Grafana, Uptime Kuma

Remote access goes through **WireGuard or Tailscale**, not open ports. The database and bus are never exposed (doc 08).

**Hosting cost (P1)**

| Option | Spec | ~Cost / month | Trade-offs |
|---|---|---|---|
| Home machine (mini-PC, desktop, Raspberry Pi 5 8 GB + NVMe) | 4+ cores, 8–16 GB, ≥ 100 GB SSD (NFR-014; 250 GB leaves room to grow) | $3–10 (electricity) | Free, and data stays home. But home internet and power outages stop the bot |
| Budget VPS (e.g. Hetzner) | 4 vCPU / 8 GB | ~€8–20 | Cheap, but prices rose [in April](https://www.igorslab.de/en/hetzner-to-significantly-increase-prices-for-cloud-and-dedicated-servers-from-april-2026/) and [June 2026](https://wz-it.com/en/blog/hetzner-price-increase-june-2026-cpx-ccx-alternatives/) and some plans were shown as unavailable |
| Mainstream VPS (DigitalOcean, Vultr, Akamai) | 4 vCPU / 8 GB | ~$40–60 | Predictable |
| Encrypted off-site backups (B2 / S3-compatible) | ~100 GB | ~$1 | — |
| LLM (P2, opt-in) | — | ~$5–15 | Capped (AR-029) |

⚠️ **The server's location has legal weight.** Running the bot in another country to reach an exchange that blocks the owner's own country may breach exchange terms or law. Doc 08 decides (CMP-006). Ubik records the owner's jurisdiction profile and warns if the server's region differs (AR-034). Live trading (P3) also needs a static egress IP and a server location the exchange permits (doc 08 SEC-023), so a home connection with a changing IP is fine for paper but not for live.

**Backups.** In P1 a nightly full backup of Postgres (pgBackRest), config and the self-recorded lake data (restic) goes encrypted to off-site storage. Self-recorded data can't be re-downloaded, so it must be backed up; public bulk files can be re-downloaded. Continuous WAL archiving (WAL is Postgres's change log) is Should in P1 and Must from P3. The documented restore is tested once before P1 exit and before each phase release, with quarterly drills from P3 (NFR-012).

**Observability.**

- JSON logs with correlation IDs (bot, order, job).
- Prometheus metrics: event-loop lag, WS reconnects, data freshness, rate-limit headroom, order latency, job duration, disk usage.
- `/healthz` (alive) and `/readyz` (DB, bus and data fresh).
- **External heartbeat monitor**: the engine pings an off-host monitor every 30 s, and 90 s of silence alerts the owner, even if the machine is down (NFR-017). This is not doc 05's dead-man's switch (RK-025, P3), which cancels resting entry orders when the heartbeat stops.

**Time sync.** `chrony` (an NTP client) with an alert above 100 ms host-vs-NTP offset (NFR-013), plus comparison against exchange server time (doc 05 RK-027). Signed requests fail outside the exchange's receive window (Binance's default is 5 s), and drift corrupts candle boundaries.

**Testing.**

| Layer | What it proves |
|---|---|
| **Property-based** (Hypothesis) | For any random sequence of fills, partial fills, fees and cancels: every ledger transaction sums to zero, spot balances never go negative, and the rebuilt state equals the snapshot |
| **Replay** | Recorded WS sessions (with disconnects) produce the same orders every time |
| **Golden backtests** | A fixed snapshot and strategy give a known trades+metrics hash. Changes need explicit approval |
| **Parity** | Backtest and paper on the same recorded day make identical decisions and compute identical indicator values |
| **Adapter contract** | P1–P2: public endpoints plus recorded-session replay (AR-006), before every ccxt upgrade. From P3: also nightly against exchange demos where the venue has one (stage S2, AR-004) |
| **Chaos** | Kill the engine between "intent saved" and "submitted" and mid-fill. Drop the WS. Restore from backup. Result: **no duplicates, no lost fills** |
| Unit, UI end-to-end (Playwright) | Modules; critical UI paths (onboarding, start/stop bot, kill switch) |

**CI** (e.g. GitHub Actions) runs lint (ruff, eslint), type checks, all of the above suites, the import-boundary check, pip-audit/npm audit, a license check, secret scanning and multi-arch builds.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| AR-030 | Ubik shall deploy with one Docker Compose command on amd64 or arm64, without exposing the DB or bus publicly, and with VPN/tunnel remote access by default. | Must | P1 | Self-hosting security. |
| AR-031 | Ubik shall back up Postgres, config and the self-recorded lake data nightly to encrypted off-host storage, with a documented restore procedure and one successful restore before P1 exit. Continuous WAL archiving shall be added (Should in P1, Must from P3). | Must | P1 | NFR-012; doc 08 SEC-031. |
| AR-032 | Ubik shall provide structured logs, metrics, liveness/readiness endpoints and an external heartbeat monitor: an off-host service that the engine pings every 30 s and that alerts the owner after 90 s of silence. | Must | P1 | Always-on engine; meets NFR-017. Not the dead-man's switch (doc 05 RK-025), which cancels orders. |
| AR-033 | CI shall block merges unless the property, replay, golden, parity and import-boundary tests and the dependency/license audits pass. | Must | P1 | Correctness gates. |
| AR-034 | Ubik shall record the owner's declared jurisdiction in the jurisdiction profile (doc 08 CMP-004, the same entity as AR-001's setting) and warn when the server's egress country or the enabled venues conflict with it (CMP-006). | Must | P1 | Legal (doc 08). |
| AR-035 | Ubik shall upgrade through one documented command that: pauses all strategies first (in live mode, from P3, only after the owner confirms); takes and verifies a backup (AR-031); applies versioned, forward-only database, lake and manifest migrations; reconciles (doc 05 EX-007) before strategies resume; and can roll back by restoring the pre-upgrade backup. Ubik shall refuse to start on a schema-version mismatch. Old run manifests shall stay re-runnable with the engine version they name. | Must | P1 | Paper runs 30 days unattended (G3), so upgrades happen mid-run. Release notes list every engine, fill-model, fee-model or rule-DSL change that alters golden-test results. The engine API version in the strategy hash changes only on breaking strategy-API changes, so routine upgrades don't reset graduation (doc 04 §1.3). |
| AR-036 | Ubik shall ship a self-hosting guide covering prerequisites (NFR-014, NFR-015), install, remote access (doc 08 SEC-021), backup and restore (AR-031), upgrade (AR-035), troubleshooting (clock sync, geo-blocks per AR-003, rate-limit bans) and uninstall. It shall also ship a first-run bootstrap that creates the owner login with MFA (doc 08 SEC-017), sets up secret storage, records the jurisdiction profile (AR-001, AR-034), selects venues and backfill scope (DA-002) with a disk estimate, starts the quick-start dataset (DA-023), and runs a pre-flight check (disk space, clock offset per NFR-013, venue reachability, no publicly exposed ports). | Must | P1 | AR-030 gives one-command deploy; this covers everything around it. Secret storage: the key-encryption key stays outside the database, in the OS keyring or systemd-creds by default, with a passphrase at boot as an option (doc 08 SEC-012); the choice is recorded in an ADR. |

---

## 11. Non-functional requirements

Ubik is **not** high-frequency trading. Targets favor reliability and correctness at minute-scale decisions. "p95" means 95% of cases meet the target.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| NFR-001 | Ubik shall go from a finalized candle to the strategy's decision in ≤ 250 ms p95 for ≤ 20 instruments. | Must | P1 | Ample for 1m+ strategies. |
| NFR-002 | The OMS shall go from approved intent to the order leaving Ubik in ≤ 300 ms p95, excluding network time. | Must | P1 | Matters for live. |
| NFR-003 | Prices and bot state shall reach the UI ≤ 1 s p95 after receipt. The UI shall be interactive in ≤ 2 s, and 1 year of 1h candles with indicators shall render in ≤ 1 s. | Should | P1 | Responsiveness (doc 07). |
| NFR-004 | Engine availability shall be ≥ 99.5% per month in P1 (≈ 3.6 h of downtime) and ≥ 99.9% for live on VPS hosting, excluding venue outages. | Must | P1 | Always-on bot. |
| NFR-005 | After a WS drop, Ubik shall reconnect in ≤ 5 s p95 and backfill missed candles in ≤ 2 min, or flag a gap. | Must | P1 | Completeness. |
| NFR-006 | After a crash or restart at any point, Ubik shall resume with **zero duplicate orders and zero lost fills** (proven by chaos tests), resuming bots only after reconciliation. Shutdown shall be graceful: stop new intents, finish persistence, then exit. | Must | P1 | Safety. |
| NFR-007 | Money arithmetic shall use Decimal/`NUMERIC` with venue precision rules. Paper ledgers shall reconcile exactly, and live differences shall be explained down to the venue's rounding (doc 05). | Must | P1 | Accounting accuracy. |
| NFR-008 | On the reference machine (4 vCPU / 8 GB), an event-driven backtest with a template strategy shall finish 1 instrument × 1 year of 1m (~525K candles) in ≤ 60 s, and 1 instrument × 5 years in ≤ 5 min. Stretch goal for P2: 5 years in ≤ 60 s. | Should | P1 | "Years of 1m in minutes." Matches doc 04 BT-018. A speed target, not a safety target. |
| NFR-009 | Vectorized screening shall evaluate ≥ 1,000 parameter sets × 1 instrument × 1 year of 1m in ≤ 10 min. | Should | P2 | Optimization (doc 04). |
| NFR-010 | The same snapshot, strategy version, config, seed and engine version shall produce byte-identical trade lists and metrics. | Must | P1 | Golden tests, honesty. |
| NFR-011 | Default retention: candles forever; raw trades 2 years; raw L2 snapshots 1 year at 1 s cadence (P1 L2 recording capped at 5 instruments, DA-003); per-minute depth bands forever; logs 30 days; metrics 90 days; paper records until the owner deletes them. Live orders, fills and ledger, and the audit log, are **never deleted automatically**; if the owner configures deletion, the floor is 10 years by default (doc 08 CMP-016) and never below the jurisdiction's minimum. Retention is configurable, with a disk forecast that warns before disk use reaches 85%. | Should | P1 | Tax and audit (doc 08). This is the single retention table; other docs cite it. Self-recorded venue data can't be re-downloaded, and doc 04's P2 cost calibration needs it. |
| NFR-012 | In P1, transactional data shall meet RPO ≤ 24 h (maximum data lost) and RTO ≤ 4 h (time to recover), with one successful restore before P1 exit. From P3, it shall meet RPO ≤ 5 min and RTO ≤ 1 h, with a successful restore drill at least quarterly. | Must | P1 (P3 targets from P3) | Recoverability, proportionate to a paper-only P1. Matches doc 08 SEC-031. Restarts without lost state are covered by NFR-006. |
| NFR-013 | Host clock offset against NTP shall stay ≤ 100 ms (alert above), and engine timestamps shall never go backwards. | Must | P1 | Signed requests, candle integrity. Host-vs-exchange offset limits are doc 05 RK-027. |
| NFR-014 | P1 shall run in 4 vCPU / 8 GB / 100 GB SSD for 20 instruments and 10 paper bots, with idle CPU < 15%. | Must | P1 | Cheap hosting. Relies on the 5-instrument L2 cap (DA-003, §6.1). |
| NFR-015 | Ubik shall run unchanged on Linux, macOS and Windows (Docker), on amd64 and arm64, with no cloud-vendor dependency. | Must | P1 | Portability. |
| NFR-016 | A ccxt-supported venue shall be addable in ≤ 1 developer-day plus contract tests. A new indicator shall be one module plus reference tests, with no core changes. | Should | P2 | Extensibility (AR-023). |
| NFR-017 | The owner shall be alerted through an external channel within 2 min (live: always; paper: outside quiet hours) of engine death, of stale data while bots are running or positions are open, of a ledger mismatch, or of disk > 85%. | Must | P1 | Observability. Engine death is detected by the external heartbeat monitor (AR-032); stale-data handling is doc 05 §4.4. Severities per doc 07 §8: engine-offline and stale-data alerts are Warnings in paper and Critical in live. |
| NFR-018 | Ubik shall stay ≤ 70% of every venue's rate limits (doc 05 EX-006) and incur zero IP bans in normal operation. | Must | P1 | Venue access. |
| NFR-019 | Core modules (OMS, ledger, risk, SimVenue, adapters) shall have ≥ 90% branch coverage. Golden and parity suites run on every merge. | Should | P1 | Quality bar. |
| NFR-020 | All dependencies shall be pinned and license-checked. GPL/AGPL components shall run only as separate, unmodified services unless an ADR approves otherwise. | Must | P1 | Keeps P4 distribution options open. Part of doc 08 CMP-029's licence policy, with doc 02 CAP-044. |

---

## Open questions for the owner

1. **Which country (and US state, if applicable) do you live in?** This decides Binance vs. Coinbase, which data sources are allowed, and whether a VPS abroad is permissible.
2. **Do you already have accounts or API keys on any exchange?** We would prefer the venue you'd eventually trade on.
3. **Home machine or VPS?** Is a home outage pausing paper trading acceptable? What is the monthly ceiling for hosting, data and AI combined?
4. **Which home currency should reports use** (USD, EUR, …)? This affects stablecoin and FX conversion and later tax exports.
5. **How many pairs and how much history to backfill?** The default is 6 pairs × 6 years of 1m.
6. **Record order-book snapshots from day one?** That's ~20–35 GB/year for up to 5 instruments, and it makes simulated fills much more realistic.
7. **Preferred external alert channel**: email, Telegram or ntfy? At least one is required in P1.
8. **Will you write Python strategies yourself**, or use templates and the builder only? *Answered by doc 10 D-07: P1 is templates and the rule builder only; Python strategies arrive in P2, in the sandbox (AR-037).*
9. **Might Ubik be open-sourced or offered to others later?** This affects tolerance for copyleft dependencies and attribution duties.
10. **For the P2 assistant**: is sending portfolio summaries to the Claude API acceptable, and what monthly cap?

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Venue restrictions change (Binance's 2026 MiCA turbulence), or the owner's jurisdiction blocks the plan | Medium | High | Jurisdiction profiles with a second venue each, venue-agnostic model (AR-001) |
| A free data source disappears (CoinDesk, May 2026) | High | Medium | Plugins, caching, graceful degradation, self-recording (DA-003, DA-007) |
| ccxt regression from frequent releases | Medium | Medium | Exact pins, contract and replay tests (AR-006) |
| Sim-live parity drift makes paper results misleading | Medium | High | Shared SimVenue, recompute-on-window indicators, parity tests (AR-012, AR-026) |
| Data-quality issues (gaps, unit changes, outliers, survivorship) inflate backtests | High | High | §5 contract, explicit gaps, point-in-time universes, quality summary in every report |
| Duplicate or lost orders after crashes (P3) | Low | Critical | Write-ahead intents, deterministic IDs, reconciliation, chaos tests (AR-015, NFR-006) |
| Python too slow for large optimizations | Medium | Medium | Vectorized screening, multiprocessing, numba; Rust/Nautilus escape hatch behind the ports |
| Single node fails (home internet, power, disk) | Medium | Medium / High in P3 | External heartbeat monitor, backups, VPS for live, restore drills |
| Clock drift | Low | Medium | chrony plus exchange-time checks (NFR-013) |
| License breaches (TradingView attribution, TSL scope, GPL/AGPL, vendor redistribution) | Medium | Medium | Data-source registry and dependency checks (DA-008, NFR-020; doc 08 CMP-007, CMP-029) |
| Hosting price or availability shifts (Hetzner 2026) | Medium | Low | Portable multi-arch Compose stack |
| LLM cost overrun or wrong numbers misleading beginners | Medium | Medium | Spend cap, tool-grounded numbers, no trading authority (AR-029) |
| Over-engineering "everything a pro firm does" before P1 ships | High | High | Strict phasing. Add ClickHouse, native adapters and NATS only behind existing interfaces |
