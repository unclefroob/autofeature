---
name: autofeature
description: |
  Build an entire feature end-to-end from a single prompt.
  Pipeline: interrogate → plan → branch → implement → test → review → ship.
  Supports automated (fire-and-forget) and checkpoint (pause-and-approve) modes.
  Works for Node.js, React, React Native, Swift, and Kotlin projects.
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
---

# AutoFeature — End-to-End Feature Builder

## Builder Principles (always active)

**Boil the Lake:** The marginal cost of completeness is near-zero with AI. Always choose the complete implementation over the 90% shortcut. Tests, edge cases, error paths — these are lakes, not oceans.

**Search Before Building:** Before designing any solution involving unfamiliar patterns, runtime capabilities, or infrastructure — check if the framework already has a built-in. The cost of checking is near-zero.

**User Sovereignty:** Recommendations are presented, not applied. When implementation requires a decision that changes what was asked for — present it, explain why, and ask. Never act unilaterally.

---

## Pipeline

```
resume? → interrogate → plan → branch → tdd-implement → verify → review → design → dx → ship
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

If no checkpoints or user chooses fresh start: proceed to Step 1.

If resuming: read `.autofeature/checkpoints/[latest].md`, reconstruct context, and jump to the phase indicated in the checkpoint's `next_step` field.

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
> **B) Checkpoint** — pause for approval after: feature spec, implementation plan, implementation, before shipping.

**In AUTOMATED mode**, apply these 6 decision principles for all choices:
1. Choose completeness — cover more edge cases, not fewer
2. Boil lakes — fix everything in blast radius of the change
3. Pragmatic — pick the cleaner option; don't deliberate
4. DRY — reuse what exists
5. Explicit over clever — readable code beats smart code
6. Bias toward action — ship, don't debate

Only interrupt for a **User Challenge**: when implementation would need to fundamentally contradict the stated feature request.

---

## Step 2: Feature Interrogation

Read and follow `/run/media/ryan/Files/dev/autofeature/adapted/feature-interrogation.md`.

**Goal:** Produce a Feature Brief at `.autofeature/designs/[slug]-[date].md`.

**Slug:** Convert feature request to kebab-case, max 5 words.
Example: "add user profile photo upload" → `user-profile-photo-upload`

**Context gathering:**
1. Read `CLAUDE.md` if it exists (project conventions, test commands, architecture)
2. Run `git log --oneline -20`
3. Detect tech stack:
   ```bash
   # Node backend
   cat package.json 2>/dev/null | grep -E '"(express|fastify|@nestjs|hapi|koa)"' && echo "NODE_BACKEND"
   # React
   cat package.json 2>/dev/null | grep '"react"' | grep -v '"react-native"' && echo "REACT"
   # React Native
   cat package.json 2>/dev/null | grep '"react-native"' && echo "REACT_NATIVE"
   # Swift
   ls *.xcodeproj *.xcworkspace Package.swift 2>/dev/null && echo "SWIFT"
   # Kotlin/Android
   ls build.gradle build.gradle.kts 2>/dev/null && echo "KOTLIN_ANDROID"
   ```
4. Search Before Building: use Grep/Glob to find existing code that might already partially solve the feature

**AUTOMATED:** Answer interrogation questions from context. State assumptions in Feature Brief.

**CHECKPOINT:** Ask the 6 builder questions interactively. Then:

> Feature Brief:
> [content]
>
> A) Approve — proceed to planning
> B) Revise — [tell me what to change]
> C) Abort

---

## Step 3: Technical Planning

Read and follow `/run/media/ryan/Files/dev/autofeature/adapted/feature-plan.md`.

**Goal:** Produce an Implementation Plan appended to the Feature Brief.

Includes:
- Step 0: Scope Challenge
- Step 1: Architecture Review (with stack-specific checks)
- Step 2: Code Quality Review
- Step 3: Unit Test Plan (with shadow path mapping: happy/nil/empty/error)
- Step 3b: Error & Rescue Map (what can fail, how it's handled, what user sees)
- Step 3c: Shadow Path Testing (4 paths per data flow)
- Step 3d: Observability Checklist (logging, error context)
- Step 4: Performance Review

**AUTOMATED:** Apply 6 decision principles. Only surface User Challenges.

**CHECKPOINT:**

> Implementation Plan:
> [plan content]
>
> A) Approve — proceed to branch creation and implementation
> B) Revise — [tell me what to change]
> C) Abort

---

## Step 4: Create Feature Branch

```bash
# Get base branch
BASE=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}' || echo "main")
git fetch origin $BASE
git checkout $BASE
git pull origin $BASE

