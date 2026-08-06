# Seedfast Claude Plugins — Development Guide

## Overview

This is the **public** Claude Code plugin marketplace for Seedfast. The repo root is the marketplace; plugins live in `plugins/<name>/`.

Not to be confused with `seedfast-ai/internal-claude-plugins`, which is private and holds team-internal tooling. Anything here is world-readable and is part of the product surface — write it for a user who has never seen the codebase.

## Directory Structure

```
.claude-plugin/marketplace.json     Marketplace catalog
plugins/
└── seedfast/                       MCP server + seeding workflow
    ├── .claude-plugin/plugin.json   Manifest, including userConfig
    ├── .mcp.json                    Seedfast MCP server (npx stdio)
    ├── README.md                    Single source of truth for this plugin's docs
    └── skills/
        ├── seeding/SKILL.md         Plan-before-run workflow, scope grammar, safety
        └── setup/SKILL.md           API key, MCP connection, DSN troubleshooting
```

## Git

Single branch: `master`. There is no `develop` here.

- Conventional commit prefixes: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`
- One concern per commit. No "and" in commit titles
- No AI attribution in commit messages
- Every commit leaves the repo valid — `claude plugin validate . --strict` must pass

## Adding a New Plugin

1. Create `plugins/<name>/` with `.claude-plugin/plugin.json`
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`
3. Add a `README.md` inside `plugins/<name>/` — the single source of truth for that plugin's docs
4. Add a row to the root `README.md` plugins table
5. Validate: `claude plugin validate . --strict`

No marketplace version bump needed — Claude Code detects changes via git.

## Documentation Structure

- **Root `README.md`** is an index — installation plus a table linking to each plugin. No plugin-specific detail.
- **`plugins/<name>/README.md`** is the single source of truth for that plugin (how it works, configuration, troubleshooting).
- **`plugins/<name>/skills/*/SKILL.md`** is Claude-facing guidance, loaded into agent context.
- Never duplicate plugin docs between root README and plugin README. Root links, plugin README explains.

## User Configuration

Secrets go in `userConfig` in `plugin.json` with `"sensitive": true`, and are referenced from `.mcp.json` as `${user_config.<key>}`. Claude Code prompts at install and stores the value itself.

Never require the user to export an environment variable or hand-edit a config file. Removing that step is the reason this marketplace exists.

## Keeping Content Truthful

The `seedfast` plugin's skills describe real MCP tool behavior — argument names, which tools need an API key, that cancellation does not roll back. These came from the live server, not from guesswork.

When the MCP server changes, re-derive rather than assume. `seedfast-ai/seedfast-mcp` has a smoke test that completes a handshake and lists tools without an API key:

```bash
node scripts/smoke-test.mjs
```

Documenting a tool argument or a CLI command that does not exist is worse than omitting it — the user hits an error and loses trust in the rest of the page.

## Testing Changes Locally

```bash
claude --plugin-dir ./plugins/seedfast
```

Loads the plugin for a single session without installing it.

```bash
claude plugin validate . --strict
```

Validates the marketplace and every plugin manifest. Warnings are errors under `--strict`.

## Releasing

1. Bump `version` in the plugin's `.claude-plugin/plugin.json`
2. Commit and push to `master`
3. Users with auto-update get the new version; others run `/plugin marketplace update seedfast`
