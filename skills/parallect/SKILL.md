---
name: parallect
description: >
  Deep research using Parallect.ai. Queries multiple AI research providers
  (Perplexity, Gemini, OpenAI, Grok, Anthropic) in parallel, then synthesizes
  results into a unified report with cross-referenced citations and conflict
  resolution. Use when the user wants to research a topic, investigate a
  question, or needs comprehensive analysis with citations. Also triggers
  when the user says things like "look this up", "research this",
  "find out about", "deep dive on", or "what do we know about".
  Do NOT use for simple factual questions you can answer from memory --
  only for topics requiring sourced, multi-perspective analysis.
user-invocable: true
allowed-tools: MCP(parallect:*)
---

# Parallect Deep Research

You have access to Parallect.ai, a multi-provider deep research platform.
It queries multiple frontier AI providers simultaneously and synthesizes
their findings into a single report with cross-referenced citations,
extracted claims, conflict resolution, and follow-on suggestions.

Research is **asynchronous** -- jobs take 4 to 45+ minutes depending
on tier and providers. You MUST poll for completion. Never block.

## Connection

Parallect is accessed via MCP. Authentication is handled automatically
through OAuth2 -- the user authorizes via their browser on first use.
No API key configuration needed.

## Gotchas

Read these first. They prevent the most common mistakes:

- **Research is async.** Calling `research` does NOT return results. It
  returns a `jobId`. You must poll `research-status` and then call
  `get-results` only after status is `"completed"`.
- **`get-results` on a running job is an error.** Don't call it until
  `research-status` returns `"completed"`. You will get `JOB_NOT_COMPLETE`.
- **You are spending the user's money.** Never submit research without
  discussing budget first. Even an XXS tier costs ~$1.
- **Polling too fast triggers rate limits.** Use exponential backoff with
  jitter (see Step 3). Don't poll every 5 seconds.
- **Balance check before research is mandatory.** If the user has no
  balance and no payment method, the research call will fail. Check first.
- **The `synthesis` field is markdown.** Present it as-is with citations.
  Don't strip the formatting.
- **`research` always creates a new thread.** Do NOT pass a `threadId`
  to the `research` tool. For follow-up research in the same thread, use
  the `follow-up` tool with the parent `jobId`.
- **`fast` mode skips synthesis.** It returns a single provider's raw
  report with no cross-referencing or conflict resolution. Only use when
  the user explicitly prioritizes speed.

## Available Tools

| Tool | Use when... | Do NOT use when... |
|------|-------------|-------------------|
| `research` | User wants to investigate a topic with sourced analysis | Question is simple enough to answer from memory |
| `research-status` | You've submitted research and need to check progress | You haven't called `research` yet |
| `get-results` | `research-status` shows `"completed"` | Job is still running or synthesizing |
| `follow-up` | User wants to dig deeper on a completed research topic | No prior completed job exists |
| `list-threads` | User refers to past research or wants to resume | You already have the threadId |
| `get-thread` | You need full context of a previous research session | You only need current job status |
| `balance` | Before starting research, or after to report remaining credits | Mid-polling (wastes a call) |
| `usage` | User asks about their spending history | Default -- only when explicitly asked |
| `list-providers` | Before research to show user what their tier includes | Already discussed budget with user |
| `search-claims` | User wants to find specific claims across past research | No research has been done yet |
| `get-claim-evidence` | User wants evidence backing a specific claim | Claim ID is not known |

## Workflow: Running Research

### Step 1: Discuss budget and check balance

Before submitting ANY research:

1. Call `balance` to check the user's current credits and payment status.
2. Call `list-providers` with the anticipated `budgetTier` to show what
   providers and cost range that tier includes.
3. Discuss budget with the user:

   | User says... | Map to tier | Max cost | Providers | Duration |
   |-------------|-------------|----------|-----------|----------|
   | "quick check", "just a glance" | XXS | ~$1 | 1 provider | ~4-5 min |
   | "quick look", "brief" | XS | ~$2 | 1-2 providers | ~8-10 min |
   | "standard", "normal" | S | ~$5 | 2 providers | ~15-18 min |
   | "thorough", "detailed" | M | ~$15 | 3-4 providers | ~15-20 min |
   | "comprehensive", "deep" | L | ~$30 | 4-5 providers | ~20-25 min |
   | "exhaustive", "everything" | XL | ~$60 | All providers | ~30-45 min |

