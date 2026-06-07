---
name: product-review
description: |
  CEO / PM / flow-walker product review — look at the product the way a founder, a product manager, and a careful first-time user would, and find gaps + broken/incomplete flows.
  Uses the Workflow tool: maps the product surface, fans out three product lenses in parallel, adversarially verifies broken-flow claims against the real code, and synthesizes a prioritized gaps-&-flows report.
  Audit mode (default): whole-product audit of the current repo. Fix mode: spin the top gaps into autofeature runs.
  Invoke as:
    /autofeature:product-review                      → audit current product
    /autofeature:product-review url: https://app.com → audit + note the live deployment
    /autofeature:product-review feature: <desc>      → evaluate a proposed feature in product context
    /autofeature:product-review fix: <gap>           → build a fix for a specific gap
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
  - WebFetch
  - Agent
  - Workflow
  - Skill
  - TaskCreate
  - TaskUpdate
  - mcp__trello__get_lists
  - mcp__trello__add_card_to_list
---

# AutoFeature Product Review — Find Gaps & Broken Flows

Runs the product-review workflow on its own — no branch, no implementation (unless you opt into a
fix). A founder/CEO lens, a PM lens, and a flow-walker lens look at the product, then a verification
pass checks every high-severity claim against the actual code so you get **real** gaps, not guesses.

This is **product** review, not engineering review. For correctness/security/tests use the pre-ship
review inside `/autofeature`. To review just **one feature** you're considering (an opinion + how to
build it) rather than the whole product, use `/autofeature:feature-review`.

## $AUTOFEATURE_HOME

```bash
# Files ship with the plugin — prefer its root; fall back to an explicit home or dev clone.
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

If `$AUTOFEATURE_HOME` doesn't exist, abort with:
`AutoFeature methodology repo missing. Expected at $AUTOFEATURE_HOME.`

---

## Step 1: Parse arguments and select mode

Arguments = everything after `/autofeature:product-review` in the user's message.

**Mode detection:**
- Contains `fix:` → **FIX MODE** — extract the gap description after `fix:` as `FIX_REQUEST`; skip to Step 5.
- Contains `feature:` → **FEATURE MODE** — extract the description after `feature:` as `FEATURE_DESC`; set workflow `mode = "feature"`.
- Otherwise → **AUDIT MODE** — set workflow `mode = "product"`.
- Contains `url:` → extract the URL as `SITE_URL` (works with audit or feature mode).

---

## Step 2: Project detection (light)

Spawn a single **Explore** agent to gather just enough signal — do not grep/glob in main context:

```
Agent({
  description: "Product-review context scan",
  subagent_type: "Explore",
  model: "haiku",   // surface map — orchestrator/model-tiers.md
  prompt: "Quick scan for a product review (no implementation).
  Repo: [pwd]
  Return only:
  1. Stack signals — package.json deps (express|fastify|@nestjs|koa) | (react|react-native) | mongoose; ios/ or android/ folders
  2. The user-facing surfaces — route file(s), screen/page dir(s), the main nav/router config
  3. Sibling-repo hints — is pwd part of a *-api / *-mobile / *-web / *-desktop family (check the parent dir)
  Under 400 words. Paths + one-line roles only."
})
```

If `pwd` is not a git repo, note that and continue — the workflow still maps from the files present.

If `feature` mode and there's no Feature Brief yet, that's fine — the workflow reads `FEATURE_DESC`
directly (pass it as `featureBrief: ""` and inject the description into the Map prompt context, or
write a minimal brief to `.autofeature/designs/[slug]-[date].md` first if you prefer a saved
artifact).

---

## Step 3: Run the product-review workflow

Read `$AUTOFEATURE_HOME/adapted/feature-product-review.md` and invoke the **Workflow** tool with the
script it contains, passing:

```
Workflow({
  script: <the script from feature-product-review.md>,
  args: {
    repo:         "[pwd]",
    mode:         "product" | "feature",
    featureBrief: "[brief path or '']",
    siteUrl:      "[SITE_URL or '']"
  }
})
```

The workflow runs in the background and returns a structured synthesis. Do **not** re-run the
analysis yourself — relay what it returns.

---

## Step 4: Render the report

Format the returned synthesis using the **Report output** block in `feature-product-review.md`.
In `product` mode, save the full report to `.autofeature/product-review-[YYYY-MM-DD].md`.

Then offer next steps:

```
Found [N] product issues ([C] critical, [H] high).

A) Build the top fixes now      ← recommended if C > 0
B) Fix a specific gap            ← pick from the numbered list
C) Create Trello cards           ← one card per significant gap
D) Report only — no changes
```

- **A / B:** set `FIX_REQUEST` from the chosen finding(s)' `recommendation` and proceed to Step 5.
- **C:** ask which list (`mcp__trello__get_lists`), then `mcp__trello__add_card_to_list` per gap
  (title = gap, body = where + recommendation + severity, footer links the saved report). Exit.
- **D:** print the report path and exit.

---

## Step 5 (FIX MODE): Build the fix via the autofeature pipeline

Hand `FIX_REQUEST` to the full autofeature pipeline
(`$AUTOFEATURE_HOME/.claude/commands/autofeature.md`):

- **Append `[skip-product-review]`** to the feature request so the pre-build product review
  (Step 4.5) does **not** recurse on a fix this review just generated.
- If a saved product-review report exists, pass its path so the Feature Brief can reference it under
  `## Product Review`.
- For multiple chosen gaps, run them **one feature at a time** (sequential autofeature runs) unless
  the user explicitly asks to batch into one.

Everything else (scope gate, planning, implement, test, review, ship) runs exactly as standard
`/autofeature`.

---

## File Reference

| File | Purpose |
|------|---------|
| `adapted/feature-product-review.md` | The methodology + the Workflow script (map → CEO/PM/flow-walker lenses → verify → synthesize) |
| `agents/product-strategist.md` | The reviewer personas — what the CEO, PM, and flow-walker lenses each own, severity rubric, output contract |

This command never branches or ships on its own. It only builds when you choose a fix in Step 4.
