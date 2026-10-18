# 11 — Hyperliquid Integration (primary venue)

*Ubik planning doc 11 · Draft v0.1 · 2026-10-18 · ID prefix: **HL** (Hyperliquid). Written in response to the owner's directive of 2026-10-18: "Make the bot compatible with Hyperliquid. This is going to be the primary way we're going to interact with crypto."*

> **How to read this doc.** Hyperliquid works differently from the centralized exchanges the other docs assumed. This doc explains it for a beginner, recommends how Ubik should use it safely, and lists the requirements (HL-001…). Where it conflicts with docs 01–09, **this doc wins for Hyperliquid** until the changes are folded into those docs and recorded in [10](10-decisions-and-open-questions.md) (proposed decisions D-45 onward, §11). Facts were checked against official sources on 2026-10-18; live numbers come from public API calls made that day (a Sunday evening, so volumes are weekend-low). Anything we could not confirm from a primary source is marked **unverified**.

## Summary

- **Hyperliquid becomes Ubik's primary venue, accessed through Ubik's own thin native adapter.** It is a blockchain with a built-in order book: you keep custody of your funds in your own wallet, and the bot signs orders with a separate "agent" key that **can trade but cannot withdraw**. That matches Ubik's "trade-only key, withdrawals refused" rule (D-24) with one important difference: **there is no IP allow-list**, so the execution gateway's own limits become the main line of defense.
- **Hyperliquid's liquidity is in perpetual futures (perps), not spot.** The BTC perp trades about $2.5B a day; the spot BTC token (a bridged IOU called UBTC) about $42M. We recommend keeping beginner safety by changing D-06 from "spot only" to **"no leverage, no shorting until P4"**: strategies stay long or flat, and may hold either Hyperliquid spot or a **1× isolated long perp** ("spot-like exposure"), with funding costs modeled and the liquidation price always shown. Shorting and leverage above 1× stay in P4 (ceiling 2×, absolute max 3×). Builder-deployed markets (HIP-3), outcome markets (HIP-4) and long-tail tokens are excluded through P3; outcome markets for good.
- **P1 changes little in shape, a lot in detail.** Paper trading still uses Ubik's simulator (`SimVenue`) on live public data, and still needs no keys. But `SimVenue` gains a Hyperliquid profile (tick rules, $10 minimum, hourly funding, mark-price triggers, liquidation), and **Ubik must record Hyperliquid data from day one**, because the public API keeps only about 3.5 days of 1-minute candles. Long backtests use labelled proxy data from Binance (perps) and Coinbase (spot).
- **The path to live is adapted.** S2 runs on Hyperliquid's testnet, which only proves the adapter works (its prices and liquidity are not real). S3 shadow needs **no key at all**, because every Hyperliquid account is public. S4 pilots run in a dedicated account funded only with the pilot budget.
- **New risks to accept or mitigate:** the account is **fully public** (anyone can see positions, orders and stops); the venue has had governance overrides, an API outage and forced "auto-deleveraging" in crashes; restricted persons (US, Ontario, sanctioned territories) may not use it at all. Ubik must check the owner's jurisdiction profile and refuse to enable live trading for a restricted person, and must never help get around that.
- **Keep Binance as the secondary venue** (already assumed for the owner's Colombia profile, Q-01): it is the main source of proxy history, the largest input to Hyperliquid's oracle, and a backup route for spot trading if Hyperliquid becomes unusable.

## 1. What changes in the plan

| Area | Current plan (CEX assumption) | With Hyperliquid primary | Docs affected |
|---|---|---|---|
| Venue | Binance + Coinbase/Kraken by jurisdiction (D-31, AR-001) | Hyperliquid primary; Binance secondary (data + backup spot venue) | 00, 06, 10 |
| Market type | Spot only, long/flat until P4 (D-06, RK-035, ST-010) | Long/flat, **no leverage, no shorting** until P4; vehicle is HL spot **or** 1× isolated long perp (§3) | 00, 04, 05, 10 |
| Custody | Funds sit on the exchange | Funds sit in the owner's own wallet on the Hyperliquid chain; the venue is also the custodian of record for the trading state | 05, 08 |
| Keys | Exchange API key, trade-only, IP-restricted, permission-checked via endpoint (SEC-008–011) | Agent (API) wallet: trades, cannot withdraw; **no IP allow-list**; expiry ≤ 180 days; master key on a hardware wallet, never on the server (§5) | 08 |
| Connectivity | ccxt behind `ExecutionVenue` (D-02, AR-005) | Native thin client (`HyperliquidFeed`, `HyperliquidVenue`); ccxt only for CEX venues and cross-checks (§4) | 02, 06, 10 |
| Order IDs | Client order ID per venue rules | 128-bit `cloid` derived from Ubik's client order ID; `expiresAfter` bounds unknown orders (§4.4) | 05 |
| Rate limits | ≤ 70% of per-IP limits (D-37) | Also an **address budget**: one action per $1 traded plus a 10,000 starting buffer (§4.7) | 05, 06 |
| Dead-man's switch | Native cancel-after where available (RK-025) | Local watchdog first; native `scheduleCancel` possibly locked behind $1M volume (unverified) | 05 |
| Data history | Binance bulk files back to 2017–2020 | HL-native history is short; record from P1 day one; proxy history with fidelity grades (§6) | 03, 04, 06 |
| Funding, mark price | P4 | Needed from P1 if perps are used as the vehicle (hourly funding, mark-price stops) | 03, 04, 05 |
| S2 exchange demo | Binance Demo Mode or contract suite | HL testnet = protocol conformance only, plus minimum-size mainnet orders (§7) | 04, 05 |
| S3 shadow | Read-only key | No key: read the public account by address | 05, 08 |
| Privacy | Exchange sees your trades | **Everyone** sees your trades, orders, stops and balances | 07, 08 |
| Tax | Exchange reports (1099-DA, DAC8/CARF) | No forms from the venue; Ubik's ledger is the only full record (§8.5) | 08 |

## 2. How Hyperliquid works (for a beginner)

**A blockchain that is also an exchange.** Hyperliquid is its own blockchain (a "Layer 1"). Its trading engine, **HyperCore**, keeps a full order book and matching engine *on-chain*: every order, cancel and fill is a transaction in a block, and blocks are produced by a set of validators. A second environment, **HyperEVM**, runs smart contracts; Ubik doesn't need it. Ubik talks to API servers run by the Hyperliquid team, which forward signed actions to the chain and reply **only after the action is in a committed block** ([API servers](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/api-servers)). So a reply always means "final", and a timeout means "unknown", not "failed".

**Your wallet is your account.** There is no sign-up or password. Your account is an Ethereum-style address; whoever holds its private key controls it. There is no "forgot password" and no support desk that can reverse a trade.

**Getting money in: USDC.** Trading collateral is **USDC**, a US-dollar stablecoin issued by Circle. The preferred deposit route is Circle's native transfer protocol (**CCTP**), which mints USDC directly on Hyperliquid; the older Arbitrum bridge is "deprecated" and holds "less than 10%" of HyperCore USDC ([USDC](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/usdc), [Bridge2](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/bridge2)). The legacy bridge still held about 399M USDC on 2026-10-18 (our Arbitrum query), loses deposits under 5 USDC, and has been emergency-locked before (about one hour in Nov 2025, [Cointelegraph](https://cointelegraph.com/news/hyperliquid-hlp-popcat-attack-3m-wipeout)). Circle retires CCTP V1 between 31 Oct and 1 Dec 2026, so any guide must use CCTP V2 ([Circle](https://www.circle.com/blog/migrate-to-cctp-v2-ahead-of-cctp-v1-legacy-deprecation)). **Ubik never moves money**; the owner deposits and withdraws by hand.

**Spot vs perps.**
- **Spot** means you own the token. On Hyperliquid, "BTC/USDC" in the app is really **UBTC/USDC**: UBTC is a token issued by the **Unit** bridge, which holds real bitcoin under a 2-of-3 multi-party signing scheme ([Unit security](https://docs.hyperunit.xyz/architecture/security)). So holding "BTC" on Hyperliquid spot means trusting Unit, an IOU. HYPE (Hyperliquid's own token) is native.
- **Perpetual futures ("perps")** are contracts that track a price without expiry. You never own the coin; you hold a position that gains or loses with the price. Hyperliquid perps are settled in USDC, but their reference price comes from USDT markets on big exchanges (technically "quanto" contracts) ([contract specs](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/contract-specifications)).

**Funding.** To keep a perp's price close to the real coin price, longs and shorts pay each other **every hour**. When the perp trades above the reference, longs pay shorts. There is a built-in "interest" baseline of 0.00125% per hour (about 11% a year paid by longs) whenever the market is calm, and a cap of 4% per hour. Payment = position size × **oracle** price × rate ([funding](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/funding)). Measured BTC funding averaged **24% a year in 2024, 10.6% in 2025 and 5.2% so far in 2026** (full API history from 2023-05-12). For a long-only bot, funding is a running cost.

**Oracle price and mark price.** The **oracle price** is a weighted median of spot prices on Binance (weight 3), OKX, Bybit (2 each), Kraken, KuCoin, Gate and MEXC (1 each), published by each validator about every 3 seconds ([oracle](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/oracle)). The **mark price** is a median of the oracle-plus-premium, Hyperliquid's own book, and CEX perp prices ([robust price indices](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/robust-price-indices)). **Stops, liquidations and unrealized P&L all use the mark price**, not the last trade.

**Margin, liquidation, ADL.**
- **Margin** is the collateral backing a perp position. **Cross margin** shares one pool across positions (a loss on one can eat the others). **Isolated margin** gives each position its own fixed pot, so it can only lose what you put in it ([margining](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/margining)).
- **Leverage** is position size ÷ margin. At **1×**, a $100 position uses $100 of margin.
- **Liquidation** happens when your margin falls below the **maintenance margin** (1.25% for BTC, 2% for ETH, 2.5% for SOL; [margin tiers](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/margin-tiers)). For a 1× isolated long, the liquidation price is close to zero at entry, then creeps up slowly as funding is paid out of the margin ([liquidations](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/liquidations)).
- **Auto-deleveraging (ADL)** is the last resort when a liquidated account goes negative: the system force-closes positions on the *winning* side, at the previous mark price, starting with the most profitable and leveraged ([ADL](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/auto-deleveraging)). It is not a loss in itself, but it closes a position you didn't ask to close. It fired market-wide for the first time on 10 Oct 2025 ([WuBlockchain](https://wublock.substack.com/p/hyperliquid-activates-cross-margin)). Accounts with no open position are never hit.

**Agent (API) wallets.** The account's **master** key signs an approval for a second key, the **agent** or **API wallet**. The agent can place, modify and cancel orders and change leverage, for the master account or its sub-accounts. It cannot withdraw or transfer funds, approve other agents or approve fees: those need the master's signature ([nonces and API wallets](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets); SDK [`basic_agent.py`](https://github.com/hyperliquid-dex/hyperliquid-python-sdk)). Agents can be given an expiry of up to 180 days. Agents only *sign*; to read data you always use the master or sub-account address.

**Public by design.** Anyone can query any address's balances, positions, open orders (including stop levels), fills and transfers, with no key ([info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint)). Every public trade message carries both traders' addresses. Copy-trading sites watch large accounts in real time.

**Who runs it.** On 2026-10-18 there were 27 active validators (our `validatorSummaries` query). The Hyper Foundation's four validators held about 43.7% of active stake: the top 3 alone can halt the chain (more than ⅓), and the top 8 pass the ⅔ threshold that signs bridge withdrawals. That is more decentralized than in 2025 (about 80% Foundation stake per [rekt.news](https://rekt.news/hyperliquidate2)) but still concentrated. Validators have voted to delist and force-settle a market (JELLY, March 2025, [The Defiant](https://thedefiant.io/news/defi/hyperliquid-to-compensate-jellyjelly-traders-and-strengthen-risk-protocols)).

## 3. Market-type recommendation

### 3.1 The tension

D-06 says "spot only, long or flat, until P4" for beginner safety. On Hyperliquid, spot is the thin side (measured 2026-10-18, 30-day averages):

| Market | 30-day avg daily volume | Depth near the touch (per side) | Round-trip taker fee (tier 0) | Extra risks |
|---|---|---|---|---|
| BTC perp | **$2.46B** | $4.6–5.4M within ±2.4 bp | 0.09% | Funding; ADL; mark/oracle mechanics |
| UBTC/USDC spot | $42M (perp is 58× bigger) | $0.39–0.44M within ±2.9 bp | 0.14% | Unit bridge custody; deeper crash wicks |
| ETH perp / UETH spot | $1.12B / $20M | ~$9.8M / $0.27–0.53M within ±10 bp | 0.09% / 0.14% | as above |
| SOL perp / USOL spot | $271M / $8M | $2.6M / $0.15M within ±10 bp | 0.09% / 0.14% | as above |
| HYPE/USDC spot (native) | $71M | – | 0.14% | Venue's own token (correlated with venue risk) |

Fees from [fees](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/fees), confirmed by the live `userFees` endpoint: perps 0.045% taker / 0.015% maker; spot 0.070% / 0.040% at tier 0.

On 10 Oct 2025 (measured 4-hour lows), Hyperliquid's **spot** UBTC fell to 100,100 while Coinbase BTC only fell to 107,000 (6–8% deeper); the HL BTC **perp** fell to 100,837 against 101,516 on Binance's perp (0.7–4.7% deeper across BTC/ETH/SOL). Thin books wick further in a crash, and that is exactly when stops fire.

### 3.2 What actually makes a position dangerous for a beginner

D-06 exists to rule out three things: **losing more than the money put in**, **being forced out by liquidation**, and **betting against the market (shorting) without understanding unlimited-loss risk**. A **1× isolated long perp** avoids all three:

- At 1× isolated, the most it can lose is its own margin, the same as owning the coin outright. Its liquidation price starts near zero (§2) and only creeps up through funding payments; at 2024's 24% average funding, it would take years.
- It cannot go short if the risk engine only allows reduce-only sells against a long.

What it adds compared with spot: **hourly funding** (a cost), **ADL** (an unwanted, but profitable, early exit), **mark-price** stop triggering, governance or delisting settlement, and a **regulatory label**: a perp is a derivative even at 1× (ESMA's 24 Feb 2026 statement treats perps as CFDs for firms serving EU retail, [ESMA35-243228190-8024](https://www.dirittobancario.it/wp-content/uploads/2026/02/Dichiarazione-ESMA-24-febbraio-2026-ESMA35-243228190-8024.pdf)).

What spot on Hyperliquid adds compared with a 1× perp: **bridge custody risk** (if Unit's guardians fail, UBTC can lose its backing, a total-loss risk a 1× perp doesn't carry), thinner books and deeper crash wicks, and higher fees. Spot avoids funding, so it is cheaper for long holds: at 2026's 5.2% BTC funding, spot's higher fees are repaid after about **3.5 days** of holding; at 2024's 24%, after about **0.8 days**.

### 3.3 Recommendation

We **confirm the lead's starting point, with three tightenings**:

1. **Reword D-06 to "no leverage and no shorting until P4"** (long or flat, gross exposure ≤ the sleeve's equity). Two "vehicles" are allowed for that exposure on Hyperliquid in every phase from P1:
   - **HL spot** on an allow-listed pair (UBTC, UETH, USOL, HYPE against USDC).
   - **1× isolated long perp** ("spot-like exposure") on an allow-listed validator-run perp (BTC, ETH; SOL after review). Leverage is set to 1× and isolated before any order and re-checked at every reconciliation; *effective* leverage (position notional at mark ÷ isolated margin) must stay ≤ 1.0, and the gateway tops up margin from the sleeve or reduces the position if funding pushes it above 1.05.
2. **Default vehicle:** BTC and ETH templates default to the **1× isolated perp** (deep book, no bridge IOU, lower fees), and HYPE to spot. Before a bot is deployed, Ubik shows a plain-language comparison: expected funding cost at the strategy's average holding time (using trailing 30-day and 2024-stress funding), fee difference, and custody chain. The owner can switch the vehicle with one click. If the owner prefers the stricter reading of D-06, a setting "spot only on Hyperliquid" disables perps entirely (open question Q-HL-02).
3. **Shorting and leverage above 1× stay in P4, not P3** (stricter than the lead's suggestion). P3's job is to prove the live pipeline on the simplest exposure; adding shorts there would mix two unknowns. In P4, under the Adventurous profile only, after a Learn module and a separate legal/terms review (CMP-010): isolated margin only, **default ceiling 2×, absolute hard ceiling 3×** for BTC/ETH (lower for others), liquidation distance ≥ 3× daily ATR (RK-036), and funding in P&L.

Always out of scope:

| Category | Phase | Why |
|---|---|---|
| HIP-3 builder-deployed perps (equities, commodities, `dex:COIN`) | Not before P4, and only after a separate review; read-only "macro context" display possible in P2 | Deployer sets the oracle and can halt and settle; mostly strict-isolated with higher ADL risk; 2× fees; oracle incidents (Trade[XYZ] SK Hynix, July 2026, ~$57–60M liquidated, [Cointelegraph](https://cointelegraph.com/news/trade-xyz-sk-hynix-perp-liquidations-price-anomaly)) |
| HIP-4 outcome markets (sports, binaries) | Never | Gambling-like; a separate undisclosed excluded-jurisdiction list in the Terms (s.1.7–1.8) |
| HIP-1 long-tail spot tokens, thin perps | Never through P3 | Manipulation targets (JELLY, POPCAT); also the main "trade-to-drain" route for a stolen agent key (§5.3) |
| Vaults, HLP deposits, staking, staking-account linking | Never automated | Not trading; staking linking is permanent and lets the staking account take funds ([fees](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/fees)) |
| USDH, USDT0, USDE quote pairs | Not through P3 | USDH is being sunset (USDH/USDC at 0.991 on a thin book, 2026-10-18); keep to USDC |

## 4. Connectivity design

### 4.1 Library choice: own thin client

| Option | Verdict | Reasons |
|---|---|---|
| **ccxt** 4.5.88 ([source](https://github.com/ccxt/ccxt/blob/master/python/ccxt/hyperliquid.py)) | Public data and cross-checks only; **not** for HL trading | On first private call it approves ccxt's own builder fee (+1 bp on every order) and sets referrer "CCXT1"; raises on the first error in a batch and loses the other orders' acks; its rate limiter under-counts info weight about 2×; no trailing stops |
| **Official Python SDK** 0.24.0, MIT ([GitHub](https://github.com/hyperliquid-dex/hyperliquid-python-sdk)) | Use **only its signing function** (pinned), as a reference and test oracle | Synchronous and float-based (conflicts with EX-009 decimals); its WebSocket manager does not reconnect by default; nonce = current millisecond, so two actions in the same ms collide |
| **Own thin client** in Python asyncio | **Recommended** | About 1,500–3,000 lines: wire format, signer, nonce manager, WS client, rate limiter, reconciler. Full control of decimals, errors and limits |

References for design: NautilusTrader's Hyperliquid adapter (LGPL, [docs](https://nautilustrader.io/docs/latest/integrations/hyperliquid)) and the TypeScript `@nktkas/hyperliquid` SDK (MIT), which already has reconnecting WebSockets and every action. Budget **weeks, not a day** (NFR-016's "a venue in ≤ 1 day via ccxt" does not apply).

### 4.2 Components (behind the existing ports)

Strategies, risk and OMS still see only `MarketDataFeed` and `ExecutionVenue` (AR-005).

```text
HyperliquidFeed  (engine/worker; public, no keys; P1)
  ws client ─ subscribe: trades, bbo, l2Book(fast), l2Book, activeAssetCtx, candle
  rest client ─ meta, spotMeta, candleSnapshot, fundingHistory, userFees(public)
HyperliquidVenue (gateway only; P3)
  wire builder ─ Decimal → normalized string, no trailing zeros, lowercase addresses
  signer      ─ L1 "phantom agent" EIP-712 scheme only (agent key); never user-signed actions
  nonce mgr   ─ one per agent; max(now_ms, last+1); single process per agent
  rate budget ─ IP weight bucket + address action bucket (§4.7)
  reconciler  ─ clearinghouseState / spotClearinghouseState / frontendOpenOrders / userFillsByTime
  account ws  ─ orderUpdates, userFills, userEvents, userFundings, userNonFundingLedgerUpdates
HyperliquidShadow (S3; public, no keys) ─ the account ws + reconciler, read-only
```

**Endpoints.** Mainnet REST `https://api.hyperliquid.xyz` (`POST /info` unsigned reads; `POST /exchange` signed actions) and WebSocket `wss://api.hyperliquid.xyz/ws`; testnet `api.hyperliquid-testnet.xyz` ([API](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api)). The API is officially "v0" and a breaking "v1" is announced without a date ([notation](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/notation)). API changes are announced on Telegram `t.me/hyperliquid_api` and Discord `#api-traders`, not a web page; AR-006's changelog watch must cover those.

### 4.3 Signing and nonces

- **Two signing schemes** ([signing](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/signing)). *L1 actions* (orders, cancels, modify, leverage, `scheduleCancel`, `noop`) are signed by the agent. *User-signed actions* (withdraw, transfers, `approveAgent`, `approveBuilderFee`) need the master. The gateway implements **only the L1 scheme**; code that builds user-signed actions must not exist in Ubik (HL-024).
- Common signing mistakes per the docs: wrong field order in msgpack, trailing zeros, uppercase addresses. A bad signature shows up as "User or API Wallet 0x… does not exist" with an address that isn't yours. Ubik pins the signer with **test vectors cross-checked against the official SDK**.
- **Nonces** ([nonces](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets)): the chain keeps the 100 highest nonces per signer; a new one must be higher than the smallest of those and never reused, and must be within (block time − 2 days, + 1 day). Ubik uses one agent per account and one process per agent, with a monotonic counter `max(now_ms, last + 1)` persisted across restarts.

### 4.4 Client order IDs (cloid) and unknown orders

- Ubik's deterministic client order ID (EX-003) is mapped to Hyperliquid's optional 128-bit **cloid**: `cloid = "0x" + first 16 bytes of SHA-256(account_address ‖ client_order_id)`. The mapping is stored with the order before sending (EX-002). A cloid is never reused. Whether duplicates are rejected, and with what error, is **unverified**: test on testnet (HL-014).
- Every order action carries **`expiresAfter`** = now + 10 s (the docs' optional expiry). After that time the action can never execute. Caution: an action rejected for a stale `expiresAfter` costs 5× the normal address budget ([exchange endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint)), so the clock must be accurate (RK-027, NFR-013).
- **UNKNOWN-order protocol** (implements EX-004/EX-005 for this venue):
  1. On a timeout or dropped connection, mark the order UNKNOWN and block the symbol and sleeve.
  2. Query `orderStatus` by cloid (weight 2). If found, adopt its state.
  3. Optionally send a `noop` with the **same nonce** as the lost action: the docs call this "a more effective way to cancel in-flight orders than the cancel action".
  4. Once `expiresAfter` plus a 5 s margin has passed and the query returns `unknownOid`, mark it REJECTED (never placed).
  This resolves every unknown order deterministically within about 20 s, inside EX-005's 30 s.
- **Batches.** One pre-validation error rejects the whole batch with a single message; otherwise each order gets its own status in the same order as sent ([error responses](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/error-responses)). The adapter handles both shapes.
- **Order statuses** map to Ubik's state machine: `open` → OPEN; `filled` → FILLED; `canceled`, `scheduledCancel`, `marginCanceled`, `reduceOnlyCanceled`, `siblingFilledCanceled`, `delistedCanceled`, `liquidatedCanceled`, `selfTradeCanceled`, `openInterestCapCanceled` → CANCELLED with a reason; the `…Rejected` family → REJECTED with a reason. A `nonUserCancel` event (a cancel Ubik didn't send) raises a Warning.

### 4.5 WebSocket feeds and reconciliation

- **Heartbeat:** the server closes a silent connection after 60 s; Ubik pings every 30 s ([timeouts](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket/timeouts-and-heartbeats)). Disconnects "may happen periodically and without announcement"; on reconnect, user streams resend a snapshot (`isSnapshot: true`) which Ubik de-duplicates ([WebSocket](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket)).
- **Market channels** ([subscriptions](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket/subscriptions)): `trades` (unique id = block time + coin + tid), `bbo`, `l2Book` (**full snapshots, not deltas**: 20 levels about every 5.4 s, or 5 levels about every 0.5 s with `fast`), `activeAssetCtx` (mark, oracle, funding, open interest, about every 1 s), `candle`.
- **Account channels** always use the **account** address (an agent address returns nothing): `orderUpdates`, `userFills`, `userEvents` (fills, funding, liquidations, non-user cancels), `userFundings`, `userNonFundingLedgerUpdates` (deposits, withdrawals, transfers).
- **Reconciliation** (EX-007) after every reconnect, at startup and every 60 s: `clearinghouseState` (perp positions, leverage, liquidation price; weight 2), `spotClearinghouseState` (weight 2), `frontendOpenOrders` (includes trigger orders; weight 20), `userFillsByTime` since the last stored fill, and `orderStatus` for in-flight cloids.
- **Ubik must store everything itself.** The venue serves only the 2,000 most recent orders and the **10,000 most recent fills**; funding and ledger queries return 500 rows per page.
- **Per-IP WebSocket limits:** 10 connections, 1,000 subscriptions, 10 distinct users, 2,000 messages sent per minute ([rate limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits)).

### 4.6 Order-type mapping and precision

| Ubik order (EX-014) | Hyperliquid mapping | Notes |
|---|---|---|
| Protected market (EX-015) | Limit with `tif: Ioc` at best price ± 0.5% cap | The API has no true market order. The SDK and ccxt default to a 5% cap; Ubik's is 0.5%. "Could not immediately match" = no fill |
| Limit GTC | `tif: Gtc` | |
| Post-only | `tif: Alo` | Rejected (not repriced) if it would cross |
| IOC | `tif: Ioc` | |
| FOK, GTD | Not native | Emulate: GTD = GTC + bot-side cancel; FOK not offered |
| Stop-market / catastrophe stop | Trigger order `{isMarket: true, tpsl: "sl"}`, **reduce-only** | Triggers on **mark**; executes with up to **10% slippage** ([TP/SL](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/take-profit-and-stop-loss-orders-tp-sl)) |
| Stop-limit | Trigger order `{isMarket: false}` with explicit limit | Prefer this with a wide limit (e.g. 3%) over the 10% default |
| Take-profit | Trigger `tpsl: "tp"`, reduce-only | |
| OCO / bracket | `grouping: normalTpsl` (children placed when the parent fills) or `positionTpsl` | |
| Trailing stop | Bot-side in P1–P3 | Native trailing stops launched ~Sept 2026 but no SDK supports them yet (unverified details) |
| TWAP | Bot-side (EX-017) | Native TWAP exists; its minimum size is unverified/conflicting |

**Precision** ([tick and lot size](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/tick-and-lot-size)): prices have at most 5 significant figures and at most `6 − szDecimals` decimals for perps (`8 − szDecimals` for spot); integer prices are always valid (so BTC's tick is $1). Sizes round to `szDecimals`. Minimum order: **$10** for perps, 10 quote units for spot. Orders too far from the oracle price are rejected (threshold unverified; make it a registry parameter). Self-trade prevention cancels the resting order ("expire maker", [STP](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/self-trade-prevention)).

**Symbols.** Perps are addressed by their index in `meta` (BTC = 0); spot pairs as `10000 + index`, named `@142` etc. (UBTC/USDC on 2026-10-18), and **testnet IDs differ from mainnet**. Ubik keys instruments on (network, asset id) and shows friendly aliases ("BTC spot (UBTC, bridged)").

### 4.7 Rate limits for a small account

Two separate budgets ([rate limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits), re-read 2026-10-18):

- **Per IP:** 1,200 weight per minute. Most account reads weigh 20; `l2Book`, `allMids`, `clearinghouseState`, `orderStatus` weigh 2; order actions weigh 1 + batch/40. D-37's 70% gives Ubik **840 per minute**.
- **Per address (actions only):** "1 request per 1 USDC traded cumulatively since address inception", plus an initial buffer of **10,000**. When exhausted, the address may send **one request every 10 seconds**, which would cripple exits. Cancels get extra room (`min(limit + 100000, limit × 2)`). A batch of *n* orders counts *n*.
- **Under congestion**, each address is limited to "2x their previous day maker share" of block space. For an address that never makes markets, what this means is **unverified**; assume stops must already be resting before a crash, not sent during one.

What it means: a fresh pilot account trading $1,000 a day earns about 1,000 actions a day on top of 10,000. A 1h–1d strategy needs a few dozen. A limit-chase algorithm requoting every 5 s would burn the whole buffer in under a day. Ubik therefore:
- keeps its own token bucket for each budget at ≤ 70%, and reserves the last 30% of the address budget for cancels and exits;
- reads `userRateLimit` (weight 20) every 5 minutes and shows it as a gauge; alerts below 30% headroom;
- refuses high-churn algorithms (EX-016 chase with < 30 s reprice) when the projected daily actions exceed the earned budget;
- may buy extra capacity with `reserveRequestWeight` (0.0005 USDC per action; $5 buys 10,000) only within an owner-set monthly cap (default $5).

### 4.8 Dead-man's switch

`scheduleCancel` cancels **all** open orders at a set time unless refreshed; at most 10 fires a day ([exchange endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint)). An error string captured in ccxt's source says it is locked until the account has traded **$1,000,000** in volume; the official docs list no gate (**unverified**; our search found no official confirmation). It is also unverified whether it cancels resting stop (trigger) orders, which would strip catastrophe stops.

Recommendation: the **local watchdog is the dead-man's switch** for Hyperliquid (RK-025's fallback path). Once the account unlocks `scheduleCancel` *and* a testnet test shows it leaves trigger orders alone, use it in addition, refreshed every 60 s (1,440 actions a day: budget accordingly). Because P1–P3 positions are at most 1×, an outage cannot liquidate them; the remaining risk is an unmanaged position, which the resting catastrophe stop covers.

## 5. Key and wallet security

### 5.1 Account layout

```text
Owner's savings address (hardware wallet)       ── never connected to Ubik
        │ owner deposits pilot budget by hand
        ▼
Ubik trading account (master = hardware wallet, dedicated address)
        │ approveAgent (signed on the hardware wallet, in the official web app)
        ▼
Agent key "ubik-<date>"  ── lives only in the execution gateway (SEC-012 envelope encryption)
        │ later, after $100k volume:
        ▼
Sub-accounts per bot group / pilot, each with its own agent
```

- **The master key never touches the server.** It is a hardware-wallet address, used only to approve or replace agents, set the account mode, and withdraw. Ubik never asks for it, never stores it, and contains no code that builds user-signed actions.
- **A dedicated trading account** holds only the money Ubik may trade. **Sub-accounts** (a separate balance under the same master, signed via `vaultAddress`) can only be created after **$100,000 of traded volume** ([sub-accounts](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/sub-accounts)), so pilots start in a fresh dedicated master account. Once available, use one sub-account per pilot or bot group, each with **its own agent** (agents shared across sub-accounts share a nonce set).
- **Native multi-sig** exists, but once converted *every* action, including each order, needs multiple signatures ([multi-sig](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/multi-sig)). Fine for the savings address, unsuitable for the trading account.

### 5.2 Agent lifecycle

- **Approve:** in the official web app's API page, signed on the hardware wallet, a **named** agent with `valid_until` = 90 days (maximum 180; matches SEC-015's 90-day rotation). The web-app path is from third-party guides (**unverified** in official docs). The key is shown once and pasted into Ubik's gateway, together with the **master address**.
- **Verify** at add, every start and daily (the SEC-008 equivalent): the agent's derived address differs from the account address (Ubik refuses a master key); `extraAgents` for the account lists exactly the expected agents with their expiry; any unknown agent → disarm and Critical alert.
- **Rotate** at T−14 days with a wizard: always a **new address** (the docs warn that reusing an agent address lets old signed actions be replayed after pruning), approve the new one, verify, then retire the old one by approving a replacement under the same name. Agents are also pruned if the account balance hits zero.
- **Revoke ("Key compromised", SEC-015):** kill switch L2, then the owner approves a new agent under the same name from the hardware wallet (this deregisters the old one), then sweeps funds to the savings address.

### 5.3 What an attacker with the agent key can still do

There is **no IP allow-list** for agents (the docs describe none), so a stolen agent key works from anywhere. It cannot withdraw, but it can:

1. **Trade-to-drain:** place resting orders from the attacker's own account in a thin market, then use the stolen key to trade into them at absurd prices. Value moves through fills. This is the main threat.
2. Raise leverage to the maximum and open large positions so they liquidate into a book the attacker controls.
3. Move collateral between the account's own spot and perp balances (`agentSendAsset`; destination must be the same account), and change the account's margin mode.
4. Cancel Ubik's protective stops and burn the address rate budget.
5. Possibly spend small amounts via `reserveRequestWeight`; whether agents can sign sub-account or vault transfers is **unverified** (test on testnet).

Mitigations:
- **Exposure on Hyperliquid is capped** to the pilot budget; profits above the cap are swept out by the owner (RK-032).
- **Symbol allow-list** in the gateway (BTC, ETH perps; UBTC, UETH, USOL, HYPE spot), checked against asset id, plus **price bands** (reject limit prices more than 1% from mark, or the protected-market cap), per-order, per-day and gross notional caps, a leverage cap of 1×, and a maximum open-order count. These protect against Ubik's own bugs; against a stolen key they only help detection.
- **Independent fill watcher:** any fill, order, leverage change, margin-mode change, transfer or agent not originated by Ubik (no Ubik cloid) → Critical alert and kill switch L2 (EX-013). It runs on the public address, so it keeps working even if the gateway is compromised.
- **Short agent expiry** and **one agent per account/process**.

## 6. Data and backtesting

### 6.1 What history exists

| Source | What | From | Cost |
|---|---|---|---|
| Public API `candleSnapshot` | Last **5,000 candles** per interval: 1m ≈ 3.5 days; 1h ≈ 208 days; 4h from 2024-07; 1d from 2020 (but synthetic `v=0` rows before **2023-02-26**) | — | Free |
| Public API `fundingHistory` | Full hourly funding (8-hourly until ~2023-06-08) | 2023-05-12 | Free |
| Official S3 archive `hyperliquid-archive` | L2 snapshots, asset contexts (mark, oracle, funding, OI). No candles, no spot. "No guarantee of timely updates" ([historical data](https://hyperliquid.gitbook.io/hyperliquid-docs/historical-data)) | ~2023-09 | Requester-pays (AWS account; a few dollars) |
| Official `hl-mainnet-node-data` | Node fills by block (format changed 2025-07-27) | — | Requester-pays |
| Hydromancer Reservoir ([docs](https://docs.hydromancer.xyz/reservoir/hyperliquid)) | Fills incl. liquidations/ADL, 1s OHLCV, L2 per minute | Complete from 2025-07-28 | "Free"; requester-pays S3; licence unverified |
| Tardis.dev ([HL](https://docs.tardis.dev/historical-data-details/hyperliquid)) | Trades, L2, BBO, asset contexts incl. mark | 2024-10-29 | From $350/month (academic) / $700 (solo); first day of each month free |
| HypeDexer, Dwellir, GoldRush | 1m candles beyond retention (from 2025-03 / 2025-07) | — | Free tiers to ~$49–249/month |
| **Binance** (`data.binance.vision`) | USDⓈ-M perp 1m klines, mark price, funding; spot klines | 2019–2020 | Free (CC BY-NC-SA, personal use) |
| **Coinbase** USD 1m | Proxy for HL spot (USDC) | years | Free REST |

Native Hyperliquid history cannot fill D-38's 4-year window: real perp trading starts 2023-02-26, UBTC spot 2025-02, UETH 2025-03, USOL 2025-05.

### 6.2 Proxy data, honestly labelled

Measured over 4,901 one-minute closes (2026-10-15 to 10-18): HL BTC perp − Binance BTCUSDT spot averaged −4.0 bp (sd 1.1); HL UBTC − Coinbase BTC-USD +0.3 bp (sd 1.7); HL spot − Binance spot −13 bp (the USDT/USD premium). 1-minute return correlations were 0.96–0.98 for perps vs Binance. Over 2.3 years of 4h closes, perp vs Binance spot averaged −0.85 bp. **Closes agree within a few basis points; crash wicks differ by whole percentage points** (§3.1).

Rules:
- Proxy **HL perps with Binance USDⓈ-M perps** (price, mark price and funding; before 2023-05, funding = Binance 8h rate ÷ 8 per hour). Proxy **HL spot with Coinbase USD** (fallback: Binance spot minus ~13 bp).
- Every bar carries `source_venue` and a **fidelity grade**: **A** HL-native tick/L2 · **B** HL-native 1m · **C** HL-native ≥ 1h with proxy intrabar · **D** CEX proxy, basis-aligned.
- Every report shows the grade mix, a **"native window only"** result next to the full window, and a **proxy-wick warning** citing 10 Oct 2025.
- Stops and liquidations are simulated on the **mark-price path** where available (recorded `activeAssetCtx`, archive `asset_ctxs`, or Binance mark price as proxy), and the report shows both mark-path and trade-path results.

### 6.3 Record from day one

From the first P1 start (extends D-18, DA-003), with one or two WebSocket connections:
- **Instruments:** BTC, ETH, SOL perps; UBTC, UETH, USOL spot; HYPE spot and perp.
- **Channels:** `trades`, `bbo`, `l2Book` fast (5 levels, ~0.5 s), `l2Book` 20 levels (~5.4 s) plus an `nSigFigs: 4` variant for wider depth bands, `activeAssetCtx` (~1 s), closed 1m `candle`.
- **Daily REST snapshots:** `meta`, `spotMeta` (tick/size rules, max leverage, margin tables, delisting), public fee schedule, `perpsAtOpenInterestCap`, `predictedFundings` (free HL vs Binance vs Bybit funding). **Hourly:** `fundingHistory`.
- **Size:** about 40–80 MB/day, **15–30 GB/year** (extrapolated from Tardis file sizes).
- DA-003's "top-20 L2 every second" is not offered by the public feed; replace it with the fast 5-level book plus the 20-level snapshot.
- **Data-quality rules:** drop synthetic `v=0, n=0` daily candles before 2023-02-26; quarantine listing-day garbage prints (UBTC's first candle opened at 100,002,060); normalize 8-hourly early funding; candle close time = open + interval − 1 ms.

### 6.4 SimVenue Hyperliquid profile

`SimVenue` (one simulator for backtest and paper) gets a Hyperliquid profile read from the venue registry:
- Fees by tier (perps 4.5 / 1.5 bp, spot 7 / 4 bp at tier 0; tier volume = perps + 2 × spot over 14 days); unverified tiers use tier 0 (D-41).
- Tick/size/minimum rules from §4.6, effective-dated; oracle-distance rejection as a parameter.
- ALO rejected if crossing; IOC remainder cancelled or rejected if nothing matches; self-trade expires the maker.
- Triggers fire on **mark**; market TP/SL fill with up to 10% slippage (stop fill = worse of trigger and next price, per 04 §4.3).
- **Hourly funding** at the recorded (or proxy) rate × oracle price; funding debits the isolated margin; effective-leverage drift checked.
- **Liquidation** at maintenance margin on mark (first stage: market orders; above $100k only 20% at a time with a 30 s cooldown; backstop below ⅔ of maintenance margin forfeits it) and **ADL** as a scenario in the stress library.
- Venue states: network-upgrade **post-only period** (IOC and market orders don't fill), OI-cap rejections, delisting settlement (1-hour oracle TWAP, [delisting](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/delisting)), and API outage while the chain runs (29 Jul 2025, 37 minutes, [Cointelegraph](https://cointelegraph.com/news/hyperliquid-reimburses-2m-crypto-traders-outage)).
- Address rate budget, so paper runs reveal churn problems before live.

### 6.5 Testnet's role

Testnet (`app.hyperliquid-testnet.xyz`) gives 1,000 mock USDC from a faucet, **only to addresses that have deposited on mainnet** ([faucet](https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/testnet-faucet)). On 2026-10-18 its BTC/ETH/SOL perps tracked mainnet within 0.02–0.5%, but the HYPE perp was 43.48 vs 76.43 on mainnet, spot markets were unrelated (testnet PURR 4.56 vs 0.087), and BTC depth was about 150× thinner. Asset IDs differ. **Testnet proves the adapter's order lifecycle only; it is never evidence about a strategy** (PT-016).

## 7. Path to live on Hyperliquid

| Stage (D-13) | On Hyperliquid | Key needed? | Exit criteria (additions to 05 §9.1) |
|---|---|---|---|
| S0 Backtest | `SimVenue` HL profile; fidelity grades; mark-path stops | No | Unchanged; plus the native-window result shown. Proxy segments may count toward the ≥ 100 round trips only if the native window has ≥ 20 (Q-HL-05) |
| S1 Paper | `SimVenue` on live HL public feed | No | Unchanged (≥ 30 days, ≥ 20 round trips) |
| S2 Exchange demo | (a) **Testnet conformance suite**: signing, nonces, cloid, every order type, rejections, batch errors, UNKNOWN-order protocol, reconnect, agent approval/rotation/revocation, `scheduleCancel` behaviour on trigger orders, kill switch L1–L3. (b) Then a few **minimum-size ($10–20) mainnet orders** after live arming, exactly as 05 §9.1 does for venues without a realistic demo | Testnet agent; then mainnet agent | Suite passes; ≥ 7 days on testnet with no unresolved UNKNOWN orders; mainnet minimum-size round trip reconciles exactly |
| S3 Shadow | Ubik reads the dedicated trading account **by its public address** (balances, positions) and prices would-be orders against the live book; nothing is sent. Booked to the paper schema (D-29) | **No key at all** | ≥ 14 days; shadow costs within the model |
| S4 Pilot | Dedicated trading account (or sub-account once unlocked), funded only with the pilot; ≤ $500 per strategy, ≤ $100 per order (≥ $10 minimum); 1× only; allow-listed symbols | Agent key, ≤ 90-day expiry | Unchanged; plus zero unexplained fills, agent expiry > 30 days at promotion |
| S5 Scaled | Per-venue cap on Hyperliquid set by the owner (RK-032); periodic sweep of profits | Agent key | Unchanged |

**Funding the account** (owner, by hand): buy USDC on a CEX or hold it on another chain; deposit via CCTP V2 (preferred) or the official app's deposit flow; minimum deposit amounts apply (the legacy bridge loses < 5 USDC). Ubik's onboarding guide shows the steps but never signs or sends a deposit. Ubik books deposits and withdrawals from `userNonFundingLedgerUpdates` (read-only) for cost basis (CMP-013).

**Account mode.** Hyperliquid has Standard, Unified and Portfolio-margin account modes; Unified and Portfolio margin are limited to 50,000 actions a day ([account abstraction modes](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/account-abstraction-modes)). We recommend **Standard** (spot and perp balances kept separate, so a spot loss can never margin-call a perp and the perp state is meaningful for reconciliation), set once by the owner. Ubik reads the mode (`userAbstraction`) at start and refuses to arm if it changed. Whether new accounts default to Unified is unverified.

## 8. Risk, legal, privacy and tax

### 8.1 Jurisdiction gate

Hyperliquid's Terms of Use (version "Last updated on June 15, 2026", [app.hyperliquid.xyz/terms](https://app.hyperliquid.xyz/terms), read from the app bundle) define **Restricted Persons** in s.1.6: persons or entities residing, located, incorporated or registered in **the United States or Ontario (Canada)**; those in sanctioned or export-controlled territories; and **citizens of those sanctioned territories** wherever they live. They "are strictly prohibited". s.1.9 and s.3.1.5 forbid VPNs, proxies or false statements to get around this. Third-party summaries agree on the US and Ontario ([datawallet](https://www.datawallet.com/es/cripto/hyperliquid-supported-and-restricted-countries)); the sanctions list itself is not published in the Terms (Cuba, Iran, North Korea, Syria, Crimea/Donetsk/Luhansk per secondary sources; Russia/Belarus **unverified**).

- The Terms literally govern "the Interface" (the web app); no separate API terms were found (**unverified**). The underlying law (US derivatives rules, Ontario securities rules, sanctions) applies however you connect, so **Ubik treats API use by a Restricted Person as prohibited**.
- **The owner (Colombian citizen, Colombia residence to confirm, Q-01)** is not a Restricted Person on the current text. The server location also matters: a server in the US or Ontario would put "location" in doubt, so the server should sit in Colombia or another non-restricted country (Q-05).
- Ubik enforces: live trading on Hyperliquid cannot be armed unless the jurisdiction profile (CMP-004) shows residence, location and citizenship outside the restricted list, **and** the server's egress country (AR-034) is outside it. A restricted answer disables Hyperliquid live trading with a plain explanation, offers the CEX profile instead, and never suggests a workaround. The Terms are re-checked every 3 months and at each phase gate, because changes take effect on posting (s.7.2).
- Regulators' views elsewhere, for context: the UK FCA put Hyperliquid on its warning list (May 2026, [FCA](https://www.fca.org.uk/news/warnings/hyperliquid)); ESMA treats perps as CFDs for firms serving EU retail (Feb 2026); MAS's investor alert (June 2026) is **unverified** from a primary source. None makes personal use illegal outside the restricted list, but users get **no investor protection or compensation scheme**. The owner's own country's view on using foreign derivatives venues needs the lawyer review already planned (CMP-002).

### 8.2 Venue, bridge and validator risk (incident lessons)

| Date | Event | Lesson for Ubik |
|---|---|---|
| Dec 2024 | DPRK-linked addresses trading; ~$500M outflows in a day amid validator-concentration fears ([Cointelegraph](https://cointelegraph.com/news/hyperliquid-outflow-north-korea-exploit-fears)) | Venue confidence can move fast; cap exposure |
| Mar 2025 | 50× ETH whale left HLP a ~$4M loss; leverage caps cut ([The Block](https://www.theblock.co/post/345866/hype-drop-hlp-vault-loss-hyperliquid-whale-liquidation)) | Rules change mid-flight; registry fields must be effective-dated |
| Mar 2025 | JELLY squeeze; validators delisted and force-settled at a chosen price | Thin markets can be settled at prices you didn't choose; allow-list majors only |
| 29 Jul 2025 | API outage 37 min while the chain ran; ~$2M refunded at discretion, KYC above $10k | You may be unable to cancel; positions must survive an outage unattended |
| 10 Oct 2025 | ~$10B liquidated; first cross-margin ADL | Expect fills you didn't order; reconcile and re-plan |
| Nov 2025 | POPCAT manipulation; bridge emergency-locked ~1 h | Withdrawals are not guaranteed instantly |
| Oct 2025, Aug 2026 | Users' master keys stolen (malware ~$21M; phishing ad ~$550k) | Most real losses are key theft or phishing, not protocol failure |

No verified 2026 protocol, bridge or consensus breach was found (absence **unverified**). Ubik's response: a **venue concentration cap** (RK-032) on everything held on Hyperliquid; Hyperliquid's own token (HYPE) counts toward the same cap because its value is tied to the venue; ADL, delisting and force-settlement events are booked as "venue-initiated" fills and alert the owner; a venue-status signal (API errors, `exchangeStatus`) sets the venue REDUCE_ONLY (RK-033/034); USDC is monitored for peg like any stablecoin (RK-029). Refunds after incidents have been discretionary, never contractual.

### 8.3 Privacy: the account is public

Everything in the trading account is visible to anyone: balances, positions, entry prices, **resting stop levels**, liquidation prices, fills and every deposit and withdrawal. If the address is ever linked to the owner (a CEX withdrawal, a social post, a KYC-gated refund), their trading and net worth are linkable.

- Disclose this plainly before arming, and in Guided mode.
- Never publish or log the address outside Ubik; keep the savings address unlinked from the trading address by transfer history where lawful.
- At pilot size the main harm is information, not stop-hunting. Still: keep catastrophe stops (RK-018) since outage protection matters more, but keep them wide (1.5× the bot stop); prefer bot-managed precise stops (not visible) plus wide native catastrophe stops.
- Assume any live strategy is copyable; don't rely on secrecy as an edge.
- Never use privacy tools to evade KYC, sanctions or the jurisdiction gate.

### 8.4 Stablecoin risk

Use **USDC only** as collateral and quote through P3. Risks: Circle can freeze addresses; CCTP attestation; the legacy bridge's validator quorum for its remaining USDC. USDH is being sunset ([The Block](https://www.theblock.co/post/401233/coinbase-hyperliquid-official-deployer-usdc)); other quote assets are excluded. RK-028/029 (value at market, peg monitoring) apply unchanged.

### 8.5 Records and tax

- **Hyperliquid issues no tax forms**; Terms s.4.4 puts tax entirely on the user. **Ubik's ledger is the only complete record**, because the venue serves only the latest 10,000 fills.
- Persist at ingest, with block time and transaction hash: every order event, fill (fee, fee token, maker/taker, `closedPnl`), **hourly funding payment** (as income or expense per payment), liquidation, ADL, deposit, withdrawal and transfer.
- Exports (CMP-014): generic CSV/JSON (P1), Koinly and CoinTracker (P3), with funding as its own row type. Address-sync tax tools (Koinly, Awaken, KoinX) cap at about 10k transactions and handle perps/funding unevenly ([Koinly](https://koinly.io/integrations/hyperliquid/)), so Ubik's export is the reference; validate a sample import before P3.
- **Colombia (if confirmed):** crypto held at 31 December is declared as assets on form 210, and gains as income or as occasional gains if held more than two years ([Asuntos Legales](https://www.asuntoslegales.com.co/actualidad/asi-es-como-debe-declarar-renta-de-forma-correcta-si-tiene-operaciones-en-criptoactivos-4170970)); providers serving Colombian residents report to DIAN under CARF from 2027 ([La FM](https://www.lafm.com.co/economia/declaracion-de-renta-como-declarar-criptomonedas-y-dolares-digitales-407024)). How perps and funding are treated is **unverified**: confirm with a Colombian accountant before P3.

## 9. Requirements

### 9.1 Venue, scope and market type

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| HL-001 | The jurisdiction profile (CMP-004/AR-001) shall support Hyperliquid as the primary venue with Binance as secondary; the venue registry (AR-002) shall hold Hyperliquid mainnet and testnet as separate networks. | Must | P1 | Owner directive 2026-10-18; Q-01 profile |
| HL-002 | Ubik shall allow only long or flat exposure with gross notional ≤ sleeve equity until P4, through either HL spot or a 1× isolated long perp on allow-listed instruments. Shorts and leverage > 1× shall be rejected by the risk engine (replaces RK-035/ST-010 wording for HL). | Must | P1 | §3.3; beginner safety without spot-only |
| HL-003 | For perp positions, Ubik shall set leverage to 1× isolated (`updateLeverage`) before the first order on each asset, assert it at every reconciliation, and set REDUCE_ONLY plus a Critical alert (live) on any mismatch. | Must | P3 (simulated P1) | A UI change or stolen key can alter leverage |
| HL-004 | Ubik shall keep effective leverage (notional at mark ÷ isolated margin) ≤ 1.0, topping up margin or reducing size when it exceeds 1.05 because of funding. | Must | P1 | Funding erodes isolated margin |
| HL-005 | Every perp position shall show its liquidation price, distance to it, accrued funding and funding as an annual rate, in all modes. | Must | P1 | Explain everything |
| HL-006 | Before deploying a bot on HL, Ubik shall show a vehicle comparison (spot vs 1× perp): fees, expected funding at the strategy's average holding time (trailing 30-day and 2024-stress rates), custody chain and liquidity; the owner may switch vehicle. A "spot only on Hyperliquid" setting shall disable perps. | Must | P1 | §3.3 point 2 |
| HL-007 | HIP-3, HIP-4, HIP-1 long-tail tokens, vaults/HLP, staking and non-USDC quote pairs shall be absent from the tradable instrument registry through P3; HIP-4 permanently. | Must | P1 | §3.3 |
| HL-008 | Shorting and leverage up to a 2× default / 3× hard ceiling on HL perps shall be available only in P4, under the Adventurous profile, after the derivatives Learn module and CMP-010 review, isolated margin only. | Must | P4 | Stricter than "P3+" on purpose |
| HL-009 | HIP-3 equity/commodity/index perps may be shown read-only as labelled "24/7 macro context" cards, hidden in Guided mode. | Could | P2 | Never tradable before a separate review |

### 9.2 Connectivity and execution

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| HL-010 | Ubik shall implement `HyperliquidFeed` (P1) and `HyperliquidVenue` (P3) as its own thin client behind the existing ports; ccxt shall not be used for HL trading. | Must | P1 / P3 | §4.1; replaces NFR-016's 1-day estimate for this venue |
| HL-011 | If ccxt is ever used for HL private calls, builder-fee approval, builder attachment and `setReferrer` shall be disabled and verified absent in tests. | Must | P3 | Hidden side effects |
| HL-012 | The signer shall implement only the L1 action scheme, be pinned with test vectors cross-checked against the official SDK, and format numbers from Decimal with no trailing zeros and lowercase addresses. | Must | P3 | EX-009; signing pitfalls |
| HL-013 | Ubik shall run one nonce manager per agent (monotonic `max(now_ms, last+1)`, persisted) and only one process per agent. | Must | P3 | Nonce collisions |
| HL-014 | Ubik shall derive each order's cloid from its client order ID (§4.4), persist it before sending, never reuse it, and verify duplicate-cloid behaviour on testnet. | Must | P1 (paper) / P3 | EX-002/003 |
| HL-015 | Every order action shall carry `expiresAfter` (default now + 10 s); UNKNOWN orders shall be resolved by `orderStatus` by cloid, optional same-nonce `noop`, and `unknownOid` after expiry + margin → REJECTED. | Must | P3 | Deterministic EX-005 |
| HL-016 | The adapter shall handle whole-batch pre-validation errors and per-order statuses, and map every HL order status and rejection to Ubik's state machine with a reason code. | Must | P3 | §4.4 |
| HL-017 | The WebSocket client shall ping every ≤ 30 s, reconnect with backoff, resubscribe, de-duplicate snapshot acks, subscribe with the account (never agent) address, and respect the per-IP limits (≤ 10 connections, ≤ 1,000 subscriptions, ≤ 10 users). | Must | P1 | AR-009 for HL |
| HL-018 | Reconciliation (EX-007) on HL shall use `clearinghouseState`, `spotClearinghouseState`, `frontendOpenOrders`, `userFillsByTime` and `orderStatus`, and shall also check leverage, margin mode, account mode and the agent list. | Must | P3 | §4.5 |
| HL-019 | Order types shall map per §4.6: protected market = IOC limit capped at 0.5%; post-only = ALO; stops = reduce-only trigger orders on mark (stop-limit with explicit limit preferred); FOK/GTD/trailing emulated bot-side. | Must | P1 (sim) / P3 | EX-014/015 |
| HL-020 | Ubik shall apply HL precision rules (5 significant figures, `6/8 − szDecimals` decimals, integer prices valid, size to `szDecimals`, $10 / 10-quote minimum) identically in all modes, from effective-dated registry snapshots. | Must | P1 | EX-010 |
| HL-021 | Ubik shall budget both the IP weight (≤ 840/min) and the address action budget (≤ 70%, with the last 30% reserved for cancels and exits), poll `userRateLimit` every 5 min, show it as a gauge, and alert below 30% headroom. | Must | P1 (sim) / P3 | D-37 extended |
| HL-022 | Ubik shall refuse to start an execution algorithm whose projected daily actions exceed the earned address budget, and may buy capacity with `reserveRequestWeight` only within an owner-set monthly cap (default $5). | Should | P3 | Small accounts |
| HL-023 | The dead-man's switch on HL shall be the local watchdog; native `scheduleCancel` shall be added only once available to the account and once testnet shows it doesn't remove trigger orders. | Must | P3 | RK-025; volume gate unverified |

### 9.3 Keys, wallets and account security

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| HL-024 | Ubik shall never accept, store or use a Hyperliquid master key, and shall contain no code that builds user-signed actions (withdraw, transfer, approveAgent, approveBuilderFee, staking link). | Must | P3 | Withdrawal refusal (D-24) by construction |
| HL-025 | The gateway shall accept an agent key only with its account (master or sub-account) address, verify the agent is listed in `extraAgents` with expiry > 30 days and differs from the account address, and refuse otherwise. Checks repeat at start and daily. | Must | P3 | SEC-008 equivalent |
| HL-026 | Because HL has no IP allow-list, the venue shall be marked "IP restriction unavailable" (SEC-011 exemption) and the compensating controls of HL-027–HL-029 are mandatory. | Must | P3 | §5.3 |
| HL-027 | The gateway shall enforce a symbol allow-list by asset id, a price band (≤ 1% from mark for limits), per-order/per-day/gross notional caps, a 1× leverage cap and a maximum open-order count, independently of the engine. | Must | P3 | Defense in depth |
| HL-028 | An independent watcher on the public address shall raise a Critical alert and trigger kill switch L2 on any order, fill, leverage or mode change, transfer or agent not originated by Ubik. | Must | P3 | EX-013; trade-to-drain |
| HL-029 | Agents shall be named, expire ≤ 90 days, be rotated from T−14 days with a new address every time, and be one per account and per process. The "Key compromised" flow (SEC-015) shall guide same-name re-approval from the hardware wallet and a sweep. | Must | P3 | Replay risk on reuse |
| HL-030 | Onboarding shall require a dedicated trading account whose master is a hardware wallet, separate from the owner's savings address, funded only with the pilot budget; sub-accounts per bot group once the $100k volume unlock is reached. | Must | P3 | §5.1 |
| HL-031 | Ubik shall read the account mode at start and refuse to arm if it differs from the owner-chosen mode (default Standard). | Must | P3 | §7 |

### 9.4 Data and backtesting

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| HL-032 | From first start Ubik shall record the §6.3 HL channels and REST snapshots for the listed instruments. | Must | P1 | 3.5 days of 1m history otherwise |
| HL-033 | At first launch Ubik shall backfill full HL funding history and the maximum `candleSnapshot` depth per interval, applying the §6.3 data-quality rules. | Must | P1 | Free |
| HL-034 | Ubik shall backfill proxy history (Binance USDⓈ-M perps incl. mark and funding; Coinbase USD spot), basis-aligned, for the D-38 window. | Must | P1 | Quick-start dataset (DA-023) |
| HL-035 | Every bar shall carry `source_venue` and fidelity grade A–D; reports shall show the grade mix, a native-window-only result and a proxy-wick warning. | Must | P1 | Honesty (D-12) |
| HL-036 | `SimVenue` shall implement the §6.4 HL profile (fees, precision, ALO/IOC semantics, mark-price triggers, hourly funding, liquidation, post-only periods, OI-cap and oracle-band rejections, address budget). | Must | P1 | Sim-live parity |
| HL-037 | Stops and liquidations shall be simulated on a mark-price path where one exists, with trade-path results shown alongside. | Must | P1 | Stops trigger on mark |
| HL-038 | The stress library (04 §6.4) shall include an HL 10 Oct 2025 scenario (deep wicks, ADL), an API outage with the chain running, a funding spike to 2024 levels, and a delisting settlement. | Should | P2 | Lessons §8.2 |
| HL-039 | Ubik may import requester-pays archives (official `asset_ctxs`, Hydromancer 1s candles) with a narrowly scoped AWS credential, after their licences are entered in the data-source registry (CMP-007). | Could | P2 | Better native history |
| HL-040 | The venue registry shall hold effective-dated HL fields: network, asset ids and aliases, `szDecimals`, decimals limits, max leverage, margin tables, margin-mode restrictions, delisting, minimum notional, fee tiers and multipliers, both rate budgets, oracle-band parameter, terms version. | Must | P1 | AR-002/CMP-003 |

### 9.5 Path to live

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| HL-041 | S2 on HL shall consist of the testnet conformance suite (§7) for ≥ 7 days, followed by minimum-size mainnet round trips after arming; testnet results shall never count as strategy evidence. | Must | P3 | PT-016 |
| HL-042 | S3 shadow on HL shall read the dedicated account by public address with no key stored, booking simulated fills to the paper schema. | Must | P3 | D-29 |
| HL-043 | S4 pilots on HL shall run at 1× on allow-listed instruments with the 05 §9.1 caps, in the dedicated account or a sub-account, with zero unexplained fills as an exit criterion. | Must | P3 | |
| HL-044 | Onboarding shall include a funding guide (CCTP V2 preferred, minimums, deposits are manual) and Ubik shall book deposits/withdrawals read-only from the ledger stream. | Should | P3 | Ubik never moves money |

### 9.6 Risk, legal, privacy and tax

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| HL-045 | Ubik shall refuse to arm live trading on HL when the jurisdiction profile shows residence, location or incorporation in the US or Ontario, or residence in or citizenship of a sanctioned territory, or when the server's egress country is restricted; it shall explain why and offer no workaround. | Must | P3 (profile check shown from P1) | Terms s.1.6, s.3.1.5; CMP-006 |
| HL-046 | The HL Terms version and restricted list shall be recorded in the venue registry and re-reviewed every 3 months and at each phase gate. | Must | P1 | Changes take effect on posting |
| HL-047 | Ubik shall cap total value on HL, including HYPE, under the per-venue cap (RK-032) and remind the owner to sweep profits above it. | Must | P3 | Venue concentration |
| HL-048 | ADL, liquidation, delisting settlement and non-user cancels shall be booked as venue-initiated events, alerted, and trigger re-planning of the affected bot. | Must | P1 (sim) / P3 | §8.2 |
| HL-049 | Before arming, Ubik shall show a plain-language disclosure that the account is public, the agent key can trade but not withdraw, and there is no investor protection or compensation scheme. | Must | P3 | §8.1, §8.3 |
| HL-050 | Ubik shall persist every fill, funding payment, fee, liquidation, ADL, deposit, withdrawal and transfer with block time and hash, and export funding as its own row type. | Must | P1 (paper) / P3 | Venue keeps only 10k fills |
| HL-051 | USDC shall be the only collateral and quote asset on HL through P3, with peg monitoring (RK-029). | Must | P1 | §8.4 |
| HL-052 | The changelog watch (AR-006) shall cover HL's Telegram API channel and Discord `#api-traders`, and alert on announced breaking changes (including "v1 API"). | Should | P1 | No changelog web page |

### 9.7 Secondary venue

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| HL-053 | Binance shall remain the secondary venue: public data for proxy history and cross-checks from P1 (via ccxt), and a spot backup venue through the existing `CcxtVenue` path in P3. | Should | P1 / P3 | §10 |
| HL-054 | Ubik shall show HL vs Binance basis and funding (`predictedFundings`) and alert when HL's mid diverges from the CEX reference by more than a threshold (default 1%) while data is fresh. | Should | P2 | Detects venue-specific dislocations |

## 10. Fallback and secondary venue

**Recommendation: keep Binance as the secondary venue**, in two roles, and don't add a third in P1.

1. **Data (P1, Must for history).** Binance supplies the long proxy history (USDⓈ-M perps with mark and funding back to 2019–2020) and is the largest input to Hyperliquid's oracle (weight 3) and mark price. Coinbase public USD candles are used only as the spot proxy; no account needed.
2. **Backup venue (P3, Should).** If Hyperliquid becomes unusable (Terms change, a sustained outage, a bridge freeze, a regulatory change in the owner's country), the owner can move a strategy to Binance spot with the CEX adapter already designed in docs 05/06. The strategy must re-enter paper on that venue, because graduation evidence is per bot (one strategy version on one account, D-40) and doesn't transfer across venues. Binance's own availability and rules for Colombian residents need confirming (Q-02).

Why not more: one developer, and every live venue costs adapter work and S2–S4 time. A second *live* venue at the same time also raises the capital spread across two counterparties, which is good for safety, so revisit at S5.

## 11. Proposed decisions for doc 10

- **D-45** Hyperliquid is the primary venue; Binance secondary (data + backup spot).
- **D-46** D-06 becomes "no leverage, no shorting until P4"; HL spot or 1× isolated long perp; default vehicle per §3.3.
- **D-47** Own thin native HL client; ccxt only for CEX venues and public cross-checks (amends D-02 for this venue).
- **D-48** Key model: master on a hardware wallet, never on the server; named agents ≤ 90 days, new address on rotation, one per account and process; compensating gateway controls instead of an IP allow-list (amends D-24).
- **D-49** UNKNOWN-order protocol on HL: cloid + `expiresAfter` + `orderStatus`/`noop`.
- **D-50** Rate limits: D-37 extended to the address budget, with a reserved exit share and a capped `reserveRequestWeight` spend.
- **D-51** HL path to live: S2 = testnet conformance + minimum-size mainnet orders; S3 = keyless public shadow; local watchdog as dead-man's switch.
- **D-52** HL data: record from P1 day one; proxy history with fidelity grades; mark-path stop simulation.
- **D-53** Jurisdiction gate for HL, Terms re-review every 3 months, no circumvention.
- **D-54** Excluded through P3: HIP-1 long-tail, HIP-3, vaults/staking, non-USDC quotes; HIP-4 never.
- **D-55** Account mode Standard, pinned and checked.

## Open questions for the owner

| ID | Question | Default if unanswered |
|---|---|---|
| Q-HL-01 | Please confirm your **country of residence and tax residence**, and that you are not a resident of the US or Ontario. Where will the server run? | Colombia resident, server in Colombia or another non-restricted country; HL live trading stays disarmed until confirmed |
| Q-HL-02 | Are you comfortable holding BTC/ETH exposure as a **1× isolated long perp** (funding cost, ADL, derivative label), or do you want **spot only** on Hyperliquid (bridged UBTC/UETH, thinner books)? | 1× perp default for BTC/ETH, spot for HYPE, switchable per bot |
| Q-HL-03 | Do you have (or will you buy) a **hardware wallet** for the Hyperliquid master account? | Required before S2 mainnet orders |
| Q-HL-04 | Do you already have a Hyperliquid account and volume? (Affects testnet faucet, fee tier, sub-account and `scheduleCancel` availability.) | New dedicated account, tier 0, no sub-accounts |
| Q-HL-05 | May proxy (Binance/Coinbase) history count toward the S0 ≥ 100 round-trip gate, and with what minimum native share? | Yes, if the native window has ≥ 20 round trips and both results are shown |
| Q-HL-06 | Will you create an AWS account for requester-pays archives (a few dollars), or pay for Tardis/Dwellir later? | No AWS in P1; free sources only |
| Q-HL-07 | How much may sit on Hyperliquid in total at S4/S5 (venue cap)? | Sum of pilot allocations + 20% |
| Q-HL-08 | Are you OK with your trading being **publicly visible** on-chain? | Yes, with the disclosure; small size |
| Q-HL-09 | Should HIP-3 equity/commodity markets appear as read-only macro context in P2, or be hidden entirely? | Read-only, Pro mode only |
| Q-HL-10 | Do you have a Colombian accountant to confirm treatment of perps and funding before P3? | Generic export; consult recommended |

## Risks

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| HR1 | **Jurisdiction or Terms change** (owner moves, Colombia rules change, HL restricts more countries) | Primary venue unusable | Jurisdiction gate (HL-045); quarterly Terms review; Binance backup path (HL-053) |
| HR2 | **Agent key theft** with no IP allow-list | Trade-to-drain losses up to the account balance | Dedicated small account; gateway controls; public-address watcher with L2 kill; short expiry |
| HR3 | **Master key phishing / malware** (most real HL losses) | Total loss of the account | Hardware wallet; bookmark-only URLs; never sign from links; savings address separate |
| HR4 | **Short native history; proxy data flatters** (wicks differ by % in crashes) | False confidence (R1) | Fidelity grades, native-window results, mark-path stops, record from day one |
| HR5 | **Address rate budget exhausted** | Exits throttled to 1 per 10 s | Dual budgets, reserved exit share, churn refusal, capped capacity purchase |
| HR6 | **Venue events**: ADL, governance force-settlement, post-only upgrade windows, API outage while the chain runs, bridge lock | Unordered fills; no cancels; delayed withdrawals | Majors only; 1× positions survive outages; venue-initiated event handling; venue cap |
| HR7 | **Validator concentration** (Foundation ~44% of stake; top 8 > ⅔) | Halt or contested state | Venue cap; sweep profits; monitor |
| HR8 | **Bridge/stablecoin failure** (Unit guardians for UBTC; Circle/CCTP; legacy bridge) | Spot tokens or collateral lose value | Prefer perps for BTC/ETH; USDC only; peg monitoring; bridged-asset cap |
| HR9 | **API v0 → v1 breaking change**, undocumented behaviours (cloid duplicates, modify semantics) | Adapter breaks | Own thin client; contract suite on testnet; changelog watch on Telegram/Discord |
| HR10 | **Funding costs** on long perps in bull markets (24% a year in 2024) | Erodes returns | Vehicle comparison; funding in every backtest; spot option |
| HR11 | **Public visibility** of positions and stops | Copying, stop-hunting, linkability to the owner | Disclosure; small size; bot-managed precise stops with wide native catastrophe stops |
| HR12 | **Scope growth** (native adapter, recorder, SimVenue profile) | P1 slips | Native adapter is public-data only in P1; trading half waits for P3 |
