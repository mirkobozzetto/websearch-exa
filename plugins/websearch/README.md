# websearch

A single `/websearch` slash command that routes your query to the right [Exa](https://exa.ai) tool based on intent. Eight specialized modes, smart filters, and structured reports with inline citations.

Replaces native `WebSearch` and `WebFetch` tools entirely.

**Compatibility**: Native plugin install is wired for Claude Code v2.1+. The skill itself is plain markdown - any AI coding agent that can read markdown skills and call MCP tools (Codex, Cursor, etc.) can use it by loading the files directly from `plugins/websearch/skills/websearch/`.

---

## Table of contents

- [Why this skill exists](#why-this-skill-exists)
- [Setup (full walkthrough)](#setup-full-walkthrough)
  - [1. Get an Exa API key](#1-get-an-exa-api-key)
  - [2. Register the Exa MCP server](#2-register-the-exa-mcp-server)
  - [3. Verify Exa MCP is connected](#3-verify-exa-mcp-is-connected)
  - [4. Install the websearch plugin](#4-install-the-websearch-plugin)
  - [5. First run](#5-first-run)
- [How the skill works](#how-the-skill-works)
- [Modes (full reference)](#modes-full-reference)
- [Filters](#filters)
- [Output options](#output-options)
- [Examples by use case](#examples-by-use-case)
- [Token cost](#token-cost)
- [Limitations & known issues](#limitations--known-issues)
- [Troubleshooting](#troubleshooting)
- [Architecture](#architecture)
- [Contributing](#contributing)
- [License](#license)

---

## Why this skill exists

Claude Code ships with `WebSearch` and `WebFetch`. They work, but:

- Quality varies wildly across queries.
- No semantic search, only keyword matching.
- No category filters (no "news only", no "research papers only").
- No structured deep-research loop.
- No citation discipline (Claude paraphrases without always linking).

Exa is a search engine built for LLMs (semantic + neural ranking). It exposes 4 tools via MCP. `websearch` wraps them under a single command with intent-based routing so you do not have to remember which Exa tool to call.

---

## Setup (full walkthrough)

This is a one-time setup. Takes about 3 minutes.

### 1. (Optional) Get an Exa API key

The hosted endpoint `https://mcp.exa.ai/mcp` works **anonymously** with a free, rate-limited tier. You only need a key if:

- You hit rate limits on the hosted endpoint, or
- You run the local npm server (key is mandatory there).

To get a key:

- Sign up at [dashboard.exa.ai](https://dashboard.exa.ai).
- Free plan available, ~1,000 searches/month at the time of writing.
- Create a key at [dashboard.exa.ai/api-keys](https://dashboard.exa.ai/api-keys).

### 2. Register the Exa MCP server

Two options.

**Option A - hosted HTTP transport (recommended)**

No local server, no env var. Works anonymously, free tier rate-limited.

```bash
claude mcp add --transport http exa https://mcp.exa.ai/mcp
```

To bypass rate limits, pass the key. Three accepted methods (priority order):

1. `Authorization: Bearer YOUR_KEY` header (recommended).
2. `x-api-key: YOUR_KEY` header.
3. `?exaApiKey=YOUR_KEY` URL query param (legacy compat).

Example with header:

```bash
claude mcp add --transport http exa https://mcp.exa.ai/mcp \
  --header "Authorization: Bearer YOUR_KEY"
```

**Option B - local npm server (needs API key)**

API key is **required** here. No anonymous tier. Hits your account directly.

```bash
claude mcp add exa -e EXA_API_KEY=YOUR_KEY -- npx -y exa-mcp-server
```

Both commands write to `~/.claude.json` (user-level MCP config on macOS/Linux). Manual JSON equivalent:

```json
{
  "mcpServers": {
    "exa": {
      "type": "http",
      "url": "https://mcp.exa.ai/mcp"
    }
  }
}
```

### 3. Verify Exa MCP is connected

```bash
claude mcp list
```

You should see `exa` listed as connected. Start a Claude Code session and the `mcp__exa__*` tools should be available:

- `mcp__exa__web_search_exa` - general search.
- `mcp__exa__web_search_advanced_exa` - filtered search (categories, dates, domains).
- `mcp__exa__get_code_context_exa` - code-focused search (legacy alias kept for compatibility).
- `mcp__exa__crawling_exa` - read full URL content (legacy alias kept for compatibility).

Tool names may evolve. The skill tries the canonical name first and falls back to legacy aliases when needed.

### 4. Install the websearch plugin

```
/plugin marketplace add mirkobozzetto/arsenal
/plugin install websearch@arsenal
```

This downloads the plugin into `~/.claude/plugins/cache/arsenal/` and registers the `/websearch` slash command.

### 5. First run

```
/websearch what is gRPC
```

Expected output: 3-5 bullet points with `[source](url)` citations, total ~2-5k tokens. If you get an error about `mcp__exa__*` not found, go back to step 3.

---

## How the skill works

A skill in Claude Code is a markdown file (`SKILL.md`) plus optional sub-files. When you type `/websearch <query>`, Claude reads `SKILL.md`, which directs it to a 4-step pipeline:

```
SKILL.md                              entry point, defines flags + state
  └─> steps/step-00-triage.md         parse flags, classify intent, pick mode
      └─> steps/step-01-search.md     execute the right Exa tool per mode
          └─> steps/step-02-deep.md   (only if --deep) gap analysis + re-query loop
              └─> steps/step-03-report.md   synthesize, cite, format, save
```

Each step is its own file so Claude only loads what is relevant (progressive disclosure - keeps token cost low for simple queries).

Reference files used during the pipeline:

- `references/domain-presets.md` - curated domain lists per ecosystem (ex: official docs sites per language, news sources, academic databases).
- `references/query-patterns.md` - how to phrase queries for each Exa tool to maximize result quality.
- `references/info-text.md` - the `--info` tutorial text (in French in the canonical source).

---

## Modes (full reference)

Modes are mutually exclusive. Default is `quick` when no mode flag is given.

| Flag | Underlying Exa tool | Best for |
|------|---------------------|----------|
| *(none)* | `web_search_exa` | Quick factual lookup, one-shot answer |
| `--deep` | `web_search_exa` (looped 1-3 passes) | Decision-grade research, multi-angle |
| `--code <q>` | `get_code_context_exa` | API snippets, library usage, patterns |
| `--docs <lib>` | `crawling_exa` (find official site + crawl) | Read library docs end-to-end |
| `--debug <err>` | `web_search_exa` (Stack Overflow + GitHub Issues) | Error troubleshooting |
| `--news <topic>` | `web_search_advanced_exa` category=news | Recent news, default 7 days back |
| `--compare A vs B` | parallel `web_search_exa` per option | Tech choice, side-by-side |
| `--research <topic>` | `web_search_advanced_exa` category=research paper | Academic papers (arxiv, ACM, IEEE, Nature) |
| `--similar <url>` | `web_search_exa` similar mode | Find semantically similar pages |
| `--info` | (none) | Display the full tutorial |

### Mode details

**quick** (default)

```
/websearch how does HTTP/3 differ from HTTP/2
```

One pass, 5 results, output as bullet points with citations. Optimized for factual questions.

**deep**

```
/websearch --deep state management React 2026
```

Decomposes the query into 3-5 sub-queries by angle (definitional, practical, comparative, recent, expert). Evaluates results after each pass, identifies gaps, iterates up to 3 passes max. Output is a structured report with thematic sections. Cost: 15-30k tokens. Use for serious research, technical decisions, deep dives.

**code**

```
/websearch --code FastAPI middleware authentication
/websearch --code Python asyncio gather timeout
```

Uses `get_code_context_exa`, biased toward GitHub, Stack Overflow, official docs. Output: code snippets first, explanation second.

**docs**

```
/websearch --docs prisma
/websearch --docs next.js app router
```

Two-step workflow: (1) find the official docs site, (2) crawl the relevant page + linked subpages. Best for reading docs without leaving the terminal.

**debug**

```
/websearch --debug "TypeError: Cannot read properties of undefined (reading 'map')"
/websearch --debug CORS error preflight blocked
```

Smart workflow: (1) clean the error (strip paths, timestamps, user data), (2) search the exact error first, (3) broaden semantically if too few results, (4) target Stack Overflow + GitHub Issues. Output: solution first, explanation second.

**news**

```
/websearch --news AI agents
/websearch --news --after 2026-05-01 Claude Code
```

Uses Exa's `category=news` filter. Default 7-day window. Output: timeline, most recent first.

**compare**

```
/websearch --compare React vs Vue vs Svelte
/websearch --compare Prisma vs Drizzle
```

Launches one parallel query per option. Output: comparison table + verdict.

**research**

```
/websearch --research transformer architecture scaling laws
```

Filters to academic papers (arxiv, ACM, IEEE, Nature). Output: summary + key findings + methodology per paper.

**similar**

```
/websearch --similar https://exa.ai/blog/search-for-ai
```

Finds pages semantically similar to a given public URL.

---

## Filters

Filters combine with any mode.

| Flag | Effect |
|------|--------|
| `--after <date>` | Results published after this date. Formats: `2026-05-01`, `"last week"`, `"3 days ago"`. |
| `--before <date>` | Results published before this date. |
| `--domain <d>` | Restrict to domains (comma-separated). Ex: `--domain github.com,stackoverflow.com`. |
| `--exclude <d>` | Exclude domains. Ex: `--exclude reddit.com,medium.com`. |
| `--fresh` | Force live crawl (bypass Exa's cache, get real-time content). |
| `--locale <CC>` | Localize results. ISO country code. Ex: `--locale FR`, `--locale US`. |
| `-n <num>` | Number of results (default 5, max 20). |

---

## Output options

| Flag | Effect |
|------|--------|
| `--save <file>` | Save the report to a markdown file. Ex: `--save ./report.md`. |
| `--json` | Structured JSON output (good for piping to another tool). |
| `--full` | Full text instead of highlights. Roughly doubles tokens but gives more context. |

---

## Examples by use case

**Daily factual question**

```
/websearch quel est le port par défaut de PostgreSQL
```

**Find a code snippet**

```
/websearch --code React useOptimistic example
```

**Debug an error**

```
/websearch --debug "ECONNREFUSED 127.0.0.1:5432"
```

**Read a library's docs**

```
/websearch --docs zod validation
```

**Tech watch (last week)**

```
/websearch --news --after "last week" Anthropic Claude
```

**Tech choice**

```
/websearch --compare Bun vs Deno vs Node.js 2026
```

**Deep research with report saved**

```
/websearch --deep --save rapport.md impact IA sur le développement logiciel
```

**Recent papers**

```
/websearch --research --after 2026-01-01 LLM code generation evaluation
```

**Find similar content**

```
/websearch --similar https://exa.ai/blog/search-for-ai
```

**Restrict to specific domains**

```
/websearch --domain github.com,stackoverflow.com rust async runtime
```

---

## Token cost

Orders of magnitude per command (input + output combined):

| Mode | Tokens (approx) |
|------|-----------------|
| quick | 2-5k |
| code / docs / debug | 3-8k |
| news / compare | 4-10k |
| research | 5-12k |
| deep | 15-30k |

`--full` roughly doubles tokens vs default highlights. `-n 20` adds ~3-5k vs default `-n 5`.

---

## Limitations & known issues

- **Exa-only by design.** No fallback to native `WebSearch` or `WebFetch`. Intentional - keeps citation discipline and lets the skill assume Exa's filter syntax.
- **Deep mode caps at 3 passes.** Hard limit to bound cost.
- **`--similar` requires a public URL.** Authenticated pages will not work.
- **News freshness** depends on Exa's index latency (typically minutes to hours, not seconds).
- **Tool names may shift.** Exa periodically renames or deprecates tools. The skill uses canonical names and known aliases, but if Exa renames again, edit `steps/step-01-search.md` to update.
- **Free tier rate limits.** 1,000 searches/month on free Exa plan. `--deep` can burn 3-5 searches in one command.

---

## Troubleshooting

**`mcp__exa__*` tools not found in session**

- Run `claude mcp list` and confirm `exa` is connected.
- Restart Claude Code (`exit`, relaunch). MCP connections initialize at session start.
- Check `~/.claude.json` contains the Exa entry.

**"API key invalid" or 401 errors**

- Regenerate at [dashboard.exa.ai/api-keys](https://dashboard.exa.ai/api-keys).
- Update env var or hosted URL.
- Verify with: `curl -H "x-api-key: YOUR_KEY" https://api.exa.ai/search -d '{"query":"test"}'`.

**Skill installed but `/websearch` not recognized**

- Run `/plugin list` and confirm `websearch@arsenal` is installed.
- Restart Claude Code. Slash commands register at session start.

**Search returns no results**

- Try without filters first. Filters are aggressive.
- Try `--fresh` if you suspect stale cache.
- Try a different mode. `--debug` for errors, `--code` for code, not all in `quick`.

---

## Architecture

```
arsenal/                                repo root (this marketplace)
├── .claude-plugin/
│   └── marketplace.json                      lists the plugin
└── plugins/
    └── websearch/                            the plugin
        ├── .claude-plugin/
        │   └── plugin.json                   plugin manifest, semver
        ├── skills/
        │   └── websearch/                    the actual skill
        │       ├── SKILL.md                  entry point, defines flags + state
        │       ├── steps/
        │       │   ├── step-00-triage.md     parse, classify intent
        │       │   ├── step-01-search.md     execute Exa tool per mode
        │       │   ├── step-02-deep.md       deep iteration loop
        │       │   └── step-03-report.md     synthesize, cite, save
        │       └── references/
        │           ├── domain-presets.md     curated domain lists
        │           ├── query-patterns.md     query formulation per tool
        │           └── info-text.md          --info tutorial text
        └── README.md                         this file
```

Progressive disclosure: Claude only loads `SKILL.md` initially (~80 lines). Steps and references load on demand based on which mode is triggered. A simple `/websearch what is X` never touches `step-02-deep.md` or the references.

---

## Contributing

Issues and PRs welcome at [github.com/mirkobozzetto/arsenal](https://github.com/mirkobozzetto/arsenal).

Keep changes focused: one mode, one flag, or one ref file per PR.

If you add a new mode:

1. Update `SKILL.md` parameters section.
2. Update `steps/step-00-triage.md` to recognize the flag.
3. Update `steps/step-01-search.md` with the dispatch branch.
4. Add an example to `references/info-text.md`.
5. Bump `plugin.json` version (minor bump for new mode, patch for tweaks).
6. Update this README's mode table.

---

## License

MIT - see [LICENSE](../../LICENSE).
