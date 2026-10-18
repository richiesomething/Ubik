# Ubik Glossary

*Companion to [07-ux-and-beginner-experience.md](07-ux-and-beginner-experience.md) · Draft v0.1 · 2026-10-18*

## Summary

- These are plain-language definitions of the terms Ubik uses. They will seed the in-app tooltips, "What does this mean?" panels and Learn hub (requirements ED-001, ED-025).
- **Tooltip rule:** each entry's first sentence is ≤ 25 words and makes sense on its own as a tooltip. Later sentences add nuance, an analogy or a warning.
- Each term is tagged with a category: *Market, Order, Indicator, Strategy, Backtest, Stats, Risk, Crypto, Exchange, Behavior, Ubik*. Tags such as "(P2)" or "(P4)" mark features that arrive in a later phase.
- Definitions must stay accurate and neutral. When exchange-specific facts change (fee levels, funding intervals), entries are updated and re-versioned (ED-026).

---

## #

**2FA / MFA (two-factor / multi-factor authentication)** · *Exchange* — A login check that needs a second proof besides your password, such as a code from an authenticator app or a *Passkey*. Ubik's own login supports it and requires it before Ubik can be reached from other devices. *Analogy:* a door that needs both a key and a PIN.

## A

**ADX (Average Directional Index) / +DI, −DI** · *Indicator* — A 0–100 gauge of how strong a trend is, whatever its direction. +DI and −DI show whether up-moves or down-moves dominate. Readings above about 20–25 suggest a trending market.

**All-time high (ATH)** · *Market* — The highest price an asset has ever traded at. It is context, not a target or a promise.

**Alpha** · *Stats* — Return beyond what simple market exposure (e.g., just holding BTC) explains. It's often credited to skill, but it can be luck. *Analogy:* a cyclist's speed after subtracting the help from a tailwind.

**Altcoin** · *Crypto* — Any cryptocurrency other than bitcoin. Many are small, illiquid and far more volatile than BTC.

**API (application programming interface)** · *Exchange* — A way for software to talk to other software. Ubik uses exchange APIs to get prices, and from P3 to place orders.

**API key** · *Exchange* — A credential that lets Ubik access your exchange account with only the permissions you choose. Enable "read" and "trade" only, and never "withdraw". Paper trading needs no key, and Ubik accepts none before live trading (P3). *Analogy:* a valet key that can drive the car but can't open the trunk.

**Arbitrage** · *Strategy* — Buying an asset where it's cheaper and simultaneously selling where it's dearer to capture the price gap. Real opportunities are small, fleeting and often eaten by fees and transfer delays. (P4)

**Arming (live)** · *Ubik* — The deliberate step that switches a strategy from simulated to real-money trading, after warnings, a comprehension quiz and a cooling-off period. Ubik also checks that your API key is trade-only and IP-locked, and you re-confirm with a passkey or 2FA. (P3)

**Ask (offer)** · *Market* — The lowest price a seller is currently willing to accept. *Analogy:* the price tag on a used car.

**Asset** · *Market* — Anything you can own and trade, such as BTC, ETH or USDT.

**ATR (Average True Range)** · *Indicator* — The average size of each candle's full price range, including gaps, over recent periods (often 14). Stated in price units, it measures volatility and is often used to set stop distances.

**Average entry price** · *Order* — The average price you paid for a position across all its buys, weighted by quantity.

**Average holding time** · *Backtest* — How long a strategy's trades typically stay open. The shorter the holding time, the more fees, slippage and delays eat into results.

## B

**Backtest** · *Backtest* — Running a strategy's rules on historical data to see how it would have performed. It simulates the past and is not a forecast. *Analogy:* a flight simulator replaying yesterday's weather; useful practice, but tomorrow's storm will differ.

**Base currency / quote currency** · *Market* — In BTC/USDT, BTC is the base (what you buy or sell) and USDT is the quote (what the price is measured and paid in).

**Basis** · *Market* — The difference between a futures price and the spot price of the same asset. (P2 data, P4 trading)

**Basis point (bp)** · *Stats* — One hundredth of a percent (0.01%). A 10 bp fee is 0.1%.

**Bear market** · *Market* — A prolonged period of falling prices, commonly described as a drop of 20% or more from a recent peak. Crypto bear markets have often seen falls of 70% or more.

**Benchmark** · *Backtest* — A simple reference to compare results against, such as just holding BTC or holding cash. If a strategy can't beat its benchmark after costs, its complexity isn't paying off.

**Beta** · *Stats* — How strongly an asset tends to move relative to a benchmark. A beta of 1.5 to BTC means it has typically moved about 1.5% when BTC moved 1%.

**Bid** · *Market* — The highest price a buyer is currently willing to pay.

**Bitcoin dominance** · *Crypto* — Bitcoin's share of the total market value of all cryptocurrencies. It's used as a rough gauge of risk appetite across crypto. (P2)

**Blockchain** · *Crypto* — A shared, append-only record of transactions that many computers maintain together. *Analogy:* a public notebook that everyone has a copy of, where pages can be added but not erased.

**Bollinger Bands** · *Indicator* — A moving average (often 20 periods) with bands usually 2 standard deviations above and below it. The bands widen when prices are volatile and narrow when calm. %B shows where price sits within the bands (0 at the lower band, 1 at the upper), and bandwidth shows how far apart they are.

**Bot** · *Ubik* — One strategy version running on one account, paper or live, with its own *Sleeve* of money. Several bots can share an account, and each bot moves along the path to live on its own (see *Graduation*).

**Breakout** · *Strategy* — A move of price beyond a recent range or level, such as above resistance. Breakout strategies buy these moves, and many breakouts quickly reverse ("false breakouts").

**Bull market** · *Market* — A prolonged period of rising prices.

