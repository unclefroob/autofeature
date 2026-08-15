# AutoFeature

Build a complete feature end-to-end from a single prompt.

**Pipeline:** interrogate → scope → product-review → plan → branch → implement → test → review → ship → test-manifest

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

### Phase 7.5: Test Manifest
Documents *exactly what was built* as a structured, re-runnable acceptance spec at
`.autofeature/tests/[slug]-[date].md` — surfaces built (routes/endpoints/screens), acceptance flows
(golden + error paths), setup, and what's out of scope. This is the precise hand-off from building to
testing, and the plan spine for `/autofeature:test`. Skipped for `micro` changes.

---

## Testing a running build — `/autofeature:test`

Acceptance-test the **live app** by driving it like a user — the complement to the headless unit
suite. It **builds its own test plan by reviewing what was built** (the Test Manifest above when
present, plus the code and the running surface — you don't hand it a plan), logs in with a URL +
credentials you give, walks the flows, and reports per-flow **PASS / FAIL / BLOCKED** with screenshots
and console/network errors.

- **Web** → drives a real browser (Claude-in-Chrome MCP)
- **iOS** → boots a simulator and drives it (ios-simulator skill + computer-use)
- **Android** → boots an emulator and drives it (adb + computer-use)

```
/autofeature:test url: https://app.com creds: user@x.com/secret   # whole product, web
/autofeature:test branch                                          # only what changed on this branch
/autofeature:test ios                                             # this repo's app in the iOS simulator
/autofeature:test manifest: .autofeature/tests/<slug>-<date>.md   # exactly the documented flows
```

Credentials are never logged, written to the report, or committed. Failures can be handed straight to
`/autofeature` to fix — this command only reports, it never edits app code.

---

## Mapping an award — `/autofeature:award-map`

Maps one Australian modern award into `rosterio-compliance-service` so it can be priced and
roster-checked, with every rule citing a clause and mechanically checkable against the award's own
words.

```
/autofeature:award-map MA000003
/autofeature:award-map MA000009 mode:automated
/autofeature:award-map MA000003 resume
```

It is **enumeration-first**, and the ordering is the whole point. Mapping MA000004 took 95 commits
because the axes along which gaps could hide were discovered one at a time, while the modelling was in
flight — a clause marked `partial` hid the late-trading span, and a clause marked `modelled` hid time
off instead of overtime pay. A word cannot bound a list; a count can.

So this command transcribes the clause list and per-clause sub-clause counts from the Commission's
consolidated PDF **first**, triages every clause against the closed rule vocabulary **second**, and
stands up the eight closure axes **before** authoring anything. Their first run reports almost
everything open, and that output is the work list — a denominator fixed before anyone reads the award
for rules, which every later file burns down.

Done is `npm run closure -- <CODE>` returning zero on all eight axes. Not an opinion, and not an
agent's answer to "are there gaps?", which is the question that used to have a new answer every time
it was asked.

**But closure proves coverage, not correctness.** It proves a clause was read and a rule is reachable —
it cannot prove a rule's day, time, priority or threshold says what the clause it cites actually says. A
transposed day passes every closure check cleanly. That gap is closed by a separate step,
`/autofeature:award-verify`, run automatically as Step 7.5: two agents independently re-derive each
predicate-bearing row from its clause text alone, blind to what shipped and to each other, and a third
argues the shipped row is wrong only where they disagree. It's re-runnable on its own too, scoped to
just the clauses a wage review or an award variation touched:

```
/autofeature:award-verify MA000003
/autofeature:award-verify MA000003 clauses: 15,15A
```

A `high`-confidence finding blocks. This raises confidence — it doesn't retire the risk, since
independent agents can still share a blind spot on an unusual clause, which is why the pipeline's third
hard checkpoint is a human reading the review pack before anything ships, not the closure run on its own.

**Neither closure nor verification is a subscription.** Both are point-in-time. If the FWC varies an
award after mapping, nothing here notices on its own. `/autofeature:award-drift` is the manual check:
re-fetches the award's text, diffs it clause-by-clause against what's stored, names exactly which rows
are affected, and flags when the pay database itself may need a fresh workbook — the text diff can't
see that side, since the workbooks have no fetch script at all. Run it when you have reason to suspect a
variation landed — it's meant to be triggered, not scheduled.

```
/autofeature:award-drift MA000003
```

When something has moved, **re-authoring never deletes the superseded row.** Every rule already carries
`operative_from`/`operative_to`, and the engine already resolves shifts against them, so a shift dated
before the variation is supposed to keep pricing against the old reading. The fix closes the old row's
window and adds a new one, both living in the same source file — a blind delete-and-reinsert would
silently break every historical or backpay calculation touching that clause.

**Merging the PR is not going live.** There's one deployed database, not a staging and a production
one, so the first time an award's rules load remotely, it's directly into what prices real shifts.
Promotion is its own gated step: connect to the deployed database, re-run `verify`, `award-verify` and
`closure` against it — a pass locally is not evidence of a pass remotely — and only after the user
confirms those results is the award considered live.

Four checkpoints stop the run even in `mode:automated` — the transcription, the triage, the sign-off
before shipping, and promotion itself — because they're the steps carrying judgement, deciding whether
the mapping is fit to be relied on, or crossing into something real customers price shifts against.
Reads `awards/gap-axes.md`, `awards/rule-tables.md`, `awards/service-conventions.md` and
`awards/verify-workflow.md`.

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
├── awards/                        ← award-mapping domain reference
│   ├── gap-axes.md                ← the eight closure axes; how a mapping finishes
│   ├── rule-tables.md             ← what the rule_* vocabulary can and cannot say
│   ├── service-conventions.md     ← rosterio-compliance-service conventions
│   └── verify-workflow.md         ← semantic verification: coverage vs. correctness
└── .claude/commands/
    ├── autofeature.md             ← the Claude Code command
    ├── award-map.md               ← award mapping
    ├── award-verify.md            ← semantic verification, standalone or as Step 7.5
    └── award-drift.md             ← manual check: has the award's text moved?
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

## Model efficiency

The pipeline fans out a lot — Explore, Plan, 1–3 architects, the test-runner, 3–5 parallel reviewers,
and the review/market Workflows (4–8 agents each). Every spawn is pinned to the cheapest capable model
tier so the whole fan-out doesn't inherit your (possibly Opus) session model:

- **Haiku** — mechanical work: test-runner, citation re-fetch, surface maps.
- **Sonnet** — the workhorse: context, planning, all architects (design + implement), code review, analysts.
- **Opus** — only the few highest-judgment steps: the market-review memo, the adversarial bear-case, cross-repo planning.

This is the **BALANCED** profile. Retune it (or switch to economy / quality-preserving, or pass a
per-run `model:` override) in `orchestrator/model-tiers.md`. Skills and the orchestrator's own loop
follow your session model — **run the command on Sonnet for the cheapest pass**; the deep-reasoning
steps still self-elevate to Opus via their pins.

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
