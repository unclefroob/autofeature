---
status: CUSTOM
description: Live acceptance-testing methodology for /autofeature:test. Derives its own test plan by reviewing what was built (Test Manifest + code + running surface — a plan is NOT given), then DRIVES the real app like a user — web in a browser (Chrome MCP), or a mobile app in a simulator/emulator (ios-simulator skill / adb + computer-use) — logging in with a URL + credentials, walking each flow, and reporting per-flow PASS/FAIL/BLOCKED with screenshots + console/network errors. Complements agents/test-runner.md (which runs HEADLESS unit/integration/e2e suites); this drives the LIVE app. Advisory — hands failures off to /autofeature, never auto-fixes.
---

# Feature Test — Live Acceptance Driver

Open the real running app, log in, and **use it like a user would** — then report what works and
what's broken. The skill **builds its own test plan by reviewing what was built** (no plan is handed
to it), drives the live surface across web / iOS / Android, and produces a per-flow report.

```
resolve target & scope → derive the test plan (review what was done) → drive the live app → report → offer fixes
```

**This is not the headless suite.** `agents/test-runner.md` runs `npm test` / Playwright / xcodebuild
and summarizes logs — keep using it for that (autofeature Step 8). **This** command exercises a
*deployed/running* product through its real UI and network, the way manual QA does. The two are
complementary; this one never runs the unit suite.

**This is not a product audit.** `/autofeature:product-review` reasons about gaps from the code.
**This** actually clicks the buttons.

---

## Step 1: Resolve target & scope

Parse from the command arguments (all optional — ask only when genuinely ambiguous):

| Intent | Args (any form) | Default if absent |
|--------|-----------------|-------------------|
| **Platform** | `web` / `ios` / `android` | Auto-detect from the repo stack (see below); ask if more than one applies and scope spans both |
| **Target** | `url: https://…` (web); `app:`/`scheme:`/`build:` or a local build (mobile) | Web: a detectable local dev server (`vite`/`next`/`expo` port); Mobile: build from the current repo |
| **Scope** | `whole`/`product`; `branch` (what changed on this branch); named flows/features; `manifest: <path>` | If a manifest exists for this feature/branch → offer it; else ask: whole-product vs branch-diff vs named |
| **Credentials** | `creds: user/pass`, `login: …` | A gitignored `.autofeature/test-credentials.local`, else env vars, else ask (see safety below) |

**Stack auto-detect (platform):**
```bash
# web signal
grep -Eq '"(react|vite|next|react-dom)"' package.json 2>/dev/null && echo WEB
# mobile signals
{ [ -f app.json ] || [ -f app.config.js ] || [ -d ios ] || [ -d android ]; } && echo MOBILE
ls ios/*.xcworkspace ios/*.xcodeproj 2>/dev/null && echo IOS
[ -d android ] && echo ANDROID
```
If the feature is reachable on multiple platforms and scope doesn't pin one, ask which to drive (or
offer to drive each in turn).

Echo what you're about to do so the user can correct course:
> Testing **[scope]** on **[platform]** at **[target]**. Building the test plan from what was
> built… (no plan needed from you.)

### Credential safety — non-negotiable

- Accept credentials inline (`creds:`), from a **gitignored** `.autofeature/test-credentials.local`,
  or from env vars. Prefer the local file for repeat runs.
- **Never** echo, log, screenshot, write into the report, or commit a credential. Redact to
  `••••` in every surface, including error messages and the saved report.
- If a `.autofeature/test-credentials.local` is used, confirm it is gitignored:
  ```bash
  git check-ignore .autofeature/test-credentials.local >/dev/null 2>&1 && echo IGNORED || echo NOT_IGNORED
  ```
  If `NOT_IGNORED`: **STOP** and tell the user to add it (offer to append
  `.autofeature/test-credentials.local` and `.autofeature/test-runs/` to `.gitignore`). Do not read a
  non-ignored creds file — it risks committing secrets.