**Buy and hold (HODL)** · *Strategy* — Buying an asset and keeping it through ups and downs. Ubik shows it as the default benchmark, because many active strategies fail to beat it after costs.

## C

**CAGR (compound annual growth rate)** · *Stats* — The steady yearly growth rate that would turn the starting value into the ending value. It smooths the path and hides how bumpy the ride was.

**Calmar ratio** · *Stats* — Annualized return divided by maximum drawdown. Higher means more return per unit of worst-case pain.

**Candlestick (candle)** · *Market* — A chart bar showing one period's open, high, low and close prices. The thick body spans open to close, and thin wicks show the high and low. Ubik marks up and down candles with fill style as well as color.

**Capacity for loss** · *Risk* — How much money you could lose without it materially affecting your life. It differs from risk tolerance, which is how much loss you feel comfortable with.

**Careful / Balanced / Adventurous (risk profiles)** · *Risk* — Ubik's three risk profiles, from strictest to loosest. Every new install starts on Careful, in every mode. Careful allows 0.25–1% risk per trade (default 1%), Balanced up to 2%, and Adventurous up to the 5% hard ceiling. Stops are mandatory except under Adventurous, which allows strategies without stops (and a few paper-stage gate overrides) only with an explicit acknowledgement. Hard ceilings are the same in all three. Moving to a looser profile needs a one-time acknowledgement; moving to a tighter one is instant. See *Risk profile*.

**Catastrophe stop** · *Order* — A wider backup stop-loss resting on the exchange itself, so a position stays protected if Ubik or its internet connection goes down. By default it sits about 1.5× further away than Ubik's own stop. Backtests and paper trading simulate it. (P3)

**Centralized exchange (CEX)** · *Exchange* — A company-run trading platform (e.g., Binance, Coinbase, Kraken) that holds customers' funds and matches their orders. It is convenient and liquid, but you trust the company with your money.

**Circuit breaker** · *Risk* — An automatic rule that halts trading when something abnormal happens, such as a large loss or bad data. Ubik's loss limits and drawdown breakers work this way. See *Drawdown breaker*, *Kill switch*.

**Closed bar** · *Market* — A candle whose time period has ended, so its values are final. Ubik's strategies, alerts and scans act only on closed bars by default; the still-forming candle is drawn differently on the chart.

**Coin vs token** · *Crypto* — A coin is native to its own blockchain (BTC, ETH, SOL). A token is issued on top of another blockchain (e.g., USDT on Ethereum).

**Compounding** · *Stats* — Earning returns on previous returns, so growth builds on itself. It works in reverse too: losses shrink the base that future gains grow from.

**Concentration cluster** · *Risk* — A group of coins that tend to move together, such as BTC, ETH and SOL, capped as if they were one bet. By default the "crypto majors" together may hold at most 75% of your equity.

**Cooldown** · *Risk* — A short forced break from new entries after losses. By default a strategy waits 3 bars after a stop-out, and pauses for 24 hours after three losing trades in a row. Not the same as a *Cooling-off period*.

**Cooling-off period** · *Ubik* — A deliberate waiting time before a risk-increasing change to live trading takes effect, giving you time to reconsider. First arming of live trading always waits 24 hours. Loosening a limit, moving to a looser risk profile or raising an allocation waits 24 hours by default; you can set 1–24 hours. Shortening the delay is itself risk-increasing, so it takes effect only after the current delay. (P3)

**Correlation** · *Stats* — A number from −1 to +1 describing how closely two assets move together. Most crypto assets are highly correlated with BTC, so owning several coins diversifies less than it seems.

**Cost basis** · *Risk* — The original value of an asset for tax purposes, usually what you paid including fees. The rules for calculating it vary by country.

**Custody** · *Crypto* — Who holds and controls the assets. On a centralized exchange, the exchange has custody of your funds.

## D

**Data snooping** · *Backtest* — Testing so many ideas on the same historical data that some look good purely by chance. See *Overfitting*.

**Day trading** · *Strategy* — Opening and closing trades within the same day. Studies of individual day traders find that the large majority lose money.

**DCA (dollar-cost averaging)** · *Strategy* — Investing a fixed amount at regular intervals regardless of price. It reduces the risk of bad timing but doesn't prevent losses if the price keeps falling.

**Dead-man's switch** · *Ubik* — A live-trading safety net that cancels resting entry orders if Ubik's engine stops sending its heartbeat for more than 60 seconds. It uses the exchange's built-in cancel-after timer where there is one, and a local *Watchdog* otherwise. Positions and protective stops stay in place. It alerts no one; that is the *External heartbeat monitor*'s job. (P3)

**Decentralized exchange (DEX)** · *Exchange* — A platform where trades happen through smart contracts on a blockchain, without a company holding your funds. (P4)

**DeFi (decentralized finance)** · *Crypto* — Financial services such as lending and trading run by blockchain programs (smart contracts) instead of companies. (P4)

**Deflated Sharpe ratio** · *Stats* — A Sharpe ratio adjusted downward for how many strategy variations were tried and for unusual return distributions. It is a guard against overfitting; Ubik warns when it falls below 0.95. (P2)

**Depeg** · *Crypto* — When a stablecoin trades noticeably away from its target value, as USDC did in March 2023 when it fell to about $0.87. Ubik prices stablecoins at their market price, warns at a 0.5% deviation, and pauses pairs quoted in that coin if it stays more than 1% off for 5 minutes.

**Depth (market depth)** · *Market* — How much is waiting to be bought and sold at each price level in the order book. Deep markets can absorb large orders without big price moves.

**Derivative** · *Market* — A contract whose value depends on another asset, such as futures, perpetuals or options. Derivatives are often traded with leverage. (P4)

**Disposition effect** · *Behavior* — The tendency to sell winning positions too early and hold losing ones too long, hoping to "get back to even".

