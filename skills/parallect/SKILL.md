---
name: parallect
description: >
  Deep research using Parallect.ai + prxhub. Before spending compute, search
  prxhub's public registry for existing research on the topic (free, no
  account). If nothing covers the question — or only partially covers it —
  run new multi-provider research on Parallect to fill the gap. Queries fan
  out to Perplexity, Gemini, OpenAI, Grok, and Anthropic in parallel, then
  synthesize into a unified report with cross-referenced citations and
  conflict resolution. Use when the user wants to research a topic,
  investigate a question, or needs sourced multi-perspective analysis. Also
  triggers on "look this up", "research this", "find out about", "deep
  dive on", "what do we know about". Do NOT use for simple factual
  questions answerable from memory — only for topics requiring sourced,
  multi-perspective analysis.
user-invocable: true
allowed-tools: MCP(parallect:*) MCP(prxhub:*)
---

# Parallect Deep Research

You have access to two research systems:

- **prxhub** — an open registry of existing research bundles (`.prx` format)
  with cryptographic provenance. **Read-only, no account required for
  public bundles.** Always search prxhub first.
- **Parallect.ai** — a multi-provider deep research platform that queries
  Perplexity, Gemini, OpenAI, Grok, and Anthropic in parallel and
  synthesizes their findings. Requires OAuth2 auth and costs real money.

Research is **asynchronous** — jobs take 4 to 90+ minutes depending on
tier and providers. You MUST poll for completion. Never block.

## The golden rule: search before you research

Spending user money to re-run research that someone already published is
wasteful. **Every research request starts with a prxhub search.** If a
relevant high-trust bundle already exists, you may have the answer for
free. If only part of the question is covered, scope new research to
fill the gap — not duplicate what's already there.

Workflow:

1. **Search prxhub** (`search_bundles`, `search_claims`) — always, no
   exceptions, no cost.
2. **Analyze what exists** — is there a bundle that answers this? Does
   it cover everything, or only some angles?
3. **If fully covered**: summarize the existing bundle and hand its
   `<owner>/<slug>` to the user. Offer to download it with
   `download_bundle`. Done. No Parallect call needed.
