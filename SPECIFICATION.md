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
| MCP Server | TypeScript · `@modelcontextprotocol/sdk` · stdio transport |
| DEX aggregator | Jupiter API v6 |
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
  index.js                   // Fastify entry point — registers routes, force-instantiates Solana client at boot
  routes/
    agent.js                 // all /v1/* agent endpoints (register, transfer, swap, deposit, withdraw, simulate, balance, limits, history, webhooks)
    orchestrator.js          // four credential-minting endpoints (see §"Orchestrator Endpoints")
  lib/
    auth.js                  // resolveAgentFromKey · resolveUserFromJwt · requireGroupAccess preHandler
    db.js                    // getServiceClient() singleton (service-role, bypasses RLS); throws at module load if Supabase env vars are missing
    solana.js                // cached getConnection() · getOperator() · getProgram()
    intents.js               // executeTransfer · executeSwap · executeLendingOp (with one nonce-mismatch resync + retry)
    jupiter.js               // Jupiter v6 quote + swap-instructions wrapper
    onchain-fleet.js         // getGroupConfig · getAgentWallet · listAgentsForGroup · decodeFixed32
    onchain-verify.js        // confirmTx · verifyAgentWallet · verifyWhitelistEntry (used by the agent-creation endpoint)
    pda.js                   // groupConfigPda · agentWalletPda · whitelistEntryPda derivation
    policy.js                // computeFee (10 bps) — only function called on the request path. TEMPLATES, applyTemplate, preflightCheck, anomalyCheck remain as helper exports for tests/tooling but are NOT invoked by route handlers.
    webhooks.js              // dispatchWebhook (fire-and-forget HMAC-signed events, refuses redirects)
    url-safety.js            // validateWebhookUrl (DNS rebinding protection)
    idempotency.js           // reserveOrAwait → complete/release pattern; PK is (key, agent_wallet_pda)
    anchor-errors.js         // façade re-exporting the program's 6000–6013 discriminants from shared/anchor-errors.js
    crypto.js                // generateApiKey · hashSecret · verifySecret · generateInviteCode · signWebhookPayload

src/                         // SPA — see §"Web App"
  lib/
    anchor-client.js         // useEnclzProgram() — wallet-bound Anchor Program for orchestrator on-chain mutations
    onchain-fleet.js         // listAgentsForGroup · listWhitelistEntries · getGroupConfig · getAgentWallet (mirror of server/lib/onchain-fleet.js)
    api.js                   // thin fetch wrapper attaching the Supabase JWT, used only for the four Fastify endpoints

mcp/                         // MCP server — separate package, client of Agent REST API (NOT YET IMPLEMENTED)
  index.ts                   // entry point — creates MCP server, registers tools, starts stdio transport
  tools.ts                   // tool definitions: input schemas + handler functions
  client.ts                  // thin HTTP client wrapping Agent REST API (uses ENCLZ_API_URL + ENCLZ_API_KEY)
  package.json               // { "name": "@enclz/mcp-server", "bin": { "enclz-mcp": "./dist/index.js" } }
```

### Registry Data Model

The chain is the source of truth for groups, agents, and whitelist entries — there are no mirror tables. The database holds only state the chain doesn't model: bcrypt-hashed credentials, one-time invitation codes, webhook subscribers, and the idempotency cache. All four surviving tables are keyed on base58 PDAs (text), not UUIDs, and have RLS enabled with no policies attached (service-role only).

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
```

There is no `groups`, `agents`, or `whitelist_entries` table. The SPA enumerates fleet metadata via Anchor account fetchers (`program.account.agentWallet.all` filtered by `memcmp` on the `group` field; `program.account.whitelistEntry.all` filtered on `added_by`) — see `src/lib/onchain-fleet.js`. The backend reads the same accounts on demand via `server/lib/onchain-fleet.js`.

### Policy Enforcement

Limits and whitelist enforcement live exclusively on-chain. The backend does NOT run a server-side preflight on the request path — it submits the Anchor instruction directly and surfaces the program's custom error codes (6000–6013) as HTTP errors via `parseAnchorError`. The chain is the only authority; mirroring the rules in JavaScript was a source of drift, so it was removed.