4. Tell the user: "Your balance is $X.XX. A [tier] research will cost
   up to $Y. That will query [providers]. Want to proceed?"

5. If balance is insufficient and no payment method is on file, direct
   them to https://parallect.ai/settings/billing before proceeding.

6. If the user set a budget preference earlier in this session, reuse it
   silently unless the topic warrants a different tier. Don't ask again.

### Step 2: Submit the research

Call `research` with:
- `query`: A specific, well-formed research question
- `budgetTier`: The tier confirmed in Step 1
- `mode`: `"methodical"` (default) for multi-provider synthesis.
  Only use `"fast"` if the user explicitly wants speed over depth.

Save the returned `jobId` and `threadId`.

Tell the user: "Research submitted to [N] providers. I'll check back
on progress in about 30 seconds."

### Step 3: Poll for completion (exponential backoff with jitter)

Research is asynchronous. Poll using `research-status` with exponential
backoff. Do NOT poll at fixed intervals.

**Polling schedule:**

| Check # | Approximate wait | Cumulative time |
|---------|-----------------|-----------------|
| 1 | ~30s | 30s |
| 2 | ~50s | 1m 20s |
| 3 | ~75s | 2m 35s |
| 4 | ~110s | 4m 25s |
| 5+ | ~120s (cap) | +2min each |

**Cap the maximum interval at 120 seconds.** Never poll more frequently
than every 30 seconds.

**On each poll, report progress naturally:**
- "1 of 3 providers finished, 2 still running..."
- "All providers done, synthesis is running now..."
- "Research complete! Let me grab the results."

**Timeout:** Research can take up to 45 minutes for large XL jobs.
If the job hasn't completed after 30 minutes, inform the user and
offer alternatives (keep polling, try fewer providers, or switch
to fast mode). For XXS/XS tiers, flag if it exceeds 15 minutes.

### Step 4: Deliver results

When `research-status` returns `"completed"`:

1. Call `get-results` with `jobId`.

2. Present findings in this order:
   a. **Brief summary** (3-5 sentences) of the key findings in your own
      words. Lead with the most important insight.
   b. **Full synthesis** -- share the `synthesis` markdown as-is. It
      contains inline citations and cross-references.
   c. **Cost report** -- "This research cost $X.XX across N providers
      (took Y minutes)."
   d. **Balance check** -- call `balance` and report remaining credits.

### Step 5: Offer follow-ons

Results include `followOnSuggestions`. Present them as numbered options:

"Based on this research, here are directions we could explore next:
1. [suggestion 1]
2. [suggestion 2]
3. [suggestion 3]

Want me to research any of these, or ask your own follow-up question?"

When the user picks one, use `follow-up` with the parent `jobId`.

## Cost Awareness Rules

You are spending the user's money at machine speed. Be responsible:

- **NEVER auto-submit research** without first establishing budget.
- **Track cumulative session spend.** After multiple queries, report total.
- **Suggest cheaper tiers proactively** for simple questions.
- **Confirm expensive operations.** For L/XL tiers, get explicit confirmation.
- **Report cost after EVERY research job.**
- **Alert at low balance.** If below $5, mention it. Below $2, warn.
- **Link to billing on insufficient funds:** https://parallect.ai/settings/billing

## Tips for Quality Research Queries

- **Be specific:** Include constraints, time period, geography, comparison criteria.
- **If the user is vague, ask ONE clarifying question** before spending money.
- **Don't over-scope.** Match query scope to budget tier.

## Mode Selection

| Mode | When to use | Duration | Output |
|------|------------|----------|--------|
| `methodical` | Accuracy and breadth matter (default) | 5-45 min | Multi-provider synthesis with claims and citations |
| `fast` | User says "quick" / "fast" / time-sensitive | 2-8 min | Single provider raw report, no synthesis |

## Session Memory

- Remember `threadId` and `jobId` from current research context.
- If the user says "the research" or "those results", use the most
  recent jobId.
- Track cumulative spend across the session as a running total.
