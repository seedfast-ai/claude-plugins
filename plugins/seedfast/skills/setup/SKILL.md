---
name: setup
description: Configure and troubleshoot the Seedfast plugin — API key setup, verifying the MCP server is connected, and diagnosing seedfast_doctor failures. Use when Seedfast tools are missing or erroring, when the API key is rejected or absent, or when the user is installing the plugin for the first time.
---

# Seedfast setup and troubleshooting

## Confirm what is actually broken

Run `seedfast_doctor` first. It returns CLI status and version, the binary path, whether `SEEDFAST_API_KEY` is configured, platform, Go runtime version, and MCP server version. Almost every setup question is answered by its output, and guessing before reading it wastes a round trip.

If the `seedfast_*` tools are not available at all, the MCP server is not connected — that is a different problem, handled below.

## The API key

Created at [seedfa.st/dashboard](https://seedfa.st/dashboard). Keys start with `sfk_`.

The plugin asks for it at install time and Claude Code stores it in the plugin's user config, so there is no file for the user to edit. To change it later, run `/plugin configure` and pick Seedfast.

For a scripted or CI setup, it can be supplied at install time instead:

```bash
claude plugin install seedfast@seedfast --config api_key=sfk_live_...
```

Only `seedfast_plan` and `seedfast_run` need the key. If `seedfast_doctor` reports it missing, schema introspection and connection tests still work — say so instead of treating the session as blocked.

**Never ask the user to paste the key into the chat.** Point them at `/plugin configure`. If they paste it anyway, do not repeat it back and do not write it into any file in the repo.

## The MCP server will not start

The server ships inside the Seedfast CLI and runs via `npx -y seedfast@latest mcp`. That means the first start downloads the package, which can take a minute on a cold cache.

Check in order:

1. **`/mcp`** — shows whether `seedfast` is connected, failed, or absent. If absent, the plugin is not enabled; `/plugin` to check.
2. **Node.js 18 or newer** — `node --version`. The CLI requires it.
3. **Network access to the npm registry** — behind a proxy or an offline machine, `npx` cannot fetch the package.

For a machine where `npx` is not viable, install the CLI directly and it will be used instead:

```bash
brew install seedfast-ai/tap/seedfast   # macOS, Linux
npm install -g seedfast                 # any platform with Node.js
```

## Connection failures

`seedfast_connections_test` masks credentials in its output, so its error text is safe to show the user verbatim. Read what it actually says before theorising:

| Symptom | Usual cause |
|---------|-------------|
| Timeout after 10s | Host unreachable — firewall, VPN, or a container that is not running |
| Authentication failed | Wrong user or password in the DSN |
| SSL required | Append `?sslmode=require` to the DSN |
| Database does not exist | Typo in the database name, or pointing at the server rather than a database |

DSN format:

```
postgres://user:password@host:5432/dbname
```

For a database in Docker Compose, the host is `localhost` from the developer's machine and the service name from inside another container. Getting this backwards is the most common local failure.

## Run history that survives a restart

By default plans and run history live in memory and vanish when the MCP server exits. To keep them, run `/plugin configure` and set the optional run-history file to an absolute path to a SQLite file.

This matters for anyone who builds a plan today and wants to execute it tomorrow.

## Verifying end to end

A clean install answers all four:

1. `/mcp` lists `seedfast` as connected.
2. `seedfast_doctor` reports CLI OK and the API key configured.
3. `seedfast_connections_test` succeeds against a throwaway database.
4. `seedfast_schema_info` returns that database's tables.

Only then is a `seedfast_plan` worth attempting.

## Support

Bugs and questions: support@seedfa.st. Documentation: [seedfa.st/docs](https://seedfa.st/docs).
