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

### Logging in, when the browser won't type a password

Claude in Chrome refuses password fields. That is deliberate and it isn't negotiable, so a test whose
login step fills `#password` no longer gets past flow one — everything behind it reports BLOCKED.

`adapted/browser-session.md` replaces that step for every browser-driven command. The fix isn't to
defeat the refusal, it's to stop needing it: a session is a cookie or a token, and typing a password
is the slowest and most privileged way to get one. So the password never enters the browser's input
path at all. Where a fixture credential has to be exchanged, it goes over HTTP — from the shell, or a
same-origin `fetch` the page itself makes, which lands an httpOnly `Set-Cookie` in the real jar.

Six rungs, highest the app supports:

| | Rung | When |
|---|------|------|
| **0** | The seeder emits sessions with the fixtures | You seed your own test data — retires the rest |
| **1** | A dev-only `/__test/login` endpoint, 404-guarded | You control the API. The Playwright answer |
| **2** | Same-origin `fetch` login from the page | httpOnly cookies JS can't write |
| **3** | Token injected at the frontend's real storage key | The API returns a token |
| **4** | Storage-state replay, captured once per role | Auth is expensive or rate-limited |
| **5** | The human's own session — reuse it, or pause and ask | SSO, MFA, **and any real credential** |

Guarded on both ends. It refuses to run against production, against anything but fixture accounts, or
against a real person's password — those are rung 5, always. And it verifies twice before driving:
the API must name the user you intended *and* the DOM must show a marker that can't render
anonymously. A session that quietly didn't take is the expensive failure — every flow then fails for
the same wrong reason, which reads exactly like a catastrophically broken product.

## Driving the browser directly — `/autofeature:ui-test`

Web only, seeded-fixture-first. Same session ladder, none of the simulator machinery.

```
/autofeature:ui-test url: http://localhost:5173                    # detect roles, drive everything
/autofeature:ui-test url: http://localhost:5173 roles: manager,employee
/autofeature:ui-test branch                                        # only what changed
/autofeature:ui-test seed                                          # re-seed fixtures first
```

It seeds the users, mints a session per role, walks each role's flows as that role, and keeps one
flow that opens the real login screen and reports it BLOCKED — so "we never test login" can't hide
inside a green run.

---

## Shaping a backend — `/autofeature:backend-api-skills`

One command for the shape of a backend API, rather than only for features that go into one.

```
/autofeature:backend-api-skills                    # audit, then offer the fix
/autofeature:backend-api-skills audit              # report only, writes nothing
/autofeature:backend-api-skills create             # greenfield skeleton
/autofeature:backend-api-skills create invoices    # one correctly-shaped domain slice
/autofeature:backend-api-skills fix all            # behaviour-preserving refactor
```

The standard it enforces is `route → controller → service → model`, and its Iron Law is that
**business logic lives in a layer that does not know HTTP exists** — the compliance test being
whether every business rule can be exercised by a unit test that never builds a request. Around
that sit the layer contracts (explicit MAY / MUST NOT per layer), reuse-before-extract, the split
between route-level guards and resource-scoped authorization, and a two-tier test contract that
puts most coverage in fast service tests.

**It detects the stack rather than assuming one.** Express+Mongoose is the reference
implementation, but it reads the repo's actual framework, ORM, validator and test runner and maps
the four layers onto them — so the audit's tells are `prisma.` on a Prisma repo and
`getRepository(` on TypeORM, and ambiguous detection stops the run instead of guessing.

`fix` mode is gated on **identical-green**: the pre-existing route suite is the behaviour contract,
so every one of those tests must pass *unchanged* and the count must match. New service unit tests
add to the total; a pre-existing test that had to be edited means behaviour changed, and the fix
gets reverted rather than the test rewritten. No green baseline, no refactor.

`/autofeature:api-standards` still works — it is now a thin alias that forwards here with the
Ritchies profile pre-selected.

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
├── backend/                       ← the portable backend API standard
│   ├── api-standard.md            ← the layering law + the V1-V14 violation catalogue
│   ├── stack-profiles.md          ← detect the stack, map the four layers onto it
│   └── scaffold.md                ← create-mode blueprints (skeleton · domain slice)
└── .claude/commands/
    ├── autofeature.md             ← the Claude Code command
    ├── backend-api-skills.md      ← audit / create / fix a backend's shape
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
