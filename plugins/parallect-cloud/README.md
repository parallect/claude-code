# parallect-cloud

Claude Code plugin for [parallect.ai](https://parallect.ai) (hosted). Runs multi-provider deep research on Parallect's managed infra - Perplexity, Gemini, OpenAI, Grok, and Anthropic in parallel - and synthesizes the results into a unified report with cross-referenced citations, typed claims with confidence scores, and follow-on suggestions.

Looking for the BYOK / self-hosted path? See [`parallect-cli`](https://pypi.org/project/parallect/) on PyPI (local, free, bring your own provider keys). The matching Claude Code plugin, `parallect-local`, is coming soon.

## Install

```bash
/plugin marketplace add parallect/claude-code
/plugin install parallect-cloud@parallect
```

Restart Claude Code when prompted; the `parallect-cloud` skill will auto-engage on research-style questions.

## What it does

When Claude Code encounters a research-style question ("what do we know about...", "deep dive on...", "competitive landscape for..."), it:

- Checks your **balance** and proposes a budget tier
- Submits the query to multiple providers **in parallel**
- Polls until complete and returns a **synthesized report**
- Surfaces **typed claims** (fact, prediction, opinion, consensus, dispute) with **confidence scores** and provider attribution
- Flags **provider divergence** - ~87% of findings are unique to a single provider
- Offers **follow-on queries** pre-scoped with sensible tiers

## Usage

Just ask naturally:

- "Research the competitive landscape for AI code editors."
- "Deep dive on GLP-1 receptor agonists in NASH treatment."
- "What's the regulatory landscape for autonomous vehicles in the EU right now?"
- "Look up best practices for API rate limiting in 2026."

Claude will discuss the budget tier before spending, then run the job and deliver a cited synthesis.

## Authentication

OAuth2. On first use, Claude Code opens your browser to authorize with your Parallect.ai account. No API keys to paste or rotate.

## Budget tiers

| Tier | Max cost | Providers | Duration  | Best for                  |
|------|----------|-----------|-----------|---------------------------|
| XXS  | ~$1      | 1         | ~4-5 min  | Quick factual lookups     |
| XS   | ~$2      | 1-2       | ~8-10 min | Brief overviews           |
| S    | ~$5      | 2         | ~15 min   | Standard questions        |
| M    | ~$15     | 3-4       | ~15-20 min| Detailed research         |
| L    | ~$30     | 4-5       | ~20-25 min| Comprehensive analysis    |
| XL   | ~$60     | All       | ~30-45 min| Exhaustive multi-provider |

Claude always confirms the tier with you before spending.

## MCP tools

| Tool                  | Purpose                                |
|-----------------------|----------------------------------------|
| `research`            | Submit a research query (async)        |
| `research_status`     | Poll job progress                      |
| `get_results`         | Retrieve completed synthesis           |
| `follow_up`           | Pursue a follow-on question            |
| `list_threads`        | List recent research threads           |
| `get_thread`          | Get full thread context                |
| `balance`             | Check credit balance                   |
| `usage`               | View spend analytics                   |
| `list_providers`      | See available providers and tiers      |
| `search_claims`       | Search claims across past research     |
| `get_claim_evidence`  | Get evidence backing a claim           |

## Pairs well with

[`prxhub`](https://github.com/parallect/claude-code/tree/main/plugins/prxhub) - the cache-first registry of shared `.prx` research bundles. Install both and Claude will check the registry *before* spending on fresh research, and offer to publish new findings back when you're done.

```bash
/plugin install prxhub@parallect
```

## Other Parallect paths

- **`parallect-cli`** (on PyPI, `pip install parallect` today; renaming to `parallect-cli` - see [prx-ecosystem#3](https://github.com/parallect/prx-ecosystem/issues/3)) - same research pipeline, runs locally, you bring the provider keys. Free to use; you pay providers directly.
- **`parallect-local`** (coming soon) - Claude Code wrapper around `parallect-cli` for users who prefer BYOK over OAuth.

## Links

- **Dashboard & billing**: [parallect.ai](https://parallect.ai)
- **Bundle registry**: [prxhub.com](https://prxhub.com)
- **Issues**: [github.com/parallect/claude-code/issues](https://github.com/parallect/claude-code/issues)
- **Terms**: [parallect.ai/terms](https://parallect.ai/terms)

## License

MIT - see [LICENSE](./LICENSE).
