# Enclz — Requirements (Agent Wallet Edition)

> **Enclz** (enclz.com) — one enclave per agent, one policy ceiling enforced on-chain.

## Vision

Give autonomous AI agents a dedicated Solana wallet with on-chain spend enforcement — so they can pay for things without a developer handing them a real private key.

## Problem

AI agents need to spend money: API calls, micro-services, bounties, tips, on-chain actions. Current options are:

- **Full private key in environment** — catastrophic if agent is compromised, hallucinates, or config leaks
- **Backend-enforced limits** — bypassable if the backend itself is compromised
- **No wallet at all** — agent can't operate autonomously

Enclz takes a fourth path: agents hold a dedicated wallet governed by on-chain spend policy. Limits and whitelist enforcement live in the smart contract — no backend compromise can override them.

## Target Users

### Orchestrator (Person A2)

Developer or researcher running one or more autonomous AI agents.

- Builds agents using LangChain, AutoGen, CrewAI, Eliza, or custom frameworks
- Agents perform real work: research, content, task coordination, API orchestration, on-chain workflows
- Agents need to pay for things: API calls, micro-services, contractor bounties, tipping
- Primary fear: agent hallucinates a large transfer, prompt injection drains wallet, or agent config is exfiltrated exposing a private key
- Manages the fleet from a web dashboard or, for headless setups, from a Node script that performs the same Sign-In-With-Solana ceremony and signs on-chain mutations with a local keypair

### AI Agent (Person B2)

Autonomous software process, not a human.

- Never sees or holds a private key — authenticated via scoped API key only
- Interacts via REST API with structured JSON requests (no natural language)
- Subject to on-chain whitelist and spend limit enforcement
- Receives confirmations via synchronous response or webhook callback

---

## Product Channels

| Channel | Who uses it | Purpose |
|---|---|---|
| **Web App** | Orchestrator (A2) | Group setup, token registry curation, agent provisioning, whitelist management, policy config, fleet dashboard, audit log, x402 budget dashboard |
| **Orchestrator on-chain mutations** | Orchestrator (A2) | Group creation, whitelist add/renew/remove, agent creation, per-agent limit updates, x402 budget approvals (SPL Token `Approve`/`Revoke`) — signed by the orchestrator's Solana wallet (Solflare in the browser; raw keypair in a Node script). Authenticated by Sign-In-With-Solana — wallet ownership is the credential. |
| **Orchestrator REST API** | Orchestrator (A2) | Credential minting (invite codes, API key rotation/revocation, fleet webhook registration), token registry CRUD, x402 payment history + aggregations. All policy / budget mutations remain on-chain. |
| **Agent activation URL** | Orchestrator → AI Agent (B2) | One-shot enclz.com/on&lt;token&gt; URL the orchestrator shares with an agent. GET returns Markdown install instructions; POST single-use, drops a bash CLI containing the agent's credentials. The agent never sees plaintext key handling. |
| **Agent REST API** | AI Agent (B2) | Transfer, swap, deposit, withdraw, balance/limit queries, simulation, webhook registration, x402 payment resolution (`/v1/x402/pay`, `/v1/x402/proxy`) |
| **Agent CLI (`enclz`)** | AI Agent (B2) | Thin curl wrapper installed by the activation URL; first positional arg is the endpoint path, optional JSON body is the request payload, with auto-generated UUID idempotency keys. Includes a dedicated `x402/pay` subcommand. |
| **MCP Server** | AI Agent (B2) | Native MCP integration: 5 mutating tools (`transfer`, `swap`, `deposit`, `withdraw`, `simulate`) + 3 read-only resources (`enclz://balance`, `enclz://limits`, `enclz://history`). Configured with `ENCLZ_API_KEY` + `ENCLZ_API_URL`. |

---

## Core Model

Each agent gets a dedicated wallet on Solana, bound at creation time to exactly one SPL mint chosen from the group's curated token registry. The wallet is governed by a spend policy — daily limit, per-transaction limit, hourly frequency cap — and a whitelist of approved recipient addresses and protocol addresses. These rules are enforced on-chain: no backend configuration or compromise can override them.

