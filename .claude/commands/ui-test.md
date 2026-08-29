---
name: ui-test
description: |
  Drive a web app's UI in a real browser (Claude in Chrome) and report what works — built for apps whose test data the harness seeds itself.
  Gets an authenticated browser WITHOUT typing a password: Chrome refuses password fields, so this seeds the fixture users, mints their sessions out-of-band, injects them, and verifies identity two ways before touching a flow.
  Derives its own plan from what was built (Test Manifest when present, plus the code and the live surface), walks each flow as each role, and reports per-flow PASS/FAIL/BLOCKED with screenshots + console/network errors.
  Advisory: hands failures to /autofeature, never auto-fixes app code. Non-production targets and fixture credentials only.
  Invoke as:
    /autofeature:ui-test url: http://localhost:5173
    /autofeature:ui-test url: http://localhost:5173 roles: manager,employee
    /autofeature:ui-test branch                 → only what changed on this branch
    /autofeature:ui-test seed                   → re-seed fixtures first, then drive
    /autofeature:ui-test                        → detect the dev server, ask what to test
argument-hint: "[url: <url>] [roles: <a,b>] [branch|whole|<flows>] [seed] [manifest: <path>]"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Agent
  - Skill
  - ToolSearch
---

# AutoFeature UI Test — browser-driven acceptance on seeded data

Drive the **real running web app** in Chrome, as each role that matters, and report what works and
what's broken. This command builds its own plan; you don't hand it one.

It exists because the login step every browser test used to open with is gone. Claude in Chrome will
not type into a password field, so a run that starts by filling `#password` ends with every
auth-gated flow marked BLOCKED. This command starts from the other end instead: the harness seeds the
users, so it already knows who they are — the session is *minted*, never typed.

**Web only, and deliberately.** For iOS and Android use `/autofeature:test`, which drives simulators.

## $AUTOFEATURE_HOME

```bash
# Files ship with the plugin — prefer its root; fall back to an explicit home or dev clone.
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

If no candidate resolves (none contains `adapted/`), abort with:
`AutoFeature methodology files not found. They ship with the plugin — reinstall it, or for a dev clone set AUTOFEATURE_HOME=/path/to/autofeature.`

---

## Step 1: Resolve target, scope & roles

Parse from the arguments (all optional):

| Intent | Args | Default if absent |
|--------|------|-------------------|
| **Target** | `url: …` | Detect a running dev server (`vite`/`next`/`expo web` port); if none, offer to start it |
| **Scope** | `whole`/`product`, `branch`, named flows, `manifest: <path>` | A manifest for this branch if one exists; else ask |
| **Roles** | `roles: manager,employee` | Every role the seeder creates — read it, don't assume one |
| **Seeding** | `seed` (re-seed first) | Use existing data; re-seed only if verification says the fixtures are gone |

Then read `$AUTOFEATURE_HOME/adapted/browser-session.md` and check its three **preconditions** —
non-production target, fixture credentials only, no real person's password. A production URL stops
here: say so, and offer to drive the user's own already-authenticated session (rung 5) instead.

Echo the target, scope and roles before driving, so the user can correct course.

---

## Step 2: Connect Chrome

Load the tools in **one** call — `ToolSearch({ query: "select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__find,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__read_console_messages,mcp__claude-in-chrome__read_network_requests,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp", max_results: 12 })` — then
`tabs_context_mcp` to see what exists before creating anything.

No browser connected → ask the user to open Chrome with the Claude extension. Do **not** degrade to
desktop computer-use; it can't reach the page's JS, which is what the session ladder runs on.

---

## Step 3: Seed the fixtures

If the repo seeds its own test data, this is where the run gets reproducible. Find the seeder (
`browser-session.md` detection step 2), read what roles and fixture credentials it creates, and run it
when `seed` was passed or when Step 4 finds the fixtures missing.

Prefer a seeder that can **emit sessions** (`--emit-sessions`) — that is rung 0, and it retires the
whole rest of the ladder. If it can't yet, note it in the report as a one-line improvement; it is
usually a small change and it makes every future run faster and less fragile.

Record the seed value. A report that names its seed is one somebody else can re-run.

---

## Step 4: Establish and verify a session per role

Follow `$AUTOFEATURE_HOME/adapted/browser-session.md` — detection, then the highest rung the app
supports, then **both** verification signals for every role: the API names the user you intended, and
the DOM shows a marker that cannot render anonymously.

**Verification failure stops the run.** Do not walk flows against a session you could not confirm —
report `BLOCKED (session not established at rung N)` with the probe response. Every flow failing for
the same wrong reason reads as a broken product and wastes the afternoon that disproves it.

Cache the resolved profile in `.autofeature/session-profile.json` (gitignored) so the next run skips
detection.

---

## Step 5: Derive the plan, drive, report

Follow `$AUTOFEATURE_HOME/adapted/feature-test.md` for plan derivation (**Step 2**), the flow-walking
discipline (**Step 3**, web section), evidence handling and the report format (**Step 4**) — with two
changes this command owns:

- **The login flow is a flow.** Keep one flow that opens the real login screen with the real fixture
  user, and report it `BLOCKED (password field — Chrome guardrail)`. That is honest, it is one line,
  and it stops "we never test login" from hiding inside a green run. Everything else runs on a minted
  session.
- **Walk each role's flows as that role**, and re-verify identity when switching. A manager-scoped
  assertion that silently ran as an employee surfaces as a permissions bug that isn't real.

After each flow pull `read_console_messages` and `read_network_requests`. A clean-looking screen with
a `500` in the network log is a **FAIL**. Save evidence under `.autofeature/test-runs/<ts>/` and
reference it by path — never dump full logs into the report.

---

## Step 6: Clean up, then hand off

Delete the data the run created before reporting; leftover fixtures poison the next run's assertions.
Close every tab you opened, leave the user's own alone.

Then per `feature-test.md` **Step 5**: offer to spin the top failures into
`/autofeature [skip-product-review] fix: …` runs with repro + evidence attached. This command
reports. It never edits app code.

---

## Relationship to other commands

| Command | What it does |
|---------|--------------|
| `/autofeature:ui-test` (this) | **Browser-only**, seeded-fixture-first, session minted not typed. The Chrome driver |
| `/autofeature:test` | Same idea across **web + iOS + Android**; shares this session ladder for web |
| `/autofeature:award-ui` | Award-specific browser run — generates its cases from award thresholds |
| `agents/test-runner.md` | The **headless** unit/integration/e2e suite. No browser, no driving |
| `/autofeature:product-review` | Product gaps reasoned **from the code**. Never clicks anything |

## File Reference

| File | Purpose |
|------|---------|
| `adapted/browser-session.md` | **The session ladder** — preconditions, detection, rungs 0–5, two-signal verification, role handling, hygiene |
| `adapted/feature-test.md` | Plan derivation, flow-walking, evidence discipline, report format, failure hand-off |
| `adapted/feature-test-manifest.md` | The Test Manifest format — the plan spine when one exists |
