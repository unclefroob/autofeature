---
name: backend-api-skills
description: |
  Get a backend API into the correct shape — create it right, or audit and refactor what's there.
  Enforces route → controller → service → model with HTTP-agnostic services, reuse-before-extract, resource-scoped authorization, and a two-tier test split. Stack-adaptive: Express+Mongoose is the reference, but it detects the repo's actual framework/ORM/validator/test runner and maps the layers onto it.
  Modes: audit (report only) · create (greenfield skeleton or one domain slice) · fix (behaviour-preserving refactor, identical-green gated).
  Invoke as: /autofeature:backend-api-skills [audit | create [<domain>] | fix [<domain>|all]] [mode:checkpoint|automated]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Agent
  - Workflow
  - Skill
  - TaskCreate
  - TaskUpdate
  - Monitor
---

# AutoFeature — Backend API Skills

One command for the shape of a backend API. It **audits** an existing backend against the standard, **creates** new backend code that satisfies it from the first commit, or **fixes** drift with a behaviour-preserving refactor.

This command supersedes `/autofeature:api-standards`, which is now a thin alias that forwards here with the Ritchies profile pre-selected. The refactor pipeline that used to live there is `fix` mode below, generalised past the one repo it was hard-wired to.

## $AUTOFEATURE_HOME + the standard

Resolve `$AUTOFEATURE_HOME` as in `autofeature.md` (plugin root / `$CLAUDE_PLUGIN_ROOT` / `$HOME/dev/autofeature`). Read, in this order:

1. **`$AUTOFEATURE_HOME/backend/api-standard.md`** — the layering law, the layer contracts, the violation catalogue (V1–V14), the verify gates. Authoritative.
2. **`$AUTOFEATURE_HOME/backend/stack-profiles.md`** — detection, layer mapping per stack, command discovery.
3. **`$AUTOFEATURE_HOME/backend/scaffold.md`** — only in `create` mode.
4. **`$AUTOFEATURE_HOME/ritchies/conventions.md`** (API section) — only when the target is a Ritchies repo. It **adds** repo-specific rules (four-pillar RBAC, `catalog:sync`, grandfathered `auth/` + `chat/`); it does not replace the standard.
5. **`<repo>/.autofeature/patterns.md`** if present — the repo's own canon **outranks** the standard on style, naming and helper choice.

Do not paraphrase the standard from memory. Read the file — it changes.

## Step 0: Target + profile

Resolve the target repo (argument path, else cwd, else a sibling that looks like a backend). Then run the detection block from `stack-profiles.md` and **state the profile back to the user in one line** before doing anything else.

If detection is ambiguous — two frameworks, no recognisable data layer, no test runner — STOP and ask. A wrong profile produces confidently wrong code in every file it touches.

Then discover the four gate commands (`typecheck`, `test`, `build`, `lint`) from `scripts` + the CI workflow. CI wins where they disagree; it is what actually blocks a merge.

## Step 1: Dispatch

Parse the first token:

| Arg | Mode | Behaviour |
|-----|------|-----------|
| `audit`, or none | **audit** | Steps 2–3. Report only. Writes nothing. |
| `create` | **create** | Greenfield skeleton (no app entrypoint found) or domain slice. |
| `create <domain>` | **create** | One domain slice into the existing backend. |
| `fix` / `fix <domain>` / `fix all` | **fix** | Behaviour-preserving refactor of drift. |