- If no credentials are available and a flow needs auth, mark those flows **BLOCKED (no credentials)**
  rather than guessing.

---

## Step 2: Build the test plan by reviewing what was done

The plan is **derived, not given.** Produce an ordered list of flows to drive.

### 2a. Manifest as spine (if one exists)
```bash
# newest manifest for this feature/branch, unless `manifest:` named one
ls -t .autofeature/tests/*.md 2>/dev/null | head -5
```
If a Test Manifest exists (see `feature-test-manifest.md`), its **Acceptance flows (AF-N)** are the
spine of the plan, and its **Setup** tells you the credentials/seed/env needed. Honor its **Out of
scope** — do not flag intended gaps as failures.

### 2b. Review what's actually there (always — manifest or not)
Delegate the read-heavy surface mapping to an **`Explore`** subagent so it doesn't fill main-loop
context. Drive the browser/simulator from the **main loop** (the live MCP session lives here) — only
the *reading* is delegated.

```
Agent({
  description: "Test surface map",
  subagent_type: "Explore",
  model: "haiku",   // surface map — orchestrator/model-tiers.md
  prompt: "Map the user-facing, testable surface of this app for acceptance testing.
  Repo: [pwd]. Scope: [whole product | changes on branch <name> vs <base> | features: <list>].
  Return (paths + one-line role, no large excerpts):
  1. Web routes/pages (router config, page components) reachable for the scope.
  2. API endpoints the scope exercises (method + path + auth requirement).
  3. Mobile screens + nav paths (navigator config) for the scope.
  4. The login/auth flow: where the login screen/route is and the field selectors or labels.
  5. Any seed-data or feature-flag preconditions implied by the code.
  6. For 'branch' scope: the user-visible behavior added/changed (read `git diff --stat <base>...HEAD`).
  Be concise — a structured surface map, under 1200 words."
})
```
For an **arbitrary URL with no repo**, skip Explore and derive the surface live: navigate the app,
read links/DOM, enumerate reachable pages.

### 2c. Synthesize the plan
Combine the manifest spine + surface map into an ordered flow list, each with: `id`, preconditions
(auth/role/seed), steps, expected result, priority (`critical` first). Filter to the requested scope.
If no manifest existed, you have just generated one on-demand — save it to
`.autofeature/test-runs/<ts>/derived-manifest.md` (format per `feature-test-manifest.md`) and offer to
promote it to `.autofeature/tests/` afterward.

**Show the plan first** (a tight list — flow ids + one-line each), then drive. In automated use,
proceed; if the user is present and the list is long, a quick "drive all N, or a subset?" is fine.

---

## Step 3: Drive the live app

Load the driving tools for the chosen platform via ToolSearch (they're deferred), then drive from the
main loop. Capture evidence with discipline (below). Per flow, record a verdict:

- **PASS** — every expected result observed; no new console errors / failed network calls in the flow.
- **FAIL** — an expected result missing/wrong, an error surfaced, a request 4xx/5xx that shouldn't be,
  or a JS exception during the flow.
- **BLOCKED** — couldn't run (no credentials, app won't boot, dependency down). Not a pass or fail.

### Web — Chrome MCP

Load: `ToolSearch({ query: "claude-in-chrome", max_results: 30 })`. Key tools:
`list_connected_browsers`, `navigate`, `find`, `read_page` / `get_page_text`, `form_input`,
`computer` (click/type/screenshot), `file_upload`, `read_console_messages`, `read_network_requests`,
`tabs_create_mcp`.

