---
name: scope
description: |
  Classify a feature's scope tier without running the full pipeline.
  Runs only the scope-gate step: micro | single-layer | cross-stack | cross-repo.
  Use it to preview how /autofeature would size a feature, decide how many specialists a change needs, or sanity-check before a full run.
  Invoke as: /autofeature:scope <feature description>
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
---

# AutoFeature — Scope Gate (standalone)

Runs the classification step of the autofeature pipeline on its own. No branch, no
implementation, no ship — just a scope tier and the fan-out it implies.

## $AUTOFEATURE_HOME

```bash
# Files ship with the plugin — prefer its root; fall back to an explicit home or dev clone.
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

If `$AUTOFEATURE_HOME` doesn't exist, abort with:
`AutoFeature methodology repo missing. Expected at $AUTOFEATURE_HOME.`

## Step 1: Parse input

`FEATURE_REQUEST` = everything after `/autofeature:scope` in the user's message.

If empty, ask the user for a one-line feature description and stop until they answer.

## Step 2: Light context scan (optional, for accuracy)

The scope gate classifies better with a little repo context. If the current directory
is a git repo, spawn a single Explore agent to gather just enough signal — do **not**
grep/glob in main context:

```
Agent({
  description: "Scope-gate context scan",
  subagent_type: "Explore",
  model: "haiku",   // lightweight classification — orchestrator/model-tiers.md
  prompt: "Quick scan for a scope classification (no implementation).

  Feature request: [FEATURE_REQUEST verbatim]
  Repo: [pwd]

  Return only:
  1. Stack signals — package.json deps matching (express|fastify|@nestjs|koa) | (react|react-native) | mongoose; presence of ios/ or android/ folders
  2. Whether the feature likely touches a route/endpoint, the data model, and/or a UI surface (best guess from existing code)
  3. Sibling-repo hints — does pwd look like part of a *-api / *-mobile / *-desktop / *-cms / *-website family (check parent dir for sibling folders)

  Under 400 words. Paths + one-line roles only."
})
```

If pwd is not a git repo, skip the scan and classify from the request text alone — note
in the output that classification is request-only (lower confidence).

## Step 3: Classify

Read and follow `$AUTOFEATURE_HOME/orchestrator/scope-gate.md`. Apply its classification
table and heuristics to `FEATURE_REQUEST` plus any context from Step 2.

For the `cross-repo` tier, you may read `$AUTOFEATURE_HOME/orchestrator/cross-repo-detect.md`
to confirm siblings actually exist — but do not create branches or modify anything.

## Step 4: Output

Print the scope analysis directly to the user (do not write a Feature Brief — this is a
standalone preview):

```
=== Scope Classification ===
Feature:    [FEATURE_REQUEST]
Tier:       [micro | single-layer | cross-stack | cross-repo]
Confidence: [high | medium | request-only]

Reasoning:  [1-2 sentences — what triggered this tier]

Would spawn:
  - [specialist agents — or "none (linear flow in main context)" for micro]

Would invoke skills:
  - [Plan, Explore, security-review, simplify, frontend-design, as applicable]

Cross-repo siblings: [list, or "n/a"]
```

Then offer the natural next step:

> Run it? `/autofeature [mode:] [FEATURE_REQUEST]` to build this end-to-end,
> or bump the tier up/down if this sizing looks off.

This command never branches, implements, or ships. It is read-only.