External addresses are whitelisted with a **TTL** (expiry timestamp) and an **approved amount cap**. Once the full approved amount is transferred to an address, the whitelist entry is automatically voided on-chain — the orchestrator must explicitly re-approve it. Intra-group agent addresses and protocol addresses (DEX router, lending pools) are permanent and unlimited.

The chain pins outbound paths (`execute_transfer`, `execute_lending_op`) to the bound mint; swaps relax the input-mint constraint but enforce custody via the output ATA's owner, so an agent can accumulate non-bound mints and swap them back to its bound mint, but cannot exfiltrate them. The bound mint is captured into `AgentWallet.mint` at `add_agent` time and is immutable — to repurpose an agent for a different mint, retire it and create a new one.

The orchestrator controls group setup and policy by signing on-chain instructions directly — from the web app via Solflare, or from a Node script holding a local keypair. Both paths use Sign-In-With-Solana for backend auth. The agent interacts only through the agent REST API using a scoped API key — no private key, no signing, no crypto knowledge required. The canonical onboarding shape is a single URL (`https://enclz.com/on<token>`) the orchestrator pastes to the agent; the agent's first `curl ... | bash` writes a thin CLI that wraps the REST surface.

### Token registry

Every group curates a per-group catalogue of SPL mints (`{symbol, decimals, label}`) that agents may bind to. The catalogue is the only place agents receive a `symbol` — there is no global "USDC", and the dashboard renders each agent's amounts in its own bound symbol. The chain owns the per-agent binding (`AgentWallet.mint`); the registry holds metadata only. An agent-creation wizard refuses to advance with an empty registry, and registry DELETE is refused while any agent is bound.

### x402 payment surface

Agents that hit an HTTP 402 Payment Required response have two ways to settle:

- **Orchestrator-delegated** (`POST /v1/x402/pay`) — the orchestrator pre-approves a fixed allowance to the platform delegate via SPL Token `Approve` on their own ATA; the backend partial-signs a `TransferChecked` against that allowance and hands the agent a `follow_up` envelope (`X-PAYMENT` for v1, `PAYMENT-SIGNATURE` for v2) to attach on its retry. The x402 facilitator on the resource server's side signs as fee payer and submits. Budget state lives entirely in SPL Token `delegate / delegatedAmount / amount` — there is no backend budget table.
- **Agent-funded** (`POST /v1/x402/proxy`) — pays from the agent's own wallet via `execute_transfer`. Subject to the agent's whitelist + spend ceiling, exactly like a regular transfer.

---

## User Flows

### Flow A — Group Setup

1. Orchestrator opens the web app (no wallet required to browse or preview) and connects their Solana wallet (browser via Solflare, or headless via a Node script that holds an Ed25519 keypair). Both paths perform Sign-In-With-Solana to obtain a Supabase JWT, then sign the on-chain `initialize_group` instruction directly.
2. A `GroupConfig` is recorded on-chain. If the on-chain instruction fails, the web app shows an error with a retry button — no partial state is persisted.
3. Orchestrator curates the per-group SPL token registry — at minimum, one mint must be registered before agents can be created. The dashboard's "Seed defaults" action bulk-inserts canonical devnet mints; manual entry pastes a mint pubkey, reads decimals via `getMint`, and POSTs to the registry (backend re-verifies decimals).
4. Orchestrator adds agent members — each picks a mint from the registry at creation time, and the chain captures it into `AgentWallet.mint`.
5. Orchestrator configures per-agent policy: whitelist of approved service endpoints, per-tx/daily/hourly limits
6. Orchestrator optionally selects a policy template as a starting point
7. For each external service address added to the whitelist, orchestrator sets a TTL (e.g., 30 days) and an approved amount cap (e.g., $50 total). Intra-group agent addresses are added automatically with no TTL and no cap.
8. Optionally, orchestrator opens the x402 dashboard and signs SPL Token `Approve` instructions delegating the platform delegate a per-currency spending cap for autonomous HTTP-402 payments. No REST call; delegation lives entirely on-chain.

