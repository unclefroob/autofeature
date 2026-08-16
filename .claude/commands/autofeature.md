---
name: autofeature
description: |
  Build an entire feature end-to-end from a single prompt.
  Pipeline: interrogate → scope-gate → plan → branch → implement (parallel specialists) → test → review → ship.
  Supports automated (fire-and-forget) and checkpoint (pause-and-approve) modes.
  Stack: Node.js + Express + Mongoose + MongoDB, React, React Native (Expo or bare). Cross-repo coordination for paired *-api / *-mobile / *-desktop projects.
  Invoke as: /autofeature [mode:] <feature description>
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
  - Agent
  - Workflow
  - Skill
  - TaskCreate
  - TaskUpdate
  - Monitor
  - mcp__trello__get_card_details
  - mcp__trello__get_card_checklists
  - mcp__trello__add_comment_to_card
---

# AutoFeature — End-to-End Feature Orchestrator

## Builder Principles (always active)

**Boil the Lake:** The marginal cost of completeness is near-zero with AI. Always choose the complete implementation over the 90% shortcut. Tests, edge cases, error paths — these are lakes, not oceans.

**Search Before Building:** Before designing any solution involving unfamiliar patterns, runtime capabilities, or infrastructure — check if the framework already has a built-in. The cost of checking is near-zero.

**User Sovereignty:** Recommendations are presented, not applied. When implementation requires a decision that changes what was asked for — present it, explain why, and ask. Never act unilaterally.

**Missing Information Is a Design Task:** "We don't hold that data" is the beginning of the work, not the end of it. When a rule, check or feature cannot be built because a fact is not captured, design the capture — which existing channel carries it, under what key and type, who answers it and when. Recording the absence and moving on is the failure mode; it produces a backlog of things nobody can schedule, and it hides how close the work actually is.

Check the existing channels first — Search Before Building applies, and most facts already have somewhere to live. Only when none fits do you specify the new record, and then you specify it concretely enough to build: the model, the field, the screen.

The one legitimate stop is a fact that cannot exist for anyone — a judgement, a state of mind, something outside any system's reach. "Was this agreement genuine" is that. "We never added the field" is not, and filing the second under the first is how a finite backlog stops being finite.

---

## Model Efficiency (always active)

Spawned agents and Workflow phases inherit the session model unless given an explicit `model:` — so on
Opus the whole fan-out is Opus, the main cost driver. **Every spawn in this pipeline carries a
`model:` per `$AUTOFEATURE_HOME/orchestrator/model-tiers.md`** (active profile: BALANCED — Sonnet
workhorse, Haiku for mechanical test-runner/maps, Opus only for the few highest-judgment steps).
Skills and this orchestrator loop follow the session model, so for the cheapest runs invoke
`/autofeature` on Sonnet; the deep-reasoning steps still self-elevate to Opus via their pins. A run may
pass `model: economy|balanced|quality` (or `model: opus|sonnet|haiku`) to shift the whole fleet.

---

## $AUTOFEATURE_HOME

All methodology + agent + orchestrator files live under one root. They **ship with the plugin**, so
the commands resolve them from the plugin's own root (`$CLAUDE_PLUGIN_ROOT`, set automatically for an
installed plugin); an explicit `$AUTOFEATURE_HOME` or a local dev clone also works:

```bash
# Files ship with the plugin — prefer its root; fall back to an explicit home or dev clone.
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

Resolved layout:
- `$AUTOFEATURE_HOME/adapted/` — methodology files (interrogation, plan, investigate, review, ship, design check, devex check)
- `$AUTOFEATURE_HOME/agents/` — specialist subagent prompts
- `$AUTOFEATURE_HOME/orchestrator/` — scope-gate, cross-repo-detect, skill-wiring
- `$AUTOFEATURE_HOME/source/` — gstack reference (do not edit)

If no candidate resolves (none contains `adapted/`), abort with: `AutoFeature methodology files not found. They ship with the plugin — reinstall it, or for a dev clone set AUTOFEATURE_HOME=/path/to/autofeature.`

---

## Pipeline

```
resume? → [trello?] → interrogate → scope-gate → product-review (pre-build) → plan → branch → cross-repo-coord →
  implement (parallel specialists) → verify (test-runner) → review (parallel + skills) → ship → test-manifest