**Diversification** · *Risk* — Spreading money across assets that don't move together, to reduce overall risk. It only helps when correlations are genuinely low.

**Donchian channel** · *Indicator* — Bands marking the highest high and lowest low of the last N bars. Breakout strategies buy when price closes above the upper band.

**Drawdown** · *Risk* — The percentage fall in account value from its previous peak to a later low. *Analogy:* how far you've slipped down the mountain since your highest point.

**Drawdown breaker** · *Risk* — An automatic limit that stops trading when losses from a peak grow too large. A strategy is halted at the smaller of 20% and 1.5 × its *Reference drawdown*. An account is paused at 15% and halted at 20%. A halt lasts until you re-arm it with a short note. You can tighten the 20% cap, but never loosen it.

**Drawdown duration (time underwater)** · *Backtest* — How long an account or strategy stays below its previous peak before recovering. More than a year underwater is psychologically hard to sit through.

## E

**EMA (exponential moving average)** · *Indicator* — A moving average that gives more weight to recent prices, so it reacts faster than a simple moving average.

**Equity curve** · *Backtest* — A chart of account value over time. A smooth curve is pleasant to look at, but it can hide risk that simply hasn't shown up yet.

**Equity (marked-to-market)** · *Risk* — Your account's value with open positions priced at current market prices. Ubik's loss limits use it, so a large open loss counts before you sell.

**Evidence label** · *Ubik* — A badge on each indicator, signal or analysis card saying how much published research supports it: Descriptive, Supported, Mixed, Weak or Untested. "Descriptive" means it measures the present without forecasting, and your own indicators start as Untested. Not the same as the *Trust check*, which rates one backtest.

**Exchange** · *Exchange* — A marketplace where assets are bought and sold. See *CEX*, *DEX*.

**Exchange demo (demo mode)** · *Exchange* — An exchange-run practice account with play money that mirrors the live market and API, such as Binance Spot Demo Mode. Ubik uses it only in stage S2, to test its live connection, never to judge a strategy. Exchanges without a realistic demo are tested another way. (P3)

**Execution** · *Order* — Turning a trading decision into filled orders, and how well that is done in terms of price, speed and cost.

**Execution gateway** · *Ubik* — The one separate part of Ubik that holds your exchange keys and sends real orders. It runs its own last-line checks (allowed coins, maximum order value, price band, order rate and pilot caps), whatever a strategy or setting says. (P3)

**Expectancy** · *Stats* — The average profit or loss per trade: (win rate × average win) − (loss rate × average loss). It needs to be positive after costs.

**Expectation band** · *Backtest* — The range of results a strategy's backtest says paper or live trading should fall within. To leave paper trading, per-round-trip returns must be no worse than the backtest's bottom 10%, and drawdown must stay within the *Reference drawdown*.

**Expected Shortfall (ES)** · *Risk* — The average loss on the worst days, those beyond the *Value at Risk*. It's a more honest risk number for crypto's *Fat tails*. (P2)

**Exposure** · *Risk* — How much of your money is currently in the market, in total or per asset.

**Exposure time (time in market)** · *Backtest* — The share of time a strategy holds a position. While out of the market, a strategy avoids crashes but can also miss rallies.

**External heartbeat monitor** · *Ubik* — An outside service that Ubik's engine checks in with every 30 seconds; after 90 seconds of silence, it alerts you. It works even if your machine is down. In paper the alert is a Warning (held during quiet hours); in live trading it is Critical. It only alerts and never touches orders (compare *Dead-man's switch*).

## F

**Fat-finger check** · *Risk* — A guard that blocks an order priced too far from the market, such as one with a typing error. By default a limit price may cross the mid-price by at most 1% for BTC and ETH, and 2% for other coins.

**Fat tails** · *Stats* — Extreme price moves happen far more often than a normal bell curve predicts. Statistics that assume a bell curve understate crash risk in crypto.

**Fear & Greed Index** · *Market* — A popular 0–100 sentiment score that combines factors like volatility, momentum and social media. It is a rough mood gauge, not a reliable timing signal. (P2)

**Fee drag** · *Backtest* — The share of a strategy's capital eaten by fees and slippage each year. At 0.6% per side, 100 full-size round trips a year cost about 120% of the capital, so frequent trading needs a large edge.

**Fees (trading fees)** · *Exchange* — What the exchange charges per trade, usually a percentage of the trade's value. Small fees add up quickly with frequent trading. See *Maker / taker*.

**Fiat** · *Market* — Government-issued money such as US dollars or euros.

**Fill / partial fill** · *Order* — An order is filled when it executes. It is partially filled when only part of the quantity has executed so far.

**FOMO (fear of missing out)** · *Behavior* — The urge to buy because prices are rising and others seem to be profiting. It is a common reason people buy near the top.

**FUD (fear, uncertainty and doubt)** · *Behavior* — Negative news or rumors, true or not, that push people to sell.

**Funding rate** · *Market* — A periodic payment between holders of long and short positions on perpetual futures that keeps the perpetual's price near spot. When the rate is positive, longs pay shorts. Payments are often made every 8 hours, but this varies by exchange. (P2 data, P4 trading)

**Futures** · *Market* — A contract to buy or sell an asset at a set price on a future date. (P4)

## G

**Gas fee** · *Crypto* — A fee paid to a blockchain network to process a transaction. It is not the same as an exchange's trading fee.

**Golden cross / death cross** · *Indicator* — The moment a shorter moving average (often 50-day) crosses above (golden) or below (death) a longer one (often 200-day). It is widely watched, lagging and frequently misleading.

**Graduation** · *Ubik* — Moving a *Bot* to the next stage on Ubik's path to live once its scorecard criteria pass. The stages are Draft → S0 Backtest → S1 Paper → S2 Exchange demo → S3 Shadow → S4 Pilot → S5 Scaled; a withdrawn version ends as *Retired*. Promotion needs your approval; demotion is automatic. (Draft, S0 and S1 tracked from P1; S2–S5 from P3)

