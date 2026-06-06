---
status: CUSTOM
description: Focused review of ONE proposed feature — the one the user is describing right now. A lean Workflow scans only the relevant code, runs a product advisor + a build advisor in parallel, and synthesizes one opinionated recommendation on whether and how to build it. Scoped to the feature, NOT the whole product (that's feature-product-review.md).
---

# Feature Advice (Scoped Feature Review)

Looks at **one feature** — the one under discussion — and returns an **opinion** plus **concrete
build advice**. It does **not** audit the product, map every surface, or hunt for unrelated gaps
(that's `feature-product-review.md`). It answers two questions about *this* feature:

1. **Is it worth building, and what's the sharpest version?** (product opinion)
2. **How should it be built, in this codebase?** (engineering advice)

The mechanism is a lean **Workflow**:

```
Scan only the code relevant to this feature
  → product advisor ∥ build advisor (parallel, independent perspectives)
    → synthesize one opinionated recommendation + ordered build plan
```

No whole-product map, no adversarial verify phase — this is design advice, not a gap audit. ~4
agents, fast.

---

## Capturing the feature ("in context")

The feature comes from **what the user is talking about**, not a fixed argument. Before running the
workflow, the caller distills a single clear **`FEATURE`** string from:
- the text after `/autofeature:feature-review`, if any, and
- the **recent conversation** (the feature the user has been describing).

If the feature is genuinely ambiguous, ask **one** quick clarifying question, then proceed. Pass the
distilled `FEATURE` (and any constraints the user stated) into the workflow.

---

## Running the review

Invoke the **Workflow** tool with the script below:

```
Workflow({
  script: <the script below, verbatim>,
  args: {
    repo:    "[pwd]",
    feature: "[the distilled feature under discussion]",
    notes:   "[any constraints/goals the user stated, or '']"
  }
})
```

> Opt-in note: this command explicitly instructs you to call the Workflow tool — that is the
> required opt-in. It spawns ~4 agents (1 scan + 2 advisors + 1 synthesis).

### The Workflow script

```js
export const meta = {
  name: 'feature-review',
  description: 'Focused review of ONE proposed feature — product opinion + engineering build advice, scoped to the feature under discussion (not the whole product)',
  phases: [
    { title: 'Scan',       detail: 'find only the code relevant to this feature' },
    { title: 'Advise',     detail: 'product advisor + build advisor in parallel' },
    { title: 'Synthesize', detail: 'one opinionated recommendation + build plan' },
  ],
}

const REPO    = (args.repo && String(args.repo).trim() && String(args.repo).toLowerCase() !== 'undefined') ? args.repo : '.'
const NOTES   = args.notes || ''
const _f      = String(args.feature ?? '').trim()
const FEATURE = _f

// Fail-fast: no feature to review. Return a valid recommendation (never throw) instead of spending
// agents on a generic review. NOTE: must catch the literal string 'undefined' — it is truthy, so a
// bare `if (!FEATURE)` would miss the exact case that bit us.
if (_f === '' || _f.toLowerCase() === 'undefined' || _f.toLowerCase() === 'null') {
  return {
    feature: '', stack: '',
    opinion: 'No feature was supplied to review.',
    recommendedApproach: 'Re-run /autofeature:feature-review with a one-line description of the feature (and the repo if not the current directory).',
    scope: { mvp: [], fastFollow: [], cut: [] },
    buildPlan: ['Name the feature, then re-run feature-review'],
    risks: [], openQuestions: ['Which feature should I review?'],
    readyToBuild: false, suggestedAutofeaturePrompt: '',
  }
}

// ---------- schemas ----------
const SCAN = {
  type: 'object', required: ['stack', 'touchpoints'],
  properties: {
    stack: { type: 'string' },
    touchpoints: { type: 'array', items: { type: 'object', required: ['path', 'role'],
      properties: { path: { type: 'string' }, role: { type: 'string' } } } },
    priorArt: { type: 'array', items: { type: 'string' } },   // existing code that already does something similar
    notes: { type: 'string' },
  },
}
const ADVICE = {
  type: 'object', required: ['lens', 'opinion', 'recommendations'],
  properties: {
    lens: { type: 'string' },
    verdict: { type: 'string' },
    opinion: { type: 'string' },
    recommendations: { type: 'array', items: { type: 'string' } },
    risks: { type: 'array', items: { type: 'string' } },
    openQuestions: { type: 'array', items: { type: 'string' } },
  },
}
const RECO = {
  type: 'object', required: ['opinion', 'recommendedApproach', 'buildPlan'],
  properties: {
    opinion: { type: 'string' },
    recommendedApproach: { type: 'string' },
    scope: { type: 'object', properties: {
      mvp: { type: 'array', items: { type: 'string' } },
      fastFollow: { type: 'array', items: { type: 'string' } },
      cut: { type: 'array', items: { type: 'string' } },
    } },
    buildPlan: { type: 'array', items: { type: 'string' } },
    risks: { type: 'array', items: { type: 'string' } },
    openQuestions: { type: 'array', items: { type: 'string' } },
    readyToBuild: { type: 'boolean' },
    suggestedAutofeaturePrompt: { type: 'string' },
  },
}

// ---------- Phase 1: Scan (only what THIS feature touches) ----------
phase('Scan')
const scan = await agent(
  `Find ONLY the code relevant to building this ONE feature. Do NOT map the whole product or audit unrelated areas.

Feature: ${FEATURE}
${NOTES ? `Constraints/goals: ${NOTES}` : ''}
Repo: ${REPO}

Return:
- stack: the relevant stack (e.g. "Express + Mongoose API", "React (Vite)", "React Native").
- touchpoints: the 3–8 files/modules this feature would create, modify, or should follow as a pattern — each with a one-line role.
- priorArt: any existing code that already does something similar (so the build reuses it, not reinvents it).
- notes: anything load-bearing the advisors should know (auth pattern, data-fetching layer, conventions).

Be concise and specific. Open the real files for the touchpoints you list.`,
  { schema: SCAN, phase: 'Scan' }
)
log(`Scanned: ${scan.stack || '?'} — ${scan.touchpoints?.length || 0} touchpoints, ${scan.priorArt?.length || 0} prior-art`)

// ---------- Phase 2: Advise (two independent perspectives, in parallel) ----------
phase('Advise')
const scanJson = JSON.stringify(scan)
const [product, build] = await parallel([
  () => agent(
    `You are a PRODUCT advisor (PM/founder sensibility). Scope: THIS feature ONLY — do not audit the rest of the product.

Feature: ${FEATURE}
${NOTES ? `Constraints/goals: ${NOTES}` : ''}
Relevant code context: ${scanJson}

Give an opinion: Is this worth building? Who is it for and what's the core job? What is the SHARPEST / smallest-valuable version (the MVP)? What should be deferred or cut? What does THIS feature specifically need to feel complete (the obvious adjacent piece a user expects)? Flag any product risk. Be opinionated and concrete. Set lens="product".`,
    { label: 'advise:product', phase: 'Advise', schema: ADVICE }
  ),
  () => agent(
    `You are a BUILD advisor (senior engineer for this stack). Scope: THIS feature ONLY.

Feature: ${FEATURE}
${NOTES ? `Constraints/goals: ${NOTES}` : ''}
Relevant code context: ${scanJson}

Advise HOW to build it in THIS codebase: the recommended approach/architecture, where it fits, the data-model / API / UI shape as relevant (sketch the contract), key technical decisions with trade-offs, the riskiest parts and edge cases, a test strategy, and a rough build sequence. REUSE the prior art and conventions from the scan — don't reinvent. Be specific to this repo's stack. Set lens="build".`,
    { label: 'advise:build', phase: 'Advise', schema: ADVICE }
  ),
])

