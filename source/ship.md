---
gstack-source: gstack/ship/SKILL.md.tmpl
extracted: 2026-04-09
note: Full extract. {{PLACEHOLDERS}} kept as markers. gstack bin references, eval suites, and metrics logging preserved for reference.
status: ORIGINAL (unmodified)
---

# Ship Workflow (Source Extract)

> Original gstack description:
> Ship workflow: detect + merge base branch, run tests, review diff, bump VERSION,
> update CHANGELOG, commit, push, create PR.

---

{{PREAMBLE}}

{{BASE_BRANCH_DETECT}}

# Ship: Fully Automated Ship Workflow

**Only stop for:**
- On the base branch (abort)
- Merge conflicts that can't be auto-resolved (stop, show conflicts)
- In-branch test failures
- Pre-landing review finds ASK items that need user judgment
- MINOR or MAJOR version bump needed (ask)

**Never stop for:**
- Uncommitted changes (always include them)
- Version bump choice (auto-pick MICRO or PATCH)
- CHANGELOG content (auto-generate from diff)
- Commit message approval (auto-commit)
- Auto-fixable review findings

---

## Step 1: Pre-flight

1. Check the current branch. If on the base branch or the repo's default branch, **abort**.
2. Run `git status` (never use `-uall`). Uncommitted changes are always included — no need to ask.
3. Run `git diff <base>...HEAD --stat` and `git log <base>..HEAD --oneline` to understand what's being shipped.
4. Check review readiness: {{REVIEW_DASHBOARD}}

---

## Step 1.5: Distribution Pipeline Check

If the diff introduces a new standalone artifact (CLI binary, library package, tool), verify that a distribution pipeline exists.

---

## Step 2: Merge the base branch (BEFORE tests)

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

**If there are merge conflicts:** Try to auto-resolve if simple. If complex, **STOP** and show them.

---

## Step 2.5: Test Framework Bootstrap

{{TEST_BOOTSTRAP}}

---

## Step 3: Run tests (on merged code)

Run both test suites in parallel:

```bash
bin/test-lane 2>&1 | tee /tmp/ship_tests.txt &
npm run test 2>&1 | tee /tmp/ship_vitest.txt &
wait
```

After both complete, check pass/fail.

**If any test fails:** Apply Test Failure Ownership Triage:
{{TEST_FAILURE_TRIAGE}}

---

## Step 3.25: Eval Suites (conditional)

[gstack-specific eval integration — omit in adapted version]

---

## Step 3.4: Test Coverage Audit

{{TEST_COVERAGE_AUDIT_SHIP}}

---

## Step 3.45: Plan Completion Audit

{{PLAN_COMPLETION_AUDIT_SHIP}}

---

{{PLAN_VERIFICATION_EXEC}}

{{LEARNINGS_SEARCH}}

{{SCOPE_DRIFT}}

---

## Step 3.5: Pre-Landing Review

Review the diff for structural issues that tests don't catch.

1. Run `git diff origin/<base>` to get the full diff.
2. Apply the review checklist in two passes:
   - **Pass 1 (CRITICAL):** SQL & Data Safety, LLM Output Trust Boundary
   - **Pass 2 (INFORMATIONAL):** All remaining categories
3. Classify each finding as AUTO-FIX or ASK.
4. Auto-fix all AUTO-FIX items.
5. If ASK items remain, present them in ONE AskUserQuestion.
6. **After all fixes (auto + user-approved):**
   - If ANY fixes were applied: commit fixed files, then **STOP** and tell the user to run `/ship` again.
   - If no fixes applied: continue to Step 4.

---

## Step 3.75: Address Greptile review comments (if PR exists)

[gstack integration — omit in adapted version]

---

{{ADVERSARIAL_STEP}}

{{LEARNINGS_LOG}}

## Step 4: Version bump (auto-decide)

1. Read the current `VERSION` file (4-digit format: `MAJOR.MINOR.PATCH.MICRO`)
2. **Auto-decide the bump level based on the diff:**
   - **MICRO** (4th digit): < 50 lines changed, trivial tweaks, typos, config
   - **PATCH** (3rd digit): 50+ lines changed, no feature signals detected
   - **MINOR** (2nd digit): **ASK the user** if ANY feature signal is detected, OR 500+ lines changed
   - **MAJOR** (1st digit): **ASK the user** — only for milestones or breaking changes
3. Write the new version to the `VERSION` file.

---

{{CHANGELOG_WORKFLOW}}

---

## Step 5.5: TODOS.md (auto-update)

Cross-reference the project's TODOS.md against the changes being shipped. Mark completed items automatically; prompt only if the file is missing or disorganized.

---

## Step 6: Commit (bisectable chunks)

**Goal:** Create small, logical commits that work well with `git bisect`.

1. Analyze the diff and group changes into logical commits.
2. **Commit ordering:**
   - Infrastructure: migrations, config changes, route additions
   - Models & services: new models, services (with their tests)
   - Controllers & views: controllers, views, components (with their tests)
   - VERSION + CHANGELOG + TODOS.md: always in the final commit
3. Rules for splitting:
   - A model and its test file go in the same commit
   - A service and its test file go in the same commit
   - If total diff is small (< 50 lines across < 4 files), a single commit is fine

---

## Step 6.5: Verification Gate

**IRON LAW: NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE.**

Before pushing, re-verify if code changed during Steps 4-6:
1. **Test verification:** If ANY code changed after Step 3's test run, re-run the test suite.
2. **Build verification:** If the project has a build step, run it.

**If tests fail here:** STOP. Do not push.

---

## Step 7: Push

```bash
git push -u origin <branch-name>
```

---

## Step 8: Create PR/MR

**If GitHub:**
```bash
gh pr create --base <base> --title "<type>: <summary>" --body "$(cat <<'EOF'
<PR body>
EOF
)"
```

**If GitLab:**
```bash
glab mr create -b <base> -t "<type>: <summary>" -d "$(cat <<'EOF'
<MR body>
EOF
)"
```

PR body sections:
- Summary
- Test Coverage
- Pre-Landing Review
- Test plan

---

## Important Rules

- **Never skip tests.** If tests fail, stop.
- **Never force push.** Use regular `git push` only.
- **Always use the 4-digit version format** from the VERSION file.
- **Split commits for bisectability** — each commit = one logical change.
- **Never push without fresh verification evidence.**
