# Parallect Plugin for Claude Code

Multi-provider deep research for [Claude Code](https://claude.ai/code). Queries Perplexity, Gemini, OpenAI, Grok, and Anthropic in parallel, then synthesizes results into a unified report with cross-referenced citations and conflict resolution.

## Install

```bash
/plugin marketplace add parallect/claude-code-plugin
```

Or add to your project's `.mcp.json`:

```json
{
  "mcpServers": {
    "parallect": {
      "type": "http",
      "url": "https://parallect.ai/api/mcp/mcp"
    }
  }
}
```

## What it does

When installed, Claude Code can:

- **Run deep research** on any topic using multiple AI providers simultaneously
- **Synthesize findings** with cross-referenced citations and conflict resolution
- **Extract claims** with provider attribution (which AI found what)
- **Detect divergence** across providers (87% of findings are unique to one provider)
- **Suggest follow-ons** for deeper investigation

## Usage

Just ask naturally:

- "Research the competitive landscape for AI code editors"
- "Deep dive on GLP-1 receptor agonists in NASH treatment"
- "What do we know about the regulatory implications of autonomous vehicles in the EU?"
- "Look up best practices for API rate limiting in 2026"

Claude will check your balance, discuss budget, submit the research, poll for results, and deliver a synthesis with citations.

## Authentication

OAuth2. On first use, Claude Code opens your browser to authorize with your Parallect.ai account. No API key to copy.

## Budget Tiers

| Tier | Max Cost | Providers | Duration | Best for |
|------|----------|-----------|----------|----------|
| XXS | ~$1 | 1 | ~4-5 min | Quick factual lookups |
| XS | ~$2 | 1-2 | ~8-10 min | Brief overviews |
| S | ~$5 | 2 | ~15-18 min | Standard questions |
| M | ~$15 | 3-4 | ~15-20 min | Detailed research |
| L | ~$30 | 4-5 | ~20-25 min | Comprehensive analysis |
| XL | ~$60 | All | ~30-45 min | Exhaustive multi-provider |

Claude always discusses budget before spending your money.

## MCP Tools

| Tool | Purpose |
|------|---------|
| `research` | Submit a research query (async) |
| `research-status` | Poll job progress |
| `get-results` | Retrieve completed synthesis |
| `follow-up` | Pursue follow-on questions |
| `list-threads` | List recent research threads |
| `get-thread` | Get full thread context |
| `balance` | Check credit balance |
| `usage` | View spend analytics |
| `list-providers` | See available providers and tiers |
| `search-claims` | Search claims across past research |
| `get-claim-evidence` | Get evidence backing a claim |

## Links

- [Parallect.ai](https://parallect.ai) -- Dashboard, billing, research history
- [Documentation](https://parallect.ai/docs) -- API and MCP reference
- [Open Source CLI](https://github.com/parallect/parallect) -- BYOK multi-provider research from your terminal

## License

MIT
