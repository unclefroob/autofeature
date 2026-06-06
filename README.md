# AutoFeature

Build a complete feature end-to-end from a single prompt.

**Pipeline:** interrogate → scope → product-review → plan → branch → implement → test → review → ship

**Supports:** Node.js, React, React Native, Swift, Kotlin

---

## Install

### As a plugin (recommended)

The commands read `adapted/`, `agents/`, and `orchestrator/` at runtime — these **ship with the
plugin** and resolve automatically from the plugin root (`$CLAUDE_PLUGIN_ROOT`). No path setup:

```
/plugin marketplace add unclefroob/autofeature
/plugin install autofeature@autofeature
```

### As a dev clone

Clone the repo; the commands prefer `$CLAUDE_PLUGIN_ROOT`, then `$AUTOFEATURE_HOME`, then
`~/dev/autofeature`:

```bash
git clone https://github.com/unclefroob/autofeature ~/dev/autofeature
# only if you cloned somewhere else:
export AUTOFEATURE_HOME=/path/to/autofeature
```

> Don't `cp` just the command file — it reads ~18 methodology/agent/orchestrator files at runtime, so
> the whole repo (or the installed plugin) must be present at the resolved root.

---

## Usage

```
/autofeature add user profile photo upload
/autofeature checkpoint: add OAuth login with Google
/autofeature automated: add rate limiting to the API
```

Or just:
```
/autofeature add push notifications for new messages
```
If no mode is specified, Claude will ask.

---

## Modes

| Mode | Behavior |
|------|----------|
| **Automated** | Runs straight through. Makes all decisions using the 6 decision principles. Produces a PR. You review at the end. |
| **Checkpoint** | Pauses after each phase for your approval: feature spec → implementation plan → code → ship. |

---

## What It Does

### Phase 1: Feature Interrogation
Reads the codebase, detects the tech stack, asks 6 focused questions (or answers them itself in automated mode). Produces a Feature Brief saved to `.autofeature/designs/`.

### Phase 1.5: Product Review (pre-build)
A CEO / PM / flow-walker pass over the product *before* any code is written. A multi-agent [Workflow](https://docs.claude.com/en/docs/claude-code) maps the product surface, fans out three product lenses in parallel, adversarially verifies any "broken flow" claim against the real code, and folds the resulting gaps into the plan. Skipped for `micro` changes. Also available standalone as `/autofeature:product-review`.

### Phase 2: Technical Planning
Engineering review of the planned approach. Checks architecture, code quality, unit test coverage, and performance. Produces an Implementation Plan.

### Phase 3: Branch Creation
Creates a `feature/[slug]` branch with an auto-generated name derived from the feature description.

### Phase 4: Implementation
Builds the feature following the plan. Reads existing conventions from CLAUDE.md and existing files. Writes clean, testable code.

### Phase 5: Unit Tests
Writes unit tests for all new code: happy paths, edge cases, error paths. Runs tests and fixes failures before proceeding.

### Phase 6: Pre-Ship Review
Reviews the diff for security issues, race conditions, type safety, and completeness gaps. Applies fixes automatically where possible.

### Phase 7: Ship
Merges main, runs final tests, creates bisectable commits, pushes, and opens a PR.

---

## File Structure

```
autofeature/
├── MANIFEST.md                    ← tracks edited vs original files
├── README.md                      ← this file
├── source/                        ← original gstack extracts (do not edit)
│   ├── office-hours.md
│   ├── plan-eng-review.md
│   ├── review.md
│   ├── ship.md
│   └── autoplan-principles.md
├── adapted/                       ← adapted versions (edit these)
│   ├── feature-interrogation.md
│   ├── feature-plan.md
│   ├── feature-review.md
│   └── feature-ship.md
└── .claude/commands/
    └── autofeature.md             ← the Claude Code command
```

**Rule:** Edit files in `adapted/` and `.claude/commands/`. Never edit `source/` files — they're the baseline for tracking what changed.

---

## Customizing

The skill reads the adapted files at runtime, so changes to them take effect immediately.

**To change how planning works:** edit `adapted/feature-plan.md`
**To add a new tech stack:** add a section to each `adapted/` file
**To change the pipeline:** edit `.claude/commands/autofeature.md`
**To change what phases checkpoint:** find the checkpoint gates in `autofeature.md`

See `MANIFEST.md` for a detailed breakdown of what was kept vs. removed from gstack.

---

## Requirements

- Claude Code (any version)
- `git` with a remote configured
- `gh` CLI for PR creation (falls back to manual instructions if not available)
- The project must have a test command (detected automatically or specified in CLAUDE.md)

No gstack installation required.

---

## Credits

Methodology adapted from [gstack](https://github.com/garrytan/gstack) by Garry Tan (MIT License). The planning, review, and ship workflows are derived from gstack's engineering review, pre-landing review, and ship skills.
