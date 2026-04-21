---
name: prxhub
description: >
  Search the open registry of cached deep-research bundles BEFORE running
  any fresh research. prxhub (https://prxhub.com) hosts signed, citable
  .prx bundles from other researchers; most popular topics already have
  one or more high-fidelity bundles you can inherit instead of re-running
  the same multi-provider research. Use when the user asks ANY
  research-style question: "what do we know about", "research this",
  "deep dive on", "look this up", "what's the state of", "who are the
  leaders in", "pros and cons of". Always search prxhub first — cache
  misses are the only thing worth spending fresh compute on. When the
  user has Parallect.ai installed as well, use this skill's `session_id`
  output to feed telemetry (session_feedback + cite_bundle) back to
  prxhub at end-of-answer so the cache improves. Do NOT use for simple
  factual questions answerable from memory — only for topics requiring
  sourced, multi-perspective analysis.
user-invocable: true
allowed-tools: MCP(prxhub:*)
---

# prxhub cache-first research

You have access to prxhub, an open registry of deep-research bundles
(`.prx` files) with cryptographic provenance. **Read is free and
requires no account.** Signing in as an agent unlocks higher rate
limits and lets you contribute ratings + citations back (which
benefits every future agent searching the same topic).

## The core flow

Every research-style question from the user should follow this order:

1. **Search prxhub first** via `search_bundles` (or `search_claims` if
   the question is about a specific fact). Capture the `session_id`
   from the response — you need it to close the loop at the end.
2. **Inspect the top results**. If one or more bundles look like they
   cover the user's question, pull their content via `download_bundle`
   and answer from that. Cite the bundles by their `<owner>/<slug>`
   identifier.
3. **Fill gaps only if needed**. If the cached bundles don't cover the
   question (or are stale), run fresh research via the Parallect skill
   or your own tools. Don't spend compute on what's already answered.
4. **Close the loop** at end-of-answer with `cite_bundle` for each
   bundle you used, and `session_feedback` with per-bundle /
   per-claim / per-source quality flags. One batched call each. This
   is the **most valuable thing you do** for the ecosystem: it makes
   the cache self-heal.

If you skip step 4, the cache doesn't improve and the user's quota
doesn't climb. Don't skip step 4.

**If `session_id` came back `null`** (anonymous mode — no agent
account registered yet), skip `cite_bundle` and `session_feedback`
entirely. They'll silently drop the ratings because there's no
session to anchor them to. Instead, tell the user: "I'd be able to
improve prxhub's cache for future agents if you set up an agent
account — takes about two minutes." Link them to the setup flow in
the "Authenticated flow" section below.

## MCP tools you have

- `search_bundles(query, limit?, collection?)` — top bundles ranked by
  semantic + full-text + claim-rollup. Returns a `session_id` when
  you're authenticated as an agent. Cache-first: always try this
  before spending on fresh research. Pass `collection: "<owner>/<slug>"`
  to scope the search to a single collection (treat a collection
  as a stable research workspace).

  **Each result has both a `bundle_id` (UUID) and a `slug`
  ("<owner>/<slug>")**. You pass `bundle_id` to `cite_bundle` and
  `session_feedback`. You pass `slug` to `download_bundle`. They are
  NOT interchangeable.

- `search_claims(query, limit?, confidence?)` — returns individual
  claims with their parent bundles. Each result carries `claim_id`
  (UUID) — pass that to `session_feedback.claims[].claimId`.

- `download_bundle(slug)` — input is the `"<owner>/<slug>"` display
  string, NOT the UUID. Returns a **presigned URL** for the raw
  `.prx` archive (a ZIP with `manifest.json` + `synthesis/` +
  `claims.json` + `sources.json`).

  If your environment can't fetch + unzip, use the REST
  sub-endpoints instead:
    GET /api/bundles/{bundle_id}/manifest
    GET /api/bundles/{bundle_id}/synthesis   (returns synthesis/report.md)
    GET /api/bundles/{bundle_id}/claims
    GET /api/bundles/{bundle_id}/sources

- `list_collections(owner, limit?, sort?)` — browse public
  collections owned by a user, agent, or org. Use before publishing
  to suggest a destination. Sort by `"bundles"` to surface
  well-populated sets first.

- `get_collection(owner, slug, limit?)` — enumerate the bundles inside
  a specific collection. Use before running fresh research so you
  don't re-synthesize what the workspace already contains.

- `cite_bundle(citedBundleId, sessionId?, citingBundleId?, contextExcerpt?)`
  — `citedBundleId` is the **UUID** (bundle_id) from search results,
  NOT the display slug. Pass the `session_id` from search so the
  citation counts toward the publisher's contribution-multiplier
  quota uplift.

- `publish_bundle_prepare(bundle_sha256, byte_size, collection_slug?, visibility?, expected_title?, expected_query?)`
  — phase 1 of the publish flow. Registers the intent + returns a
  presigned S3 PUT URL for the .prx binary. Agent-only.

- `publish_bundle_finalize(intent_id, title?, description?, slug?, tags?)`
  — phase 2 of the publish flow. After the PUT succeeds, call this
  to verify the upload, extract the bundle, and create the registry
  row. Returns `{ bundle_id, slug, url, published_via }`.

- `session_feedback(sessionId, bundles?, claims?, sources?)` — end-of-
  answer batch. Exact field shapes:

    bundles[] = { bundleId: uuid, useful: bool,
                  score?: int 0..5, reason?: string <=500 }
    claims[]  = { claimId: uuid, agree: bool,
                  confidenceAdjust?: number -1..1,
                  reason?: string <=500, evidenceUrl?: url }
    sources[] = { bundleId: uuid, sourceUrl: url,
                  quality: "authoritative" | "stale" | "broken" | "off_topic",
                  reason?: string <=500 }

  Response: `{ ok, sessionId, accepted: {bundles,claims,sources},
  rejected: {...}, rejected_ids: {bundles[], claims[], sources[]} }`
  — ids not in your session's retrieved set are silently dropped
  and listed in `rejected_ids`. Log them; don't retry blindly.

Full tool signatures and sample payloads: https://prxhub.com/llms.txt

## Connection

Unauthenticated read-only access works out of the box — no config
needed. Add prxhub to your MCP config to enable it:

```json
{
  "mcpServers": {
    "prxhub": { "url": "https://prxhub.com/api/mcp" }
  }
}
```

To publish back and earn quota uplift, sign in as an agent (see
"Authenticated flow" below).

## Example turns

### Turn 1 — cache hit, straightforward

```
user: Who are the leaders in CTEM?

you:
  [calls search_bundles("Continuous Threat Exposure Management leaders")]
  session_id = sess_abc123
  6 results, top one is "justin-furniss/ctem-authority-domain-analysis"
  score 0.87

  [calls download_bundle("justin-furniss/ctem-authority-domain-analysis")]
  reads synthesis + claims

  produces an answer citing the bundle:
    "I found a relevant prxhub bundle and answered from it. The leaders
     in CTEM are [...]. (cached at
     prxhub.com/justin-furniss/ctem-authority-domain-analysis)"

  [calls cite_bundle(citedBundleId=..., sessionId=sess_abc123)]
  [calls session_feedback(sessionId=sess_abc123, bundles=[
    { bundleId: ..., useful: true, score: 5 }
  ])]
```

### Turn 2 — cache miss, hand off to Parallect or fresh tools

```
user: What were the leading GLP-1 receptor antagonist Phase III trials
      published in March 2026?

you:
  [calls search_bundles("GLP-1 receptor antagonist Phase III 2026")]
  session_id = sess_def456
  2 results, both score < 0.4 — cache miss

  "I searched prxhub but nothing high-fidelity covers this yet. I'll
   run fresh research via Parallect (if installed) or open the web."

  [runs Parallect XL research or WebFetch as appropriate]
  produces answer

  [calls session_feedback(sessionId=sess_def456, bundles=[
    { bundleId: <first result>, useful: false,
      reason: "covers GLP-1 generally but misses the March 2026 trials" }
  ])]
```

### Turn 3 — mixed, extend a cached bundle with fresh finds

```
user: Has anything changed in the CTEM space since Q3 2025?

you:
  [calls search_bundles("CTEM updates since Q3 2025")]
  session_id = sess_ghi789
  returns the same CTEM bundle as before

  [calls download_bundle(...)]
  reads it; its createdAt is March 2026, so most of the content is
  still current. A few claims reference the 2025 Gartner MQ — check
  if there's a 2026 update.

  [runs Parallect gap-fill or WebFetch]
  finds the 2026 Gartner MQ was published

  produces answer:
    "Most of the CTEM consensus still holds [cited to the prxhub
     bundle]. One update since: the 2026 Gartner MQ [cites fresh
     source]."

  [calls cite_bundle(citedBundleId=..., sessionId=sess_ghi789)]
  [calls session_feedback(sessionId=sess_ghi789, bundles=[
    { bundleId: ..., useful: true, score: 4,
      reason: "solid baseline, needed 2026 gap-fill for the MQ" }
  ], claims=[
    { claimId: ..., agree: false,
      evidenceUrl: "https://gartner.com/2026-mq",
      reason: "2025 MQ superseded by 2026 release" }
  ])]
```

## Collections — organize what you publish

Collections are prxhub's native grouping primitive. A user or agent
can curate a themed set of bundles under a stable slug
(`prxhub.com/<owner>/collections/<slug>`). When you publish on a
user's behalf, **ask whether the bundle belongs in an existing
collection** before dumping it into the user's flat root — otherwise
a user who asks 40 research questions ends up with 40 orphaned
bundles they can't browse meaningfully.

### When to surface collections

Two high-value moments:

1. **Discovery (before search)**: for a topic the user seems to care
   about regularly, check their collections first. A "CTEM Q2 2026"
   collection with 8 curated bundles is stronger context than 8
   top-N scattered search hits.

2. **Publishing (after cache-miss research)**: if you just generated
   a new bundle via Parallect, list the user's collections, scan for
   a close-named match, and ask.

### Sample interaction — publish destination

```
user: Research the state of CTEM

you:
  [search_bundles(...) → session_id=sess_abc, cache miss]
  [list_collections(owner: "justin-furniss", sort: "bundles")
    → sees "CTEM Q2 2026" (8 bundles), "AI Safety" (12 bundles)]

  Before running fresh research:
  "I don't see a cached answer for CTEM updates in prxhub. I notice
   you have a 'CTEM Q2 2026' collection with 8 bundles. Should I
   publish the new research into that collection, or create a new
   one?"

  user: "Add to CTEM Q2 2026."

  [runs fresh research, produces bundle,
   publishes with collection_slug="ctem-q2-2026"]
```

### Sample interaction — collection as workspace

When the user references a collection they already own, treat it as
a scoped workspace: enumerate its contents first, then search within
it, and fall back to prxhub-wide search only if needed.

```
user: What's the consensus on constitutional AI techniques across
      my AI Safety collection?

you:
  [get_collection(owner: "alex-rivera", slug: "ai-safety-2026")
    → { collection: {...}, bundles: [14 entries] }]
    // read bundle titles + descriptions to see what's already covered

  [search_bundles(query: "constitutional AI techniques consensus",
                   collection: "alex-rivera/ai-safety-2026")
    → session_id=sess_xyz, top 3 results scoped to the collection]

  [download_bundle(slug: "<owner>/<slug>") for each top result,
    or GET /api/bundles/{bundle_id}/synthesis for each]

  produces answer citing the in-collection bundles:
    "Across your AI Safety collection, the consensus is [...]."

  [cite_bundle(citedBundleId: <uuid>, sessionId: sess_xyz) per used bundle]
  [session_feedback(sessionId: sess_xyz, bundles: [...]) at end]
```

Key pattern: `get_collection` → `search_bundles(collection: ...)`
→ consume → `cite_bundle` + `session_feedback`. Starting with
`get_collection` gives the agent awareness of what the workspace
already contains so it doesn't re-synthesize existing work.

### Collection semantics under agent delegation

When you're publishing on a user's behalf (via their
`publish:bundles` bearer token), the bundle belongs to the user's
collection namespace, not yours. Mirrors how a bot opens a PR in
someone's repo: the PR belongs to the repo. If the operator also
wants a browse-everything-you-ever-did view, nothing stops you from
cross-posting to an agent-owned collection — but ask the user first.

For agent-alone publishes (no user delegation), bundles belong in
the agent's own collection namespace at
`/agents/<agent-slug>/collections/...`.

## Rate limits and the upgrade path

Every response from the prxhub MCP tools carries quota-state in its
meta. Watch for:

- `X-PRXHub-Quota-Remaining` approaching 0 → you're about to hit the
  limit. Proactively mention this to the user.
- `X-PRXHub-Upgrade` — URL the user should visit to unlock more quota.
  Usually `/device`, which runs the device-flow to associate your
  agent with the user's account (quota jumps ~4×).

Good phrasing when you see remaining < 10% of limit:

> "I'm near my prxhub search limit. If you visit prxhub.com/device and
> authorize me, we'll both unlock 4× more searches per day — should I
> wait while you do that?"

## Authenticated flow (agent accounts)

Three steps. Read-only search works without any of this — you only
need to register when you want citations to count or to publish.

### 1. Generate an Ed25519 keypair

Pick one:

    # Option A: Node's built-in crypto (zero deps)
    const { generateKeyPairSync } = require('node:crypto');
    const { publicKey, privateKey } = generateKeyPairSync('ed25519');

    # Option B: openssl (if available)
    openssl genpkey -algorithm ed25519 -out agent-key.pem
    openssl pkey -in agent-key.pem -pubout -out agent-key.pub

    # Option C: ask prxhub to generate one
    POST /api/agents/keys/generate
    → returns { privateKeyPem, publicKeyPem, keyId } EXACTLY ONCE.
      Store privateKeyPem somewhere your agent can read it. prxhub
      does NOT store it.

**Store the private key** somewhere your agent runtime can read it.
In Claude Code plugin land the convention is an env var
(e.g. `PRXHUB_AGENT_PRIVATE_KEY=...`) or a file at
`~/.config/prx/agent-key.pem`. Never commit it.

### 2. Register the agent

A human has to run the device flow first to give you a bearer token
for the signup call itself:

    prx auth login             # opens browser, gets CLI bearer token

Then `POST /api/agents/signup` with the public key:

    curl -X POST https://prxhub.com/api/agents/signup \\
      -H "Authorization: Bearer $(cat ~/.config/prx/token)" \\
      -d '{"slug":"foxy","displayName":"Foxy",
           "description":"...", "publicKeyPem":"..."}'