```

---

## Step 0: Resume or Start

**Check for an existing checkpoint first:**

```bash
ls .autofeature/checkpoints/ 2>/dev/null | sort -r | head -5
```

If checkpoints exist from a previous autofeature run on this branch, show them:
> Found previous autofeature checkpoint(s) for this branch:
> - [timestamp] [title] ([status])
>
> A) Resume from most recent checkpoint
> B) Start fresh

If resuming: read `.autofeature/checkpoints/[latest].md`, reconstruct context, and jump to the phase indicated in the checkpoint's `next_step` field. Also read `.autofeature/coordination.md` if present (for cross-repo runs) — siblings, branch names, ship order.

---

## Step 0.5: Trello Card Detection (if URL present)

**Scan the feature request for a Trello card URL:**

Look for `trello.com/c/` anywhere in the ARGUMENTS string.

**If found:**

Read and follow `$AUTOFEATURE_HOME/orchestrator/trello-scope.md`. That file handles:
- MCP availability check (graceful fallback if not connected)
- Fetching card name, description, labels, checklists via `mcp__trello__*` tools
- Generating a technical scope analysis
- Showing a checkpoint preview with 4 options (post + use card / post + use prompt / skip post + use card / skip entirely)
- Posting the scope comment to the Trello card (if approved)

On return, `FEATURE_REQUEST` is set to either the card content or the original prompt depending on the user's choice. Use `FEATURE_REQUEST` as the canonical input for all downstream steps (Step 1 mode detection, Step 2 interrogation, Step 3 scope gate).

If `trello-scope.md` set `TRELLO_SCOPE_TIER`, skip the scope gate classification in Step 3 and use that value directly.

**If not found:** Skip this step entirely. `FEATURE_REQUEST` = original ARGUMENTS.

---

## Step 1: Parse Input and Select Mode

Feature request = everything after `/autofeature` in the user's message.

**Detect mode from prompt:**
- Contains "auto" or "automated" → **AUTOMATED**
- Contains "checkpoint" or "step by step" or "pause" → **CHECKPOINT**
- Neither → ask:

> Which mode?
>
> **A) Automated** — run straight through, make all decisions using best judgment, produce a PR. You review at the end.
>
> **B) Checkpoint** — pause for approval after: scope gate, feature spec, implementation plan, implementation, before shipping.

**In AUTOMATED mode**, apply these 6 decision principles for all choices:
1. Choose completeness — cover more edge cases, not fewer
2. Boil lakes — fix everything in blast radius of the change
3. Pragmatic — pick the cleaner option; don't deliberate
4. DRY — reuse what exists
5. Explicit over clever — readable code beats smart code
6. Bias toward action — ship, don't debate

Only interrupt for a **User Challenge**: when implementation would need to fundamentally contradict the stated feature request.

**Model tier:** scan the request for a `model:` override (`economy|balanced|quality` or `opus|sonnet|haiku`); absent → BALANCED. Apply it per `$AUTOFEATURE_HOME/orchestrator/model-tiers.md` to every agent spawned below.

**Create pipeline tasks via TaskCreate:**

```
Task 1: Feature Interrogation        → "Interrogating feature requirements"
Task 2: Scope Classification         → "Classifying feature scope"
Task 3: Cross-Repo Detection         → "Detecting sibling repos"
Task 4: Product Review (pre-build)   → "Reviewing product fit — gaps & flows"
Task 5: Technical Planning           → "Planning implementation"
Task 6: Create Feature Branch(es)    → "Creating feature branch(es)"
Task 7: Implementation (parallel)    → "Implementing feature"
Task 8: Verification Gate            → "Running test suite"
Task 9: Pre-Ship Review              → "Running pre-ship review"
Task 10: Ship                        → "Shipping — commits, push, PR(s)"
```

Set up `addBlockedBy` dependencies so each task blocks on the previous one.

> Task 4 (Product Review) is **skipped for `micro` tier** and when the feature request carries a
> `[skip-product-review]` marker — mark it completed immediately in those cases (see Step 4.5).

---

## Step 2: Feature Interrogation

Read and follow `$AUTOFEATURE_HOME/adapted/feature-interrogation.md`.

**Goal:** Produce a Feature Brief at `.autofeature/designs/[slug]-[date].md`.

**Slug:** Convert feature request to kebab-case, max 5 words.
Example: "add user profile photo upload" → `user-profile-photo-upload`

### 2a. Context gathering — delegate to Explore subagent

Don't grep + glob in main context. Spawn Explore:

```
Agent({
  description: "Autofeature context scan",
  subagent_type: "Explore",
  model: "sonnet",            // build-context — orchestrator/model-tiers.md
  prompt: "Gather context for autofeature run.

  Feature request: [verbatim user request]
  Repo: [pwd]

  Find and return file paths (no excerpts unless load-bearing):
  1. Project conventions: CLAUDE.md, README, lint config, formatter config, tsconfig
  2. Tech stack signals: package.json deps that match (express|fastify|@nestjs|hapi|koa) | (react|react-native) | mongoose, ios/ folder, android/ folder
  3. Test setup: test command from package.json, jest/vitest/playwright configs, existing test directories
  4. E2E setup: @playwright/test in deps, playwright.config files, e2e/ directories
  5. Existing code that might already partially solve this feature — grep terms from the feature request
  6. The auth pattern used in the repo (middleware name, where it's applied)
  7. The data model file(s) for affected entities
  8. The API client / data-fetching layer used by frontends in this repo

  Return as a structured list with one-line role per file. Under 1500 words."
})
```

Save the result to the Feature Brief under `## Context` and use it for downstream steps. The orchestrator does not need to redo any of this.

### 2b. Stack detection (record in brief)

From the Explore result, record:
- `STACK = NODE_BACKEND | REACT | REACT_NATIVE` (one or more)
- `HAS_E2E = true | false`, `E2E_TOOL = playwright | cypress | none`
- `TEST_CMD = [from package.json]`

### 2c. Interrogation questions

**AUTOMATED:** Answer interrogation questions from the Explore context. State assumptions in Feature Brief.

**CHECKPOINT:** Ask the 6 builder questions interactively. Then:

> Feature Brief:
> [content]
>
> A) Approve — proceed to scope classification
> B) Revise — [tell me what to change]
> C) Abort

