---
name: ritchies
description: |
  Build a feature across the Ritchies employee-portal platform (API + web + iOS + Android) from a single prompt.
  A Ritchies-specialized wrapper over the standard autofeature pipeline: hard-wires the four repos, always scaffolds the API first as the contract source of truth, then fans out only the clients you pick (web / iOS / Android) against the frozen contract.
  Enforces the repos' deliberate conventions (no Retrofit/Koin on Android, no react-query on web, iOS-parity mobile, four-pillar RBAC, employee-number auth).
  Ingests Claude-design HTML exports by RENDERING them (Chrome MCP) into a Design Flow Map + screenshots, and verifies the built app against them.
  Pipeline: design-ingest → interrogate → scope-gate → plan → branch → API (contract) → pick clients → implement (parallel architects) → test → review → ship → design-parity.
  iOS verification hands off to a Mac session via /autofeature:ritchies-ios-test, and iOS is deliberately NOT merged to main on ship — that merge starts an App Store Connect build, so it is a release decision.
  Invoke as: /autofeature:ritchies [mode:] <feature description>
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
  - Agent
  - Workflow
  - Skill
  - TaskCreate
  - TaskUpdate
  - Monitor
  - mcp__trello__get_card_details
  - mcp__trello__get_card_checklists
  - mcp__trello__add_comment_to_card
  - mcp__chrome-devtools__new_page
  - mcp__chrome-devtools__navigate_page
  - mcp__chrome-devtools__resize_page
  - mcp__chrome-devtools__take_screenshot
  - mcp__chrome-devtools__take_snapshot
  - mcp__chrome-devtools__click
  - mcp__chrome-devtools__list_pages
  - mcp__chrome-devtools__close_page
---

# AutoFeature — Ritchies Platform Orchestrator

This runs the **standard autofeature pipeline** (`$AUTOFEATURE_HOME/.claude/commands/autofeature.md`) with Ritchies-specific overrides. **Read `autofeature.md` and follow it end to end**, applying the overrides in this file at the steps named below. Everything not overridden here (Builder Principles, Model Efficiency / model tiers, resume, interrogation, scope-gate, review, ship, test-manifest) behaves exactly as in the base command.

## $AUTOFEATURE_HOME

Resolve it the same way the base command does:
```bash
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```
If none resolves, abort with the base command's message.

## Ritchies context — load first

Before Step 2, **read `$AUTOFEATURE_HOME/ritchies/conventions.md` in full** and treat it as authoritative for every architect prompt below. Key facts it establishes:
- The platform is an **internal employee portal** (not a storefront). Domain today: auth/RBAC, announcements, chat, capability-gated dashboard.
- Auth is **8-digit employee number** + JWT access/refresh with single-flight 401 refresh. API base `https://ritchies-platform-api-dev.azurewebsites.net`, endpoints `/api/<domain>`, plain-JSON responses, `{ "error": … }` errors.
- Authorization = **four-pillar scoped RBAC**; **capabilities are the authority**, the 1-7 level is a preset/label. Clients gate on `GET /api/me/permissions`.
- **Announcements is the reference feature** in every repo — architects mirror its shape.

## New Step 1.5: Design Ingestion (run BEFORE interrogation)

Ritchies features arrive as **Claude-design HTML exports**, and the flows are only legible when the HTML is **rendered**, not read as text. **Read `$AUTOFEATURE_HOME/ritchies/design-ingest.md` and follow its Phase 1** before Step 2:

