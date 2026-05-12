# Enclz — Research

> Session: April 23, 2026. Sources: web search, Colosseum Copilot (5,400+ hackathon projects).
> Last updated: 2026-05-13. Competitor Map refresh added Turnkey, Zerion, Alchemy.

---

## Validation Sprint

### Demand Signals

| Signal | Strength | Source |
|---|---|---|
| $45M AI trading agent breach (2026) — on-chain enforcement would have capped blast radius | Strong | [KuCoin security report](https://www.kucoin.com/blog/en-ai-trading-agent-vulnerability-2026-how-a-45m-crypto-security-breach-exposed-protocol-risks) |
| CertiK documented OpenClaw AI agent draining wallets via malicious skills | Strong | [CoinTelegraph / CertiK](https://cointelegraph.com/news/ai-agent-openclaw-security-risk-certik) |
| 88% of orgs using AI agents reported confirmed or suspected security incident | Strong | [Gravitee — State of AI Agent Security](https://www.gravitee.io/state-of-ai-agent-security) |
| x402 on Solana: 150M+ agent transactions, $50M+ volume since May 2025 | Strong | [ainvest.com](https://www.ainvest.com/news/solana-ai-agent-payments-50m-flow-test-network-liquidity-2603/) |
| Solana Foundation exec: AI agents projected to drive 99% of on-chain transactions within 2 years | Strong | [CryptoBriefing](https://cryptobriefing.com/autonomous-blockchain-transactions-growth/) |
| Mastercard partnered with lobster.cash (Crossmint) for agent payments | Moderate | [PRNewswire, April 2026](https://www.prnewswire.com/news-releases/lobstercash-partners-with-mastercard-to-enable-secure-ai-agent-payments-for-all-existing-card-holders-302743740.html) |
| Coinbase AgentKit, Privy, Openfort, Turnkey, Trust Wallet all launched agent wallet products 2025–2026 | Moderate | [Openfort — Best Agent Wallets for Developers](https://www.openfort.io/blog/best-agent-wallets-for-developers) |
| Alchemy added agent wallets to its CLI (Privy custody + session scoping) — 2026-05-08 | Strong | [Alchemy blog — Agent Wallets in Alchemy CLI](https://www.alchemy.com/blog/agent-wallets-alchemy-cli) |
| Zerion shipped open-source CLI with scoped policies for autonomous agents — running on 60+ EVM chains plus Solana | Moderate | [zeriontech/zerion-ai](https://github.com/zeriontech/zerion-ai) |
| Turnkey publishes detailed competitor analysis (analyst-grade dossier) — wallet-infra category now mature enough to warrant comparable third-party teardowns | Moderate | Internal dossier, 2026-05-10 (see `dossier-turnkey-competitive-analysis.md`) |
| CoinDesk published "AI agents set to power crypto payments but a hidden flaw could expose wallets" | Moderate | [CoinDesk, April 2026](https://www.coindesk.com/tech/2026/04/13/ai-agents-are-set-to-power-crypto-payments-but-a-hidden-flaw-could-expose-wallets) |

**Demand score: 3/3 (strong).**

### Crypto Necessity

Remove blockchain → limits live in a backend database, bypassable if backend is compromised or operator turns malicious. On-chain enforcement is load-bearing: the smart contract is the policy, independently verifiable on Solana Explorer. Blockchain is not ornamental.

### Go / No-Go

**Verdict: GO. Confidence: 0.65 (medium).**

4 of 5 go-criteria met: demand ✓, technical feasibility ✓, time to MVP (borderline) ✓, crypto necessity ✓, unfair advantage (partial).

No hard no-go triggers. Openfort is a real on-chain enforcement competitor — differentiation required but achievable.

---

## Competitor Map

| Product | On-chain enforcement | Solana-native | No SDK | Agent-first DX | Simulate |
|---|---|---|---|---|---|
| **Enclz** | ✓ Anchor | ✓ Native | ✓ REST + MCP | ✓ SKILL.md + MCP | ✓ |
| Openfort | ✓ ERC-4337/EIP-7702 | ~ EVM-first | ✗ SDK required | ~ Generic | ✗ |
| Turnkey | ✗ TEE-enforced policy, off-chain | ~ Multi-chain (signs Solana) | ✗ Stamping SDKs required | ~ Generic embedded-wallet | ✗ |
| Alchemy CLI agent wallets | ✗ Session + expiry only, no spend cap | ~ Multi-chain (Privy custody) | ~ CLI + dashboard | ~ Coding-agent focused | ✗ |
| lobster.cash (Crossmint) | ✗ Off-chain | ✓ + card rails | ✓ | ~ Card + USDC focus | ✗ |
| Coinbase AgentKit | ✗ App layer | ~ Recently added | ✗ SDK required | ~ Generic | ✗ |
| Privy Server Wallets | ✗ Off-chain | ~ Multi-chain | ✗ SDK required | ~ Generic | ✗ |
| Zerion CLI | ✗ Client-side scoped policies | ~ 60+ EVM + Solana | ~ Fork-and-build CLI | ✓ Agent-runtime native | ~ Tx simulation built in |
| Trust Wallet Agent Kit | ✗ Unknown | ~ Multi-chain | ✗ SDK required | ✗ | ✗ |

**Primary threats — three distinct angles:**

1. *Openfort* — only other product with truly on-chain enforceable policies, but EVM-first (ERC-4337 / EIP-7702), SDK-required, and no agent-specific DX. We compete on Solana-native + no-SDK distribution surface.
2. *Turnkey* — runaway distribution among AI-agent and trading-bot customers (the same accounts we want to reach), strong policy DSL, but off-chain TEE-bounded and operationally custodial. We compete on trust model and per-signature cost.
3. *Alchemy CLI agent wallets* — newest entrant (2026-05-08), enormous installed dev base, but policy model is session + expiry only with no spend cap. We compete on enforcement depth.

Enclz must own: *Solana-native + no-SDK + SKILL.md context injection + MCP server + simulation endpoint + trustless on-chain policy with per-tx / daily / hourly ceilings + TTL-bounded whitelist*. The full set is the moat; any individual axis can be matched by one of the three above.

### Notes on Key Competitors

**[Openfort](https://www.openfort.io/blog/agent-wallet-solutions-for-developers):** Only other product with on-chain enforceable policies + Solana support. Uses session keys encoding contract/method/spend cap/time window. Open-source signer (OpenSigner) is self-hostable — trust advantage. Multi-chain via one SDK. Does not have simulation endpoint or agent-context injection artifacts.

**[lobster.cash (Crossmint)](https://www.prnewswire.com/news-releases/lobstercash-partners-with-mastercard-to-enable-secure-ai-agent-payments-for-all-existing-card-holders-302743740.html):** Solana-native, Mastercard partnership (April 2026). Dual-rail: USDC wallet + virtual card with limits. "Verifiable Intent" framework co-developed with Google — cryptographically ties each agent transaction to user's approval. Off-chain limits only. Strong distribution via Mastercard.

**[Coinbase AgentKit](https://www.coinbase.com/en-gb/developer-platform/products/agentkit):** Framework-agnostic, wallet-agnostic. No built-in spending limits — application layer. Recently added Solana. Largest developer mindshare.

**[Turnkey](https://www.turnkey.com):** TEE-based custodial signer running inside AWS Nitro Enclaves with reproducible-build attestation receipts. Strong, well-documented policy engine (JSON + CEL-style DSL) with typed views over EVM / Solana / Bitcoin / Tron transactions. Sells two SKUs: Embedded Wallets and Company Wallets, plus encryption-key storage as a separate product. Outsized share of Telegram trading-bot customers (BananaGun, Bonkbot, Maestro, BullpenFi, Unibot) and named AI-agent platforms (PassApp, Spectral Labs). $52.5M raised across 3 rounds, Series B led by Bain Capital Crypto (June 2025), ex-Coinbase Custody founding team. *Where they fall short for Enclz's wedge:* policy enforcement is off-chain (a signed "Ruling" from a Policy Engine enclave consumed by a Signer enclave). Operationally custodial — Turnkey holds the quorum that can re-provision enclaves, so a subpoena to Turnkey is the failure mode. AWS Nitro is the single TEE substrate. Pricing is per-signature ($0.05 at Pro tier, $0.0015 floor at Enterprise) — meaningful line item for AI-agent and trading-bot workloads at 1M+ sig/month volumes. *Strategic frame:* Turnkey's customers are our most concentrated outbound list (see `icp-leads.md` 2026-05-12 entry); pitch is same enforcement, on chain, zero per-signature cost.

**[Alchemy CLI agent wallets](https://www.alchemy.com/blog/agent-wallets-alchemy-cli)** (launched 2026-05-08): Agent wallets exposed through the Alchemy CLI for coding-agent runtimes (Claude Code, Cursor, etc.). Three commands take a developer from install to a connected wallet (`npm i -g @alchemy/cli`, `alchemy auth`, `alchemy wallet connect`). Privy holds the keys; Alchemy gates access via "scoped, time-bound" sessions. Multi-chain via Alchemy's existing RPC stack (100+ chains, Solana included). *Where they fall short for Enclz's wedge:* policy is *capability + session expiry* only — no per-tx ceiling, no daily cap, no hourly frequency cap, no recipient whitelist with TTL. Architecture is custodial-by-construction (Privy custody + Alchemy session validation). *Strategic frame:* huge installed distribution (every web3 dev with an Alchemy account), thinner enforcement model. Our counter is *what's enforced*, not *speed of integration*. Concrete pitch line: "Alchemy + Privy give agents a custodial wallet with session-based scoping. We give them a non-custodial PDA with on-chain spend policy. Different guarantees."

**[Zerion](https://zerion.io):** Multi-chain self-custodial wallet (60+ EVM chains plus Solana). Their Zerion API powers wallet/portfolio data for 200+ teams (Uniswap, Privy, Backpack, Kraken). New product: open-source [Zerion CLI](https://github.com/zeriontech/zerion-ai) for agent runtimes, with built-in scoped policies — "lock to a chain, set spend limits, define expiry windows, block certain actions." Same conceptual vocabulary as Enclz. *Where they fall short for Enclz's wedge:* policy enforcement is client-side in the CLI. A compromised CLI runtime, a leaked key, or a prompt-injected model still produces signed transactions outside policy. The CLI is the gate; the chain doesn't care. *Strategic frame:* dual-role — *partial competitor* (overlapping conceptual category) and *partnership target* (open-source CLI; we could submit a PR adding an "Enclz policy provider" alongside their existing scoped-policy system, gaining distribution into their 200+ team developer surface). Treat as we treat Pay.sh: parallel infrastructure, not zero-sum.

---

## Colosseum Hackathon Landscape

*Data from Colosseum Copilot, 5,400+ projects searched.*

### Most Similar Projects

| Project | Hackathon | Prize | Core approach | Gap vs Enclz |
|---|---|---|---|---|
| **Latinum Agentic Commerce** | Breakout 2025 | **1st AI — $25k** | MCP-compatible wallet, agents pay for services | No whitelist/spend limits, no on-chain enforcement |
| **Mercantill** | Cypherpunk 2025 | **4th Stablecoins — $10k** | Squads Grid-based banking infra for AI agents: audit trails, programmable spending limits, multi-sig controls | Wraps Squads multisig (coordination overhead, third-party stack) instead of native Anchor PDA policy; no TTL + amount-capped per-recipient ceiling; no swap/yield extension |
| **Agent-Cred** | Cypherpunk 2025 | None | AI agent payment infra, hotkey/coldkey arch | No on-chain enforcement, SDK required |
| **AgentVault** | Cypherpunk 2025 | None | Non-custodial trading agent control plane, idempotency keys, kill-switch | Trading-focused, application-layer controls |
| **Blockpal Smart Delegation** | Breakout 2025 | None | Delegation + permission guardrails for agents | Delegation model not policy enforcement |
| **SMART WALLET** | Radar 2024 | None | PDA-based wallet, spending limits, no private key sharing | User-facing not agent-infrastructure |
| **Project Plutus** | Breakout 2025 | **2nd AI — $20k** | AI agent deployment + management platform | Platform play, not payment enforcement |
| **AI Economy Protocol** | Cypherpunk 2025 | None | Autonomous agent marketplace with payments | Agent-to-agent marketplace, no policy layer |

**Key finding:** No hackathon project in the dataset implements on-chain spend enforcement via a *dedicated Anchor program* with whitelist PDAs + nonce + per-agent policy. Mercantill comes closest pitch-wise but rides on Squads Grid (multisig wrapper); Enclz's native PDA mechanism is novel in this dataset.

### Crowdedness

**Crowdedness score: 88/100 (highly crowded).** "Solana AI Agent Infrastructure" cluster `v1-c14`: **325 projects — the #1 most populated of 30 ML-derived clusters** across the entire 5,404-project corpus. Cluster win rate 14/325 = 4.3% (roughly the hackathon-wide average — contested, not saturated to death). Search similarity scores for Enclz's specific angle (on-chain policy enforcement) are low (0.03–0.06) — the *mechanism* is open within a crowded *category*. The threat is Mercantill: same elevator pitch, Cypherpunk 2025 prizewinner, must be out-differentiated on the native-PDA / TTL + amount-capped per-recipient enforcement / swap-extension axes.

### Winner Gap Analysis

**Winners overindex on (build for these):**
- Oracle primitives (+0.27 lift)
- Capital inefficiency / real financial problems (+0.81 lift)
- Fragmented liquidity problems (+0.22 lift)

**Winners underindex on (avoid):**
- NFTs (−0.66), token-gating (−0.56), tokenized rewards (−1.0)
- Smart contract escrow (−1.0), on-chain verification as a feature (−1.0)
- High platform fees / high barrier to entry as problem framing (−1.0)

**Enclz alignment:** Good. Solves a real financial/security problem, targets developers, on-chain enforcement as primitive. Not consumer-facing.

### Strategic Insight from Hackathon Data

**Latinum won 1st place AI at Breakout with MCP-compatible wallet.** The winning pattern: meet agents where they run (MCP runtimes), minimize integration friction. Enclz's `SKILL.md` + `openapi.json` + MCP server covers all three distribution channels simultaneously. This directly mirrors the approach that won.

---

## Risks

| Risk | Severity | Description |
|---|---|---|
| Openfort competitive convergence | High | If they add agent-first DX (SKILL.md equivalent, simulation, Solana-native UX), differentiation narrows. Window is now. |
| Institutional consolidation | High | Coinbase + Mastercard throwing weight behind competitors. Well-funded player could acquire Openfort or double down on Solana on-chain enforcement. |
| Smart contract audit | Medium | Required before mainnet. $10–40k, 4–8 week lead time. Bug in spend enforcement or nonce logic is catastrophic. |
| Developer adoption friction | Medium | "Survives backend compromise" claim needs a demo that makes it visceral, not just a technical argument. |
| Custody interpretation | Medium | Backend operator keypair signs all transfers. Potential money transmission classification in some jurisdictions. Legal review needed before mainnet. |
| On-chain state drift | Low | Backend mirrors on-chain limit state for pre-flight checks. Needs observability to catch divergence. |

