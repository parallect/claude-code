# Parallect Plugin for Claude Code

Multi-provider deep research inside [Claude Code](https://claude.ai/code). Searches [prxhub](https://prxhub.com)'s public research registry first — free, no account. Only falls back to paid [Parallect.ai](https://parallect.ai) research if the registry doesn't have the answer.

## Install

Run these two commands in Claude Code (or in a terminal with the `claude` CLI):

```bash
claude plugin marketplace add parallect/claude-code-plugin
claude plugin install parallect@parallect
```

Or, from inside an interactive Claude Code session, use the slash-command equivalents:

```
/plugin marketplace add parallect/claude-code-plugin
/plugin install parallect@parallect
```

**Restart Claude Code after install** so the MCP servers and skill are loaded into your session.

## What it does

Once installed, Claude Code can:

- **Search prxhub first** — before spending any money, the skill searches prxhub's public registry for existing research bundles on your topic. If someone already published a high-quality answer, you get it for free with a citable `<owner>/<slug>` link.
- **Run deep research** on topics not already in the registry, fanning out across Perplexity, Gemini, OpenAI, Grok, and Anthropic in parallel and synthesizing their findings with cross-provider conflict resolution.
- **Extract claims** with provider attribution (which AI found what, with what confidence).
- **Detect divergence** across providers — ~85% of findings are unique to one provider, which is the argument for multi-provider in the first place.
- **Suggest follow-ons** and publish the finished bundle back to prxhub so the next person benefits.

## Usage

Just ask naturally:

- "Research the competitive landscape for AI code editors."
- "Deep dive on GLP-1 receptor agonists in NASH treatment."
- "What do we know about EU autonomous-vehicle regulation?"
- "Look up best practices for API rate limiting in 2026."

Claude will search prxhub, report whether the answer already exists, and only spend money if you're filling a gap.

## Authentication

- **prxhub** — public read, no account required. Works out of the box.
- **Parallect.ai** — OAuth2 on first use. Claude Code opens your browser to authorize with your Parallect account. No API keys to copy.

## Tiers — real production numbers

The product exposes five tier names. Each maps to a backing `(budgetTier, mode)` pair the API understands. Numbers below are from 300+ completed production jobs through 2026-04-20.

| Tier | API values | Budget ceiling | Typical cost (p50) | Typical duration (p50 → p90) | Best for |
|------|-----------|----------------|----------------------|-------------------------------|----------|
| **Nano** | `XXS` + `fast` | $1.50 | $1.59 | 4 min → 11 min | Quick fact-check |
| **Lite** | `XS` + `fast` | $2.50 | $3.42 | 18 min → 32 min | Light research |
| **Normal** (default) | `M` + `fast` | $5.00 | $5.06 | 15 min → 33 min | Verified daily-driver |
| **Deep** | `L` + `methodical` | $15.00 | $7.80 | 28 min → 47 min | Sequential cross-check |
| **Max** | `XL` + `methodical` | $30.00 | $8.96 | 31 min → 69 min | Investment-grade |

The budget ceiling is a cap, not an estimate — real cost is typically 30–60% of the ceiling for Deep and Max. Claude always discusses budget before spending money.

## MCP tools

### prxhub (free, unauthenticated)

| Tool | Purpose |
|------|---------|
| `search_bundles` | Find existing `.prx` research bundles by relevance (semantic + full-text + claim rollup) |
| `search_claims` | Find specific claims across all public bundles with confidence levels |
| `download_bundle` | Get a presigned URL for a bundle's raw `.prx` archive |

### Parallect (OAuth2, billable)

| Tool | Purpose |
|------|---------|
| `research` | Submit a new research query (async) |
| `research-status` | Poll job progress |
| `get-results` | Retrieve completed synthesis |
| `follow-up` | Pursue follow-on questions in the same thread |
| `list-threads` | List recent research threads |
| `get-thread` | Get full context of a prior thread |
| `balance` | Check credit balance |
| `usage` | View spend analytics |
| `list-providers` | See providers available at each tier |
| `search-claims` | Search claims across YOUR past private research (use prxhub's `search_claims` for public) |
| `get-claim-evidence` | Get evidence backing a specific claim |

## Troubleshooting

### The plugin doesn't show up after install

Restart Claude Code. Plugins are only loaded at session start.

Then verify:

```bash
claude plugin list                 # should show parallect@parallect enabled
```

Inside Claude Code, `/mcp` should list both `parallect` and `prxhub` as connected servers.

### "I get 'Hello from prx!' when I run `prx publish`"

That's an unrelated squatter package on PyPI. Our CLI is published as **`prx-cli`**:

```bash
pip uninstall prx
pip install prx-cli
```

### Updating the plugin

```bash
claude plugin marketplace update parallect
claude plugin update parallect@parallect
```

Restart Claude Code after updating.

## Links

- [Parallect.ai](https://parallect.ai) — dashboard, billing, research history
- [prxhub](https://prxhub.com) — the public research registry
- [Documentation](https://parallect.ai/docs) — API and MCP reference
- [Open-source CLI (`parallect`)](https://github.com/parallect/parallect) — BYOK multi-provider research from your terminal
- [Hub CLI (`prx-cli`)](https://github.com/parallect/prx) — `.prx` format toolkit + prxhub publish/pull

## License

MIT
