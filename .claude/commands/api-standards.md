---
name: api-standards
description: |
  Audit and refactor the Ritchies platform API to the route → controller → service → model standard, behavior-preserving.
  Finds controllers that inline business logic, extracts it into HTTP-agnostic service modules, reuses existing services, de-duplicates shared helpers, and adds service unit tests — WITHOUT changing any endpoint's behavior (the existing supertest suite is the safety net).
  Run it anytime the API drifts from the standard, on the whole API or one domain.
  Invoke as: /autofeature:api-standards [audit | <domain> | all] [mode:checkpoint|automated]
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

# AutoFeature — API Standards Refactor

A **behavior-preserving refactor** command for `ritchies-platform-api`. It does NOT add features or change any endpoint's request/response. It restructures existing code to the layering standard and returns the suite green. It reuses the standard autofeature ship/review machinery but is a refactor lens, not the feature builder.

## $AUTOFEATURE_HOME + the standard

Resolve `$AUTOFEATURE_HOME` as in `autofeature.md` (plugin root / `$CLAUDE_PLUGIN_ROOT` / `$HOME/dev/autofeature`). **Read `$AUTOFEATURE_HOME/ritchies/conventions.md` (API section) — it is the authoritative definition of the standard.** In short:

- **route → controller → service → model.** Business logic lives in `src/services/<domain>Service.ts` as an HTTP-agnostic **function module** (exported `async function`s, not a class — matching `chat/service.ts`, `auth/resolver.ts`, `auth/audit.ts`). Controllers stay thin: inline zod parse → resolve access → call service → shape response → `try/catch → next`. Routes + models unchanged.
- **Reuse before extract:** import existing services (`auth/resolver` `resolveAccess`/`can`/`summarizeAccess`, `auth/audit` `recordAudit`/`commitCatalogChange`, `chat/service`, `membership`, `reads`, `hub`, `authTokens`, `teamMembership`) rather than re-implementing.
- **Extract shared helpers** used by 2+ domains into `src/services/*` (e.g. a shared `format.ts`) instead of duplicating.
- `auth/` and `chat/` are grandfathered domain-module services — keep importing, do NOT relocate them.

## Target repo

```bash
CWD=$(pwd); PARENT=$(dirname "$CWD")
# Prefer the api repo whether invoked from it or a sibling
for c in "$CWD" "$PARENT/ritchies-platform-api" "$CWD/ritchies-platform-api"; do
  [ -f "$c/src/app.ts" ] && grep -q '"express"' "$c/package.json" 2>/dev/null && { API="$c"; break; }
done
[ -z "$API" ] && { echo "Run from ritchies-platform-api or its parent dir."; exit 1; }
```

## Iron Law for this command

**No refactor without a green baseline, and the same suite must be green after.** The existing `src/controllers/__tests__/*.routes.test.ts` supertest suite is the behavior contract — a passing→passing transition is the proof the refactor changed structure, not behavior. If the baseline is already red, STOP and report; do not refactor on top of failing tests.

## Pipeline

```
resolve target → baseline (typecheck + jest, must be green) → audit → scope-gate →
  [shared-helper extraction pass] → per-domain refactor (parallel where safe) →
  verify (typecheck + jest identical-green + build) → review → ship
```

## Step 1: Baseline

Run the suite and typecheck via the `test-runner` agent (or `npm test && npm run typecheck`). Record the green baseline (test count, e.g. "122/122"). If red, abort with the failures — the user fixes those first.

## Step 2: Audit

Scan `src/controllers/*.ts`, `src/services/` (may not exist yet), `src/auth/`, `src/chat/`. For each controller, flag violations against the standard:

- **Inline business logic / Mongoose queries** inside handlers (the tell: `.find(`/`.aggregate(`/`.create(` and multi-step domain rules in the controller rather than a service).
- **Missing `services/<domain>Service.ts`** for a domain that has non-trivial logic.
- **Duplicated helpers** across files (grep for repeated function bodies — the known offenders are `initialsOf` and `relativeTime`, currently in BOTH `announcementsController` and `chat/service.ts`).
- **Scoped-cap checks** done wrong (`requireCapability` used for scoped caps instead of in-service `resolveAccess`/`can`).
- **Missing service unit tests** (`src/services/__tests__/<domain>Service.test.ts`).