### Flow B — Agent Registration

1. Orchestrator clicks **Add Agent** in the web app (or runs the equivalent Node script) and picks a registry mint. The orchestrator's wallet signs the on-chain `add_agent` instruction; once confirmed, the SPA POSTs `{tx_sig, agent_wallet_pda, token_mint}` to `POST /v1/orchestrator/groups/:group_config_pda/agents`.
2. Backend verifies the on-chain agent belongs to the route's group, asserts `AgentWallet.mint == token_mint`, asserts the mint is in `group_tokens`, then mints a one-time activation token shaped `on<8 alphanumeric>` — shown once, expires in 24 hours.
3. The dashboard renders the activation URL `https://enclz.com/on<token>` and a copy-paste snippet `Activate wallet: https://enclz.com/on<token>`. The orchestrator shares this single line with the agent (chat client, email, system prompt — anywhere).
4. The agent (human or LLM) follows the URL:
   - `GET https://enclz.com/on<token>` returns a Markdown page with install instructions, side-effect free, so the same URL can be opened repeatedly.
   - The install command runs `curl -sSf -X POST https://enclz.com/api/v1/onboard/on<token> | bash`, which atomically redeems the token and writes the agent CLI to `~/.local/bin/enclz` with the API key, `agent_wallet_pda`, and base URL embedded as shell constants. The plaintext API key never lands in the agent's context — only the CLI binary holds it.
   - Subsequent POSTs against the same token return 404. Tokens expire after 24h; orchestrators can re-mint via the dashboard.
5. The agent is now active: `enclz balance`, `enclz transfer '<json>'`, `enclz x402/pay '<challenge>'`, etc. — all subject to on-chain policy.

For headless setups, `POST /v1/register` accepts the same code and returns the API key as JSON (instead of a bash script) for programmatic flows that don't want a shell binary on disk.

If the API key is compromised: orchestrator revokes it in the dashboard and clicks "Rotate key" to atomically revoke + mint a new one, OR uses "Re-invite" to mint a fresh activation URL without rotating the key.

### Flow C — Agent Transfer

1. Agent calls `POST /v1/transfer` with `to`, `amount`, `token`, optional `memo` and `task_id`, and optional idempotency key
2. Backend authenticates via scoped API key, resolves agent's wallet
3. Backend pre-flight checks limits (mirrors on-chain state)
4. Transfer executes — whitelist and spend limits enforced on-chain
5. Response returns `tx_sig`, `status`, remaining daily and hourly headroom
6. If webhook registered: async `transfer.confirmed` event fires to callback URL

If recipient is not whitelisted, the whitelist entry has expired, the approved amount is exhausted, or a spend limit is exceeded: `403` with structured error code, no transaction submitted. On RPC timeout or transient failure: `503` with `retry_after` field — agent retries with same idempotency key.

### Flow D — Agent Swap

1. Agent calls `POST /v1/swap` with `from_token`, `to_token`, `amount`
2. Backend fetches Jupiter quote, executes swap via whitelisted DEX router
3. Response returns `tx_sig`, `received_amount`, `rate`

### Flow E — Deposit / Withdraw (Yield)

Orchestrator whitelists a lending protocol (e.g., Kamino). Agents can then deposit and withdraw.

Deposit:
1. Agent calls `POST /v1/deposit` with `token`, `amount`, optional `protocol` label
2. Backend routes to the whitelisted lending protocol, executes deposit
3. Response returns `tx_sig`, `deposited_amount`, current APY

Withdrawal:
1. Agent calls `POST /v1/withdraw` with `token`, `amount`, optional `protocol` label
2. Backend redeems from the protocol, returns funds to agent wallet
3. Response returns `tx_sig`, `received_amount`, `yield_earned`

### Flow F — Balance, Limits, History

