---
name: skill-wiring
purpose: When and how to invoke existing Claude Code skills inside autofeature
status: CUSTOM
---

# Skill Wiring

The autofeature pipeline invokes other installed skills at specific phases. This file documents which skill runs when and why.

## Skills used

| Skill | Phase | Mode | Purpose |
|-------|-------|------|---------|
| `Plan` (subagent) | Step 3 | always | Owns the technical planning so the orchestrator's context stays clean |
| `Explore` (subagent) | Step 2 | always | Owns context gathering (grep/glob/file scans) for the same reason |
| `frontend-design` (skill) | Step 5 | when UI is added | Generates polished, non-generic UI for new components |
| `security-review` (skill) | Step 7 | when scope hits security triggers | Real security audit beyond the inline checklist |
| `simplify` (skill) | Step 7 | after Step 7 fixes | Final pass for reuse, quality, and dead code |

Note: `Plan` and `Explore` are built-in subagent_types invoked via the `Agent` tool. `frontend-design`, `security-review`, `simplify` are user-invocable skills invoked via the `Skill` tool.

## Step 2: Explore agent for context gathering

Replaces manual grep/glob in the orchestrator's main context.

```
Agent({
  description: "Autofeature context scan",
  subagent_type: "Explore",
  prompt: "Gather context for autofeature run on: [feature description].
    Find:
    - Relevant existing modules / files that touch this domain
    - Existing tests for that domain
    - The project's test command, lint command, and build command
    - The auth pattern (if the feature involves protected resources)
    - The data model file for the affected entity (if any)
    Return file paths only with one-line role descriptions. No code excerpts unless directly load-bearing."
})
```

## Step 3: Plan subagent for technical planning

Replaces the orchestrator drafting the plan inline.

```
Agent({
  description: "Autofeature technical plan",
  subagent_type: "Plan",
  prompt: "Read the Feature Brief at .autofeature/designs/[slug]-[date].md.
    Read these methodology files:
    - $AUTOFEATURE_HOME/adapted/feature-plan.md
    Produce the Implementation Plan section with:
    - Step 0: Scope Challenge
    - Step 1: Architecture Review (stack-specific)
    - Step 2: Code Quality Review
    - Step 3: Unit Test Plan with shadow paths
    - Step 3b: Error & Rescue Map
    - Step 3c: Shadow Path Testing
    - Step 3d: Observability Checklist
    - Step 4: Performance Review
    Append to the same Feature Brief file. Return only a 5-line summary of decisions made."
})
```

## Step 5: frontend-design skill for new UI

When the feature creates new components or pages (not just tweaks existing), invoke `frontend-design` BEFORE the implementation specialists wire up the data layer. This way the implementer has a real component shape to plug into, and the result avoids generic AI aesthetics.

```
Skill({
  skill: "frontend-design:frontend-design",
  args: "Generate the [component name] component for [feature description].
    It must: [list of requirements from Frontend Plan]
    Style system: [tailwind | styled-components | css-modules — match repo]
    Save to: [target path]"
})
```

Skip this for: pure data-display tweaks, admin-only screens where polish is not required, single-component edits.

## Step 7: security-review skill

Invoke when ANY of these is true:
- Feature touches auth, sessions, tokens, refresh flow
- Feature accepts file uploads
- Feature renders user-supplied content as HTML or markdown
- Feature constructs database queries from request input
- Feature changes CORS, CSP, or other security headers
- Feature changes how secrets/env vars are loaded
- Feature deserializes data from external sources

```
Skill({
  skill: "security-review",
  args: ""  // skill operates on pending changes on current branch
})
```

Run it AFTER the inline review fan-out (Critical/Informational/Testing/Design/DX agents) but BEFORE applying fixes — so its findings are in the same triage queue.

## Step 7: simplify skill (after fixes)

Run AFTER all review fixes have been applied. Final pass for:
- Code that could reuse an existing helper
- Dead branches / unused params introduced during the back-and-forth
- Over-abstraction — a helper added for a one-off use

```
Skill({
  skill: "simplify",
  args: ""
})
```

Skip in `micro` scope tier — overkill for tiny changes.

## Order of operations within Step 7

```
1. Spawn parallel review agents (Critical / Info / Testing / Design / DX)
2. While they run: invoke security-review skill (if scope triggers)
3. Collect all findings into Fix-First triage queue
4. AUTO-FIX what's mechanical
5. ASK user about judgment calls (batched — one AskUserQuestion call)
6. Apply approved fixes
7. Re-run test-runner agent
8. Invoke simplify skill (skip for micro tier)
9. Re-run test-runner one more time if simplify made changes
```

## Skills NOT invoked (intentionally)

- `init` — autofeature already produces a Feature Brief; don't also write CLAUDE.md
- `review` — autofeature has its own review fan-out; the standalone /review skill duplicates work
- `loop` / `schedule` — autofeature is a one-shot pipeline; no looping or scheduling
- `claude-api` — only invoke if the feature itself involves Anthropic API code, in which case the relevant architect agent reads it themselves

## Detection of skill availability

Some skills are plugin-installed and may not be present. Before invoking, the orchestrator should be tolerant:
- If `frontend-design` skill isn't available → fall back to react-architect / react-native-architect handling the UI design
- If `security-review` skill isn't available → fall back to the inline security checks in feature-review-checklist.md
- If `simplify` skill isn't available → fall back to a final inline pass against feature-review-checklist.md's "reuse" section
- `Plan` and `Explore` subagents are built-in, always available

The orchestrator can detect availability by checking the skills list in its system reminder (skills appear under "available-skills"). Don't try to invoke a skill not listed there.
