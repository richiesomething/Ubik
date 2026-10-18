# Ubik Glossary

*Companion to [07-ux-and-beginner-experience.md](07-ux-and-beginner-experience.md) · Draft v0.1 · 2026-10-18*

## Summary

- These are plain-language definitions of the terms Ubik uses. They will seed the in-app tooltips, "What does this mean?" panels and Learn hub (requirements ED-001, ED-025).
- **Tooltip rule:** each entry's first sentence is ≤ 25 words and makes sense on its own as a tooltip. Later sentences add nuance, an analogy or a warning.
- Each term is tagged with a category: *Market, Order, Indicator, Strategy, Backtest, Stats, Risk, Crypto, Exchange, Behavior, Ubik*. Tags such as "(P2)" or "(P4)" mark features that arrive in a later phase.
- Definitions must stay accurate and neutral. When exchange-specific facts change (fee levels, funding intervals), entries are updated and re-versioned (ED-026).

---

## #

**2FA (two-factor authentication)** · *Exchange* — A login check that needs a second proof besides your password, such as a code from an authenticator app. *Analogy:* a door that needs both a key and a PIN.

## A

**All-time high (ATH)** · *Market* — The highest price an asset has ever traded at. It is context, not a target or a promise.

**Alpha** · *Stats* — Return beyond what simple market exposure (e.g., just holding BTC) explains. It's often credited to skill, but it can be luck. *Analogy:* a cyclist's speed after subtracting the help from a tailwind.

**Altcoin** · *Crypto* — Any cryptocurrency other than bitcoin. Many are small, illiquid and far more volatile than BTC.

**API (application programming interface)** · *Exchange* — A way for software to talk to other software. Ubik uses exchange APIs to get prices, and from P3 to place orders.

**API key** · *Exchange* — A credential that lets Ubik access your exchange account with only the permissions you choose. Enable "read" and "trade" only, and never "withdraw". *Analogy:* a valet key that can drive the car but can't open the trunk.

**Arbitrage** · *Strategy* — Buying an asset where it's cheaper and simultaneously selling where it's dearer to capture the price gap. Real opportunities are small, fleeting and often eaten by fees and transfer delays. (P4)

**Arming (live)** · *Ubik* — The deliberate step that switches a strategy from simulated to real-money trading, after warnings, a comprehension quiz and a cooling-off period. (P3)

**Ask (offer)** · *Market* — The lowest price a seller is currently willing to accept. *Analogy:* the price tag on a used car.

**Asset** · *Market* — Anything you can own and trade, such as BTC, ETH or USDT.

**ATR (Average True Range)** · *Indicator* — The average size of each candle's full price range, including gaps, over recent periods (often 14). Stated in price units, it measures volatility and is often used to set stop distances.

**Average entry price** · *Order* — The average price you paid for a position across all its buys, weighted by quantity.

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

**Bollinger Bands** · *Indicator* — A moving average (often 20 periods) with bands usually 2 standard deviations above and below it. The bands widen when prices are volatile and narrow when calm.

**Breakout** · *Strategy* — A move of price beyond a recent range or level, such as above resistance. Breakout strategies buy these moves, and many breakouts quickly reverse ("false breakouts").

**Bull market** · *Market* — A prolonged period of rising prices.

**Buy and hold (HODL)** · *Strategy* — Buying an asset and keeping it through ups and downs. Ubik shows it as the default benchmark, because many active strategies fail to beat it after costs.

## C

**CAGR (compound annual growth rate)** · *Stats* — The steady yearly growth rate that would turn the starting value into the ending value. It smooths the path and hides how bumpy the ride was.

**Calmar ratio** · *Stats* — Annualized return divided by maximum drawdown. Higher means more return per unit of worst-case pain.

**Candlestick (candle)** · *Market* — A chart bar showing one period's open, high, low and close prices. The thick body spans open to close, and thin wicks show the high and low. Ubik marks up and down candles with fill style as well as color.