1. **Connect:** `list_connected_browsers`. If none, ask the user to open Chrome with the Claude
   extension (do **not** fall back to desktop computer-use for a browser — it's read-only there).
2. **Login once:** `navigate` to the app/login URL → `find`/`read_page` to locate the username &
   password fields → `form_input` to fill (redacted) → submit (click the button via `computer` or
   `form_input` submit). Verify you're authenticated (look for a logged-in marker) before walking
   flows. If login fails → mark all auth-required flows **BLOCKED (login failed)** and say so.
3. **Walk each flow:** perform the steps (`navigate` / `find` + `computer` click / `form_input` /
   `file_upload`), then assert the expected visible state via `read_page` / `get_page_text` / `find`.
4. **Capture errors continuously:** after each flow, pull `read_console_messages` (JS errors,
   warnings) and `read_network_requests` (any 4xx/5xx, failed requests). A clean-looking UI with a
   500 in the network log is a **FAIL**.
5. **Screenshot** at each flow's key checkpoint and on any failure (`computer` screenshot action).

> Local build with no running server? Use the **Claude Preview** MCP
> (`ToolSearch({ query: "preview", max_results: 30 })` → `preview_start` on the dev command, then
> `preview_navigate`/`preview_click`/`preview_fill`/`preview_screenshot`/`preview_console_logs`/
> `preview_network`) instead of Chrome MCP.

### iOS — simulator + computer-use

Lifecycle via `xcrun simctl` (or the **`ios-simulator`** skill if available —
`Skill({ skill: "ios-simulator", args: "boot + install + launch <build> on <device>" })` — it wraps
these); **input** via computer-use on the Simulator window.

1. **Boot + launch the app:**
   ```bash
   xcrun simctl list devices available | grep -i iphone        # pick a booted/available device
   xcrun simctl boot "iPhone 15" 2>/dev/null; open -a Simulator
   # Expo:        npx expo run:ios            (or press i against a running `expo start`)
   # bare RN:     npx react-native run-ios
   # native:      xcodebuild -scheme <S> -destination 'platform=iOS Simulator,name=iPhone 15' build
   #              && xcrun simctl install booted <built.app> && xcrun simctl launch booted <bundle-id>
   ```
   If the app won't build/launch → flows are **BLOCKED (app would not launch)**; capture the build
   error tail (not the full log).
2. **Grant input access:** computer-use needs the Simulator —
   `ToolSearch({ query: "computer-use", max_results: 30 })` then `request_access` for **Simulator**
   (native app → full tier: clicks + typing allowed).
3. **Drive:** `screenshot` → locate the control → `left_click` at its coordinates → `type` for text
   fields → `scroll` for lists. Re-`screenshot` to confirm each expected state. (If `idb` or Maestro
   is installed, you may use them for taps instead — note which you used.)
4. **Logs:** `xcrun simctl spawn booted log stream --level=error` (or the Metro/Expo console for RN)
   — extract only the failing lines for the flow.

### Android — emulator + computer-use

1. **Boot + install:**
   ```bash
   emulator -list-avds && emulator -avd <avd> -no-snapshot &      # or rely on a running emulator
   adb wait-for-device
   # Expo:    npx expo run:android
   # bare RN: npx react-native run-android
   # apk:     adb install -r <app>.apk && adb shell am start -n <pkg>/<activity>
   ```
   Won't build/launch → **BLOCKED**; capture the error tail.
2. **Drive:** computer-use `request_access` for the emulator window, then `screenshot` →
   `left_click` / `type` / `scroll`. (Or `adb shell input tap x y` / `input text` as an alternative —
   note which you used.)
3. **Logs:** `adb logcat -d *:E` (errors only) around each flow — extract the relevant lines.

### Evidence discipline (protects context)

- Save screenshots to `.autofeature/test-runs/<timestamp>/screenshots/` and reference them by **path**
  in the report. Inline (show) only the few that demonstrate a failure.
- **Never dump full logs.** Extract the specific failing console line / network entry / stack top
  (≤5 lines) per failure. This mirrors `test-runner.md`'s "keep logs out of context" rule.
- Bound any raw output with `tail`/`grep`.

---

## Step 4: Report

Write `.autofeature/test-runs/<timestamp>-report.md` and print the summary. Format:

```markdown
# Test Run — <feature/scope> · <platform> · <date>

**Target:** <url or build>   **Scope:** <whole | branch | flows>   **Manifest:** <path or "derived">
**Result:** N passed · M failed · K blocked

## Flows
| id | flow | verdict | evidence |
|----|------|---------|----------|
| AF-1 | Upload profile photo | ✅ PASS | screenshots/af-1-*.png |
| AF-2 | Reject oversized file | ❌ FAIL | screenshots/af-2-fail.png |
| AF-3 | Unauthorized blocked   | ⛔ BLOCKED (no creds) | — |

## Failures (detail)
### AF-2 — Reject oversized file · severity: high
- **Expected:** inline error naming the size limit; old avatar unchanged.
- **Observed:** 20MB file uploaded successfully; no validation.
- **Evidence:** screenshots/af-2-fail.png; network: `POST /api/users/42/avatar → 200`.
- **Repro:** login → /settings/profile → Change photo → pick 20MB jpg → Confirm.
- **Likely cause (optional):** missing server-side size check in the avatar controller.

## Console / network errors seen
- [flow] [error line — file:line if available]

## Notes
- [flake, env issues, anything BLOCKED and why]
```

Severity: **critical** (core flow broken / data loss / security) · **high** (feature broken, no
workaround) · **medium** (degraded) · **low** (cosmetic). Credentials are `••••` everywhere.

---

## Step 5: Triage & hand-off (offer, never auto-fix)

This command **reports**; fixing is `/autofeature`'s job (mirrors the product-review hand-off).
After the report:

- If failures exist, offer to spin the top ones into fix runs:
  > Found [N] failures. Want me to fix the top ones?
  > A) Build fixes — I'll run `/autofeature [skip-product-review] fix the following: …` for the
  >    critical/high failures (the `[skip-product-review]` marker prevents the pre-build product panel
  >    from re-running on a fix).
  > B) Create Trello cards for them (if Trello MCP is connected).
  > C) Just leave the report — I'll handle it.
