---
status: CUSTOM
description: Market & fundability review. A Workflow frames the product/idea into a venture brief, runs three analysts in parallel (market demand + sizing, competitive gap, VC fundability) with cited live web research, stress-tests the thesis with an adversarial bear-case analyst, and synthesizes a managing-partner investment memo. Answers is-it-useful, where-are-the-market-gaps, and can-I-get-funding.
---

# Market Review (Useful? Market Gap? Fundable?)

Evaluates a product or idea from the **outside in** — not its code quality, but whether it should
exist as a business and whether it can raise money. Three questions:

1. **Is it useful?** — real, acute problem; who has it; demand; market size.
2. **Is there a gap in the market?** — competitors, substitutes, white space, defensibility, why-now.
3. **Can I get funding?** — venture-scale?, comparable raises, what a partner asks, fundable verdict.

The mechanism is a **Workflow** with **live, cited web research** and an **adversarial bear case**:

```
Frame the idea into a venture brief
  → market analyst ∥ market-gap analyst ∥ VC analyst   (parallel, each cites sources)
    → bear-case analyst stress-tests the thesis & numbers
      → managing-partner synthesis → investment memo + verdict
```

Personas live in `agents/`: `market-analyst.md`, `market-gap-analyst.md`, `vc-analyst.md`,
`bear-case-analyst.md`. Those files are the **editable specs** — the operational prompts that run are
inline in the Workflow script below; keep the two in sync (workflows can't read files at runtime).

> **This is decision-support, not financial advice or a funding guarantee.** Market sizes,
> competitors, and funding comps are the easiest things to get wrong — the design forces sourced
> research, confidence flags, and an adversarial pass, but every estimate still needs the founder's
> own validation.

---

## Running the review

Invoke the **Workflow** tool with the script below:

```
Workflow({
  script: <the script below, verbatim>,
  args: {
    repo:        "[pwd]",
    idea:        "[the product/idea under review, distilled from the user + conversation]",
    notes:       "[stated stage / traction / constraints, or '']",
    webResearch: true       // false = model-knowledge-only, numbers flagged unsourced
  }
})
```

> Opt-in note: this command explicitly instructs you to call the Workflow tool — that is the
> required opt-in. It spawns 6 agents (1 frame + 3 analysts + 1 bear + 1 synthesis) and makes live
> web calls when `webResearch` is true.

### The Workflow script

```js
export const meta = {
  name: 'market-review',
  description: 'Market & fundability review — frame, research (market + gap + VC, cited), bear-case stress test, managing-partner investment memo',
  phases: [
    { title: 'Frame',      detail: 'capture the idea into a venture brief' },
    { title: 'Research',   detail: 'market + market-gap + VC analysts in parallel (cited web research)' },
    { title: 'Bear case',  detail: 'adversarial skeptic stress-tests the thesis & claims' },
    { title: 'Synthesize', detail: 'managing-partner investment memo + verdict' },
  ],
}

const REPO  = args.repo
const IDEA  = args.idea
const NOTES = args.notes || ''
const WEB   = args.webResearch !== false   // default true

const webLine = WEB
  ? 'Use WebSearch and WebFetch to ground every quantitative claim in a real, recent source. Cite sources as { claim, url, note }. Mark anything you cannot source as "unsourced" and lower confidence. NEVER fabricate numbers, competitors, or funding comps.'
  : 'Web research is DISABLED — reason from general knowledge, but tag every market size / competitor / funding figure as "model-knowledge-only — verify" and set confidence low. Do not present guesses as facts.'

// ---------- schemas ----------
const SRC = { type: 'array', items: { type: 'object', properties: { claim: { type: 'string' }, url: { type: 'string' }, note: { type: 'string' } } } }
const VENTURE_BRIEF = {
  type: 'object', required: ['product', 'problem', 'icp'],
  properties: {
    product: { type: 'string' }, oneLiner: { type: 'string' }, problem: { type: 'string' },
    icp: { type: 'string' }, businessModelGuess: { type: 'string' }, stage: { type: 'string' }, tractionNotes: { type: 'string' },
  },
}
const MARKET = {
  type: 'object', required: ['usefulnessVerdict', 'marketSize', 'confidence'],
  properties: {
    problem: { type: 'string' }, painkillerOrVitamin: { type: 'string' },
    alternatives: { type: 'array', items: { type: 'string' } },
    demandSignals: { type: 'array', items: { type: 'string' } },
    marketSize: { type: 'object', properties: { tam: { type: 'string' }, sam: { type: 'string' }, som: { type: 'string' }, method: { type: 'string' }, year: { type: 'string' } } },
    tailwinds: { type: 'array', items: { type: 'string' } },
    usefulnessVerdict: { type: 'string' }, sources: SRC, confidence: { type: 'string' },
  },
}
const GAP = {
  type: 'object', required: ['gapVerdict', 'confidence'],
  properties: {
    competitors: { type: 'array', items: { type: 'object', properties: {
      name: { type: 'string' }, type: { type: 'string' }, positioning: { type: 'string' }, pricing: { type: 'string' },
      tractionOrFunding: { type: 'string' }, strength: { type: 'string' }, weakness: { type: 'string' }, url: { type: 'string' } } } },
    whiteSpace: { type: 'array', items: { type: 'string' } },
    differentiation: { type: 'string' }, moat: { type: 'string' }, copyability: { type: 'string' }, whyNow: { type: 'string' },
    gapVerdict: { type: 'string' }, sources: SRC, confidence: { type: 'string' },
  },
}
const VC = {
  type: 'object', required: ['fundability', 'confidence'],
  properties: {
    ventureScale: { type: 'object', properties: { verdict: { type: 'string' }, why: { type: 'string' } } },
    businessModel: { type: 'string' }, unitEconomics: { type: 'string' }, stageBenchmark: { type: 'string' },
    comparableRaises: { type: 'array', items: { type: 'object', properties: {
      company: { type: 'string' }, stage: { type: 'string' }, amount: { type: 'string' }, valuation: { type: 'string' },
      investors: { type: 'string' }, year: { type: 'string' }, url: { type: 'string' } } } },
    partnerObjections: { type: 'array', items: { type: 'string' } },
    fundability: { type: 'object', properties: {
      fundable: { type: 'string' }, stage: { type: 'string' }, checkRange: { type: 'string' }, valuationBallpark: { type: 'string' },
      milestones: { type: 'array', items: { type: 'string' } }, altFunding: { type: 'array', items: { type: 'string' } } } },
    sources: SRC, confidence: { type: 'string' },
  },
}
const BEAR = {
  type: 'object', required: ['bearThesis', 'killShots', 'confidence'],
  properties: {
    bearThesis: { type: 'string' },
    weakestClaims: { type: 'array', items: { type: 'object', properties: { claim: { type: 'string' }, fromAnalyst: { type: 'string' }, why: { type: 'string' }, howToTest: { type: 'string' } } } },
    graveyards: { type: 'array', items: { type: 'object', properties: { who: { type: 'string' }, whatHappened: { type: 'string' }, whyFailed: { type: 'string' }, url: { type: 'string' } } } },
    killShots: { type: 'array', items: { type: 'string' } },
    preMortem: { type: 'string' }, passReasons: { type: 'array', items: { type: 'string' } }, confidence: { type: 'string' },
  },
}
const MEMO = {
  type: 'object', required: ['verdict', 'summary', 'fundability'],
  properties: {
    verdict: { type: 'object', properties: { useful: { type: 'string' }, marketGap: { type: 'string' }, fundable: { type: 'string' } } },
    summary: { type: 'string' }, usefulness: { type: 'string' }, marketGap: { type: 'string' },
    fundability: { type: 'object', properties: {
      verdict: { type: 'string' }, stage: { type: 'string' }, checkRange: { type: 'string' }, valuationBallpark: { type: 'string' },
      milestones: { type: 'array', items: { type: 'string' } }, altFunding: { type: 'array', items: { type: 'string' } } } },
    risks: { type: 'array', items: { type: 'string' } }, nextSteps: { type: 'array', items: { type: 'string' } },
    investorOnePager: { type: 'string' }, confidence: { type: 'string' },
    citationsChecked: { type: 'object', properties: { confirmed: { type: 'number' }, unconfirmed: { type: 'number' }, dead: { type: 'number' }, stale: { type: 'number' } } },
  },
}
// Per-citation re-verification verdict. 'dead' = URL didn't resolve; 'unconfirmed' = page loaded
// but the figure is absent/different; 'confirmed' = page loads AND states the figure. Keep the
// three states distinct — a confident citation to a real page that doesn't say what was claimed
// is the most useful signal.
const CLAIM_VERDICT = {
  type: 'object', required: ['supported', 'reason'],
  properties: {
    supported: { type: 'string', enum: ['confirmed', 'unconfirmed', 'dead'] },
    reason: { type: 'string' }, foundFigure: { type: 'string' }, stale: { type: 'boolean' },
  },
}

// ---------- Phase 1: Frame ----------
phase('Frame')
const brief = await agent(
  `Frame this for a market & fundability review — distill a venture brief.

Idea/product (from the user): ${IDEA}
${NOTES ? `Stated stage/traction/constraints: ${NOTES}` : ''}
Repo: ${REPO} — if it's a real project, read the README / landing copy to ground what the product actually is, who it's for, and the problem it solves. If it's just an idea, work from the description.

Return: product, oneLiner, problem, icp (ideal customer), businessModelGuess, stage (idea|prototype|mvp|launched|revenue), tractionNotes (anything the user said about users/revenue/signups).`,
  { schema: VENTURE_BRIEF, phase: 'Frame' }
)
const briefJson = JSON.stringify(brief)
log(`Framed: ${brief.oneLiner || brief.product || IDEA}`)

// ---------- Phase 2: Research (3 analysts in parallel; barrier so the bear sees all) ----------
phase('Research')
const [market, gap, vc] = await parallel([
  () => agent(
    `[MARKET ANALYST] Judge usefulness & demand, and size the market.
${webLine}
Venture brief: ${briefJson}
Assess: is the problem real/acute and who has it (ICP); painkiller vs vitamin; how it's solved today; demand signals; market size TAM/SAM/SOM built BOTTOM-UP (target customers × realistic ACV) and reconciled top-down — show the math and the year; tailwinds / why-now. Lead with bottom-up sizing. Return the structured contract with sources[] and a confidence level.`,
    { label: 'research:market', phase: 'Research', schema: MARKET }
  ),
  () => agent(
    `[MARKET-GAP ANALYST] Map competition and find the defensible gap.
${webLine}
Venture brief: ${briefJson}
Assess: direct competitors, substitutes (incl. status quo / DIY), and adjacent players that could enter — each with positioning, pricing, traction/funding, strength, weakness, url; the white space (unowned positions / underserved segments); the wedge (concrete reason a first customer switches); the moat (what compounds, or "copyable") and copyability by a funded incumbent; why-now. There is ALWAYS a status quo — if you find "no competitors," find the substitute. Return the structured contract with sources[] and confidence.`,
    { label: 'research:gap', phase: 'Research', schema: GAP }
  ),
  () => agent(
    `[VC ANALYST] Judge fundability like an early-stage partner.
${webLine}
Venture brief: ${briefJson}
First answer HONESTLY whether this is venture-scale or a good non-VC business. Then: business model + unit-economics sketch; the likely stage and its traction bar vs where this is; comparable RECENT raises (company, stage, amount, valuation, investors, year, url) — sourced; the 3–5 hardest partner objections; and a verdict { fundable, stage, checkRange, valuationBallpark, milestones, altFunding }. Do not fabricate funding numbers — if unsourced, say so and lower confidence. Return the structured contract.`,
    { label: 'research:vc', phase: 'Research', schema: VC }
  ),
])

