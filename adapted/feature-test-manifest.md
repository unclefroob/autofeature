---
status: CUSTOM
description: Defines the Test Manifest — the structured, re-runnable acceptance spec that documents exactly what an autofeature run built (surfaces + acceptance flows + setup), so it can be tested later. Single source of truth for the manifest format, referenced by BOTH the autofeature ship step (producer, writes it at Step 10.5) and /autofeature:test (consumer, uses it as the test-plan spine; can also generate one on-demand for code built without autofeature).
---

# Feature Test Manifest

A **Test Manifest** is the answer to "what exactly was built, and how do I test it?" — a precise,
human- and machine-readable acceptance spec produced when a feature ships. It is the durable
hand-off between **building** (`/autofeature`) and **testing** (`/autofeature:test`).

- **Producer:** `/autofeature` and `/autofeature:fullrun` write one at ship time (autofeature.md
  Step 10.5).
- **Consumer:** `/autofeature:test` reads the newest (or a named) manifest and uses its acceptance
  flows as the **spine** of the test plan it derives. When no manifest exists (a product built
  without autofeature, or an arbitrary URL), `/autofeature:test` generates one on-demand using the
  same format — so the artifact exists either way.

**Location:** `.autofeature/tests/[slug]-[date].md` — same `[slug]-[date]` convention as the Feature
Brief in `.autofeature/designs/`. One manifest per shipped feature; they accumulate.

> A manifest is an **accelerator, not a contract `/autofeature:test` blindly trusts.** The tester
> always *also* looks at what's actually there (code + running surface) — the manifest tells it where
> to look first and what "correct" means, not what to skip.

---

## Why this exists (and what it is not)

The PR body already carries a freeform "Test plan" checklist (see `feature-ship.md` Step 9). The
manifest **generalizes that into a structured artifact** that survives the PR, is re-runnable, and is
precise enough for an automated driver to execute without guessing.

| It IS | It is NOT |
|-------|-----------|
| A per-feature acceptance spec (surfaces + flows + expected results) | A unit/integration test file (that's the repo's own test suite) |
| The plan spine for `/autofeature:test` | A test *runner* (it carries no commands to run a suite) |
| Scoped to **what this run built/changed** | A whole-product map (use `/autofeature:product-review` for that) |
| Plain markdown a human can read and edit | A rigid schema — extra prose is fine; the headed sections are what matter |

---

## Format

Required sections, in this order. Keep flows **observable from the running app** — each step is
something a user (or a driver) does, and each expected result is something visibly true afterward.

### 1. Header (frontmatter + title)

```markdown
---
kind: test-manifest
feature: <human feature name>
slug: <kebab-slug>            # matches the Feature Brief
date: <YYYY-MM-DD>
branch: feature/<slug>
pr: <PR URL or "">
brief: .autofeature/designs/<slug>-<date>.md
platforms: [web]              # any of: web, ios, android  (what this feature is reachable on)
scope: single-layer           # micro | single-layer | cross-stack | cross-repo
---

# Test Manifest — <feature name>
```

### 2. Setup (preconditions to test at all)

What a tester must have before any flow runs. **Never put real secrets here** — name the *kind* of
credential/role needed and where it comes from, not the value.

```markdown
## Setup

- **Environments:** local (`http://localhost:5173`) | deployed (`<url>` — preview/staging)
- **Credentials/roles needed:** e.g. "a logged-in standard user", "an admin user". Supplied at test
  time via `creds:` / a gitignored `.autofeature/test-credentials.local` / env — NOT stored here.
- **Seed data:** e.g. "at least one existing project owned by the test user".
- **Feature flags / config:** e.g. "`FEATURE_PHOTO_UPLOAD=true`".
- **Dependencies:** e.g. "API must be running for the web app to function".
```

### 3. Surfaces built (only what THIS run created or changed)

The concrete, addressable surface. Omit a subsection if the run didn't touch it.

```markdown
## Surfaces built

### Web routes / pages
| Route | Purpose | Auth |
|-------|---------|------|
| `/settings/profile` | Profile photo upload UI | required |

### API endpoints
| Method | Path | Auth | Request → Response |
|--------|------|------|--------------------|
| `POST` | `/api/users/:id/avatar` | bearer | multipart file → `{ url }` |

### Mobile screens
| Screen | Nav path | Purpose |
|--------|----------|---------|
| `EditProfile` | Profile → "Edit" | Pick + upload avatar |
```

### 4. Acceptance flows (the testable spine)

The heart of the manifest. Each flow is a numbered, observable journey. Cover the **golden path**
plus the **key edge/error paths** the feature must handle (the shadow paths from the plan:
nil / empty / invalid / unauthorized). Give each a stable `id` so reports and re-runs can reference it.

```markdown
## Acceptance flows

### AF-1 — Upload a profile photo (golden path) · priority: critical
- **Precondition:** logged in as a standard user, on `/settings/profile`.
- **Steps:**
  1. Click "Change photo".
  2. Choose a valid JPEG/PNG under the size limit.
  3. Confirm.
