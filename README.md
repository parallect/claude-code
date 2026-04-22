# Parallect marketplace for Claude Code

The canonical Claude Code plugin marketplace for the Parallect ecosystem. Two focused plugins, independently versioned, one install flow.

## Install

```bash
/plugin marketplace add parallect/claude-code-plugin
```

Then install whichever plugins you want:

```bash
/plugin install parallect@parallect   # multi-provider deep research
/plugin install prxhub@parallect      # cache-first .prx bundle registry
```

Install both and Claude Code will hit the registry before spending on fresh research, and auto-publish new work back to the registry when you're done.

## Plugins in this marketplace

### [`parallect`](https://github.com/parallect/parallect-plugin)

Multi-provider deep research. Runs Perplexity, Gemini, OpenAI, Grok, and Anthropic in parallel and synthesizes results with cross-referenced citations, typed claims with confidence scores, and follow-on suggestions.

- Homepage: [parallect.ai](https://parallect.ai)
- Source: [parallect/parallect-plugin](https://github.com/parallect/parallect-plugin)

### [`prxhub`](https://github.com/parallect/prxhub-plugin)

Cache-first access to the [prxhub.com](https://prxhub.com) bundle registry. Before your agent spends minutes (and money) re-researching a topic, it checks the registry for existing high-fidelity `.prx` bundles and inherits the work. When it produces new research, it publishes a signed bundle back.

- Homepage: [prxhub.com](https://prxhub.com)
- Source: [parallect/prxhub-plugin](https://github.com/parallect/prxhub-plugin)

## Why two plugins instead of one?

They're logically distinct products. The registry is free and anonymous — anyone can read. Parallect is the paid research engine. Splitting them lets you install only what you need and lets each ship on its own cadence. When installed together, the prxhub skill will offer to delegate cache-misses to parallect automatically.

## Updating

```bash
/plugin marketplace update parallect
```

Each plugin pulls its latest release from its own repo.

## Submitting to the official Anthropic directory

Both plugins are also under review for inclusion in Anthropic's official plugin directory. Once verified, install paths may shorten to `/plugin install <name>@<official-marketplace-name>`. This self-hosted marketplace will continue to work regardless.

## License

MIT — see [LICENSE](./LICENSE).
