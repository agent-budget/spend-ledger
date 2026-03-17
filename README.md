# agent-budget

Payment tracker for OpenClaw agents. Detects payments made through wallet tools, logs them to a tamper-evident local ledger, and presents them in a dashboard.

## Security

All transaction data stays on your machine:

- Log is stored at `data/transactions.jsonl`, mode `0600`
- Dashboard binds to `127.0.0.1:18920` — not reachable from the network
- No payment data, transaction records, or personal information is sent anywhere
- No wallet keys or credentials are accessed — the skill observes tool call results only
- Outbound network: fetches a community-curated tool pattern list from `api.agent-budget.net/patterns.json` hourly; no payment data is included

See [TECHNICAL.md](TECHNICAL.md) for the full security model, data model, and architecture.

## Install

```bash
cp -r agent-budget ~/.openclaw/skills/agent-budget
systemctl --user restart openclaw-gateway
```

## Use

Ask your agent:
- "What have you spent today?"
- "Show me spending by service this week"
- "Start the budget dashboard"

Or use the CLI directly:

```bash
# Spending summary
~/.openclaw/skills/agent-budget/scripts/query-log.sh --summary daily

# Visual dashboard at http://127.0.0.1:18920
~/.openclaw/skills/agent-budget/scripts/dashboard.sh start

# Verify log integrity
~/.openclaw/skills/agent-budget/scripts/query-log.sh --verify
```

## Requirements

- Node.js 18+
- OpenClaw 2026.3.x+
- No npm dependencies

## Project Structure

```
agent-budget/
├── SKILL.md              # Agent-facing instructions (what the LLM reads)
├── TECHNICAL.md          # Architecture, security model, data model
├── server/
│   ├── transactions.js   # Append, query, summarize, hash chain, export
│   ├── detectors.js      # Payment detection registry
│   ├── patterns-sync.js  # Community pattern sync from api.agent-budget.net
│   ├── server.js         # HTTP server and API endpoints
│   └── index.html        # Dashboard UI (no build step, no CDN)
├── scripts/
│   ├── log-transaction.sh
│   ├── query-log.sh
│   └── dashboard.sh
└── data/                 # Created at runtime, all mode 0600
    ├── transactions.jsonl
    ├── tracked-tools.json
    ├── community-patterns.json
    └── install-id.json
```

## Tests

```bash
node --test test/test.js
```

## License

MIT