// aggregate sources across analysts (plain JS)
const allSources = [market, gap, vc].filter(Boolean).flatMap(a => a.sources || [])
const distinctUrls = new Set(allSources.map(s => (s.url || '').trim()).filter(Boolean))
log(`Research done — ${distinctUrls.size} distinct sources cited (market/${market?.confidence || '?'}, gap/${gap?.confidence || '?'}, vc/${vc?.confidence || '?'})`)

// ---------- Phase 3: Bear case ----------
phase('Bear case')
const bear = await agent(
  `[BEAR-CASE ANALYST] Argue why this FAILS and stress-test the analysts.
${WEB ? 'Use WebSearch/WebFetch to find startups that tried this and died, and why. Cite them.' : 'Web disabled — reason from general knowledge; tag graveyards as unverified.'}
Venture brief: ${briefJson}
Market findings: ${JSON.stringify(market)}
Gap findings: ${JSON.stringify(gap)}
VC findings: ${JSON.stringify(vc)}

Build the strongest case to PASS. Flag inflated/unsourced numbers (especially top-down TAMs), white space that's actually a graveyard, illusory moats, cherry-picked or stale comps, demand signals that are noise. List graveyards (who tried, what happened, why, url). Name the 1–3 kill-shots. Write an 18-month pre-mortem. If after a genuine attempt you can't build a strong bear case, SAY SO — that's a bullish signal. Return the structured contract.`,
  { schema: BEAR, phase: 'Bear case' }
)

