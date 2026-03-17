# agent-budget

An OpenClaw skill that tracks every payment your agent makes and presents it as a clear, reviewable transaction statement.

---

## Problem

OpenClaw agents can now hold wallets and spend real money. Multiple skills enable this — agent-wallet-cli, v402, ClawRouter, Vault-0, payment-skill, and others. But no skill answers the basic question every agent owner has:

**"What did my agent spend my money on?"**

The existing monitoring dashboards (ClawMetry, openclaw-dashboard) track LLM inference costs — how many tokens were consumed and what the API calls cost. They don't track what the agent *bought* in the real world via x402 micropayments or other payment protocols.

The wallet skills have transaction history commands, but those show raw blockchain data — addresses and amounts. They don't show "your agent paid $0.05 to AlphaClaw for an alpha synthesis report while working on your morning research task."

agent-budget fills this gap. It's the credit card statement for your agent.

---

## What It Does (v0.1)

One thing: **observe and report**.

The skill watches for tool calls that involve payment, captures the details after they complete, and writes structured transaction records to a local log. A local web dashboard presents these records as a human-readable statement.

### Transaction Capture

For each payment the agent makes, agent-budget logs:

- **Timestamp**: When the payment occurred
- **Service**: What was paid (endpoint URL, service name if resolvable)
- **Amount**: How much, in what currency/token
- **Chain/Network**: Where the payment settled (Base, Solana, etc.)
- **Transaction ID**: On-chain tx hash for verification
- **Triggering task**: What the agent was doing when it made the payment (session context, skill name, user request that initiated the chain)
- **Tool used**: Which wallet/payment skill executed the payment (agent-wallet-cli, v402, ClawRouter, etc.)
- **Status**: Success, failure, or pending

### Dashboard

A local web app served on localhost (same pattern as existing OpenClaw dashboards). Shows:

- **Transaction list**: Chronological feed of all payments, searchable and filterable
- **Daily/weekly/monthly rollups**: Total spend by time period
- **Per-service breakdown**: Which services the agent is spending the most on
- **Per-skill breakdown**: Which skills are triggering the most spending
- **Running balance**: Current wallet balance and burn rate
- **Export**: CSV/JSON export for accounting, tax, expense reports

### No Controls in v0.1

No budget limits. No spend blocking. No enforcement hooks. Just visibility. This is intentional:

1. It works today with the existing `tool_result_persist` hook — no dependency on the `before_tool_call` hook that hasn't been wired up yet
2. Real usage data from v0.1 will inform how controls should actually work in v0.2
3. Tracking is valuable on its own — many users don't even know what their agent is spending

---

## Architecture

### Skill Component

```
~/.openclaw/skills/agent-budget/
├── SKILL.md              # Skill instructions for the agent
├── scripts/
│   ├── log-transaction.sh    # Appends a transaction record to the log
│   ├── query-log.sh          # Queries the transaction log (filters, rollups)
│   └── dashboard.sh          # Starts/stops the local dashboard server
├── server/
│   ├── server.js             # Express/Hono local server for dashboard
│   ├── index.html            # Dashboard UI (single-page app)
│   └── api.js                # REST endpoints for dashboard data
├── data/
│   └── transactions.jsonl    # Append-only transaction log (one JSON object per line)
└── references/
    └── payment-tools.md      # Known payment tool signatures for detection
```

### How It Hooks In

agent-budget uses the existing OpenClaw hook system. It does NOT require the `before_tool_call` hook that is currently unwired. It uses:

1. **`tool_result_persist` hook**: Fires after every tool call completes. agent-budget inspects each tool result to determine if a payment occurred. This is synchronous and available today.

2. **Pattern matching on tool calls**: The skill maintains a registry of known payment tool signatures — tool names, argument patterns, and response shapes that indicate a payment was made. Initial supported tools:
   - `agent-wallet-cli x402` — detects the x402 subcommand and extracts amount, recipient, tx hash from output
   - `v402` scripts — detects v402-pay.mjs calls and extracts payment details
   - ClawRouter — detects model routing calls and extracts per-request cost from response
   - Generic x402 — detects any tool response containing x402 payment confirmation headers
   - `payment-skill` (second-state) — detects pay script calls

