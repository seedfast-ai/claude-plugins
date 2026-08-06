# Seedfast Claude Plugins

Official Claude Code plugins for [Seedfast](https://seedfa.st) — realistic, relationally valid PostgreSQL test data generated from a live schema.

The Seedfast engine is closed source and runs as a hosted service. This marketplace is the open part: the plugin that wires the MCP server into Claude Code, plus the workflow knowledge that makes it useful. Installing a plugin is the entire setup — no config file to hand-edit, no environment variables to export.

## Installation

```
/plugin marketplace add seedfast-ai/claude-plugins
/plugin install seedfast@seedfast
```

Claude Code prompts for anything the plugin needs during install. Then say what you want:

> seed my local postgres with a few hundred orders across 50 customers

Non-interactively, for CI or a scripted team setup:

```bash
claude plugin marketplace add seedfast-ai/claude-plugins
claude plugin install seedfast@seedfast --config api_key=sfk_live_...
```

### Team-wide setup (optional)

Add to a project's `.claude/settings.json` so teammates are prompted automatically:

```json
{
  "extraKnownMarketplaces": {
    "seedfast": {
      "source": {
        "source": "github",
        "repo": "seedfast-ai/claude-plugins"
      }
    }
  }
}
```

## Plugins

| Plugin | Description |
|--------|-------------|
| [seedfast](plugins/seedfast/) | Seeds a PostgreSQL database from a plain-English scope. Bundles the Seedfast MCP server and the plan-before-run workflow that keeps seeding safe |

## Requirements

Node.js 18 or newer. The MCP server ships inside the Seedfast CLI and runs from `npx`, so there is nothing to install separately.

## Using Seedfast outside Claude Code

The MCP server works with any MCP client. Client-by-client configuration, the full tool reference, and a no-API-key smoke test live in [seedfast-ai/seedfast-mcp](https://github.com/seedfast-ai/seedfast-mcp).

## Links

- [Documentation](https://seedfa.st/docs)
- [Dashboard](https://seedfa.st/dashboard) — create an API key
- Questions and bug reports: support@seedfa.st

## License

MIT. See [LICENSE](LICENSE).
