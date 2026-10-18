# Security, Legal & Compliance Plan

> **Not legal, tax, or financial advice.** This is a planning document written by a non-lawyer. Sources were checked on 2026-10-18; laws and exchange terms change often. Items marked **⚖ Lawyer** need review by a qualified professional in the owner's jurisdiction before the related phase ships.

## Summary

- **Isolate the exchange keys.** Exchange API keys are the most valuable thing Ubik will ever hold. Only one small component, the *execution gateway*, should be able to decrypt them. Strategies, the web UI and the AI assistant must have no way to reach them. Ubik must **check with the exchange and refuse any key that can withdraw or transfer funds**, and must require keys to be locked to Ubik's server IP address (*IP allow-listing*). Trade-only keys can still be abused: in the 3Commas leak, attackers used them to place trades that drained users' accounts. So Ubik also needs a symbol allow-list, a dedicated exchange sub-account, and a kill switch.
- **P1 stores no trade-capable keys at all.** Paper trading needs only public market data. P1 still builds the foundations that P3 depends on: login with multi-factor authentication (MFA), private-by-default networking, pinned dependencies, an audit log, and separate paper and live ledgers.
- **Going live is a deliberate "arming" step, not a config edit.** I partly disagree with "mostly a configuration change". Strategy code and settings should carry over unchanged. But *turning on* live trading should require re-authentication, a risk quiz and a cooling-off period. Config files can only *disarm*. The general rule: anything that raises risk is slow and deliberate; anything that lowers risk is instant.
- **Personal use is the low-risk zone; sharing Ubik is where licences start.** A bot trading your own money on your own accounts generally needs no financial licence. Exchange terms, tax law and market-manipulation law still apply. Signals, copy trading, managed strategies or a hosted service for other people can count as regulated advice, portfolio management or order handling under EU MiCA, US or UK rules. That step needs a lawyer before it ships.
- **Tax-grade records from day one, with paper and live kept apart.** Ubik needs an append-only ledger that records cost basis, fees, fiat value at the time of the trade, and which account the trade was in. Exports: generic CSV/JSON, plus Koinly and CoinTracker formats. Exchanges now report trades directly to tax authorities: EU DAC8 and OECD CARF started collecting data on 2026-01-01, and US brokers file Form 1099-DA. Ubik's records must reconcile with what the exchanges report.
- **Treat the AI assistant as untrusted.** It never sees secrets and can only *propose* actions; the user confirms them in the normal UI. It cannot place orders, arm live trading or change limits. News and social text passed to it are labelled as untrusted data, because they can carry hidden instructions (*prompt injection*).

---

## 1. Principles