**Grid trading** · *Strategy* — Placing buy and sell orders at fixed intervals around the price to profit from back-and-forth moves. It works in sideways markets but can pile up losing positions in a strong trend.

**Guardrail** · *Ubik* — A default safety limit or check in Ubik, such as the *Drawdown breaker*, that protects you from large mistakes.

**Guided mode / Pro mode** · *Ubik* — Two depths of the same app: Guided shows the essentials in plain words, and Pro shows every control and metric. The mode never changes your risk limits or safety gates; those come from your *Risk profile*. You can switch at any time.

## H

**Halving** · *Crypto* — A scheduled event, about every four years, that cuts the amount of new bitcoin created per block in half.

**Hedge** · *Risk* — A position taken to offset the risk of another position. *Analogy:* buying insurance.

**Heikin-Ashi** · *Market* — Smoothed candles built from averaged prices, which make trends look cleaner. Real trades never happen at Heikin-Ashi prices, so Ubik's chart notes that fills can't occur there. (P2)

**High-water mark** · *Risk* — The highest value an account or strategy has reached. Drawdown is measured from it.

## I

**Implementation shortfall** · *Order* — The total cost of a trade compared with the price when the strategy decided to trade. It adds up the delay, the spread, your own market impact and fees. See *TCA*.

**Implied volatility** · *Market* — The future volatility that options prices suggest traders expect. (P2 data)

**In-sample / out-of-sample** · *Backtest* — In-sample data is used to build and tune a strategy. Out-of-sample data is held back and used only to check whether the strategy still works. *Analogy:* practice exam versus the real exam.

**Incident card** · *Ubik* — Ubik's standard five-part explanation when something goes wrong, such as a blocked order, an outage or a halted strategy. It says what happened, what Ubik already did, where your positions stand, what you can do, and links to "Explain this".

**Indicator** · *Indicator* — A calculation from price and volume data, such as RSI or a moving average, that summarizes what has happened. Indicators describe the past; they don't predict the future.

**IP allowlist** · *Exchange* — An exchange setting that makes an API key work only from internet addresses you specify. It limits damage if the key leaks.

## K

**Kelly criterion** · *Risk* — A formula for the bet size that maximizes long-run growth if you know your true edge. Real edges are uncertain, so full Kelly is dangerously aggressive.

**Kill criteria** · *Strategy* — The conditions, declared before paper trading starts, under which a strategy is stopped. By default this is the strategy *Drawdown breaker*; you can set stricter ones but not looser.

**Kill switch** · *Ubik* — Ubik's emergency stop, with three levels: Pause blocks new entries, Cancel also cancels all open orders, and Flatten also closes all positions. *Pause all* (L1) is always one click or tap away and needs no confirmation. Cancel (L2) takes at most two clicks or taps and finishes within 10 seconds; Flatten (L3) also asks you to confirm. It works in paper from P1, and from the server command line if the web app is down.

**KYC (know your customer)** · *Exchange* — The identity checks exchanges are legally required to carry out before letting you trade or withdraw.

## L

**Latency** · *Exchange* — The delay between something happening and the system reacting, such as data arriving or an order reaching the exchange.

**Leverage** · *Risk* — Trading with borrowed money to control a larger position than your own funds. At 10× leverage, a roughly 10% move against you can wipe out your stake. Ubik doesn't offer it before P4.

**Limit order** · *Order* — An order to buy or sell only at a specified price or better. It controls price, but it may never fill if the market doesn't reach that price.

**Liquidation** · *Risk* — The exchange forcibly closing a leveraged position because losses have used up the required margin. (P4)

**Liquidity** · *Market* — How easily you can buy or sell without moving the price much. BTC/USDT on major exchanges is very liquid; many small coins are not.

**Live trading** · *Ubik* — Trading with real money on a real exchange. Ubik always labels it "LIVE · real money". (P3)

**Long** · *Market* — Owning an asset, or betting that its price will rise.

**Look-ahead bias** · *Backtest* — A backtest error where the strategy uses information that wasn't available yet at that moment, such as a candle's close before the candle finished. It makes results look unrealistically good.

**Lot size (step size)** · *Exchange* — The smallest quantity increment an exchange accepts for an order on a given pair.

## M

**MACD (moving average convergence divergence)** · *Indicator* — The difference between a 12-period and a 26-period EMA, shown with a 9-period EMA "signal line". It is used to gauge trend direction and momentum.

**Maker / taker** · *Exchange* — A maker's order waits in the order book and adds liquidity. A taker's order fills immediately against existing orders and removes liquidity. Taker fees are usually higher.

**Margin** · *Risk* — Money set aside as collateral for a leveraged position. (P4)

**Mark price** · *Exchange* — A fair-value price that derivatives exchanges use to calculate unrealized profit and loss and liquidations, so that one odd trade can't trigger them. (P4)

**Market cap (market capitalization)** · *Crypto* — Price × circulating supply. It is a rough measure of an asset's total size.

**Market Health Summary** · *Ubik* — Ubik's one-screen overview of a coin or the whole market: trend, volatility, liquidity, momentum, upcoming events and data freshness. Cards are colored by condition, never by buy or sell, and there is no overall buy/sell score.

**Market maker** · *Strategy* — A trader or firm that continuously offers to both buy and sell, earning the spread and providing liquidity. (P4)

**Market order** · *Order* — An order to buy or sell immediately at the best available prices. It fills fast, but the final price isn't guaranteed. See *Slippage*.

**Matching engine** · *Backtest* — The code that decides whether, when and at what price a simulated order fills. Ubik uses the same one for backtests and paper trading, so their results are comparable.

