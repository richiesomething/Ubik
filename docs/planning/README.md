# Ubik planning docs

Initial planning for **Ubik**, a personal crypto trading platform: market and technical analysis, honest backtesting, and automated trading, with **simulated money first** and real money later. Nothing has been built yet; these docs define what to build and in what order.

## Read this first (15 minutes)

1. [00 Vision & Scope](00-vision-and-scope.md): what Ubik is, its principles, goals and phases.
2. [10 Decisions & Open Questions](10-decisions-and-open-questions.md): what's decided, and **the six questions you need to answer before development starts**.
3. [09 Roadmap](09-roadmap.md): Phase 0, then P1 as five usable slices, then P2–P4.
4. [01 Requirements Overview](01-requirements-overview.md): how the ~500 requirements fit together, the cross-cutting rules, and P1's definition of done.

## Domain specs (deep dives)

| Doc | Covers | IDs |
|---|---|---|
| [02 Landscape & Pro Capabilities](02-landscape-and-pro-capabilities.md) | How professional trading firms optimize trading; competitors and open-source tools; build vs reuse | CAP |
| [03 Market & Technical Analysis](03-market-and-technical-analysis.md) | Indicators, charts, regimes, volatility, derivatives, on-chain, sentiment, scanner, alerts, evidence labels | TA, MA, SIG |
| [04 Strategies, Backtesting & Paper Trading](04-strategies-backtesting-paper-trading.md) | Strategy model, templates, rule builder, backtester, overfitting safeguards, metrics, paper trading, graduation | ST, BT, PT |
| [05 Risk, Portfolio & Execution](05-risk-portfolio-execution.md) | Risk engine, sizing, kill switch, portfolio allocation, order management, execution, TCA, path to live | RK, PF, EX |
| [06 Data & Architecture](06-data-and-architecture.md) | Exchanges, data sources, storage, system architecture, tech stack, deployment, non-functional targets | DA, AR, NFR |
| [07 UX & Beginner Experience](07-ux-and-beginner-experience.md) | Personas, journeys, screens, Guided/Pro modes, education, AI assistant, ethical design | UX, ED |
| [08 Security, Legal & Compliance](08-security-legal-compliance.md) | Threat model, API-key handling, app security, regulation, tax records, disclaimers, privacy | SEC, CMP |
| [Glossary](glossary.md) | Plain-language definitions of every term; the source for in-app tooltips | — |

## How these were made

The planning lead set the vision, scope and working assumptions. Seven specialist research agents wrote the domain specs in parallel, using web research to check facts as of October 2026. A review pass then checked the docs against each other for contradictions, broken cross-references and gaps, and the planning lead wrote the combined overview, roadmap and decisions log. Facts that change quickly (prices, API limits, regulations, library versions) carry inline sources. Re-check them at each phase gate.

**Not financial, legal or tax advice.** Doc 08 marks where a lawyer or tax adviser is needed.