3. **Fallback: wallet balance diffing**: For payment tools agent-budget doesn't recognize, it can periodically check wallet balance and flag unexplained decreases. This is a safety net, not the primary detection method.

### Data Model

Each transaction record is a JSON object appended to `transactions.jsonl`:

```json
{
  "id": "txn_1710523847_a3f2",
  "timestamp": "2026-03-15T14:30:47Z",
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
  "context": {
    "session_key": "agent:main:telegram:+1234567890",
    "skill": "stock-research",
    "user_request": "get me an AAPL stock report",
    "tool_name": "agent-wallet-cli",
    "tool_args_summary": "x402 POST https://alphaclaw.example.com/hunt --max-amount 0.10"
  },
  "status": "confirmed"
}
```

JSONL (one JSON object per line) is chosen because:
- Append-only writes are safe against corruption (no need to parse/rewrite a whole file)
- Easy to grep, tail, and stream
- Each line is independently parseable
- Natural fit for an audit log

### Dashboard Server

A lightweight local web server (Express or Hono) that reads `transactions.jsonl` and serves the dashboard UI. Runs on a configurable localhost port (default: 18920).

The dashboard is a single HTML file with embedded JS/CSS — no build step, no node_modules, no framework. It fetches data from the local API endpoints:

```
GET /api/transactions?from=2026-03-01&to=2026-03-15&service=alphaclaw
GET /api/summary/daily?from=2026-03-01
GET /api/summary/by-service?from=2026-03-01
GET /api/summary/by-skill?from=2026-03-01
GET /api/balance
GET /api/export?format=csv&from=2026-03-01
```

The server has no external network access. It reads local files and serves to localhost only.

---

## Security Considerations

### Sensitive Data

Transaction logs contain financial data and context about user requests. The skill must:

- Store `transactions.jsonl` with restrictive file permissions (owner read/write only)
- Never send transaction data to any external service
- Never include full wallet private keys or session tokens in logs
- Truncate or hash sensitive tool arguments (e.g., only store first/last 6 chars of wallet addresses)
- Dashboard server binds to 127.0.0.1 only, never 0.0.0.0

### Tamper Evidence

Even in v0.1 (no enforcement), the log should be tamper-evident so that if controls are added later, the historical record is trustworthy. Each transaction record includes a hash of the previous record:

```json
{
  "id": "txn_1710523847_a3f2",
  "prev_hash": "sha256:e3b0c44298fc1c149afbf4c8996fb924...",
  ...
}
```

This is a simple hash chain — not a blockchain, just each record linking to the previous one. If anyone modifies a historical record, the chain breaks. This is cheap to compute and lays groundwork for the VALET audit chain if it's ever needed.

### Skill Safety

The skill itself should be minimal in what it requests:
- No exec permissions needed (only reads tool results and writes to its own data directory)
- No network access needed (dashboard is localhost only)
- No access to wallet keys or credentials
- Read-only observation of tool call results

---

## Pluggable Payment Backend Detection

The skill needs to recognize when a payment happened across different wallet tools. This is done through a detector registry — a set of pattern matchers, one per known payment tool:

```
references/payment-tools.md defines:

## agent-wallet-cli
- Tool name pattern: "agent-wallet-cli" or "wallet"
- Payment indicator: args contain "x402" subcommand
- Amount extraction: parse "--max-amount" from args, actual amount from response
- Tx hash extraction: parse "tx_hash" or "transaction" from response JSON

## v402
- Tool name pattern: "v402-pay" or "v402-http"
- Payment indicator: script name contains "pay" or "http"
- Amount extraction: parse from response JSON "amount" field
- Tx hash extraction: parse from response JSON "signature" field

## ClawRouter
- Tool name pattern: response contains "cost" or "price" field
- Payment indicator: response includes x402 payment confirmation
- Amount extraction: "cost" field in response metadata
- Tx hash extraction: may not be available (aggregated billing)

## Generic x402
- Tool name pattern: any tool
- Payment indicator: response contains "X-PAYMENT-RESPONSE" header or "x402" in response body
- Amount/tx extraction: parse from x402 response payload
```

