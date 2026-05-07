---
name: scope-gate
purpose: Classify feature size before subagent fan-out
status: CUSTOM
---

# Scope Gate

The orchestrator runs this **after Step 1 (mode select) and before Step 2 (interrogation)**. Its job: classify the feature so we don't spawn 5 specialists for a one-line config tweak.

## Why

Subagents have a real cost: each one is a separate model call with its own context allocation. For a small feature, the orchestrator can do the work itself in less time than it takes to brief 3 agents. Fan-out is for genuine parallelism, not theatre.

## Classification

Read the feature request. Pick the smallest tier that honestly fits.

| Tier | Definition | Fan-out |
|------|------------|---------|
| **micro** | One file edited. No new tests beyond touching existing ones. No schema, no route, no UI surface added. | None. Linear flow in main context. Skip Steps 3 (Plan agent), 5 fan-out, 7 review fan-out. Still run tests + ship. |
| **single-layer** | Backend-only OR frontend-only. New endpoint OR new screen, but not both. Single repo. | One specialist agent (the matching architect). Plan agent for Step 3. Single review pass. |
| **cross-stack** | Backend + frontend in the same repo (full-stack monorepo, or a project with both `server/` and `client/`). | 2-3 specialists in parallel (backend architect + frontend architect, plus mongo-data-modeler if schema work). API contract handled inline by orchestrator (no broker needed for one repo). Full review fan-out. |
| **cross-repo** | Touches `*-api` AND one or more `*-mobile` / `*-app` / `*-desktop` / `*-cms` / `*-website` siblings. | All relevant architects in parallel + `api-contract-broker` + `mongo-data-modeler` if schema. Coordinated branches. Full review fan-out per repo. |

## Heuristics for classification

Ask yourself, in order:

1. **Does the feature add or change a route/endpoint?** If no, and no UI is being added, it's likely `micro`.
2. **Does it touch the data model (new collection, new field with index, migration)?** If yes, never `micro`. Mongo-data-modeler is in scope.
3. **Does it require BOTH a backend change and a frontend change?**
   - Same repo → `cross-stack`
   - Different repos → `cross-repo` (run cross-repo-detect.md to confirm siblings)
4. **Does the user phrase suggest scope?** ("just rename...", "add a tiny endpoint" → tend toward smaller. "Build a thing dashboard with create/edit/delete and notifications" → tend toward larger.) The phrase is a hint, not authority.

## Output

Append to the Feature Brief under a `## Scope` heading:

```markdown
## Scope

**Tier:** [micro | single-layer | cross-stack | cross-repo]

**Reasoning:** [1-2 sentences — what triggered this tier]

**Subagents to spawn:**
- [list — or "none" for micro]

**Skills to invoke:**
- [list — Plan, Explore, security-review, simplify, frontend-design, as applicable]
```

## Mode interaction

- **AUTOMATED mode:** classify silently, log to brief, proceed.
- **CHECKPOINT mode:** show the classification to the user before proceeding:

> Scope classification: **[tier]**
>
> Subagents to spawn: [list]
>
> A) Proceed
> B) Bump up a tier (more specialists, slower, more thorough)
> C) Bump down a tier (fewer specialists, faster, less coverage)

## Safety rules

- **Never skip tests** regardless of tier. The verify gate runs even for `micro`.
- **Never skip the security-review skill** if the feature touches: auth, sessions, tokens, file uploads, request/response bodies that flow to a database, anything user-supplied that's rendered as HTML.
- **Never skip the api-contract-broker** for cross-repo work. The broker is what justifies running multiple repos in one autofeature pass — without it you might as well run them separately.

## When to override the gate

User says "I want all the agents" or "go full force" → run as `cross-repo` regardless. The gate is a default, not a constraint.
