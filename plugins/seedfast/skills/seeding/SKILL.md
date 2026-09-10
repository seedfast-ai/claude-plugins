---
name: seeding
description: Fill a PostgreSQL database with realistic, relationally valid test data using Seedfast. Use when the user wants to seed, populate, or fill a database, needs test/demo/staging data, has empty tables to work against, wants a dev database that behaves like production, or says "seed the database", "seed my database", "populate my postgres with test data", "fill the staging database", "generate test data for these tables", "my dev database is empty", "I need demo data", or mentions Seedfast, fixtures, or synthetic data.
---

# Seeding a database with Seedfast

Seedfast reads a live PostgreSQL schema and generates data that satisfies it — foreign keys resolve, constraints hold, and values look like the domain rather than `test_user_1`. The work happens on Seedfast's backend; the MCP tools here drive it.

## The loop

```
doctor  ->  connections_test  ->  schema_info  ->  plan  ->  [user approves]  ->  run  ->  run_status (poll)
```

Skipping straight to `seedfast_run` is allowed, but only when the user has clearly said "just seed it" and the target is obviously disposable.

### 1. Check the environment

Call `seedfast_doctor` first, once per session. It reports CLI status and version, whether the API key is configured, and the platform. If it reports a missing API key, stop and route the user to the `setup` skill instead of retrying.

### 2. Verify the connection before anything else

`seedfast_connections_test` with the DSN. It opens a pool and pings with a 10-second timeout, and masks credentials in its output. This catches a wrong password or a closed firewall port in one second instead of surfacing it as a confusing planner failure a minute later.

DSN format:

```
postgres://user:password@host:5432/dbname
postgres://user:password@host:5432/dbname?sslmode=require
```

Never print a DSN back to the user with the password intact. When you need to refer to a database, name it (`the staging DB`), don't echo the string.

### 3. Read the schema before writing a scope

`seedfast_schema_info` returns tables, columns, primary keys, foreign keys, and approximate row counts. Use it to ground the scope in tables that actually exist.

Row counts come from `pg_class.reltuples`. They are approximate, can be stale between `ANALYZE` runs, and are `-1` on never-analyzed tables. Treat them as a size hint, never as a fact to report.

The `dsn` argument is optional here — with it omitted, the server falls back to `SEEDFAST_DSN` or `DATABASE_URL` in its own environment.

### 4. Plan, then let the user look

`seedfast_plan` generates a plan without writing a single row, and stores it in the session. It returns a plan ID, the scope echoed back, and the table list.

Show the user the table list and the scope before running. This is the whole point of the plan step — it is the last cheap moment to catch "that scope also touches `billing_invoices`".

`seedfast_plan` requires an API key.

### 5. Run it

`seedfast_run` returns immediately with a `runId`; the seeding proceeds in the background.

- Pass `planId` to execute an approved plan. The scope is derived from the plan's tables and the `scope` argument is ignored.
- Without `planId`, `scope` is required.
- Always pass an `idempotencyKey`. A retry carrying the same key returns the existing run instead of starting a new one, which is the difference between a dropped connection costing nothing and costing the user a doubled `orders` table.

### 6. Poll to completion

`seedfast_run_status` is safe to call repeatedly. It reports state (`pending`, `running`, `completed`, `failed`, `cancelled`), progress as completed vs total tables, row totals, the table currently being seeded, and a summary once finished.

Poll at a human pace — a few seconds between calls, not a tight loop. Report progress to the user as it moves rather than going silent for two minutes.

### 7. Answer questions the run raises

A run can move to `awaiting_input` when the backend needs a decision about scope or a replan. `seedfast_run_status` surfaces the `questionId`; the full text is at `seedfast://runs/{runId}/pending_question`.

Reply with `seedfast_run_answer`:

- `answer.human_answer = true` approves the current plan or scope as-is.
- `answer.human_answer = false` plus `answer.raw` with a textual refinement (`"seed only the org schema"`) adjusts it.