---

## Step 3: Scope Gate

Read and follow `$AUTOFEATURE_HOME/orchestrator/scope-gate.md`.

Classify into: `micro | single-layer | cross-stack | cross-repo`. Append the `## Scope` section to the Feature Brief.

**CHECKPOINT** mode: show the classification before proceeding (per scope-gate.md).

The scope tier determines:
- Which specialist agents are spawned in Step 6
- Whether `api-contract-broker` runs
- Whether `mongo-data-modeler` runs
- Whether `simplify` skill runs in Step 8

---

## Step 4: Cross-Repo Detection

If scope tier ∈ {`cross-repo`} OR the feature description mentions multiple surfaces:

Read and follow `$AUTOFEATURE_HOME/orchestrator/cross-repo-detect.md`.

If siblings are found and selected:
- Write `.autofeature/coordination.md` listing siblings, branch name, ship order
- All subsequent file paths in the run are relative to a "current target repo" that the orchestrator iterates over

For non-cross-repo tiers: skip this step.

---

## Step 4.5: Product Review (Pre-Build)

A CEO / PM / flow-walker pass over the **product** before any code is written: does this feature, in
the context of what already exists, make product sense — and does it leave gaps or broken flows?
Runs the multi-agent product-review **Workflow**; its findings feed the plan.

