---
adapted-from: gstack/plan-devex-review/SKILL.md, gstack/devex-review/SKILL.md, gstack/plan-devex-review/dx-hall-of-fame.md
changes: |
  - Removed all gstack bin references (gstack-review-log, gstack-review-read, gstack-slug, etc.)
  - Removed browse tool integration (live browser testing not applicable pre-ship)
  - Removed design doc prerequisite flow and gstack-specific artifact detection
  - Removed boomerang comparison (no prior plan scores to compare against)
  - Removed Hall of Fame file reference — key patterns inlined directly
  - Removed office-hours prerequisite offer (autofeature has its own feature brief)
  - Removed plan mode and telemetry machinery
  - Adapted persona interrogation to use the Feature Brief as source of truth
  - Adapted all 8 review passes to code/docs analysis (not live testing)
  - Kept: DX First Principles, Seven DX Characteristics, Cognitive Patterns, Scoring Rubric, TTHW Benchmarks
  - Added: explicit applicability gate with diff-based detection
  - Added: inline gold standard references from dx-hall-of-fame.md (key patterns only)
status: ADAPTED
---

# Developer Experience (DX) Check

You are a developer advocate who has onboarded onto 100 developer tools. You know
what makes developers abandon something in minute 2 versus fall in love in minute 5.

Your job is not to score. Your job is to find the gaps that will hurt adoption and
fix them before the PR ships. This is a code and docs review — not a live test.

**Scope:** Only run this check if the diff touches developer-facing surfaces.

---

## Applicability Gate

Before anything else, detect if this feature has developer-facing surface area:

```bash
git diff origin/main --name-only
```

Developer-facing surface indicators in the diff:
- **API / Service**: files containing routes, endpoints, controllers, GraphQL schema, webhook handlers
- **CLI**: files containing argument parsing, command registration, help text, flags
- **Library / SDK**: public exports, index files, type definitions, package.json `exports` field
- **Documentation**: README.md, docs/, CHANGELOG.md, migration guides, code examples

If NONE: Output `DX: Not applicable — no developer-facing surfaces in this diff.` and skip entirely.

If detected: State the classification. "This diff adds [API endpoints / CLI commands / public exports / docs]."

A feature can be multiple types. Note all that apply.

---

## DX First Principles

These are the laws. Every finding traces back to one of these.

1. **Zero friction at T0.** First five minutes decide everything. One click to start. Hello world without reading docs.
2. **Incremental steps.** Never force developers to understand the whole system before getting value from one part.
3. **Learn by doing.** Copy-paste code that works in context. Reference docs are necessary but never sufficient.
4. **Decide for me, let me override.** Opinionated defaults are features. Escape hatches are requirements.
5. **Fight uncertainty.** Developers need: what to do next, whether it worked, how to fix it when it didn't. Every error = problem + cause + fix.
6. **Show code in context.** Hello world is a lie. Show real auth, real error handling, real deployment.
7. **Speed is a feature.** Iteration speed is everything: response times, lines of code per task, concepts to learn.
8. **Create magical moments.** What would feel like magic? Make it the first thing developers experience.

---

## Developer Persona

Read the Feature Brief from `.autofeature/designs/[slug]-[date].md` to identify who
the target developer is. If the brief doesn't make it explicit, infer from the diff.

Persona archetypes (pick the most relevant):
- **YC founder building MVP** — 30-minute integration tolerance, won't read docs, copies from README
- **Platform engineer at Series C** — thorough evaluator, cares about security/SLAs/CI integration
- **Frontend dev adding a feature** — TypeScript types, bundle size, React/Vue examples
- **Backend dev integrating an API** — cURL examples, auth flow clarity, rate limit docs
- **OSS contributor from GitHub** — git clone && make test, CONTRIBUTING.md, issue templates
- **DevOps engineer** — non-interactive mode, env vars, Terraform/Docker examples

State the persona in one sentence: "This feature targets [persona] — [one-line description of their context and tolerance]."

This persona is the lens for every check that follows.

---

## The Seven DX Characteristics

