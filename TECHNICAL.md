# Technical Architecture

## Overview

agent-budget is an OpenClaw skill that provides financial observability for autonomous agents. It answers the question "what did my agent spend?" by detecting payment tool calls, logging them to a tamper-evident local ledger, and presenting them through a local web dashboard.

The system has three layers:

1. **Detection** — pattern matching on tool calls to identify payments
2. **Storage** — append-only, hash-chained JSONL ledger with deduplication
3. **Presentation** — HTTP API + single-page dashboard, localhost only

All three layers run locally. Nothing leaves the machine.

---

## Security Model

Security is not a feature of agent-budget — it is the reason agent-budget exists. An agent spending money on your behalf without accountability is a fundamental trust problem. The security model is designed around three principles:

### 1. The log is the source of truth

Every financial claim the system makes traces back to a record in `transactions.jsonl`. There is no in-memory state, no database, no cache that could diverge from the log. If the log says it happened, it happened. If the log doesn't say it, it didn't.

### 2. The log is tamper-evident

Each transaction record contains a `prev_hash` field: the SHA-256 hash of the previous record's complete JSON line. The first record references `sha256:genesis`. This creates a hash chain — a lightweight tamper-evidence mechanism.

```
Record 0:  prev_hash = "sha256:genesis"
Record 1:  prev_hash = sha256(JSON line of Record 0)
Record 2:  prev_hash = sha256(JSON line of Record 1)
...
```

If anyone modifies, deletes, or reorders a historical record, the chain breaks. The `verifyChain()` function walks the entire log and reports exactly where a break occurs. The dashboard shows chain status in the header. The agent can verify integrity via `query-log.sh --verify`.

**What this protects against:**
- A compromised agent silently editing past records to hide spending
- Manual tampering with the log file
- Corruption from partial writes (each line is independently valid JSON; the chain catches if a line was altered after being written)

**What this does NOT protect against:**
- An attacker with file system access who rewrites the entire log and recomputes all hashes. This is a local file, not a distributed ledger. For v0.1 (observe and report), this is acceptable. v0.4 introduces signed receipts that provide cryptographic non-repudiation.

### 3. Minimal attack surface

agent-budget is designed to need as little access as possible:

| Capability | Required? | Why |
|---|---|---|
| Read tool call results | Yes | Core function — detecting payments |
| Write to its own data directory | Yes | Transaction log, tracked tools, submissions |
| Network access (outbound) | Fetch only | Fetches `api.agent-budget.net/patterns.json` hourly — no payment data included |
| Network access (inbound) | Localhost only | Dashboard server binds to `127.0.0.1`, never `0.0.0.0` |
| Wallet keys or credentials | **No** | Observes results, never initiates payments |
| Shell execution (exec) | **No** | Shell scripts are tools the agent calls, not the skill itself |
| Access to other skills' state | **No** | Only sees tool call results passed through the hook |

### File Permissions

- `transactions.jsonl` is created with mode `0600` (owner read/write only)
- `tracked-tools.json` and `submissions.jsonl` are created with mode `0600`
- The dashboard server binds to `127.0.0.1` — unreachable from other machines on the network

### Data Sanitization

Tool arguments are truncated to 200 characters in the log. Full arguments are never stored. This prevents accidental logging of:
- API keys passed as tool arguments
- Full wallet addresses (only fragments appear in truncated summaries)
- Long prompt content that might contain sensitive user data

Transaction hashes and idempotency keys are stored in full because they are needed for deduplication and verification.

### No Credential Storage

agent-budget never stores, reads, or has access to:
- Wallet private keys
- API keys (except fragments that appear in truncated tool argument summaries)
- Session tokens
- User credentials of any kind

It operates purely on the output side of tool calls — it sees what happened after the fact, not the credentials used to make it happen.

### Deduplication as a Safety Mechanism

Duplicate payments are a real risk with autonomous agents. Retries, loops, and race conditions can cause the same payment to execute multiple times. agent-budget deduplicates on two axes:

- **Transaction hash**: If a tx_hash already exists in the log, the new record is rejected
- **Idempotency key**: If an idempotency_key already exists, the record is rejected

