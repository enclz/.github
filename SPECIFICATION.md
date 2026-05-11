# Enclz — Technical Specification (Agent Wallet Edition)

## Tech Stack

| Layer | Technology |
|---|---|
| Smart contract | Rust · Anchor framework |
| Blockchain | Solana (devnet → mainnet-beta) |
| On-chain security | Program-owned PDAs · whitelist PDAs · per-wallet operator nonce |
| Backend | Node.js · JavaScript |
| Web app | React 18 · Vite · Tailwind |
| Wallet (Orchestrator) | Solana wallet adapter — Solflare adapter explicitly registered, others via Wallet Standard auto-detection |
| Orchestrator auth | Sign-In-With-Solana (`supabase.auth.signInWithWeb3({ chain: 'solana', wallet, statement })`) → Supabase JWT. Same ceremony for browser and Node. |
| REST API (agents + orchestrator) | Fastify 4 · Bearer token auth (Supabase JWT for orchestrator, agent API key for agents) · webhook callbacks |
| MCP Server | TypeScript · `@modelcontextprotocol/sdk` · stdio transport · package `@enclz/mcp` |
| DEX aggregator | Jupiter API · default base `https://lite-api.jup.ag/swap/v1` (override via `JUPITER_BASE_URL`) |
| Payment protocol | x402 v1 & v2 (orchestrator-delegated SPL Token `Approve`; backend partial-signs `TransferChecked`, facilitator settles) |
| RPC provider | QuickNode |

---

## System Overview

```mermaid
graph TD
    ORC["👤 Orchestrator (A2)\nbrowser (Solflare) or headless Node script\nboth use SIWS → Supabase JWT"]
    AGT["🤖 AI Agent (B2)\nREST API — Bearer token"]
    MCP_CLIENT["🤖 MCP Agent (B2)\nClaude · Cursor · any MCP runtime"]

    WEB["🌐 Web App (React + Vite)\nSolana wallet adapter · group setup · whitelist config\nagent fleet dashboard · invite code generation"]
    OAPI["Orchestrator REST API (Fastify)\n4 credential-minting endpoints only:\nmint invite · rotate key · revoke · register fleet webhook"]
    AAPI["Agent REST API (Fastify)\n/v1/* — transfer · swap · balance · simulate · webhooks"]
    MCP["MCP Server (TypeScript)\nstdio transport · 8 tools\nENCLZ_API_KEY auth"]

    BE["⚙️ Backend Service (Fastify · Node.js)\nresolveAgentFromKey · resolveUserFromJwt · requireGroupAccess\nSolana client · Jupiter client\nWebhook dispatcher · idempotency cache"]

    RPC["Solana RPC (QuickNode)"]
    JUP["Jupiter API"]

    CHAIN["🔗 Solana on-chain\nEnclz Program\nGroupConfig PDA · AgentWallet PDAs · WhitelistEntry PDAs"]

    ORC -->|HTTPS · SIWS JWT| WEB
    ORC -->|SIWS JWT (Node script)| OAPI
    WEB -->|signed by orchestrator's wallet:\ninitialize_group · add_agent · add_to_whitelist\nrenew_whitelist_entry · remove_from_whitelist\nupdate_agent_limits · emergency_withdraw| RPC
    OAPI -->|validated request| BE

    AGT -->|Bearer token| AAPI
    AAPI -->|validated intent| BE

    MCP_CLIENT -->|MCP tool call| MCP
    MCP -->|Bearer token| AAPI

    BE -->|execute_transfer| RPC
    BE -->|swap quote + tx| JUP

    RPC --> CHAIN
```

---

## Smart Contract (Anchor / Rust)

### Accounts

#### `GroupConfig` PDA
Seed: `["group", owner_pubkey]`

```
owner:                Pubkey    // Orchestrator's Solana wallet (policy admin)
backend_operator:     Pubkey    // backend's signing keypair (authorized caller)
protocol_fee_wallet:  Pubkey    // Enclz protocol ATA receives fees
agent_count:          u8
group_name:           [u8; 32]  // human label for the group ("acme-trading-desk"); written verbatim, no on-chain validation
```

#### `AgentWallet` PDA
Seed: `["wallet", group_pubkey, agent_index as u8]`

```
group:               Pubkey
mint:                Pubkey    // settlement SPL mint — bound at add_agent, immutable, only mint that can ever leave custody via execute_transfer / execute_lending_op
display_name:        [u8; 32]  // human label for fleet dashboard ("research-bot-1")
daily_limit:         u64       // settlement-mint native units (e.g. 6-decimal USDC)
per_tx_limit:        u64
hourly_tx_cap:       u8
spent_today:         u64       // resets at UTC midnight; tracks execute_transfer + execute_lending_op only — execute_swap does not bump it
tx_count_this_hour:  u8        // resets on the hour; advances for every privileged instruction including swaps
last_spend_reset:    i64
last_hour_reset:     i64
operator_nonce:      u64       // incremented per privileged instruction — replay protection
bump:                u8        // canonical PDA bump cached to skip re-derivation
```

**Default limits at initialization:**
- `daily_limit`: 10 USDC (10_000_000)
- `per_tx_limit`: 1 USDC (1_000_000)
- `hourly_tx_cap`: 5

Token accounts (ATAs) are owned by the `AgentWallet` PDA. `add_agent` initializes the ATA for the bound mint; additional ATAs (for mints accumulated via swaps) are created on demand by the orchestrator.

**Mint binding.** The `mint` field is set once at `add_agent` time and never mutated. It defines what can leave agent custody via outbound paths: `execute_transfer` and `execute_lending_op` reject any `*_token_account.mint != agent_wallet.mint`. To repurpose an agent for a different settlement mint, retire it and create a new one — agents are cheap and binding is intentional.

#### `WhitelistEntry` PDA
Seed: `["whitelist", group_pubkey, target_address]`

```
label:            [u8; 32]  // "Helius RPC service", "Kamino USDC vault"
target:           Pubkey    // target address, stored redundantly for read-side convenience
added_by:         Pubkey    // audit trail
entry_type:       u8        // 0 = intra-group agent (permanent), 1 = external recipient (TTL + amount-capped), 2 = protocol address (permanent)
ttl_expires_at:   i64       // Unix timestamp; 0 = no expiry (used for entry_type 0 and 2)
approved_amount:  u64       // USDC 6-decimal; 0 = unlimited (used for entry_type 0 and 2)
amount_used:      u64       // cumulative amount transferred to this address; incremented on every execute_transfer
bump:             u8        // canonical PDA bump cached to skip re-derivation
```

Existence = whitelisted. Close account = removed.

`entry_type = 0` (intra-group): auto-added when an agent is created (`add_agent`). `ttl_expires_at = 0`, `approved_amount = 0`. Permanent.

`entry_type = 1` (external recipient): TTL and approved_amount set by orchestrator. On `amount_used >= approved_amount`, the account is closed automatically by `execute_transfer` (auto-void). Expired entries (`now > ttl_expires_at`) are rejected and must be closed + re-created by the orchestrator.

`entry_type = 2` (protocol): enables deposit/withdraw. `ttl_expires_at = 0`, `approved_amount = 0`. Permanent. The DEX swap router is whitelisted as `entry_type = 2` at group initialization.

### Instructions

#### `initialize_group`
Creates `GroupConfig` PDA and a `WhitelistEntry` PDA for the DEX swap router (`entry_type = 2`, permanent), so swaps work immediately after provisioning. Owner signs.

```
Accounts:
  owner               [signer, writable]
  group_config        [writable, init]  PDA
  dex_router_entry    [writable, init]  PDA  // type-2 whitelist for the swap router
  system_program

Args:
  group_name:           [u8; 32] // fixed-width human label, written verbatim (no on-chain validation)
  backend_operator:     Pubkey
  protocol_fee_wallet:  Pubkey   // Enclz's fee collection wallet
  dex_router:           Pubkey   // swap aggregator program ID; whitelisted as type-2
```

#### `add_agent`
Creates an `AgentWallet` PDA + its ATA for the supplied mint. Auto-adds the agent's PDA to the group whitelist as `entry_type = 0` (intra-group, permanent, unlimited). Call once per token the agent will hold; subsequent agents in the same group reuse the same `mint` argument shape.

```
Accounts:
  owner                      [signer, writable]
  group_config               [writable]
  agent_wallet               [writable, init]  PDA
  intra_group_entry          [writable, init]  PDA  // type-0 whitelist seeded on the agent_wallet pubkey
  agent_token_account        [writable, init]       // ATA owned by agent_wallet
  mint                       []                     // SPL mint for the ATA
  token_program              []
  associated_token_program   []
  system_program

Args:
  display_name:   [u8; 32]
  daily_limit:    Option<u64>   // overrides default if provided
  per_tx_limit:   Option<u64>
  hourly_tx_cap:  Option<u8>
```