**Max drawdown** · *Risk* — The largest peak-to-low fall over a period. It measures the worst stretch you would have lived through. Recovering from −50% takes +100%.

**Mean reversion** · *Strategy* — The idea that prices tend to return toward an average after moving far from it. It fails when a big move is the start of a new trend.

**Mid price** · *Market* — The price halfway between the best bid and the best ask.

**Min notional** · *Exchange* — The smallest order value (price × quantity) an exchange accepts.

**Momentum** · *Strategy* — The tendency of assets that have been rising (or falling) to keep doing so for a while. The term also covers indicators that measure the speed of price change.

**Monte Carlo simulation** · *Stats* — Running many randomized versions of a scenario, such as reshuffling a backtest's trades, to see a range of possible outcomes instead of one number. (P2)

**Moving average (SMA)** · *Indicator* — The average closing price over the last N periods, recalculated each period, which smooths out noise. "SMA" means simple moving average. See *EMA*.

## N

**Notional value** · *Risk* — The total value of a position (price × quantity), regardless of how much of your own money backs it.

## O

**OBV (On-Balance Volume)** · *Indicator* — A running total that adds a bar's volume when price closes up and subtracts it when price closes down. It hints whether volume is backing a move.

**OCO (one-cancels-the-other)** · *Order* — A pair of orders, such as a take-profit and a stop-loss, where filling one automatically cancels the other.

**OHLCV** · *Market* — Open, high, low, close and volume: the five numbers that describe each candle.

**OMS (order management system)** · *Ubik* — The part of Ubik that tracks every order from creation to fill or cancellation and keeps its records in line with the venue's. When an order's fate is unclear, it asks the venue before acting again (see *UNKNOWN order*).

**On-chain data** · *Crypto* — Information read directly from a blockchain, such as transaction counts, active addresses or coins moving to exchanges. (P2)

**Open interest** · *Market* — The total number of derivative contracts (e.g., futures) that are open and not yet closed. Rising open interest means more positions, and often more leverage, in the market. (P2 data)

**Order** · *Order* — An instruction to an exchange to buy or sell. Common types are market, limit, stop and stop-limit orders.

**Order book** · *Market* — The live list of all waiting buy orders (bids) and sell orders (asks) at each price on an exchange.

**Out-of-sample vault** · *Backtest* — The most recent slice of history (by default 25%, at least 6 months), kept out of tuning and out of the default backtest. You can open it once per strategy version; every opening is counted, and its result is shown separately. *Analogy:* a sealed exam paper. See *In-sample / out-of-sample*.

**Overfitting (curve fitting)** · *Backtest* — Tuning a strategy so closely to past data that it captures random noise instead of a real pattern. The result looks great in backtests and disappoints live. *Analogy:* memorizing last year's exam answers instead of learning the subject.

**Overtrading** · *Behavior* — Trading more than a plan calls for, often out of boredom or emotion. It raises costs and usually lowers returns.

## P

**P&L (profit and loss)** · *Risk* — How much you've gained or lost. *Realized* P&L comes from closed trades. *Unrealized* P&L is the gain or loss on open positions at today's price, which can still change.

**Pair (trading pair)** · *Market* — Two assets traded against each other, such as BTC/USDT. See *Base currency / quote currency*.

**Paper trading** · *Ubik* — Simulated trading with pretend money against live market prices. Ubik always labels it "PAPER · simulated money". It teaches the mechanics, but not the emotions or real-world fill quality.

**Parameter** · *Strategy* — An adjustable setting in a strategy, such as the length of a moving average. The more parameters you tune, the easier it is to overfit.

**Parameter plateau** · *Backtest* — A range of parameter settings where neighbouring values perform about equally well. Robust strategies sit on plateaus; overfit ones sit on sharp, isolated peaks. Ubik's parameter heatmaps show a plateau score.

**Parity replay** · *Ubik* — Re-running a paper bot's exact strategy version through the backtester on the data recorded while it traded, to check both made the same decisions. Ubik runs it daily and reports signal match, fill difference and tracking error; leaving paper needs at least 95% signal match. Not the same as *Shadow mode*.

**Participation rate (cap)** · *Backtest* — The largest share of a bar's traded volume that a simulated order may fill, 10% by default. It stops backtests and paper trading from assuming unlimited liquidity.

**Passkey** · *Ubik* — A phishing-resistant login stored on your phone, laptop or security key that only works on the genuine site. It's Ubik's preferred way to log in. See *2FA / MFA*.

**Pause all** · *Ubik* — The button on every screen that triggers kill-switch level L1 with one click or tap: no new entries, and waiting entry orders are cancelled. Protective stops and exits keep working. Resuming needs confirmation. See *Kill switch*.

**Payoff ratio** · *Stats* — Average winning trade divided by average losing trade. Read it together with the win rate.

**PBO (probability of backtest overfitting)** · *Stats* — The estimated chance that the best-looking setting from an optimization does worse than a typical setting on unseen data. Above 0.5, Ubik shows a strong warning. (P2)

**Perpetual futures (perps)** · *Market* — Futures contracts with no expiry date, kept close to the spot price by the funding rate. They are the most traded crypto derivative. (P4)

**Pilot (stage S4)** · *Ubik* — The first real-money stage: live trading under hard caps, by default at most $500 per strategy and $100 per order. It lasts at least 30 days and 20 round trips before scaling up is considered. (P3)

**Pivot points (classic)** · *Indicator* — Reference levels calculated from the previous period's high, low and close, often watched as possible support or resistance. They are not predictions.

**Point-in-time data** · *Backtest* — Data stored as it was known at each moment, including later corrections and coins that were later delisted, so backtests can't use hindsight. See *Look-ahead bias*, *Survivorship bias*.

**Portfolio** · *Risk* — All the positions and cash held in an account or group of strategies.