This prevents double-counting in reports. It does NOT prevent the duplicate payment itself (that requires `before_tool_call` enforcement in v0.2), but it ensures the owner's view of spending is accurate.

### Loop Detection

Each transaction records a `context.input_hash` — a SHA-256 hash of the tool arguments. If an agent calls the same expensive service with identical inputs repeatedly, this shows up clearly in the log: multiple records with the same `input_hash`. The dashboard can surface this pattern in future versions, and v0.2 budget controls can use it as a trigger ("alert if the same input_hash appears more than N times in an hour").

### Failure Classification

Not all failed payments are equal:

- **`pre_payment`**: The tool call failed before money moved (insufficient funds, network error, service unavailable). No financial impact.
- **`post_payment`**: The payment was taken but the tool call still returned an error (service accepted payment but returned bad data, timeout after payment). Financial impact — the owner paid but didn't get what they paid for.

This distinction is critical for dispute resolution and for understanding actual spend vs. reported spend.

---

## Detection Architecture

### Detector Registry

Payment detection uses a registry of detector functions, evaluated in priority order:

```
1. User-tracked tools     — Custom patterns defined by the user (highest priority)
2. agent-wallet-cli       — Detects x402 subcommand in args
3. v402                   — Detects v402-pay/v402-http script calls
4. ClawRouter             — Detects by tool name (requires name match, not just cost/price fields)
5. payment-skill          — Detects pay script calls
6. Heuristic              — Broad pattern matching on tool names and argument shapes
7. Generic x402           — Catch-all for any x402 payment confirmation in response body
```

The first detector to return a match wins. This ordering ensures specific, high-confidence detectors fire before broad heuristic ones.

### Heuristic Detection

The heuristic detector catches payment tools that don't match any specific detector. It uses three signal layers:

**Tool name matching**: Regex against known payment-related tool names:
```
stripe|paypal|venmo|square|shopify|braintree|crypto_transfer|
send_money|donate|checkout|purchase|buy|invoice|subscribe|billing
```

**Argument scanning**: Looks for co-occurrence of monetary keywords (`amount`, `price`, `total`, `cost`, `fee`) with at least one of:
- Currency signals (`currency`, `usd`, `eur`, `usdc`, `btc`)
- Recipient signals (`recipient`, `payee`, `wallet_address`, `iban`)
- Payment method signals (`payment_method`, `card`, `token`, `pm_`)

**Result confirmation**: Checks for success markers (`succeeded`, `completed`, `charged`, `payment confirmed`) and transaction ID patterns (`ch_`, `pi_`, `pm_`, `0x[a-f0-9]{64}`, `txn_`).

**False positive prevention**: The heuristic detector requires either a tool name match OR (argument signals AND result confirmation AND non-zero amount). A tool that returns price data without actually charging anything will not trigger.

### User-Tracked Tools

Users can add custom tool patterns via the dashboard's "Track Tools" tab or the `POST /api/tracked-tools` endpoint. Patterns can be exact tool names or regex. User-tracked patterns have the highest detector priority — they fire before any built-in detector.

### Community Patterns

On startup and every hour, the skill fetches `api.agent-budget.net/patterns.json` and caches it to `data/community-patterns.json`. `detectUserTracked()` merges local and community patterns at detection time — local patterns take priority on any conflict. This allows the published pattern list to expand without requiring a skill update.

