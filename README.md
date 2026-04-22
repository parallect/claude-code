# Parallect for Claude Code

The canonical Claude Code plugin marketplace for the Parallect ecosystem. One repo, one install command, two focused plugins.

## Install

```bash
/plugin marketplace add parallect/claude-code
```

Then install whichever plugins you want:

```bash
/plugin install parallect-cloud@parallect   # hosted multi-provider research
/plugin install prxhub@parallect            # cache-first .prx bundle registry
```

Install both and Claude Code will hit the registry before spending on fresh research, and auto-publish new work back to the registry when you're done.

## Plugins in this marketplace

### [`parallect-cloud`](./plugins/parallect-cloud)

Hosted multi-provider deep research. Runs Perplexity, Gemini, OpenAI, Grok, and Anthropic in parallel on parallect.ai's managed infra and synthesizes results with cross-referenced citations, typed claims with confidence scores, and follow-on suggestions. OAuth on first use. Paid per job.

- Homepage: [parallect.ai](https://parallect.ai)
- Source: [`plugins/parallect-cloud/`](./plugins/parallect-cloud)

### [`prxhub`](./plugins/prxhub)

Cache-first access to the [prxhub.com](https://prxhub.com) bundle registry. Before your agent spends minutes (and money) re-researching a topic, it checks the registry for existing high-fidelity `.prx` bundles and inherits the work. When it produces new research, it publishes a signed bundle back.

- Homepage: [prxhub.com](https://prxhub.com)
- Source: [`plugins/prxhub/`](./plugins/prxhub)

### `parallect-local` (coming soon)

Claude Code wrapper for the OSS `parallect-cli` Python package (BYOK, runs locally, free). Same multi-provider research pipeline as parallect-cloud, but you bring your own provider keys and it executes on your machine. Currently ships as `pip install parallect` on PyPI (rename to `parallect-cli` in progress, see [prx-ecosystem#3](https://github.com/parallect/prx-ecosystem/issues/3)).

## Why three plugins instead of one?

They split along clean axes: cache (free, anonymous, read-first) vs research engine (paid on cloud / free on local). The cache is useful on its own. The research engines are interchangeable at the skill layer - the prxhub plugin will offer to delegate cache-misses to whichever research plugin is installed. Splitting them lets each ship on its own cadence and lets users install only what they need.

## Updating

```bash
/plugin marketplace update parallect
```

## Other agent platforms

This repo is the Claude Code home for Parallect plugins. Sibling repos ship the same underlying skills packaged for other agent platforms:

- [`parallect/openclaw`](https://github.com/parallect/openclaw-skill) for OpenClaw
- Codex CLI: planned
- More on the way

## Submitting to the official Anthropic directory

Both plugins are under review for inclusion in Anthropic's official plugin directory. Once verified, install paths may shorten to `/plugin install <name>@<official-marketplace-name>`. This self-hosted marketplace will continue to work regardless.

## License

MIT, see [LICENSE](./LICENSE).