# Create feature branch
BRANCH="feature/[slug]"
git checkout -b "$BRANCH"
echo "Branch created: $BRANCH"
```

---

## Step 5: TDD Implementation

Follow the Implementation Plan from Step 3 using test-first development.

**Before writing any code:**
- Re-read `CLAUDE.md` for conventions (naming, folder structure, import style)
- Read the existing files you'll modify — understand the pattern before adding to it

**The TDD cycle (per function/component):**

```
RED → verify RED → GREEN → verify GREEN → REFACTOR
```

For each function or component in the implementation plan:

1. **RED — Write the failing test first.** One behavior per test. Clear name. Real code (no mocks unless unavoidable).
2. **Verify RED — Run the test and watch it fail.** Confirm it fails because the feature is missing, not because of a typo. If it passes immediately, the test is testing existing behavior — fix the test.
3. **GREEN — Write the minimal code to pass.** Simplest implementation that makes the test pass. No extras, no YAGNI.
4. **Verify GREEN — Run the test and confirm it passes.** Also confirm no other tests broke.
5. **REFACTOR — Clean up.** Remove duplication, improve names, extract helpers. Keep tests green.

Then move to the next function.

**Test framework detection:**
```bash
cat package.json 2>/dev/null | python3 -c "
import sys, json
d = json.load(sys.stdin)
deps = {**d.get('dependencies',{}), **d.get('devDependencies',{})}
print('jest' if 'jest' in deps else 'vitest' if 'vitest' in deps else 'unknown')
" 2>/dev/null
ls *Tests/*.swift 2>/dev/null && echo "XCTEST"
ls src/test/ 2>/dev/null && echo "JUNIT"
```

**Coverage per new function/component:**
- Happy path — expected input → expected output
- Nil/empty/zero edge cases
- Error paths — dependency failure → correct behavior
- Shadow paths from the Error & Rescue Map

**Per-stack conventions:** (see `feature-plan.md` Step 4 for full details)
- Node.js: async/await throughout, TypeScript strict types, explicit error handling
- React: functional components, hooks, typed props
- React Native: Platform.OS checks, FlatList for lists, StyleSheet.create()
- Swift: protocol-oriented dependencies (for testability), async/await or DispatchQueue
- Kotlin: ViewModels, coroutines with proper scope, null-safe operators

**Write testable code:** Pure functions where possible. I/O injected as dependencies. Avoid side effects in constructors.

If tests fail unexpectedly during implementation: **invoke `feature-investigate.md`** before attempting a fix. Follow the Iron Law: no fix without root cause.

**Save checkpoint after implementation:**
```bash
mkdir -p .autofeature/checkpoints
cat > .autofeature/checkpoints/$(date +%Y%m%d-%H%M%S)-impl.md << 'EOF'
---
status: post-implementation
branch: BRANCH_NAME
next_step: verify
---

## Working on: [feature name]

### Implemented
[list of files created/modified]

### Remaining
1. Full test suite run
2. Pre-ship review
3. Ship

### Notes
[any gotchas, open questions, decisions made]
EOF
```

**CHECKPOINT:**

> Implementation complete (TDD):
>
> Created: [files]
> Modified: [files]
> Tests written: N (all watched fail then pass)
>
> A) Proceed to full verification
> B) Revise — [tell me what to change]

---

## Step 6: Verification Gate

**The Iron Law: no completion claim without fresh evidence.**

Run the full test suite now and paste the output. Do not summarize, do not say "tests pass" — show it.

```bash
# Run detected test command (or from CLAUDE.md)
[test command] 2>&1 | tee /tmp/autofeature_tests.txt
```

Required evidence before proceeding:
- Test command output showing pass/fail counts
- Zero failures
- No new warnings compared to baseline

If any tests fail: **invoke `feature-investigate.md`** before attempting a fix. Write a regression test that fails without the fix and passes with it. Re-run the full suite after fixing and show the output again.

Do NOT proceed to review without showing test output proving all pass.

**CHECKPOINT:**

> Verification:
>
> [paste test output — e.g. "34 passed, 0 failed"]
>
> A) Proceed to review and ship
> B) Add more tests for [scenario]
> C) Stop here — I'll review manually

---

## Step 7: Pre-Ship Review

Read and follow `/run/media/ryan/Files/dev/autofeature/adapted/feature-review.md`.

Runs:
1. Critical pass (security, race conditions, SQL, enums)
2. Informational pass (type safety, async issues, completeness gaps)
3. Testing pass (missing negative paths, edge cases, isolation)
4. Design pass — reads `feature-design-check.md` (only if frontend files changed)
5. DX pass — reads `feature-devex-check.md` (only if API/CLI/SDK/docs changed)

Apply Fix-First: AUTO-FIX what can be fixed mechanically, batch ASK items.

If fixes were applied: re-run tests (Step 6) before continuing.

---

## Step 8: Ship

Read and follow `/run/media/ryan/Files/dev/autofeature/adapted/feature-ship.md`.

Steps:
1. Pre-flight (branch check, diff summary)
2. Detect test command
3. Merge base branch
4. Final test run — **paste output, do not claim passing without evidence**
5. Verification gate — zero failures confirmed
6. Bisectable commits
7. Push
8. Create PR

**PR body includes:**
- Summary of all changes
- Unit test results
- Review findings (N auto-fixed, M approved, K skipped)
- Design review summary (if ran)
- DX review summary (if ran)
- Manual test steps

**Output the PR URL.**

---

## Step 9: Cleanup Checkpoint

After successful ship, mark checkpoint as complete:

```bash
cat >> .autofeature/checkpoints/$(ls -t .autofeature/checkpoints/ | head -1) << 'EOF'

### Completed
PR: [PR URL]
Status: SHIPPED
EOF
```

---

## Final Output

```
=== AutoFeature Complete ===
Feature:  [feature name]
Branch:   feature/[slug]
PR:       [PR URL]

Created:  N files
Modified: M files
Tests:    K written (TDD, all watched fail→pass), suite: all passing
Review:   N auto-fixed, M approved, K skipped
Design:   [N issues / "not applicable"]
DX:       [N issues / "not applicable"]

Feature Brief: .autofeature/designs/[slug]-[date].md
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

---

## Methodology References

This skill reads these files at runtime. Edit them to change behavior.

| File | Purpose | Based on |
|------|---------|----------|
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-interrogation.md` | Feature understanding | gstack /office-hours (builder mode) |
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-plan.md` | Technical planning | gstack /plan-eng-review + plan-ceo-review (error maps, shadow paths) |
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-investigate.md` | Debugging when tests fail | gstack /investigate |
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-review.md` | Pre-ship code review | gstack /review |
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-review-checklist.md` | Review categories | gstack review/checklist.md + specialists |
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-design-check.md` | UI quality check | gstack review/design-checklist.md |
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-devex-check.md` | Developer experience check | gstack /plan-devex-review + /devex-review |
| `/run/media/ryan/Files/dev/autofeature/adapted/feature-ship.md` | Commit, push, PR | gstack /ship |
| `/run/media/ryan/Files/dev/autofeature/source/ethos.md` | Decision principles | gstack ETHOS.md |
| `/run/media/ryan/Files/dev/autofeature/source/autoplan-principles.md` | 6 decision principles | gstack /autoplan |
