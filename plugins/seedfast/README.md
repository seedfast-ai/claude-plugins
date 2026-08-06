# Seedfast

Fills a PostgreSQL database with realistic, relationally valid test data generated from its live schema — foreign keys resolve, constraints hold, and values look like the domain rather than `test_user_1`.

The plugin bundles the [Seedfast MCP server](https://github.com/seedfast-ai/seedfast-mcp) and the workflow discipline around it, so installing it is the only setup step. No `claude_desktop_config.json` edit, no environment variables to export.

## Installation

```
/plugin marketplace add seedfast-ai/claude-plugins
/plugin install seedfast@seedfast
```

Claude Code prompts for a Seedfast API key during install. Create one at [seedfa.st/dashboard](https://seedfa.st/dashboard) — it starts with `sfk_`.

Non-interactively, for CI or a scripted team setup:

```bash
claude plugin marketplace add seedfast-ai/claude-plugins
claude plugin install seedfast@seedfast --config api_key=sfk_live_...
```

Then just say what you want:

> seed my local postgres with a few hundred orders across 50 customers

## How it works

```
doctor            is the CLI healthy, is the API key configured
connections_test  can we reach the database at all (10s, credentials masked)
schema_info       tables, columns, PKs, FKs, approximate row counts
plan              what would be seeded — no rows written
  -> you approve
run               async, returns a runId
run_status        poll until completed
```

The plan step is the point of the whole thing. It is the last cheap moment to notice that a scope reading "seed the customer tables" also reaches `billing_invoices`. The `seeding` skill enforces the review rather than letting the model run straight through.

Runs are asynchronous, and the backend can pause one to ask a question — a scope that is ambiguous, a replan it wants signed off. The skill routes that question to you instead of answering it on your behalf.

## What ships in the plugin

| Component | Purpose |
|-----------|---------|
| `.mcp.json` | The Seedfast MCP server: 13 tools, 4 resources, 2 prompts, run via `npx` |
| `skills/seeding` | The plan-before-run workflow, scope grammar, failure triage, safety rules |
| `skills/setup` | API key setup, MCP connection diagnosis, DSN troubleshooting |

Both skills load themselves when they are relevant. `/seedfast:seeding` and `/seedfast:setup` invoke them directly.

## Configuration

| Setting | Required | What it does |
|---------|----------|--------------|
| Seedfast API key | yes | Authenticates planning and seeding runs |
| Run history file | no | Absolute path to a SQLite file. Without it, plans and run history live in memory and disappear when the MCP server exits |

Change either later with `/plugin configure` inside Claude Code.

Only `seedfast_plan` and `seedfast_run` reach the Seedfast backend. Schema introspection, connection tests, plan management, and run polling all work without a key.

## Requirements

- Node.js 18 or newer — the MCP server runs from `npx`
- A reachable PostgreSQL database

Prefer a local binary over `npx`? Install the CLI and it gets used instead:

```bash
brew install seedfast-ai/tap/seedfast   # macOS, Linux
npm install -g seedfast                 # any platform with Node.js
```

## Two things to know before the first run

**Seeding writes to a real database.** The skill confirms the target is not production before the first run of a session, but the DSN is yours to get right.

**Cancelling does not roll back.** `seedfast_run_cancel` stops the run; rows already inserted stay inserted.

## Databases

PostgreSQL today. Going PostgreSQL-first was deliberate — its constraint and relationship system is the richest one out there, which is exactly where generating believable data gets hard. MySQL, Oracle, and SQLite are in development.

## Links

- [Documentation](https://seedfa.st/docs)
- [MCP setup guide](https://seedfa.st/docs/mcp-setup-guide) — for clients other than Claude Code
- [MCP server repository](https://github.com/seedfast-ai/seedfast-mcp) — tool reference and smoke test
- Questions and bug reports: support@seedfa.st