**Position** · *Market* — The amount of an asset you currently hold (long) or owe (short).

**Position sizing** · *Risk* — Deciding how much to buy or sell on each trade. Size is often set by how much you'd lose if the stop-loss were hit.

**Post-only order** · *Order* — A limit order that is cancelled if it would fill immediately, so it only ever waits in the order book and pays maker fees.

**Profit factor** · *Stats* — Gross profits divided by gross losses. Above 1 means profits exceeded losses, but a very high value from few trades is a warning sign.

**Protected market order** · *Order* — Ubik's default urgent order: an immediate-or-cancel limit order capped 0.5% beyond the best price, so a thin market can't fill it at a disastrous price. Any unfilled part is reported. See *Time in force*.

**PSR / MinTRL (probabilistic Sharpe ratio / minimum track record length)** · *Stats* — PSR is how confident you can be that a strategy's true Sharpe ratio is above zero, given its track length and unusual returns. MinTRL is how long a record must be to reach that confidence. (P2)

**Pump and dump** · *Behavior* — A scheme where promoters hype an asset to push its price up, then sell to latecomers.

## R

**Rate limit** · *Exchange* — The maximum number of API requests an exchange accepts in a time window. Exceeding it causes errors or temporary bans.

**Rate of Change (ROC)** · *Indicator* — The percentage change in price over the last N bars, a simple measure of momentum.

**Realized volatility / volatility percentile** · *Indicator* — How much price actually moved recently, measured as the annualized standard deviation of returns. The percentile ranks today's level against the past year; Ubik calls the bottom third Calm, the middle Normal and the top third Stormy.

**Rebalancing** · *Risk* — Periodically adjusting holdings back to their target weights.

**Reconciliation** · *Ubik* — Checking that Ubik's records of balances, positions and orders match the venue's own records. A mismatch pauses trading where it was found until the records agree. (P1 against the paper simulator; P3 against the exchange)

**Reference drawdown** · *Risk* — The worst drawdown in a strategy version's full backtest, including the out-of-sample vault, after costs. Ubik sets the strategy *Drawdown breaker*, the pilot's drawdown limit and automatic demotion as multiples of it.

**Regime (market regime)** · *Market* — The market's prevailing character, such as trending or sideways, calm or stormy. Ubik labels trend as Up, Down or Sideways, and volatility as Calm, Normal or Stormy. Strategies that work in one regime often fail in another.

**Relative volume (RVOL)** · *Indicator* — Current volume divided by the usual volume for that time of day. A value of 2 means twice as busy as normal.

**Repainting** · *Indicator* — When an indicator, pattern or signal changes its past values as new bars arrive, making history look more accurate than it was in real time. Ubik lets strategies use such tools only in their confirmed form.

**Resistance** · *Indicator* — A price level where rising prices have repeatedly stalled. It describes past behavior and is not a guaranteed ceiling. See *Support*.

**Retired (strategy version)** · *Ubik* — The final state of a strategy version that has been withdrawn: it never trades again. Retired versions still count in the *Trial ledger*.

**Return** · *Stats* — Gain or loss as a percentage of the starting value.

**Risk engine** · *Ubik* — The part of Ubik that checks every order against your risk limits and can block it, shrink it, or pause or halt trading.

**Risk of ruin** · *Risk* — The probability of losing so much that you can't recover or continue trading.

**Risk per trade** · *Risk* — How much you'd lose if a trade hit its stop-loss, usually given as a percentage of the account (e.g., 1%).

**Risk profile** · *Risk* — A named, versioned set of limits that decides how much risk Ubik allows, such as how much one trade may lose. There are three: Careful (the default), Balanced and Adventurous. It is separate from *Guided mode / Pro mode*, which only change what you see.

**Risk–reward ratio** · *Risk* — A trade's potential loss compared with its potential gain, such as risking $10 to make $20. It only means something alongside a realistic win rate.

**Round trip** · *Backtest* — One complete trade: an entry plus its matching exit. Every trade count, report and graduation gate in Ubik counts round trips, not individual orders or fills. DCA and rebalancing strategies are judged on completed cycles instead.

**RSI (relative strength index)** · *Indicator* — A 0–100 momentum indicator comparing recent gains with recent losses, usually over 14 periods. Readings above 70 are called "overbought" and below 30 "oversold", but in strong trends RSI can stay there a long time.

**Rug pull** · *Crypto* — A scam where a project's creators abandon it and take investors' money. It is most common with small tokens on decentralized exchanges.

## S

**Sample size** · *Stats* — The number of observations, such as trades, behind a result. Ten trades say very little, and small samples make any statistic unreliable.

**Sandbox (code isolation)** · *Ubik* — A locked-down process that runs custom, AI-written or third-party code with no internet, no secrets and strict limits on CPU, memory and time. P1 runs no such code at all. (P2)

**Scalping** · *Strategy* — Very short-term trading that aims for many tiny profits. It is extremely sensitive to fees and slippage.

**Scanner** · *Ubik* — A tool that searches many markets for conditions you define, such as "RSI below 30 on the daily chart". (P2)

**Seed phrase** · *Crypto* — The 12–24 words that can restore a self-custody crypto wallet. Anyone who has it controls the funds, and Ubik will never ask for it.

**Sentiment** · *Market* — The overall mood of market participants, from fearful to greedy, estimated from surveys, social media or indices. (P2)

**Shadow mode (stage S3)** · *Ubik* — A pre-live stage where a strategy watches your real exchange account through a read-only key and records the orders it would place, sending none. It sizes them against your real balances and prices them against the live order book; its simulated fills never touch your live records. Not the same as a *Parity replay*. (P3)

**Sharpe ratio** · *Stats* — Return per unit of volatility: average return above a risk-free rate, divided by the standard deviation of returns, usually annualized. Higher is better, but it understates the risk of rare crashes.

