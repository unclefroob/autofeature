---
status: CUSTOM
description: CEO / PM / flow-walker product review. Maps the product surface, fans out three product lenses in parallel via the Workflow tool, adversarially verifies broken-flow and gap claims against the real code, and synthesizes a prioritized gaps-&-flows report. Runs pre-build inside /autofeature and standalone via /autofeature:product-review.
---

# Feature Product Review (CEO / PM / Flow-Walker)

Looks at a product the way a **founder/CEO**, a **product manager**, and a careful **first-time
user** would — and finds **gaps** and **broken or incomplete flows** *before* engineering time is
spent. It answers: *is this the right thing to build, what's missing, and does the user journey
actually complete end-to-end?*

This is **not** an engineering code review. For correctness/security/tests, see
`feature-review.md` + `feature-review-checklist.md`. This pass is about **product fit and flow
completeness**, performed by the personas defined in `agents/product-strategist.md`.

The mechanism is the **Workflow tool** — deterministic multi-agent orchestration:

```
Map the product surface
   → fan out CEO + PM + flow-walker lenses in parallel
      → dedupe across lenses
         → adversarially verify high-severity gap/flow claims against the real code
            → synthesize a prioritized report + recommended next features
```

Findings severity (full rubric in `agents/product-strategist.md`):
- 🔴 **critical** — a core journey is broken/dead-ends, or the feature ships a half-capability
- 🟡 **high** — a clearly-expected capability is missing, or a secondary flow is broken
- 🟢 **medium** — a real rough edge (missing empty/error state, no confirmation, no off-switch)
- ⚪ **low** — polish or a strategic opportunity worth noting, not now

---

## Two modes

| Mode | When | What the lenses judge |
|------|------|-----------------------|
| `feature` | **Pre-build**, inside `/autofeature` (Step 4.5) — a feature is proposed but not yet coded | Does this feature, in the context of the existing product, make sense? What gaps does it leave? Does it complete the journey it touches? |
| `product` | **Standalone** `/autofeature:product-review` with no `feature:` arg | Audit the whole product as it exists today — find every gap and broken flow. |

Both modes use the **same workflow**; only the `mode` arg and the framing differ.

---

## Running the review

Invoke the **Workflow** tool with the script below. Pass `args` as a JSON object:

```
Workflow({
  script: <the script below, verbatim>,
  args: {
    repo:         "<pwd>",
    mode:         "feature" | "product",
    featureBrief: "<path to .autofeature/designs/[slug]-[date].md>"  // '' for whole-product audit
    siteUrl:      "<https://… or ''>"                                 // optional live deployment
  }
})
```

The workflow runs in the background and returns a structured synthesis object. **Do not** re-do the
review yourself — relay and act on what it returns.

> Opt-in note: this command/step explicitly instructs you to call the Workflow tool — that is the
> required opt-in. The workflow spawns ~5–10 agents (1 map + 3 lenses + up to 8 verifications + 1
> synthesis), capped and logged.

### The Workflow script