Agent queries current state without executing any transaction:
- `GET /v1/balance` — token balances, daily/hourly headroom
- `GET /v1/limits` — full spend policy, whitelist
- `GET /v1/history` — paginated transaction log

### Flow G — Simulation (Dry Run)

Agent calls `POST /v1/intents/simulate` with the same body it would send to a real intent (transfer, swap, deposit, withdraw). The response indicates whether the live call would succeed under the same enforcement the chain applies, plus a refusal reason and the agent's remaining headroom — no transaction submitted, no fee consumed.

Agents pre-check before committing, especially in limit-sensitive workflows and cost-bounded research runs. Simulation must share the enforcement path of the corresponding live intent so it cannot diverge from the policy actually applied at execution time.

### Flow H — Incoming Payment Notification

Any incoming transfer to an agent wallet triggers a `payment.received` event to the agent's registered webhook URL.

### Flow I — Anomaly Alerts

Orchestrator registers a webhook for policy events across their fleet:
- `policy.limit_threshold` — agent reaches 80% of daily limit
- `policy.limit_exceeded_attempt` — agent request rejected for exceeding limit
- `policy.whitelist_violation` — agent attempted transfer to non-whitelisted address
- `policy.whitelist_expiring` — a whitelist entry TTL expires within 24 hours (fires once)
- `policy.whitelist_amount_threshold` — a whitelist entry has consumed 80% of its approved amount
- `policy.whitelist_voided` — a whitelist entry was auto-voided (approved amount fully drained)

Allows orchestrators to detect runaway agents or injection attacks without polling. Expiry and amount alerts give orchestrators time to re-approve before agents are blocked.

### Flow J — x402 Payment Resolution

1. Agent makes an HTTP request to a paywalled resource (`enclz x402/pay '<verbatim 402 challenge>'` or a direct `POST /v1/x402/pay`).
2. Backend decodes the v1/v2 challenge, verifies the `scheme` is Solana `exact`, looks up the `asset` mint against the group's token registry for decimals, and reads the orchestrator's delegated allowance live from chain.
3. If the platform delegate is approved with sufficient allowance and the budget ATA has sufficient balance, the backend partial-signs a `TransferChecked` against the orchestrator's ATA and returns `follow_up = { url, headers }` where `headers` is a single-entry map containing the x402 payment payload.
4. Agent attaches `headers` on its retry to the resource server. The x402 facilitator on the resource server's side fills the fee-payer signature slot, submits on chain, and returns the resource.
5. The signed offer is recorded in `x402_transactions`; orchestrators see it in the dashboard's history table and aggregations.
6. For agent-funded payments (whitelist-enforced, no orchestrator delegation), agents instead call `POST /v1/x402/proxy` with the parsed challenge — equivalent to a manual `transfer` to the `payTo` address with all the usual on-chain ceilings applying.

If the orchestrator has not approved the delegate, or the allowance is exhausted, the backend returns `402 x402_budget_not_configured | x402_budget_exhausted | insufficient_x402_budget`. The agent surfaces this back to the caller; the orchestrator opens the x402 dashboard and signs an `Approve` to top up.

---

## Security Model

The actual threat model is backend abuse and agent misbehavior, not key theft. On-chain policies address this directly:

| Threat | Protection |
|---|---|
| Backend hacked, attacker attempts to drain funds | Blocked — only whitelisted recipients, capped amounts |
| Backend operator turns malicious | Blocked — same ceiling applies |
| Agent hallucinates a large transfer | Blocked — per-tx limit caps damage; whitelist blocks arbitrary recipients |
| Prompt injection tricks agent into draining wallet | Contained — whitelist + frequency cap + approved amount cap limit blast radius; even a fully exploited agent can only drain the pre-approved amount to pre-approved addresses |
| Agent API key exfiltrated | Contained — API key has no key custody; attacker can only move funds within on-chain policy ceiling |
| Agent enters infinite spend loop | Blocked — hourly frequency cap enforced on-chain |
| Compromised orchestrator account | Partial — limits still apply post-compromise; emergency withdraw available |

