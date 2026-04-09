---
gstack-source: gstack/review/SKILL.md.tmpl
extracted: 2026-04-09
note: Full extract. {{PLACEHOLDERS}} kept as markers. Greptile and gstack-bin references preserved for reference.
status: ORIGINAL (unmodified)
---

# Pre-Landing PR Review (Source Extract)

> Original gstack description:
> Pre-landing PR review. Analyzes diff against the base branch for SQL safety, LLM trust
> boundary violations, conditional side effects, and other structural issues.

---

{{PREAMBLE}}

{{BASE_BRANCH_DETECT}}

# Pre-Landing PR Review

You are running the `/review` workflow. Analyze the current branch's diff against the base branch for structural issues that tests don't catch.

---

## Step 1: Check branch

1. Run `git branch --show-current` to get the current branch.
2. If on the base branch, output: **"Nothing to review — you're on the base branch or have no changes against it."** and stop.
3. Run `git fetch origin <base> --quiet && git diff origin/<base> --stat` to check if there's a diff. If no diff, output the same message and stop.

---

{{SCOPE_DRIFT}}

{{PLAN_COMPLETION_AUDIT_REVIEW}}

## Step 2: Read the checklist

Read `.claude/skills/review/checklist.md`.

**If the file cannot be read, STOP and report the error.** Do not proceed without the checklist.

---

## Step 2.5: Check for Greptile review comments

[gstack integration — omit in adapted version]

---

## Step 3: Get the diff

Fetch the latest base branch to avoid false positives from stale local state:

```bash
git fetch origin <base> --quiet
```

Run `git diff origin/<base>` to get the full diff.

---

{{LEARNINGS_SEARCH}}

## Step 4: Critical pass (core review)

Apply the CRITICAL categories from the checklist against the diff:
SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary, Shell Injection, Enum & Value Completeness.

Also apply INFORMATIONAL categories: Async/Sync Mixing, Column/Field Name Safety, Type Coercion, View/Frontend, Time Window Safety, Completeness Gaps.

**Enum & Value Completeness requires reading code OUTSIDE the diff.** When the diff introduces a new enum value, status, tier, or type constant, use Grep to find all files that reference sibling values.

**Search-before-recommending:** When recommending a fix pattern (especially for concurrency, caching, auth, or framework-specific behavior):
- Verify the pattern is current best practice for the framework version in use
- Check if a built-in solution exists in newer versions before recommending a workaround

{{CONFIDENCE_CALIBRATION}}

---

{{REVIEW_ARMY}}

---

## Step 5: Fix-First Review

**Every finding gets action — not just critical ones.**

{{CROSS_REVIEW_DEDUP}}

### Step 5a: Classify each finding

For each finding, classify as AUTO-FIX or ASK per the Fix-First Heuristic:
- Critical findings lean toward ASK
- Informational findings lean toward AUTO-FIX

### Step 5b: Auto-fix all AUTO-FIX items

Apply each fix directly. For each one, output a one-line summary:
`[AUTO-FIXED] [file:line] Problem → what you did`

### Step 5c: Batch-ask about ASK items

If there are ASK items remaining, present them in ONE AskUserQuestion:

- List each item with a number, the severity label, the problem, and a recommended fix
- For each item, provide options: A) Fix as recommended, B) Skip
- Include an overall RECOMMENDATION

Example format:
```
I auto-fixed 5 issues. 2 need your input:

1. [CRITICAL] app/models/post.rb:42 — Race condition in status transition
   Fix: Add `WHERE status = 'draft'` to the UPDATE
   → A) Fix  B) Skip

2. [INFORMATIONAL] app/services/generator.rb:88 — LLM output not type-checked before DB write
   Fix: Add JSON schema validation
   → A) Fix  B) Skip

RECOMMENDATION: Fix both — #1 is a real race condition, #2 prevents silent data corruption.
```

### Step 5d: Apply user-approved fixes

Apply fixes for items where the user chose "Fix."

---

## Step 5.5: TODOS cross-reference

Read `TODOS.md` in the repository root (if it exists). Cross-reference the PR against open TODOs.

---

## Step 5.6: Documentation staleness check

Cross-reference the diff against documentation files. If code changes affect features described in a doc that was NOT updated in this branch, flag it as an INFORMATIONAL finding.

---

{{ADVERSARIAL_STEP}}

---

## Important Rules

- **Read the FULL diff before commenting.** Do not flag issues already addressed in the diff.
- **Fix-first, not read-only.** AUTO-FIX items are applied directly. ASK items are only applied after user approval. Never commit, push, or create PRs — that's /ship's job.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.
- **Verify claims:** If you claim "this pattern is safe" → cite the specific line proving safety.