**Capacity for loss** · *Risk* — How much money you could lose without it materially affecting your life. It differs from risk tolerance, which is how much loss you feel comfortable with.

**Centralized exchange (CEX)** · *Exchange* — A company-run trading platform (e.g., Binance, Coinbase, Kraken) that holds customers' funds and matches their orders. It is convenient and liquid, but you trust the company with your money.

**Circuit breaker** · *Risk* — An automatic rule that halts trading when something abnormal happens, such as a large loss or bad data. Ubik's risk engine uses circuit breakers. See *Kill switch*.

**Coin vs token** · *Crypto* — A coin is native to its own blockchain (BTC, ETH, SOL). A token is issued on top of another blockchain (e.g., USDT on Ethereum).

**Compounding** · *Stats* — Earning returns on previous returns, so growth builds on itself. It works in reverse too: losses shrink the base that future gains grow from.

**Cooling-off period** · *Ubik* — A deliberate waiting time before a risky change takes effect, giving you time to reconsider. Ubik uses 24 hours for risk-increasing changes to live trading.

**Correlation** · *Stats* — A number from −1 to +1 describing how closely two assets move together. Most crypto assets are highly correlated with BTC, so owning several coins diversifies less than it seems.

**Cost basis** · *Risk* — The original value of an asset for tax purposes, usually what you paid including fees. The rules for calculating it vary by country.

**Custody** · *Crypto* — Who holds and controls the assets. On a centralized exchange, the exchange has custody of your funds.

## D

**Data snooping** · *Backtest* — Testing so many ideas on the same historical data that some look good purely by chance. See *Overfitting*.

**Day trading** · *Strategy* — Opening and closing trades within the same day. Studies of individual day traders find that the large majority lose money.

**DCA (dollar-cost averaging)** · *Strategy* — Investing a fixed amount at regular intervals regardless of price. It reduces the risk of bad timing but doesn't prevent losses if the price keeps falling.

**Decentralized exchange (DEX)** · *Exchange* — A platform where trades happen through smart contracts on a blockchain, without a company holding your funds. (P4)

**DeFi (decentralized finance)** · *Crypto* — Financial services such as lending and trading run by blockchain programs (smart contracts) instead of companies. (P4)

**Deflated Sharpe ratio** · *Stats* — A Sharpe ratio adjusted downward for how many strategy variations were tried and for unusual return distributions. It is a guard against overfitting. (P2)

**Depth (market depth)** · *Market* — How much is waiting to be bought and sold at each price level in the order book. Deep markets can absorb large orders without big price moves.

**Derivative** · *Market* — A contract whose value depends on another asset, such as futures, perpetuals or options. Derivatives are often traded with leverage. (P4)

**Disposition effect** · *Behavior* — The tendency to sell winning positions too early and hold losing ones too long, hoping to "get back to even".

**Diversification** · *Risk* — Spreading money across assets that don't move together, to reduce overall risk. It only helps when correlations are genuinely low.

**Drawdown** · *Risk* — The percentage fall in account value from its previous peak to a later low. *Analogy:* how far you've slipped down the mountain since your highest point.

## E

**EMA (exponential moving average)** · *Indicator* — A moving average that gives more weight to recent prices, so it reacts faster than a simple moving average.

**Equity curve** · *Backtest* — A chart of account value over time. A smooth curve is pleasant to look at, but it can hide risk that simply hasn't shown up yet.

**Exchange** · *Exchange* — A marketplace where assets are bought and sold. See *CEX*, *DEX*.

**Execution** · *Order* — Turning a trading decision into filled orders, and how well that is done in terms of price, speed and cost.

**Expectancy** · *Stats* — The average profit or loss per trade: (win rate × average win) − (loss rate × average loss). It needs to be positive after costs.

**Exposure** · *Risk* — How much of your money is currently in the market, in total or per asset.