**Skip this step entirely if:**
- Scope tier is `micro` (a one-file change doesn't warrant a product panel), **or**
- `FEATURE_REQUEST` contains the `[skip-product-review]` marker (this run is itself a fix the
  product review generated, or an SEO/narrow fix that opted out — prevents recursion).

Mark Task 4 completed and proceed to Step 5 in those cases.

**Otherwise:**

Read and follow `$AUTOFEATURE_HOME/adapted/feature-product-review.md` in **`feature` mode**. Invoke
the **Workflow** tool with the script it contains:

```
Workflow({
  script: <the script from feature-product-review.md>,
  args: {
    repo:         "[pwd]",
    mode:         "feature",
    featureBrief: ".autofeature/designs/[slug]-[date].md",
    siteUrl:      "[SITE_URL if a deployed site is known, else '']"
  }
})
```

The workflow maps the existing product + the proposed feature, fans out the three lenses, verifies
high-severity gap/flow claims against the real code, and returns a synthesis. Append it to the
Feature Brief under `## Product Review` (summary, prioritized findings, recommended next features) —
the Plan subagent in Step 5 reads this section.

### Mode handling

**AUTOMATED:** Fold 🔴/🟡 **in-scope** gaps into the work (Step 5's plan picks them up from the
brief). Surface a **User Challenge** (emergency stop) *only* when a finding means the feature **as
requested** would ship a broken core flow or a half-capability — i.e. building it as-asked
contradicts the user's actual goal. Otherwise proceed; never silently expand scope beyond closing
in-scope gaps.

**CHECKPOINT:**
> Product review (pre-build) found [C critical, H high] gaps/flow issues for this feature:
>
> [summary + top findings]
>
> A) Proceed as planned — log the rest as fast-follows
> B) Expand this feature's scope to close the 🔴/🟡 in-scope gaps before building
> C) Revise the feature request — [tell me what to change]
> D) Abort

---

## Step 5: Technical Planning

### 5a. Delegate the plan to the Plan subagent

```
Agent({
  description: "Autofeature technical plan",
  subagent_type: "Plan",
  model: "sonnet",            // cross-repo → "opus" (orchestrator/model-tiers.md)
  prompt: "Read the Feature Brief at .autofeature/designs/[slug]-[date].md.
  Also read $AUTOFEATURE_HOME/adapted/feature-plan.md for methodology.

  If the brief has a '## Product Review' section (pre-build gaps & flow issues), fold its in-scope
  🔴/🟡 gaps into the plan and note any deferred to fast-follow. These are product gaps to close,
  not engineering nits.

  Produce the Implementation Plan section including:
  - Step 0: Scope Challenge
  - Step 1: Architecture Review (stack-specific based on STACK in brief)
  - Step 2: Code Quality Review
  - Step 3: Unit Test Plan with shadow paths (happy/nil/empty/error)
  - Step 3b: Error & Rescue Map
  - Step 3c: Shadow Path Testing
  - Step 3d: Observability Checklist
  - Step 4: Performance Review

  Append to the Feature Brief under '## Implementation Plan'. Return a 5-line summary of decisions made and any User Challenges surfaced."
})
```

### 5b. Parallel stack-specialist designs (skip for `micro` tier)

After the Plan subagent returns, fan out the architects in parallel based on STACK and scope tier:

For each applicable architect, compose a spawn prompt:
1. Read `$AUTOFEATURE_HOME/agents/<agent-name>.md`
2. Append: feature brief path, mode=`design`, repo path
3. Spawn via `Agent` (subagent_type=`general-purpose`, **`model: "sonnet"`** per `orchestrator/model-tiers.md`)

**Send all applicable Agent calls in a SINGLE message for true parallelism.**

