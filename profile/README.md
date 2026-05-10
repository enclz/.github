# Enclz

> One enclave per agent, one policy ceiling enforced on-chain.

Per-tx, daily, and hourly spend caps live in an Anchor program on Solana. Whitelist entries auto-void on-chain when their TTL or approved-amount cap is consumed. The agent never holds a private key — it authenticates with a scoped API key and submits structured intents via REST or MCP. A fully exploited agent can only move funds to pre-approved addresses, up to pre-approved amounts — independent of any backend.

## Verified engineering

| Signal | State | Proof |
|---|---|---|
| Live app | Solflare SIWS, devnet | <https://enclz.com> |
| Live docs | Docusaurus | <https://docs.enclz.com> |
| Devnet program | Anchor on Solana devnet | [`45PiBcnk…AFaLW`](https://explorer.solana.com/address/45PiBcnkKhZbzb5GQDhJ9Rikwiz3DUzyoBwiKHbAFaLW?cluster=devnet) |
| Agent SKILL.md | published | <https://enclz.com/SKILL.md> |
| OpenAPI 3.1 | published | <https://enclz.com/openapi.json> |
| Program SDK | npm | [`@enclz/sdk`](https://www.npmjs.com/package/@enclz/sdk) |
| MCP server | npm | [`@enclz/mcp`](https://www.npmjs.com/package/@enclz/mcp) |
| Program CI | passing | <https://github.com/enclz/solana/actions/workflows/program-ci.yml> |

## Quickstart

Paste this into Claude Desktop, Cursor, or any LLM agent. Replace `<token>` with the invitation URL your orchestrator hands you.

```
Read https://enclz.com/SKILL.md and follow the activation flow.

I have an Enclz invitation URL: https://enclz.com/<token>
Provision the wallet, store the API key per the skill's hard rules,
and confirm by reading enclz://balance and enclz://limits.

Do not echo or print the API key.
Do not move funds without simulating first.
```

## Documentation

| Doc | What's in it |
|---|---|
| [REQUIREMENTS.md](REQUIREMENTS.md) | Vision, principals, full feature list, user flows, security model, policy templates. |
| [SPECIFICATION.md](SPECIFICATION.md) | Tech stack, Anchor accounts + instructions, REST endpoints, webhook payloads, error taxonomy, MCP server, web app routes. |
| [RESEARCH.md](RESEARCH.md) | Validation sprint, demand signals, competitor map, risks. |
| [MARKETING.md](MARKETING.md) | Positioning, personas, GTM, demo flow, competitive matrix. |
| [decisions/](decisions/) | Architecture Decision Records — numbered, dated, one design choice each. |

## Founder

Denis Majus — [github.com/imajus](https://github.com/imajus) · [t.me/imajus](https://t.me/imajus) · [x.com/denismajus](https://x.com/denismajus)