- **Expected:** the new avatar renders immediately; a success toast appears; on reload the avatar
  persists; `POST /api/users/:id/avatar` returns 200 with a `url`.

### AF-2 — Reject an oversized / wrong-type file (error path) · priority: normal
- **Precondition:** as AF-1.
- **Steps:** choose a 20MB file (or a `.txt`).
- **Expected:** inline validation error names the limit/type; no upload request is sent OR it returns
  4xx and the UI shows the error; the old avatar is unchanged.

### AF-3 — Unauthorized upload is blocked · priority: normal
- **Precondition:** logged out (or a different user's id).
- **Steps:** attempt the upload (or hit the endpoint directly).
- **Expected:** 401/403; no write occurs.
```

### 5. Out of scope / not covered

Explicit, so the tester does not flag an intended gap as a failure.

```markdown
## Out of scope / not covered

- Image cropping/rotation (deferred to fast-follow).
- Avatar CDN cache invalidation (handled separately).
```

---

## Generating the manifest (producer — autofeature ship, Step 10.5)

Write the manifest from material already gathered in the run — do **not** start a fresh
investigation:

1. **Source the facts:**
   - Feature name, slug, scope tier, platforms (STACK) → from the Feature Brief (`.autofeature/designs/[slug]-[date].md`).
   - Branch + PR URL → from the ship step.
   - Surfaces → from the diff and the architect plan sections in the brief: new/changed routes (web
     router files), endpoints (Express routes/controllers), screens (RN/Swift navigators). Run
     `git diff --stat origin/<base>...HEAD` and read the architect `## *Plan*` sections rather than
     re-deriving.
   - Flows → expand the brief's golden path + the plan's **shadow paths / error map** (nil / empty /
     invalid / unauthorized) into AF-1, AF-2, … . The PR "Test plan" checklist is the seed; make each
     item a full flow with preconditions + expected result.
   - Setup → from the brief's auth pattern, seed-data needs, and any feature flags introduced.
   - Out of scope → from anything the plan explicitly deferred to fast-follow / cut.
2. **Write** `.autofeature/tests/[slug]-[date].md` using the template below.
3. **Link it** from the PR body: under "Test plan", add `Manifest: .autofeature/tests/[slug]-[date].md
   — run \`/autofeature:test\` to execute these flows.`
4. **Scope:** emit for every run **except `micro`** (a one-file change rarely needs an acceptance
   spec — but still allowed if the change is user-facing). Cross-repo runs: write one manifest per
   repo touched, or a single manifest with a `### Surfaces built` block per repo — match how
   `coordination.md` split the work.

Keep it factual and tied to the diff. If the run built nothing user-observable (pure refactor),
write a short manifest whose flows assert "no behavior change" against the affected surfaces.

---

## Generating the manifest (consumer — `/autofeature:test`, on-demand)

When `/autofeature:test` finds no manifest for the target, it derives one with the same format from
what's actually there (this is the "review what was done" step, done live):

- **Has a repo:** the Explore surface-map (routes/endpoints/screens) + recent `git log`/diff become
  Surfaces; the tester proposes acceptance flows per surface (golden + obvious error paths) and
  confirms the list with the user before driving.
- **Arbitrary URL, no repo:** crawl the running app (navigate, read the DOM/links) to enumerate
  surfaces, then propose flows.

The on-demand manifest is saved alongside the run (`.autofeature/test-runs/<ts>/derived-manifest.md`)
and can be promoted to `.autofeature/tests/` if the user wants to keep it.

---

## Template (copy, fill, write to `.autofeature/tests/[slug]-[date].md`)

```markdown
---
kind: test-manifest
feature: 
slug: 
date: 
branch: 
pr: 
brief: 
platforms: []
scope: 
---

# Test Manifest — <feature>

## Setup
- **Environments:** 
- **Credentials/roles needed:** 
- **Seed data:** 
- **Feature flags / config:** 
- **Dependencies:** 

## Surfaces built
### Web routes / pages
| Route | Purpose | Auth |
|-------|---------|------|

### API endpoints
| Method | Path | Auth | Request → Response |
|--------|------|------|--------------------|

### Mobile screens
| Screen | Nav path | Purpose |
|--------|----------|---------|

## Acceptance flows

### AF-1 — <golden path> · priority: critical
- **Precondition:** 
- **Steps:**
  1. 
- **Expected:** 

### AF-2 — <key error/edge path> · priority: normal
- **Precondition:** 
- **Steps:**
  1. 
- **Expected:** 

## Out of scope / not covered
- 
```

---

## Guardrails

- **No secrets in the manifest.** Name the kind of credential/role, never the value. Credentials are
  supplied at test time and redacted from all output.
- **Observable expectations only.** Every "Expected" must be checkable from the running app (a visible
  state, a toast, a status code) — not "the function returns X" (that's a unit test).
- **Scoped to the run.** Document what this feature built/changed, not the whole product.
- **Stable ids.** `AF-N` ids are referenced by test reports and re-runs — don't renumber existing
  flows when adding new ones to a manifest.