**Exposure time (time in market)** · *Backtest* — The share of time a strategy holds a position. While out of the market, a strategy avoids crashes but can also miss rallies.

## F

**Fear & Greed Index** · *Market* — A popular 0–100 sentiment score that combines factors like volatility, momentum and social media. It is a rough mood gauge, not a reliable timing signal. (P2)

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

**Graduation** · *Ubik* — Moving a strategy to the next stage on the path to live (backtest → paper → exchange demo → shadow → small pilot → scaled), once its scorecard criteria pass. Promotion needs your approval; demotion is automatic. (P3)

**Grid trading** · *Strategy* — Placing buy and sell orders at fixed intervals around the price to profit from back-and-forth moves. It works in sideways markets but can pile up losing positions in a strong trend.

**Guardrail** · *Ubik* — A default safety limit or check in Ubik, such as a maximum-drawdown pause, that protects you from large mistakes.

## H

**Halving** · *Crypto* — A scheduled event, about every four years, that cuts the amount of new bitcoin created per block in half.

**Hedge** · *Risk* — A position taken to offset the risk of another position. *Analogy:* buying insurance.

**High-water mark** · *Risk* — The highest value an account or strategy has reached. Drawdown is measured from it.

## I

**Implied volatility** · *Market* — The future volatility that options prices suggest traders expect. (P2 data)

**In-sample / out-of-sample** · *Backtest* — In-sample data is used to build and tune a strategy. Out-of-sample data is held back and used only to check whether the strategy still works. *Analogy:* practice exam versus the real exam.

**Indicator** · *Indicator* — A calculation from price and volume data, such as RSI or a moving average, that summarizes what has happened. Indicators describe the past; they don't predict the future.

**IP allowlist** · *Exchange* — An exchange setting that makes an API key work only from internet addresses you specify. It limits damage if the key leaks.

## K

**Kelly criterion** · *Risk* — A formula for the bet size that maximizes long-run growth if you know your true edge. Real edges are uncertain, so full Kelly is dangerously aggressive.

**Kill switch** · *Ubik* — An emergency control that immediately stops all trading and, depending on settings, cancels open orders. It is always one click away.

**KYC (know your customer)** · *Exchange* — The identity checks exchanges are legally required to carry out before letting you trade or withdraw.

## L

**Latency** · *Exchange* — The delay between something happening and the system reacting, such as data arriving or an order reaching the exchange.

**Leverage** · *Risk* — Trading with borrowed money to control a larger position than your own funds. At 10× leverage, a roughly 10% move against you can wipe out your stake. Ubik doesn't offer it before P4.

**Limit order** · *Order* — An order to buy or sell only at a specified price or better. It controls price, but it may never fill if the market doesn't reach that price.

**Liquidation** · *Risk* — The exchange forcibly closing a leveraged position because losses have used up the required margin. (P4)

**Liquidity** · *Market* — How easily you can buy or sell without moving the price much. BTC/USDT on major exchanges is very liquid; many small coins are not.

**Live trading** · *Ubik* — Trading with real money on a real exchange. Ubik always labels it LIVE. (P3)

**Long** · *Market* — Owning an asset, or betting that its price will rise.

**Look-ahead bias** · *Backtest* — A backtest error where the strategy uses information that wasn't available yet at that moment, such as a candle's close before the candle finished. It makes results look unrealistically good.

**Lot size (step size)** · *Exchange* — The smallest quantity increment an exchange accepts for an order on a given pair.

## M

**MACD (moving average convergence divergence)** · *Indicator* — The difference between a 12-period and a 26-period EMA, shown with a 9-period EMA "signal line". It is used to gauge trend direction and momentum.

**Maker / taker** · *Exchange* — A maker's order waits in the order book and adds liquidity. A taker's order fills immediately against existing orders and removes liquidity. Taker fees are usually higher.

**Margin** · *Risk* — Money set aside as collateral for a leveraged position. (P4)