### Spend Policy

Each agent wallet has independently configurable limits. Defaults are tight to bound hallucination and loop damage:

- **Daily spend limit** — default $10/day
- **Per-transaction limit** — default $1/tx
- **Hourly frequency cap** — default 5 tx/hour

Orchestrator can raise limits per-agent. Upper bounds are enforced on-chain — no backend call can exceed what the smart contract allows.

### Whitelist

Transfers are only allowed to addresses explicitly approved by the orchestrator. Three categories with different lifecycle rules:

**Intra-group addresses** — other agent wallets within the same group:
- Added automatically when an agent is created
- Permanent, no TTL, no amount cap
- Intra-group transfers always allowed (within per-tx and daily limits)

**External recipient addresses** — wallets the agent is allowed to pay:
- External service addresses (APIs, contractors, bounty recipients)
- Exchange deposit addresses
- Require: **TTL** (expiry timestamp) + **approved amount cap** (total USDC the agent may send to this address)
- On expiry: whitelist entry is void — transfer to this address returns `whitelist_expired`
- On amount exhaustion: whitelist entry is auto-closed on-chain — transfer returns `whitelist_amount_exhausted`
- Orchestrator re-approves by creating a new whitelist entry with fresh TTL and amount

**Protocol addresses** — DeFi integrations:
- DEX swap router (whitelisted at group initialization, permanent)
- Lending protocol pools (e.g., Kamino) — added explicitly by orchestrator, permanent

No address outside the whitelist can receive funds regardless of backend state. TTL and amount exhaustion checks are enforced on-chain in `execute_transfer`.

### Authentication

**Agent API key** — scoped credential issued at registration. Never stored in plaintext (bcrypt hash only). Revocable instantly. Agent never sees or holds the underlying wallet private key.

**Orchestrator auth** — Sign-In-With-Solana (SIWS) for both browser (Solflare) and headless (Node) orchestrators, producing a Supabase JWT. The same JWT authorizes the web app and every `/v1/orchestrator/*` REST endpoint (credential minting, token registry CRUD, x402 history + stats); on-chain mutations (group create, whitelist, agent create, limit updates, x402 budget approvals) are signed directly by the orchestrator's wallet. Wallet ownership IS the credential, and `requireGroupAccess` derives the group from the JWT's wallet pubkey via `["group", wallet_pubkey]` PDA derivation, so the URL's `:group_config_pda` must match for the call to succeed. The repo ships `examples/orchestrator-node.js` as a canonical demonstration of programmatic SIWS — accepts a Solana keypair via `ORCHESTRATOR_SECRET_KEY` (base58 string OR JSON-array byte format from the Solana CLI `id.json`), signs in, creates a group, creates one agent, prints the activation URL.

### Limitations