```js
export const meta = {
  name: 'product-review',
  description: 'CEO / PM / flow-walker product review — map the product surface, find gaps and broken flows, verify against code, synthesize a prioritized report',
  phases: [
    { title: 'Map',        detail: 'map product surface, user journeys, entities' },
    { title: 'Review',     detail: 'CEO + PM + flow-walker lenses in parallel' },
    { title: 'Verify',     detail: 'adversarially verify high-severity gap/flow claims against the code' },
    { title: 'Synthesize', detail: 'dedupe, prioritize, recommend next features' },
  ],
}

const REPO     = args.repo
const MODE     = args.mode || 'product'        // 'feature' (pre-build) | 'product' (whole-product audit)
const BRIEF    = args.featureBrief || ''        // path to Feature Brief; '' for whole-product audit
const SITE_URL = args.siteUrl || ''             // optional live URL
const VERIFY_CAP = 8                            // max gap/flow claims to adversarially verify

const keyOf = f => (f.title || '').toLowerCase().replace(/[^a-z0-9]+/g, ' ').trim()
              + '|' + (f.where || '').toLowerCase().replace(/[^a-z0-9]+/g, ' ').trim()
const SEV = { critical: 0, high: 1, medium: 2, low: 3 }

// ---------- schemas ----------
const PRODUCT_MAP = {
  type: 'object', required: ['surfaces', 'journeys', 'entities'],
  properties: {
    summary: { type: 'string' },
    surfaces: { type: 'array', items: { type: 'object', required: ['kind', 'name', 'role'],
      properties: { kind: { type: 'string' }, name: { type: 'string' }, role: { type: 'string' }, location: { type: 'string' } } } },
    journeys: { type: 'array', items: { type: 'object', required: ['name', 'steps'],
      properties: { name: { type: 'string' }, steps: { type: 'array', items: { type: 'string' } } } } },
    entities: { type: 'array', items: { type: 'string' } },
    proposedFeature: { type: 'string' },
  },
}
const FINDINGS = {
  type: 'object', required: ['findings'],
  properties: { findings: { type: 'array', items: {
    type: 'object', required: ['lens', 'type', 'title', 'severity', 'where', 'evidence', 'recommendation'],
    properties: {
      lens: { type: 'string', enum: ['ceo', 'pm', 'flow'] },
      type: { type: 'string', enum: ['gap', 'broken-flow', 'risk', 'opportunity'] },
      title: { type: 'string' }, severity: { type: 'string', enum: ['critical', 'high', 'medium', 'low'] },
      where: { type: 'string' }, evidence: { type: 'string' }, recommendation: { type: 'string' },
      confidence: { type: 'string', enum: ['high', 'medium', 'low'] },
    } } } },
}
const VERDICT = {
  type: 'object', required: ['refuted', 'reason'],
  properties: { refuted: { type: 'boolean' }, reason: { type: 'string' }, evidence: { type: 'string' } },
}
const SYNTHESIS = {
  type: 'object', required: ['summary', 'findings', 'recommendedNextFeatures'],
  properties: {
    summary: { type: 'string' },
    counts: { type: 'object', properties: { critical: { type: 'number' }, high: { type: 'number' }, medium: { type: 'number' }, low: { type: 'number' } } },
    findings: { type: 'array', items: { type: 'object', required: ['title', 'type', 'severity', 'where', 'recommendation'],
      properties: { title: { type: 'string' }, type: { type: 'string' }, severity: { type: 'string' }, where: { type: 'string' },
        lens: { type: 'string' }, recommendation: { type: 'string' }, verified: { type: 'string' } } } },
    recommendedNextFeatures: { type: 'array', items: { type: 'object', required: ['title', 'why'],
      properties: { title: { type: 'string' }, why: { type: 'string' }, severity: { type: 'string' } } } },
  },
}

// ---------- Phase 1: Map ----------
phase('Map')
const featureLine = MODE === 'feature'
  ? `A feature is PROPOSED but NOT yet built. Read the Feature Brief at ${BRIEF}; capture what it proposes as "proposedFeature", and map the EXISTING product around it so the lenses can judge fit and gaps.`
  : `Map the WHOLE product as it exists today.`
const siteLine = SITE_URL
  ? `A live deployment exists at ${SITE_URL} — you may note what a user/crawler sees there, but the codebase is the source of truth.`
  : ''
const map = await agent(
  `You are mapping a software product's surface for a product review. Repo: ${REPO}.
${featureLine}
${siteLine}

Produce a Product Map:
- surfaces: every user-facing entry point — routes/endpoints, screens/pages, major features. For each: kind (route|screen|feature|endpoint), name, one-line role, location (file/path).
- journeys: the 3–7 PRIMARY user journeys (e.g. "sign up → onboard → first value", "create X → see it → edit → delete"), each an ordered list of concrete steps.
- entities: the core data entities (collections/models/tables).
${MODE === 'feature' ? '- proposedFeature: one paragraph on what the brief proposes.' : ''}

Ground everything in real files — do not invent surfaces that don't exist in the repo. Be concise.`,
  { schema: PRODUCT_MAP, phase: 'Map', model: 'haiku' }   // tiers: orchestrator/model-tiers.md
)
log(`Mapped ${map.surfaces?.length || 0} surfaces, ${map.journeys?.length || 0} journeys`)

// ---------- Phase 2: Review (3 lenses; barrier so we dedupe before verifying) ----------
phase('Review')
const mapJson = JSON.stringify(map)
const focus = MODE === 'feature'
  ? `Evaluate the PROPOSED feature ("${map.proposedFeature || 'see brief'}") IN THE CONTEXT of this product: does it make sense, what gaps does it leave, does it complete the journey it touches, and what adjacent gaps does it expose?`
  : `Audit the WHOLE product as it exists.`

