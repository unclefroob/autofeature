---
adapted-from: source/review.md, /run/media/ryan/Files/dev/autofeature/adapted/feature-review-checklist.md
changes: |
  - Removed gstack bin references entirely
  - Removed {{PLACEHOLDERS}}
  - Removed Greptile integration
  - Removed adversarial step (specialist subagents)
  - Removed scope drift and plan completion audit
  - Removed learnings search/log
  - Review checklist now lives in feature-review-checklist.md (referenced, not inline)
  - Added design check phase (references feature-design-check.md)
  - Kept Fix-First flow unchanged
  - Mode behavior unchanged (review is always the same regardless of mode)
status: ADAPTED
---

# Pre-Ship Code Review

Review the diff against main for structural issues that unit tests don't catch. Fix-first — not read-only.

---

## Step 1: Check branch

```bash
git branch --show-current
git fetch origin main --quiet
git diff origin/main --stat
```

If on main with no diff: stop — "Nothing to review."

---

## Step 2: Get the full diff

```bash
git fetch origin main --quiet
git diff origin/main
```

**Read the entire diff before flagging anything.** Do not flag issues already resolved in the diff.

---

## Step 3: Critical and Informational Pass

Read `/run/media/ryan/Files/dev/autofeature/adapted/feature-review-checklist.md` and apply all categories against the diff:

1. **Pass 1 (Critical):** SQL & Data Safety, Race Conditions, Security (Auth & Input), Injection Vectors, Cryptographic Issues, Secrets Exposure, Swift/Kotlin critical, Enum Completeness
2. **Pass 2 (Informational):** Node/TypeScript, React, React Native, Swift, Kotlin, Completeness Gaps, CI/CD
3. **Testing Pass:** Missing negative-path tests, missing edge-case coverage, test isolation, flakiness risk, coverage gaps

For the Enum Completeness check: use Grep to find all consumers of sibling values. Read those files — this check requires reading code outside the diff.

---

## Step 4: Design Pass (frontend only)

Read `/run/media/ryan/Files/dev/autofeature/adapted/feature-design-check.md`.

**Detect if frontend files changed:**
```bash
git diff origin/main --name-only | grep -E '\.(tsx|jsx|ts|js|css|scss|sass|less|html|swift|xml)$' | grep -v -E '(test|spec|__tests__|\.test\.|\.spec\.)'
```

If frontend files are in the diff: run all 6 design check categories. Otherwise skip silently.

---

## Step 4b: DX Pass (developer-facing features only)

Read `/run/media/ryan/Files/dev/autofeature/adapted/feature-devex-check.md`.

This pass checks developer experience quality for features that expose developer-facing surfaces:
- API endpoints, REST routes, GraphQL schema, webhooks
- CLI commands, flags, argument parsing
- Public library/SDK exports, type definitions
- Documentation changes (README, docs/, CHANGELOG, migration guides)

The file itself contains the applicability gate — it will self-exit if the diff has no
developer-facing surfaces. Run it unconditionally; it will determine its own scope.

---

## Step 5: Fix-First

### 5a: Classify findings

Per the Fix-First Heuristic in `feature-review-checklist.md`:
- **AUTO-FIX** — mechanical, one obviously right answer, low blast radius
- **ASK** — security/critical, requires design judgment, >20 lines, user-visible behavior

### 5b: Apply AUTO-FIX items

Apply directly. Output one line per fix:
`[AUTO-FIXED] [file:line] Problem → what was done`

### 5c: Batch ASK items

Present ALL ASK items in ONE `AskUserQuestion`:

```
Auto-fixed N issues. M need your input:

1. [CRITICAL] src/api/users.ts:42 — Missing auth check on DELETE /users/:id
   Fix: Add requireAuth() middleware
   → A) Fix  B) Skip

2. [CRITICAL] src/db/posts.ts:88 — SQL string interpolation
   Fix: Use parameterized query
   → A) Fix  B) Skip

RECOMMENDATION: Fix both.
```

If ≤ 3 ASK items: individual AskUserQuestion calls are fine.

### 5d: Apply approved fixes

Apply fixes for items where user chose Fix.

---

## Step 6: Verify claims

Before finalizing:
- "This is safe" → cite the specific line proving it
- "This is handled elsewhere" → read and cite the handling code
- "Tests cover this" → name the test file and function
- "Probably fine" → flag as unverified, not assumed safe

---

## Step 7: Output summary

```
Review: N issues found
  AUTO-FIXED: M items
  FIXED (approved): K items
  SKIPPED: L items

Design: N issues found (or "Design: No frontend files changed.")
```

---

## Rules

- Read the full diff first. Never flag something already fixed in the same diff.
- Fix-first always. Review is not a report — it produces fixes.
- Be terse. One line problem, one line fix.
- Never commit or push — that's the ship step's job.