Enclz does not protect against:
- Orchestrator intentionally draining agent wallets (mitigated by design: orchestrator funds wallets, doesn't withdraw from them)
- Smart contract bugs (mitigated by audit before mainnet)
- Solana network-level issues (outside scope)

---

## Policy Templates

Orchestrators select a template when creating an agent; any field can be overridden.

| Template | per-tx limit | daily limit | hourly cap | Whitelist preset |
|---|---|---|---|---|
| `research-agent` | $0.10 | $1.00 | 10 tx | Known data API endpoints |
| `micro-payment-agent` | $1.00 | $10.00 | 5 tx | Specified at creation |
| `payment-agent` | $10.00 | $100.00 | 20 tx | Exchange addresses + specified |
| `custom` | Orchestrator-defined | Orchestrator-defined | Orchestrator-defined | Fully manual |

Templates are advisory — the orchestrator can override any field. The on-chain policy is what matters.

---

## Agent Integration Resources

Rather than maintaining language-specific SDKs, Enclz ships three integration artifacts, all derived from a single source of truth (`openapi.json`, generated from Fastify route schemas):

**`openapi.json`** — Machine-readable OpenAPI 3.1 spec covering all agent REST endpoints. Generated by `npm run openapi:generate` (boots Fastify in spec-only mode and dumps `app.swagger()`) and validated in CI via `git diff --exit-code`. Consumed by code generators, API clients, and AI assistants. The published `servers:` array contains exactly one entry: `https://enclz.com` (no localhost / staging / `${ENCLZ_API_URL}`-style placeholders).

**`SKILL.md`** — Markdown file designed to be injected into an agent's system prompt or context. Self-contained: intro prose, generated tool table (5 rows), generated resource table (3 rows), generated error-remediation table, idempotency-header note, simulate-first guidance. Sections between marker comments are regenerated from `openapi.json`; intro / idempotency / simulate-first prose is hand-authored. All curl examples use the literal canonical host `https://enclz.com` rather than env-var placeholders so the file is paste-shareable.

**MCP Server (`@enclz/mcp-server`)** — Model Context Protocol server wrapping the Agent REST API over stdio. Exposes 5 mutating tools (`transfer`, `swap`, `deposit`, `withdraw`, `simulate` — bare names, no `enclz_` prefix because MCP clients namespace by server name) and 3 read-only resources (`enclz://balance`, `enclz://limits`, `enclz://history`). Configured with two env vars: `ENCLZ_API_KEY` (issued at registration) and `ENCLZ_API_URL` (default `https://enclz.com`). Compatible with any MCP runtime: Claude Desktop, Cursor, Claude Code, or custom agents built with the MCP SDK.

All three agent-facing surfaces describe the runtime in business terms only. On-chain rejections reach the agent as curated business codes (`whitelist_violation`, `daily_limit_exceeded`, etc.) plus human-readable remediation prose — the program's 6000–6013 discriminants are scrubbed by `agentSafeError`. A CI lint pass (`scripts/lint-agent-artifacts.mjs`) enforces this on `openapi.json`, `SKILL.md`, and the built MCP bundle.

**Activation URL** is the canonical handoff — `https://enclz.com/on<token>` is what the orchestrator copies to the agent. The CLI installed by the activation URL is a thin curl wrapper, not a strongly-typed SDK; agents that prefer a typed surface use the MCP server.

---

## Program Integration Resources

The "no SDK required" claim above describes the **agent integration path** (Agent REST API + MCP server). A separate audience — direct on-chain callers — does need typed program bindings: the Enclz backend itself, programs composing with Enclz via CPI, security researchers, and auditors. For these consumers Enclz ships two distribution channels for the Anchor IDL and TypeScript types.

**`@enclz/sdk`** — npm package re-exporting the Anchor IDL JSON, the `Enclz` TypeScript type, and the program ID. Consumers get `Program<Enclz>` typing in two lines: `npm install @enclz/sdk @coral-xyz/anchor @solana/web3.js`, then `new Program<Enclz>(IDL, provider)`. The package version is single-sourced from `programs/enclz/Cargo.toml` via the IDL `metadata.version`, so the published SDK version always identifies the program version that produced it. `@coral-xyz/anchor` is a peer dependency — the SDK ships only types and IDL JSON, never the Anchor runtime.

**On-chain IDL** — the same IDL is published on-chain on every cluster Enclz is deployed to, so `Program.fetchIdl(programId, provider)` returns a typed program handle without installing any package. Useful for explorers, auditors, and integrators who want to verify the IDL against on-chain bytecode rather than trust an npm tarball.

Neither channel is required for AI agents using the Agent REST API or MCP server — those paths remain SDK-free. The program bindings exist for direct on-chain integration, which is a fundamentally different use case from agent invocation.

---

## Monetization

**Protocol fee** — Enclz deducts a flat 10 basis points (0.1%) from every outbound transfer and swap at execution time. Collected on-chain. No separate billing, no off-chain settlement.

The fee is charged to the sending agent's wallet at execution time, deducted from the transfer amount. It counts against the agent's daily spend limit (like any other transfer).

---

## Web App Features (Orchestrator)

- **Landing page (no wallet required)**: product overview, live devnet demo (transfer blocked by whitelist shown on Solana Explorer), policy template previews. Wallet connection only required to take action.
- Connect Solana wallet, create groups
- **Tokens dashboard** (`/group/:id/tokens`): curate the per-group SPL token catalogue — paste a mint, see decimals auto-resolved via `getMint`, add `{symbol, decimals, label}` to the registry. Per-row delete is enabled only when no agent is bound to the mint. A "Seed defaults" action bulk-inserts canonical devnet mints in one click. Required: at least one mint must be registered before agents can be created.
- Add agents — policy template selection, limit override, mint selection from the registry, activation URL generation. The dashboard's success surface includes integration links to `openapi.json`, `SKILL.md`, and the MCP quick-start.
- Manage whitelist:
  - Intra-group agents listed as permanent entries (read-only)
  - Add external address: set label, TTL (expiry date), approved amount cap
  - Renew / top up: extend TTL or increase approved amount for an existing entry
  - Remove: close whitelist entry before expiry
  - Protocol addresses (DEX router, lending pools): permanent, no cap
- Configure per-agent spend limits
- Per-agent spend audit log (timestamp, amount, recipient, memo, task_id, agent ID). Each amount renders in the agent's bound symbol — the dashboard reads `AgentWallet.mint` and joins against the token registry for `{symbol, decimals}`. The fleet spend chart stacks one series per symbol.
- Whitelist approval dashboard: shows each external entry's TTL countdown, amount used vs. cap, status (active / expiring soon / voided)
- Revoke agent API key — immediately invalidates credential
- Re-invite agent — issues a fresh activation URL without rotating the key; a separate rotate-key action revokes + re-mints atomically
- Agent fleet dashboard — daily spend vs. limit, hourly tx rate, remaining headroom per agent, plus 24h confirmed/blocked tx counts and a 7-day stacked spend chart per registry symbol
- **x402 dashboard** (`/group/:id/x402`): editorial headline summarising current approvals ("Approved up to 50.00 USDC for x402 payments. Currently holding 12.30 USDC."), Configure modal that signs SPL Token `Approve`/`Revoke` per dirty currency, 30-day stacked bar chart of spending per symbol, per-endpoint aggregation table, recent payments list with Solana Explorer links
- Anomaly alert configuration — set webhook URL for policy events across the fleet
- Error recovery: all failed on-chain operations surface a human-readable error with a retry action; no silent failures. The agent wire scrubs Anchor-level internals (`AnchorError`, `Custom program error`, `0x` discriminants) — sanitization applies only to the agent-facing surface; owner-facing dashboard pages still get the typed Anchor titles and remediation hints via `friendlyAnchorMessage`.

---

## Deferred

- Backend lending-protocol adapters (Kamino, Save) so agents call `/v1/deposit` / `/v1/withdraw` with `{token, amount}` instead of raw `cpi_data` bytes, and responses surface `current_apy` / `yield_earned`.
- Incoming-payment notifications (`payment.received` webhook) via a wallet monitor that subscribes to every agent ATA over `connection.onAccountChange`.
- Anomaly webhook fan-out (`policy.limit_threshold`, `policy.whitelist_amount_threshold`, `policy.whitelist_voided`, `policy.limit_exceeded_attempt`, `policy.whitelist_violation`, `policy.whitelist_expiring`) — including a scheduled job for the TTL-expiring alert.
- Dashboard surfaces for `update_agent_limits`, `emergency_withdraw`, and `update_backend_operator`.
- Budget pool — orchestrator sets shared budget ceiling across agent fleet.
- Multi-sig for high-value operations (require orchestrator co-sign above threshold).
- Human custody model (SMS / Telegram / WhatsApp channels, NLP intent parsing).
- Fiat on/off-ramp integration.
- Cross-chain transactions.

---

## Future Scope

- Per-task budget allocation: agent requests ephemeral sub-limit for a specific task, orchestrator approves
