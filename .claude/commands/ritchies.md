---
name: ritchies
description: |
  Build a feature across the Ritchies employee-portal platform (API + web + iOS + Android) from a single prompt.
  A Ritchies-specialized wrapper over the standard autofeature pipeline: hard-wires the four repos, always scaffolds the API first as the contract source of truth, then fans out only the clients you pick (web / iOS / Android) against the frozen contract.
  Enforces the repos' deliberate conventions (no Retrofit/Koin on Android, no react-query on web, iOS-parity mobile, four-pillar RBAC, employee-number auth).
  Ingests Claude-design HTML exports by RENDERING them (Chrome MCP) into a Design Flow Map + screenshots, and verifies the built app against them.
  Pipeline: design-ingest → interrogate → scope-gate → plan → branch → API (contract) → pick clients → implement (parallel architects) → test → review → ship → design-parity.
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

**Before reusing a branch name, check it isn't a stale leftover.** The client repos often already have `feature/<slug>` pointing at a commit *older* than `main`. In each repo run `git rev-list --count main..<branch>`; `0` means it carries nothing, so `git branch -f <branch> main`. Do not `git checkout` onto it with a dirty tree — that produces a merge conflict in files the feature never touched.

## Override — Step 7b: implementation (per architect, parallel)

For the API and each chosen client, spawn the same architect in `implement` mode, all in one message. **Pass the Design Flow Map (`.autofeature/designs/<feature>-design-<date>.md`) + its screenshots to every UI architect** — they must build every screen, state, and access-level variant it enumerates, not just the happy path. Reinforce the per-repo guardrails from `ritchies/conventions.md`, especially:
- **API:** enforce **route → controller → service → model** — business logic in `src/services/<domain>Service.ts` (HTTP-agnostic function module), controllers stay thin (inline zod parse → resolve access → call service → shape response → `try/catch → next`). **Reuse existing services** (`auth/resolver`, `auth/audit`, `chat/service`, `announcementsService`, `services/format` — the backend-api-skills refactor landed, so import rather than re-inline) and **extract shared helpers** into `src/services/*` rather than duplicating. New capability keys need `catalog:sync` run per environment before any preset holds them. `errorHandler` stays last in `app.ts`; route gates via `requireCapabilityHeld`/`requireAnyCapabilityHeld`; scoped caps checked in service/handler via `can()`. **Any envelope that reports a limit or policy to clients must read the same persisted store the enforcement path reads** — a readout built from module constants will drift from what the server actually enforces, and any transport-level ceiling (multer `fileSize`) must be ≥ the largest value an admin can set. Write both the service unit test and the route supertest. CI gates on **typecheck + jest + build** — all must pass.
- **Web:** one `lib/<domain>.ts` (axios + types + helpers), page in `pages/app|admin/`, `<Route>` + `RequireCapability` in `App.tsx`, capability key in `lib/access.ts`. Follow the **settled-result idiom** (loading derived, no sync setState in effects — the lint rule is an error). Vitest exists: give new `lib/` logic a test. Accents via `lib/brand.ts` + `onBrand()`, marks via `components/Logo.tsx`.
- **iOS:** `Features/<Name>/` with `@Observable <Name>Store`, `(level, capabilities, onClose)` view, wire into `DashboardView` + `AccessModel`; `xcodegen generate` after adding files; Swift Testing decode test.
- **Android:** the `kotlin-compose-architect` grain — no Retrofit/Koin/repository/nav-graph; `remember { <Feature>Store(scope) }`; wire into `DashboardScreen` `when` + `AccessModel`; mirror iOS.

## Override — Step 8: verification (honest, per repo)

- **API:** run the jest suite (base test-runner) — it must be green; typecheck + build too (that's the CI gate).
- **Web:** `npm run typecheck` + `npm run lint` (0 errors) + `npm test` (vitest) + `vite build` — all four, locally; SWA CI only builds.
- **iOS:** prefer `xcodebuild`; if unavailable, `swiftc -typecheck` and report "compile-checked only." Never claim "tested on simulator" for a type-check. Note that adding/moving files needs a real Xcode build (`swiftc` is blind to the project file). **`ritchies-mobile` has never been compiled on this machine in its history** — the PR body must say "compile-checked only, never built" rather than anything that implies otherwise.
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
