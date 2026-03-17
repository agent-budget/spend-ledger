---
name: agent-budget
description: Tracks every payment your agent makes and presents it as a reviewable transaction statement. Dashboard at http://127.0.0.1:18920 (run dashboard.sh start)
metadata:
  { "openclaw": { "emoji": "📦" } }
---

# agent-budget

You have access to an agent spending tracker. Payments made through wallet tools (agent-wallet-cli, v402, ClawRouter, payment-skill, or any x402-compatible tool) are detected and logged automatically. The log is tamper-evident (hash-chained) and deduplicated by transaction hash and idempotency key.

## Available Tools

### log-transaction.sh

Manually log a payment transaction. Use when automatic detection missed a payment or for recording manual expenditures. Duplicates (same tx_hash or idempotency_key) are rejected.

```bash
log-transaction.sh '{"service":{"url":"https://example.com/api","name":"Example Service"},"amount":{"value":"0.05","currency":"USDC","chain":"base"},"tx_hash":"0xabc...","idempotency_key":"req_123","context":{"skill":"research","user_request":"find AAPL data","input_hash":"a1b2c3"},"execution_time_ms":450,"status":"confirmed"}'
```

### query-log.sh

Query the transaction log. Use when the user asks about spending.

```bash
# All transactions
query-log.sh

# Filter by date range
query-log.sh --from 2026-03-01 --to 2026-03-15

# Filter by service name
query-log.sh --service alphaclaw

# Daily/weekly/monthly spending summary
query-log.sh --summary daily

# Breakdown by service or skill
query-log.sh --by-service
query-log.sh --by-skill

# Verify hash chain integrity
query-log.sh --verify
```

### dashboard.sh

Manage the local web dashboard for visual spending review.

```bash
dashboard.sh start    # Start dashboard (http://127.0.0.1:18920)
dashboard.sh stop     # Stop dashboard
dashboard.sh status   # Check if running
dashboard.sh url      # Print dashboard URL
```

## Automatic Detection

When a tool call completes, agent-budget inspects the tool name, arguments, and result to determine if a payment occurred. Detection covers:

- **Known wallet tools**: agent-wallet-cli (x402), v402, ClawRouter, payment-skill
- **Heuristic detection**: tools named `stripe_*`, `paypal_*`, `checkout`, `purchase`, `buy`, etc., plus argument patterns containing monetary amounts with currency/recipient signals
- **Generic x402**: any tool response containing `X-PAYMENT-RESPONSE` headers or x402 payment confirmations
- **User-tracked tools**: custom patterns added by the user via the dashboard; optionally submitted to maintainers for inclusion in the community list
- **Community patterns**: curated tool patterns fetched from `api.agent-budget.net/patterns.json` and refreshed hourly — expands detection coverage automatically as new payment tools are discovered by the community

For each detected payment, the log captures: service URL/name, amount/currency/chain, transaction hash, idempotency key, triggering skill, user request, input hash (for loop detection), execution time, failure type (pre_payment vs post_payment), and status.

## When to Use

- **User asks "what did you spend?"** → `query-log.sh` or `query-log.sh --summary daily`
- **User asks about a specific service** → `query-log.sh --by-service` or `query-log.sh --service <name>`
- **User wants the visual dashboard** → `dashboard.sh start` and share the URL
- **User wants to export for accounting** → Direct to dashboard export buttons, or `query-log.sh` with formatting
- **Detection missed a payment** → `log-transaction.sh` to record manually
- **User wants to verify log integrity** → `query-log.sh --verify`
