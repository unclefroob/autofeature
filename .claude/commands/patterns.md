---
name: patterns
description: |
  Coding-pattern census, canon, and enforcement — extract the repo's de-facto conventions into .autofeature/patterns.md and keep code converged on them.
  Audit mode (default): censuses the repo per dimension (layering, validation, auth/scoping, errors, data access, dates, logging, naming, tests), names each variant + the dominant one, flags drift with file:line evidence, and drafts .autofeature/patterns.md (canonical forms, banned variants, canonical-helper registry).
  Establish mode: resolves the open ⚖ decisions with you, finalizes patterns.md to ACTIVE, and wires enforcement (eslint ratchet rules, husky pre-push, optional CI, dead-scaffolding removal).
  Check mode: diff-scoped conformance of the current branch against patterns.md — also runs inside every /autofeature pre-ship review.
  Fix mode: converges one drift area via the autofeature pipeline (never edits inline).
  The main pipeline reads patterns.md in every run: specialists get it as PATTERNS_FILE (canon overrides repo sampling), and ship write-backs keep it current.
  Invoke as:
    /autofeature:patterns                     → audit current repo, draft the canon
    /autofeature:patterns establish           → resolve decisions, activate canon, wire enforcement
    /autofeature:patterns check               → check current branch diff against the canon
    /autofeature:patterns fix: <drift area>   → converge one drift area (hands off to /autofeature)
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Agent
  - Skill
  - mcp__trello__addCard
  - mcp__trello__getLists
---

# AutoFeature Patterns — Audit, Establish, Check, Fix

Extract the canon, don't invent one. The methodology (dimensions, canon selection rules, severity
rubric, patterns.md format, procedures) lives in `adapted/feature-patterns-audit.md` — this command
orchestrates it. Read that file in full before executing any mode.

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

## Step 1: Parse Arguments and Select Mode

Arguments = everything after `/autofeature:patterns` in the user's message.

**Mode detection:**
- Contains `establish` → **ESTABLISH MODE**
- Contains `check` → **CHECK MODE**
- Contains `fix:` → **FIX MODE** — extract the drift area after `fix:` as `FIX_SCOPE`
- Empty or `audit` → **AUDIT MODE**

**Patterns file** (all modes): `PATTERNS_FILE = .autofeature/patterns.md` if it exists. Record its
`status:` line (DRAFT / ACTIVE) — it changes behavior in every mode below.

---

## Step 2 (AUDIT MODE): Census the Repo

### 2a. Surface map

Spawn one lightweight agent — do **not** grep in main context:

```
Agent({
  description: "Patterns surface map",
  subagent_type: "Explore",
  model: "haiku",   // enumerate-only — orchestrator/model-tiers.md
  prompt: "Read the 'Audit procedure' step 1 of $AUTOFEATURE_HOME/adapted/feature-patterns-audit.md
  and produce the surface map for the repo at [pwd]:
  - src/ structure (top 2 levels) with file counts per layer directory
  - language (TS/JS/mixed), framework + validation + logging deps from package.json
  - tooling configs present and whether anything RUNS them: eslint/prettier/tsconfig strictness,
    husky/lint-staged, CI workflows (.github/workflows, gitlab-ci, etc.)
  - convention docs: CLAUDE.md, ARCHITECTURE.md, CONTRIBUTING.md, docs/ — one line on what each prescribes
  - era spread: for each main source directory, `git log --diff-filter=A --format=%ad --date=short`
    on a few files — report oldest/newest so census agents can sample across eras
  Paths + counts + one-liners only. Under 500 words."
})
```

### 2b. Census fan-out

Spawn the four census groups from the methodology (A: structure/handlers/naming/tests ·
B: validation/auth-scoping/security + helper hunt · C: errors/logging + dead scaffolding ·
D: data access/dates + perf) in a **single parallel message**:

```
[single message — four Agent calls]

Agent({
  description: "Patterns census — [group]",
  subagent_type: "general-purpose",
  model: "sonnet",   // census judgment — orchestrator/model-tiers.md
  prompt: "Read $AUTOFEATURE_HOME/adapted/feature-patterns-audit.md in full — dimensions table,
  canon selection rules, severity rubric. You own census group [A|B|C|D]: dimensions [list].
  Repo: [pwd]. Surface map: [2a result].

  Sample at least 12 files for your dimensions, spread across old / mid / new eras per the map.
  For each dimension return: variants found (rough counts — grep is fine), the dominant variant,
  and 2-3 file:line drift examples (the SAME concern done differently). Also return, where your
  group owns it: [group B] the canonical-helper hunt (helpers + their hand-rolled twins) and
  sampled security smells; [group C] dead scaffolding with zero-usage evidence; [group D] sampled
  performance smells.

  Census, not lint — strongest evidence only, no exhaustive listings. Read-only. Under 900 words."
})
```

### 2c. Merge, report, draft

1. Dedupe and merge the four census results; apply the **canon selection rules** from the
   methodology (dominant+sound → Canonical; dominant-but-deprecated → `⚖ PROPOSED` with a
   recommendation; no signal → Silent).
2. Write the audit report to `.autofeature/patterns-audit-[YYYY-MM-DD].md` per the report format.
   Set `AUDIT_PATH`.