With no argument, run the audit and then **offer** the fix — never refactor on an unqualified invocation. Default mode is **checkpoint** (pause after each domain's verify); `mode:automated` runs straight through.

---

# AUDIT MODE

## Step 2: Baseline (audit is read-only, but the baseline is the headline)

Run `typecheck` and `test`. Record the result. **A red baseline is the finding that outranks every layering finding** — report it first and do not bury it under structural advice. It also means `fix` mode cannot run.

## Step 3: Scan

Delegate the scan to an `Explore` agent (model per `orchestrator/model-tiers.md`) to keep the orchestrator's context clean; keep only the report. Feed it the detected profile and the violation catalogue, and have it check every domain against V1–V14 using the **stack-appropriate tells** from `stack-profiles.md` — `.find(`/`.aggregate(` for Mongoose, `prisma.` for Prisma, `getRepository(` for TypeORM, raw SQL for query builders.

Produce a ranked report. Per domain:

- the violations found, each with `file:line` evidence and its catalogue ID
- the proposed service surface (the functions that should exist)
- which **existing** services it should reuse instead of re-implementing
- which helpers should be hoisted to shared modules
- test-tier gap: does it have service unit tests, or only route tests?

Then a repo-level section: duplicated helpers across domains (V5), anything re-deriving permissions (V11), and the test-tier balance.

**Rank by risk, not by clause order.** 🔴 rows first — a scoped-auth violation (V3) matters more than ten duplicated helpers. Order the fix backlog by which violations can actually hurt someone.

**Separate bugs from violations.** If the scan finds a genuine defect — a rule that is *wrong*, not just misplaced — report it in its own section. Do not fold it into the refactor backlog; see the Guardrails.

Present the report. In audit mode, stop here and ask whether to proceed to `fix`, and on which domains.

---

# CREATE MODE

## Step 4: Pick the blueprint

No app entrypoint → **Blueprint A** (greenfield skeleton) from `scaffold.md`, then Blueprint B for the first domain. Entrypoint exists → **Blueprint B** only.

For greenfield, confirm the stack via `AskUserQuestion` before generating — you are choosing the repo's framework, data layer, validator and test runner for years, and inferring that from an empty directory is a guess. For a domain slice into an existing repo, the profile is already detected; do not re-ask.

## Step 5: Reuse pass (before any code)

Grep the existing service layer for logic the new domain needs — access resolution, audit recording, formatting, membership, visibility. **This runs before generation, not after.** A new domain that re-implements the access resolver is a 🔴 on arrival, and the cheapest moment to prevent it is before the file exists.

## Step 6: Generate

Spawn the stack's architect specialist — `express-mongo-architect` for Node (read `$AUTOFEATURE_HOME/agents/express-mongo-architect.md`, inline it, `subagent_type: general-purpose`, `model:` per `orchestrator/model-tiers.md`). Pass it: the detected profile, the relevant blueprint from `scaffold.md`, the reuse-pass findings, the repo's `patterns.md` if present, and the gate commands.

Generate in dependency order — model → service → controller → route → mount → service tests → route tests. Independent domains may run in parallel (send the spawns in one message); a shared helper two domains both need is extracted **first**, in its own step, because that pass edits multiple files coherently.

Greenfield: get the spine green with its own trivial tests **before** the first domain slice. Do not scaffold six domains before running anything.

---

# FIX MODE

The behaviour-preserving refactor, inherited from `api-standards` and generalised.

## Iron Law for this mode

**No refactor without a green baseline, and the same suite must be green after.** The pre-existing route/integration suite is the behaviour contract; a passing→passing transition is the proof the change moved structure, not behaviour. If the baseline is red, STOP and report — the user fixes that first. Refactoring on top of failing tests destroys the only evidence you had.

## Step 7: Scope

`fix <domain>` → that domain. `fix all` → every violating domain, ordered by severity from the audit, confirmed via `AskUserQuestion` before touching code.

## Step 8: Shared-helper extraction (barrier — once, first)

If the audit found helpers used by 2+ domains (V5), extract them before any per-domain work, because this pass edits multiple domains coherently:

1. Create the shared module in the service layer.
2. Update every importer; delete the local copies.
3. Add unit tests for the pure helpers.
4. Verify: typecheck + tests still identical-green. Commit as its own step.

## Step 9: Per-domain refactor

For each in-scope domain, spawn the stack's architect with a **REFACTOR** job. The job spec, verbatim intent:

> Refactor `<domain>` to route → controller → service → model, **without changing any endpoint's behaviour**. Move ALL business logic and data-layer queries from the controller into the domain's service module (HTTP-agnostic function module, matching this repo's existing service style). Leave the controller as: parse → resolve access → call the service → shape the SAME response → `try/catch → next`. Reuse existing services — do not re-implement. Do NOT touch the route file, the model, or any response shape. Add service unit tests. The existing route tests must pass UNCHANGED — **if you need to edit one, you changed behaviour; stop and flag it.**

Domains that only need thinning can run in parallel. One domain = one reviewable commit.

## Step 10: Verify — identical-green, honest

After each domain (checkpoint) or at the end (automated):

- `typecheck` clean.
- `test` — pre-existing test count matches the baseline and stays green. New service unit tests ADD to the count. **No pre-existing route test may change or fail.** If one had to change, the refactor altered behaviour — revert and reconsider.
- `build` clean.

Report before/after counts explicitly (e.g. "122/122 route tests green before and after; +14 new service unit tests → 136/136"). **Never claim green without running it.** If a gate could not be run, say which and why — an unrun gate is not a passed gate.

---

# ALL MODES

## Step 11: Review + ship

Run the base pre-ship review (parallel review fan-out + the `simplify` skill — for `fix` mode its reuse/dead-code lens is especially apt). If `.autofeature/patterns.md` exists, run `/autofeature:patterns check` on the diff.

Ship per `autofeature.md` Step 10: a branch (`refactor/backend-api-<domain-or-all>` for fix, `feat/backend-<domain>` for create) off the default branch, one commit per domain, and a PR body listing the domains touched, the services reused or extracted, and the before/after gate results. Save a checkpoint in `.autofeature/checkpoints/`.

## Guardrails

- **Structure only.** `fix` adds no endpoints, changes no responses, adds no capabilities. `create` adds exactly what was asked for.
- **A bug found is reported, never silently fixed inside a refactor.** Mixing a behaviour fix into a structural change makes the diff unreviewable and destroys the identical-behaviour proof — the one thing that made the refactor safe to merge.
- **Don't relocate grandfathered domain modules.** Pre-existing service code organised as domain folders keeps working where it is; import from it. Relocation is behaviour risk with no behavioural payoff.
- **Don't impose directory names.** `services/fooService.ts`, `foo/service.ts` and `domain/foo.ts` all satisfy the law. Match the repo.
- **Repo canon outranks this standard** on style, naming and helper choice. Where they genuinely conflict, surface it rather than silently picking.
- **Scope discipline.** A refactor that balloons past the agreed domains is a smell — stop and re-scope with the user.
- **Capability keys are a deploy step.** If the repo has a capability catalogue or permission registry, registering a new key in code is not enough; name the per-environment sync step in the PR body or the feature silently fails after deploy.
