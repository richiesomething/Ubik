# Ubik — Vision & Scope

> Status: **Draft v0.1** (initial planning, 2026-10-18). Nothing has been built yet. This document frames every other planning doc in this folder; when they disagree, this one wins until a decision in [10-decisions-and-open-questions.md](10-decisions-and-open-questions.md) changes it.

## Summary

- **Ubik is a personal crypto trading platform**: it analyzes markets, lets you design and backtest trading strategies, and runs them automatically. For now it uses **simulated money (paper trading)** only; real money comes later.
- It aims to give an individual the **toolkit professional trading firms use**: market and technical analysis, rigorous backtesting, risk management, execution quality, and performance analytics. It scales those tools down to one person and is honest about where an individual cannot compete.
- It must be **beginner friendly**: every number, chart, and decision gets an explanation in plain language. Advanced depth is there, but unfolds only as you need it.
- Two design rules shape the architecture from day one. **Sim-live parity**: the same strategy code runs in backtest, paper, and live. **Risk engine first**: no order reaches an exchange, simulated or real, without passing risk checks.
- **Honest by default**: Ubik actively warns about overfit backtests, unrealistic expectations, and hidden risk, rather than selling the dream.

## 1. Problem

Retail crypto traders fall into two camps:

1. **Beginners** use exchange "bots" or paid bot platforms. These are easy to start with, but their backtests are often shallow or flattering (no realistic fees or slippage, no out-of-sample testing), and they explain very little. People lose money on strategies they don't understand.
2. **Technical hobbyists** stitch together open-source frameworks, notebooks, and charting tools. These are powerful but fragmented, assume programming and quant knowledge, and are easy to fool yourself with (look-ahead bias, overfitting).

Professional firms avoid these traps through process: clean data, skeptical research, independent validation, hard risk limits, execution measurement, and post-trade review. **Ubik's thesis is that this process can be packaged so one person can follow it, and learn it, without a team.**

## 2. Vision

> *"A trading desk in a box that teaches you as you use it."*

Ubik lets one person do the following:

1. **Understand the market**: charts, indicators, market regime, volatility, derivatives, on-chain and sentiment context, each explained in plain language.
2. **Turn ideas into strategies**: start from templates or describe an idea in words; Ubik helps build it as rules (or Python for advanced users).
3. **Test them honestly**: realistic backtests with fees, slippage, and out-of-sample checks, plus clear warnings when results are probably too good to be true.
4. **Run them safely**: paper-trade with live market data and simulated money under strict risk limits. Later, graduate proven strategies to small real-money allocations.
5. **Learn and improve**: a trade journal, performance attribution, and an AI assistant that can answer "why did my bot do that?"

## 3. Product principles

These are ordered. When two conflict, the higher one wins.

| # | Principle | What it means in practice |
|---|---|---|
| 1 | **Safety before features** | Risk limits, kill switch, and stale-data halts are on by default and can't be bypassed by strategies. Live trading (later) needs explicit opt-in, re-authentication, and passing graduation criteria. API keys never have withdrawal permission. |
| 2 | **Honest by default** | Never present a backtest without costs, a benchmark, and its out-of-sample result. Flag likely overfitting. No promises, hype, or gamification that encourages overtrading. Paper and live are visually unmistakable. |
| 3 | **Sim-live parity** | One strategy codebase, one risk engine, and one order model across backtest, paper, and live. Only the clock, data feed, and exchange adapter change between modes. |
| 4 | **Explain everything** | Every metric, indicator, signal, warning, and blocked trade has a "what does this mean?" explanation. |
| 5 | **Pro depth, beginner surface** | Progressive disclosure: Beginner mode shows the essentials with sensible defaults, and Pro mode exposes the full toolkit. Risk information is never hidden in either mode. |
| 6 | **Evidence over folklore** | Analysis features carry an honest "how reliable is this?" label. Anything that can drive a strategy must be backtestable without look-ahead. |
| 7 | **Own your data and keys** | Self-hostable, single-user first. Your keys, trades, and strategies stay on infrastructure you control. Secrets are never sent to the AI assistant. |
| 8 | **Reproducible** | Any backtest can be re-run to the same result: pinned data snapshot, config, and code version. |

## 4. Goals and success criteria

### 4.1 Product goals

| Goal | Measure (target for end of P1 unless noted) |
|---|---|
| G1. A beginner can get from zero to a first meaningful backtest quickly | A first-time user goes from first launch to a completed backtest of a template strategy, plus a plain-language summary of the result, in **≤ 10 minutes** without outside help |
| G2. Backtests are trustworthy | Golden-test strategies reproduce independently computed results exactly (to the smallest currency unit). Re-running any saved backtest gives identical results. Every report includes fees, slippage, a benchmark, and out-of-sample results |
| G3. Paper trading behaves like real trading would | For the same period, paper-trading results and a backtest replay stay within an agreed tolerance (see the backtesting doc). Paper trading runs **30 days unattended** with no lost state, and restarts never duplicate orders |
| G4. Risk controls always hold | 100% of orders that violate a configured limit are blocked in automated tests across all three modes. The kill switch flattens and halts within seconds |
| G5. Everything is explained | 100% of the metrics, indicators, and warnings shown have a plain-language explanation and a glossary entry |
| G6. Live trading is a configuration change, not a rewrite (P3) | Switching a graduated strategy from paper to live needs no strategy code changes. Only the account, keys, and capital allocation change |

