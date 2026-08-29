---
name: test
description: |
  Acceptance-test a LIVE running app by driving it like a user — not the headless unit suite.
  Opens the browser (Chrome MCP) at a URL you give, or a mobile app in an iOS simulator / Android emulator, logs in with the credentials you provide, walks the flows, and reports per-flow PASS/FAIL/BLOCKED with screenshots + console/network errors.
  Builds its OWN test plan by reviewing what was built (the autofeature Test Manifest when present, plus the code + running surface) — a plan is NOT required from you. Tests the whole product or a subset of features.
  Advisory: hands failures off to /autofeature, never auto-fixes app code. Credentials are never logged, written to the report, or committed.
  Invoke as:
    /autofeature:test url: https://app.com creds: user@x.com/secret
    /autofeature:test ios                      → drive this repo's app in the iOS simulator
    /autofeature:test branch                   → test only what changed on the current branch
    /autofeature:test manifest: .autofeature/tests/<slug>-<date>.md
    /autofeature:test                          → ask what to test, then derive the plan
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebFetch
  - Agent
  - Skill
  - TaskCreate
  - TaskUpdate
  - Monitor
  - mcp__trello__get_card_details
  - mcp__trello__add_comment_to_card
---

# AutoFeature Test — Live Acceptance Driver

Drive the **real running app** and report what works and what's broken. This command **builds its own
test plan by reviewing what was built** (you don't hand it one), then opens the app — web in a
browser, or mobile in a simulator/emulator — logs in, walks each flow, and writes a per-flow report.

It complements, and does **not** replace, the headless suite (`agents/test-runner.md` / autofeature
Step 8). This one clicks the actual buttons.

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

## Step 1: Resolve target, scope & credentials

Read `$AUTOFEATURE_HOME/adapted/feature-test.md` and follow **Step 1**. Parse from the arguments
(all optional): platform (`web`/`ios`/`android`, auto-detected from the stack if absent), target
(`url:` / mobile build), scope (`whole` / `branch` / named flows / `manifest:`), and credentials
(`creds:` / a gitignored `.autofeature/test-credentials.local` / env).

**Credential safety is non-negotiable** (per feature-test.md): never echo, log, screenshot, write to
the report, or commit a credential; redact to `••••` everywhere; refuse a non-gitignored creds file.

**Credentials mint a session; they are never typed.** Chrome refuses password fields, so read
`$AUTOFEATURE_HOME/adapted/browser-session.md` and check its three preconditions here — non-production
target, fixture credentials only, no real person's password. A production target or a human's real
credential means rung 5 (the user authenticates in their own browser), not a blocked run.

Echo what you're about to test so the user can correct course before driving.

---

## Step 2: Preflight — confirm the driver is available

Confirm the right driving surface is connected for the chosen platform (these MCP tools are deferred —
load them via ToolSearch as feature-test.md Step 3 describes):

- **Web** → Claude-in-Chrome MCP: `ToolSearch({ query: "claude-in-chrome", max_results: 30 })`, then
  `list_connected_browsers`. If no browser is connected, ask the user to open Chrome with the Claude
  extension. (Local build with no server → Claude Preview MCP instead.) Don't degrade to desktop
  computer-use — it can't reach the page's JS, which is what the session ladder runs on.
- **iOS** → `xcrun simctl` available (and the `ios-simulator` skill if installed) for lifecycle +
  `ToolSearch({ query: "computer-use", max_results: 30 })` → `request_access` for **Simulator** for input.
- **Android** → `adb`/`emulator` on PATH + computer-use access for the emulator window.

If the required driver isn't available for the chosen platform, surface it and offer alternatives
(switch platform / point at a deployed URL / abort) — don't silently degrade to a wrong tool.

---

## Step 3: Derive the plan, drive, and report

Continue following `$AUTOFEATURE_HOME/adapted/feature-test.md`:

- **Step 2 (Build the plan):** review what was built — use the Test Manifest as the spine if one
  exists (`.autofeature/tests/*.md` — format in `feature-test-manifest.md`), and **always** also map
  the live surface (delegate the read-heavy mapping to an `Explore` subagent; drive from the main
  loop). Synthesize an ordered flow list filtered to the scope; show it, then drive.
- **Step 2.5 (Session):** for web, establish auth per `adapted/browser-session.md` — highest supported
  rung, then **both** verification signals before any flow runs. An unconfirmed session is
  `BLOCKED (session not established at rung N)` with the probe response, never a walked flow.
- **Step 3 (Drive):** per-platform playbook (Chrome MCP / simulator + computer-use / emulator + adb).
  Capture screenshots + console/network errors with the evidence discipline (save to
  `.autofeature/test-runs/<ts>/`, reference by path, never dump full logs).
- **Step 4 (Report):** write `.autofeature/test-runs/<ts>-report.md` and print the per-flow summary
  (PASS / FAIL / BLOCKED + severity + evidence + repro).

Do **not** redo the methodology here — relay what driving the app produced.

---

## Step 4: Hand off failures (offer — never auto-fix)

Per feature-test.md Step 5: offer to spin the top failures into `/autofeature [skip-product-review]
fix: …` runs (passing repro + evidence) and/or Trello cards. This command reports; building fixes is
`/autofeature`'s job. It never edits app code.

---

## Relationship to other commands

| Command | What it does |
|---------|--------------|
| `/autofeature:test` (this) | **Drives the live app** and reports per-flow PASS/FAIL — derives its own plan |
| `/autofeature` Step 8 / `agents/test-runner.md` | Runs the **headless** unit/integration/e2e suite |
| `/autofeature:ui-test` | **Web-only** browser driver for seeded fixtures — same session ladder, no simulators |
| `/autofeature:product-review` | Finds product gaps **from the code** (no driving) |
| `/autofeature` | Builds & ships a feature — and emits the Test Manifest this command consumes |

Natural flow: `/autofeature` (build → ships + writes the manifest) → `/autofeature:test` (drive it) →
`/autofeature fix:` (close any failures).

## File Reference

| File | Purpose |
|------|---------|
| `adapted/feature-test.md` | The methodology — drivers (web/iOS/Android), plan derivation, credential safety, evidence, report, hand-off |
| `adapted/browser-session.md` | Authenticating the browser **without typing a password** — preconditions, detection, rungs 0–5, two-signal verification |
| `adapted/feature-test-manifest.md` | The Test Manifest format — the plan spine when present; on-demand generation |