4. **If partially covered**: scope a new Parallect query specifically
   to the gaps (what the existing bundle didn't answer). Tell the user
   what's already known and what the new research will add.
5. **If nothing relevant**: run fresh Parallect research on the full
   question.

Never skip step 1. The user should never pay for research that's
already publicly available.

## Connection

Both MCPs are registered via the plugin's `.mcp.json`:

```json
{
  "mcpServers": {
    "parallect": { "url": "https://parallect.ai/api/mcp/mcp" },
    "prxhub":    { "url": "https://prxhub.com/api/mcp" }
  }
}
```

- **prxhub** — public, unauthenticated for read tools. No setup.
- **Parallect** — OAuth2 on first use. The user authorizes via their
  browser the first time you call a parallect tool; it persists after.

## When to use hosted Parallect.ai vs local BYOK

Two genuinely different paths. Pick based on user preference:

| | Hosted (parallect.ai SaaS) | Local BYOK (OSS CLI) |
|---|---|---|
| Provider keys | Parallect holds them, consolidated billing | User brings their own OpenAI / Anthropic / Gemini / Grok / Perplexity keys |
| Auth | OAuth2 via browser (one-time) | No server auth — env vars / config file |
| Available here | ✅ via MCP tools | ❌ shell access required |
| Bundle output | Served by `/api/v1/jobs/<jobId>/prx` (Parallect stores it) | Lands on local disk, `-o bundle.prx` |
| Research history | Persisted to user's dashboard | Local only — user's machine keeps it |
| Session-feedback loop | Closable via MCP tools | Closable via hybrid flow (see below) |

Ask the user which they prefer BEFORE the first research call if both
are plausible. "Do you want to use Parallect.ai's hosted keys or your
own API keys locally?" The answer determines every downstream step.

### Hosted path (MCP tools available — default for this skill)

Everything described in the rest of this document: `research`,
`research-status`, `get-results`, `follow-up`, etc. via the
`parallect` MCP server. Publish via Option C of the prxhub skill
(`publish:bundles` delegation + two-phase publish).

### Local BYOK path (requires shell access)

User installs the OSS CLIs and configures their own provider keys:

```bash
pip install parallect prx-cli
```

Provider keys go in env vars OR `~/.config/parallect/config.toml`:

```bash
export PARALLECT_OPENAI_API_KEY=sk-...
export PARALLECT_ANTHROPIC_API_KEY=sk-ant-...
export PARALLECT_GOOGLE_API_KEY=AIza...
export PARALLECT_PERPLEXITY_API_KEY=pplx-...
export PARALLECT_XAI_API_KEY=xai-...         # Grok
```

Or once via `parallect config` (interactive TUI).

Run research locally:

```bash
parallect research "<structured query>" \
  -p openai,anthropic,gemini \
  --deep \                     # selects premium models (o3, opus-4, 2.5-pro)
  -o research.prx              # lands on disk at ./research.prx
```

Omit `--deep` for the default fast tier if cost matters.

Configure prx-cli for publishing (one-time):

```bash
prx auth login --scope publish:bundles   # device flow, opens browser
prx keys generate                         # Ed25519 signing key
```

Publish:

```bash
prx publish research.prx \
  --visibility public \
  --collection <slug> \
  --tags "tag1,tag2"
```

`prx publish` wraps the two-phase `publish_bundle_prepare` +
`publish_bundle_finalize` flow internally. The user doesn't call
those tools manually. Bundle lands at `prxhub.com/<user>/<slug>`.

### Hybrid: keep the session-feedback loop closable

CLI-only local research **severs the telemetry loop** by default —
because no prxhub `session_id` ever exists to bind citations +
feedback to. To preserve the loop while using local BYOK research,
interleave MCP calls around the shell steps:

```
1. prxhub MCP: search_bundles(query)   → capture session_id
2. (shell) parallect research ... -o research.prx
3. (shell) prx publish research.prx --collection <slug>
   → CLI prints: "Published: <owner>/<slug>  (bundle_id: <uuid>)"
4. Parse bundle_id from step 3's stdout, OR if the CLI version
   doesn't print the UUID, resolve it with:
     prxhub MCP: search_bundles(
       query: "<the exact owner/slug string from prx publish output,
               e.g. 'alex-rivera/rust-async-2026' — pass it literally
               as the query, not a semantic paraphrase>",
       limit: 1
     )
     → results[0].bundle_id is the UUID you need
5. prxhub MCP: cite_bundle(
     citedBundleId: <bundle_id uuid>,
     sessionId: <session_id from step 1>
   )
6. prxhub MCP: session_feedback(
     sessionId: <session_id from step 1>,
     bundles: [{ bundleId: <bundle_id uuid>, useful: true, score: 5 }]
   )
```

This is the pattern to use when the user prefers BYOK but still
wants their research to contribute to the cache-quality signal.

**Extracting the bundle_id from `prx publish`:**

Recent `prx-cli` versions print the full URL + UUID in a JSON block
on success. Older versions only print the slug. If your shell wrapper
doesn't surface the UUID, the search-by-slug fallback in step 4
above always works because your newly-published bundle will be the
top hit for its own slug.

**If you don't do step 1** (go CLI-only from the start), the loop
is severed — `session_id` is null and `cite_bundle` +
`session_feedback` silently drop. The bundle still publishes and
is searchable by future agents, but THIS session contributes
nothing to the rater-reputation signal.

Tell the user this tradeoff explicitly if they're about to go
CLI-only. Most will accept it; some will want the hybrid flow.

## Gotchas

Read these first. They prevent the most common mistakes:

- **Research is async.** `research` returns a `jobId`, NOT results.
  You must poll `research-status` and then call `get-results` only after
  status is `"completed"`. Calling `get-results` early returns
  `JOB_NOT_COMPLETE`.
- **Poll with exponential backoff, not fixed intervals.** Polling every
  5 seconds burns rate limits. See Step 3.
- **You are spending the user's money at machine speed.** Never submit
  research without discussing budget first. Balance check before the
  first research of a session is mandatory.
- **`research` always creates a new thread.** Do NOT pass a `threadId`
  to `research`. For follow-ups in the same thread, use `follow-up` with
  the parent `jobId`.
- **`fast` mode skips synthesis.** It returns a single provider's raw
  report. Only use when the user explicitly prioritizes speed.
- **The `synthesis` field is markdown.** Present as-is with citations
  intact. Don't strip formatting.
- **Duration depends on tier, not just mode.** An XL methodical can run
  90+ minutes. Don't tell the user it'll be done in 5 minutes.

## Available tools

### prxhub (free, unauthenticated)

| Tool | Use when... |
|------|-------------|
| `search_bundles` | Always, before any Parallect research. Finds existing `.prx` bundles by semantic + full-text + claim relevance. |
| `search_claims` | User asks about a specific fact/claim/statistic. Searches extracted claims across all public bundles with confidence levels. |
| `download_bundle` | User wants the raw `.prx` archive for a bundle matched via search. Returns a presigned URL. |

### Parallect (OAuth2, billable)

| Tool | Use when... | Do NOT use when... |
|------|-------------|-------------------|
| `research` | Existing prxhub bundles don't cover the question | Prxhub already has it |
| `research-status` | You've submitted research and need to check progress | You haven't called `research` yet |
| `get-results` | `research-status` shows `"completed"` | Job is still running/synthesizing |
| `follow-up` | User wants to dig deeper on a completed job | No prior completed job exists |
| `list-threads` | User refers to past research or wants to resume | You already have the threadId |
| `get-thread` | You need full context of a previous session | You only need current job status |
| `balance` | Before first research of session; or after to report remaining credits | Mid-polling (wastes a call) |
| `usage` | User asks about spending history | Default — only when explicitly asked |
| `list-providers` | Before research to show user what their tier includes | Budget already discussed |
| `search-claims` | User wants to search across THEIR past private research | Public research → use prxhub `search_claims` |
| `get-claim-evidence` | User wants evidence backing a specific claim ID | Claim ID is not known |

## Tiers — the 5-tier search abstraction

The product exposes **five** tier names. Each maps to a (budgetTier, mode)
pair the MCP API accepts. **Use the UI names with the user; pass the
API values to the MCP.**

| UI tier | API `budgetTier` | API `mode` | Blurb | Advertised ceiling | Real p50 cost | Real p50 → p90 duration |
|---------|-----------------|-----------|-------|---------------------|----------------|--------------------------|
| **Nano** | `XXS` | `fast` | Quick fact-check, single-answer lookup | $1.50 | **$1.59** | **4 min → 11 min** |
| **Lite** | `XS` | `fast` | Light research, multiple sources | $2.50 | **$3.42** ⚠ | **18 min → 32 min** ⚠ |
| **Normal** (default) | `M` | `fast` | Verified daily-driver across 5 providers | $5.00 | **$5.06** | **15 min → 33 min** |
| **Deep** | `L` | `methodical` | Sequential cross-checking across 7 providers | $15.00 | **$7.80** | **28 min → 47 min** |
| **Max** | `XL` | `methodical` | Every provider, deepest analysis, investment-grade | $30.00 | **$8.96** | **31 min → 69 min** |

**Numbers are p50/p90 from 300+ completed production jobs through
2026-04-20.** The advertised ceiling is a cap, not an estimate. Deep
and Max typically cost 30–60% of the cap.

⚠ **Lite currently runs longer and costs more than advertised** because
it maps to `XS` + `fast` but the typical provider mix is slow. Don't
oversell Lite's speed. If a user asks for "fast and cheap," Nano is
more reliable.

When the user describes the work, map like this:

| User says... | UI tier |
|--------------|---------|
| "quick check", "just a glance", "fact-check" | Nano |
| "quick look", "brief", "lite" | Lite |
| "standard", "normal", "daily driver", "default" | Normal |
| "thorough", "detailed", "deep dive" | Deep |
| "comprehensive", "exhaustive", "investment-grade", "nail it", "one shot" | Max |

### How to call the MCP

The `research` tool takes **two** separate parameters — `budgetTier` and
`mode`. Set both from the tier the user picked:

```
Nano   → budgetTier: "XXS", mode: "fast"
Lite   → budgetTier: "XS",  mode: "fast"
Normal → budgetTier: "M",   mode: "fast"      ← note: fast, not methodical
Deep   → budgetTier: "L",   mode: "methodical"
Max    → budgetTier: "XL",  mode: "methodical"
```

The MCP's `budgetTier` enum is `["XXS","XS","S","M","L","XL"]`. `S` is
an unused intermediate — the 5-tier system skips it. Don't pass UI
names (`Normal`, `Max`, etc.) to the MCP; they'll be rejected.

## Workflow: running research

### Step 1 — Search prxhub first (always, no cost)

Before ANY Parallect call:

```
search_bundles(query: "<user's topic, broad>", limit: 10)
search_claims(query: "<specific claim if user asked about a fact>", limit: 10)
```

Look at the top results. Three outcomes:

**a) A bundle closely matches.** Summarize it, show the
`<owner>/<slug>`, and ask if the user wants more detail (offer
`download_bundle`) or whether this is enough. Don't spend money.