#### `execute_transfer`
Called by the backend operator for every direct token transfer. Validates nonce, whitelist, and limits, then executes the SPL token transfer. Deducts the protocol fee at execution time. Swaps and lending operations have their own instructions (`execute_swap`, `execute_lending_op`).

```
Accounts:
  backend_operator         [signer, writable]  // pays rent if recipient ATA needs init
  group_config             []
  group_owner              [writable]   // group_config.owner — receives rent when an external whitelist entry auto-voids
  agent_wallet             [writable]   // limits + nonce updated here
  from_token_account       [writable]   // agent vault ATA
  recipient_wallet         []            // pubkey constrained != protocol_fee_wallet and != agent_wallet PDA
  to_token_account         [writable]   // recipient ATA (auto-created via init_if_needed if missing)
  whitelist_entry          [writable]   // PDA must exist for recipient address; mutated for type-1 amount_used and may be closed on auto-void
  protocol_fee_token_acct  [writable]   // Enclz protocol ATA
  mint                     []            // must match agent_wallet.mint
  token_program            []
  associated_token_program []            // needed for ATA init_if_needed
  system_program

Args:
  amount:          u64
  expected_nonce:  u64   // must match agent_wallet.operator_nonce
  agent_index:     u8    // reconstructs the agent_wallet PDA seed for the SPL transfer CPI signer
```

Enforcement (in order):
1. Verify `expected_nonce == agent_wallet.operator_nonce` → reject if mismatch
2. Increment `operator_nonce`
3. Reset `spent_today` / `tx_count_this_hour` if timestamps have rolled over
4. Reject if `amount > per_tx_limit`
5. Reject if `spent_today + amount > daily_limit`
6. Reject if `tx_count_this_hour >= hourly_tx_cap`
7. Reject if `recipient_wallet == group_config.protocol_fee_wallet` or `recipient_wallet == agent_wallet` PDA → `recipient_invalid` (checked at Anchor constraint layer, before duplicate-mut check)
8. The `whitelist_entry` PDA must exist for the recipient address — Anchor's seed constraint enforces this during account resolution. A missing PDA surfaces as Anchor's `AccountNotInitialized` (3012); the backend translates this to `whitelist_violation` for the REST response.
9. If `whitelist_entry.entry_type == 1` (external recipient):
   a. Reject if `now > whitelist_entry.ttl_expires_at` → `whitelist_expired`
   b. Reject if `whitelist_entry.amount_used + amount > whitelist_entry.approved_amount` → `whitelist_amount_exhausted`
10. Compute protocol fee:
    - `protocol_fee = ceil(amount * 10 / 10_000)`  (10 bps = 0.1%, integer ceil)
    - `total = amount + protocol_fee`
11. Execute SPL token transfer: `amount` to recipient, `protocol_fee` to `protocol_fee_wallet` (total drained from agent = `amount + fee`)
12. Increment `spent_today` (by `amount`, not `total` — fee is overhead, not spend) and `tx_count_this_hour`
13. If `whitelist_entry.entry_type == 1`: increment `whitelist_entry.amount_used` by `amount`
    - If `whitelist_entry.amount_used >= whitelist_entry.approved_amount`: close `whitelist_entry` PDA (auto-void, rent returned to owner)

#### `execute_swap`
Called by the backend operator for every swap routed through the whitelisted DEX aggregator. Authorisation rides on a `entry_type = 2` (protocol) `WhitelistEntry` keyed on the aggregator program ID, so the orchestrator can rotate router versions by editing the whitelist alone — no program redeploy. The handler deducts the protocol fee from the input mint **before** invoking the aggregator CPI; output mint is unknown to Enclz at signing time, which makes input-side fee the only deterministic option.

**Mint policy.** `execute_swap` is the only privileged path that does NOT pin the input or output mint to `agent_wallet.mint`. Instead, the load-bearing safety constraint is `to_token_account.owner == agent_wallet` PDA — swap proceeds always remain in custody of the agent_wallet PDA. A compromised operator can rotate the agent's holdings between any mints but cannot exfiltrate them; the only outbound path remains `execute_transfer`, which is pinned to the bound mint.

```
Accounts:
  backend_operator           [signer, writable]   // also pays rent for fee-ATA init_if_needed
  group_config               []
  agent_wallet               [writable]   // nonce + hourly counter updated here
  from_token_account         [writable]   // agent input ATA (any mint owned by the agent_wallet PDA)
  to_token_account           [writable]   // agent output ATA — owner MUST equal agent_wallet (custody pin)
  whitelist_entry            []           // type-2 PDA keyed on jupiter_program.key()
  input_mint                 []           // SPL mint of the input ATA — used to derive the fee-ATA seed
  protocol_fee_token_acct    [writable, init_if_needed]  // Enclz protocol ATA on the input mint; created on first use of a novel mint, rent paid by backend_operator
  protocol_fee_wallet        []           // address-bound to group_config.protocol_fee_wallet; authority for the fee ATA
  jupiter_program            []           // swap aggregator program; authorised via whitelist_entry
  token_program              []
  associated_token_program   []
  system_program
  // remaining_accounts: variable account list shaped by the route plan; passed to the aggregator CPI verbatim

Args:
  amount_in:           u64    // gross input — fee math runs on this
  minimum_amount_out:  u64    // surfaced in IDL; aggregator enforces it inside route_data
  expected_nonce:      u64
  agent_index:         u8
  route_data:          Vec<u8>   // raw aggregator instruction payload built by the backend
```

Enforcement order:
1. `expected_nonce == agent_wallet.operator_nonce` → bump nonce.
2. Roll `tx_count_this_hour` if the hour boundary has elapsed (note: `spent_today` / `last_spend_reset` are NOT touched on the swap path — daily and per-tx limits are denominated in the bound mint and meaningless across arbitrary swap mints).
3. `tx_count_this_hour < hourly_tx_cap` is the only spend-policy gate. `per_tx_limit` and `daily_limit` are NOT enforced on swaps; the custody pin removes the theft threat those limits guarded against.
4. Reject if `whitelist_entry.entry_type != 2` → `whitelist_violation`.
5. Compute fee from `amount_in` (10 bps, in the input mint).
6. Transfer fee from agent input ATA to protocol fee ATA (PDA-signed CPI). The fee ATA is created lazily via `init_if_needed` on first use of a novel input mint; rent is charged to `backend_operator`.
7. Invoke the aggregator program via CPI with `route_data` and `remaining_accounts`, signed by the agent_wallet PDA.
8. Increment `tx_count_this_hour` by 1. `spent_today` is unchanged.

#### `execute_lending_op`
Unified deposit/withdraw against a whitelisted lending program. Backend dispatches `/v1/deposit` to `op_type = 0` and `/v1/withdraw` to `op_type = 1`; these discriminants are part of the on-chain ABI and are pinned in `lib.rs` tests.

```
Accounts:
  backend_operator         [signer]
  group_config             []
  agent_wallet             [writable]
  agent_token_account      [writable]   // agent ATA on the lending mint
  whitelist_entry          []           // type-2 PDA keyed on lending_program.key()
  protocol_fee_token_acct  [writable]   // Enclz protocol ATA on the same mint
  lending_program          []           // authorised via whitelist_entry
  token_program            []
  system_program
  // remaining_accounts: lending-program-specific account list passed verbatim to the CPI

Args:
  op_type:        u8         // 0 = deposit, 1 = withdraw
  amount:         u64        // gross input — limit checks run on this
  expected_nonce: u64
  agent_index:    u8
  cpi_data:       Vec<u8>    // raw lending-program instruction payload built by the backend
```

Enforcement order:
1. `op_type ∈ {0, 1}` and `amount > 0`.
2. Nonce check → bump.
3. Counter rollover; limit checks against `amount`.
4. Reject if `whitelist_entry.entry_type != 2`.
5. Dispatch:
   - `DEPOSIT`: transfer fee from agent ATA to protocol ATA, then invoke the lending CPI for the principal.
   - `WITHDRAW`: snapshot agent ATA balance, invoke the lending CPI to redeem, reload the ATA, take the fee out of the realised delta. The agent keeps `redeemed - fee`.
6. Update counters.

#### `add_to_whitelist`
Owner-only. Creates a `WhitelistEntry` PDA.

```
Accounts:
  owner            [signer, writable]
  group_config     []
  whitelist_entry  [writable, init]  PDA
  system_program

Args:
  target_address:  Pubkey
  label:           [u8; 32]
  entry_type:      u8    // 0 = intra-group (auto, permanent), 1 = external recipient (TTL + capped), 2 = protocol (permanent)
  ttl_expires_at:  i64   // required for entry_type 1; 0 for entry_type 0 and 2
  approved_amount: u64   // required for entry_type 1 (USDC 6-decimal); 0 for entry_type 0 and 2
```

#### `renew_whitelist_entry`
Owner-only. Updates `ttl_expires_at` and/or `approved_amount` for an existing `entry_type = 1` entry. Cannot be used on intra-group or protocol entries.