**Mark price** · *Exchange* — A fair-value price that derivatives exchanges use to calculate unrealized profit and loss and liquidations, so that one odd trade can't trigger them. (P4)

**Market cap (market capitalization)** · *Crypto* — Price × circulating supply. It is a rough measure of an asset's total size.

**Market maker** · *Strategy* — A trader or firm that continuously offers to both buy and sell, earning the spread and providing liquidity. (P4)

**Market order** · *Order* — An order to buy or sell immediately at the best available prices. It fills fast, but the final price isn't guaranteed. See *Slippage*.

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

**OCO (one-cancels-the-other)** · *Order* — A pair of orders, such as a take-profit and a stop-loss, where filling one automatically cancels the other.

**OHLCV** · *Market* — Open, high, low, close and volume: the five numbers that describe each candle.

**On-chain data** · *Crypto* — Information read directly from a blockchain, such as transaction counts, active addresses or coins moving to exchanges. (P2)

**Open interest** · *Market* — The total number of derivative contracts (e.g., futures) that are open and not yet closed. Rising open interest means more positions, and often more leverage, in the market. (P2 data)

**Order** · *Order* — An instruction to an exchange to buy or sell. Common types are market, limit, stop and stop-limit orders.

**Order book** · *Market* — The live list of all waiting buy orders (bids) and sell orders (asks) at each price on an exchange.

**Overfitting (curve fitting)** · *Backtest* — Tuning a strategy so closely to past data that it captures random noise instead of a real pattern. The result looks great in backtests and disappoints live. *Analogy:* memorizing last year's exam answers instead of learning the subject.

**Overtrading** · *Behavior* — Trading more than a plan calls for, often out of boredom or emotion. It raises costs and usually lowers returns.

## P

**P&L (profit and loss)** · *Risk* — How much you've gained or lost. *Realized* P&L comes from closed trades. *Unrealized* P&L is the gain or loss on open positions at today's price, which can still change.

**Pair (trading pair)** · *Market* — Two assets traded against each other, such as BTC/USDT. See *Base currency / quote currency*.

**Paper trading** · *Ubik* — Simulated trading with pretend money against live market prices. Ubik always labels it PAPER. It teaches the mechanics, but not the emotions or real-world fill quality.

**Parameter** · *Strategy* — An adjustable setting in a strategy, such as the length of a moving average. The more parameters you tune, the easier it is to overfit.

**Payoff ratio** · *Stats* — Average winning trade divided by average losing trade. Read it together with the win rate.

**Perpetual futures (perps)** · *Market* — Futures contracts with no expiry date, kept close to the spot price by the funding rate. They are the most traded crypto derivative. (P4)

**Portfolio** · *Risk* — All the positions and cash held in an account or group of strategies.

**Position** · *Market* — The amount of an asset you currently hold (long) or owe (short).

**Position sizing** · *Risk* — Deciding how much to buy or sell on each trade. Size is often set by how much you'd lose if the stop-loss were hit.

**Post-only order** · *Order* — A limit order that is cancelled rather than filled immediately, so it always pays maker fees.

**Profit factor** · *Stats* — Gross profits divided by gross losses. Above 1 means profits exceeded losses, but a very high value from few trades is a warning sign.

**Pump and dump** · *Behavior* — A scheme where promoters hype an asset to push its price up, then sell to latecomers.

## R

**Rate limit** · *Exchange* — The maximum number of API requests an exchange accepts in a time window. Exceeding it causes errors or temporary bans.

**Rebalancing** · *Risk* — Periodically adjusting holdings back to their target weights.

**Reconciliation** · *Ubik* — Checking that Ubik's records of balances, positions and orders match the exchange's own records. (P3)

**Regime (market regime)** · *Market* — The market's prevailing character: trending up, trending down, sideways, calm or volatile. Strategies that work in one regime often fail in another.

**Resistance** · *Indicator* — A price level where rising prices have repeatedly stalled. It describes past behavior and is not a guaranteed ceiling. See *Support*.