- **Ask every run:** "Path to the design export for this feature? (a file or the folder of `.dc.html` screens)". If the user provides nothing, note that ingest is skipped and proceed from their text description only.
- **Render, don't parse** (Chrome DevTools MCP): open each screen at a phone viewport, `take_screenshot` every frame, `take_snapshot` for structure, and `click` through interactive elements to reveal loading/empty/error/sheet states. Prefer the `- Standalone.html` variant; group role/level variants (`… Team Member` vs `… Manager`, `Levels 1-3` vs `4 plus`).
- **Emit the Design Flow Map** to `.autofeature/designs/<feature>-design-<date>.md` (+ screenshots in `design-shots/<feature>/`): ordered screens, per-screen states, the access-level/capability variant table, the data model the UI implies, and any **open design questions** (multiple options shown, ambiguous copy) — surface those to the user, never guess or silently pick between design variants.
- The Flow Map becomes the **UI source of truth**: feed it (and the screenshots) to every architect alongside `ritchies/conventions.md` in Step 5b/7b, and feed the **UI-implied fields** into the API design + `api-contract-broker` so the backend returns what the mockups show.

This runs even for API-only work when a design is provided (the mockups still pin the contract fields). Skip it cleanly when no design is given.

## Override — Step 4: Cross-Repo Detection (hard-wired, replaces the generic detector)

Do NOT run the generic `orchestrator/cross-repo-detect.md` — its suffix-stripping mis-links these repos. Instead resolve the four Ritchies repos by fixed name under the current repo's parent dir:

```bash
CWD=$(pwd); PARENT=$(dirname "$CWD")
API="$PARENT/ritchies-platform-api"
WEB="$PARENT/ritchies-platform-web"
IOS="$PARENT/ritchies-mobile"
AND="$PARENT/ritchies-android"
for r in "$API" "$WEB" "$IOS" "$AND"; do
  [ -d "$r/.git" ] && echo "found: $r" || echo "MISSING: $r"
done
```
If any is missing, tell the user which and ask whether to proceed with the ones present. This is always a **cross-repo** run (contract broker is active), even when only one client is chosen — the API is always in scope.

## Override — Step 5b: which architects, and the API-first + client-picker flow

**The API is always in scope and is scaffolded first** so its contract is the single source of truth. After the Plan subagent returns:

