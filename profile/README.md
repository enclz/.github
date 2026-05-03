# Enclz

> One enclave per agent, one policy ceiling enforced on-chain.

## Concept

**Enclz gives AI agents a dedicated Solana wallet with on-chain spend enforcement.** The smart contract is the policy: limits, whitelists, and TTL-bound approvals are evaluated by an Anchor program on every transfer. The agent never holds a private key — it authenticates with a scoped API key and submits structured intents via REST or MCP.

Two principals interact with the system:

- **Orchestrator** — developer or platform operator who provisions agents, configures policy, and manages the whitelist (web app or REST API).
- **Agent** — autonomous software process that transfers, swaps, deposits, and withdraws via the Agent REST API or the `@enclz/mcp-server` MCP tool.

## Problem

AI agents need to spend money — paid APIs, micro-services, bounties, on-chain actions — but every existing option is unsafe:

- **Private key in the environment**: a hallucination, prompt injection, or config leak drains the wallet.
- **Backend-enforced limits**: bypassable the moment the backend is compromised or the operator turns malicious.
- **No wallet at all**: the agent can't operate autonomously.

Enclz takes a fourth path. Per-tx, daily, and hourly caps live in the smart contract. External recipients are whitelisted with a TTL and an approved amount cap that auto-voids on-chain when consumed. A fully exploited agent can only move funds to pre-approved addresses, up to pre-approved amounts, within pre-approved limits — independent of any backend.

## Features

- **On-chain spend policy** — per-tx, daily, and hourly caps enforced by the Anchor program; raising limits requires the orchestrator's signature.
- **TTL + amount-capped whitelist** — external addresses get an expiry and an approved amount; the entry PDA auto-closes on-chain when the cap is consumed.
- **Three whitelist categories** — intra-group agents (permanent), external recipients (TTL + capped), protocol addresses like the DEX router and lending pools (permanent).
- **No private key for the agent** — scoped API key only; revocable instantly; reissued via one-time invitation code.
- **Agent REST API** — transfer, swap, deposit, withdraw, balance, limits, history, simulate, webhooks.
- **MCP server** (`@enclz/mcp-server`) — drop-in tool integration for Claude Desktop, Cursor, and any MCP runtime; one env var to configure.
- **Simulation endpoint** — dry-run any transfer or swap against current limits and whitelist state without submitting a transaction.
- **Webhooks** — `transfer.confirmed`, `payment.received`, plus fleet-level policy alerts (`policy.limit_threshold`, `policy.whitelist_expiring`, `policy.whitelist_amount_threshold`, `policy.whitelist_voided`, `policy.whitelist_violation`).
- **Jupiter swap + lending integrations** — swaps via Jupiter v6; deposits and withdrawals against whitelisted protocols (e.g. Kamino).
- **Policy templates** — `research-agent`, `micro-payment-agent`, `payment-agent`, `custom`.
- **Orchestrator web app** — Solflare wallet connection, group setup wizard, agent fleet dashboard, whitelist management with renew/top-up flows.
- **Protocol fee** — flat 10 bps deducted on-chain at execution time; no off-chain billing.

## Documentation

| Doc | What's in it |
|---|---|
| [REQUIREMENTS.md](REQUIREMENTS.md) | Vision, target users, user flows (group setup, registration, transfer, swap, deposit/withdraw, simulation, webhooks), security model, policy templates, web app feature list. |
| [SPECIFICATION.md](SPECIFICATION.md) | Full technical spec: tech stack, Anchor accounts and instructions, backend module layout, registry schema, transaction execution pipeline, REST API endpoints, webhook payloads, error taxonomy, MCP server, web app routes, external integrations. |
| [RESEARCH.md](RESEARCH.md) | Validation sprint, demand signals, competitor map (Openfort, lobster.cash, Coinbase AgentKit, Privy, Trust Wallet), Colosseum hackathon landscape, risks. |
| [MARKETING.md](MARKETING.md) | Positioning, orchestrator personas, regulatory framing, go-to-market channels, hackathon demo flow, customer development script, competitive differentiation matrix. |

## Stack

Solana · Anchor · Node.js · Express · `@solflare-wallet/sdk` · Jupiter API v6 · QuickNode RPC · `@modelcontextprotocol/sdk` · Eitherway (web app).