`server/lib/policy.js` exports `computeFee` (10 bps protocol fee) — the only function called on the request path. `TEMPLATES`, `applyTemplate`, `preflightCheck`, and `anomalyCheck` are kept as helper exports for tests and tooling but are NOT invoked by route handlers. Anomaly events (`policy.limit_threshold`, `policy.whitelist_amount_threshold`, `policy.whitelist_voided`, etc.) require these helpers to be re-attached to the post-execution path; until then those webhooks do not fire.

After a confirmed `execute_transfer` / `execute_swap` / `execute_lending_op`, the backend re-fetches the on-chain `AgentWallet` account to populate `daily_remaining` / `hourly_tx_remaining` in the response — there is no JS mirror to read.

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

Anomaly thresholds (`policy.limit_threshold`, `policy.whitelist_amount_threshold`, `policy.whitelist_voided`, `policy.limit_exceeded_attempt`, `policy.whitelist_violation`, `policy.whitelist_expiring`) and the `payment.received` webhook are not yet wired on the request path. They require, respectively, calling `anomalyCheck` after re-fetching on-chain state, hooking the Anchor error path for rejection events, a scheduled job for TTL-expiring alerts, and a wallet monitor (`onAccountChange` subscriptions to every agent ATA).

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

The JWT is issued by `supabase.auth.signInWithWeb3({ chain: 'solana', wallet, statement })` — the same Sign-In-With-Solana ceremony used by the browser SPA and headless Node scripts. There is no separate orchestrator API key.

`requireGroupAccess` preHandler:
1. Validates the JWT via `db.auth.getUser(token)` (5-minute in-memory cache).
2. Extracts the wallet pubkey from `user.user_metadata.custom_claims.address` / `user.user_metadata.address`.
3. Computes `groupConfigPda(walletPubkey).toBase58()` and compares it to the URL's `:group_config_pda` param.
4. Rejects 403 on mismatch, 401 on malformed pubkey claim.

There is no database join for ownership — the chain's PDA derivation IS the proof.

---

### Orchestrator Endpoints

The orchestrator REST surface is intentionally minimal: only credential-minting endpoints. Every other orchestrator action — creating a group, creating an agent on-chain, adding/renewing/removing whitelist entries, updating per-agent limits, emergency withdraw, rotating the backend operator — is performed by the orchestrator's wallet signing the corresponding Anchor instruction directly (Solflare in the browser, raw keypair in a Node script). The chain account is the persisted state; there is no mirror row to write.

URL path params:
- `:group_config_pda` — base58 `GroupConfig` PDA (every owner has exactly one group, since the PDA is `["group", owner_pubkey]`).
- `:agent_wallet_pda` — base58 `AgentWallet` PDA.

#### `POST /v1/orchestrator/groups/:group_config_pda/agents`

Mint a one-time invitation code for an agent the orchestrator has just created on-chain. The orchestrator's wallet must have already signed and confirmed `add_agent`; this endpoint verifies the resulting `AgentWallet` PDA belongs to the route's group, then issues the invitation.

```js
// Request
{
  "tx_sig":           "string",   // confirmed signature of the add_agent instruction
  "agent_wallet_pda": "string"    // base58
}

// Response 200
{
  "agent_wallet_pda": "string",
  "invitation_code":  "string"   // one-time, expires 24h, shown once
}

// Response 400
{ "error": "onchain_mismatch", "message": "agent_wallet_pda does not belong to :group_config_pda" }
```

#### `POST /v1/orchestrator/groups/:group_config_pda/agents/:agent_wallet_pda/rotate-key`

Issue a new invitation code for an existing agent (e.g., after a suspected key compromise). Atomically revokes any prior credential and mints a fresh code.

```js
// Response 200
{ "agent_wallet_pda": "string", "invitation_code": "string" }
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

### Agent Endpoints

#### `POST /v1/register`

Exchange one-time invitation code for an API key. Code invalidated immediately after use.

```js
// Request
{ "invitation_code": "string" }

// Response 200
{ "agent_wallet_pda": "string", "api_key": "string" }   // api_key shown once

