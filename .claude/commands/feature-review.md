---
name: feature-review
description: |
  Focused review of the ONE feature you're talking about — not the whole product. Gives an opinion (is it worth building? what's the sharpest/MVP version?) plus engineering build advice (approach, where it fits, risks, test strategy, what to cut), ending with a ready-to-run /autofeature prompt.
  Uses a lean Workflow: scans only the relevant code, runs a product advisor + a build advisor in parallel, and synthesizes one opinionated recommendation.
  For a whole-product gap/flow audit use /autofeature:product-review instead.
  Invoke as:
    /autofeature:feature-review <feature description>
    /autofeature:feature-review            → reviews the feature under discussion in this conversation
---

# AutoFeature Feature Review — Opinion + Build Advice

A scoped, advisory review of a single feature: *should we build this, and how?* It reviews **only
the feature you're describing**, reuses your codebase's patterns, and hands back an opinionated
recommendation you can act on (or run straight through `/autofeature`).

This is **not** the whole-product audit — for gaps & broken flows across the product, use
`/autofeature:product-review`. This is **not** the pre-ship code review either. It's design advice,
before you build.

## $AUTOFEATURE_HOME

```bash
AUTOFEATURE_HOME="${AUTOFEATURE_HOME:-$HOME/dev/autofeature}"
```

If `$AUTOFEATURE_HOME` doesn't exist, abort with:
`AutoFeature methodology repo missing. Expected at $AUTOFEATURE_HOME.`

---

## Step 1: Capture the feature from context

The feature is **what the user is talking about** — not just a literal argument. Distill a single
clear `FEATURE` string from:
- the text after `/autofeature:feature-review`, if any, **and**
- the **recent conversation** (the feature the user has been describing).

Also capture any **constraints/goals** the user stated (deadline, must-reuse-X, "keep it small",
target users) as `NOTES`.

- If both the argument and the conversation are empty/ambiguous about which feature, ask **one**
  short clarifying question: *"Which feature should I review — [best guess]?"* Then proceed.
- Briefly echo what you're reviewing so the user can correct course:
  > Reviewing: **[FEATURE]** — [one-line restatement]. Running a focused product + build review…

---

## Step 2: Light project check (optional)

If `pwd` is a git repo, the workflow's Scan phase will read the relevant code itself — you do **not**
need to grep/glob in main context. If `pwd` is not a repo (pure greenfield idea), note that the
advice will be stack-general, and continue.

---

## Step 3: Run the feature-review workflow

Read `$AUTOFEATURE_HOME/adapted/feature-advice.md` and invoke the **Workflow** tool with the script
it contains, passing:

```
Workflow({
  script: <the script from feature-advice.md>,
  args: {
    repo:    "[pwd]",
    feature: "[FEATURE]",
    notes:   "[NOTES or '']"
  }
})
```

The workflow scans only the relevant code, runs a product advisor + a build advisor in parallel, and
returns one synthesized recommendation. Do **not** redo the analysis yourself — relay what it
returns.

---

## Step 4: Render the recommendation

Format the returned recommendation using the **Report output** block in `feature-advice.md` —
conversational and opinionated, not an audit table. Lead with the **Opinion**, then the recommended
approach, scope split (MVP / fast-follow / cut), build plan, risks, and open questions.

Then offer the natural next step:

- If `readyToBuild` is **true**:
  > Ready to build. Want me to run it?
  > `/autofeature [suggestedAutofeaturePrompt]`
- If `readyToBuild` is **false**:
  > A couple of decisions first (above). Answer those and I'll either re-review or build it.

This command never branches, writes code, or ships — it only advises. Building is `/autofeature`'s job.

---

## File Reference

| File | Purpose |
|------|---------|
| `adapted/feature-advice.md` | The methodology + the Workflow script (scan → product advisor ∥ build advisor → synthesize) |

| Want instead… | Use |
|---------------|-----|
| A whole-product gap/flow audit | `/autofeature:product-review` |
| Just the scope tier of this feature | `/autofeature:scope` |
| To actually build it | `/autofeature [mode:] <feature>` |