const LENSES = [
  { key: 'ceo', brief: `Wear the FOUNDER/CEO lens. Judge value & differentiation (does it move activation/retention/revenue/core-loop?), coherence with the product story, opportunity cost (is a higher-leverage adjacent gap more valuable to close first?), moat/risk, and whether the slice is the complete valuable unit or a half-feature a user hits the edge of. type ∈ {gap,risk,opportunity}.` },
  { key: 'pm', brief: `Wear the PRODUCT MANAGER lens. Nail the job-to-be-done and WHO it serves. Your #1 job: find MISSING CAPABILITIES a reasonable user expects (create-but-no-edit/delete, notify-but-no-off-switch, invite-but-no-pending-state) and unhandled edge/empty/error/permission/offline states. Prioritize the gaps. type ∈ {gap,risk,opportunity}.` },
  { key: 'flow', brief: `Wear the FLOW-WALKER lens — a skeptical first-time user. For EACH primary journey in the map, trace it step-by-step THROUGH THE ACTUAL CODE (route→handler→screen→state→data). At every step ask: does the next step exist (or is it a TODO/dead link/unimplemented route/404)? Is the loop closed (confirmation, land somewhere sensible, result reflected)? Is the round-trip whole (create→list→open→edit→delete)? For API+web/mobile, does the journey survive the hop (endpoint exists, response shape matches the screen)? Name the SPECIFIC broken step and the file/route where the chain breaks. type is usually broken-flow. Set confidence:low for any flow you suspect but can't fully confirm in code.` },
]

const reviews = (await parallel(LENSES.map(l => () =>
  agent(
    `${l.brief}

${focus}

Product Map (JSON):
${mapJson}

Repo: ${REPO}. Open the real files for anything you call out — a finding not grounded in a file/route/screen is a guess; drop it or mark confidence:low.
Severity: critical = core journey broken / half-capability hit immediately; high = clearly-expected capability missing or secondary flow broken; medium = real rough edge (missing empty/error state, no confirmation/off-switch); low = polish/opportunity. Don't inflate. Return an EMPTY findings list if your lens found nothing real.`,
    { label: `review:${l.key}`, phase: 'Review', schema: FINDINGS, model: 'sonnet' }
  )
))).filter(Boolean)

const raw = reviews.flatMap(r => r.findings || [])
log(`${raw.length} raw findings across ${reviews.length} lenses`)

// dedupe across lenses (plain JS — genuinely needs all findings at once)
const merged = new Map()
for (const f of raw) {
  const k = keyOf(f)
  if (!merged.has(k)) merged.set(k, { ...f, lenses: [f.lens] })
  else {
    const e = merged.get(k)
    if (!e.lenses.includes(f.lens)) e.lenses.push(f.lens)
    if (SEV[f.severity] < SEV[e.severity]) e.severity = f.severity   // keep the worst severity
  }
}
const deduped = [...merged.values()]
log(`${deduped.length} findings after dedupe`)

// ---------- Phase 3: Verify high-severity gap/flow claims against the code ----------
phase('Verify')
const isHighFlowOrGap = f => (f.type === 'broken-flow' || f.type === 'gap') && (f.severity === 'critical' || f.severity === 'high')
const eligible = deduped.filter(isHighFlowOrGap).sort((a, b) => SEV[a.severity] - SEV[b.severity])
const eligibleKeys = new Set(eligible.map(keyOf))
let toVerify = eligible
let capped = 0
if (toVerify.length > VERIFY_CAP) { capped = toVerify.length - VERIFY_CAP; toVerify = toVerify.slice(0, VERIFY_CAP) }
if (capped) log(`Verifying top ${VERIFY_CAP} of ${eligible.length} high-severity claims (${capped} flagged not-auto-verified in the report)`)

const verifyKeys = new Set(toVerify.map(keyOf))
const verdicts = (await parallel(toVerify.map(f => () =>
  agent(
    `Adversarially VERIFY this product-review claim by trying to REFUTE it against the actual code. Default to refuted:true if the claim turns out wrong.

Claim (${f.type}, ${f.severity}): ${f.title}
Where: ${f.where}
Evidence given: ${f.evidence}

Repo: ${REPO}. Open the relevant files/routes. Refute (refuted:true) if the capability/route/screen/handler ACTUALLY EXISTS, the flow IS closed, or the round-trip IS complete and the reviewer simply missed it. Confirm (refuted:false) only if the gap/broken step is real and you can point to the absence. Be specific in evidence.`,
    { label: `verify:${(f.title || '').slice(0, 36)}`, phase: 'Verify', schema: VERDICT, model: 'sonnet' }
  ).then(v => ({ key: keyOf(f), verdict: v }))
))).filter(Boolean)
const verdictByKey = new Map(verdicts.map(v => [v.key, v.verdict]))

const survivors = []
let refuted = 0
for (const f of deduped) {
  const k = keyOf(f)
  if (verifyKeys.has(k)) {
    const v = verdictByKey.get(k)
    if (v && v.refuted) { refuted++; continue }              // false positive — drop it
    f.verified = v ? 'confirmed' : 'unverified'
    f.verifyNote = v ? v.reason : ''
  } else if (eligibleKeys.has(k)) {
    f.verified = 'not-auto-verified'                          // capped off
  } else {
    f.verified = 'n/a'                                        // medium/low or risk/opportunity
  }
  survivors.push(f)
}
log(`${survivors.length} findings after verification (${refuted} refuted as false positives)`)