3. Patterns file:
   - No `PATTERNS_FILE`, or status DRAFT → write/rewrite `.autofeature/patterns.md` as
     `status: DRAFT` per the template (≤150 lines; repo-specific decisions only).
   - Status ACTIVE → leave it untouched; put proposed changes in the report's
     `## Proposed amendments` section instead.
4. Anything 🔴-exploitable found during the census: surface it to the user **now**, first, above
   the pattern summary — don't bury a live auth hole under style findings. Recommend
   `security-review` for depth; this audit samples, it does not prove absence.

Show the user: the verdict paragraph, drift areas ranked by severity with counts, the
`⚖ PROPOSED` decisions list, and the two artifact paths.

### 2d. Offer next steps

```
Pattern census done: [N] dimensions with a clear canon, [M] ⚖ decisions open, [K] drift findings (🔴 [a] / 🟡 [b] / 🟢 [c]).

A) Establish now — resolve the ⚖ decisions and activate the canon     ← recommended
B) Converge a drift area — pick one to fix via /autofeature
C) Create a Trello card for this work
D) Audit only — stop here
```

**A** → proceed to Step 3. **B** → show drift areas; user picks; proceed to Step 5 with it as
`FIX_SCOPE`. **C** → `mcp__trello__getLists`, ask which list, card titled
`Patterns establish — [repo]` with the decisions list + `AUDIT_PATH`. Exit. **D** → print paths, exit.

---

## Step 3 (ESTABLISH MODE): Activate the Canon + Wire Enforcement

Establish is **always interactive** — never run it from inside a pipeline.

1. **Preconditions:** a DRAFT `patterns.md` + audit report exist. Missing → run Step 2 first, then
   continue. Already ACTIVE → confirm the user wants to re-open it (re-establish), else exit.
2. **Resolve decisions:** one batched AskUserQuestion covering every `⚖ PROPOSED` entry — each
   option list led by the audit's recommendation. No decision may be resolved silently.
3. **Finalize:** apply the decisions to `patterns.md`, set `status: ACTIVE`, date the Decisions log.
4. **Enforcement menu** (per methodology; ask once which items to apply, default = ratchet + husky):
   - eslint `no-restricted-*` rules for expressible Banned variants (messages point at patterns.md)
   - husky pre-push: lint + typecheck
   - CI workflow (lint + typecheck + tests) — recommend explicitly if the repo has **no CI at all**
   - dead-scaffolding removal — one commit each, fresh zero-usage grep evidence in the commit body
   - doc pointer from the repo's architecture/CLAUDE doc to patterns.md
5. **Branch + commits:** work on `patterns/establish-[YYYY-MM-DD]`, bisectable commits
   (canon file → ratchet → hooks/CI → deletions). Run the repo's lint + typecheck + test commands;
   paste real output. Present the diff summary. Push / PR only if the user says so.

---

## Step 4 (CHECK MODE): Diff Conformance

Requires `PATTERNS_FILE` — if absent, say so and suggest running the audit; exit cleanly (the
pipeline treats "no patterns file" as a silent skip, not an error).

Spawn one agent (read-only):

```
Agent({
  description: "Patterns check — branch diff",
  subagent_type: "general-purpose",
  model: "sonnet",   // orchestrator/model-tiers.md
  prompt: "Read [PATTERNS_FILE] and the 'Check procedure' in
  $AUTOFEATURE_HOME/adapted/feature-patterns-audit.md.
  Repo: [pwd]. Scope: files changed vs [base branch] (fall back to staged changes).

  Check ONLY the changed files, ONLY against: Canonical/Banned entries for the dimensions each
  file touches, the canonical-helper registry (did the diff re-derive a registered question?),
  and the auth-coverage invariant for any new/changed route.

  Return `Patterns check: CONFORMS` or a violations list — file:line · canon rule · fix — with
  🔴/🟡/🟢 severity, plus ⚖ areas the diff touches (informational). No edits. Under 600 words."
})
```

Report the result. DRAFT canon → prefix findings with `(advisory — canon is DRAFT)`. When invoked
from the pipeline (Step 9), findings join the Fix-First triage queue; standalone, offer
`/autofeature:patterns fix: <area>` for anything systemic.

---

## Step 5 (FIX MODE): Converge One Drift Area

Convergence is code-semantics work — hand it to the pipeline, never edit inline here.

1. If `FIX_SCOPE` names a drift area from a prior audit, pull its canon entry + file list from
   `AUDIT_PATH`; otherwise run a scoped Step 2 census for just that area first.
2. Compose the feature request:
   `[skip-product-review] Pattern convergence: migrate [drift area] to the canonical form in
   .autofeature/patterns.md ([canon entry]). Files: [list from audit]. No behavior changes;
   tests must stay green.`
3. `Skill({ skill: "autofeature:autofeature", args: "[composed request]" })`
4. One drift area per run — convergence PRs must stay reviewable.

---

## File Reference

| File | Purpose |
|------|---------|
| `adapted/feature-patterns-audit.md` | Methodology — dimensions, canon selection rules, severity rubric, patterns.md template, all four procedures, write-back protocol |
| `.autofeature/patterns.md` (per-repo, committed) | The canon — read by every pipeline run (PATTERNS_FILE), maintained by this command + ship write-backs |
| `.autofeature/patterns-audit-[date].md` (output) | Census report — drift findings, smells, verdict, decisions agenda |