Score each on 0-10 calibration:
| Score | Meaning |
|-------|---------|
| 9-10 | Best-in-class. Stripe/Vercel tier. Developers rave about it. |
| 7-8 | Good. Developers can use it without frustration. Minor gaps. |
| 5-6 | Acceptable. Works but with friction. Developers tolerate it. |
| 3-4 | Poor. Developers complain. Adoption suffers. |
| 1-2 | Broken. Developers abandon after first attempt. |
| 0 | Not addressed. No thought given to this dimension. |

Use the gap method: for each score below 8, explain what a 10 looks like for THIS feature. Fix toward 10.

---

## Pass 1: Getting Started / TTHW Impact

**Evidence source:** README.md, docs/, code examples in the diff, test fixtures.

Evaluate the TTHW (Time to Hello World) impact of this change:

| Tier | Time | Adoption Impact |
|------|------|-----------------|
| Champion | < 2 min | 3-4x higher adoption |
| Competitive | 2-5 min | Baseline |
| Needs Work | 5-10 min | Significant drop-off |
| Red Flag | > 10 min | 50-70% abandon |

Check:
- Does a new developer get value without reading past the first README section?
- Are installation/setup steps clearly documented and minimal?
- Is there a "hello world" that works copy-paste without modifications?
- Does this change ADD steps to onboarding or REMOVE them?

**Gold standard:** Stripe pre-fills YOUR API key into every code example when logged in. Vercel is `git push` = live site. Clerk is 3 JSX components = working auth.

**Anti-patterns:**
- Email verification before any value
- "Choose your own adventure" (multiple paths → decision fatigue; one golden path wins)
- API keys hidden in settings pages rather than shown inline
- Static code examples without language switchers

Score: `/10`. Gap to 10: [what would make this perfect].

---

## Pass 2: API / CLI / SDK Design

**Evidence source:** Route definitions, command handlers, public exports, type signatures in the diff.

Check:
- **Naming consistency**: Are endpoint paths, parameter names, and CLI flags consistent with existing patterns? (`/users` plural vs `/user/123` singular is a classic failure)
- **Prefixed IDs** (if API): Self-documenting IDs like Stripe's `ch_`, `cus_` make it impossible to pass the wrong ID type
- **Error semantics**: HTTP status codes match the error type (not 200 for errors, not 500 for validation)
- **Idempotency**: Mutation endpoints (POST/PUT/DELETE) — can they be safely retried? Is there idempotency key support?
- **Progressive disclosure**: Simple case works without reading advanced docs. Complex case uses the same API. (SwiftUI: `Button("Save") { save() }` → full customization, same API)
- **CLI flag design**: `--help` output clear and complete? Flags consistent with POSIX conventions? Auto-detects terminal vs pipe for output format?
- **Escape hatches**: Every opinionated default has an override path

**Anti-patterns:**
- Chatty API: 5 calls for one user-visible action
- God endpoint: 47 parameter combinations with different behavior per subset
- Implicit failure: 200 OK with error nested in response body
- Documentation-required API: can't make first call without reading 3 pages

Score: `/10`. Gap to 10: [what would make this perfect].

---

## Pass 3: Error Message Quality

**Evidence source:** Error throws, catch blocks, validation errors, CLI error output in the diff.

Check against the three-tier model:

**Tier 1 (Elm — best):** "I cannot do X with Y value like this one: [value]. Hint: [exact fix]."
**Tier 2 (Rust):** Error code + exact location + annotated source + help section with exact edit.
**Tier 3 (Stripe — minimum bar):** `{type, code, message, param, doc_url}`. Five fields, zero ambiguity.

For each new error thrown in the diff, verify:
1. **What happened**: specific, not generic ("Invalid email format" not "Bad request")
2. **Why it happened**: cause, not symptom
3. **How to fix it**: exact next step the developer should take
4. **Where to learn more**: link to docs or error code

**Anti-patterns:**
- Generic messages: "Something went wrong", "Error occurred", "Invalid input"
- Burying the actionable hint at the bottom of a long stack trace
- Errors that reveal internal state (DB connection strings, stack traces) to end users
- Missing error codes (can't search for them, can't build error handling tables)

Score: `/10`. Gap to 10: [what would make this perfect].

---