// ---------- Phase 3.5: Verify citations (re-fetch each cited URL; web only) ----------
// The bear case checks plausibility, not citation validity. This pass re-opens the cited URLs
// and confirms they actually support the stated figure — funding comps are the easiest to fabricate.
let verified = []
const citeCounts = { confirmed: 0, unconfirmed: 0, dead: 0, stale: 0 }
if (WEB) {
  phase('Verify')
  const VERIFY_CAP = (args.maxVerify && Number(args.maxVerify)) || 6
  // candidates, funding-comps first (highest fabrication risk), then the other cited sources
  const compClaims = (vc?.comparableRaises || []).filter(c => c && c.url).map(c => ({
    kind: 'comp', url: c.url,
    what: `${c.company || '?'} ${c.stage || ''} ${c.amount || ''} @ ${c.valuation || '?'} (${c.year || '?'})`.replace(/\s+/g, ' ').trim(),
  }))
  const sourceClaims = allSources.filter(s => s && s.url).map(s => ({ kind: 'source', url: s.url, what: s.claim || s.note || 'cited figure' }))
  const seenU = new Set()
  const candidates = [...compClaims, ...sourceClaims].filter(c => {
    const u = (c.url || '').trim(); if (!u || seenU.has(u)) return false; seenU.add(u); return true
  })
  let toVerify = candidates, vcap = 0
  if (toVerify.length > VERIFY_CAP) { vcap = toVerify.length - VERIFY_CAP; toVerify = toVerify.slice(0, VERIFY_CAP) }
  if (vcap) log(`Verifying top ${VERIFY_CAP} of ${candidates.length} citations (${vcap} not re-checked — flagged in memo)`)
  verified = (await parallel(toVerify.map(c => () =>
    agent(
      `Re-fetch this URL with WebFetch and judge whether the page actually CONTAINS and SUPPORTS the cited claim. Do not trust the citation — open it.
URL: ${c.url}
Cited for: "${c.what}"
supported = "confirmed" only if the page loads AND states the figure/fact; "unconfirmed" if it loads but the number is absent or different; "dead" if it 404s or doesn't resolve. foundFigure = the actual number on the page (or ''). stale = true if the supporting figure is clearly >18 months old.`,
      { label: `verify:${c.kind}`, phase: 'Verify', schema: CLAIM_VERDICT }
    ).then(v => ({ ...c, ...v }))
  ))).filter(Boolean)
  for (const v of verified) {
    if (v.supported === 'confirmed') citeCounts.confirmed++
    else if (v.supported === 'dead') citeCounts.dead++
    else citeCounts.unconfirmed++
    if (v.stale) citeCounts.stale++
  }
  log(`Citations: ${citeCounts.confirmed} confirmed, ${citeCounts.unconfirmed} unconfirmed, ${citeCounts.dead} dead${citeCounts.stale ? `, ${citeCounts.stale} stale` : ''}`)
}