| STACK | Tier | Agents spawned |
|-------|------|----------------|
| NODE_BACKEND only | single-layer | express-mongo-architect (+ mongo-data-modeler if schema work) |
| REACT only | single-layer | react-architect |
| REACT_NATIVE only | single-layer | react-native-architect |
| NODE_BACKEND + (REACT or REACT_NATIVE) | cross-stack | both architects + mongo-data-modeler if schema |
| Cross-repo | cross-repo | every applicable architect (one per repo) + mongo-data-modeler |

Each architect appends its plan section to the Feature Brief.

### 5c. Contract reconciliation (cross-stack and cross-repo only)

For `cross-stack` (single repo): orchestrator reads each architect's section, identifies request/response shape mismatches, resolves inline.

For `cross-repo`: spawn `api-contract-broker`:

```
Read $AUTOFEATURE_HOME/agents/api-contract-broker.md
Agent({
  description: "API contract reconciliation",
  subagent_type: "general-purpose",
  model: "sonnet",            // orchestrator/model-tiers.md
  prompt: "[api-contract-broker.md content]

  Job: Reconcile contracts across the architects' plans in .autofeature/designs/[slug]-[date].md.
  Repos involved: [list from coordination.md]
  Return a Contract Distribution block per repo."
})
```

The broker output is added to the Feature Brief and used to brief the implementers in Step 7.

### 5d. Mode handling

**AUTOMATED:** Apply 6 decision principles. Only surface User Challenges.

**CHECKPOINT:**
> Implementation Plan + Specialist Designs:
> [summary of architect outputs]
>
> A) Approve — proceed to branch creation and implementation
> B) Revise — [tell me what to change]
> C) Abort

---

## Step 6: Create Feature Branch(es)

For the primary repo:

```bash
BASE=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}' || echo "main")
git fetch origin $BASE
git checkout $BASE
git pull origin $BASE
BRANCH="feature/[slug]"
git checkout -b "$BRANCH"
echo "Branch created: $BRANCH"
```

For cross-repo coordination, repeat in each sibling repo listed in `.autofeature/coordination.md`. Skip dirty siblings (already filtered in Step 4) — surface them again if any are dirty:

> Sibling [name] is dirty. Skipping — handle separately. Continuing with: [remaining list]

---

## Step 7: Implementation

### 7a. Pre-implementation: frontend-design skill (when applicable)

If the plan creates new UI components/pages AND the `frontend-design` skill is available AND scope tier ≠ `micro`:

```
Skill({
  skill: "frontend-design:frontend-design",
  args: "Generate the [component(s) named in plan] for [feature description].
  Style system: [tailwind | styled-components | css-modules | StyleSheet — match repo].
  Save to: [target paths from plan]."
})
```

Skip for: data-display tweaks, admin-only screens, single-component edits.

### 7b. Specialist implementers (parallel where possible)

For each architect that produced a design, spawn the same agent in `implement` mode. Send all in a single message for parallelism:

```
[single message] — all architects at model: "sonnet" (orchestrator/model-tiers.md)
Agent({ description: "Backend impl", model: "sonnet", prompt: "[express-mongo-architect.md] + mode=implement + brief path + branch" })
Agent({ description: "Web impl",     model: "sonnet", prompt: "[react-architect.md] + mode=implement + brief path + branch" })
Agent({ description: "Mobile impl",  model: "sonnet", prompt: "[react-native-architect.md] + mode=implement + brief path + branch" })
```

Each implementer follows TDD per `feature-plan.md` (RED → verify RED → GREEN → verify GREEN → REFACTOR).

For `micro` scope: orchestrator implements directly in main context — no fan-out.

### 7c. E2E test writing (Playwright — React/web only)

If STACK includes REACT and HAS_E2E=true (or user opted into installing Playwright):

The react-architect writes a Playwright test for the golden path as part of its `implement` mode. See `feature-interrogation.md` for setup details.

For React Native, skip — manual platform testing is the verification.

