---
adapted-from: source/investigate.md
changes: |
  - Removed gstack freeze/hook mechanism (scope lock simplified to a note)
  - Removed gstack learnings search/log
  - Added stack-specific pattern table for Node.js, React, React Native, Swift, Kotlin
  - Added stack-specific debugging commands and tools
  - Kept Iron Law, 5-phase structure, 3-strike rule, DEBUG REPORT format unchanged
  - Removed {{PLACEHOLDERS}}
status: ADAPTED
---

# Systematic Debugging

Invoked when a bug is encountered during feature implementation, or when tests fail and the cause isn't obvious from the error message alone.

## Iron Law

**NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST.**

Fixing symptoms creates whack-a-mole debugging. Every fix that doesn't address root cause makes the next bug harder to find. Find the root cause, then fix it.

---

## Phase 1: Root Cause Investigation

1. **Collect symptoms.** Read the error messages, stack traces, and reproduction steps. If insufficient context, ask ONE question at a time.

2. **Read the code.** Trace the code path from the symptom back to potential causes. Use Grep to find all references, Read to understand the logic.

3. **Check recent changes:**
   ```bash
   git log --oneline -20 -- <affected-files>
   git diff HEAD~5..HEAD -- <affected-files>
   ```
   Was this working before this feature branch? What changed?

4. **Reproduce deterministically.** If you can't reliably trigger it, gather more evidence before proceeding.

Output: **"Root cause hypothesis: [specific, testable claim about what is wrong and why]"**

---

## Phase 2: Pattern Analysis

Check if the bug matches a known pattern:

| Pattern | Signature | Where to look |
|---------|-----------|---------------|
| Race condition | Intermittent, timing-dependent, passes in isolation | Concurrent async operations, shared mutable state |
| Null/undefined propagation | TypeError, Cannot read property | Missing null checks on optional values |
| State corruption | Stale UI, inconsistent data | React state updates, async race, mutable closures |
| Integration failure | Timeout, unexpected response format | API call boundaries, network layer |
| Config/env drift | Works locally, fails in CI or staging | Env vars, feature flags, different node versions |
| Stale cache | Shows old data, works after hard refresh | Browser cache, memo/useCallback deps, Redis |

### Stack-specific patterns

**Node.js:**
- Unhandled promise rejection silently swallowed → add `.catch()` or `try/catch`
- Callback called multiple times (bug in third-party lib or incorrect logic)
- `require()` caching stale module state in tests (mock with `jest.resetModules()`)
- Port conflict from previous process not cleaned up

**React:**
- `useEffect` with wrong deps array → stale closure over old state/props
- Multiple React instances (common with monorepos/links) → deduplicate react in bundler
- Key prop instability causing unnecessary remounts
- State update on unmounted component → missing cleanup in useEffect

**React Native:**
- Metro bundler serving cached JS → clear cache: `npx react-native start --reset-cache`
- Native module not linked → pod install / gradle sync
- Platform-specific code failing on one platform only
- AsyncStorage or native DB returning stale data

**Swift / iOS:**
- Thread safety violation (purple runtime error) → wrap in DispatchQueue.main.async
- Optional force-unwrap crash → add guard/if-let
- ARC retain cycle (memory warning, then crash) → use `[weak self]`
- Xcode simulator state stale → Device > Erase All Content and Settings

**Kotlin / Android:**
- NetworkOnMainThreadException → move to Dispatchers.IO
- NullPointerException on `!!` → use `?.` or guard with `?:`
- ClassCastException on Parcelable → verify all classes implement Parcelable correctly
- LiveData not observed on main thread

### Check for prior occurrences

```bash
# Find prior fixes in the same area
git log --oneline --all --grep="fix" -- <affected-directory>

# Find similar error in git history
git log --all -S "<error message keyword>" --oneline
```

Recurring bugs in the same files are an architectural smell, not coincidence.

---

## Phase 3: Hypothesis Testing

Before writing ANY fix, verify your hypothesis.

1. **Add a temporary diagnostic** at the suspected root cause:
   - Node.js/React/RN: `console.log('[DEBUG]', variableName)` or `debugger;`
   - Swift: `print("[DEBUG] \(variableName)")` or set a breakpoint
   - Kotlin: `Log.d("DEBUG", "variable: $variableName")` or set a breakpoint

2. **Run the reproduction.** Does the evidence match the hypothesis?

3. **If wrong:** Before forming the next hypothesis, search for the error message (sanitized — strip any PII, IPs, or proprietary data). Return to Phase 1. Do not guess.

4. **3-strike rule:** If 3 hypotheses fail, **STOP**:

   > 3 hypotheses tested, none match. This may be an architectural issue.
   >
   > A) Continue — I have a new hypothesis: [describe]
   > B) Add logging and observe next occurrence
   > C) Escalate — needs someone who knows this area of the system

**Red flags:**
- "Quick fix for now" — fix it right or escalate
- Proposing a fix before tracing data flow — that's guessing
- Each fix reveals a new problem elsewhere — wrong layer, not wrong code

---

## Phase 4: Implementation

Once root cause is confirmed:

1. **Fix root cause, not symptom.** Smallest change that eliminates the actual problem.

2. **Minimal diff.** Fewest files touched, fewest lines changed. No opportunistic refactoring.

3. **Write a regression test** that:
   - **Fails** without the fix (proves the test catches the bug)
   - **Passes** with the fix (proves the fix works)

4. **Run the full test suite.** Paste output. No new failures allowed.

5. **If fix touches >5 files:** Flag the blast radius:
   > This fix spans N files — larger than expected for a single bug.
   > A) Proceed — root cause genuinely spans these files
   > B) Split — fix critical path now, defer the rest
   > C) Rethink — there may be a more targeted approach

---

## Phase 5: Verification & Report

**Reproduce the original bug and confirm it's fixed.** This is not optional.

Run tests. Paste output.

```
DEBUG REPORT
════════════════════════════════════════
Symptom:         [what was observed]
Root cause:      [what was actually wrong, with file:line]
Fix:             [what was changed]
Evidence:        [test output or reproduction showing fix works]
Regression test: [file:line of new test]
Related:         [any TODOS, prior bugs in same area]
Status:          DONE | DONE_WITH_CONCERNS | BLOCKED
════════════════════════════════════════
```

---

## Rules

- **Never apply a fix you cannot verify.** If you can't reproduce and confirm, don't ship it.
- **Never say "this should fix it."** Prove it with test output.
- **3+ failed hypotheses → question the architecture**, not your hypothesis quality.
- Remove all temporary debug logs before committing.