**Least privilege** (every key and component gets the minimum access it needs) · **secure and private by default** · **fail safe** (when in doubt, stop opening positions; don't guess) · **asymmetric friction** (safer is instant; riskier needs re-authentication, an explanation and sometimes a delay) · **assume breach** (limit the damage any single compromise can do) · **honesty** (warnings sit next to the numbers they qualify).

## 2. Assets and trust boundaries

| Asset | Why it matters | Phase |
|---|---|---|
| Exchange API secrets | Direct control over real funds | P3+ |
| Ubik login (credentials, passkeys, sessions) | Can arm trading, change limits, read finances | P1+ |
| Order path (strategy → risk check → gateway → exchange) | Tampering here means unintended trades | P1 (paper), P3 |
| Market-data integrity | Bad prices lead to bad trades | P1+ |
| Trade ledger and audit log | Tax evidence; investigating incidents | P1+ |
| LLM API key and chat history | Cost abuse; reveals the owner's finances | P2+ |
| Personal data; backups plus the encryption master key | Privacy; anyone holding both a backup and the key can recover everything | P1+ |

The full component design is in `06`. The security-relevant shape:

```
[Browser] --TLS--> [Web/API] --> [Engine: strategies (sandboxed)] --order intents--> [Risk check]
                                                                                        |
[LLM gateway: no secrets, read-only tools]                         [Execution gateway: ONLY holder of
[Market-data ingest: untrusted input, validated]                    decrypted exchange keys] --TLS--> [Exchange]
```

## 3. Threat model

The table uses STRIDE, a standard checklist of threat types: **S**poofing identity, **T**ampering, **R**epudiation (denying an action), **I**nformation disclosure, **D**enial of service, **E**levation of privilege.

| # | Threat (STRIDE) | Real-world lesson | Key mitigations |
|---|---|---|---|
| T1 | **API-key theft** (I) from the database, logs, backups, the owner's PC, or a browser extension | **3Commas (2022):** about 100k users' keys leaked, followed by unauthorized trades and $20M+ in reported losses. The company first denied the leak, then confirmed it ([BleepingComputer](https://www.bleepingcomputer.com/news/security/crypto-platform-3commas-admits-hackers-stole-api-keys), [HAPI analysis](https://forklog.com/en/news/api-key-leaks-and-exchange-inaction-a-hapi-analysis-of-the-3commas-incident)). **MEXC Chrome extension (2025–26):** it silently enabled withdrawal permission on a newly created key, hid the checkbox tick in the page, and sent the key to a Telegram bot ([Socket](https://socket.dev/blog/malicious-chrome-extension-steals-mexc-api-keys)). | Envelope encryption; a single key-holding gateway; check permissions through the exchange API rather than the UI; IP allow-list; create keys in a browser profile with no extensions |
| T2 | **Abuse of a stolen trade-only key** (E) | In the 3Commas losses, victims' accounts bought illiquid coins at inflated prices from the attackers' own sell orders. "Trade-only" does not mean "safe". | IP allow-list; symbol allow-list in the gateway; sub-account holding only Ubik's capital; alert on unknown orders |
| T3 | **Ubik account takeover** (S) via credential stuffing, session theft, or phishing | **AIXBT (Mar 2025):** an attacker got into an AI trading agent's dashboard and moved about 55 ETH ([The Block](https://www.theblock.co/post/346911/)). | Passkeys or TOTP; no public exposure by default; re-authentication before sensitive actions; device alerts |
| T4 | **Compromised server or VPS** (E/I) | Exposed admin ports, SSH passwords, unpatched hosts. | Private network access; host hardening; key-encryption key held outside the database; IP allow-list; revoke runbook |
| T5 | **Malicious or buggy strategy code** (T/E), whether user-written, LLM-generated or downloaded | **Knight Capital (2012):** a dormant code path sent more than 4M orders in 45 minutes, causing a $460M+ loss ([SEC](https://www.sec.gov/news/press-release/2013-222)). **Fake "trading bot" YouTube tutorials** drained more than $900k ([SentinelLABS](https://www.sentinelone.com/?p=125939)). | Declarative strategies in P1; sandbox from P2; every order passes the risk check; order-rate limits; kill switch |
| T6 | **Supply-chain attack** on dependencies or the build (T) | **npm, Sep 2025:** a phished maintainer pushed crypto-stealing versions of `chalk`/`debug`, followed by the token-stealing "Shai-Hulud" worm ([Pulsedive](https://blog.pulsedive.com/npm-compromise-the-wrath-of-the-shai-hulud-supply-chain-attack/)). **PyPI `ccxt-mexc-futures` (2025)** stole exchange API keys ([JFrog](https://jfrog.com/blog/malicious-pypi-package-hijacks-mexc-orders-steals-crypto-tokens/)). **Bybit (Feb 2025, ~$1.5B):** a vendor developer's machine was compromised and the vendor's web-app JavaScript was swapped ([BleepingComputer](https://www.bleepingcomputer.com/news/security/lazarus-hacked-bybit-via-a-breached-safe-wallet-developer-machine/)). **"Slopsquatting":** attackers register package names that LLMs hallucinate ([CSA](https://labs.cloudsecurityalliance.org/research/csa-research-note-slopsquatting-ai-supply-chain-20260419-csa/)). | Hash-pinned lockfiles; delay adopting new releases; review before adding dependencies; scanning; signed releases; CI hardening |
| T7 | **Phishing or social engineering of the owner** (S) | The **Coinbase insider breach (May 2025)** leaked customer data that was then used to phish those customers; estimated cost $180–400M ([SecurityInfoWatch](https://www.securityinfowatch.com/cybersecurity/article/55290995/coinbase-reveals-insider-bribery-scheme-led-to-data-breach-potential-400m-cost)). | Passkeys (can't be phished); "Ubik will never ask for…" onboarding; signed updates only |
| T8 | **Tampered or bad market data** (T) | Flash wicks, stale feeds, outages, thin-market manipulation. | TLS; staleness and sanity checks; cross-check against a second source; circuit breakers (`05`, `06`) |
| T9 | **Prompt injection against the AI assistant** (T/E) via news, social posts, token metadata, or strategy comments | **Princeton/Sentient (2025):** "memory injection" planted false memories in crypto agents built on ElizaOS, later redirecting transfers ([Decrypt](https://decrypt.co/318200/elizaos-vulnerability-ai-gaslit-losing-millions)). OWASP ranks prompt injection as the #1 LLM risk ([OWASP](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)). | No secrets; read-only tools; proposals only, confirmed by the user; content labelled untrusted; no memory writes from untrusted text |
| T10 | **Insider or cloud-provider risk** (I) | The same Coinbase case: insiders at an outsourced vendor were bribed. | Client-side encrypted backups; the master key is never stored with the backup provider; minimal data sent to the LLM |
| T11 | **Exchange failure or hack** (D) | **Bitget (Sep 2026):** about $387.5M stolen via third-party zero-days ([summary of Reuters/FT reporting](https://en.wikipedia.org/wiki/Bitget)). | Keep only trading capital on exchanges; spread across venues (`05`); runbook |
| T12 | **Record tampering or repudiation** (R) | Edited history breaks tax evidence and incident investigations. | Hash-chained, append-only logs with off-host checkpoints |
| T13 | **Accidental live trading** (T) | A paper config pointed at live keys. | No trade keys in P1; arming ceremony; separate keys per environment; visibly different paper and live UI |

## 4. Architecture-level security requirements

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-001 | Ubik shall confine decryption and use of exchange trading credentials to a single **execution-gateway** process. Strategy code, the web/API layer and the AI assistant shall have no code path or network route to them. | Must | P3 (design in P1) | Limits the blast radius of T1, T5, T9. `06` places the component. |
| SEC-002 | Every order, whether from a strategy, the manual UI or an AI proposal, shall pass the pre-trade risk check (`05`) before reaching the gateway. Paper mode runs the same check. | Must | P1 | Sim-live parity. |
| SEC-003 | Live trading shall start only through an authenticated **arming** action (SEC-016 step-up plus CMP-022 flow) recorded in the audit log. Config files and environment variables may disarm live trading but not arm it. | Must | P3 | T13; stops malicious config edits. |
| SEC-004 | Paper and live shall use separate credential stores, ledgers, and visibly different UI themes. Paper mode shall need no private exchange credentials. Testnet and live keys shall be separate, and Ubik shall refuse a key fingerprint already registered in another environment. | Must | P1 | T13. |
| SEC-005 | A **kill switch** (UI, server CLI, mobile-friendly authenticated action) shall within 10 s stop strategies, cancel open orders, and block new orders until re-armed. | Must | P1 (paper), P3 | Knight Capital lesson. Semantics in `05`. |
| SEC-006 | The gateway shall enforce a per-deployment **symbol allow-list** and a maximum order notional, independent of strategy configuration. | Must | P3 | T2, T5. |

## 5. Exchange API-key handling

An **API key** is a key/secret pair that lets software act on your exchange account. Exchanges let you choose what each key can do: read, trade, or withdraw.

**Owner's key checklist (plain language, shown in P3 onboarding):**
1. Create an exchange **sub-account** for Ubik and fund it only with money you're willing for Ubik to trade.
2. Create the key in a browser profile with no extensions.
3. Tick **read** and **trade** only. Never tick withdraw, transfer, or "add withdrawal addresses".
4. Restrict the key to Ubik's server IP address; Ubik shows you which address to use.
5. Set an expiry date if the exchange offers one. Turn on the exchange's withdrawal-address allow-list and account 2FA.
6. Paste the key into Ubik once. Ubik checks it with the exchange and refuses it if anything is wrong.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-007 | In P1, Ubik shall not accept or store trade-capable or withdrawal-capable exchange keys. Read-only keys are allowed only for importing balances. | Must | P1 | Removes the top risk while trading is simulated. |
| SEC-008 | When a key is added, at every engine start, and daily, Ubik shall query the exchange's key-permission endpoint where one exists. Examples: Binance `GET /sapi/v1/account/apiRestrictions` (`enableWithdrawals`, `ipRestrict`, transfer flags) ([docs](https://developers.binance.com/legacy-docs/zh-CN/wallet/account/api-key-permission)); Coinbase `GET /api/v3/brokerage/key_permissions` (`can_transfer`) ([docs](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/data-api/get-api-key-permissions.md)). | Must | P3 | Verify on the server, not in the UI (MEXC lesson). `06` re-checks field names per exchange. |
| SEC-009 | Ubik shall **refuse** keys with withdrawal, internal or universal transfer, or withdrawal-address-management permission. If a check fails or such a permission appears later, Ubik shall disarm live trading and alert the owner. | Must | P3 | Kraken warns that "Add Withdrawal Addresses" plus "Withdraw" lets an attacker drain an account ([Kraken](https://support.kraken.com/articles/withdrawal-addresses-api-permission)). |
| SEC-010 | Where no permission endpoint exists, Ubik shall require a guided attestation, mark the venue "permissions unverified" with a persistent warning, and cap that venue at the lowest capital stage. | Must | P3 | Venues differ. |
| SEC-011 | Ubik shall refuse live keys that are not IP-restricted on exchanges that support IP allow-listing. | Must | P3 | A stolen key becomes useless from other networks. Binance also reportedly expires unrestricted keys ([TradersPost](https://blog.traderspost.io/article/managing-binance-api-key-expiry-on-traderspost)). Needs SEC-023. |
| SEC-012 | Secrets shall be stored with **envelope encryption**. Each secret is encrypted with its own data key (AES-256-GCM or XChaCha20-Poly1305), and that data key is wrapped by a **key-encryption key (KEK)** kept *outside* the database: OS keyring or TPM (e.g. systemd-creds), cloud KMS, Vault/OpenBao, or an unlock passphrase run through Argon2id. | Must | P3 | A database dump or backup alone reveals nothing. KEK-at-boot allows unattended restarts; passphrase-at-boot is safer. Owner chooses (Open questions). |
| SEC-013 | Secrets shall be **write-only** via the API. The UI shows only label, exchange, key-ID suffix, verified permissions, IP-restriction status and dates. Decrypted secrets exist only in gateway memory and are excluded from core dumps. | Must | P3 | — |
| SEC-014 | No secret (exchange, LLM, database, session) shall appear in logs, traces, error reports, metrics, URLs, browser storage, LLM prompts or unencrypted backups. This shall be enforced by a typed secret wrapper, log redaction, and a CI test that plants a canary secret and searches all outputs for it. | Must | P1 | 28.65M secrets leaked to public GitHub in 2025, and AI-assisted commits leaked at about 2× the baseline rate ([GitGuardian via THN](https://thehackernews.com/2026/03/the-state-of-secrets-sprawl-2026-9.html?m=1)). |
| SEC-015 | Ubik shall prompt key rotation (default every 90 days) with a guided flow that verifies the new key before retiring the old one. A **"Key compromised"** button shall trigger the kill switch, wipe the stored secret, log the event, and deep-link to the exchange's revoke page. | Must | P3 | Response speed matters. |

## 6. Application security

### 6.1 Authentication, sessions, authorization

A **passkey** (WebAuthn) is a login credential held by your phone, laptop or hardware key. It can't be phished because it only works on the real site. **TOTP** is the 6-digit code from an authenticator app. NIST's 2025 guidance (SP 800-63-4) accepts synced passkeys and requires services to *offer* a phishing-resistant option ([Yubico summary](https://yubico.com/blog/new-nist-guidance-on-passkeys-key-takeaways-for-enterprises)).

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-016 | **Step-up re-authentication** (within the last 5 minutes, phishing-resistant factor) shall be required to: arm live trading; add, replace or revoke keys; raise any limit or allocation; extend the symbol allow-list; disable a guardrail; change MFA; export data; restore a backup; promote a strategy to live. | Must | P3 (P1 for MFA changes and export) | Protects against a hijacked open session. |
| SEC-017 | All UI and API routes shall require login, even for a single user. Ubik shall support passkeys (preferred), or a password (≥12 characters, checked against breached-password lists, hashed with Argon2id) plus TOTP. No SMS. MFA is mandatory before any non-localhost exposure. Login is rate-limited. Recovery codes are issued, and an MFA bypass is possible only through the server-console CLI. | Must | P1 | T3, T7. |
| SEC-018 | Risk-increasing changes (first arming, raising limits or allocation) shall take effect only after a cooling-off delay (default 24 h for first arming; 1–24 h configurable for limit increases), with a notification. Risk-reducing changes are immediate. | Should | P3 | Modelled on the UK 24-hour cooling-off period for first-time crypto investors ([FCA](https://www.fca.org.uk/publications/good-and-poor-practice/assessing-compliance-back-end-cryptoasset-financial-promotions-rules)). UX in `07`. |
| SEC-019 | Sessions shall use `HttpOnly; Secure; SameSite=Strict` cookies, rotate on login, enforce idle (30 min while armed) and absolute (12 h) timeouts, use CSRF protection, and offer a device list with remote revoke. | Must | P1 | — |
| SEC-020 | Every user-owned row shall carry `owner_id` (later `tenant_id`) from P1. Data access shall go through a deny-by-default authorization layer with automated cross-owner access tests. | Must | P1 | Keeps the P4 multi-user path open without rework. |

### 6.2 Network exposure and host security (self-hosting)

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-021 | By default Ubik shall bind to localhost or a private interface. The installer shall refuse public exposure without TLS and MFA. Documented remote access: a WireGuard/Tailscale private network (recommended), or a reverse proxy with automatic TLS (Caddy/Traefik). The database, message bus and metrics shall never be published. | Must | P1 | Self-hosted bot dashboards exposed to the internet are a recurring weakness. |
| SEC-022 | The web UI shall send a strict CSP (no inline scripts), HSTS, `frame-ancestors 'none'`, and a restrictive Referrer-Policy. Assets shall be self-hosted, or loaded with Subresource Integrity (a hash check on each file). | Must | P1 | Bybit lesson: a swapped front-end file. |
| SEC-023 | Live trading shall require a **static egress IP** (for SEC-011) and a server location the exchange permits (CMP-006). Both shall be checked by a health check. | Must | P3 | A home connection with a changing IP breaks allow-listing. A small VPS is often the better choice for live trading. |
| SEC-024 | The host baseline shall include automatic security updates, SSH key-only login, a default-deny firewall, non-root containers with read-only root filesystems where possible, disk encryption, and least-privilege database roles. | Must | P3 (Should P1) | T4. |

### 6.3 Supply chain and secure development

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-025 | Dependencies shall be locked with hashes (`uv`/pip `--require-hashes`, `npm ci`). Automated update PRs (Renovate/Dependabot) shall wait several days (default 7) before adopting new releases, except security fixes. | Must | P1 | Most malicious releases are caught within hours or days. |
| SEC-026 | Adding a direct dependency shall require recorded review: exact name checked against the official project (typosquat/slopsquat check), maintainer and age, OpenSSF Scorecard, and necessity. Packages that touch keys or orders get extra scrutiny. LLM-suggested package names count as unverified. | Must | P1 | T6. |
| SEC-027 | CI shall run dependency scanning (pip-audit/OSV-Scanner, npm audit), static analysis (Semgrep/CodeQL), secret scanning (gitleaks plus push protection) and container scanning (Trivy). Actions shall be pinned to commit SHAs, `GITHUB_TOKEN` least-privilege, no secrets exposed to fork PRs, and npm install scripts disabled. Code-hosting and registry accounts need phishing-resistant MFA. Releases are blocked on critical findings. | Must | P1 | Shai-Hulud stole tokens from CI. |
| SEC-028 | Release images shall be signed (Sigstore cosign) and ship with an SBOM (software bill of materials). The updater verifies signatures and won't update the engine while positions are open without confirmation. | Should | P3 | — |
| SEC-029 | Changes to the gateway, risk engine, auth or crypto code shall need reviewed PRs with a security checklist. The threat model is revisited at each phase gate. Before P3: an OWASP ASVS 5.0 Level 2 self-assessment and an external penetration test of the auth, key and order paths. | Must | P3 (Should P1 for review) | Real money raises the bar. |

### 6.4 Audit logs, backups, monitoring, incident response

A **hash-chained log** is one where each entry includes a fingerprint (hash) of the previous one. Editing or deleting old entries breaks the chain, so tampering is detectable.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-030 | Ubik shall keep an append-only, hash-chained audit log of: auth and MFA events; session changes; key events; arming and disarming; kill-switch use; limit and config changes; strategy promotions; every order intent, risk decision, exchange acknowledgement and fill; AI tool calls and proposals. The app's database role cannot UPDATE or DELETE audit rows. A signed daily checkpoint is exported off-host, and a verify command detects breaks. | Must | P1 (paper events), P3 (checkpoints) | T12; feeds CMP-011. |
| SEC-031 | Automated, encrypted, off-host backups (e.g. restic/age) of the database and config. The KEK is backed up separately (password manager or printed recovery sheet). Restores are tested quarterly and before each phase release. RPO/RTO targets come from `06`. | Must | P1 (restore tests Must from P3) | An untested backup can't be relied on. |
| SEC-032 | Security alerts (push, email or Telegram, no secrets) for: new device login, repeated MFA failures, key changes, failed permission re-checks, arming, limit raises, unknown orders found in reconciliation, exchange auth errors. | Must | P3 | — |
| SEC-033 | Incident runbooks, rehearsed before P3, for: key compromise, server compromise, exchange hack or outage, dependency compromise, lost KEK. | Must | P3 | — |

### 6.5 Strategy code sandboxing (user-written and LLM-generated)

A **sandbox** runs untrusted code in a locked box: no internet, no secrets, limited CPU, memory and time. Filtering source code (e.g. RestrictedPython) is *not* a security boundary on its own; isolation has to happen at the process or container level.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-034 | P1 strategies shall be declarative (templates and rules interpreted by Ubik). Ubik shall execute no user-supplied code in P1. | Must | P1 | `04` defines the rule model. |
| SEC-035 | Custom Python strategies, including LLM-generated ones, shall run in a separate sandboxed process or container with: no network, no environment secrets, a read-only filesystem apart from a scratch directory, CPU/memory/wall-clock limits, and syscall filtering (gVisor or seccomp). Imports come only from a vetted allow-list, and strategies cannot install packages. The only interface is market data in and order intents out. | Must | P2 | T5, T9, slopsquatting. |
| SEC-036 | Strategy versions shall be content-hashed and immutable, and every order records the hash. Only versions that passed the backtest and paper gates in `04`/`05` may be promoted to live. No hot-editing of live code. Imported third-party strategies are flagged "untrusted" and shown in readable form, with a scam warning. | Must | P3 (flagging P2) | Accountability; SentinelLABS lesson. |

### 6.6 AI assistant security

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| SEC-037 | The assistant shall have no access to secrets. Its context shall be built from a field allow-list, with tests proving no credential can reach a prompt. | Must | P2 | — |
| SEC-038 | The assistant's tools shall be read-only. Trading output is a *proposal* rendered in the standard UI, confirmed by the user, and then subject to SEC-002. The assistant cannot arm trading, change limits, manage keys, or change its own permissions. | Must | P2 | T9; AIXBT lesson. Labelling content as untrusted (SEC-039) is not enough on its own, so this capability limit is the real protection. |
| SEC-039 | External content (news, social posts, token metadata, web pages, imported strategy comments) shall be passed to the model labelled as untrusted data. It shall not be written to persistent assistant memory without user confirmation. | Must | P2 | Memory-injection research. |
| SEC-040 | The LLM key shall have a spending cap and rate limits. Prompts, tool calls and proposals are logged (redacted), with a "what the AI saw" view. | Should | P2 | Cost abuse; transparency. |

## 7. Phase-appropriate security bar

| Area | P1 Paper Trader | P2 Analyst | P3 Live (real money) | P4 Multi-user / Advanced |
|---|---|---|---|---|
| Exchange keys | None (public data); optional read-only | Same | Trade-only, verified, IP-locked, envelope-encrypted, in a sub-account | Per-tenant KEKs in a KMS/HSM; tenant-isolation tests |
| Login | Required; passkey or password + TOTP; MFA before any remote exposure | Same | Phishing-resistant factor for arming; step-up; cooling-off | SSO; admin separation; fraud monitoring |
| Network | Private by default | Same | Static egress IP; hardened host | Production cloud, WAF, DDoS protection, segmentation |
| Code execution | Declarative strategies only | Sandboxed custom and LLM-generated code | Only immutable, promoted versions | Hostile-tenant isolation (gVisor/Firecracker microVMs) |
| Supply chain and SDLC | Lockfiles, scanning, dependency review | Same | Signed releases, SBOM, ASVS L2, pen test | Continuous testing, bug bounty |
| Logs and backups | Audit log; encrypted backups | Same | Hash-chain checkpoints; tested restores; runbooks | SIEM, 24/7 alerting |
| Assurance | Self-review | Self-review | External pen test | **SOC 2 Type II** (independent audit of security controls) or ISO 27001 before onboarding paying users |

## 8. Legal and regulatory landscape (⚖ Lawyer for every jurisdiction-specific conclusion)

### 8.1 The core distinction: personal tool vs service to others

| Level | What Ubik does | Typical exposure |
|---|---|---|
| L0 | Owner trades **own money on own exchange accounts** | Generally no financial licence. Exchange terms, tax and market-abuse law still apply. |
| L1 | Owner **publishes source code** for others to self-host (no hosting, keys or custody) | Usually low. Marketing claims, disclaimers and data licences matter. |
| L2 | **Hosted SaaS**: users bring their own keys and strategies | Holding trade-capable keys and routing orders for others may look like reception/transmission or execution of orders (MiCA), "arranging deals" (UK), or "control" under some US state licences. ⚖ |
| L3 | Ubik **provides signals, recommended strategies or copy trading**, or markets templates as profitable | **Advice** or **portfolio management** under MiCA; investment advice (US, if securities are involved); CTA rules for futures (US); financial-promotion rules (UK) |
| L4–L5 | **Managing others' money**; custody, exchange or matching | Full licensing; fund and commodity-pool rules. Avoid. |

"Let my friend use my bot with their account" can move the owner from L0 to L2–L4 without them noticing.

### 8.2 European Union

- **MiCA** (the EU's Markets in Crypto-Assets Regulation) has applied to crypto-asset service providers (**CASPs**) since 2024-12-30. Grandfathering for existing firms ended **no later than 2026-07-01**, earlier in several member states. Only authorised CASPs may now serve EU clients ([Elvinger Hoss](https://elvingerhoss.lu/insights/publications/mica-end-transitional-period-1-july-2026), [KPMG](https://kpmg.com/cy/en/insights/risk-and-regulation/mica-the-end-of-the-transitional-period-for-casps.html)). An EU-resident owner should only connect Ubik to exchanges listed on ESMA's CASP register.
- **USDT is generally unavailable to EEA users.** Major exchanges delisted non-MiCA stablecoin pairs around March 2025; USDC and EUR pairs remain ([Binance notice](https://forum.firo.org/t/binance-will-delist-non-mica-compliant-stablecoin-trading-pairs-for-eea-users-on-2025-03-31/3919), [DL News on Kraken](https://www.dlnews.com/articles/markets/kraken-delists-tether-usdt-in-europe-as-mica-rules-take-hold)). **This adjusts the working assumption of "vs USDT/USD":** quote currencies must be configurable by jurisdiction (CMP-004).
- **Copy and auto-trading** are not defined in MiCA. ESMA (April 2025) says they must be classified as advice or portfolio management using the MiFID II precedent ([Conventus Law](https://conventuslaw.com/report/eu-update-on-mica-implementation-implications-for-copy-trading-services/), [ESMA briefing](https://www.esma.europa.eu/sites/default/files/2023-03/ESMA35-42-1428_Supervisory_Briefing_on_Copy_Trading.pdf)).
- **Market abuse:** MiCA Art. 91 bars *any person*, including private individuals, from placing orders that give false signals, secure artificial prices, or disrupt trading ([ESMA rulebook](https://www.esma.europa.eu/publications-and-data/interactive-single-rulebook/mica/article-91-prohibition-market-manipulation)).
- **AI Act Art. 50:** the duty to tell people they're talking to an AI has applied since 2026-08-02 and was not delayed by the 2026 Digital Omnibus ([CSA](https://labs.cloudsecurityalliance.org/research/csa-research-note-eu-ai-act-article-50-transparency-20260729/)). It is relevant if Ubik is offered to others.

### 8.3 United States

- **There is no market-structure statute yet.** The **CLARITY Act** passed the House 294–134 (July 2025), but **Senate cloture failed 49–50 on 2026-09-15**. A motion to reconsider keeps it technically alive ([DeFi Rate](https://defirate.com/clarity-act-fact-sheet/), [Crypto Times](https://www.cryptotimes.io/2026/09/20/clarity-act-fails-49-50-in-us-senate-as-sec-cftc-move-ahead-on-crypto-rules-within-48-hours/)). The **GENIUS Act** (stablecoins) has been law since 2025-07-18.
- **Agencies are acting by rulemaking instead:** the SEC proposed "Regulation Crypto Assets" (comments due 2026-10-20), and a CFTC crypto-markets pre-rule is at OIRA, the White House office that reviews draft rules (same source). Revisit before P3.
- **Market conduct:** the CFTC has anti-fraud and anti-manipulation authority over spot crypto commodities (CEA §6(c)(1), Rule 180.1).
- **Advice:** impersonal publications may fall under the Advisers Act **publisher exclusion** (*Lowe v. SEC*). A 2024 decision explicitly distinguished **auto-trading** services that execute trades for subscribers ([NLR](https://natlawreview.com/article/judge-dismisses-case-against-seeking-alpha-implications-publishers-financial)). Futures (P4) raise CTA/CPO questions. ⚖
- **State licensing:** NY BitLicense, and California's DFAL licence requirement since 2026-07-01 ([Jenner](https://www.jenner.com/en/news-insights/client-alerts/its-april-do-you-know-where-your-california-digital-assets-license-is)). Relevant at L2+.

### 8.4 United Kingdom and elsewhere

- **UK:** the FCA's final crypto rules were published on **2026-06-30** (PS26/9–13). They include a crypto market-abuse regime and the Consumer Duty (the FCA's requirement that firms deliver good outcomes for retail customers). Applications opened 2026-09-30, and the regime starts **2027-10-25** ([A&O Shearman](https://www.aoshearman.com/en/insights/ao-shearman-on-fintech-and-digital-assets/final-rules-for-new-uk-crypto-regime)). Crypto financial-promotion rules (risk warnings, 24 h cooling-off, appropriateness checks) have applied since 2023-10-08 to anyone marketing crypto to UK consumers.
- **Other jurisdictions** vary widely; some restrict crypto or particular exchanges outright. **The owner's jurisdiction is the biggest unknown in this document.**

### 8.5 Exchange terms, market data, market conduct

- **Exchange terms:** exchanges can cut off API access at their discretion, and some APIs exclude uses such as HFT or arbitrage (e.g. [Binance Convert](https://developers.binance.com/en/docs/products/convert/Introduction)). Using a VPN, or a VPS in another country, to get around geographic restrictions typically breaches the terms and risks frozen funds ([CoinLedger](https://coinledger.io/learn/are-vpns-legal-for-crypto)).
- **Data licences:** personal use is usually fine; redistribution usually isn't. CoinGecko requires attribution, and an Enterprise plan for redistribution ([CoinGecko](https://www.coingecko.com/en/api/pricing)). Binance's bulk historical data is licensed **CC BY-NC-SA 4.0**, which means non-commercial use only ([terms](https://data.binance.vision/Binance_Vision-Terms_of_Use.pdf)). That is fine for personal backtests, but a problem for a commercial product.
- **Never do these:** *wash trading* (trading with yourself, including across your own accounts, to fake volume), *spoofing or layering* (placing orders you intend to cancel, to move the price), *pump-and-dump*. All are illegal in most places. Don't confuse wash *trading* with a wash *sale*, which is a tax rule about selling at a loss and re-buying.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| CMP-001 | Ubik P1–P3 shall be scoped and documented as a single-owner tool trading the owner's own funds on the owner's own accounts. Multi-user, signal, shared-strategy and copy-trading features stay disabled behind a compliance flag. | Must | P1 | Stays at L0. |
| CMP-002 | A qualified lawyer shall review before (a) the first live trade (jurisdiction, tax and exchange-eligibility check) and (b) any distribution at L1–L4. No signals, copy trading, managed strategies, or trading on accounts the user doesn't own, until (b) sign-off and any licences are in place. | Must | P3 / P4 | ⚖ |
| CMP-003 | Ubik shall keep an **exchange compliance register**. For each venue: terms version and date reviewed, whether API/automated trading is allowed, rate limits, restricted jurisdictions, data-use terms, licence status (e.g. MiCA), and permission-endpoint support. Reviewed every 6 months and whenever an exchange notifies a change. | Must | P3 (Should P1) | — |
| CMP-004 | Ubik shall store a **jurisdiction profile** (country, plus state where relevant, and tax residence). It uses this to warn about restricted exchanges, hide unavailable pairs (e.g. non-MiCA stablecoins for EEA users), and set tax-export defaults. | Must | P1 | The owner's jurisdiction is unknown. |
| CMP-005 | Ubik shall use only official APIs, respect documented rate limits (with backoff), and not scrape web endpoints. | Must | P1 | Exchange terms. |
| CMP-006 | Ubik shall contain no geo-circumvention features (VPN or proxy rotation). It shall warn when the server's egress country differs from the owner's declared country for an exchange that restricts by location. | Must | P1 | Fund-freeze risk. |
| CMP-007 | Ubik shall keep a **data-source registry** (licence, permitted personal/commercial/redistribution use, attribution) and display the required attributions. Every source must be re-cleared or replaced before any multi-user or commercial release. | Must | P1 / P4 | `06` picks sources. |
| CMP-008 | Ubik shall prevent self-matching by checking for crossing its own resting orders across all accounts it controls, and shall use exchange **self-trade prevention** where available. Paper mode simulates the same check. | Must | P1 (paper), P3 | Wash-trading avoidance. |
| CMP-009 | Ubik shall ship no templates designed to spoof, layer, wash trade or pump. The strategy validator flags high cancel-to-fill ratios and rapid placed-then-cancelled orders away from the market, and per-strategy order and cancel rate limits apply. Neither Ubik nor its assistant auto-posts public market commentary. | Must | P3 | MiCA Art. 91, CFTC, UK market-abuse regime. |
| CMP-010 | Market making, cross-exchange arbitrage and derivatives/leverage shall each pass a separate legal and terms-of-service review before being enabled. | Must | P4 | ⚖ |

## 9. Tax and record-keeping

In many countries **every disposal is taxable**. That includes a sale to fiat, a crypto-to-crypto swap, and sometimes a fee paid in a token such as BNB. **Cost basis** is what you paid, including fees; gain = proceeds − cost basis. The rules for matching sales to purchases differ by country: FIFO, average cost, or the UK's same-day / 30-day / pool rules ([HMRC CRYPTO22200](https://gov.uk/hmrc-internal-manuals/cryptoassets-manual/crypto22200)). The US requires **per-wallet or per-account** basis tracking from 2025 ([Rev. Proc. 2024-28](https://irs.gov/pub/irs-drop/rp-24-28.pdf)).

**Exchanges now report to tax authorities:**
- **US Form 1099-DA:** gross proceeds for 2025 sales, plus cost basis for "covered" assets acquired on or after 2026-01-01 ([IRS](https://www.irs.gov/instructions/i1099da)). Offshore exchanges may not issue one, but the tax is still owed.
- **EU DAC8 and OECD CARF:** data collection started 2026-01-01, first reports are due in 2027, and the EU-level exchange happens by 2027-09-30. The UK implements CARF via SI 2025/744, with the first report due 2027-05-31. Around 50 jurisdictions are committed to 2027 exchanges; the US is targeting 2029 ([FinTelegram](https://fintelegram.com/dac8-carf-2026-crypto-tax-transparency-switch-is-on/), [CoinTracker](https://cointracker.io/blog/carf-compliance-guide-for-crypto-platforms)).
- **What this means for Ubik:** authorities see what the exchange saw. Ubik's job is complete, reconcilable records and clean exports. It is **not** the authoritative tax calculator.

**Export formats** (re-verify before P3):
- **Koinly Universal CSV:** Date in UTC (`YYYY-MM-DD HH:mm:ss`), Sent and Received Amount/Currency, Fee Amount/Currency, optional Net Worth, Label, Description, TxHash ([Koinly](https://support.koinly.io/en/articles/9489976-how-to-create-a-custom-csv-file-with-your-data)).
- **CoinTracker CSV:** Date (`MM/DD/YYYY HH:MM:SS`, UTC), Received and Sent Quantity/Currency, Fee Amount/Currency, Tag ([CoinTracker](https://support.cointracker.io/hc/en-us/articles/4413071299729)).
- **Generic Ubik CSV/JSON** containing every ledger field. Tax tools can usually also import straight from exchange APIs; Ubik's export is for reconciliation and for attributing trades to strategies.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| CMP-011 | Ubik shall keep an append-only **trade ledger**. Each entry records: exchange; account/sub-account; exchange trade ID and order ID; client order ID; exchange and local timestamps (UTC); pair; side; quantity; price; gross amount; fee amount and fee asset; fiat value at execution with its rate source; strategy ID and hash; mode (paper or live). Corrections are written as new reversing entries, never as edits. | Must | P1 (paper), P3 | Tax evidence; SEC-030. |
| CMP-012 | Paper and live ledgers shall be stored separately (separate schemas, with a database constraint preventing mixing). Tax exports include **live only** by default. Any paper export is watermarked "SIMULATED — NOT REAL TRADES" in the file name, header and every row. | Must | P1 | Honesty; prevents filing simulated trades. |
| CMP-013 | Ubik shall track lots per account/sub-account, record fees paid in non-quote assets as separate disposals where applicable, and import deposits, withdrawals and transfers (read-only) so basis is complete. | Must | P3 | US per-wallet rule; BNB-style fees. |
| CMP-014 | Ubik shall export generic CSV/JSON (full fidelity), Koinly Universal CSV and CoinTracker CSV. Mappings are versioned and validated against a sample import before each release. Any in-app realized P&L uses a configurable method (FIFO, HIFO, average cost, UK pooling) and is labelled "estimate — not tax advice". | Must | P3 (generic export Should P1) | — |
| CMP-015 | Ubik shall reconcile its ledger daily against exchange history and balances, and produce a year-end report the owner can compare with 1099-DA, DAC8 or CARF-based statements. | Must | P3 | `05` covers operational reconciliation. |
| CMP-016 | Live records are retained for at least 10 years by default (configurable, but not below the jurisdiction minimum). Deletion requests warn about tax-retention duties. | Must | P3 | ⚖ / tax adviser. |

## 10. Disclaimers and responsible use

The CFTC Rule 4.41 hypothetical-performance legend ("hypothetical or simulated performance results have certain inherent limitations…") is a good model for wording.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| CMP-017 | First run shall require acknowledgement that Ubik is an educational and analytical tool, **not financial, investment, tax or legal advice**, and that the user is responsible for their decisions. The acceptance is stored with version and timestamp. | Must | P1 | ⚖ wording before any distribution. |
| CMP-018 | Every backtest or paper-performance number, chart, report and export shall carry an adjacent **hypothetical-performance disclaimer**: not real trading, costs and liquidity are imperfectly modelled, past or simulated results don't predict the future, and overfitting is a risk. | Must | P1 | Overfitting controls are in `04`. |
| CMP-019 | UI copy, docs and templates shall be checked for hype terms ("guaranteed", "risk-free", "passive income", "can't lose"). Performance is never shown without max drawdown and costs. | Must | P1 | `07` owns copy. |
| CMP-020 | Strategy templates shall be labelled "educational examples, not recommendations", with no expected-return marketing. | Must | P1 | Also keeps Ubik away from L3. |
| CMP-021 | The AI assistant shall identify itself as an AI, state that it can be wrong and is not advice, never promise returns, and show sources and uncertainty for market claims. | Must | P2 | EU AI Act Art. 50 if offered to others. |
| CMP-022 | The **live-arming flow** shall show a plain warning ("you can lose all the money you allocate to Ubik"). It then requires: a short comprehension quiz; display of the active limits, maximum loss and kill switch; confirmation of legal age, jurisdiction and exchange eligibility; and the SEC-018 cooling-off period. All acknowledgements are stored. | Must | P3 | Staged capital rollout is in `05`. |
| CMP-023 | If Ubik is ever offered to others, it shall enforce minimum age (18+ or the local age of majority) and exclude restricted jurisdictions. Identity verification (KYC) stays with the exchanges unless counsel advises otherwise. | Must | P4 | ⚖ |

## 11. Privacy

**Data Ubik stores:** email and username; password hash and passkey public keys; encrypted MFA secrets; key metadata and encrypted secrets; balances, positions and trades (sensitive financial data); strategies; AI chat history; IP addresses and devices in audit logs; backups.

**What goes to an LLM provider:** only the fields on the allow-list. By default that is market data, indicator values and the user's question. Portfolio values are sent only if the owner opts in. Credentials, keys, email and account IDs are never sent. Prefer settings with no training on API data and short or zero retention, and keep a local-model option.

**GDPR:** purely personal use by an individual is generally outside GDPR (the "household exemption"). P4 multi-user changes that.

| ID | Requirement | Priority | Phase | Rationale / notes |
|---|---|---|---|---|
| CMP-024 | Ubik shall keep a data inventory, collect only what features need, and send **no telemetry by default**. Crash reports are opt-in and scrubbed. | Must | P1 | Local-first. |
| CMP-025 | The LLM data policy shall be enforced in code: a field allow-list, portfolio data opt-in, the provider and model visible to the user, and local-model support. | Must | P2 | — |
| CMP-026 | Ubik shall offer full machine-readable data export and account deletion, with tax-retained records explained and handled separately. | Should | P2 (Must P4) | — |
| CMP-027 | Any multi-user release shall include a privacy programme: privacy notice, lawful basis, processor agreements (cloud, LLM, email), record of processing, DPIA (data-protection impact assessment) for financial data and AI, breach notification within 72 h (GDPR), international-transfer safeguards, and coverage of UK GDPR and US state privacy laws. SOC 2 Type II (or ISO 27001) readiness is needed before onboarding paying users. | Must | P4 | ⚖ |

---

## Dependencies on other docs

- **`05`**: risk limits (SEC-002/006), kill-switch semantics (SEC-005), staged capital and reconciliation (CMP-015/022).
- **`06`**: placement of the gateway and sandbox, KMS/secret-store choice, per-exchange permission endpoints, RPO/RTO, static egress IP, data-source licences (CMP-007).
- **`04`**: declarative strategy model (SEC-034), promotion gates (SEC-036), overfitting disclosures (CMP-018).
- **`07`**: arming and cooling-off UX, disclaimer placement, "Ubik will never ask for…" onboarding, glossary (wash trading vs wash sale; API permissions).
- **`03`**: jurisdiction-aware pairs (no USDT for EEA users); news and sentiment feeds are untrusted LLM input.

## Open questions for the owner

1. **Which country (and US state, if applicable) do you live in and pay tax in?** This drives allowed exchanges, available pairs, tax rules and export defaults.
2. **Will anyone else ever use Ubik?** For example: open-sourcing it, friends using it, selling strategies or signals, or a SaaS. When?
3. **Hosting:** home machine or VPS? Can you get a **static IP** for P3? Which country would the server be in?
4. **Which exchanges** do you already have verified accounts on? Do they offer sub-accounts?
5. **After a reboot**, should Ubik resume trading on its own (key held in the OS keyring or a KMS), or wait for you to enter a passphrase (safer, but trading stops until you do)?
6. Do you have passkeys or a **hardware security key** (e.g. YubiKey)?
7. Which **tax tool or accountant** do you use?
8. **LLM provider:** which one? Are you willing to share portfolio values with it (opt-in), or would you prefer a local model?
9. Is there budget for a **one-time legal and tax consult** before P3 and before any distribution?

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Stolen trade-only key abused (pump-and-dump through your account) | Loss of the sub-account's funds | IP and symbol allow-lists, sub-account, alerts, kill switch |
| Scope creep from personal tool to regulated service ("just let my friend use it") | Unlicensed activity; personal liability | CMP-001/002 flag and legal gate |
| Rules shift mid-build (CLARITY pending; UK regime starts Oct 2027; SEC/CFTC rulemaking) | Rework | Review this doc at each phase gate; keep the CMP-003 register current |
| Exchange terms violation (location, rate limits, data use) | Frozen account; lost data access | CMP-003/005/006/007 |
| Incomplete tax records, or paper trades mixed with live | Wrong filings; penalties | CMP-011–016 |
| Supply-chain compromise | Key theft; rogue orders | SEC-025–028 |
| Prompt injection or over-trust in the AI | Bad decisions; data leak | SEC-037–040, CMP-021 |
| Self-hosting misconfiguration | Account takeover | SEC-021 secure defaults |
| Security friction makes beginners switch off safeguards | Guardrails bypassed | Explain *why* in plain words (`07`); turning a guardrail off needs step-up and is logged |
| Exchange hack or insolvency (e.g. Bitget 2026) | Loss of funds held on the exchange | Minimal balances on exchanges; diversify venues (`05`) |