// Response 400
{ "error": "invalid_invitation_code", "message": "Code expired, already used, or not found." }
```

#### `POST /v1/transfer`

Transfer tokens to a whitelisted address.

```js
// Request
{
  "to":               "string",   // base58 Solana address — must be whitelisted
  "amount":           number,     // token units (e.g. 2.50 for 2.50 USDC)
  "token":            "string",   // "USDC" | "SOL" | "PUSD"
  "memo":             "string",   // optional — stored with transaction
  "task_id":          "string",   // optional — orchestrator-defined task reference
  "idempotency_key":  "string"    // optional; same key = same response, no re-execution
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

Swap tokens via the whitelisted Jupiter router. Swap router is whitelisted at group initialization.

```js
// Request
{
  "from_token":       "string",
  "to_token":         "string",
  "amount":           number,
  "task_id":          "string",    // optional
  "idempotency_key":  "string"     // optional
}

// Response 200
{
  "tx_sig":              "string",
  "status":              "confirmed",
  "received_amount":     number,
  "rate":                number,
  "daily_remaining":     number,
  "hourly_tx_remaining": number
}
```

#### `POST /v1/deposit`

Deposit tokens into a whitelisted lending protocol.

> **Current shape (pass-through CPI).** The backend does not yet wrap a Kamino/Save adapter, so the agent supplies the lending program's pubkey AND the prebuilt CPI instruction bytes. The backend validates the lending program is whitelisted (`entry_type = 2`) for the agent's group and submits `execute_lending_op(0, ...)` to the Enclz program, which CPI-invokes the lending program with `cpi_data`. The "no SDK / no crypto knowledge required" promise lands when a backend adapter is added (see deferred work).

```js
// Request
{
  "lending_program":  "string",   // base58 — must be whitelisted entry_type = 2 for this agent's group
  "amount":           number,
  "cpi_data":         "string",   // base64-encoded CPI instruction bytes for the lending program
  "task_id":          "string",   // optional
  "idempotency_key":  "string"    // optional
}

// Response 200
{
  "tx_sig":               "string",
  "status":               "confirmed",
  "lending_program":      "string",
  "deposited_amount":     number,
  "daily_remaining":      number,
  "hourly_tx_remaining":  number
}

// Response 403
{ "error": "whitelist_violation" | "daily_limit_exceeded" | "per_tx_limit_exceeded" | "hourly_cap_exceeded" }
```

`current_apy` is not yet populated (no APY oracle wired); will be added when a backend adapter lands.

#### `POST /v1/withdraw`

Withdraw tokens from a whitelisted lending protocol. Same pass-through CPI shape as deposit.

```js
// Request
{
  "lending_program":  "string",
  "amount":           number,
  "cpi_data":         "string",   // base64-encoded CPI instruction bytes
  "task_id":          "string",
  "idempotency_key":  "string"
}

// Response 200
{
  "tx_sig":               "string",
  "status":               "confirmed",
  "lending_program":      "string",
  "received_amount":      number,
  "daily_remaining":      number,
  "hourly_tx_remaining":  number
}

// Response 403
{ "error": "whitelist_violation" | "insufficient_balance" }
```

`yield_earned` is not yet populated (no yield-tracking wired); will be added with the backend adapter.

#### `POST /v1/intents/simulate`

Dry-run check: would this transfer/swap succeed? No transaction submitted, no state changed.

```js
// Request — same body shape as /v1/transfer or /v1/swap
{
  "action":  "transfer" | "swap",
  "to":      "string",           // for transfer
  "amount":  number,
  "token":   "string",
  "from_token": "string",        // for swap
  "to_token":   "string"         // for swap
}

// Response 200
{
  "would_succeed":      boolean,
  "reason":             null | "whitelist_violation" | "daily_limit_exceeded" | "per_tx_limit_exceeded" | "hourly_cap_exceeded" | "insufficient_balance",
  "daily_remaining":    number,
  "hourly_tx_remaining": number,
  "estimated_fee":      number    // protocol fee in USDC
}
```

#### `GET /v1/balance`

Current balances and spend headroom.

```js
// Response 200
{
  "balances": { "USDC": number, "SOL": number, "PUSD": number },
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
| `submission_failed` | 503 | Transient RPC or network failure; retry with same idempotency key |
| `nonce_mismatch` | 500 | Backend operator nonce out of sync with on-chain state — backend resyncs and retries internally |
| `internal_error` | 500 | Unexpected backend or chain error |

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

TypeScript package publishing Enclz operations as native MCP tools. Installed once; any MCP-enabled runtime discovers tools automatically.

**Package:** `@enclz/mcp-server`  
**Transport:** stdio (standard MCP)  
**Config:** two env vars — `ENCLZ_API_KEY` (agent API key from registration) and `ENCLZ_API_URL` (default: `https://api.enclz.com`)

**Tools exposed:**

| Tool name | Maps to | Description |
|---|---|---|
| `enclz_transfer` | `POST /v1/transfer` | Transfer tokens to a whitelisted address |
| `enclz_swap` | `POST /v1/swap` | Swap tokens via Jupiter |
| `enclz_deposit` | `POST /v1/deposit` | Deposit into a whitelisted lending protocol |
| `enclz_withdraw` | `POST /v1/withdraw` | Withdraw from a whitelisted lending protocol |
| `enclz_simulate` | `POST /v1/intents/simulate` | Dry-run: would this transfer/swap succeed? |
| `enclz_balance` | `GET /v1/balance` | Token balances and spend headroom |
| `enclz_limits` | `GET /v1/limits` | Full spend policy and whitelist |
| `enclz_history` | `GET /v1/history` | Paginated transaction log |

Each tool input schema mirrors the corresponding REST request body. Tool output returns the REST response JSON directly, plus a human-readable `summary` field for LLM reasoning.

**Claude Desktop config example:**
```json
{
  "mcpServers": {
    "enclz": {
      "command": "npx",
      "args": ["-y", "@enclz/mcp-server"],
      "env": {
        "ENCLZ_API_KEY": "<agent-api-key>",
        "ENCLZ_API_URL": "https://api.enclz.com"
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
/group/:group_config_pda/agents/new            add agent — template selection, limit config, invite code
/group/:group_config_pda/agents/:agent_wallet_pda   per-agent detail — audit log, policy, revoke/re-invite
/group/:group_config_pda/whitelist             manage approved addresses — TTL/amount per external entry, renew/top-up actions
/group/:group_config_pda/whitelist/new         add external address — set label, TTL, approved amount
/group/:group_config_pda/settings              fleet webhook config
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

Step 4: Add first agent
        → orchestrator enters display name, selects policy template
        → SPA calls program.methods.addAgent(...) via useEnclzProgram() (signed by the orchestrator's wallet)
        → after `confirmed`, SPA POSTs {tx_sig, agent_wallet_pda} to /v1/orchestrator/groups/:group_config_pda/agents to mint the invitation code
        → on chain failure: error banner with retry
        → system displays one-time invitation code

Step 5: Configure whitelist
        → DEX router auto-whitelisted (entry_type 2, permanent) at Step 3 via initialize_group's dex_router argument
        → orchestrator adds external service addresses: label + TTL (days) + approved amount (USDC) — signed via useEnclzProgram()
        → intra-group agent entry already present from Step 4 (entry_type 0, permanent, auto-added by add_agent)
```

---

## External Integrations

### Solana Wallet Adapter
Package: `@solana/wallet-adapter-react` + `@solana/wallet-adapter-react-ui` + `@solana/wallet-adapter-wallets`.
Used by: Web app for wallet connection and on-chain signing. `WalletProviders` wraps the app with `ConnectionProvider` (QuickNode RPC + WSS) and `WalletProvider` (Solflare adapter explicitly registered, others via Wallet Standard auto-detection).
The `useEnclzProgram()` hook (`src/lib/anchor-client.js`) returns a `Program<Enclz>` whose provider's wallet is the connected adapter — used to sign every on-chain orchestrator mutation.

### Jupiter API
Base URL: `https://quote-api.jup.ag/v6`
- `GET /quote?inputMint=&outputMint=&amount=&slippageBps=50`
- `POST /swap` with quote → swap transaction (base64 serialized)

Token mints (mainnet):
- USDC: `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`
- SOL (wrapped): `So11111111111111111111111111111111111111112`

### Solana RPC
Provider: **QuickNode**
Calls: `sendTransaction`, `getAccountInfo`, `getTokenAccountBalance`, `getSignaturesForAddress`, `onAccountChange` (websocket)
