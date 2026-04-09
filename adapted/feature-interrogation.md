---
adapted-from: source/office-hours.md
changes: |
  - Removed startup diagnostic mode entirely (builder mode only)
  - Removed gstack-specific design doc save path (uses .autofeature/designs/ locally)
  - Removed SLUG_EVAL and gstack bin references
  - Simplified context gathering (no gstack learnings search)
  - Added tech stack awareness for Node.js, React, React Native, Swift, Kotlin
status: ADAPTED
---

# Feature Interrogation

Builder-mode feature interrogation. Understand the feature fully before planning or building.

**Goal:** Produce a clear Feature Brief that serves as the source of truth for planning and implementation.

---

## Step 1: Context Gathering

Before asking anything, understand what already exists.

1. Read `CLAUDE.md` if it exists — it contains project conventions, test commands, and architecture notes.
2. Run `git log --oneline -20` to understand recent history.
3. Run `git diff origin/main --stat 2>/dev/null` to see current state.
4. **Detect tech stack:**
   ```bash
   # Node.js backend
   [ -f package.json ] && cat package.json | grep -E '"(express|fastify|nestjs|hapi|koa)"' && echo "NODE_BACKEND"
   # React frontend
   [ -f package.json ] && cat package.json | grep '"react"' && echo "REACT"
   # React Native
   [ -f package.json ] && cat package.json | grep '"react-native"' && echo "REACT_NATIVE"
   # Swift / iOS
   ls *.xcodeproj 2>/dev/null || ls *.xcworkspace 2>/dev/null || ls Package.swift 2>/dev/null && echo "SWIFT"
   # Kotlin / Android
   ls build.gradle 2>/dev/null || ls build.gradle.kts 2>/dev/null && echo "KOTLIN_ANDROID"
   ```
5. Use Grep/Glob to find existing code most relevant to the requested feature.
6. Output a one-paragraph summary: "Here's what I understand about this project and the area you want to change: ..."

---

## Step 2: The Builder Conversation

Ask these questions to build a complete understanding of the feature. In **automated mode**, make reasonable assumptions and state them. In **checkpoint mode**, ask interactively via AskUserQuestion.

**Question 1: What problem does this solve?**
What is someone currently unable to do, or doing poorly, that this feature fixes? Be specific about the before/after.

**Question 2: Who is the primary user of this feature?**
Which user role, persona, or part of the app? Is this user-facing, internal tooling, or a system/background process?

**Question 3: What is the simplest version that's useful?**
What can be cut without making the feature pointless? What is the minimum that delivers the core value?

**Question 4: What does success look like?**
What is the observable outcome when this feature works correctly? How would you manually test it?

**Question 5: What existing code does this touch or extend?**
Based on the context gathered in Step 1, which existing modules, components, services, or APIs will this feature interact with?

**Question 6: What could go wrong?**
What are the edge cases, error states, or failure modes the implementation needs to handle?

---

## Step 3: Feature Brief Output

Produce a Feature Brief and save it to `.autofeature/designs/[slug]-[date].md`.

```markdown
# Feature: [name]
**Date:** YYYY-MM-DD
**Branch:** feature/[slug]
**Stack:** [detected stack]
**Status:** Draft

## Problem
[One paragraph: what is broken or missing, who is affected]

## Solution
[One paragraph: what will be built and why it solves the problem]

## User Story
As a [user/system], I want to [action] so that [outcome].

## Scope: IN
- [concrete deliverable 1]
- [concrete deliverable 2]

## Scope: OUT
- [explicitly deferred item — with reason]

## Existing Code to Touch
- [file or module]: [how it will be changed]

## Edge Cases to Handle
- [edge case 1]
- [edge case 2]

## Test Scenarios
- [scenario 1: what to test and expected outcome]
- [scenario 2]

## Open Questions
- [anything that needs a decision before or during implementation]
```

In **automated mode**: produce the Feature Brief immediately using best judgment from the context gathered. Note any assumptions made.

In **checkpoint mode**: present the draft Feature Brief and ask for approval before proceeding to planning.

---

## Anti-Patterns

- **Don't skip context gathering.** Reading the codebase first prevents planning features the codebase already has.
- **Don't agree with the feature description uncritically.** If the stated problem has a simpler solution already in the codebase, say so.
- **Don't let "simplest version" become a cop-out.** Simple doesn't mean incomplete. The simplest version should still handle the core edge cases.
- **Don't start planning before edge cases are identified.** Edge cases discovered mid-implementation are 10x more expensive.