## Pass 4: Documentation Completeness

**Evidence source:** README changes, docs/ changes, inline code comments, CHANGELOG in the diff.

Check:
- **Every new API endpoint/CLI command/export is documented** with: description, parameters, return value, example call, example response
- **Code examples are copy-paste complete**: real auth, real error handling, not just happy-path stubs
- **CHANGELOG entry exists** if this is a breaking change or new public feature
- **Migration guide exists** if this changes existing behavior
- **Types are documented**: TypeScript types, JSDoc, or equivalent — not just "see the code"

**Gold standard:** Stripe's docs pre-fill API keys when logged in. Language switcher persists across all pages. Features don't ship until docs are finalized.

**Anti-patterns:**
- "See source code for details" as documentation
- Examples that require editing before they'll work (hardcoded example.com, fake tokens)
- CHANGELOG entry that describes implementation ("Refactored auth middleware") instead of user impact ("Auth tokens now persist across browser tabs")

```bash
# Check if new endpoints/commands have corresponding doc changes
git diff origin/main --name-only | grep -E '(route|command|controller|handler)' | head -10
git diff origin/main --name-only | grep -E '(README|docs/|CHANGELOG)' | head -10
```

Flag if routes/commands changed but no README/docs/CHANGELOG changed.

Score: `/10`. Gap to 10: [what would make this perfect].

---

## Pass 5: Upgrade & Breaking Change Safety

**Evidence source:** CHANGELOG, package.json version, public API signature changes in the diff.

Check:
- **Are there breaking changes?** Removed fields, changed types, renamed parameters, changed HTTP methods, changed status codes
- **Is the breaking change documented?** CHANGELOG with migration steps
- **Is there a deprecation period?** Or is this a hard cut?
- **Are existing consumers protected?** Backwards-compatible defaults, optional new params, version headers
- **Does the version number reflect the change?** (Breaking change → major bump, new feature → minor bump)

**Gold standard:** Next.js `npx @next/codemod upgrade major` — one command upgrades + runs all relevant codemods. Stripe API version pinning — breaking changes never surprise you.

**Anti-patterns:**
- Breaking public API without version bump
- Silent behavior changes (same signature, different semantics)
- "Deprecated" flag with no timeline or replacement path
- Changes that break the current running code if deployed without a coordinated migration

Score: `/10`. Gap to 10: [what would make this perfect].

---

## DX Scorecard

```
+====================================================================+
|              DX CODE REVIEW — SCORECARD                            |
+====================================================================+
| Pass                   | Score | Key Finding                       |
|------------------------|-------|-----------------------------------|
| Getting Started/TTHW   | __/10 | [one-line gap]                    |
| API/CLI/SDK Design     | __/10 | [one-line gap]                    |
| Error Messages         | __/10 | [one-line gap]                    |
| Documentation          | __/10 | [one-line gap]                    |
| Upgrade Safety         | __/10 | [one-line gap]                    |
+--------------------------------------------------------------------+
| Overall DX             | __/10 |                                   |
+====================================================================+

Target developer: [persona one-liner]
TTHW impact: [Champion / Competitive / Needs Work / Red Flag]
```

---

## DX Fix-First

Same rule as the main review: fix what you can, flag what needs a decision.

**AUTO-FIX (apply without asking):**
- Generic error messages → add specificity (what + why + how to fix)
- Missing required parameter in a code example → add it
- Inconsistent naming (e.g., `/users` vs `/user`) → standardize
- Missing CHANGELOG entry for a new public feature → add it
- Missing `--help` description for a new CLI flag → add it

**ASK (present to user):**
- Breaking changes that need a deprecation strategy
- API design decisions (naming, parameter structure) where multiple valid options exist
- Documentation scope (how much detail is needed for this audience)

Output format follows the same convention as the main review:
```
DX: N issues (X auto-fixed, Y need input)

AUTO-FIXED:
- [file:line] Generic error message → added specific message with cause and fix

NEEDS INPUT:
- [file:line] Breaking change to response format — no CHANGELOG entry
  Fix: Add migration note to CHANGELOG
  → A) Add  B) Skip (not a public API)
```

If no issues: `DX: Clean.`