// ---------- Phase 4: Synthesize ----------
phase('Synthesize')
const memo = await agent(
  `[MANAGING PARTNER] Write the investment memo. Weigh the bull case (market/gap/vc) against the bear case and reach a BALANCED, decisive verdict. DOWNGRADE or DROP any bull claim the bear successfully undercut. Be explicit about confidence and that estimates need the founder's validation.

Venture brief: ${briefJson}
Market: ${JSON.stringify(market)}
Gap: ${JSON.stringify(gap)}
VC: ${JSON.stringify(vc)}
Bear: ${JSON.stringify(bear)}
Distinct sources cited so far: ${distinctUrls.size}
Citation re-verification (${WEB ? 'on' : 'off — figures unverified'}): ${JSON.stringify(citeCounts)}
Per-citation verdicts (re-fetched URLs): ${JSON.stringify(verified)}

DOWNGRADE confidence and append a "⚠ unverified" tag to any figure whose citation came back "unconfirmed" or "dead"; append "⚠ stale" to a confirmed comp >18 months old. KEEP unverifiable comps in the memo with the loud tag — don't drop the lead. Treat an unverified citation exactly like a claim the bear successfully undercut.

Produce:
- verdict: { useful: "yes|no|partly — one line", marketGap: "clear|contested|none — one line", fundable: "yes|no|conditional — one line" }
- summary: 3–5 sentences a founder reads first — the honest bottom line.
- usefulness: the usefulness call with the demand evidence.
- marketGap: the gap (or lack of one) and how contested.
- fundability: { verdict, stage, checkRange, valuationBallpark, milestones[], altFunding[] } — reconciled with the bear case.
- risks: top risks (merge VC objections + bear kill-shots; dedupe).
- nextSteps: 3–6 concrete next actions (validate X, build Y, talk to Z investors, hit milestone M).
- investorOnePager: a tight, honest paragraph for an investor (problem, who, wedge, why-now, the ask) — no hype.
- citationsChecked: copy the counts object above ({ confirmed, unconfirmed, dead, stale }).
- confidence: overall + what would raise it.

Be honest above all. If the answer is "useful but NOT venture-fundable," say exactly that and point to the right funding path.`,
  { schema: MEMO, phase: 'Synthesize' }
)