// ---------- Phase 4: Synthesize ----------
phase('Synthesize')
const synthesis = await agent(
  `Synthesize a product review into a single prioritized report. ${MODE === 'feature'
    ? `The review was for a PROPOSED feature: ${map.proposedFeature || '(see brief)'} — distinguish gaps the feature itself must close (in-scope) from adjacent gaps worth doing later.`
    : 'This was a whole-product audit.'}

Findings (deduped + verified; "verified":"confirmed" survived an adversarial check; refuted ones already removed):
${JSON.stringify(survivors)}

Produce:
- summary: 2–4 sentences a founder reads first — the headline gaps/flow breaks and the single most important thing to fix.
- counts: findings by severity.
- findings: prioritized list (critical → low), where/recommendation concrete, near-duplicates merged. Carry over each finding's "verified" value.
- recommendedNextFeatures: 2–6 concrete, buildable features that close the most valuable gaps — each phrased as it could be handed to /autofeature, with a one-line "why".

Be honest about severity. If the product/feature is in good shape, say so plainly.`,
  { schema: SYNTHESIS, phase: 'Synthesize', model: 'sonnet' }
)

return {
  mode: MODE,
  mapped: { surfaces: map.surfaces?.length || 0, journeys: map.journeys?.length || 0 },
  refuted,
  capped,
  ...synthesis,
}
```

---

## Report output

After the workflow returns, render the report to the user (and, in `feature` mode, append it to
the Feature Brief under `## Product Review`):

```
=== Product Review: [project / feature name] ===
Mode:     [feature (pre-build) | product (whole-product audit)]
Mapped:   [N surfaces, M journeys]   Live URL: [SITE_URL or "not checked"]
Verified: [K high-severity claims checked against code, R refuted as false positives]

[summary — 2–4 sentences]

🔴 CRITICAL ([n])
  1. [title] — where: [locus] — fix: [recommendation]  ✓verified
🟡 HIGH ([n])
  1. [title] — where: [locus] — fix: [recommendation]  ⚠not-auto-verified
🟢 MEDIUM ([n])
  ...
⚪ LOW ([n])
  ...

Recommended next features:
  - [title] — [why]
  - ...
```

Mark each finding with its `verified` status (`✓verified`, `⚠not-auto-verified`, or nothing for
medium/low). In `product` mode, save the full report to `.autofeature/product-review-[YYYY-MM-DD].md`.

---

## What happens with the findings

### Pre-build (`feature` mode, inside /autofeature — Step 4.5)

- **AUTOMATED:** Fold 🔴/🟡 in-scope gaps into the work (the Plan subagent in Step 5 reads the
  `## Product Review` section). Surface **only a User Challenge** when a finding means the feature
  as requested ships a broken core flow or a half-capability — i.e. building it as-asked would
  contradict the user's actual goal. Otherwise proceed; never silently expand scope.
- **CHECKPOINT:** Show the report, then:
  > Product review found [C critical, H high] gaps/flow issues for this feature.
  >
  > A) Proceed as planned — log gaps as fast-follows
  > B) Expand this feature's scope to close the 🔴/🟡 in-scope gaps before building
  > C) Revise the feature request — [tell me what to change]
  > D) Abort

### Standalone (`product` mode) — Report + offer to fix

After the report, offer (mirrors `/autofeature:seo`):

```
Found [N] product issues ([C] critical, [H] high). 

A) Build the top fixes now      → spin the highest-value gaps into /autofeature runs
B) Fix a specific gap            → pick from the numbered list
C) Create Trello cards           → one card per significant gap
D) Report only — no changes
```

- **A / B:** For each chosen gap, hand its `recommendation` (and `recommendedNextFeatures` entry)
  to the autofeature pipeline as a feature request. **Append `[skip-product-review]`** to that
  request so the pre-build review (Step 4.5) doesn't recurse on a fix it just generated. Run
  sequentially (one feature at a time) unless the user asks to batch.
- **C:** Ask which list (`mcp__trello__get_lists`), then one `mcp__trello__add_card_to_list` per
  gap — title = the gap, description = where + recommendation + severity, footer links the saved
  report.
- **D:** Print the report path and exit.

---

## Guardrails

- **Codebase is the source of truth.** Every finding must be grounded in a file/route/screen. The
  Verify phase exists to kill plausible-but-wrong claims — trust its `refuted` verdicts.
- **Don't re-run on your own fixes.** A `[skip-product-review]` marker in the feature request means
  the pre-build step is skipped (prevents recursion when fixing a gap this review found).
- **This is product, not engineering.** Don't duplicate `feature-review.md`. If a finding is really
  a bug/security issue, note it and let the pre-ship review own it.
- **No silent caps.** If more than 8 high-severity claims exist, the report flags the unverified
  remainder as `⚠not-auto-verified` rather than hiding them.