**Return** · *Stats* — Gain or loss as a percentage of the starting value.

**Risk engine** · *Ubik* — The part of Ubik that checks every order against your risk limits and can block it, resize it or pause a strategy.

**Risk of ruin** · *Risk* — The probability of losing so much that you can't recover or continue trading.

**Risk per trade** · *Risk* — How much you'd lose if a trade hit its stop-loss, usually given as a percentage of the account (e.g., 1%).

**Risk–reward ratio** · *Risk* — A trade's potential loss compared with its potential gain, such as risking $10 to make $20. It only means something alongside a realistic win rate.

**RSI (relative strength index)** · *Indicator* — A 0–100 momentum indicator comparing recent gains with recent losses, usually over 14 periods. Readings above 70 are called "overbought" and below 30 "oversold", but in strong trends RSI can stay there a long time.

**Rug pull** · *Crypto* — A scam where a project's creators abandon it and take investors' money. It is most common with small tokens on decentralized exchanges.

## S

**Sample size** · *Stats* — The number of observations, such as trades, behind a result. Ten trades say very little, and small samples make any statistic unreliable.

**Scalping** · *Strategy* — Very short-term trading that aims for many tiny profits. It is extremely sensitive to fees and slippage.

**Scanner** · *Ubik* — A tool that searches many markets for conditions you define, such as "RSI below 30 on the daily chart". (P2)

**Seed phrase** · *Crypto* — The 12–24 words that can restore a self-custody crypto wallet. Anyone who has it controls the funds, and Ubik will never ask for it.

**Sentiment** · *Market* — The overall mood of market participants, from fearful to greedy, estimated from surveys, social media or indices. (P2)

**Shadow mode** · *Ubik* — A pre-live stage where a strategy watches your real exchange account with a read-only key and records the orders it would have placed, without sending any. (P3)

**Sharpe ratio** · *Stats* — Return per unit of volatility: average return above a risk-free rate, divided by the standard deviation of returns, usually annualized. Higher is better, but it understates the risk of rare crashes.

**Short (short selling)** · *Market* — Profiting from a price fall by selling a borrowed asset and buying it back later. Losses can exceed the amount you put in, and it isn't available in spot-only phases.

**Signal** · *Strategy* — The output of a strategy's rules at a given moment: buy, sell or do nothing.

**Sim-live parity** · *Ubik* — The principle that the same strategy code runs unchanged in backtest, paper and live modes, so their results are comparable.

**Slippage** · *Exchange* — The difference between the price you expected and the price you actually got. It grows with order size, volatility and thin liquidity.

**Sortino ratio** · *Stats* — Like the Sharpe ratio, but only downside volatility counts as risk.

**Spot trading** · *Market* — Buying and selling the actual asset for immediate delivery, with no leverage and no expiry. Ubik is spot-only through P3.

**Spread (bid–ask spread)** · *Market* — The gap between the highest bid and the lowest ask. Crossing it is a hidden cost of trading immediately.

**Stablecoin** · *Crypto* — A crypto token designed to hold a steady value, usually $1, such as USDT or USDC. Stablecoins can lose their peg; UST collapsed in 2022.

**Standard deviation** · *Stats* — The typical distance of values from their average. In finance it is the usual measure of volatility.

**Statistical significance** · *Stats* — How unlikely a result would be if it were due to chance alone. Many backtest "edges" are not statistically significant.

**Stop-limit order** · *Order* — An order that becomes a limit order once a trigger price is reached. It controls price, but in a fast move it may not fill at all.

**Stop-loss** · *Order* — An order that exits a position when price reaches a set level, to cap the loss. In fast markets or price gaps, it may fill worse than the stop price.

**Stop order (stop-market)** · *Order* — An order that becomes a market order once a trigger price is reached. It will fill, but the price isn't guaranteed.

**Strategy** · *Strategy* — A complete set of rules for what to trade, when to enter and exit, and how much to risk.