Produce a ranked report: per controller, the violations, the proposed `services/<domain>Service.ts` surface, which existing services it should reuse, and which helpers to hoist. Delegate the scan to an `Explore` agent to keep context clean; keep the report.

## Step 3: Scope

Parse the argument:
- `audit` (or no arg) → run Steps 1-2 and PRESENT the report only; ask whether to proceed and on which domains. Do not refactor yet.
- `<domain>` (e.g. `announcements`) → refactor just that domain.
- `all` → refactor every violating domain.

For `all`/multi-domain, confirm the ordered list via `AskUserQuestion` (domains ranked by violation severity) before touching code. Default mode is **checkpoint** (pause after each domain's verify); `mode:automated` runs straight through.

## Step 4: Shared-helper extraction pass (barrier — do this FIRST, once)

If the audit found helpers used by 2+ domains, extract them before per-domain work, because this pass edits multiple controllers coherently:
1. Create `src/services/<name>.ts` (e.g. `format.ts` exporting `initialsOf`, `relativeTime`) — a pure function module.
2. Update every importer to use it; delete the local copies.
3. Add `src/services/__tests__/<name>.test.ts` covering the pure helpers.
4. Verify: typecheck + jest still identical-green. Commit this as its own step.

## Step 5: Per-domain refactor (behavior-preserving)

For each in-scope domain, spawn `express-mongo-architect` (read `$AUTOFEATURE_HOME/agents/express-mongo-architect.md`, inline it, `subagent_type: general-purpose`, `model:` per `orchestrator/model-tiers.md`) with a **REFACTOR** job, feeding it the Brief-less job spec + the API section of `ritchies/conventions.md` + the audit entry for that domain. The job spec, verbatim intent:

> Refactor `<domain>` to route → controller → service → model, **without changing any endpoint's behavior**. Move ALL business logic + Mongoose queries from `controllers/<domain>Controller.ts` into a new `services/<domain>Service.ts` (HTTP-agnostic function module, matching `chat/service.ts` style). Leave the controller as: zod parse → resolve access → call the service → shape the SAME response → `try/catch → next`. Reuse existing services (`auth/resolver`, `auth/audit`, `chat/service`, …) — do not re-implement. Do NOT touch the route file, the model, or any response shape. Add `services/__tests__/<domain>Service.test.ts` unit tests. The existing `controllers/__tests__/<domain>.routes.test.ts` must pass UNCHANGED — if you need to edit it, you changed behavior; stop and flag it.

Domains that only need thinning can run in parallel (send the spawns in one message); if two domains share a helper not yet extracted, extract it in Step 4 first. One domain = one reviewable commit.

## Step 6: Verify — identical-green, honest

After each domain (checkpoint) or at the end (automated):
- `npm run typecheck` clean.
- `npm test` — the route-suite count must match the baseline and stay green (new service unit tests ADD to the count; no route test may change or fail). If a route test had to change, the refactor altered behavior — revert and reconsider.
- `npm run build` clean (that's the CI gate).
Report the before/after counts (e.g. "122/122 routes green before and after; +14 new service unit tests → 136/136"). Never claim green without running it.

## Step 7: Review + ship

Run the base pre-ship review (parallel review fan-out + `simplify` skill — this is a refactor, so `simplify`'s reuse/dead-code lens is especially apt). Then ship per `autofeature.md` Step 10: a branch like `refactor/api-standards-<domain-or-all>` off `main`, one commit per domain, a PR whose body lists each domain moved to the service layer, the reused/extracted services, and the before/after test counts. Save a checkpoint in `.autofeature/checkpoints/`.

## Guardrails

- **Pure structural change.** No new endpoints, no changed responses, no new capabilities. If the audit surfaces a genuine bug (not just a layering issue), report it separately — do not silently "fix" it inside a refactor.
- **Don't relocate `auth/` or `chat/`.** They are the reusable services; import from them.
- **Errors stay funneled** through the central `errorHandler` (last in `app.ts`); services `throw` typed errors, controllers `next(error)`.
- **Scope discipline:** a refactor that balloons past the chosen domains is a smell — stop and re-scope with the user.