### 7d. Save checkpoint after implementation

```bash
mkdir -p .autofeature/checkpoints
cat > .autofeature/checkpoints/$(date +%Y%m%d-%H%M%S)-impl.md << 'EOF'
---
status: post-implementation
branch: BRANCH_NAME
next_step: verify
scope: [tier]
---

## Working on: [feature name]

### Implemented (per architect)
- Backend: [files from express-mongo-architect summary]
- Web:     [files from react-architect summary]
- Mobile:  [files from react-native-architect summary]

### Open contract questions
[from architect summaries]

### Remaining
1. Full test suite run (test-runner)
2. Pre-ship review (parallel + security-review + simplify)
3. Ship
EOF
```

**CHECKPOINT:**
> Implementation complete:
>
> [per-architect summary]
>
> A) Proceed to verification
> B) Revise — [tell me what to change]

---

## Step 8: Verification Gate

**The Iron Law: no completion claim without fresh evidence.**

### 8a. Unit + integration tests via test-runner agent

```
Read $AUTOFEATURE_HOME/agents/test-runner.md
Agent({
  description: "Run unit/integration suite",
  subagent_type: "general-purpose",
  model: "haiku",             // mechanical — orchestrator/model-tiers.md
  prompt: "[test-runner.md content]

  Job: Run the unit and integration test suite for repo at [pwd].
  Mode: unit (auto-detect command).
  Return a structured summary."
})
```

The test-runner returns a <2KB summary. The orchestrator NEVER ingests full test logs.

### 8b. Playwright E2E (if HAS_E2E=true)

```
Agent({
  description: "Run Playwright suite",
  subagent_type: "general-purpose",
  model: "haiku",             // mechanical — orchestrator/model-tiers.md
  prompt: "[test-runner.md content]

  Job: Run the Playwright E2E suite for repo at [pwd].
  Mode: e2e.
  Return a structured summary."
})
```

### 8c. Cross-repo: run test-runner per repo

For coordinated runs, run unit tests in EACH repo. Track results in `.autofeature/coordination.md`. Do NOT proceed to ship if any repo fails.

### 8d. Failure handling

If tests fail: invoke `$AUTOFEATURE_HOME/adapted/feature-investigate.md` before attempting a fix. Iron Law: no fix without root cause. Write a regression test that fails without the fix and passes with it. Re-run test-runner after fixing.

If 3 hypotheses fail during debugging → Emergency Stop, escalate to user.

**CHECKPOINT:**
> Verification:
>
> Unit:  [test-runner summary]
> E2E:   [test-runner summary or "n/a"]
>
> A) Proceed to review
> B) Add more tests for [scenario]
> C) Stop here — I'll review manually

---

## Step 9: Pre-Ship Review

Read `$AUTOFEATURE_HOME/adapted/feature-review.md`.

### 9a. Parallel review fan-out

Spawn these as simultaneous agents in a SINGLE message (each at **`model: "sonnet"`** per `orchestrator/model-tiers.md`):

1. **Critical pass** — security, race conditions, SQL/NoSQL injection, unhandled enums, mass-assignment
2. **Informational pass** — type safety, async issues, completeness gaps
3. **Testing pass** — missing negative paths, edge cases, test isolation

If frontend files changed, also launch:
4. **Design pass** — reads `feature-design-check.md`, reviews changed UI files

If API/CLI/SDK/docs changed, also launch:
5. **DX pass** — reads `feature-devex-check.md`, reviews changed interface files

Each agent reads `feature-review-checklist.md` + the changed files, returns structured findings (CRITICAL / INFO / TESTING) with `file:line` references.

### 9b. security-review skill

While the parallel agents run, invoke `security-review` if the feature touches ANY of:
- Auth, sessions, tokens, refresh flow
- File uploads
- User-supplied content rendered as HTML/markdown
- DB queries built from request input
- CORS/CSP/security headers
- Secret/env var loading
- External data deserialization