**Bring the question to the user.** Do not auto-approve on their behalf — the backend asks precisely when the right call is not obvious. The CLI blocks for up to 5 minutes waiting for the reply, so answer promptly once the user decides.

## Writing scopes

Scopes are plain English, interpreted by the backend. Do not pre-parse them into a DSL.

```
seed the users and posts tables with 1000 rows each
HR schema only
everything except the audit and billing tables
enough orders across 50 customers to exercise the reporting dashboard
```

Two things make a scope good: naming real tables from `seedfast_schema_info`, and saying how much. "Some test data" produces a plan nobody can review.

For more patterns, request the server's `scope-examples` prompt (`general`, `ci`, or `exploration`). For a production-shaped target, request the `seed-production-db` prompt before starting.

## Safety

**Seedfast writes rows to a real database.** Before the first `seedfast_run` of a session, confirm the target is a development, staging, or test database. If the DSN host looks production-shaped — `prod`, `live`, a customer domain, an RDS writer endpoint — stop and ask outright.

**Cancellation does not roll back.** `seedfast_run_cancel` stops the run, but rows already inserted stay inserted. A cancelled run leaves a partially seeded database that someone has to clean up. Say so when you cancel.

**`seedfast_plan_delete` is irreversible** and does not touch runs started from that plan.

## Guardrails

**Never invent a table name.** Every table a scope names comes from `seedfast_schema_info`. A guessed name produces nothing and reports nothing about having produced nothing.

**Never read success out of the `seedfast_run` response.** That call returns before the first insert, carrying a run ID and `pending`. Only `seedfast_run_status` reporting `completed` describes a result.

**Never start a second run to fix a slow one.** Poll it, or cancel it and then start a single run with a corrected scope. Two runs against the same tables leave a mess that has to be cleaned up by hand.

**Counted rows are the only real evidence.** Once a run reports `completed`, run `SELECT count(*)` against the tables the scope named and put those numbers beside the ones the scope asked for. Status text describes what the backend thinks it did; the counts are what you report to the user.

## Plans

Plans live in the MCP session, in memory, unless the user configured a run-history file. They do not survive an MCP server restart. Do not promise a user that a plan will be there tomorrow.

| Tool | Use |
|------|-----|
| `seedfast_plans_list` | Find plan IDs (accepts `limit`) |
| `seedfast_plan_get` | Full table list and preview for one plan |
| `seedfast_plan_create` | Store a hand-built plan, skipping the planner round-trip. Needs `scope` and at least one entry in `tables` |
| `seedfast_plan_update` | Change `tables`, `scope`, or `preview`. Only non-empty fields overwrite — omitted fields are preserved, and there is no way to clear a field back to empty |
| `seedfast_plan_delete` | Discard a plan. Irreversible |

`seedfast_plan_create` is the fast path when the tables are already known from a previous run, since it needs no API key and no backend call.

## Resources

Read these directly when the status text is not enough:

| URI | Contents |
|-----|----------|
| `seedfast://runs/{runId}/summary` | Run summary as JSON |
| `seedfast://runs/{runId}/log` | Event log, NDJSON — the place to look when a run fails |
| `seedfast://runs/{runId}/pending_question` | The question an `awaiting_input` run is blocked on |
| `seedfast://plans/{planId}` | Full plan as JSON |

## When a run fails

1. `seedfast_run_status` for the error message and the failed-tables map.
2. `seedfast://runs/{runId}/log` for the events leading up to it.
3. Read the failure before re-running. A constraint the generator could not satisfy will fail again identically; the scope or the schema is what needs to change.

## Which tools need the API key

Only `seedfast_plan` and `seedfast_run` reach the Seedfast backend. Everything else — `doctor`, `connections_test`, `schema_info`, all plan management, run status, cancel, and answer — works without a key. A missing key is not a reason to abandon schema exploration.

## Scale

PostgreSQL only today. MySQL, Oracle, and SQLite are in development. If the user points this at a MySQL database, say that plainly rather than trying the DSN.
