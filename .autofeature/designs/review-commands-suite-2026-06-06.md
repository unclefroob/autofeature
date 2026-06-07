# Feature Brief — Review-Command Suite (self-review subject)

This brief scopes a product review to **"what was just built"**: a suite of three new review
commands added to the autofeature plugin, their agents, methodologies, and pipeline integration.
The "product" under review is the autofeature plugin; the "user journeys" are how a user invokes
these commands and flows through to an outcome. Treat the slash commands as surfaces, the
`adapted/*.md` methodologies + Workflow scripts as the engines, and the `agents/*.md` as personas.

## What was built (the proposed feature = this suite)

1. **/autofeature:product-review** — whole-product CEO/PM/flow-walker review.
   - Command: `.claude/commands/product-review.md` (modes: audit / `feature:` / `url:` / `fix:`)
   - Methodology + Workflow: `adapted/feature-product-review.md` (map → 3 lenses → adversarial verify → synthesize)
   - Agent: `agents/product-strategist.md`
   - Pipeline integration: `autofeature.md` **Step 4.5** (pre-build), Task 4 inserted (Ship → Task 10),
     `[skip-product-review]` marker to prevent recursion; `fullrun.md` Task renumber (Deploy → Task 11);
     `seo.md` fix carries the skip marker.

2. **/autofeature:feature-review** — scoped review of ONE feature (opinion + build advice).
   - Command: `.claude/commands/feature-review.md`
   - Methodology + Workflow: `adapted/feature-advice.md` (scan → product advisor ∥ build advisor → synthesize)
   - Captures the feature from conversation context; ends with a ready-to-run /autofeature prompt.

3. **/autofeature:market-review** — market demand + competitive gap + VC fundability.
   - Command: `.claude/commands/market-review.md` (live cited web research; `offline:` toggle)
   - Methodology + Workflow: `adapted/market-review.md` (frame → market ∥ gap ∥ VC → bear case → synthesize)
   - Agents: `agents/market-analyst.md`, `market-gap-analyst.md`, `vc-analyst.md`, `bear-case-analyst.md`
   - Output: investment memo saved to `.autofeature/market-review-[date].md`

## Intended user journeys (flows to check for dead-ends)

- **Discover & pick:** a user wants to evaluate something → picks the right command among
  scope / feature-review / product-review / market-review / autofeature. (Are the boundaries clear?
  Do cross-references resolve?)
- **product-review → fix:** run audit → choose "Build the top fixes" (A) or "Fix a specific gap" (B)
  → hand off to `/autofeature` with `[skip-product-review]`. (Does the returned synthesis carry the
  data each option needs — e.g. recommendations to build from?)
- **feature-review → build:** review a feature → `readyToBuild` + `suggestedAutofeaturePrompt` →
  run `/autofeature`. (Is the handoff real and runnable?)
- **market-review → build:** memo → option B "turn gaps into a build plan" → feature-review/autofeature.
- **Pre-build integration:** `/autofeature` on a non-micro feature → Step 4.5 runs product-review in
  feature mode → findings fold into the plan via the `## Product Review` brief section. (Does the
  Plan subagent actually consume it? Is the skip logic correct? Task numbering consistent?)

## Conventions the suite should honor

- Workflow scripts: `meta` pure literal; no `Date.now`/`Math.random`; agent prompts self-contained
  (workflows can't read agent files at runtime — agent files are editable specs, prompts live inline).
- Each command: `$AUTOFEATURE_HOME` guard, `allowed-tools` includes `Workflow` (+ `WebSearch`/`WebFetch`
  for market-review), frontmatter `name`, File Reference table.
- Docs: MANIFEST rows, agents/README roster, version bump (now 1.6.0), plugin.json/marketplace.json.

## Out of scope for this review

The pre-existing architects (express-mongo, react, react-native, swift), seo, scope, fullrun, and the
core build pipeline — except where the new suite touches them (Step 4.5, task renumbering, skip marker).