return {
  idea: IDEA,
  web: WEB,
  brief,
  ...memo,
  sourcesCited: distinctUrls.size,
  sources: allSources,
  citationsChecked: citeCounts,        // authoritative JS counts (override any model value)
  verifiedCitations: verified,
  bear: { killShots: bear?.killShots || [], graveyards: bear?.graveyards || [] },
}
```

---

## Report output — Investment Memo

Render the returned memo as an investment memo, and save the full version to
`.autofeature/market-review-[YYYY-MM-DD].md`:

```
=== Market Review: [product / idea] ===
Stage: [stage]   Web research: [on/off]
Sources cited: [N]  ·  Re-verified: [confirmed] ✓ confirmed / [unconfirmed] ⚠ unconfirmed / [dead] ✗ dead[ · [stale] stale]

VERDICT
  Useful?       [yes|no|partly] — [one line]
  Market gap?   [clear|contested|none] — [one line]
  Fundable?     [yes|no|conditional] — [one line]

Bottom line
  [summary — the honest read]

1. Useful?
  [usefulness — demand evidence, painkiller/vitamin, market size TAM/SAM/SOM with method]
  (tag each high-stakes figure inline per its citation verdict: [verified ✓] / [unverified ⚠] / [stale])

2. Market gap?
  [the gap, competitors/substitutes, the wedge, moat & copyability, why-now]

3. Fundable?
  Verdict:        [verdict]
  Likely stage:   [stage]  ·  Check: [checkRange]  ·  Valuation: [valuationBallpark]
  Comparable raises: [company stage amount @ valuation (year) — tag [verified ✓] / [unverified ⚠] / [stale] each]
  Milestones to unlock it: [milestones]
  If not VC:      [altFunding]

Top risks
  - [risk]   (incl. bear-case kill-shots)

Bear case
  - Kill-shots: [killShots]
  - Graveyards: [who tried & died, why]

Next steps
  1. [step]

Investor one-pager
  [investorOnePager]

Sources ([N])
  - [claim] — [url]
  …

⚠ Estimates are decision-support, not verified fact. Validate market size, comps, and demand before
  betting on them. This is not financial advice or a guarantee of funding.
```

After the memo, offer:

```
A) Pressure-test a specific number / claim   → re-run research on one dimension
B) Turn the gaps into a build plan           → /autofeature:feature-review or /autofeature
C) Save & done
```

---

## Relationship to the other commands

| Command | Question |
|---------|----------|
| `/autofeature:market-review` (this) | Should this exist as a business — useful? gap? fundable? |
| `/autofeature:product-review` | Does the built product have gaps / broken flows? |
| `/autofeature:feature-review` | Should we build this one feature, and how? |
| `/autofeature` | Build & ship it. |

Natural flow: **market-review** (is the business real?) → **feature-review / product-review**
(is the product right?) → **/autofeature** (build it).

## Guardrails

- **Sourced or flagged.** Every market/competitor/funding number is either cited or explicitly marked
  unsourced/low-confidence. The bear-case pass exists to catch the rest. Never launder a guess into a fact.
- **Honest on venture-scale.** Plenty of useful products aren't VC-fundable. Say so and point to the
  right funding rather than forcing a unicorn frame.
- **Advisory only.** This command researches and advises. It does not raise money, contact investors,
  or build anything.