**b) Multiple partial matches.** Call out what's already covered and
what's still open. Propose new research scoped ONLY to the gaps.
Example: "There's an existing bundle on X and another on Y. Z isn't
covered — I'll research just Z."

**c) Nothing relevant.** Proceed to Step 2 with the full query.

Report the prxhub search cost transparently: "$0 — prxhub search is
free."

### Step 2 — Budget + balance check (first Parallect call only)

If Step 1 concluded new research is needed:

1. Call `balance` to check the user's credits + payment status.
2. Call `list-providers` with the anticipated tier to show what
   providers and cost range the tier covers.
3. Tell the user, using real numbers (not marketing ceilings):
   > "Prxhub doesn't have this. To answer it I'd run a **Normal** (M)
   > methodical research — typically $5-7 across 4-5 providers, ~20-30
   > minutes. Your balance is $X.XX. Proceed?"
4. If balance < required and no payment method on file, direct them to
   https://parallect.ai/settings/billing.
5. If the user set a budget preference earlier in session, reuse it
   silently unless the topic warrants different. Don't ask twice.

### Step 3 — Submit research

Call `research` with:
- `query`: the well-formed research question (see "writing great
  queries" below)
- `budgetTier`: API value (`XXS`/`XS`/`M`/`L`/`XL`), derived from the
  user-facing tier they picked
- `mode`: derived from the tier — `fast` for Nano/Lite/Normal,
  `methodical` for Deep/Max. Don't ask the user separately for mode;
  the tier choice implies it.

Save the returned `jobId` and `threadId`.

Tell the user: "Research submitted to [N] providers. I'll check back
in about 45 seconds, then every 1-2 minutes after that."

### Step 4 — Poll for completion (exponential backoff)

Research is asynchronous. Poll `research-status` using backoff:

| Check # | Approximate wait | Cumulative |
|---------|-----------------|------------|
| 1 | ~45s | 45s |
| 2 | ~60s | 1m 45s |
| 3 | ~90s | 3m 15s |
| 4 | ~120s (cap) | 5m 15s |
| 5+ | ~120s each | +2min each |

**Cap at 120 seconds. Never poll more frequently than every 30 seconds.**

Report progress naturally each check, based on `phase` and provider
completion counts:
- "1 of 4 providers finished, 3 still running..."
- "All providers done, synthesis is running now..."
- "Revising the synthesis — this is the last stage..."
- "Research complete! Fetching the results."

**Timeout expectations** (real p90 from production):
- Nano: warn past 12 min
- Lite: warn past 17 min
- Normal: warn past 35 min
- Deep: warn past 50 min
- Max: warn past 75 min, check in with user past 90 min

If the job is stuck much past p90, call `research-status` for the
latest `phase` and error info. Offer: keep polling / switch to fast
mode for a partial result / cancel.

### Step 5 — Deliver results

When `research-status` returns `"completed"`:

1. Call `get-results` with `jobId`.
2. Present findings in this order:
   - **Brief summary** (3-5 sentences) of key findings in your own
     words. Lead with the most important insight.
   - **Full synthesis** — share the `synthesis` markdown as-is. It
     contains inline citations and cross-references.
   - **Cost report** — "This research cost $X.XX across N providers
     in Y minutes."
   - **Balance check** — call `balance` and report remaining credits.

3. **Offer publishing to prxhub — always, on every cache-miss research
   path (hosted, BYOK, or imported).** At the end of every successful
   research, regardless of which path produced the `.prx`:
   > "This research could help others who ask similar questions. Want
   > to publish the bundle to prxhub? It's free and creates a citable
   > permanent URL. (Or keep it private — your call.)"

   This prompt is mandatory. Do not ship fresh research without
   offering publish — doing so silently drops the contribution to
   the cache. Applies equally whether the research came from hosted
   Parallect MCP, local parallect CLI + BYOK, or a `.prx` the user
   handed you.

   When the user pre-specified "publish to X collection" in their
   original ask, skip the prompt and go straight to publishing —
   they already opted in.

   Publish path depends on what your environment supports:

   **If you have shell access** (recommended today):

   ```bash
   parallect export --format prx <jobId> \
     | prx publish --visibility public \
         --collection <slug>         # optional
   ```

   The `prx-cli` package is `prx-cli` on PyPI (not `prx`). It handles
   SHA-256 hashing and the two-phase upload to prxhub automatically.

   **If you're pure MCP (no shell)**: the MCP `get-results` tool
   returns `synthesis` markdown + metadata but not the raw `.prx`
   bytes. HOWEVER, Parallect.ai exposes a REST endpoint that does:

   ```
   GET https://parallect.ai/api/v1/jobs/{jobId}/prx
   Authorization: Bearer <your Parallect bearer — same token your
                          MCP session uses>
   → binary .prx archive bytes
   ```

   This is the escape hatch for pure-MCP agents. The full flow:

   ```
   1. parallect MCP: research(query)           → jobId
   2. parallect MCP: research-status(jobId)    → poll to "completed"
   3. parallect MCP: get-results(jobId)        → synthesis for user
   4. fetch GET /api/v1/jobs/<jobId>/prx       → .prx bytes
   5. compute SHA-256 + byte_size locally
   6. prxhub MCP: publish_bundle_prepare(...)
   7. fetch: PUT <upload_url> with the bytes
   8. prxhub MCP: publish_bundle_finalize(...)
   9. prxhub MCP: cite_bundle + session_feedback
   ```

   The full step-by-step is documented in the prxhub skill's
   "Publishing a bundle — how it works" section. Both skills now
   agree: pure-MCP cache-miss → publish is wireable end-to-end as
   long as your MCP client surfaces the Parallect bearer token to
   `fetch` (most do — it's a standard MCP client config).

   If your runtime for some reason cannot fetch with the Parallect
   bearer (unusual), fall back to shell: `parallect export --format
   prx <jobId> | prx publish`. Or deliver the synthesis and tell the
   user "publish requires shell in this environment."

   Do NOT fabricate a `publish_target: "prxhub"` parameter on
   `research` — that is not a supported Parallect tool argument. The
   REST endpoint above is the documented path.

### Step 6 — Follow-ons

Results include `followOnSuggestions`. Present as numbered options:
> "Based on this research, here are directions we could explore next:
>  1. [suggestion 1]
>  2. [suggestion 2]
>  3. [suggestion 3]
>
>  Want me to research any of these, or ask a different follow-up?"

When user picks one, use `follow-up` with the parent `jobId` (NOT
`research`). Also: re-check prxhub first — a follow-on question may
have a different answer already in the registry.

## Writing great research queries — this matters a lot

Multi-provider research only works well when the query fans out. A
one-line question gets one-line thinking from each provider, then the
synthesis has nothing to reconcile. **A structured query gets
structured answers that the synthesis can cross-reference.**

### Bad query (one-line, narrow)
> "Is Next.js faster than Remix?"

Each provider returns "yes, sometimes, it depends" with no common
axes. Synthesis is weak.

### Good query (structured, decomposable)
> For each of Next.js 15 and Remix 2: (1) time-to-first-byte under
> cold-start SSR for a minimal product page, (2) time-to-interactive
> after hydration, (3) bundle size overhead per route vs. competitor,
> (4) developer-experience tradeoffs cited by teams who switched in
> either direction in 2025. Cite specific benchmarks (Vercel, Cloudflare,
> independent blog posts with numbers). Conclude with a recommendation
> framework: "use A when X, use B when Y".

Each provider has identifiable sub-tasks, the synthesis has common
axes to compare across, and the cross-provider verification catches
disagreement.

### The structure to aim for

Good research queries have four parts:

1. **Scope** — what exactly are we investigating, with constraints
   (time period, geography, versions, comparison criteria)
2. **Decomposition** — sub-questions the research should answer, each
   of which is answerable with evidence
3. **Evidence preference** — what kinds of sources count (primary
   docs / peer-reviewed / benchmarks / industry reports / etc.)
4. **Deliverable shape** — what the output should look like (comparison
   table / week-by-week playbook / ranked recommendations / etc.)

Aim for 100-200 word queries. Shorter than that usually lacks
structure; much longer starts to over-constrain.

### If the user is vague, ask ONE clarifying question — don't guess

Examples of good clarifying questions:
- "Are you comparing these for a consumer app or enterprise? The
  answer may differ."
- "How recent should the sources be — are 2024 findings fine, or does
  this need to be 2026?"
- "Is this for a decision you're making this week, or background
  knowledge? That changes whether Nano/Lite is enough."

Don't ask more than one. If still vague after one clarification,
propose a reasonable scoping and confirm.

## Cost awareness rules

- **NEVER auto-submit research** without establishing budget and
  confirming prxhub doesn't already have the answer.
- **Track cumulative session spend.** After multiple queries, report
  running total.
- **Suggest cheaper tiers proactively** when the question is simple.
  Nano/Lite are often enough for narrow factual questions.
- **Confirm expensive operations.** For Deep/Max (L/XL), get explicit
  confirmation even if budget was set higher in-session.
- **Report cost after EVERY research job.**
- **Alert at low balance.** If < $5, mention it. If < $2, warn.
- **Insufficient funds → billing link:** https://parallect.ai/settings/billing

## Mode selection — set automatically by tier, rarely override

The (tier → mode) mapping is fixed above. You generally should not
override mode separately from tier. But if the user has already picked
a tier and explicitly asks for the other mode, the semantics are:

| Mode | What it means |
|------|---------------|
| `fast` | Parallel fan-out across providers, synthesis skipped. Returns a single provider's raw report. Used by Nano/Lite/Normal by default. |
| `methodical` | Sequential provider runs with cross-provider gap analysis between, then full synthesis with claim extraction and conflict resolution. Used by Deep/Max by default. |

At matching tiers, fast and methodical cost almost the same — the
difference is depth, not price. Don't override mode to save money;
override only if the user explicitly wants more/less cross-checking.

## Session memory

- Remember `threadId` and `jobId` from current research context.
- If user says "the research" or "those results", use the most recent
  completed `jobId`.
- Track cumulative spend across the session as a running total.
- If the user asks a follow-on after a long gap, re-run `search_bundles`
  on prxhub first — the registry may have grown since.

## What NOT to do

- Don't summarize the synthesis and throw away citations. The whole
  point of Parallect is citable, cross-referenced findings. Preserve
  them.
- Don't run research on topics you can answer from memory. If the
  question is "what's 2+2" or "explain how TCP works", don't call
  anything. Answer directly.
- Don't re-submit the same query to Parallect if it failed. Call
  `research-status` first and read the error. `JOB_NOT_COMPLETE` is
  not a failure — keep polling. Actual failures usually mean a
  provider outage; backoff + retry once, then escalate to user.
- Don't pass UI tier names (`Normal`, `Deep`, `Max`) to the MCP. The
  `budgetTier` enum accepts only `XXS`/`XS`/`S`/`M`/`L`/`XL`.
- Don't skip the prxhub search step, even if you're confident the
  topic is niche. The registry is growing and you will be wrong.

---

Built with care by the Parallect team. Report issues:
https://github.com/parallect/claude-code-plugin/issues
