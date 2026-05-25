# arsenal

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Plugin version](https://img.shields.io/badge/websearch-1.0.0-blue.svg)](./plugins/websearch)

Mirko Bozzetto's curated skills for AI coding agents.

Skills are markdown-based, portable across providers (Claude Code, Codex, Cursor, etc.). Native plugin install is wired for Claude Code; other providers can load the skill files directly.

First plugin: `/websearch` - a power web search skill that wraps [Exa MCP](https://exa.ai) under a single command with 8 intent-routed modes. Replaces native `WebSearch` and `WebFetch` with semantic search, structured deep research, code lookup, docs crawling, and inline-cited reports.

---

## Quick install (Claude Code)

```bash
# 1. (Optional) Sign up at https://dashboard.exa.ai and copy an API key
#    Free anonymous tier works without a key, rate-limited.
#    Add a key only when you hit rate limits.

# 2. Add the Exa MCP server (hosted, recommended)
claude mcp add --transport http exa https://mcp.exa.ai/mcp

# 3. Verify connection
claude mcp list

# 4. Inside Claude Code:
/plugin marketplace add mirkobozzetto/arsenal
/plugin install websearch@arsenal

# 5. Try it
/websearch what is gRPC
```

Full setup walkthrough, mode reference, troubleshooting → [plugins/websearch/README.md](./plugins/websearch/README.md).

---

## Install on other agents (Codex, Cursor, etc.)

Skills here are plain markdown. To use them outside Claude Code:

1. Clone the repo or copy `plugins/websearch/skills/websearch/` into your agent's skill/prompt directory.
2. Wire the agent's MCP config to Exa (hosted or local). Most modern coding agents support MCP.
3. Invoke per your agent's slash/skill mechanism.

The skill body assumes the 4 Exa MCP tools (`web_search_exa`, `get_code_context_exa`, `crawling_exa`, `web_search_advanced_exa`). Any agent that can call those will work.

---

## What `/websearch` does

One slash command, 8 modes selected by flag:

| Mode | Flag | Use case |
|------|------|----------|
| Quick | *(none)* | Factual lookup, one-shot |
| Deep research | `--deep` | Multi-pass with gap analysis |
| Code | `--code` | API snippets, library usage |
| Docs | `--docs <lib>` | Crawl official docs |
| Debug | `--debug <err>` | Paste error, find fix |
| News | `--news` | Recent news, category filtered |
| Compare | `--compare A vs B` | Side-by-side tech choice |
| Research | `--research` | Academic papers |
| Similar | `--similar <url>` | Find similar pages |

Filters (combinable): `--after`, `--before`, `--domain`, `--exclude`, `--fresh`, `--locale`, `-n`.
Output: `--save <file>`, `--json`, `--full`.
Help: `--info` (full tutorial inside the skill).

---

## Plugins in this marketplace

| Plugin | Version | Description |
|--------|---------|-------------|
| [websearch](./plugins/websearch) | 1.0.0 | Intent-routed web search via Exa MCP (8 modes) |

More plugins may land here over time. Marketplace name stays `arsenal`.

---

## Prerequisites

- Claude Code v2.1+ for the native plugin install (slash plugin commands).
- Exa MCP server connected (`mcp__exa__*` tools available).
- Exa API key **optional** - hosted endpoint `https://mcp.exa.ai/mcp` works anonymously with a free, rate-limited tier. Add a key via header `Authorization: Bearer KEY` or `x-api-key: KEY` to bypass rate limits.

Get a free key at [dashboard.exa.ai/api-keys](https://dashboard.exa.ai/api-keys). See [docs.exa.ai/reference/exa-mcp](https://docs.exa.ai/reference/exa-mcp) for Exa MCP details.

---

## Why a marketplace, not a bare skill

- **Versioned** - semver tags, no "did the file change" guesswork.
- **`/plugin update`** works out of the box.
- **Same install pattern** as official Anthropic plugins.
- **Extensible** - more plugins can ship under this marketplace later without breaking installs.

---

## Documentation map

- [Full plugin README](./plugins/websearch/README.md) - setup, modes, filters, troubleshooting, architecture.
- [SKILL.md](./plugins/websearch/skills/websearch/SKILL.md) - skill entry point.
- [steps/](./plugins/websearch/skills/websearch/steps) - pipeline (triage → search → deep → report).
- [references/](./plugins/websearch/skills/websearch/references) - domain presets, query patterns, full tutorial.

---

## Contributing

Open an issue or PR at [github.com/mirkobozzetto/arsenal](https://github.com/mirkobozzetto/arsenal). One concern per PR. Bump semver on any change shipping to users.

---

## License

MIT - see [LICENSE](./LICENSE).