**Short (short selling)** · *Market* — Profiting from a price fall by selling a borrowed asset and buying it back later. Losses can exceed the amount you put in, and it isn't available in spot-only phases.

**Signal** · *Strategy* — The output of a strategy's rules at a given moment: buy, sell or do nothing.

**Signal Lab** · *Ubik* — Ubik's research tool for checking how a signal's past readings were followed by price moves, after costs and on unseen data. Every evaluation counts in the *Trial ledger*. (P2)

**Sim-live parity** · *Ubik* — The principle that the same strategy code runs unchanged in backtest, paper and live modes, so their results are comparable.

**Skew** · *Stats* — The lopsidedness of returns. Strongly negative skew means small, steady gains with rare large losses, a pattern that can hide serious risk.

**Sleeve** · *Ubik* — A bot's own slice of an account's money, with its own positions, fills and profit and loss. A sleeve can only sell coins it holds, so one bot can never sell another's position. Unallocated money stays in a reserve, and manual orders get their own sleeve.

**Slippage** · *Exchange* — The difference between the price you expected and the price you actually got. It grows with order size, volatility and thin liquidity.

**Sortino ratio** · *Stats* — Like the Sharpe ratio, but only downside volatility counts as risk.

**Spot trading** · *Market* — Buying and selling the actual asset for immediate delivery, with no leverage and no expiry. Ubik is spot-only through P3.

**Spread (bid–ask spread)** · *Market* — The gap between the highest bid and the lowest ask. Crossing it is a hidden cost of trading immediately.

**Stablecoin** · *Crypto* — A crypto token designed to hold a steady value, usually $1, such as USDT or USDC. Stablecoins can lose their peg; UST collapsed in 2022.

**Stale data** · *Ubik* — Market data that has stopped updating, such as no trades or quotes for more than 10 seconds. A gap in the order-book feed or a missing 1-minute bar also counts. While a coin's data is stale, Ubik opens no new positions, always lets cancellations through, and exits only with protected orders priced from fresh data. A whole exchange that is stale for over 30 seconds is paused. Stale data is a data-health flag, not a *Trading state*.

**Standard deviation** · *Stats* — The typical distance of values from their average. In finance it is the usual measure of volatility.

**Statistical significance** · *Stats* — How unlikely a result would be if it were due to chance alone. Many backtest "edges" are not statistically significant.

**Step-up re-authentication** · *Ubik* — Proving it's you again, within the last five minutes, before a sensitive action such as arming live trading, adding exchange keys or raising a limit. (P3; P1 for MFA changes and data exports)

**Stochastic oscillator** · *Indicator* — A 0–100 momentum gauge showing where the latest close sits within the recent high–low range. Readings above 80 or below 20 count as stretched, but can stay there in strong trends.

**Stop-limit order** · *Order* — An order that becomes a limit order once a trigger price is reached. It controls price, but in a fast move it may not fill at all.

**Stop-loss** · *Order* — An order that exits a position when price reaches a set level, to cap the loss. In fast markets or price gaps, it may fill worse than the stop price.

**Stop order (stop-market)** · *Order* — An order that becomes a market order once a trigger price is reached. It will fill, but the price isn't guaranteed.

**Strategy** · *Strategy* — A complete set of rules for what to trade, when to enter and exit, and how much to risk.

**Stress test** · *Backtest* — Running a strategy through past crises, such as the March 2020 crash or the FTX collapse, to see how it would have coped. Invented shocks, such as an instant 30% price gap, are added in P2.

**Sub-account** · *Exchange* — A separate account inside your main exchange account. Giving Ubik its own sub-account, funded only with money you're willing to let it trade, limits what a bug or a leaked key can touch.

**Supertrend** · *Indicator* — A trend line set a multiple of ATR away from price that flips sides when price closes through it, signalling a possible change of trend.

**Support** · *Indicator* — A price level where falling prices have repeatedly stopped and bounced. It describes past behavior and is not a guaranteed floor.

**Survivorship bias** · *Backtest* — Analyzing only assets that still exist today and ignoring those that failed or were delisted. It makes past results look better than they really were.

**Swing high / swing low** · *Indicator* — A peak or trough confirmed only after later bars fail to go beyond it. Higher highs and higher lows describe an uptrend; lower highs and lower lows, a downtrend.

**Swing trading** · *Strategy* — Holding trades for days to weeks to capture medium-sized price moves.

## T

**Take-profit** · *Order* — An order that closes a position when price reaches a target gain.

**TCA (transaction cost analysis)** · *Exchange* — Measuring what trading actually cost (fees, spread, slippage) and comparing it with expectations. Ubik shows it for every order, plus each strategy's assumed vs actual costs. (P1 on simulated paper fills; P2 aggregated reports; P3 real fills)

**Technical analysis** · *Indicator* — Making trading decisions from price and volume charts and indicators. Evidence for its predictive power is mixed, so Ubik treats each idea as something to test.

**Testnet** · *Exchange* — An exchange's separate practice market with fake funds and its own prices, useful for testing API connections. Its prices and liquidity aren't real, so Ubik never uses a testnet to judge a strategy or for graduation; see *Exchange demo*. Some exchanges call theirs a "sandbox", which is unrelated to Ubik's code *Sandbox*.

**Tick size** · *Exchange* — The smallest price increment an exchange allows for a pair, e.g., $0.01.

**Ticker (symbol)** · *Market* — A short code identifying an asset or pair, such as BTC or BTC/USDT.

**Tilt** · *Behavior* — An emotional state, usually after losses, that leads to impulsive trades that break your own rules.

**Timeframe** · *Market* — The period each candle represents, such as 1 minute, 4 hours or 1 day. Shorter timeframes mean more noise, more trades and higher costs.