New payment tools can be supported by adding a new section to `payment-tools.md`. The detection logic reads this file and applies the matching rules. This means the community can contribute new detectors without modifying the core skill code.

---

## Installation

```bash
clawhub install agent-budget
```

Or manually:

```bash
git clone https://github.com/[your-org]/agent-budget.git ~/.openclaw/skills/agent-budget
```

### Configuration

In `~/.openclaw/openclaw.json`, enable the skill and optionally configure:

```json
{
  "skills": {
    "agent-budget": {
      "enabled": true,
      "config": {
        "dashboard_port": 18920,
        "log_path": "~/.openclaw/skills/agent-budget/data/transactions.jsonl",
        "auto_start_dashboard": false,
        "balance_check_interval": 300
      }
    }
  }
}
```

### Starting the Dashboard

```bash
# Via the agent (chat command)
"show me my agent's spending"

# Via CLI
~/.openclaw/skills/agent-budget/scripts/dashboard.sh start

# Then open
http://localhost:18920
```

---

## Future Expansion (Not in v0.1)

These are documented here for architectural awareness — they inform design decisions in v0.1 but are not built yet.

### v0.2: Budget Controls

- Per-service daily/weekly/monthly spending caps
- Per-transaction maximum amount
- Service allowlists and blocklists
- Alert thresholds (notify owner on Telegram/WhatsApp when spend exceeds limit)
- Global pause (freeze all agent payments with one tap)
- **Requires**: `before_tool_call` hook to be wired up in OpenClaw Gateway (Issue #5943). We would contribute that PR as part of v0.2 work.
- **UX**: Controls are set directly from the dashboard statement view. See a charge you don't like, tap it, set a limit. The budget rule is created from observed behavior, not upfront policy writing.

### v0.3: Multi-Agent Support

- Track spending across multiple agents on the same Gateway
- Per-agent budget allocation
- Consolidated statement across all agents

### v0.4: Receipt Sharing and Verification

- Agents can present receipts to each other as proof of payment
- Receipts are signed by the paying agent's key
- Service providers can verify receipts without seeing the full transaction log
- This is where the hash chain becomes an audit trail that other agents can reference

### v0.5: Integration with Identity Standards

- Link transaction records to ERC-8004 agent identities
- Reputation data: agents that stay within budget build trust scores
- Delegation records: formal authorization documents attached to budget rules
- This is where agent-budget connects to the broader VALET / SecureBraid vision

---

## Design Principles

1. **Observe first, control later.** Visibility is valuable on its own. Don't gate usefulness on enforcement capabilities.

2. **No new protocols.** agent-budget doesn't define a payment protocol. It watches existing ones (x402, L402, direct transfers) and reports what it sees.

3. **No external dependencies.** The dashboard is a single HTML file. The log is a JSONL file. The server is a minimal Node.js script. No databases, no cloud services, no build tools.

4. **Secure by default.** Localhost only. Restrictive file permissions. No network egress. Minimal skill permissions. Hash-chained log for tamper evidence.

5. **Pluggable detection.** New payment tools are supported by adding pattern definitions, not by modifying core code.

6. **Expansion-ready.** The data model, hook architecture, and dashboard layout all have room for budget controls, multi-agent support, and identity integration without rearchitecting.

---

## References

- OpenClaw hooks system: https://docs.openclaw.ai/tools/automation/hooks
- OpenClaw Gateway architecture: hub-and-spoke, single Node.js process, WebSocket on port 18789
- `tool_result_persist` hook: fires synchronously after tool completion, can transform results before session persistence
- `before_tool_call` hook: defined in plugins/hooks.js but not wired into tool execution pipeline (Issue #5943, #7597, #12311, #30504)
- Existing payment skills: agent-wallet-cli, v402, ClawRouter, Vault-0, payment-skill, Claw Cash, autonomous-agent
- Existing monitoring: ClawMetry, openclaw-dashboard (mudrii), openclaw-dashboard (tugcan), ezBookkeeping
- x402 protocol: https://x402.org — HTTP 402 Payment Required with stablecoin settlement
- ClawHub skill format: SKILL.md + optional scripts + metadata frontmatter