```
Skill({ skill: "security-review", args: "" })
```

Add findings to the same triage queue.

### 9c. Fix-First triage

Collect all findings:
- **AUTO-FIX** — mechanical issues (typos, missing await, wrong type, missing null check, unused import)
- **ASK** — judgment calls (architecture, naming, abstraction)

Apply auto-fixes immediately. Batch ASK items into a single AskUserQuestion call (CHECKPOINT) or apply 6 decision principles (AUTOMATED).

After fixes: re-run test-runner.

### 9d. simplify skill (skip for `micro` tier)

```
Skill({ skill: "simplify", args: "" })
```

If simplify makes changes, run test-runner once more.

---

## Step 10: Ship

Read `$AUTOFEATURE_HOME/adapted/feature-ship.md`.

### 10a. Single repo

1. Pre-flight (branch check, diff summary)
2. Merge base branch (resolve any conflicts → emergency stop if not auto-resolvable)
3. Final test-runner run — paste summary, do not claim passing without evidence
4. Bisectable commits (one logical change per commit)
5. Push
6. Create PR via `gh pr create`

### 10b. Cross-repo coordinated ship

Order matters because frontends depend on the API:

```
1. Ship API repo first
2. Wait for CI green on API repo (test-runner against the deployed branch if env exists; otherwise just confirm CI passes)
3. Ship CMS / website (any backend-adjacent web)
4. Ship mobile / desktop last
```

Update `.autofeature/coordination.md` per repo as each ships.

### 10c. PR body template

```markdown
## Summary
[1-3 bullet points of what was built]

## Architecture
[brief, if non-obvious — link to Feature Brief]

## Test results
- Unit: [counts from test-runner]
- E2E:  [counts or "n/a"]

## Review
- Auto-fixed: N findings
- Approved by user: M findings
- Skipped (with reason): K findings
- Security review: [skill summary or "n/a"]
- Simplify pass: [N improvements or "n/a"]

## Cross-repo
[for coordinated runs — link to sibling PRs]

## Manual test steps
1. [...]
```

Output the PR URL(s).

---

## Step 10.5: Emit Test Manifest