// ---------- Phase 3: Synthesize one opinionated recommendation ----------
phase('Synthesize')
const reco = await agent(
  `Synthesize ONE opinionated recommendation for building this feature — advice a builder will act on directly.

Feature: ${FEATURE}
${NOTES ? `Constraints/goals: ${NOTES}` : ''}
Product advice: ${JSON.stringify(product)}
Build advice: ${JSON.stringify(build)}

Produce:
- opinion: 2–4 sentences — is it worth building, and what's the sharpest version? Take a clear stance.
- recommendedApproach: the approach you'd actually take, in this codebase.
- scope: { mvp: [build now], fastFollow: [soon after], cut: [explicitly not now] }.
- buildPlan: an ordered list of concrete build steps (backend → API → UI, or as fits the stack).
- risks: the few things most likely to bite (edge cases, data, perf, auth).
- openQuestions: decisions that need the user before building.
- readyToBuild: true if this could go straight to /autofeature, false if a decision is needed first.
- suggestedAutofeaturePrompt: a single ready-to-run feature request for "/autofeature [mode:] …" that captures the MVP scope.

Be decisive. If it's a bad idea or premature, say so and explain the better move.`,
  { schema: RECO, phase: 'Synthesize' }
)

return { feature: FEATURE, stack: scan.stack || '', ...reco }
```

---

## Report output

Render the returned recommendation as conversational, opinionated advice (not an audit table):

```
=== Feature Review: [feature] ===
Stack: [stack]

Opinion
  [opinion — take a clear stance]

Recommended approach
  [recommendedApproach]

Scope
  Build now (MVP):  [mvp …]
  Fast-follow:      [fastFollow …]
  Cut / defer:      [cut …]

Build plan
  1. [step]
  2. [step]
  …

Risks & edge cases
  - [risk]

Open questions
  - [question]

Ready to build? → /autofeature [suggestedAutofeaturePrompt]
```

If `readyToBuild` is true, end with the ready-to-run prompt and offer to kick it off. If false, lead
with the open questions — the feature isn't ready until they're answered.

---

## Relationship to the other review commands

| Command | Scope | Output |
|---------|-------|--------|
| `/autofeature:feature-review` (this) | **One feature** under discussion | Opinion + how to build it |
| `/autofeature:product-review` | **Whole product** (or a feature *in product context*) | Gaps & broken flows across the product |
| `/autofeature:scope` | One feature | Just the size tier + fan-out it implies |
| `/autofeature` | One feature | Actually builds and ships it |

Natural flow: `feature-review` (should we / how?) → `/autofeature` (build it).

## Guardrails

- **Stay scoped.** This reviews the feature under discussion — not the product. If the user wants a
  product-wide audit, point them at `/autofeature:product-review`.
- **Be opinionated.** The value is a clear stance, not a hedge. Recommend, with reasons.
- **Advice, not code.** This command does not branch, write, or ship. Building is `/autofeature`'s job.