1. **Design the API slice first.** Spawn `express-mongo-architect` (+ `mongo-data-modeler` if there's real schema work) against `ritchies-platform-api`, in `design` mode. Feed it the Brief, the Plan, and the API section of `ritchies/conventions.md`. It appends its plan (models, **services**, controllers, routes, `/api/<domain>` endpoint shapes, capabilities) to the Feature Brief. **Enforce the required layering** from the conventions doc, overriding the architect's default "mirror the nearest controller" (several older controllers inline their logic and must not be copied):
   - **route → controller → service → model**, where the **service** (`src/services/<domain>Service.ts`) holds ALL business logic as an HTTP-agnostic function module (matching the `chat/service.ts` / `auth/resolver.ts` style — exported `async function`s, not a class), and the **controller** is thin (zod parse → resolve access → call service → shape response → `next`).
   - **Reuse before building:** the architect must first grep `src/services/`, `src/auth/`, `src/chat/` and REUSE existing services (`resolveAccess`/`can`/`summarizeAccess`, `recordAudit`/`commitCatalogChange`, `chat/service`, `membership`, `reads`, `hub`, `authTokens`, `teamMembership`) rather than re-implementing. Its design section must name which existing services it reuses.
   - **Extract shared logic:** anything used by 2+ domains goes into a reusable `src/services/*` module (e.g. a shared `format.ts` for the `initialsOf`/`relativeTime` helpers currently duplicated between `announcementsController` and `chat/service.ts`) — never a third copy.
   - Plan **two test layers**: fast `services/__tests__/<domain>Service.test.ts` unit tests for the logic, plus the supertest `controllers/__tests__/<domain>.routes.test.ts` for wiring + auth.

2. **Freeze the contract.** Run `api-contract-broker` (base Step 5c) over the API design so the endpoint request/response JSON shapes and capability/tile-key strings are pinned before any client is designed. Every client consumes these verbatim.

3. **Ask which clients to build.** Use `AskUserQuestion` (multiSelect) — "Which clients should this feature ship to?" with options **Web**, **iOS**, **Android** (none preselected; the user picks). If the feature is obviously client-specific from the prompt, still confirm. Record the choice in the Brief.

4. **Design the chosen clients in parallel** against the frozen contract, each in `design` mode, each fed the matching section of `ritchies/conventions.md`:
   | Client picked | Architect | Repo |
   |---|---|---|
   | Web | `react-architect` | `ritchies-platform-web` |
   | iOS | `swift-architect` | `ritchies-mobile` |
   | Android | `kotlin-compose-architect` (`$AUTOFEATURE_HOME/agents/kotlin-compose-architect.md`) | `ritchies-android` |

   Dispatch exactly like the base command: read the architect `.md`, inline it into an `Agent` spawn (`subagent_type: "general-purpose"`, `model:` per `orchestrator/model-tiers.md`), pass Brief path + Plan + repo root + "`api-contract-broker` active" + the conventions doc + **the Design Flow Map path + screenshots dir** (from Step 1.5, when a design was ingested). Send all chosen-client spawns in **one message** for parallelism.

**Parity rule:** when both iOS and Android are picked, tell each architect to mirror the other field-for-field (identical DTO keys, domain shapes, tile keys, capability strings). When both are built, they must consume the same contract the web client does.

## Override — Step 6: branches

Create the feature branch in the API repo and in each **chosen** client repo (same branch name), off each repo's `main`. Coordinate the ship (base Step 10b) across exactly those repos.

**`main` is not a resting place in this platform — it is a deploy trigger, and what it triggers differs per repo.** Know which one you are about to pull before you merge anything:

| Repo | Merging to `main` does | So merge when |
|---|---|---|
| `ritchies-platform-api` | **Auto-deploys to the dev App Service.** | The API is ready to be live on dev. This is usually what you want — clients cannot be tested against endpoints that are not deployed. |
| `ritchies-platform-web` | **Auto-deploys the Static Web App.** | Same. |
| `ritchies-mobile` (iOS) | **Kicks off an App Store Connect build.** | ⚠️ **ONLY when you actually intend to cut a build for App Store Connect.** Never as routine tidying-up at the end of a feature. |
| `ritchies-android` | Nothing automatic today. | Anytime. |

**iOS therefore does NOT follow the other repos' ship step.** Land the work on the feature branch, open the PR, and **leave it open**. Say so in the PR body and in the report: *"iOS is on `feature/<slug>` and deliberately not merged — merging `main` starts an App Store Connect build, so that merge is a release decision, not a ship step."*

Ask the user explicitly when a release is intended, and merge iOS only on a clear yes. An unwanted build is not catastrophic, but it burns pipeline time, produces a build number nobody asked for, and quietly turns "we finished a ticket" into "we cut a release" — a claim the rest of the team will act on.

The iOS pipeline is **`.github/workflows/ios-release.yml` in `ritchies-mobile`**: on every push to `main` a GitHub macOS runner regenerates the project with xcodegen, archives and signs an `.ipa`, and uploads it to **TestFlight** via App Store Connect. No Xcode Cloud, no fastlane. It deliberately does NOT run the unit tests — macOS runner minutes bill at ten times the Linux rate — so a merge ships whatever is on main without a test gate. Run `RitchiesTests` before merging, never after.

**Verify this by reading the repo, not by listing a directory from memory.** `ls .github/workflows` in the wrong working directory returns "No such file", which looks exactly like "there is no pipeline" and is how this was previously recorded backwards. Use `git ls-tree -r origin/main --name-only | grep github` after a `git fetch`, or `gh workflow list`; a stale `origin/main` ref will also lie to you.

**Before reusing a branch name, check it isn't a stale leftover.** The client repos often already have `feature/<slug>` pointing at a commit *older* than `main`. In each repo run `git rev-list --count main..<branch>`; `0` means it carries nothing, so `git branch -f <branch> main`. Do not `git checkout` onto it with a dirty tree — that produces a merge conflict in files the feature never touched.

## Override — Step 7b: implementation (per architect, parallel)

For the API and each chosen client, spawn the same architect in `implement` mode, all in one message. **Pass the Design Flow Map (`.autofeature/designs/<feature>-design-<date>.md`) + its screenshots to every UI architect** — they must build every screen, state, and access-level variant it enumerates, not just the happy path. Reinforce the per-repo guardrails from `ritchies/conventions.md`, especially:
- **API:** enforce **route → controller → service → model** — business logic in `src/services/<domain>Service.ts` (HTTP-agnostic function module), controllers stay thin (inline zod parse → resolve access → call service → shape response → `try/catch → next`). **Reuse existing services** (`auth/resolver`, `auth/audit`, `chat/service`, `announcementsService`, `services/format` — the backend-api-skills refactor landed, so import rather than re-inline) and **extract shared helpers** into `src/services/*` rather than duplicating. New capability keys need `catalog:sync` run per environment before any preset holds them. `errorHandler` stays last in `app.ts`; route gates via `requireCapabilityHeld`/`requireAnyCapabilityHeld`; scoped caps checked in service/handler via `can()`. **Any envelope that reports a limit or policy to clients must read the same persisted store the enforcement path reads** — a readout built from module constants will drift from what the server actually enforces, and any transport-level ceiling (multer `fileSize`) must be ≥ the largest value an admin can set. Write both the service unit test and the route supertest. CI gates on **typecheck + jest + build** — all must pass.
- **Web:** one `lib/<domain>.ts` (axios + types + helpers), page in `pages/app|admin/`, `<Route>` + `RequireCapability` in `App.tsx`, capability key in `lib/access.ts`. Follow the **settled-result idiom** (loading derived, no sync setState in effects — the lint rule is an error). Vitest exists: give new `lib/` logic a test. Accents via `lib/brand.ts` + `onBrand()`, marks via `components/Logo.tsx`.
- **iOS:** `Features/<Name>/` with `@Observable <Name>Store`, `(level, capabilities, onClose)` view, wire into `DashboardView` + `AccessModel`; `xcodegen generate` after adding files; Swift Testing decode test.
- **Android:** the `kotlin-compose-architect` grain — no Retrofit/Koin/repository/nav-graph; `remember { <Feature>Store(scope) }`; wire into `DashboardScreen` `when` + `AccessModel`; mirror iOS.

## Override — Step 8: verification (honest, per repo)

**DRIVE THE FEATURE YOU JUST BUILT. FIRST. BEFORE ANY REGRESSION SUITE.**

A build that compiles and a feature that works are different claims, and only the
second is what the ticket asked for. The failure this rule exists to stop is
specific and it has happened: a whole afternoon of Mac time went on re-running
twelve instrumented suites over cards that were ALREADY verified, while the
feature built that same day had never had a single screen opened on any client.
Regression on settled work is the cheapest, safest-feeling thing available and it
is almost never the most valuable. Order of work is:

1. **Drive the new feature on every client you built it for.** Open the screens.
   Fill the inputs. Press the buttons. Confirm the writes landed server-side.
2. Then the automated suites.
3. Then regression over untouched areas — and only if something suggests risk.

**"The API is not deployed" is NEVER a reason to skip step 1.** Run one locally.
It takes about two minutes and every repo already has what it needs:

```bash
# in-memory Mongo (mongodb-memory-server is already a dependency)
node .dev-mongo.mjs &        # MongoMemoryServer on :27077, prints MONGO_READY
# the API, with .env.local sourced for the mandatory JWT secrets
PORT=3100 MONGODB_URI=mongodb://127.0.0.1:27077/ritchies \
  CATALOG_SYNC_ON_BOOT=on \        # publishes any NEW capability key
  <FEATURE>_SEED_ON_BOOT=on \      # seeds the feature's own fixtures
  LEVEL_USERS_SEED_ON_BOOT=on SEED_LEVEL_PASSWORD=local1234 \
  npx tsx src/index.ts
```
Accounts are `11111111`…`77777777` / `local1234`. Point each client at it:
- **Android:** `./gradlew :app:assembleDebug -PapiBaseUrl=http://10.0.2.2:3100`
- **iOS:** launch argument `-api_base_url http://127.0.0.1:3100` — `APIConfig`
  reads that UserDefaults key first, so NO rebuild is needed, and the simulator
  reaches the host's localhost directly. DEBUG accepts plain http.
- **Web:** point vite at it and drive with Chrome DevTools MCP.

**A NEW CAPABILITY KEY MAKES ITS OWN FEATURE INVISIBLE UNTIL `catalog:sync` RUNS.**
So a missing tile is ambiguous. Resolve it by reading `GET /api/me/permissions`
BEFORE looking at any screen, never by reasoning backwards from a blank space.
And check the Edit-tiles CATALOGUE, not the dashboard grid — the grid is a
user-chosen subset (and is capped, so something may have to come off first), so
absence there is evidence of nothing.

**Confirm writes against the API, not the screen.** Count `/api/<domain>` rows
before and after. Two traps that have each cost real time: the on-screen keyboard
silently swallows taps on pinned action buttons, so a working button looks dead;
and coordinates from a UI dump go stale the moment focus changes the layout, so a
tap lands on the previous field. Both look exactly like product bugs. Before
reporting either, run a control — tap a control you know works and confirm focus
moved.

### The keyboard is a window, and it covers your buttons

Android here is **edge-to-edge**, so the window does NOT shrink when the soft
keyboard opens. A screen with a pinned action bar must say `imePadding()` on its
root, or those actions sit UNDERNEATH the keyboard — present in the semantics
tree, clickable, reporting exactly the bounds you would measure, and physically
behind an IME window that eats the tap. `LoginScreen`, `LockScreen` and the chat
composer already do it; copy them.

This shipped in Forms and the symptom was baffling from the outside: one form
would not submit while another did, from the same build. The difference was that
the failing form's first field takes TYPING, so filling it raises the keyboard,
while the working one is completed by TAPS and never raises it. **A form you fill
by typing was unsubmittable; a form you fill by tapping was fine.** Suspect this
whenever "the button does nothing" depends on which form, not which build.

The log line that settled it is the shape to look for: a tap at the button's own
centre recorded `onValue <field> = 'Ryan '`. The tap that should have submitted
typed a space into the field above.

**`adb shell input keyevent 111` (ESCAPE) does NOT close the IME.** Check with
`adb shell dumpsys input_method | grep mInputShown` — it still reports true.
`keyevent 4` (BACK) does close it. Assuming 111 worked turned every subsequent
dead tap into a mystery and cost most of a debugging session.

**"No error, no write" has more than one cause, and they look identical.**
On one screen it was a buried button; on another client, the same symptom was an
open select menu still eating the tap, so the field was never set and the submit
tap merely dismissed the menu. From the outside both are: tapped, nothing
happened, no message. Only an instrument separates them — a log line caught the
buried-button case typing a space into the field above. So do not diagnose this
symptom from the symptom; and before submitting, assert the CONTROLS hold what
you think you set (read a select's label back), not just that you tapped them.

**Every keyboard-dismissal trick is wrong in some way — verify, do not assume.**
`keyevent 111` does not close the IME on Android (`dumpsys input_method` still
says `mInputShown=true`); `keyevent 4` does. On iOS, swiping down over the form
to dismiss passes across the predictive-text bar, and a swipe that lands on a
suggestion ACCEPTS it — a field typed as "Ryan K" was stored as "Ryan K has ",
which reads exactly like a text-field defect. Use the Return key for single-line
fields and a navigation-bar tap for a textarea, then ASSERT the keyboard is
actually down and ASSERT the field still holds what you typed. A harness that
edits the user's data is worse than one that fails.

**Instrument before you tap again.** Blind tapping answers "did it work"; a
temporary `Log.d` in the handler answers "what actually happened", which is the
question. Add them, read `adb logcat`, remove them before committing. A
layout-level defect like this cannot be seen by any JVM test — the test has to be
instrumented, raise a real keyboard, and assert the control moved above it (and
assert the IME inset is non-zero, so a keyboard that fails to appear fails the
test instead of passing it vacuously).

### One rule, two copies, on opposite sides of a boundary

Three separate bugs in one day, all the same shape, and the shape is sharper than
"do not duplicate":

| The rule | Copy A | Copy B |
|---|---|---|
| which tiles the home grid shows | capabilities-changed path | server-seed path |
| what a Settings row is called | server label | client screen title |
| what a seeded user record holds | the CLI seeder | the in-process seeder |

In every case the copies sat **across a boundary that stopped anyone reading them
together** — two code paths, two repositories, two entry points. That is why
reviewing either side alone looks correct, and why fixing one copy feels
finished.

**The tell is a green test with broken behaviour.** Patch copy A, the test that
covers copy A passes, and the path that actually runs is still wrong. That
happened with the home grid (fixed on one path, still broken on relaunch), with
the labels (landing fixed, screens still saying the old thing), and with the
seeder (CLI patched, boot path untouched, and the new test failed while the code
looked right).

So when you fix something that has a name — an ordering, a label, a default —
**grep for the rule, not the file.** If it appears twice, extract one function
before fixing either. Two careful edits is the failure mode, not the remedy.

### Tap targets, and why a single tap never proves one

Two defects in one day came from the same place, and both looked like working
code and read as harness failures.

**A SwiftUI Button's hit region is its LABEL, not the frame around it.** Stretch
a Button with `.frame(maxWidth: .infinity, alignment: .leading)` and the text
sits hard left while the rest of the row belongs to nobody. `.contentShape`
applied to the Button from the OUTSIDE does not fix it — it does not re-describe
the label. The frames and the shape go ON THE LABEL:

```swift
Button { … } label: {
    Text("Choose a date")
        .frame(maxWidth: .infinity, alignment: .leading)
        .frame(minHeight: 44)
        .contentShape(Rectangle())
}
```

**Then check 44 in BOTH dimensions.** A control 44 tall and 30 wide, sitting
beside another control, gives away taps to its neighbour. Quieter than a dead
button, and therefore later to be reported.

**Never assert on user-facing text by searching for words you chose. Dump and
diff.** A search for "incorrect", "not valid", "wrong", "didn't match" and "did
not match" came back empty against a screen that plainly said *"That code is not
right, or it has expired."* The absence of your own vocabulary was reported as
the absence of a message, and a defect was filed against working code. Copy is
somebody else's choice — take a baseline of the screen's text, act, diff, and
read what actually appeared. That surfaces the wording without your having to
guess a single word of it.

**Do not assume a control's LABEL either, and read state BEFORE you act.** A
helper that looked for a control labelled exactly "Back" found nothing on a
screen whose exit says "Back to dashboard", and reported a dead end on a screen
that exits perfectly well — manufacturing, from the opposite direction, the exact
defect it had been sent to check. The same run read "which screen am I on" AFTER
tapping rather than before, so a transition that did happen printed as one that
did not. Enumerate the controls that are actually present, name the one you
tapped in the log, and sample position before acting as well as after.

**Re-resolve an element at the moment you use it.** A control captured through
`allElementsBoundByIndex` binds a POSITION, not a control. Typing into a field
re-rendered the sheet, the index shifted, and a tap intended for the send-code
button landed on an unrelated row. It was only noticed because the framework
reported "not hittable" against an identifier that had never been asked for.
Anything held across a re-render must be looked up again by predicate.

**A test's wait must EXCEED the app's own timeout, or the test manufactures the
failure it is looking for.** A harness waited 40s for a row while the client was
correctly waiting up to 120s for the response — so on a cold container the test
gave up first and reported a broken screen, while the app was behaving exactly as
designed. The arithmetic is the whole bug:

```
app request timeout   120s
harness wait           40s   <- fails first, every cold start
```

The trap is that it does not fail every run. It fails only when the server is
cold, which is precisely when a reader is primed to believe a cold-start
regression. So the instrument produces its most convincing false positive in the
one condition it was built to test.

When you raise a client timeout, find every wait that **gates on a SERVER
RESPONSE** and now sits under it. A wait on locally-rendered state is not in
scope however small it is — a wait for a button to appear after a local tap sits
under any network timeout and always will, correctly.

That qualifier is not pedantry. Run without it on this suite and you get 107 hits
out of 118 waits, a diff nobody should make and no reason to suspect you are
wrong. With it you get two, and both turn out to be the same assertion on the
same endpoint — which is the tell that it is one mistake made twice rather than
several independent numbers.

The discriminator: does the element's appearance require a network round trip
that has not already happened in this test? One that has not bitten yet is
usually one that runs after something else warmed the server, not one that is
safe.

**"No output" is never evidence on its own.** Five false leads in one night, all
producing nothing and all meaning something different: a tap outside the window,
an element tap that silently scrolled first, a disabled button mistaken for a
covered one, a stale index, and a matcher looking for words nobody wrote. Before
reporting an absence, establish that the instrument could have detected a
presence.

**Whether the keyboard covers a control depends on CONTENT LENGTH, not
construction.** Two sheets with identical code — primary button inside the scroll,
same modifiers — measured +24pt clear and −119pt covered, because one has four
short fields above its button and the other has a free-text area. So "this screen
matches one that is fine" proves nothing, and a screen that passes today fails
when somebody adds a field. Measure each one, and prefer pinning the primary in
`safeAreaInset(edge: .bottom)` (iOS) or `imePadding()` on a pinned bar (Android)
so the answer stops depending on how much text is above it.

**An ELEMENT tap cannot detect this at all.** XCUITest scrolls a control into
view before tapping it, so `element.tap()` passes on a build where the button is
buried — it proves the control is reachable AFTER scrolling, not that a finger
can reach it where it is drawn. Only a tap at an absolute window coordinate sees
the truth. Any keyboard test written with element taps is measuring nothing,
which is a large and quiet category.

**Read `isEnabled` BEFORE tapping, or you cannot tell two failures apart.** "The
button is disabled because the form is incomplete" and "the button is covered by
the keyboard" both look like a tap that did nothing. One is correct behaviour and
one is a defect. A single tap cannot separate them and the wrong one gets filed.

**FIRST CHECK THE CONTROL IS ON SCREEN. This produced a false defect.**

A control 63pt tall starting at y=864 in an 874pt window has NINE points visible.
Its frame is reported in full, `exists`, `isHittable` and `isEnabled` are all
true, and a normalised tap at dx=0.9 lands twenty-two points BELOW the bottom of
the window — outside the app entirely. Nothing in the accessibility tree says so.
The tell is the container: a row whose container is taller than the screen
(402x1242 against a 402x874 window) is inside a scroll view and may be below the
fold.

So a sweep helper must refuse to run, loudly, on a control not fully within the
window. Scroll it into view, re-read the frame, THEN sweep. Without that
precondition the sweep measures VISIBILITY and reports it as a hit region.

**A settle test cannot rescue this, and believing it can is the trap.** The
obvious hypothesis for "dead at one offset, alive at another" is an animation
read too early, so you poll until the tree stops moving. Here the tree never
moved at all — identical element counts on every poll of every run — and that
stability made the false defect look MORE credible. A stable measurement of the
wrong thing is stable. Confirming that nothing is changing does not confirm you
are looking at the right place.

It is also the same shape as the tap-target bug two sections down: the tree
describes something other than what a finger can touch. When a control reports
every property as healthy and still does nothing, stop reading properties and
establish where the tap physically landed.

**Sweep across a control, right side first.** Tapping one point yields a verdict;
sweeping yields a BOUNDARY, and the boundary is what names the cause. Start at
dx=0.9 and only continue if it fails: the far side is the part that breaks, so
one tap there proves the thing that was broken. Starting at dx=0.1 lands on the
words and passes on a build whose right half is still dead — which is exactly how
a half-fix went green here.

**The half-fix was worse than the original bug.** A control dead everywhere gets
reported the first day. A control that works where the developer taps and fails
where the user taps survives every green run until someone with a different thumb
finds it. When a fix turns a reliable failure into an intermittent one, that is a
regression even though more cases now pass.

**Accessibility data will not save you here.** `hittable=true` and `enabled=true`
describe the frame, not the label inside it. The tree agrees with the code and
both disagree with reality, so the only instrument that works is tapping and
watching — take a screenshot rather than continuing to reason about frames you
cannot see.

### Per repo

- **API:** run the jest suite (base test-runner) — it must be green; typecheck + build too (that's the CI gate).
- **Web:** `npm run typecheck` + `npm run lint` (0 errors) + `npm test` (vitest) + `vite build` — all four, locally; SWA CI only builds.
- **iOS: run `/autofeature:ritchies-ios-test`.** This machine has no Xcode and never will, so iOS verification is a hand-off to a Claude session on the Mac; that skill covers picking the right session (**by machine name, never by project name**), what to send, demanding the compile result before anything else, and the simulator pass. It ends in one of four verdicts and forbids warming one into another.

  If no Mac is reachable, fall back to `swiftc -typecheck` and report **"compile-checked only"** — never "tested on simulator". Be aware `swiftc` is blind to the project file, so a newly added file that never made it into the target still passes. Default to **"never compiled"** when no compiler ran at all; that is an accurate gap, and it is worth more than a confident guess.
- **Android:** this environment **can** build — try `:app:assembleDebug` with SDK + JDK17 + gradle-8.9 (see the architect's Verification section). If the toolchain isn't present, say "compile-checked only / NOT built." Never inflate.

## New Step 10.6: Design Parity (after ship, only if a design was ingested)

If Step 1.5 produced a Design Flow Map, run **Phase 2 of `$AUTOFEATURE_HOME/ritchies/design-ingest.md`** to verify the built app matches the design (reusing the `adapted/feature-test.md` / `/autofeature:test` machinery):
- Drive the built app at the access level(s) the design targets — web via Chrome DevTools MCP, iOS in the simulator, Android in the emulator — and `take_screenshot` of each screen/state the Flow Map listed.
- Compare each built screen against its `design-shots/<feature>/` reference: layout, the elements the Flow Map named, the correct per-level variant, and the states (empty/error/loading). Report **per-screen PASS / DIVERGES (what differs) / MISSING**.
- Advisory only — hand divergences back to the user or into a follow-up run; do not silently rewrite the app or the design to force a match. Brand/token exactness defers to the `ritchies-design` skill. Note honestly which platforms you could actually drive (e.g. iOS may be compile-only with no simulator).

## Everything else

Interrogation, scope-gate, product-review, planning, review fan-out (+ `security-review` on triggers, `simplify` after fixes), ship, PR bodies, and the test-manifest all run per `autofeature.md`. Design docs and checkpoints go in each touched repo's `.autofeature/designs/<slug>-<date>.md` and `.autofeature/checkpoints/`.

## Brand

The **friendliest-team badge already contains the RITCHIES wordmark inside its artwork**, so the two never share a surface — if a screen adopts the badge, the wordmark comes off that screen. It is a black disc (reads as a hole on brand blue) and its script stops resolving below ~56px. It stays a culture asset: sign-in heroes and splashes, full-bleed alpha, never clipped round or given a ring/backing.

For any UI surface, defer to the `ritchies-design` skill (`~/.claude/skills/ritchies-design/`) and reuse its tokens/primitives; the corporate style guide V15 is the final authority and every client is on `#0039A6` now. Follow `ritchies/conventions.md`'s Brand standard and UX state standard sections: sourced accents with computed foregrounds, wordmark/R-mark/badge used in their proper places, SOON treatment for unwired tiles, no invented counts, loading/failed/empty always distinguishable with a visible retry.
