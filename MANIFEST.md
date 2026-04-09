# AutoFeature — File Manifest

Tracks the status of every file: original gstack extracts vs. adapted/custom files.

---

## Legend

| Status | Meaning |
|--------|---------|
| `ORIGINAL` | Direct extract from gstack. Content is unchanged from source. |
| `ADAPTED` | Based on a gstack source, but modified. See `changes:` frontmatter in the file. |
| `CUSTOM` | New file — no gstack source. Written from scratch. |

---

## Source Files (`source/`)

Original extracts from gstack. **Do not edit these** — they are the reference baseline. To change methodology, edit the corresponding `adapted/` file instead. When gstack releases updates, diff the new gstack version against these files to decide what to pull in.

| File | gstack Source | Status | Notes |
|------|--------------|--------|-------|
| `source/ethos.md` | `gstack/ETHOS.md` | `ORIGINAL` | Full extract. Core builder philosophy. |
| `source/office-hours.md` | `gstack/office-hours/SKILL.md.tmpl` | `ORIGINAL` | Partial — builder mode only. Startup diagnostic omitted. |
| `source/plan-eng-review.md` | `gstack/plan-eng-review/SKILL.md.tmpl` | `ORIGINAL` | Full extract. `{{PLACEHOLDERS}}` kept as markers. |
| `source/review.md` | `gstack/review/SKILL.md.tmpl` | `ORIGINAL` | Full extract. `{{PLACEHOLDERS}}` kept as markers. |
| `source/ship.md` | `gstack/ship/SKILL.md.tmpl` | `ORIGINAL` | Full extract. `{{PLACEHOLDERS}}` kept as markers. |
| `source/autoplan-principles.md` | `gstack/autoplan/SKILL.md.tmpl` | `ORIGINAL` | Partial — 6 decision principles and classification only. |
| `source/investigate.md` | `gstack/investigate/SKILL.md.tmpl` | `ORIGINAL` | Full extract. gstack freeze hook reference preserved. |
| `source/review-checklist.md` | `gstack/review/checklist.md` | `ORIGINAL` | Full extract. Unmodified. |
| `source/review-design-checklist.md` | `gstack/review/design-checklist.md` | `ORIGINAL` | Full extract. Unmodified. |
| `source/review-specialists-security.md` | `gstack/review/specialists/security.md` | `ORIGINAL` | Full extract. Unmodified. |
| `source/review-specialists-testing.md` | `gstack/review/specialists/testing.md` | `ORIGINAL` | Full extract. Unmodified. |

---

## Adapted Files (`adapted/`)

Working files used by the skill. Edit these freely — changes take effect immediately.

| File | Adapted From | Status | Key Changes |
|------|-------------|--------|-------------|
| `adapted/feature-interrogation.md` | `source/office-hours.md` | `ADAPTED` | Builder mode only; no gstack bins; added stack detection; simplified design doc path |
| `adapted/feature-plan.md` | `source/plan-eng-review.md` | `ADAPTED` | No gstack bins; unit tests only; Node/React/RN/Swift/Kotlin checks; **added error mapping, shadow path testing, observability checklist** |
| `adapted/feature-investigate.md` | `source/investigate.md` | `ADAPTED` | No gstack freeze mechanism; added stack-specific patterns and debug commands; kept Iron Law + 5 phases |
| `adapted/feature-review.md` | `source/review.md` | `ADAPTED` | No Greptile; no adversarial step; no gstack bins; references feature-review-checklist.md and feature-design-check.md |
| `adapted/feature-review-checklist.md` | `source/review-checklist.md` + `source/review-specialists-security.md` + `source/review-specialists-testing.md` | `ADAPTED` | Rails/Python refs replaced with Node/React/RN/Swift/Kotlin; specialists merged inline; Fix-First heuristic unchanged |
| `adapted/feature-design-check.md` | `source/review-design-checklist.md` | `ADAPTED` | No gstack-diff-scope; added React Native specific checks; added Swift/SwiftUI checks; added missing UI states category |
| `adapted/feature-ship.md` | `source/ship.md` | `ADAPTED` | No gstack bins; no eval suites; no gstack metrics; detects test command dynamically; simplified CHANGELOG handling |

---

## Skill Files (`.claude/commands/`)

The Claude Code custom command. No gstack source — written from scratch to orchestrate all adapted files.

| File | Status | Description |
|------|--------|-------------|
| `.claude/commands/autofeature.md` | `CUSTOM` | Main orchestration. Reads adapted files at runtime. Supports automated + checkpoint modes. Includes: ETHOS principles, checkpoint/resume, investigate invocation on test failure, design check. |

---

## How to Make Changes

**To tweak a methodology (e.g., change how planning works):**
→ Edit the relevant `adapted/` file

**To change the orchestration pipeline:**
→ Edit `.claude/commands/autofeature.md`

**To add a new tech stack:**
→ Add stack-specific sections to each `adapted/` file that currently has stack sections

**To sync with upstream gstack changes:**
1. Re-extract the relevant `source/` file from the new gstack version
2. Diff it against the current `source/` file
3. Apply relevant changes to the corresponding `adapted/` file manually
4. Update the `extracted:` date in the source file's frontmatter
5. Update this MANIFEST if anything changed structurally

---

## What Was Removed from gstack Sources

Intentionally excluded from adapted files:

| Feature | Reason |
|---------|--------|
| gstack bin references (`gstack-slug`, `gstack-review-log`, etc.) | Requires gstack installed |
| `{{PLACEHOLDERS}}` (PREAMBLE, BROWSE_SETUP, etc.) | Requires gstack template compiler |
| Greptile PR review integration | External service dependency |
| Eval suites (LLM-as-judge) | gstack-specific test infrastructure |
| Review dashboard | Requires gstack state files |
| Adversarial review (specialist subagents) | gstack multi-agent infrastructure |
| Learnings search/log (`~/.gstack/` state) | gstack state directory |
| Document-release auto-invoke | Separate gstack skill dependency |
| Worktree parallelization | gstack parallel agent infrastructure |
| VERSION file management (4-digit format) | gstack versioning convention |
| gstack metrics logging | gstack analytics |
| Startup diagnostic mode (office-hours) | Not needed for feature building |
| Codex outside voice | Requires OpenAI Codex CLI |
| Design review phases (`/plan-design-review`) | Out of scope |
| Browser QA (`/qa`) | Unit tests only |
| Plan-CEO review (scope modes, 11 sections) | Too heavy for this use case — error mapping and shadow paths extracted instead |
| gstack freeze/hook mechanism | Requires gstack hooks in settings.json |
| Retro / metrics tracking | Separate concern |
| Document release | Separate concern |

---

## What Was Added Beyond gstack Sources

Additions not in any gstack source:

| Addition | File | Purpose |
|----------|------|---------|
| Checkpoint/resume capability | `.claude/commands/autofeature.md` | Resume interrupted feature builds |
| React Native specific design checks | `adapted/feature-design-check.md` | RN has different design concerns than web |
| Swift/SwiftUI design checks | `adapted/feature-design-check.md` | iOS-specific accessibility + layout patterns |
| Missing UI states category | `adapted/feature-design-check.md` | Loading/empty/error states are common omission |
| Stack-specific debug commands | `adapted/feature-investigate.md` | Cache clearing, simulator reset, Metro bundler |
| Methodology reference table | `.claude/commands/autofeature.md` | Single place to see which file does what |