- Pass each chosen failure to `/autofeature` with its repro steps + likely cause + evidence path so
  the build has full context.
- Never edit app code from this command.

---

## Escalation / hard stops

Stop and ask (AskUserQuestion) when:
1. **No credentials** and the scope is auth-gated (offer: provide creds / test only public flows / abort).
2. **App/site won't load or build** (offer: fix the launch / point me at a deployed URL / abort).
3. **A destructive action** is on the path (delete account, send money, send email/messages, place an
   order). **Never** perform it on a real/shared environment without explicit confirmation — prefer a
   throwaway/seed account. Treat email/message sends and payments as outward-facing per global rules.
4. **The browser extension / a simulator isn't available** for the chosen platform.

---

## Guardrails

- **Derive the plan; don't expect one.** Reviewing what was built is the first job, every run.
- **Drive in the main loop.** The browser/simulator MCP session is bound here; only delegate *reading*
  (the Explore surface map) to a subagent. Don't try to drive one live session from parallel agents.
- **Evidence or it didn't happen.** A PASS needs an observed expected state; a FAIL needs a screenshot
  or a captured error. No verdicts from assumption.
- **Secrets stay secret.** Redact credentials everywhere; refuse a non-gitignored creds file.
- **Report, don't fix.** Hand failures to `/autofeature`.
- **Right tool per surface.** Web → Chrome MCP (or Claude Preview for local builds); native sim/emulator
  → computer-use for input + simctl/adb for lifecycle. Don't pixel-hunt a web app, don't expect Chrome
  MCP to drive a simulator.

---

## File Reference

| File | Purpose |
|------|---------|
| `adapted/feature-test-manifest.md` | The Test Manifest format — the plan spine when present; how to derive one on-demand |
| `agents/test-runner.md` | The complementary **headless** suite runner (unit/integration/e2e) — not driven here |

| Want instead… | Use |
|---------------|-----|
| Run the unit/integration/e2e suite | (autofeature Step 8 / `agents/test-runner.md`) |
| Find product gaps from the code (no driving) | `/autofeature:product-review` |
| Build a fix for a failure | `/autofeature [skip-product-review] fix: <failure>` |