```
Accounts:
  owner            [signer]
  group_config     []
  whitelist_entry  [writable]

Args:
  ttl_expires_at:  i64   // new expiry; must be > now
  approved_amount: u64   // new cap; must be >= current amount_used
```

This is the primary DAU driver: orchestrators return to renew approvals before they expire or top up the amount cap when it runs low.

#### `remove_from_whitelist`
Owner-only. Closes the `WhitelistEntry` PDA, reclaiming rent.

#### `update_agent_limits`
Owner-only. Adjusts spend limits for a specific agent.

```
Args:
  daily_limit:    Option<u64>
  per_tx_limit:   Option<u64>
  hourly_tx_cap:  Option<u8>
```

#### `emergency_withdraw`
Owner-only. Bypasses spend limits and operator nonce, sweeps the full balance of one agent ATA to a destination ATA via SPL token CPI signed by the agent_wallet PDA.

**Mint policy: parity, not absolute pin.** Both ATAs must agree on mint (`agent_token_account.mint == destination_token_account.mint`) to prevent typo-driven cross-mint transfers, but neither side is pinned to `agent_wallet.mint`. This lets the owner sweep any mint the agent has accumulated via `execute_swap` (the agent's PDA can hold ATAs for arbitrary mints under the bind-agent-mint policy). To recover multiple mints, invoke `emergency_withdraw` once per mint with that mint's pair of ATAs.

```
Accounts:
  owner                      [signer]
  group_config               []
  agent_wallet               []
  agent_token_account        [writable]   // any mint, owner == agent_wallet PDA
  destination_token_account  [writable]   // mint must match agent_token_account.mint
  token_program              []

Args:
  agent_index:  u8
```

#### `update_backend_operator`
Owner-only. Rotates the authorized backend keypair.

---

## Backend Service (Node.js)

### Module Structure

```
server/
  index.js                   // Fastify entry point — registers token-registry + activation routes, force-instantiates Solana + x402 delegate clients at boot
  app.js                     // buildApp() — registers swagger + every /api/v1/* schema-annotated route; reused by the openapi.json generator
  routes/
    agent.js                 // /v1/* agent endpoints (register, transfer, swap, deposit, withdraw, simulate, balance, limits, history, webhooks)
    orchestrator.js          // orchestrator endpoints: createAgent · mintAgentInvitation · rotateAgentKey · revokeAgentKey · registerFleetWebhook · listGroupTokens · addGroupToken · deleteGroupToken · seedDefaultTokens
    activate.js              // public agent onboarding — GET /:token (Markdown landing) + POST /api/v1/onboard/:token (single-use bash CLI drop)
    x402-history.js          // GET /v1/orchestrator/groups/:group_config_pda/x402/history · GET /…/x402/stats
  lib/
    auth.js                  // resolveAgentFromKey · resolveUserFromJwt · requireGroupAccess preHandler
    db.js                    // getServiceClient() singleton (service-role, bypasses RLS); throws at module load if Supabase env vars are missing
    solana.js                // cached getConnection() · getOperator() · getProgram()
    intents.js               // executeTransfer · executeSwap · executeLendingOp (with one nonce-mismatch resync + retry)
    jupiter.js               // Jupiter quote + swap-instructions wrapper (default base `lite-api.jup.ag/swap/v1`)
    onchain-fleet.js         // getGroupConfig · getAgentWallet · listAgentsForGroup · decodeFixed32
    onchain-verify.js        // confirmTx · verifyAgentWallet · verifyWhitelistEntry (used by the agent-creation endpoint)
    group-tokens.js          // loadTokenMeta — backend cache of (group_config_pda, mint) → {symbol, decimals, label} from `group_tokens` (60s TTL)
    invitations.js           // mintInvitation · redeemInvitation — single-use `on<8alphanumeric>` codes hashed with bcrypt
    x402-pay.js              // POST /v1/x402/pay handler: decode v1/v2 challenges, partial-sign TransferChecked with delegate keypair, return follow-up
    x402-v1.js / x402-v2.js  // version-specific challenge parsers + follow-up envelope builders
    pda.js                   // groupConfigPda · agentWalletPda · whitelistEntryPda derivation
    policy.js                // computeFee (10 bps) — only function called on the request path. TEMPLATES, applyTemplate, preflightCheck, anomalyCheck remain as helper exports for tests/tooling but are NOT invoked by route handlers.
    schemas.js               // JSON Schemas for every /api/v1/* route — single source of truth for `openapi.json`
    webhooks.js              // dispatchWebhook (fire-and-forget HMAC-signed events, refuses redirects)
    url-safety.js            // validateWebhookUrl (DNS rebinding protection)
    idempotency.js           // reserveOrAwait → complete/release pattern; PK is (key, agent_wallet_pda)
    anchor-errors.js         // façade re-exporting the program's 6000–6013 discriminants; exports agentSafeError that sanitizes on-chain internals off the agent wire
    crypto.js                // generateApiKey · hashSecret · verifySecret · generateInviteCode · signWebhookPayload

src/                         // SPA — see §"Web App"
  lib/
    anchor-client.js         // useEnclzProgram() — wallet-bound Anchor Program for orchestrator on-chain mutations
    onchain-fleet.js         // listAgentsForGroup · listWhitelistEntries · getGroupConfig · getAgentWallet (mirror of server/lib/onchain-fleet.js)
    api.js                   // thin fetch wrapper attaching the Supabase JWT, used by the orchestrator REST surface (credentials, token registry, x402 history/stats)
    quicknode.js             // shared `Connection` instance, SWR-style caches, accountSubscribe helpers for the dashboard

mcp/                         // @enclz/mcp — standalone TypeScript package, MCP stdio transport
  index.ts                   // entry: ENCLZ_API_KEY + ENCLZ_API_URL env, register 5 tools + 3 resources, stdio MCP server
  tools.ts                   // bare-named tools (no `enclz_` prefix): `transfer`, `swap`, `deposit`, `withdraw`, `simulate` — input schemas derived from `openapi.json`
  resources.ts               // MCP resources: `enclz://balance`, `enclz://limits`, `enclz://history` (read-only views over GET endpoints)
  client.ts                  // thin HTTP client wrapping Agent REST API
  generated/                 // build artifacts derived from ../openapi.json
  package.json               // { "name": "@enclz/mcp", "bin": { "enclz-mcp": "./dist/index.js" } }
```

### Registry Data Model

The chain is the source of truth for groups, agents, and whitelist entries. The database holds state the chain doesn't model: bcrypt-hashed credentials, one-time invitation codes, webhook subscribers, the idempotency cache, the per-group SPL token catalogue, and the x402 payment audit log. Every table is keyed on base58 PDAs (text) and is service-role only — Fastify uses the service-role client, which bypasses RLS.

```js
// agent_credentials table
{
  agent_wallet_pda:  string,   // base58 — primary identifier
  api_key_hash:      string,   // bcrypt hash — plaintext never stored
  revoked:           boolean,
  created_at:        Date,
  revoked_at:        Date | null,
}

// agent_invitations table
{
  agent_wallet_pda:  string,   // base58 — primary identifier (the on-chain agent already exists at invite time)
  code_hash:         string,   // hash of invitation code — plaintext never stored
  used:              boolean,
  expires_at:        Date,     // 24 hours from creation
  created_at:        Date,
}

// agent_webhooks table
{
  id:                string,   // uuid
  group_config_pda:  string,   // base58 — set for every row
  agent_wallet_pda:  string,   // base58 — null for fleet-level orchestrator webhooks
  url:               string,   // HTTPS callback URL
  secret:            string,   // HMAC signing secret for payload verification
  event_types:       string[], // e.g. ["transfer.confirmed","policy.limit_threshold"]
  active:            boolean,
  created_at:        Date,
}

// idempotency_cache table  (PK: (key, agent_wallet_pda))
{
  key:               string,   // idempotency_key from request
  agent_wallet_pda:  string,   // base58
  status:            string,   // 'in_flight' | 'completed'
  response:          json,     // cached response body — populated when status = 'completed'
  created_at:        Date,
  expires_at:        Date,     // 24h after creation; cleanup_idempotency_cache() drops stale rows
}

// group_tokens table  (PK: (group_config_pda, mint))
{
  group_config_pda:  string,   // base58
  mint:              string,   // base58 SPL mint
  symbol:            string,   // "USDC", "PYUSD", etc.
  decimals:          number,   // 0..18, re-verified on insert via getMint(connection, mint)
  label:             string,   // optional human label
  added_at:          Date,
}

// x402_transactions table  (PK: id — uuid)
{
  id:                string,   // uuid
  group_config_pda:  string,   // base58
  agent_wallet_pda:  string,   // base58
  resource_url:      string,   // resource the agent paid for
  amount:            string,   // atomic units (numeric / bigint), denominated in `mint`
  mint:              string,   // base58 SPL mint
  pay_to:            string,   // base58 — recipient ATA / wallet from the challenge
  x402_version:      string,   // "v1" | "v2"
  tx_sig:            string,   // base58 of the delegate's ed25519 signature over the partial-signed tx (nullable)
  created_at:        Date,
}
```

The SPA enumerates fleet metadata via Anchor account fetchers (`program.account.agentWallet.all` filtered by `memcmp` on the `group` field; `program.account.whitelistEntry.all` filtered on `added_by`) — see `src/lib/onchain-fleet.js`. The backend reads the same accounts on demand via `server/lib/onchain-fleet.js`.

### Policy Enforcement

Limits and whitelist enforcement live exclusively on-chain. The backend submits the Anchor instruction directly and surfaces the program's custom error codes (6000–6013) as HTTP errors via `parseAnchorError`. The chain is the only authority.

`server/lib/policy.js` exports `computeFee` (10 bps protocol fee) — the only policy function on the request path. `TEMPLATES` and `applyTemplate` are helper exports for tests and the policy-template UI.

After a confirmed `execute_transfer` / `execute_swap` / `execute_lending_op`, the backend re-fetches the on-chain `AgentWallet` account to populate `daily_remaining` / `hourly_tx_remaining` in the response.

### Policy Templates

```js
// policy/templates.js
const TEMPLATES = {
  'research-agent': {
    per_tx_limit: 0.10,     // USDC
    daily_limit:  1.00,
    hourly_tx_cap: 10,
  },
  'micro-payment-agent': {
    per_tx_limit: 1.00,
    daily_limit:  10.00,
    hourly_tx_cap: 5,
  },
  'payment-agent': {
    per_tx_limit: 10.00,
    daily_limit:  100.00,
    hourly_tx_cap: 20,
  },
};
```

### Transaction Execution Pipeline

For every agent intent (transfer / swap / deposit / withdraw):

```
resolveAgentFromKey(api_key)
  → bcrypt-match against agent_credentials.api_key_hash
  → fetch on-chain AgentWallet + GroupConfig (chain is the source of truth)
  → reject 401 if revoked

idempotency.reserveOrAwait(key, agent_wallet_pda)
  → INSERT … ON CONFLICT DO NOTHING into idempotency_cache
  → if loser of race: poll until winner publishes the response
  → if cached completed row: return cached response

if transfer:
  validate recipient is base58
  buildExecuteTransferIx(agent, recipient, amount, token)

if swap:
  fetchJupiterQuote(from_token, to_token, amount)
  fetchJupiterSwapInstructions(...)
  extract swapInstruction.data → routeData (Buffer)
  buildExecuteSwapIx(agent, amount_in, min_out, route_data)

if deposit / withdraw:
  validate `lending_program` is whitelisted (entry_type = 2) for the agent's group
  buildExecuteLendingOpIx(agent, op_type, amount, lending_program, cpi_data)

submit_tx(signed by operator keypair)
  → on NonceMismatch (6006): resolveNonceMismatch (one resync + retry)
  → on other Anchor errors (6000–6013): parseAnchorError → HTTP 403 with structured code
  → on RPC submission failure: 503 with retry_after

re-fetch AgentWallet on-chain → populate daily_remaining / hourly_tx_remaining in response

idempotency.complete(key, agent_wallet_pda, response)

dispatchWebhook(agent_wallet_pda, 'transfer.confirmed' | 'swap.confirmed', payload)
```

### Wallet Monitor

On backend startup, subscribes to every registered agent wallet ATA via QuickNode websocket (`connection.onAccountChange`). On balance increase, dispatches `payment.received` webhook to the agent's registered webhook URL.

```
onAccountChange(ata_pubkey)
  → fetch new balance
  → if balance > last_known_balance:
      resolve agent from registry
      dispatch_webhook(agent, 'payment.received', { amount, from: null })
      update last_known_balance
```

---

## REST API

### Agent Authentication

All `/v1/*` endpoints (except `/v1/register`) require:
```
Authorization: Bearer <api_key>
```

Scoped API key issued at registration. Never stored in plaintext. Resolves to `agent_id`. Revoked credentials return `401`.

### Orchestrator Authentication

All `/v1/orchestrator/*` endpoints require:
```
Authorization: Bearer <supabase_jwt>
```

The JWT is issued by `supabase.auth.signInWithWeb3({ chain: 'solana', wallet, statement })` — the same Sign-In-With-Solana ceremony used by the browser SPA and headless Node scripts. Wallet ownership IS the orchestrator credential.

`requireGroupAccess` preHandler:
1. Validates the JWT via `db.auth.getUser(token)` (5-minute in-memory cache).
2. Extracts the wallet pubkey from `user.user_metadata.custom_claims.address` / `user.user_metadata.address`.
3. Computes `groupConfigPda(walletPubkey).toBase58()` and compares it to the URL's `:group_config_pda` param.
4. Rejects 403 on mismatch, 401 on malformed pubkey claim.

The chain's PDA derivation IS the ownership proof.

---

### Orchestrator Endpoints

The orchestrator REST surface covers three concerns: minting / rotating / revoking one-time-visible plaintext secrets that the chain cannot model (invitation codes, agent API keys, webhook signing secrets), curating the per-group SPL token registry, and exposing the x402 payment audit log + aggregations. Every other orchestrator action — group creation, whitelist add/renew/remove, agent creation, per-agent limit updates, emergency withdraw, backend-operator rotation, x402 budget approvals — is performed by the orchestrator's wallet signing the corresponding Anchor (or SPL Token) instruction directly (Solflare in the browser, raw keypair in a Node script). The chain account is the persisted state.

URL path params:
- `:group_config_pda` — base58 `GroupConfig` PDA (every owner has exactly one group, since the PDA is `["group", owner_pubkey]`).
- `:agent_wallet_pda` — base58 `AgentWallet` PDA.
- `:mint` — base58 SPL mint, in token-registry DELETE.

#### `POST /v1/orchestrator/groups/:group_config_pda/agents`

Mint a one-time invitation code for an agent the orchestrator has just created on-chain. The orchestrator's wallet must have already signed and confirmed `add_agent`; this endpoint verifies the resulting `AgentWallet` PDA belongs to the route's group AND `AgentWallet.mint == token_mint` AND `(group_config_pda, token_mint)` exists in `group_tokens`, then issues the invitation.

```js
// Request
{
  "tx_sig":           "string",   // confirmed signature of the add_agent instruction
  "agent_wallet_pda": "string",   // base58
  "token_mint":       "string"    // base58 — must equal AgentWallet.mint AND be registered in group_tokens
}

// Response 200
{
  "agent_wallet_pda": "string",
  "invitation_code":  "string",  // `on<8 alphanumeric>` — one-time, expires 24h, shown once
  "expires_at":       "string"   // ISO8601
}

// Response 400 — error in { onchain_mismatch | token_mint_mismatch | token_not_in_registry }
```

`token_mint` is NOT persisted (the chain owns the binding via `AgentWallet.mint`); it is accepted purely to validate that the orchestrator's selection matches both the chain state and the registry.

#### `POST /v1/orchestrator/groups/:group_config_pda/agents/:agent_wallet_pda/invitation`

Re-mint an activation link for an existing agent. Any prior pending invitation for this agent is invalidated atomically so only the fresh code can be redeemed. Distinct from `rotate-key`, which also re-issues the underlying API key — this endpoint only re-issues the onboarding token.

```js
// Response 200
{ "agent_wallet_pda": "string", "invitation_code": "string", "expires_at": "string" }
```

#### `POST /v1/orchestrator/groups/:group_config_pda/agents/:agent_wallet_pda/rotate-key`

Revoke the existing credential and mint a new plaintext API key for the agent (e.g., after a suspected compromise). Single atomic transition.

```js
// Response 200
{ "agent_wallet_pda": "string", "api_key": "string" }   // api_key shown once
```

#### `POST /v1/orchestrator/groups/:group_config_pda/agents/:agent_wallet_pda/revoke`

Revoke the agent's API key immediately without minting a replacement.

```js
// Response 200
{ "agent_wallet_pda": "string", "revoked": true }
```

After revocation, the orchestrator calls `rotate-key` (above) when ready to re-issue.

#### `POST /v1/orchestrator/groups/:group_config_pda/webhooks`

Register a fleet-level webhook for policy events across all agents in the group.

```js
// Request
{
  "url":         "string",   // must be HTTPS — validateWebhookUrl rejects loopback / RFC1918 / link-local / etc.
  "event_types": ["string"]  // ["policy.limit_threshold","policy.limit_exceeded_attempt","policy.whitelist_violation","policy.whitelist_expiring","policy.whitelist_amount_threshold","policy.whitelist_voided"]
}

// Response 200
{ "webhook_id": "string", "signing_secret": "string" }  // signing_secret shown once
```

#### Token registry endpoints

The per-group SPL token catalogue is the source of `{symbol, decimals, label}` lookups for the SPA, backend, and orchestrator. Mints listed here are the only ones agents may bind to at `add_agent` time. Each agent's bound mint is captured into `AgentWallet.mint` on-chain — the registry is metadata only.

`GET /v1/orchestrator/groups/:group_config_pda/tokens` — list registry rows.

```js
// Response 200
{ "tokens": [ { "mint": "string", "symbol": "string", "decimals": number, "label": "string", "added_at": "string" } ] }
```

`POST /v1/orchestrator/groups/:group_config_pda/tokens` — add a mint. The backend re-fetches `getMint(connection, mint)` and rejects `400 invalid_decimals` if the body's `decimals` doesn't match the on-chain value (defends against a client-side spoof that would silently break amount conversion). `400 invalid_mint` if the pubkey has no SPL token program owner.

```js
// Request
{ "mint": "string", "symbol": "string", "decimals": number, "label": "string" }

// Response 201
{ "mint": "string", "symbol": "string", "decimals": number, "label": "string", "added_at": "string" }
```

`DELETE /v1/orchestrator/groups/:group_config_pda/tokens/:mint` — remove a mint. Refused `409 token_in_use` with a `bound_agent_count` field if any agent in the group has `AgentWallet.mint == :mint` (resolved on-chain via `listAgentsForGroup`). Otherwise `204`.

`POST /v1/orchestrator/groups/:group_config_pda/seed-defaults` — convenience for the SPA / Node example: bulk-inserts the canonical devnet mints (USDC, PYUSD, etc.) into the registry, skipping rows already present. Returns the list of inserted entries.

#### x402 orchestrator endpoints

x402 budget state (the orchestrator's spending cap for autonomous HTTP-402 payments) is held on-chain as SPL Token delegation: the orchestrator signs `Approve` from the SPA pointing the platform delegate (pubkey derived from `X402_DELEGATE_KEYPAIR`) at their own ATA, with `delegated_amount` as the cap. The dashboard reads `TokenAccount.delegate / delegatedAmount / amount` live from chain.

`GET /v1/orchestrator/groups/:group_config_pda/x402/history` — paginated x402 payment history. Query params `limit` (default 50, max 200) and `offset` (default 0).

```js
// Response 200
{
  "transactions": [
    { "id": "string", "agent_wallet_pda": "string", "resource_url": "string", "amount": "string", "mint": "string", "pay_to": "string", "tx_sig": "string", "x402_version": "string", "created_at": "string" }
  ],
  "total":  number,
  "limit":  number,
  "offset": number
}
```

`GET /v1/orchestrator/groups/:group_config_pda/x402/stats` — trailing-30-day aggregations bucketed by `(resource_url, mint)` and `(UTC date, mint)`. Amounts in atomic units; the caller scales to human units using the registry decimals for each `mint`.

```js
// Response 200
{
  "endpoints": [ { "url": "string", "mint": "string", "total_spent": "string", "tx_count": number, "last_used": "string" } ],
  "daily":     [ { "date":  "string", "mint": "string", "total_spent": "string", "tx_count": number } ]
}
```

#### Orchestrator actions performed on-chain (no REST endpoint)

| Action | Anchor instruction | Signed by |
|---|---|---|
| Create group | `initialize_group(group_name, backend_operator, protocol_fee_wallet, dex_router)` | orchestrator's wallet |
| Create agent | `add_agent(display_name, daily_limit?, per_tx_limit?, hourly_tx_cap?)` | orchestrator's wallet |
| Update per-agent limits | `update_agent_limits(daily_limit?, per_tx_limit?, hourly_tx_cap?)` | orchestrator's wallet |
| Add whitelist entry | `add_to_whitelist(target, label, entry_type, ttl_expires_at, approved_amount)` | orchestrator's wallet |
| Renew whitelist entry | `renew_whitelist_entry(target, ttl_expires_at, approved_amount)` | orchestrator's wallet |
| Remove whitelist entry | `remove_from_whitelist(target)` | orchestrator's wallet |
| Emergency withdraw | `emergency_withdraw(destination)` | orchestrator's wallet |
| Rotate backend operator | `update_backend_operator(new_operator)` | orchestrator's wallet |

These are signed via `useEnclzProgram()` in the SPA (`src/lib/anchor-client.js`) or via the equivalent Anchor `Program` constructor in a Node script. After `confirmed`, the dashboard re-fetches via `listAgentsForGroup` / `listWhitelistEntries`. The agent-creation flow is the only one that follows up with a backend POST (to mint the invitation code).

---

### Agent Activation (Public)

The canonical agent-onboarding flow is a single URL the orchestrator shares with the agent: `https://enclz.com/on<8 alphanumeric>`. The agent (or a human pasting the URL into a terminal) follows two routes:

#### `GET /:token` — Markdown landing

Returns a Markdown response (`Content-Type: text/markdown; charset=utf-8`, `Cache-Control: no-store`) describing how to install the bash CLI. Side-effect free: multiple GETs with the same token return identical bodies regardless of redemption state. No DB writes.

#### `POST /api/v1/onboard/:token` — single-use installer

Atomically redeems the matching `agent_invitations` row (sets `used = true`), mints a fresh agent API key, persists its bcrypt hash to `agent_credentials`, and returns an executable bash script (`Content-Type: text/x-shellscript; charset=utf-8`). The script begins with `#!/usr/bin/env bash` and embeds three single-quoted constants: `_K=<api_key>`, `_U=https://enclz.com`, `_P=<agent_wallet_pda>`. Pipe straight to `bash` to install `~/.local/bin/enclz`; the script never enters the agent's context.

The installed CLI is a thin curl wrapper. The first positional argument is the endpoint path (relative to `/api/v1/`), the optional second is a raw JSON body. With a body it POSTs `Content-Type: application/json`; without one it GETs. All requests carry `Authorization: Bearer <api_key>` and `X-Enclz-Version: 0.1.0`. The CLI also recognizes `enclz wallet` (prints `_P`), `enclz help`, and `enclz x402/pay '<challenge>' [--v1|--v2] [--idempotency-key <key>]`. Idempotency keys are auto-generated (UUID) when not supplied.

Single-use: a second POST with the same token returns `404 invalid_or_expired_token`. Tokens expire 24h after mint. `POST /api/v1/onboard/:token?api_url=<url>` is honored only when the request's `Host` header is `localhost:*` / `127.0.0.1:*` — production silently drops the override.

### Agent Endpoints

#### `POST /v1/register`

Direct JSON exchange of an invitation code for an API key, for orchestrators that mint the credential without using the activation URL (e.g. headless automation). Same single-use semantics as the activation flow's POST step; the response is JSON `{ agent_wallet_pda, api_key }` instead of a bash script.

```js
// Request
{ "invitation_code": "string" }

// Response 200
{ "agent_wallet_pda": "string", "api_key": "string" }   // api_key shown once

// Response 400
{ "error": "invalid_invitation_code", "message": "Code expired, already used, or not found." }
```

#### `POST /v1/transfer`

Transfer tokens to a whitelisted address. The mint is sourced from `AgentWallet.mint` (chain-loaded by `resolveAgentFromKey`); decimals from the `group_tokens` registry. The optional `mint` body field is accepted only as a hint and ignored when it disagrees with `AgentWallet.mint` — the bound mint is canonical.

```js
// Request
{
  "to":               "string",   // base58 Solana address — must be whitelisted
  "amount":           number,     // human units (e.g. 2.50 for 2.50 USDC)
  "mint":             "string",   // optional override / hint; ignored if it disagrees with AgentWallet.mint
  "memo":             "string",   // optional — stored with transaction
  "task_id":          "string",   // optional — orchestrator-defined task reference
  "idempotency_key":  "string"    // required (header `Idempotency-Key` also accepted)
}

// Response 200
{
  "tx_sig":               "string",
  "status":               "confirmed",
  "net_amount":           number,   // after protocol fee
  "protocol_fee":         number,
  "daily_remaining":      number,
  "hourly_tx_remaining":  number
}

// Response 403
{
  "error":   "whitelist_violation" | "whitelist_expired" | "whitelist_amount_exhausted" | "daily_limit_exceeded" | "per_tx_limit_exceeded" | "hourly_cap_exceeded",
  "message": "string"
}

// Response 503 (transient RPC or submission failure)
{
  "error":       "submission_failed",
  "message":     "string",
  "retry_after": number   // seconds; agent retries with same idempotency_key
}
```

#### `POST /v1/swap`

Swap tokens via the whitelisted Jupiter router. Swap router is whitelisted at group initialization. Both mints are validated against the `group_tokens` registry so the backend has known decimals for `amount`. The `from_token` / `to_token` symbol fields stand alongside `from_mint` / `to_mint` purely for human readability — the mint pubkeys are canonical.

The on-chain v0.3.0 `execute_swap` enforces custody via `to_token_account.owner == agent_wallet.key()` and does NOT pin the input mint to `AgentWallet.mint` — agents that have accumulated a non-bound mint can swap it back out. Daily limits are NOT touched on the swap path (denominated only in the bound mint), so swap responses omit `daily_remaining` but still include `hourly_tx_remaining`.

```js
// Request
{
  "from_token":         "string",   // human symbol, e.g. "USDC"
  "to_token":           "string",   // human symbol, e.g. "SOL"
  "from_mint":          "string",   // base58 — must be in group_tokens
  "to_mint":            "string",   // base58 — must be in group_tokens
  "amount":             number,     // human units of `from_mint`
  "minimum_amount_out": number,     // optional; defaults to aggregator quote
  "slippage_bps":       number,     // optional; default 50
  "task_id":            "string",   // optional
  "idempotency_key":    "string"    // required
}

// Response 200
{
  "tx_sig":              "string",
  "status":              "confirmed",
  "received_amount":     number,
  "rate":                number,
  "hourly_tx_remaining": number
}
```

#### `POST /v1/deposit`

Deposit tokens into a whitelisted lending protocol.

Pass-through CPI shape: the agent supplies the lending program's pubkey AND the prebuilt CPI instruction bytes. The backend validates the lending program is whitelisted (`entry_type = 2`) for the agent's group and submits `execute_lending_op(0, ...)` to the Enclz program, which CPI-invokes the lending program with `cpi_data`. The mint comes from `AgentWallet.mint` (chain-loaded).

```js
// Request
{
  "lending_program":  "string",     // base58 — must be whitelisted entry_type = 2 for this agent's group
  "amount":           number,
  "cpi_data":         [number],     // raw instruction bytes (array of 0..255) forwarded verbatim to the lending program
  "task_id":          "string",     // optional
  "idempotency_key":  "string"      // required
}

// Response 200
{
  "tx_sig":           "string",
  "status":           "confirmed",
  "deposited_amount": number
}

// Response 403
{ "error": "whitelist_violation" | "daily_limit_exceeded" | "per_tx_limit_exceeded" | "hourly_cap_exceeded" }
```

#### `POST /v1/withdraw`

Withdraw tokens from a whitelisted lending protocol. Same pass-through CPI shape as deposit.

```js
// Request
{
  "lending_program":  "string",
  "amount":           number,
  "cpi_data":         [number],
  "task_id":          "string",
  "idempotency_key":  "string"
}

// Response 200
{
  "tx_sig":          "string",
  "status":          "confirmed",
  "received_amount": number
}

// Response 403
{ "error": "whitelist_violation" | "insufficient_balance" }
```

#### `POST /v1/intents/simulate`

Dry-run check: would this transfer succeed? Mirrors `/v1/transfer` and surfaces policy violations as `reason` plus the agent's remaining headroom. Implemented via `connection.simulateTransaction(tx, { sigVerify: false, replaceRecentBlockhash: true, commitment: 'confirmed' })` against the same instruction `/v1/transfer` would build, so the on-chain enforcement path is exercised. No transaction submitted; no body `mint` field — the bound mint comes from `AgentWallet.mint`.

```js
// Request
{
  "to":     "string",   // optional — when omitted, the simulation tests pure spend-headroom and protocol fee math
  "amount": number,
  "mint":   "string"    // optional override; ignored if it disagrees with AgentWallet.mint
}

// Response 200
{
  "would_succeed":       boolean,
  "reason":              null | "whitelist_violation" | "daily_limit_exceeded" | "per_tx_limit_exceeded" | "hourly_cap_exceeded" | "insufficient_balance",
  "daily_remaining":     number,
  "hourly_tx_remaining": number,
  "estimated_fee":       number    // protocol fee in the bound mint's human units
}
```

#### `GET /v1/balance`

Current balances and spend headroom. Balances are keyed by symbol; the backend enumerates the group's `group_tokens` registry, computes each agent ATA, and reads `getMultipleAccountsInfo` for native SOL and SPL token amounts (one entry per registered mint plus `"SOL"` for native lamports).

```js
// Response 200
{
  "balances": { "<symbol>": number, ... },
  "daily_limit":         number,
  "daily_remaining":     number,
  "per_tx_limit":        number,
  "hourly_tx_cap":       number,
  "hourly_tx_remaining": number
}
```

#### `GET /v1/limits`

Full spend policy and whitelist.

```js
// Response 200
{
  "daily_limit":         number,
  "daily_remaining":     number,
  "per_tx_limit":        number,
  "hourly_tx_cap":       number,
  "hourly_tx_remaining": number,
  "whitelist": [
    {
      "address":          "string",
      "label":            "string",
      "entry_type":       "intra_group" | "external" | "protocol",
      "ttl_expires_at":   number | null,   // Unix timestamp; null for permanent entries
      "approved_amount":  number | null,   // null for permanent entries
      "amount_used":      number | null,   // null for permanent entries
      "amount_remaining": number | null    // null for permanent entries
    }
  ]
}
```

#### `GET /v1/history`

Paginated transaction log for this agent's wallet.

```js
// Query params: ?limit=20&before=<tx_sig>
// Response 200
{
  "transactions": [{
    "tx_sig":       "string",
    "action":       "transfer" | "swap" | "deposit" | "withdraw",
    "amount":       number,
    "net_amount":   number,
    "protocol_fee": number,
    "token":        "string",
    "to":           "string",
    "memo":         "string",
    "task_id":      "string",
    "timestamp":    number
  }]
}
```

#### `POST /v1/webhooks`

Register callback URL for async transaction confirmations and policy events for this agent.

```js
// Request
{
  "url":          "string",    // must be HTTPS
  "event_types":  ["string"]   // ["transfer.confirmed","swap.confirmed","payment.received","policy.limit_threshold"]
}

// Response 200
{ "webhook_id": "string", "signing_secret": "string" }   // signing_secret shown once
```

#### `POST /v1/x402/pay` — orchestrator-delegated payment

Resolves an x402 HTTP 402 Payment Required challenge using the orchestrator's pre-approved SPL Token delegation. The agent submits the verbatim challenge string (v1 raw JSON or v2 base64-encoded `PAYMENT-REQUIRED` header value — encoding is auto-detected); the backend decodes, validates the scheme is `exact` on a Solana network, looks up the challenge's `asset` mint against the group's `group_tokens` registry (decimals required for v2's human `amount` and v1's `maxAmountRequired` conversion), reads the orchestrator's ATA `delegate / delegatedAmount / amount` live from chain, and rejects with `402 x402_budget_not_configured | x402_budget_exhausted | insufficient_x402_budget` when the delegation is missing or insufficient.

On success the backend builds a V0 transaction `[ComputeBudget SetLimit, ComputeBudget SetPrice, Token TransferChecked]` with the orchestrator's budget ATA as source, the challenge's `payTo` ATA as destination, the platform delegate (pubkey derived from `X402_DELEGATE_KEYPAIR`) as TransferChecked authority, and the challenge's `extra.feePayer` as the transaction fee payer. The delegate partial-signs; the fee-payer signature slot is left empty. The signed offer is recorded in `x402_transactions`. The response carries a `follow_up` envelope the agent attaches to its retry request — the x402 facilitator on the resource server's side asserts the transaction is fully signed (signing as fee payer itself) and submits it on chain. The backend does NOT call `sendRawTransaction`.

```js
// Request
{
  "challenge":       "string",   // verbatim v1 JSON or v2 base64; auto-detected
  "version":         "v1"|"v2",  // optional override; auto-detected from x402Version field otherwise
  "idempotency_key": "string"    // optional
}

// Response 200
{
  "accepted": true,
  "follow_up": {
    "url":     "string",   // resource URL from the challenge
    "headers": { "X-PAYMENT": "string" }            // v1, base64-wrapped PaymentPayload
              | { "PAYMENT-SIGNATURE": "string" }   // v2
  }
}

// Response 400 — { invalid_challenge | scheme_unsupported | token_not_in_registry }
// Response 402 — { x402_budget_not_configured | x402_budget_exhausted | insufficient_x402_budget }
```

#### `POST /v1/x402/proxy` — agent-funded payment via `execute_transfer`

Alternative x402 resolution that pays directly from the agent's own wallet, subject to the on-chain whitelist and spend-policy ceiling (not the orchestrator's delegated allowance). The backend parses the v2 `PaymentRequired` body, ensures `scheme === "exact"` and `network` starts with `solana:`, then submits `executeTransfer` via the standard intent pipeline using the agent's resolved context (bound mint + registry decimals). The challenge's `payTo` must already be on the agent's whitelist; otherwise the on-chain program rejects with `whitelist_violation` and the proxy surfaces it through `agentSafeError`. Returns the confirmed `tx_sig` as proof of payment.

```js
// Request
{
  "challenge":       { ... },     // x402 v2 PaymentRequired object
  "idempotency_key": "string"     // optional
}

// Response 200
{
  "tx_sig":              "string",
  "accepted":            { ... },  // the matched payment requirement
  "daily_remaining":     number,
  "hourly_tx_remaining": number
}

// Response 400 — { invalid_challenge | scheme_unsupported }
// Response 403 — propagated from execute_transfer (whitelist / limits)
```

### Webhook Event Payloads

All webhook requests are signed with `X-Enclavez-Signature: sha256=<hmac>`.

```js
// transfer.confirmed
{
  "event":    "transfer.confirmed",
  "agent_wallet_pda": "string",
  "tx_sig":   "string",
  "amount":   number,
  "net_amount": number,
  "to":       "string",
  "memo":     "string",
  "task_id":  "string",
  "timestamp": number
}

// payment.received
{
  "event":    "payment.received",
  "agent_wallet_pda": "string",
  "amount":   number,
  "token":    "string",
  "timestamp": number
}

// policy.limit_threshold  (80% of daily limit reached)
{
  "event":           "policy.limit_threshold",
  "agent_wallet_pda": "string",
  "daily_limit":     number,
  "daily_spent":     number,
  "daily_remaining": number,
  "timestamp":       number
}

// policy.limit_exceeded_attempt
{
  "event":     "policy.limit_exceeded_attempt",
  "agent_wallet_pda": "string",
  "attempted_amount": number,
  "reason":    "daily_limit_exceeded" | "per_tx_limit_exceeded" | "hourly_cap_exceeded",
  "timestamp": number
}

// policy.whitelist_violation
{
  "event":              "policy.whitelist_violation",
  "agent_wallet_pda": "string",
  "attempted_recipient": "string",
  "timestamp":          number
}

// policy.whitelist_expiring  (fired once, 24h before TTL expiry)
{
  "event":           "policy.whitelist_expiring",
  "group_config_pda": "string",
  "address":         "string",
  "label":           "string",
  "ttl_expires_at":  number,
  "amount_remaining": number,
  "timestamp":       number
}

// policy.whitelist_amount_threshold  (fired when 80% of approved_amount consumed)
{
  "event":            "policy.whitelist_amount_threshold",
  "group_config_pda": "string",
  "agent_wallet_pda": "string",
  "address":          "string",
  "label":            "string",
  "approved_amount":  number,
  "amount_used":      number,
  "amount_remaining": number,
  "timestamp":        number
}

// policy.whitelist_voided  (fired when approved_amount fully consumed and entry auto-closed)
{
  "event":           "policy.whitelist_voided",
  "group_config_pda": "string",
  "agent_wallet_pda": "string",
  "address":         "string",
  "label":           "string",
  "approved_amount": number,
  "timestamp":       number
}
```

### Error Format

All error responses follow:
```js
{ "error": "error_code", "message": "human-readable description" }
```

HTTP status codes: `400` bad request, `401` unauthorized, `403` policy violation, `404` not found, `409` conflict (idempotency), `429` rate limit, `500` internal.

**Full error code taxonomy:**

| Code | HTTP | Meaning |
|---|---|---|
| `invalid_invitation_code` | 400 | Code expired, used, or not found |
| `invalid_amount` | 400 | Amount ≤ 0 or non-numeric |
| `invalid_address` | 400 | Not a valid base58 Solana address |
| `invalid_ttl` | 400 | Whitelist `ttl_expires_at` is not in the future |
| `unauthorized` | 401 | Missing or invalid API key |
| `revoked_credential` | 401 | API key has been revoked |
| `whitelist_violation` | 403 | Recipient address not in whitelist |
| `whitelist_expired` | 403 | Whitelist entry TTL has expired — orchestrator must re-approve |
| `whitelist_amount_exhausted` | 403 | Whitelist approved amount fully consumed — orchestrator must renew |
| `daily_limit_exceeded` | 403 | Transfer would exceed daily spend ceiling |
| `per_tx_limit_exceeded` | 403 | Transfer exceeds per-transaction limit |
| `hourly_cap_exceeded` | 403 | Hourly transaction count cap reached |
| `no_whitelisted_protocol` | 403 | No eligible lending protocol in whitelist |
| `insufficient_deposit_balance` | 403 | Withdraw amount exceeds protocol deposit |
| `insufficient_balance` | 403 | Agent wallet has insufficient token balance |
| `idempotency_conflict` | 409 | Same idempotency key used with different parameters |
| `idempotency_in_progress` | 409 | Concurrent retry: the prior request is still running |
| `token_not_in_registry` | 400 | Mint absent from the group's `group_tokens` registry |
| `invalid_decimals` | 400 | Body `decimals` doesn't match `getMint` on chain |
| `invalid_mint` | 400 | Pubkey is not a real SPL mint |
| `token_in_use` | 409 | Token registry DELETE refused — at least one agent has `AgentWallet.mint == :mint` |
| `validation_error` | 400 | JSON Schema rejection; `details[]` names each failing field path |
| `scheme_unsupported` | 400 | x402 challenge scheme is not Solana `exact` |
| `invalid_challenge` | 400 | x402 challenge missing required fields or malformed |
| `x402_budget_not_configured` | 402 | Orchestrator has not approved the platform delegate on the challenge asset |
| `x402_budget_exhausted` | 402 | Cumulative offers would exceed the orchestrator's `delegatedAmount` |
| `insufficient_x402_budget` | 402 | Budget ATA `amount < amount_in` |
| `x402_delegate_mismatch` | 400 | Budget ATA `delegate` is set but doesn't match the platform delegate |
| `jupiter_unavailable` | 502 | Jupiter quote / swap-instructions endpoint failed |
| `submission_failed` | 503 | Transient RPC or network failure; retry with same idempotency key |
| `nonce_mismatch` | (internal) | Consumed by the backend's `resolveNonceMismatch` retry path — never on the agent wire. Leaks past retry surface as `internal_error` |
| `internal_error` | 500 / 502 | Unexpected backend or chain error |

---

## Agent Integration Resources

### `openapi.json`

OpenAPI 3.1 spec covering all agent REST endpoints (`/v1/*`). Generated from route definitions and kept in sync. Consumed by API clients, Postman, and AI code assistants.

Location: `docs/openapi.json`

### `SKILL.md`

Markdown file designed for injection into agent system prompts or context windows. Describes:
- Available operations (transfer, swap, deposit, withdraw, balance, limits, history, simulate)
- Request/response field formats
- Error codes and how to handle them
- Policy constraints (whitelist requirement, limit semantics)
- Idempotency key usage pattern
- When to use simulate before transfer

Compatible with LangChain tool context injection, AutoGen skill description blocks, and plain system-prompt augmentation. No SDK required — agents call the REST API directly using any HTTP client.

Location: served at `https://enclz.com/SKILL.md` (canonical)

### MCP Server

TypeScript package publishing Enclz operations as native MCP tools and resources. Installed once; any MCP runtime discovers tools and resources automatically.

**Package:** `@enclz/mcp`  
**Transport:** stdio (standard MCP)  
**Config:** two env vars — `ENCLZ_API_KEY` (agent API key from registration) and `ENCLZ_API_URL` (default: `https://enclz.com`)

Mutating operations are exposed as MCP **tools**; read-only queries are exposed as MCP **resources** (with `mimeType: application/json`). Tool names are bare — MCP clients namespace by server name, so the redundant `enclz_` prefix from earlier drafts has been dropped. Input schemas are generated from `openapi.json` at build time (`mcp/generated/schemas.ts`); the build fails if `openapi.json` is missing.

**Tools (5):**

| Tool | Wraps endpoint |
|---|---|
| `transfer` | `POST /v1/transfer` |
| `swap` | `POST /v1/swap` |
| `deposit` | `POST /v1/deposit` |
| `withdraw` | `POST /v1/withdraw` |
| `simulate` | `POST /v1/intents/simulate` |

**Resources (3):**

| URI | Wraps endpoint |
|---|---|
| `enclz://balance` | `GET /v1/balance` |
| `enclz://limits`  | `GET /v1/limits` |
| `enclz://history` | `GET /v1/history` |

Tool input schemas mirror the corresponding REST request body. Tool output returns the REST response JSON directly. The MCP server forwards `Idempotency-Key` when the tool input includes one, and surfaces non-2xx backend responses verbatim through MCP's tool-error envelope. Resource reads are not cached across calls.

The agent-facing artifacts (`openapi.json`, `SKILL.md`, MCP tool/resource descriptions) describe the runtime in business terms only — strings like `Anchor`, `Solana`, `PDA`, `Custom program error`, `discriminant`, `onchain_error`, and `0x[0-9a-f]+` literals are forbidden by the artifact linter (`scripts/lint-agent-artifacts.mjs`).

**Claude Desktop config example:**
```json
{
  "mcpServers": {
    "enclz": {
      "command": "npx",
      "args": ["-y", "@enclz/mcp"],
      "env": {
        "ENCLZ_API_KEY": "<agent-api-key>",
        "ENCLZ_API_URL": "https://enclz.com"
      }
    }
  }
}
```

The MCP server is a thin client — it contains no business logic. All enforcement (whitelist, spend limits, nonce) happens in the backend and on-chain exactly as with direct REST calls.

Location: `mcp/`

---

## Program Integration Resources

Distinct from agent integration above. Program-level bindings exist for direct on-chain callers — the Enclz backend itself, programs composing with Enclz via CPI, auditors, and security researchers. AI agents do not consume these; they use the Agent REST API or MCP server.

### `@enclz/sdk` (npm)

TypeScript package re-exporting the Anchor IDL JSON, the `Enclz` TypeScript type, and the program ID for `Program<Enclz>` construction.

**Package:** `@enclz/sdk`
**Install:** `npm install @enclz/sdk @coral-xyz/anchor @solana/web3.js`
**Peer dependencies:** `@coral-xyz/anchor` (the SDK ships only types + IDL JSON, never the Anchor runtime)

```typescript
import { IDL, PROGRAM_ID, type Enclz } from "@enclz/sdk";
import { Program, AnchorProvider } from "@coral-xyz/anchor";

const program = new Program<Enclz>(IDL, AnchorProvider.env());
```

**Versioning:** `version` is single-sourced from `programs/enclz/Cargo.toml` via the IDL `metadata.version`. Bumping the program version automatically bumps the SDK on the next `npm run build:sdk`. Published SDK version always identifies the program version that produced it.

Location: `sdk/` (built artifacts in `sdk/dist/`, source IDL/types in `sdk/src/` are gitignored and regenerated from `target/`).

### On-chain IDL

The same IDL is uploaded on-chain on every cluster Enclz is deployed to via `anchor idl init` / `anchor idl upgrade`. Consumers can fetch it without installing the npm package:

```typescript
import { Program } from "@coral-xyz/anchor";
const idl = await Program.fetchIdl(programId, provider);
const program = new Program(idl!, provider);
```

Useful for explorers, auditors, and integrators who prefer to verify the IDL against on-chain bytecode rather than depend on an npm tarball. Upload requires the program upgrade authority signature, so this channel is owned by whoever holds the deployer keypair.

---

## Web App

### Stack
React 18 + Vite + Tailwind. Solana Wallet Adapter for orchestrator wallet connection (Solflare adapter explicitly registered, others via Wallet Standard auto-detection). Sign-In-With-Solana via `supabase.auth.signInWithWeb3({ chain: 'solana', wallet, statement })` produces the Supabase JWT used to authorize backend calls. On-chain mutations are signed via `useEnclzProgram()` (`src/lib/anchor-client.js`) — a wallet-bound `Program<Enclz>` constructed from `@enclz/sdk`'s IDL. Live on-chain reads use a single `Connection` configured against QuickNode (RPC + WSS).

### Routes

The `:id` route param is the **base58 `group_config_pda`** (every owner has exactly one group, since the PDA is `["group", owner_pubkey]`); `:agentId` is the agent's `agent_wallet_pda`.

```
/                                              landing — no wallet required; product overview, live devnet demo, policy template previews
/demo                                          interactive demo — browse a read-only example fleet dashboard without connecting wallet
/connect                                       wallet connection entry point (redirected here from any authenticated route)
/setup                                         group setup wizard (wallet required from Step 2 onward)
/group/:group_config_pda                       fleet dashboard (agents, spend overview, anomaly feed)
/group/:group_config_pda/agents                agent list — status, daily spend, hourly rate, headroom
/group/:group_config_pda/agents/new            add agent — template selection, limit config, invite code (refuses to advance on empty token registry)
/group/:group_config_pda/agents/:agent_wallet_pda   per-agent detail — audit log, policy, revoke/re-invite
/group/:group_config_pda/whitelist             manage approved addresses — TTL/amount per external entry, renew/top-up actions
/group/:group_config_pda/whitelist/new         add external address — set label, TTL, approved amount
/group/:group_config_pda/tokens                per-group SPL token registry — add/remove mints, view bound-agent count
/group/:group_config_pda/x402                  x402 payment dashboard — approval headline, Configure modal (SPL Token Approve/Revoke), 30-day spend chart, per-endpoint stats, recent payments
/group/:group_config_pda/settings              fleet webhook config
/on<8 alphanumeric>                            public agent activation landing (Markdown); paired with POST /api/v1/onboard/<token>
```

### Group Setup Wizard

```
Step 1: Preview (no wallet)
        → landing page shows product value, live devnet demo, template examples
        → "Create Group" CTA advances to Step 2 and prompts wallet connection

Step 2: Connect Solana wallet (Solflare or any Wallet Standard wallet)
        → user picks a wallet from the modal; Solflare is auto-promoted to the top
        → SPA calls supabase.auth.signInWithWeb3({ chain: 'solana', wallet, statement })
        → wallet signs the SIWS message; Supabase issues a JWT
        → on failure: error banner with retry button; no state written

Step 3: Configure group
        → set group name
        → SPA calls program.methods.initializeGroup(group_name, backend_operator, protocol_fee_wallet, dex_router) via useEnclzProgram() (signed by the orchestrator's wallet)
        → on chain failure: error banner with retry; PDA either created or not (idempotent)

Step 4: Curate the token registry
        → orchestrator opens /group/:group_config_pda/tokens and adds at least one SPL mint
        → SPA pastes a mint pubkey, reads decimals via getMint(connection, mint), POSTs {mint, symbol, decimals, label} to /v1/orchestrator/groups/:group_config_pda/tokens
        → backend re-verifies decimals against on-chain getMint and rejects mismatches
        → seed-defaults endpoint exists for one-click bulk insert of canonical devnet mints
        → registry is required: agent creation refuses to advance with an empty registry

Step 5: Add first agent
        → orchestrator enters display name, selects policy template, picks a token from the registry
        → SPA calls program.methods.addAgent(...) via useEnclzProgram() (signed by the orchestrator's wallet), passing the selected mint into the AgentWallet ATA seed
        → after `confirmed`, SPA POSTs {tx_sig, agent_wallet_pda, token_mint} to /v1/orchestrator/groups/:group_config_pda/agents to mint the invitation code
        → backend asserts AgentWallet.mint == token_mint and the mint is registered before issuing the code
        → on chain failure: error banner with retry
        → system displays one-time invitation URL (https://enclz.com/on<token>) — the operator shares this string verbatim with the agent

Step 6: Configure whitelist
        → DEX router auto-whitelisted (entry_type 2, permanent) at Step 3 via initialize_group's dex_router argument (resolved from JUPITER_PROGRAM_ID)
        → orchestrator adds external service addresses: label + TTL (days) + approved amount — signed via useEnclzProgram()
        → intra-group agent entry already present from Step 5 (entry_type 0, permanent, auto-added by add_agent)

Step 7 (optional): Approve x402 budget
        → orchestrator opens /group/:group_config_pda/x402
        → Configure modal builds one SPL Token Approve (or Revoke for 0) per dirty currency targeting VITE_X402_DELEGATE_PUBKEY
        → orchestrator's wallet signs and submits the transaction
        → after `confirmed`, the dashboard refetches on-chain delegation state and renders the new headline
```

---

## External Integrations

### Solana Wallet Adapter
Package: `@solana/wallet-adapter-react` + `@solana/wallet-adapter-react-ui` + `@solana/wallet-adapter-wallets`.
Used by: Web app for wallet connection and on-chain signing. `WalletProviders` wraps the app with `ConnectionProvider` (QuickNode RPC + WSS) and `WalletProvider` (Solflare adapter explicitly registered, others via Wallet Standard auto-detection).
The `useEnclzProgram()` hook (`src/lib/anchor-client.js`) returns a `Program<Enclz>` whose provider's wallet is the connected adapter — used to sign every on-chain orchestrator mutation.

### Jupiter API
Base URL: `https://lite-api.jup.ag/swap/v1` (override via `JUPITER_BASE_URL`, e.g. `https://api.jup.ag/swap/v1` for paid mirrors)
- `GET /quote?inputMint=&outputMint=&amount=&slippageBps=50`
- `POST /swap-instructions` → returns `{ swapInstruction: { data: <base64>, ... }, ... }`. The backend base64-decodes `swapInstruction.data` into the `routeData` Buffer the on-chain program CPI-invokes — serializing the whole JSON envelope is invalid.

Token mints are NOT hard-coded; each group's `group_tokens` registry is the source of truth for which mints agents may bind to. The default-seed endpoint is the only place canonical devnet mints are referenced.

### x402 Facilitator
The backend never talks to the facilitator directly. The agent embeds the backend's `follow_up.headers` (`X-PAYMENT` for v1, `PAYMENT-SIGNATURE` for v2) on its retry to the resource server, and the facilitator on the resource server's side asserts the transaction is fully signed (signing as fee payer itself) and submits via `sendAndConfirmSignedTransaction`. Enclz's role ends at the partial-sign.

### Solana RPC
Provider: **QuickNode**
Calls: `sendTransaction`, `getAccountInfo`, `getTokenAccountBalance`, `getSignaturesForAddress`, `getParsedTransactions`, `getMultipleAccountsInfo`, `onAccountChange` (websocket via `VITE_QUICKNODE_WSS_URL`). The SPA reads on-chain data directly — no Fastify proxy is in the dashboard read path. When no `VITE_QUICKNODE_RPC_URL` is configured, the SPA falls back to `https://api.devnet.solana.com` without a `wsEndpoint` (subscriptions disabled).