**Time in force** · *Order* — How long an order stays active. *GTC* (good-'til-cancelled) stays until cancelled. *IOC* (immediate-or-cancel) fills what it can now and cancels the rest. *FOK* (fill-or-kill) fills completely now or not at all.

**Top-5 trade share** · *Backtest* — The share of a strategy's total profit that came from its five best trades. Above 50% means the result depends on a few lucky trades.

**Trading state** · *Ubik* — How much trading Ubik currently allows: Running (everything), Paused (only orders that shrink positions) or Halted (cancellations only). A state can apply to everything, one exchange, one strategy or one coin, and the strictest one wins. Internally they are ACTIVE, REDUCE_ONLY and HALTED. Stale data is a data-health flag, not a trading state.

**Trailing stop** · *Order* — A stop-loss that follows the price at a set distance as it moves in your favor, locking in part of the gain.

**Trend** · *Market* — The general direction of price over a period: up, down or sideways.

**Trend following** · *Strategy* — A strategy that buys assets that are rising and exits when the trend breaks. It typically takes many small losses in sideways markets and occasional large gains.

**Trial ledger** · *Backtest* — Ubik's record of every strategy and signal variation you have tested, including AI drafts and retired versions. The more variations tried, the more likely the best result is luck. Ubik shows the count as "variations tried".

**Trust check** · *Ubik* — Ubik's rating of how far to trust a backtest: Low, Low–Medium, Medium or Higher, with its reasons listed. The reasons cover the number of round trips, the period tested, costs, variations tried and out-of-sample testing. It never promises profit, and it differs from an *Evidence label*, which rates an indicator or signal.

**Turnover** · *Backtest* — How much a strategy trades relative to its capital. High turnover means high costs.

**TWAP (time-weighted average price)** · *Order* — An execution method that splits a large order into equal pieces spread over time to reduce its price impact. (P2)

## U

**Underwater chart** · *Backtest* — A chart of drawdown over time, showing how far below its previous peak the account was at each moment.

**UNKNOWN order** · *Order* — An order whose fate is unclear after a timeout or lost connection: it may or may not exist on the exchange. Ubik blocks new orders on that coin, asks the venue, and pauses the coin if it isn't resolved within 30 seconds.

**USDT (Tether)** · *Crypto* — The most widely used stablecoin, intended to track $1. It is issued by a private company and isn't the same as dollars in a bank, so Ubik labels USDT and USD separately.

**UTC (Coordinated Universal Time)** · *Market* — The reference clock most exchanges use. Ubik stores times in UTC and shows them in your local time zone.

## V

**Value at Risk (VaR)** · *Risk* — The loss you should stay under on most days; a 95% VaR is not exceeded on about 19 days out of 20. It says nothing about how bad the other day can be; see *Expected Shortfall*. (P2)

**Volatility** · *Risk* — How much and how fast prices move, usually measured as the standard deviation of returns. Crypto is typically far more volatile than major stock indices.

**Volume** · *Market* — The amount traded in a period. Some exchanges' reported volume is inflated by wash trading.

**VWAP (volume-weighted average price)** · *Indicator* — The average price over a period, weighted by how much traded at each price. It is also used as a yardstick for execution quality.

## W

**Walk-forward analysis** · *Backtest* — Repeatedly tuning a strategy on one window of past data, then testing it on the next unseen window, rolling forward through time. It is more honest than one big optimization. (P2)

**Walk-forward efficiency** · *Backtest* — Out-of-sample return divided by in-sample return in a walk-forward test. Well below about 0.5 hints at overfitting. (P2)

**Wallet (hot / cold)** · *Crypto* — Software or hardware that holds the keys controlling crypto. A hot wallet is online and convenient; a cold wallet is offline and safer for storage.

**Warm-up period** · *Indicator* — The amount of history an indicator needs before its values are reliable. Ubik gives no signals and places no backtest trades until warm-up is complete.

**Wash sale** · *Risk* — Selling an asset at a loss and buying it back shortly after, mainly to claim the tax loss. Some countries restrict this; crypto rules vary by jurisdiction and have been changing. Not the same as wash trading.

**Wash trading** · *Market* — Trading with yourself to fake activity. It inflates reported volume, can mislead analysis, and is market manipulation. Not the same as a wash sale.

**Watchdog** · *Ubik* — A small local process that cancels resting entry orders when Ubik's engine goes silent, on exchanges without a built-in cancel-after timer. It is part of the *Dead-man's switch*. (P3)

**Whale** · *Crypto* — A holder or trader with a very large amount of an asset, whose trades can move prices.

**Wick (shadow)** · *Market* — The thin lines above and below a candle's body, showing that period's high and low.

**Win rate** · *Stats* — The percentage of trades that made money. On its own it means little: 30% can be profitable if wins are large, and 90% can lose money if the losses are huge.

**Withdrawal permission** · *Exchange* — An API key permission that allows moving funds off the exchange. Ubik never needs it, so keep it disabled.

---

## Open questions for the owner

1. Should the glossary also appear as a standalone, searchable page in the Learn hub (recommended), as well as in tooltips?
2. Do you want definitions in any language besides English?
3. Are there terms you've encountered (from YouTube, X, Discord, other bots) that confused you and should be added?

## Risks

- **Inaccuracy spreads everywhere.** Because tooltips, lessons and the assistant all reuse these entries, one wrong definition is repeated across the app. *Mitigation:* expert review before P1 release, versioning and change log (ED-025, ED-026).
- **Oversimplification.** Short definitions can hide important caveats, such as stop-losses not guaranteeing the exit price. *Mitigation:* the key caveat goes in the second sentence, with a lesson link for depth.
- **Exchange-specific drift.** Funding intervals, fee tiers and order types change. *Mitigation:* pull such values from exchange metadata at runtime where possible, rather than hard-coding them in text.
