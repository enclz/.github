# 0001 — Solana as the policy storage and enforcement layer

- **Status:** Accepted
- **Date:** 2026-05-10
- **Decision-maker:** Denis Majus (founder)

## Context

Enclz's whole pitch hinges on a single property: **a fully exploited agent — leaked API key, compromised backend, malicious operator — can still only move funds within pre-approved limits to pre-approved addresses**. The thing that holds this property is wherever the policy state physically lives. So the question of "where does the policy live?" is not a tech preference; it's the entire trust model.

Three classes of options were on the table:

1. **Off-chain (Postgres + backend).** Backend reads policy from a row, enforces it before signing transfers from a hot wallet.
2. **On-chain on a non-Solana L1/L2.** Same enforcement model, different chain.
3. **On-chain on Solana.** Anchor program owns the policy PDAs; every transfer is a CPI through the program.

## Decision

**Policy state and enforcement live on Solana.** The Anchor program (`programs/enclz/`) owns three PDA families — `GroupConfig`, `AgentWallet`, `WhitelistEntry` — and is the only path through which agent funds can move. Limits, whitelist entries (with TTL + amount caps), and counters are all program state. The backend is reduced to credential dispatch (bcrypt-hashed API keys, one-time invite codes, webhooks, idempotency cache) — it never holds policy authority.

## Alternatives considered

### Alt A — Off-chain (Postgres + backend)

**Why it's tempting:** simpler stack, no chain to deploy, no audit, no rent. Can iterate on policy semantics in a single migration.

**Why we rejected it:** the trust model collapses to "trust the backend." A breached backend, a malicious operator, or even a bad migration silently moves more money than the policy says. We then have to convince every prospective user that *our* operations are trustworthy — which is the exact thing competitors with off-chain enforcement (Privy, Crossmint, Coinbase AgentKit, Cobo) struggle to differentiate on. Going on-chain is the only way to make the claim independently verifiable.

### Alt B — Ethereum / EVM L2

**Why it's tempting:** larger developer ecosystem, more agent-related tooling (ERC-7715, smart-account standards), USDC native on every L2, easier press story.

**Why we rejected it:**

- **Per-tx cost.** Even on the cheapest L2s (Base, Arbitrum, Optimism), a `whitelistedTransfer` call costs $0.001–$0.05. Agent micro-payments are 10¢ — 50% of agents' transactions would be eaten by gas. Solana's flat ~$0.0005 keeps the unit economics workable.
- **Latency.** EVM L2 finality is 1–10s on the optimistic side, instant only on the L2 sequencer's word. Solana's 400ms block time + sub-second finality is closer to the "did my transfer go through?" expectations of an agent in a chat loop.
- **State rent vs. account rent.** On Solana, opening a `WhitelistEntry` PDA costs ~0.002 SOL rent that's recovered on close. On EVM, every storage slot stays paid forever. Auto-voiding whitelist entries — central to our pitch — is *cheaper* on Solana, not more expensive.
- **The "agent gets a wallet" pattern is younger on Solana.** ERC-4337 / smart accounts are crowded; Solana has less competition for this exact niche right now.

### Alt C — Bitcoin / Cosmos / Aptos / etc.

Considered briefly, dismissed quickly. None has the combination of (a) cheap, fast micro-payments, (b) a programmable policy layer at L1, and (c) sufficient stablecoin liquidity.

## Consequences

### Accepted

- **Provability.** Every limit, every whitelist entry, every transfer is a public on-chain artifact. Marketing can link an Explorer URL instead of asking the reader to trust a database. (See the "Verified engineering" table in `profile/README.md`.)
- **Backend simplicity.** Fastify owns four endpoints (mint API key, rotate, revoke, register webhook). It can't be the source of a policy bypass because it isn't authoritative for policy.
- **Hackathon native.** Colosseum, Solana Foundation programs, QuickNode Startup Program — Solana ecosystem has the funding and exposure surface a Solana project most easily plugs into.
- **Single-chain cost model.** No cross-chain bridges, no reconciliation between L2 and L1 — the agent transfers are atomic on one chain.

### Costs we accepted

- **Solana-only liquidity.** Agents that want to spend on Ethereum mainnet need a bridge step we don't own. Acceptable because the agent payment volume on Solana is growing fastest and the alternative (multi-chain enforcement) doubles the security surface.
- **Anchor 1.0+ tooling churn.** Anchor's release cadence breaks workflows (e.g., `keys sync` rewriting `Anchor.toml`, `surfpool` vs `solana-test-validator` defaults). We've absorbed this by pinning workflows in `solana/CLAUDE.md`.
- **No "soft delete" or rollback.** A buggy Anchor deploy can lock funds. Mitigated with: upgrade authority controlled by a multisig (post-mainnet), all-state migrations tested on devnet, `emergency_withdraw` instruction reserved for the upgrade authority.

### Future revisions

If a credible cross-chain agent-payments standard emerges (ERC-7715 with cheap L1/L2 anchoring, or a Solana ↔ EVM message-passing primitive that preserves the policy enforcement guarantee), this decision can be revisited as **0002 — supporting non-Solana enforcement**. Until then, single-chain on Solana stands.
