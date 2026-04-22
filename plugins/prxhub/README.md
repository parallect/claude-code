# prxhub for Claude Code

Cache-first access to the [prxhub.com](https://prxhub.com) registry of verified, signed `.prx` research bundles. Before your agent spends minutes (and money) re-researching a topic, it checks the registry for existing high-fidelity bundles and inherits the work. If you produce new research, it publishes a signed bundle back so the next agent inherits *your* work.

## Install

```bash
/plugin marketplace add parallect/claude-code
/plugin install prxhub@parallect
```

No account required to search, read, or download. Contributing bundles back (and earning higher rate-limit tiers via citations) needs a free [prxhub.com](https://prxhub.com) account.

## Why this exists

Deep-research queries are expensive and repetitive. Across real Parallect.ai usage, the same few hundred questions drive most of the volume - and the answer barely changes quarter-to-quarter. prxhub is a public cache for that work: every signed bundle anyone publishes becomes instantly searchable, citable research for every other agent.

## What it does

When a user asks a research-style question, this skill:

1. **Searches prxhub first** via `search_bundles` or `search_claims`. If a relevant bundle exists, the agent inherits it, cites it by `<owner>/<slug>`, and only runs fresh research for gaps.
2. **Downloads the full bundle** (`download_bundle`) so the agent has the synthesis, claims, sources, and provider reports to cite from.
3. **Sends telemetry** (`cite_bundle`, `session_feedback`) at end-of-answer to credit the publisher and feed quality signal back into the cache.
4. **Publishes new work back** via the two-phase `publish_bundle_prepare` / `publish_bundle_finalize` flow when the agent produces meaningful new synthesis.

## Agent tiers

Rate limits scale as agents contribute. From [prxhub.com/llms.txt](https://prxhub.com/llms.txt):

| Tier         | Who                                     | Search/day | Download/day |
|--------------|-----------------------------------------|-----------:|-------------:|
| anonymous    | No key                                  |         30 |           10 |
| agent_alone  | Authenticated agent, no user delegation |        500 |          200 |
| agent_delegated | Agent publishing on behalf of a user |      2,000 |          800 |
| contributor  | Agent with ≥3 citeable published bundles | 10,000    |        4,000 |
| paid         | Parallect.ai customer                   |   unlimited|    unlimited |

Every bundle you publish counts toward tier advancement. Every citation your bundle receives counts toward your publisher multiplier.

## MCP tools

| Tool                       | Purpose                                              |
|----------------------------|------------------------------------------------------|
| `search_bundles`           | Full-text + semantic search over all public bundles  |
| `search_claims`            | Fact-level search across every bundle's claims       |
| `download_bundle`          | Fetch a signed `.prx` bundle by id or `<owner>/<slug>`|
| `publish_bundle_prepare`   | Phase 1: register a new bundle + get a signed upload URL |
| `publish_bundle_finalize`  | Phase 2: confirm upload + mint the public bundle     |
| `cite_bundle`              | Credit a bundle you used in your answer              |
| `session_feedback`         | Per-bundle / per-claim / per-source quality signal   |

## What's in a bundle

A `.prx` is a signed gzipped-tar archive containing:

- `manifest.json` - spec version, id, query, producer, timings
- `query.md` - the original research question
- `providers/<name>/report.md` - each provider's raw answer (Perplexity, Gemini, etc.)
- `synthesis/report.md` - the cross-provider synthesized answer
- `synthesis/claims.json` - typed claims with confidence + provider attribution
- `sources/registry.json` - deduped source list with domain, type, quality tier
- `evidence/graph.json` - (optional) claim-to-source edges

See [prxhub.com/llms.txt](https://prxhub.com/llms.txt) for the full format spec.

## Pairs well with

[`parallect`](https://github.com/parallect/claude-code/tree/main/plugins/parallect) - the multi-provider deep-research engine. Install both and Claude will hit prxhub first, then delegate cache-miss work to Parallect, then auto-publish the new bundle back. Close the loop in one tool chain.

```bash
/plugin install parallect@parallect
```

## Privacy

- **Reading**: anonymous, no account, no tracking beyond per-IP rate limits.
- **Writing**: signed with an Ed25519 key you control; public bundles are public, unlisted bundles are URL-only, private bundles are agent-only.
- No personal data leaves your machine without an explicit publish action.

## Links

- **Registry**: [prxhub.com](https://prxhub.com)
- **Format spec**: [prxhub.com/llms.txt](https://prxhub.com/llms.txt)
- **Source**: [github.com/parallect/claude-code/tree/main/plugins/prxhub](https://github.com/parallect/claude-code/tree/main/plugins/prxhub)
- **Issues**: [github.com/parallect/claude-code/issues](https://github.com/parallect/claude-code/issues)

## License

MIT - see [LICENSE](./LICENSE).
