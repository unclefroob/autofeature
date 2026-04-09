---
adapted-from: source/ship.md
changes: |
  - Removed all gstack bin references (gstack-review-log, gstack-slug, gstack-metrics, etc.)
  - Removed eval suites step (gstack-specific)
  - Removed Greptile integration
  - Removed document-release auto-invoke
  - Removed test-lane (gstack-specific test runner) — detect project test command instead
  - Removed adversarial step, learnings log, scope drift, plan completion audit
  - Removed VERSION file management (projects may not use gstack versioning)
  - Simplified CHANGELOG to optional (only if project has one)
  - Kept: merge base, run tests, pre-landing review, bisectable commits, push, create PR
  - Added: tech stack test command detection
  - Both modes behave the same — ship is always non-interactive except for hard stops
status: ADAPTED
---

# Feature Ship

Non-interactive ship workflow. Merge base → run tests → review → commit → push → PR.

**Hard stops (always ask):**
- On the base branch
- Merge conflicts that can't be auto-resolved
- Test failures in new code (not pre-existing)
- Pre-landing review finds ASK items (security/critical issues)

**Never stop for:**
- Uncommitted changes (always include them)
- Commit message approval (auto-commit)
- Auto-fixable review findings

---

## Step 1: Pre-flight

```bash
git branch --show-current
git status
git diff origin/main...HEAD --stat
git log origin/main..HEAD --oneline
```

1. If on main/master: **ABORT** — "Ship from a feature branch."
2. If no diff: **ABORT** — "Nothing to ship — no changes against main."
3. Output: branch name, number of commits, files changed.

---

## Step 2: Detect test command

Check CLAUDE.md first — it may specify the test command explicitly.

If not, detect from project structure:

```bash
# Check package.json for test script
cat package.json 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('scripts',{}).get('test','NOT_FOUND'))" 2>/dev/null

# Swift / Xcode project
ls *.xcodeproj 2>/dev/null && echo "SWIFT_XCODE"
ls Package.swift 2>/dev/null && echo "SWIFT_SPM"

# Kotlin / Android
ls gradlew 2>/dev/null && echo "GRADLE"
```

**Test command mapping:**
- `npm test` / `npx vitest` / `npx jest` — Node.js / React / React Native
- `xcodebuild test -scheme [scheme] -destination 'platform=iOS Simulator,...'` — iOS/Xcode
- `swift test` — Swift Package Manager
- `./gradlew test` — Kotlin/Android unit tests
- `./gradlew connectedAndroidTest` — Kotlin/Android instrumented tests

If test command cannot be determined, use AskUserQuestion to ask.

---

## Step 3: Merge the base branch

```bash
git fetch origin main && git merge origin/main --no-edit
```

**If merge conflicts:**
- Auto-resolve simple conflicts (CHANGELOG ordering, package-lock version bumps)
- **STOP** for complex conflicts — show them and ask how to resolve

**If already up to date:** Continue.

---

## Step 4: Run tests

Run the detected test command. Capture output.

```bash
# Example for Node.js
npm test 2>&1 | tee /tmp/autofeature_tests.txt
```

**If tests fail:**
1. Read the failure output
2. Determine: is this failure in code from THIS branch or pre-existing?
3. If pre-existing (exists on main too): note it, continue
4. If in-branch failure: **STOP** — fix the failing test before proceeding

Output: pass/fail counts, duration.

---

## Step 5: Pre-Landing Review

Run the review from `adapted/feature-review.md`:

1. `git diff origin/main` — get the full diff
2. Run Critical Pass + Informational Pass
3. Apply Fix-First heuristic
4. Auto-fix AUTO-FIX items, commit fixed files
5. If ASK items remain, present via AskUserQuestion
6. If fixes were applied: **STOP** — re-run from Step 4

Output: review summary.

---

## Step 6: Commit (bisectable)

Group changes into logical commits. Each commit = one logical unit.

**Commit ordering:**
1. Infrastructure (config, migrations, new routes)
2. Core logic (services, models, business logic + their tests)
3. UI layer (components, screens, views + their tests)
4. Final: any remaining changes

**Commit message format:** `<type>: <summary>`
Types: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`

**Rules:**
- Source file and its test file go in the same commit
- If total diff < 50 lines across < 4 files: single commit is fine
- Each commit must be independently valid (no broken imports)

```bash
git add [specific files]
git commit -m "feat: [summary]"
```

---

## Step 7: Verification gate

**IRON LAW: NO PUSHING WITHOUT FRESH TEST EVIDENCE.**

If any code changed during Steps 5-6 (review fixes, commits):

```bash
npm test 2>&1 | tail -20   # or equivalent for detected stack
```

If tests fail: **STOP**. Fix before pushing.

---

## Step 8: Push

```bash
git push -u origin $(git branch --show-current)
```

---

## Step 9: Create PR

Check if a PR already exists:

```bash
gh pr view --json url,number,state 2>/dev/null
```

If open PR exists: update the body.
If no PR: create one.

```bash
gh pr create \
  --base main \
  --title "feat: [feature name from branch/Feature Brief]" \
  --body "$(cat <<'EOF'
## Summary
[Summarize what this PR does — reference the Feature Brief if available]

## Changes
- [file/module]: [what changed]

## Unit Tests
- [test file]: [what is covered]

## Test Results
- Tests: [pass count] passing, [fail count] failing
- Command: [test command used]

## Pre-Landing Review
[Review summary — N issues found, M auto-fixed, K skipped]

## Test plan
- [ ] [manual test step 1]
- [ ] [manual test step 2]

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

If `gh` is not available: output branch name and push URL, instruct to create PR manually.

**Output the PR URL.**

---

## Rules

- **Never skip tests.** Test failures in new code = hard stop.
- **Never force push.** Regular `git push` only.
- **Never push without fresh verification evidence.**
- **Always use specific file names when staging.** Never `git add .` — it can include build artifacts, secrets, or generated files.
- **If CLAUDE.md has a different test command: use that, not the detected one.**
