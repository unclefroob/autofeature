---
gstack-source: gstack/autoplan/SKILL.md.tmpl
extracted: 2026-04-09
note: Partial extract — the 6 decision principles and decision classification only.
status: ORIGINAL (unmodified)
---

# AutoPlan Decision Principles (Source Extract)

> Original gstack description:
> Auto-review pipeline — reads the full CEO, design, eng, and DX review skills from disk
> and runs them sequentially with auto-decisions using 6 decision principles.

---

## The 6 Decision Principles

These rules auto-answer every intermediate question:

1. **Choose completeness** — Ship the whole thing. Pick the approach that covers more edge cases.
2. **Boil lakes** — Fix everything in the blast radius (files modified by this plan + direct importers). Auto-approve expansions that are in blast radius AND < 1 day effort (< 5 files, no new infra).
3. **Pragmatic** — If two options fix the same thing, pick the cleaner one. 5 seconds choosing, not 5 minutes.
4. **DRY** — Duplicates existing functionality? Reject. Reuse what exists.
5. **Explicit over clever** — 10-line obvious fix > 200-line abstraction. Pick what a new contributor reads in 30 seconds.
6. **Bias toward action** — Merge > review cycles > stale deliberation. Flag concerns but don't block.

**Conflict resolution (context-dependent tiebreakers):**
- **CEO phase:** Completeness + Boil Lakes dominate.
- **Eng phase:** Explicit + Pragmatic dominate.
- **Design phase:** Explicit + Completeness dominate.

---

## Decision Classification

Every auto-decision is classified:

**Mechanical** — one clearly right answer. Auto-decide silently.

**Taste** — reasonable people could disagree. Auto-decide with recommendation, but surface at the final gate. Three natural sources:
1. **Close approaches** — top two are both viable with different tradeoffs.
2. **Borderline scope** — in blast radius but ambiguous.
3. **Model disagreements** — models recommend differently and have valid points.

**User Challenge** — both models agree the user's stated direction should change.
This is qualitatively different from taste decisions. NEVER auto-decided.

User Challenges require:
- **What the user said:** (their original direction)
- **What both models recommend:** (the change)
- **Why:** (the models' reasoning)
- **What context we might be missing:** (explicit acknowledgment of blind spots)
- **If we're wrong, the cost is:** (what happens if the user's original direction was right)

The user's original direction is the default. The models must make the case for change.

---

## Sequential Execution — MANDATORY

Phases MUST execute in strict order.
Each phase MUST complete fully before the next begins.
NEVER run phases in parallel — each builds on the previous.

Between each phase, emit a phase-transition summary and verify that all required
outputs from the prior phase are written before starting the next.

---

## What "Auto-Decide" Means

Auto-decide replaces the USER'S judgment with the 6 principles. It does NOT replace
the ANALYSIS. Every section must still be executed at full depth.

**Two exceptions — never auto-decided:**
1. Premises — require human judgment about what problem to solve.
2. User Challenges — when both models believe the user's direction should change.
