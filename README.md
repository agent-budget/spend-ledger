# agent-budget

The credit card statement for your OpenClaw agent.

agent-budget tracks every payment your agent makes — what service, how much, when, and why — and presents it as a clear, reviewable transaction statement. Think of it as an expense report for autonomous AI agents.

## The Problem

OpenClaw agents can hold wallets and spend real money via x402 micropayments. Multiple skills enable this (agent-wallet-cli, v402, ClawRouter, payment-skill, and others). But nobody answers the basic question every agent owner has:

**"What did my agent spend my money on?"**

Existing dashboards track LLM inference costs (tokens consumed, API call prices). Wallet tools show raw blockchain data (addresses and amounts). Neither tells you "your agent paid $0.05 to AlphaClaw for a research report while working on your morning task."

## What It Does

**Observe and report.** No enforcement, no blocking — just visibility.

- **Automatic detection**: Watches tool calls for payment patterns across known wallet tools, heuristic matching (Stripe, PayPal, etc.), and generic x402 signals
- **Structured logging**: Every payment is recorded as a JSON line with service, amount, chain, tx hash, triggering skill, user request, execution time, and status
- **Tamper-evident**: Each record is hash-chained to the previous one. If anyone modifies history, the chain breaks
- **Deduplicated**: Same transaction hash or idempotency key won't be logged twice
- **Dashboard**: Local web app with transaction list, daily/weekly/monthly rollups, per-service and per-skill breakdowns, and CSV/JSON export
- **Manual tracking**: Log payments the auto-detection missed, and add custom tool patterns the detector should watch for
- **Community patterns**: Running skills fetch a curated pattern list from `api.agent-budget.net/patterns.json` hourly — new community-contributed patterns are picked up automatically, no skill update required. When adding a tracked tool, optionally submit it to maintainers; patterns that reach 3 unique submitters are published

## Quick Start

### Install

```bash
# Copy into OpenClaw managed skills directory
cp -r agent-budget ~/.openclaw/skills/agent-budget

# Restart the gateway
systemctl --user restart openclaw-gateway

# Verify
openclaw skills list | grep budget
```

### Use

Talk to your agent:
- "What have you spent today?"
- "Show me spending by service"
- "Start the budget dashboard"

Or use the CLI directly:

```bash
# Query spending
~/.openclaw/skills/agent-budget/scripts/query-log.sh --summary daily

# Start dashboard
~/.openclaw/skills/agent-budget/scripts/dashboard.sh start
# Opens at http://127.0.0.1:18920

# Log a transaction manually
~/.openclaw/skills/agent-budget/scripts/log-transaction.sh '{"service":{"name":"Example"},"amount":{"value":"0.05","currency":"USDC"}}'

# Verify log integrity
~/.openclaw/skills/agent-budget/scripts/query-log.sh --verify
```

## Requirements

- Node.js 18+
- OpenClaw 2026.3.x+
- No external dependencies (zero npm packages)

## Project Structure

```
agent-budget/
├── SKILL.md              # Agent-facing instructions (what the LLM reads)
├── README.md             # This file (human-facing overview)
├── TECHNICAL.md          # Architecture, security model, data model
├── PROPOSAL.md           # Original design proposal
├── package.json
├── scripts/
│   ├── log-transaction.sh    # CLI: append a transaction
│   ├── query-log.sh          # CLI: query/filter/summarize the log
│   └── dashboard.sh          # CLI: start/stop the dashboard server
├── server/
│   ├── transactions.js       # Data layer: append, query, summarize, hash chain, export
│   ├── detectors.js          # Payment detection: registry of pattern matchers
│   ├── patterns-sync.js      # Community pattern sync: fetch/cache api.agent-budget.net/patterns.json
│   ├── server.js             # HTTP server: API endpoints + dashboard
│   └── index.html            # Dashboard UI (single HTML file, no build step)
├── references/
│   └── payment-tools.md      # Detector signatures for each known payment tool
├── data/
│   ├── transactions.jsonl      # Append-only transaction log (created at runtime)
│   ├── tracked-tools.json      # User-defined tool patterns (created at runtime)
│   ├── community-patterns.json # Cached community patterns from api.agent-budget.net (created at runtime)
│   ├── install-id.json         # Stable per-install UUID for anonymous submissions (created at runtime)
│   └── submissions.jsonl       # Local log of patterns sent to maintainers (created at runtime)
└── test/
    └── test.js               # 32 tests (transactions, detectors, server API)
```

## Running Tests

```bash
cd agent-budget
node --test test/test.js
```

## Roadmap

- **v0.1** (current): Observe and report. Transaction logging, dashboard, manual tracking.
- **v0.2**: Budget controls. Per-service caps, blocklists, alerts. Requires `before_tool_call` hook in OpenClaw Gateway.
- **v0.3**: Multi-agent support. Per-agent budgets and consolidated statements.
- **v0.4**: Receipt sharing. Agents present signed receipts to each other as proof of payment.
- **v0.5**: Identity integration. Link to ERC-8004 agent identities and VALET delegations.

## License

MIT