Document exactly what this run built, so it can be acceptance-tested later (by `/autofeature:test`, or
by hand). This is part of the ship phase — **no separate pipeline Task** (folded under Task 10, so it
does not collide with `fullrun.md`'s added Task 11 / Deploy).

**Skip only for `micro` tier** — unless the micro change is user-facing, in which case still emit a
short manifest.

Read `$AUTOFEATURE_HOME/adapted/feature-test-manifest.md` and write
`.autofeature/tests/[slug]-[date].md` from material **already gathered in this run** — do not start a
fresh investigation:

- Feature name, slug, scope tier, platforms (STACK), branch, PR URL → from the Feature Brief + Step 10.
- **Surfaces built** (web routes / API endpoints / mobile screens) → from
  `git diff --stat origin/[base]...HEAD` and the architect `## *Plan*` sections of the brief.
- **Acceptance flows (AF-N)** → expand the brief's golden path + the plan's **shadow paths / error
  map** (nil / empty / invalid / unauthorized) into full flows with preconditions + expected results.
  The PR "Manual test steps" are the seed — make each a complete, observable flow.
- **Setup** + **Out of scope** → from the brief's auth/seed/feature-flag needs and anything the plan
  deferred to fast-follow / cut.

For cross-repo runs, emit one manifest per repo touched (or one manifest with a Surfaces block per
repo), matching the split in `.autofeature/coordination.md`.

Then link it from the PR body — add to the "Manual test steps" section:
> Test Manifest: `.autofeature/tests/[slug]-[date].md` — run `/autofeature:test` to drive these flows.

---

## Step 11: Cleanup Checkpoint

```bash
cat >> .autofeature/checkpoints/$(ls -t .autofeature/checkpoints/ | head -1) << 'EOF'

### Completed
PR(s): [URLs]
Status: SHIPPED
EOF
```

---

## Final Output

```
=== AutoFeature Complete ===
Feature:  [feature name]
Scope:    [micro | single-layer | cross-stack | cross-repo]
Branch:   feature/[slug]
PR(s):    [URL list]

Specialists used:
  - express-mongo-architect: [yes/no]
  - react-architect:         [yes/no]
  - react-native-architect:  [yes/no]
  - mongo-data-modeler:      [yes/no]
  - api-contract-broker:     [yes/no]
  - test-runner:             [yes — N runs]

Files: N created, M modified
Tests: K written, suite all passing
Review: N auto-fixed, M approved, K skipped
Security review: [N issues / "n/a"]
Simplify: [N improvements / "n/a"]

Feature Brief:  .autofeature/designs/[slug]-[date].md
Test Manifest:  .autofeature/tests/[slug]-[date].md   (run /autofeature:test to drive it — skipped for micro)
```

---

## Emergency Stops (always, regardless of mode)

Use AskUserQuestion with clear options:

1. **Merge conflict** that can't be auto-resolved
2. **Tests failing in new code** after one fix attempt
3. **Critical security issue** in review (missing auth, injection vector, etc.)
4. **Ambiguous feature scope** — request could mean two different things
5. **3 hypotheses tested during debugging** — time to escalate
6. **User Challenge** — implementation requires contradicting the stated request
7. **Sibling repo dirty** during cross-repo run — ask whether to proceed without it
8. **Architect agent disagreement** unresolvable by api-contract-broker
9. **Product gap challenge (pre-build)** — the Step 4.5 product review finds the feature *as requested* ships a broken core flow or a half-capability; confirm scope before building

---

## File Reference

This orchestrator reads these at runtime. Edit them to change behavior.

### Methodology (`$AUTOFEATURE_HOME/adapted/`)
| File | Purpose |
|------|---------|
| `feature-interrogation.md` | Feature understanding |
| `feature-product-review.md` | Pre-build CEO/PM/flow-walker product review — gaps & broken flows (Workflow) |
| `feature-plan.md` | Technical planning + error maps + shadow paths |
| `feature-investigate.md` | Debugging when tests fail |
| `feature-review.md` | Pre-ship code review |
| `feature-review-checklist.md` | Review categories |
| `feature-design-check.md` | UI quality check |
| `feature-devex-check.md` | Developer experience check |
| `feature-ship.md` | Commit, push, PR |
| `feature-test-manifest.md` | Test Manifest format — emitted at Step 10.5 to document what was built (consumed by `/autofeature:test`) |

### Agents (`$AUTOFEATURE_HOME/agents/`)
| File | Purpose |
|------|---------|
| `express-mongo-architect.md` | Backend (Express + Mongoose) design + impl |
| `react-architect.md` | React web design + impl |
| `react-native-architect.md` | React Native design + impl |
| `mongo-data-modeler.md` | Schema, indexes, queries, migrations |
| `api-contract-broker.md` | Cross-repo contract reconciliation |
| `product-strategist.md` | CEO / PM / flow-walker product reviewer — pre-build gaps & broken flows |
| `test-runner.md` | Test execution + summary |

### Orchestrator helpers (`$AUTOFEATURE_HOME/orchestrator/`)
| File | Purpose |
|------|---------|
| `scope-gate.md` | Classify feature size before fan-out |
| `cross-repo-detect.md` | Find sibling `*-api`/`*-mobile`/etc repos |
| `skill-wiring.md` | When to invoke Plan/Explore/security-review/simplify/frontend-design |
| `trello-scope.md` | Fetch Trello card via MCP, generate technical scope, post comment |
| `model-tiers.md` | Per-task model tier (cost/quality) — what `model:` each spawned agent gets |

### Source (`$AUTOFEATURE_HOME/source/`)
gstack reference baseline. Do not edit — diff against gstack updates instead.