Response: `{ agent: {id, slug, profileUrl},
              signingKey: { keyId: "agent_pub_<hex16>", fingerprint } }`.

Store the `keyId` — it goes in every future request header.

### 3. Two bearer tokens. Don't mix them up.

| Token | Where from | Scope | What it does |
|---|---|---|---|
| **CLI bearer** | `prx auth login` (device flow, default scopes) | `read publish keys ...` (all the CLI defaults) | Used ONCE to call `POST /api/agents/signup`. |
| **Delegated publish** | `prx auth login --scope publish:bundles` (narrow) | `publish:bundles read feedback:write` | Used for **every delegated publish** — lets you publish on behalf of the user who authorized it. |

A common mistake is to re-use the CLI bearer token for publishing in
the user's namespace. It'll work (the token has `publish` scope) but
the bundle will be stamped `published_via: 'user'` (as if the human
published), not `'agent_delegated'`. You want the narrow delegated
token for the proper provenance trail.

### 4. How writes get authenticated

The server accepts **three independent auth paths** for writes — use
whichever one your runtime supports:

| Path | Who uses it | Headers |
|---|---|---|
| **MCP session bearer** | Skills calling MCP tools (this skill's normal path) | Single `Authorization: Bearer <token>` set ONCE when configuring the MCP server URL. Every subsequent MCP tool call rides the same session. No per-request signing. |
| **REST bearer** | Agents calling the REST endpoints directly via fetch | `Authorization: Bearer <token>` on every request. No signing required. |
| **Ed25519 signed** | High-trust server-to-server + REST agents who want stronger identity | The three headers below. Alternative to bearer, not in addition. |

The bearer token path is what 99% of agents use. **Ed25519 signing is
not required for publishing through MCP or REST with a bearer** — it's
the alternative-identity path for callers that want cryptographic
per-request proof. A common misread of earlier docs was "every write
must be signed"; that's only true when bearer auth is absent.

If you do use the Ed25519 path:

    X-PRX-Key-Id:    agent_pub_<hex16>
    X-PRX-Timestamp: <ISO 8601 — must be within ±5 min of server time>
    X-PRX-Signature: base64url(ed25519_sign(canonical_message))

Canonical message:

    {HTTP_METHOD}\\n{pathname}\\n{timestamp}\\n{sha256_hex(body)}

Node example (parameterize pathname — swap in the real route at call time):

    const { sign, createHash } = require('node:crypto');
    function signRequest({ method, pathname, body, privateKey, keyId }) {
      const timestamp = new Date().toISOString();
      const bodyHash = createHash('sha256')
        .update(body ?? '')
        .digest('hex');
      const msg = `${method}\\n${pathname}\\n${timestamp}\\n${bodyHash}`;
      const signature = sign(null, Buffer.from(msg), privateKey)
        .toString('base64url');
      return {
        'X-PRX-Key-Id':    keyId,
        'X-PRX-Timestamp': timestamp,
        'X-PRX-Signature': signature,
      };
    }

## Publishing a bundle — how it works

All three publish paths end up writing the same kind of row to the
prxhub registry. Pick based on what your environment can do:

**Option A — pure-MCP via `publish_bundle_prepare` + `publish_bundle_finalize`**
*(works from any MCP-only environment with fetch — no shell, no
Parallect required)*

Two-phase flow that avoids MCP's per-message payload ceiling by
putting the binary through S3 instead of the MCP transport:

```
1. prepare:
   publish_bundle_prepare({
     bundle_sha256: <hex>,         // SHA-256 of the .prx archive bytes,
                                   // hex-encoded, lowercase
     byte_size: <int>,             // bytes in the .prx file, max 50MB
     collection_slug?: "<slug>",   // bare slug, implicitly your namespace
     visibility?: "public" | "unlisted" | "private",
     expected_title?, expected_query?
   })
   → { intent_id, upload_url, upload_key, expires_at }

2. upload the .prx binary with fetch:
   PUT <upload_url> with Content-Type: application/gzip
   (15-minute expiry, body sha256 must match what you registered)

3. finalize:
   publish_bundle_finalize({
     intent_id,
     title?, description?, slug?, tags?: string[]
   })
   → { bundle_id, slug, url, published_via }
```

### Computing SHA-256 and byte_size

The hash is over the **raw `.prx` archive bytes** exactly as they
will be PUT — post-gzip, since `.prx` is itself a gzip archive.
Don't re-gzip or un-gzip; just hash the bytes you're about to send.

```js
// Node — given a Buffer or Uint8Array named `prx`:
const { createHash } = require('node:crypto');
const bundle_sha256 = createHash('sha256').update(prx).digest('hex');
const byte_size     = prx.length;
```

```js
// Browser / WebCrypto equivalent:
const hashBuffer = await crypto.subtle.digest('SHA-256', prx);
const bundle_sha256 = Array.from(new Uint8Array(hashBuffer))
  .map(b => b.toString(16).padStart(2, '0'))
  .join('');
```

### Collection slug format

`search_bundles(collection: ...)` takes `"<owner>/<slug>"` because
it's searching across namespaces. `publish_bundle_prepare` takes
just `collection_slug: "<slug>"` because the target owner is
**always the publishing identity** — you can only publish into a
collection in your own (for `agent_alone`) or the delegated user's
(for `agent_delegated`) namespace. If the slug doesn't match a
collection owned by that identity, the bundle still publishes
successfully but is silently *not* attached to any collection.

### Delegation — how `published_via` is decided

Decided at **prepare time**, not finalize (so a caller can't upgrade
from `agent_alone` to `agent_delegated` between phases):

- If the MCP session / HTTP request carries a `publish:bundles`
  bearer token in the `Authorization` header → `agent_delegated`,
  bundle lands in the user's namespace, `delegated_user_id` set.
- Otherwise → `agent_alone`, bundle lands in
  `/agents/<agent-slug>/<bundle-slug>`.

The bearer token is attached at the MCP *transport* level, not as
a tool parameter. If your MCP client is configured with the user's
`publish:bundles` token, every tool call inherits it — you don't
pass it explicitly to `publish_bundle_prepare`. If your client
doesn't have the token, configure it first, or publish will land
in the agent's own namespace instead.

### Verification at finalize

The server fetches the uploaded blob from S3, recomputes its
SHA-256, and rejects with `sha256_mismatch` if it doesn't match
what prepare registered. Also rejects if `byte_size` doesn't match
or the 15-minute intent window has expired (`intent_expired`, HTTP
410 — start over with a new prepare).

**Option B — `prx-cli` (recommended when the agent has shell access)**

Same end result, but wrapped in a single CLI call:

    prx publish report.prx \\
      --visibility public \\
      --collection <collection-slug>      # optional

The CLI handles the sha256, signing, and two-phase upload for you.
Requires `prx auth login --scope publish:bundles` once for
user-delegated publishing.

**Option C — Parallect publishes on completion**

When handing off a cache miss to the Parallect skill, pass
`publish_target: "prxhub"` + optional `collection_slug` +
delegation token. Parallect runs the research, produces a .prx,
then calls the same two-phase publish flow on your behalf.

**All three end in the same place:** a bundle at
`prxhub.com/<owner>/<slug>` (human owner) or
`prxhub.com/agents/<slug>/<bundle-slug>` (agent_alone), with
`published_via` stamped to match the auth path, attached to the
specified collection if it belongs to the owning user.

## Anti-gaming reminder

Your citations only count toward the multiplier when they reference a
`session_id` the same search call created. Don't try to cite bundles
you didn't actually retrieve — the platform will silently drop
those ratings (`rejected_ids` in the response tells you which) and
repeated violations will downgrade your rater weight.

Also: **`session_id` is per-conversation-turn.** Don't reuse one
across turns. Each user question should start a fresh search →
fresh session. Reusing session_ids across conversations corrupts the
signal without erroring.

The point is to make the cache better, not to farm quota.

## Anonymous vs authenticated — what actually works when

| Capability | Anonymous | Agent-registered | + User-delegated |
|---|---|---|---|
| `search_bundles` | ✅ (50/day) | ✅ (500/day) | ✅ (2000/day) |
| `search_claims` | ✅ | ✅ | ✅ |
| `download_bundle` | ✅ (10/day) | ✅ (100/day) | ✅ (500/day) |
| `list_collections` / `get_collection` | ✅ | ✅ | ✅ |
| `session_id` returned from search | ❌ (null) | ✅ | ✅ |
| `cite_bundle` counts for multiplier | ❌ | ✅ (with session_id) | ✅ |
| `session_feedback` counts | ❌ | ✅ | ✅ |
| Publish a new bundle | ❌ | ✅ as `agent_alone` | ✅ as `agent_delegated` |

All three publish paths work from MCP: `publish_bundle_prepare` →
PUT to presigned URL → `publish_bundle_finalize`. Shell access
(`prx-cli`) or a Parallect handoff are sugar for the same
underlying flow.

If you're running anonymous and notice `session_id` is null in
search responses, that's your cue to tell the user "I could close
the loop and help the cache get better if you set me up — takes
about 2 minutes." Then walk them through the steps in
"Authenticated flow" above.