### 4.2 Non-goals (explicitly out of scope)

- **High-frequency trading, co-location, and latency arbitrage.** An individual cannot compete there. Ubik targets decisions on a seconds-to-days timescale, not microseconds.
- **Holding user funds (custody).** Ubik never holds money; it trades through the user's own exchange accounts. **Withdrawal permissions are refused.**
- **Managing other people's money, or selling signals and copy-trading**, at least until a legal review (see the security & compliance doc). That changes the regulatory picture entirely.
- **Guaranteed returns or "set and forget" marketing.** Ubik is a tool for disciplined experimentation, not a money printer.
- **Native mobile apps in P1.** The web UI must be usable on mobile, and push notifications come later.
- **DeFi/DEX trading and leverage in P1–P3.** Leveraged derivatives are P4, behind extra safeguards.

## 5. Users

The detailed personas are in [07-ux-and-beginner-experience.md](07-ux-and-beginner-experience.md). In short:

- **The owner**, who starts as a motivated learner and grows into an advanced user. This is the primary design target.
- **Complete beginners**: curious and risk-unaware, needing guardrails and education.
- **Advanced / quant users**: want Python strategies, raw data, optimization, and full control.

Ubik is **single-user and self-hosted first**. The architecture keeps a path to multi-user open, but nothing in P1–P3 depends on it.

## 6. Scope by phase

| Phase | Name | Scope (headline) | Real money? |
|---|---|---|---|
| **P1** | MVP "Paper Trader" | Market data for a few major spot pairs on 1–2 exchanges; charting and core indicators; template and rule-based strategies; realistic event-driven backtester; paper trading with simulated balances; core risk engine and kill switch; beginner onboarding, explanations, and glossary | No |
| **P2** | "Analyst" | Market regimes, volatility, correlations, derivatives, on-chain and sentiment analysis; scanner and alerts; walk-forward optimization with overfitting statistics; multi-strategy portfolios; AI assistant; richer reports and journal | No |
| **P3** | "Live" | Real exchange trading with staged rollout (shadow mode, then tiny capital, then scaled), graduation criteria, hardened security, reconciliation, monitoring and alerting, incident runbooks, tax exports | **Yes, small and staged** |
| **P4** | "Advanced" | Derivatives and leverage (with extra guardrails), ML-assisted strategies, multi-exchange execution, market making, possibly multi-user | Yes |

[09-roadmap.md](09-roadmap.md) breaks these into milestones with exit criteria.

## 7. Constraints and assumptions

| Item | Assumption | Validated in |
|---|---|---|
| Users | Single owner, self-hosted (home machine or small VPS) | Decisions log |
| Markets | Spot crypto on major centralized exchanges; BTC, ETH, SOL and similar liquid pairs first | [06](06-data-and-architecture.md) |
| Data | Live market data even in paper mode; free or low-cost sources first | [06](06-data-and-architecture.md) |
| Stack (hypothesis) | Python engine and analysis; TypeScript/React web UI; PostgreSQL plus a time-series or columnar store | [06](06-data-and-architecture.md) |
| AI | Claude-powered assistant for explanations and strategy drafting; never holds secrets or places trades without confirmation | [07](07-ux-and-beginner-experience.md), [08](08-security-legal-compliance.md) |
| Jurisdiction | **Unknown.** It affects exchange choice, regulation, and tax | [10](10-decisions-and-open-questions.md) |
| Budget | Unknown. Planning assumes a hobby budget (tens of dollars a month for hosting and AI usage) and open-source components | [10](10-decisions-and-open-questions.md) |

## 8. Document map

| Doc | What it answers |
|---|---|
| [00 Vision & Scope](00-vision-and-scope.md) | Why Ubik exists, principles, goals, scope |
| [01 Requirements Overview](01-requirements-overview.md) | The consolidated requirement set, MVP cut, and cross-cutting non-functional requirements |
| [02 Landscape & Pro Capabilities](02-landscape-and-pro-capabilities.md) | What pro firms do, what competitors offer, build vs reuse |
| [03 Market & Technical Analysis](03-market-and-technical-analysis.md) | Indicators, market analysis, scanner, alerts |
| [04 Strategies, Backtesting & Paper Trading](04-strategies-backtesting-paper-trading.md) | Strategy model, backtester, validation, metrics, paper trading |
| [05 Risk, Portfolio & Execution](05-risk-portfolio-execution.md) | Risk engine, sizing, portfolio, order management, path to live |
| [06 Data & Architecture](06-data-and-architecture.md) | Exchanges, data, storage, system design, tech stack, non-functional requirements |
| [07 UX & Beginner Experience](07-ux-and-beginner-experience.md) | Personas, journeys, screens, education, AI assistant |
| [08 Security, Legal & Compliance](08-security-legal-compliance.md) | Threat model, key handling, regulation, tax, disclaimers |
| [09 Roadmap](09-roadmap.md) | Milestones, sequencing, exit criteria |
| [10 Decisions & Open Questions](10-decisions-and-open-questions.md) | What's decided, what the owner needs to decide |
| [Glossary](glossary.md) | Plain-language definitions of every term |