Community patterns are maintained at [github.com/mattpolly/agent-budget.net](https://github.com/mattpolly/agent-budget.net). The fetch is read-only; no payment data or transaction records are included in any request to that host. The only outbound write is an opt-in pattern submission from the dashboard (`data/submissions.jsonl` records what was sent locally).

### Adding New Detectors

To support a new payment tool:

1. Document the tool's signature in `references/payment-tools.md`
2. Add a detector function in `server/detectors.js`
3. Register it in the `detectors` array (before the heuristic catch-all)

Or, for quick user-level support: add a tracked tool pattern via the dashboard.

---

## Data Model

### Transaction Record

Each record is a single JSON object, one per line in `transactions.jsonl`:

```json
{
  "id": "txn_1710523847_a3f2",
  "timestamp": "2026-03-15T14:30:47Z",
  "prev_hash": "sha256:e3b0c44298fc1c149afbf4c8996fb924...",
  "service": {
    "url": "https://alphaclaw.example.com/hunt",
    "name": "AlphaClaw Coordinator",
    "category": "research"
  },
  "amount": {
    "value": "0.05",
    "currency": "USDC",
    "chain": "base"
  },
  "tx_hash": "0xabc123...",
  "idempotency_key": "req_unique_123",
  "context": {
    "session_key": "agent:main:telegram:+1234567890",
    "skill": "stock-research",
    "user_request": "get me an AAPL stock report",
    "tool_name": "agent-wallet-cli",
    "tool_args_summary": "x402 POST https://alphaclaw.example.com/hunt --max-am…",
    "input_hash": "a1b2c3d4e5f6..."
  },
  "execution_time_ms": 1250,
  "failure_type": null,
  "status": "confirmed",
  "source": "auto"
}
```

### Field Reference

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique identifier (`txn_{unix_timestamp}_{random_hex}`) |
| `timestamp` | ISO 8601 | When the payment was detected |
| `prev_hash` | string | SHA-256 of the previous record's JSON line (`sha256:genesis` for first) |
| `service.url` | string\|null | Endpoint URL that was paid |
| `service.name` | string | Human-readable service name (extracted from URL hostname or tool output) |
| `service.category` | string\|null | Optional category (e.g., "research", "llm-routing") |
| `amount.value` | string | Payment amount as a decimal string |
| `amount.currency` | string | Currency or token symbol (e.g., "USDC", "USD", "ETH") |
| `amount.chain` | string\|null | Settlement chain/network (e.g., "base", "solana") |
| `tx_hash` | string\|null | On-chain transaction hash for verification |
| `idempotency_key` | string\|null | Request idempotency key (prevents duplicate logging on retries) |
| `context.session_key` | string\|null | OpenClaw session identifier |
| `context.skill` | string\|null | Which skill triggered the payment |
| `context.user_request` | string\|null | The human request that initiated the chain |
| `context.tool_name` | string\|null | Which wallet/payment tool executed the payment |
| `context.tool_args_summary` | string\|null | Truncated tool arguments (max 200 chars) |
| `context.input_hash` | string\|null | SHA-256 of tool arguments (for loop detection) |
| `execution_time_ms` | number\|null | Time from tool call start to completion |
| `failure_type` | string\|null | `null` (success), `"pre_payment"`, or `"post_payment"` |
| `status` | string | `"confirmed"`, `"failed"`, or `"pending"` |
| `source` | string | `"auto"` (detected) or `"manual"` (user-entered) |

### Why JSONL

- **Append-only writes** are safe against corruption — no need to parse and rewrite the entire file
- **Each line is independently parseable** — a corrupt line doesn't destroy the whole log
- **Easy to grep, tail, and stream** — standard Unix tool compatibility
- **Natural fit for an audit log** — chronological, immutable-by-convention

### Storage Locations

| File | Purpose | Created |
|---|---|---|
| `data/transactions.jsonl` | Transaction log | On first payment detection |
| `data/tracked-tools.json` | User-defined tool patterns | On first tracked tool addition |
| `data/community-patterns.json` | Cached community pattern list | On first successful sync |
| `data/install-id.json` | Stable per-install UUID (opt-in submissions only) | On first opt-in submission |
| `data/submissions.jsonl` | Local log of patterns sent to maintainers | On first opt-in submission |
| `data/dashboard.pid` | Dashboard server PID | On dashboard start |

All files are created with mode `0600` (owner read/write only).

---

## Server Architecture

### API Endpoints

All endpoints are served on `127.0.0.1:18920` (configurable via `AGENT_BUDGET_PORT`).

| Method | Path | Description |
|---|---|---|
| GET | `/` | Dashboard HTML |
| GET | `/api/transactions` | List transactions (filterable by `from`, `to`, `service`, `skill`, `status`, `source`) |
| GET | `/api/summary/daily` | Daily spending rollups |
| GET | `/api/summary/weekly` | Weekly spending rollups |
| GET | `/api/summary/monthly` | Monthly spending rollups |
| GET | `/api/summary/by-service` | Spending grouped by service |
| GET | `/api/summary/by-skill` | Spending grouped by skill |
| GET | `/api/summary/by-tool` | Spending grouped by payment tool |
| GET | `/api/balance` | Wallet balance (placeholder — not yet integrated) |
| GET | `/api/verify` | Hash chain integrity check |
| GET | `/api/export` | Export as CSV or JSON (`format=csv\|json`) |
| GET | `/api/tracked-tools` | List user-tracked tool patterns |
| POST | `/api/transactions` | Log a manual transaction |
| POST | `/api/tracked-tools` | Add a tracked tool pattern (with optional `send_to_maintainers`) |
| DELETE | `/api/tracked-tools/:pattern` | Remove a tracked tool pattern |

### No External Dependencies

The server uses Node.js built-in `http` module. No Express, no Hono, no npm packages. The dashboard is a single HTML file with embedded CSS and JavaScript — no build step, no framework, no CDN loads.

This is a deliberate choice:
- **No supply chain risk** — no `node_modules`, no transitive dependencies to audit
- **No build tooling** — the code that ships is the code that runs
- **Auditable** — the entire server is one file (~200 lines); the entire dashboard is one file (~400 lines)

### Localhost Binding

The server binds to `127.0.0.1`, not `0.0.0.0`. This means:
- Accessible from the local machine only
- Not reachable from other machines on the network
- Not reachable from the internet
- No authentication needed (the binding IS the access control)

---

## Hook Integration

### Current: tool_result_persist (v0.1)

agent-budget uses the `tool_result_persist` hook, which fires synchronously after every tool call completes. This hook is available in OpenClaw today and does not require any core changes.

The hook receives the tool name, arguments, and result. agent-budget passes these to the detector registry, and if a payment is detected, appends a record to the log.

**Limitation**: This hook fires after the tool has already executed. agent-budget can observe and log, but cannot block or modify the payment. This is by design for v0.1.

### Future: before_tool_call (v0.2)

The `before_tool_call` hook is defined in OpenClaw's `plugins/hooks.js` and exported via `createHookRunner()`, but is not currently wired into the tool execution pipeline. Multiple open issues request this (#5943, #7597, #12311, #30504).

When wired, this hook would allow agent-budget to:
- Check proposed payments against budget limits before they execute
- Block payments that exceed caps or hit blocklisted services
- Require human approval for payments above a threshold
- Pause tool execution and resume after approval

This is the architectural path to v0.2 budget controls.

---

## Design Decisions

### Why observe-only in v0.1?

1. **Works today** — `tool_result_persist` is available now; `before_tool_call` is not
2. **Usage data informs controls** — real spending patterns from v0.1 will show what controls are actually needed
3. **Visibility is independently valuable** — most agent owners don't even know what their agent spends
4. **Lower risk** — a logging bug loses visibility; an enforcement bug blocks legitimate payments

### Why hash chain instead of a database?

- A JSONL file is simpler, more portable, and easier to audit than SQLite or any database
- The hash chain provides tamper evidence without requiring a database's ACID guarantees
- Append-only writes are naturally safe against corruption
- The log is small enough to read entirely into memory for queries (even 100,000 transactions is ~50MB)
- If scale becomes a problem, the architecture supports sharding by time period

### Why no authentication on the dashboard?

The dashboard binds to `127.0.0.1`. Only processes on the local machine can reach it. Adding authentication would add complexity without adding security — if an attacker is running code on your machine, they can read the JSONL file directly regardless of dashboard auth.

### Why copy-install instead of symlink?

OpenClaw's `resolveContainedSkillPath` resolves symlinks and rejects paths that escape the skills directory root. This is a security feature — it prevents a skill from symlinking to arbitrary file system locations. agent-budget respects this by requiring a copy installation.

---

## Testing

32 tests across three suites:

- **transactions** (15 tests): Append, hash chain, deduplication (tx_hash and idempotency_key), read, query, filter by source, summarize, group by, verify chain, CSV export, input hashing
- **detectors** (11 tests): agent-wallet-cli, v402, generic x402, payment-skill, heuristic (stripe name, payment args + confirmation, false positive avoidance), failure type classification, idempotency key extraction
- **server API** (6 tests): POST transaction, duplicate rejection, GET transactions, POST/GET/DELETE tracked tools

```bash
node --test test/test.js
```