**Support** · *Indicator* — A price level where falling prices have repeatedly stopped and bounced. It describes past behavior and is not a guaranteed floor.

**Survivorship bias** · *Backtest* — Analyzing only assets that still exist today and ignoring those that failed or were delisted. It makes past results look better than they really were.

**Swing trading** · *Strategy* — Holding trades for days to weeks to capture medium-sized price moves.

## T

**Take-profit** · *Order* — An order that closes a position when price reaches a target gain.

**TCA (transaction cost analysis)** · *Exchange* — Measuring what trading actually cost (fees, spread, slippage) and comparing it with expectations. (P2–P3)

**Technical analysis** · *Indicator* — Making trading decisions from price and volume charts and indicators. Evidence for its predictive power is mixed, so Ubik treats each idea as something to test.

**Testnet (sandbox)** · *Exchange* — An exchange's practice environment with fake funds, used to test API connections safely.

**Tick size** · *Exchange* — The smallest price increment an exchange allows for a pair, e.g., $0.01.

**Ticker (symbol)** · *Market* — A short code identifying an asset or pair, such as BTC or BTC/USDT.

**Tilt** · *Behavior* — An emotional state, usually after losses, that leads to impulsive trades that break your own rules.

**Timeframe** · *Market* — The period each candle represents, such as 1 minute, 4 hours or 1 day. Shorter timeframes mean more noise, more trades and higher costs.

**Time in force** · *Order* — How long an order stays active. *GTC* (good-'til-cancelled) stays until cancelled. *IOC* (immediate-or-cancel) fills what it can now and cancels the rest. *FOK* (fill-or-kill) fills completely now or not at all.

**Trailing stop** · *Order* — A stop-loss that follows the price at a set distance as it moves in your favor, locking in part of the gain.

**Trend** · *Market* — The general direction of price over a period: up, down or sideways.

**Trend following** · *Strategy* — A strategy that buys assets that are rising and exits when the trend breaks. It typically takes many small losses in sideways markets and occasional large gains.

**Turnover** · *Backtest* — How much a strategy trades relative to its capital. High turnover means high costs.

**TWAP (time-weighted average price)** · *Order* — An execution method that splits a large order into equal pieces spread over time to reduce its price impact. (P3)

## U

**Underwater chart** · *Backtest* — A chart of drawdown over time, showing how far below its previous peak the account was at each moment.

**USDT (Tether)** · *Crypto* — The most widely used stablecoin, intended to track $1. It is issued by a private company and isn't the same as dollars in a bank, so Ubik labels USDT and USD separately.

**UTC (Coordinated Universal Time)** · *Market* — The reference clock most exchanges use. Ubik stores times in UTC and shows them in your local time zone.

## V

**Volatility** · *Risk* — How much and how fast prices move, usually measured as the standard deviation of returns. Crypto is typically far more volatile than major stock indices.

**Volume** · *Market* — The amount traded in a period. Some exchanges' reported volume is inflated by wash trading.

**VWAP (volume-weighted average price)** · *Indicator* — The average price over a period, weighted by how much traded at each price. It is also used as a yardstick for execution quality.

## W

**Walk-forward analysis** · *Backtest* — Repeatedly tuning a strategy on one window of past data, then testing it on the next unseen window, rolling forward through time. It is more honest than one big optimization. (P2)

**Wallet (hot / cold)** · *Crypto* — Software or hardware that holds the keys controlling crypto. A hot wallet is online and convenient; a cold wallet is offline and safer for storage.

**Wash sale** · *Risk* — Selling an asset at a loss and buying it back shortly after, mainly to claim the tax loss. Some countries restrict this; crypto rules vary by jurisdiction and have been changing. Not the same as wash trading.

**Wash trading** · *Market* — Trading with yourself to fake activity. It inflates reported volume, can mislead analysis, and is market manipulation. Not the same as a wash sale.

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
