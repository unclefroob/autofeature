---
name: ritchies-design-ingest
purpose: Turn Claude-design HTML exports into a rendered Design Flow Map + screenshots that drive the build, and verify the built app against them
status: CUSTOM
---

# Ritchies Design Ingestion

Ritchies features arrive as **Claude-design HTML exports**, and the real spec — the screens, the per-role/per-level variants, the states, the flow between them — is only legible when the HTML is **rendered**, not when the markup is read as text. These exports are large (often >1 MB), JS-driven, and use non-semantic markup, so grepping the DOM misses most of the flow. **Always render.**

Two phases: **Ingest** (front of the pipeline, produces the Design Flow Map) and **Parity** (after the build, verifies the app matches).

## What the exports look like

- Delivered as a folder (often an unzipped download) like `<something>/mobile-app-splash-screen-design/project/` containing many `*.dc.html` files.
- Each `*.dc.html` is the **editable canvas** format (heavy, includes editor runtime). Many have a sibling `<name> - Standalone.html` — the **clean rendered** version. **Prefer the `- Standalone.html` for rendering** when present; fall back to the `.dc.html`.
- One feature usually spans **several files**, including role/level variants: e.g. `Chat - Team Member`, `Chat Design B - Manager`, `Settings Phone Levels 1-3` vs `Settings Phone Levels 4 plus`, `FINAL Dashboard Access Levels`. `FINAL …` / `Working` generally supersede earlier drafts.
- Files are phone mockups — render at a phone viewport.

## Phase 1 — Ingest (before interrogation)

### 1. Get the path (ask every run)
Prompt: **"Path to the design export for this feature? (a file or the folder of `.dc.html` screens)"** Do not auto-scan. Accept a single file or a directory. If a directory, list the candidate screens and confirm which belong to this feature (prefer `FINAL`/`Working`, prefer `- Standalone.html`, drop obvious old drafts and `copy` duplicates). If the user gives nothing, tell them the ingest is skipped and the build proceeds from their text description only.

### 2. Render every screen (Chrome DevTools MCP)
For each selected file, using the `mcp__chrome-devtools__*` tools:
- `new_page` / `navigate_page` to the local `file://<absolute-path>` URL. (If a `file://` load is blocked or renders blank, serve the folder — `python3 -m http.server` in the export dir — and open `http://localhost:<port>/<file>`.)
- `resize_page` to a phone viewport (e.g. 390×844) since these are phone frames.
- `take_screenshot` (full page) → save to `.autofeature/designs/design-shots/<feature>/<screen-name>.png`.
- `take_snapshot` (accessibility tree) to capture the structure/labels/text for the Flow Map — this is your reliable read of the content, not a grep of the HTML.

### 3. Surface interaction-only flows
Static frames show one state; interactions reveal the rest. From the snapshot, enumerate interactive elements (tabs, buttons, toggles, list rows, FABs). For each meaningful one: `click` it, `take_screenshot` of the revealed state, note the transition, then return. Capture at minimum: default, and any loading / empty / error / expanded / sheet / modal states the design includes. Stop when clicks stop revealing new screens.

### 4. Read the variants
When multiple files are role/level variants of the same surface, render each and record **what differs by `AccessLevel` / capability** (which tiles, actions, sections appear at level 1 vs 4 vs 7). This maps directly onto the app's `AccessModel` / `TileCatalog` gating — capture it as a gate table, not prose.

### 5. Emit the Design Flow Map
Write `.autofeature/designs/<feature>-design-<date>.md`:

```markdown
# Design Flow Map — <feature>

Source files: [list, with which is authoritative]
Rendered: <N> screens, <M> interaction states (screenshots in design-shots/<feature>/)

## Screens (in flow order)
1. <ScreenName> — <one-line purpose> — ![](design-shots/<feature>/<screen>.png)
   - Key elements: [from snapshot — headers, lists, actions, inputs]
   - Reached from: <prev screen + trigger> ; leads to: <next screen + trigger>
2. ...

## States per screen
- <Screen>: default / loading / empty / error / <sheet|modal> — [which the design shows]

## Access-level / capability variants
| Surface / element | L1 | L2 | L4 | L7 | capability |
|---|---|---|---|---|---|
| <tile/action> | ✓/�— | ... | ... | ... | <cap key> |

## Data model implied by the UI
- <entity>: fields visible in the mockups → [name, type, source] — flag any the API must return

## Open design questions (do NOT guess — surface to the user)
- Multiple options present (e.g. "Chat Design A vs Design B") → which to build?
- Anything shown but ambiguous (placeholder copy, lorem, inconsistent states)
```

### 6. Hand-offs
- The Flow Map is the **UI source of truth** fed to every architect alongside `ritchies/conventions.md`. Screens/states/variants map to: web pages+routes, iOS `Features/<Name>/` views + `AccessModel`, Android `ui/<feature>/` screens + `AccessModel`.
- **Fields visible in the design that the API must supply** are contract inputs — feed them to the API design and `api-contract-broker` so the backend returns them.
- **Never invent flows the design doesn't show, and never silently pick between design options** — list them under Open design questions and ask.

## Phase 2 — Parity (after the build)

After ship, verify the built app renders the design's flows. Reuse the `adapted/feature-test.md` / `/autofeature:test` machinery:
- Drive the built app — web via Chrome DevTools MCP at the preview/local URL; iOS in the simulator / Android in the emulator — logged in at the access level(s) the design targets.
- Walk the same flow, `take_screenshot` of each screen/state captured in Phase 1.
- Compare each built screen against its `design-shots/` reference: layout, presence of the elements the Flow Map listed, the correct per-level variant, and the states (empty/error/loading). Report **per-screen PASS / DIVERGES (what differs) / MISSING**.
- Advisory only: hand divergences back to the user or into a follow-up run; do not silently rewrite the design or the app to force a match. Brand/token exactness defers to the `ritchies-design` skill.

## Notes
- The mockups are already brand-styled, but architects still take tokens from the `ritchies-design` skill (the brand-blue split `#002491` web vs `#0039A6` mobile still applies).
- Keep screenshots in `.autofeature/designs/design-shots/<feature>/` so the Flow Map, the PR, and the parity check all reference the same images.
