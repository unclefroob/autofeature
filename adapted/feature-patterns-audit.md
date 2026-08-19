---
adapted-from: (none — custom)
status: CUSTOM
purpose: |
  Pattern census methodology — extract a repo's de-facto coding canon into
  .autofeature/patterns.md, measure drift against it, and keep it enforced.
  Consumed by .claude/commands/patterns.md (audit/establish/check/fix modes)
  and by the main pipeline (Step 2a discovery, specialist prompts, Step 9 check,
  Step 10 write-back).
---

# Pattern Census — Methodology

## Philosophy

**Census, not lint.** A linter flags every instance; a census names the variants of each concern,
counts them, and decides which one is canon. The output is a decision file, not a findings dump.

**Extraction, not invention.** Mature repos already have patterns — helpers, docs, middleware
chains, lint ratchets. The job is to write down which variant won, not to import a style guide.
Generic best practice (async error propagation, `.lean()` on reads, validation at the boundary)
already lives in the architect agents and `feature-review-checklist.md`. **If a rule would be true
of every repo on this stack, it does not belong in patterns.md.** Only repo-specific decisions go in.

**The amplifier problem.** The architects are told to "read 2-3 existing files and match the repo."
When the repo disagrees with itself (26 controllers do X, the 3 newest do Y), the specialist matches
whichever variant it happened to sample — usually the majority, which is often the *deprecated*
one. patterns.md exists to resolve exactly this ambiguity. Its Canonical decisions override
sampling; where it is silent, sampling remains the rule.

**Dominant ≠ canonical.** The most common variant is the default candidate, not the winner.
Selection rules below.

---

## The dimensions

Census each dimension: variants found (rough counts — grep is fine), the dominant variant,
drift examples (`file:line`, the SAME concern done differently), and dead scaffolding
(installed-but-unused libraries, zero-usage middleware/helpers).

| # | Dimension | What to census | Drift signals |
|---|-----------|----------------|---------------|
| 1 | Layering & structure | route → controller → service → model (or whatever the repo does); who may touch the ORM; how DB handles/connections are passed | data access above its layer; lint-ratchet violations |
| 2 | Route/handler style | router wiring, middleware chains, handler shape (arrow-const vs function, typed req), async error propagation | bare async handlers; inline handlers; mixed export styles |
| 3 | Input validation | library vs manual; where it runs (middleware / controller / service); id + enum + range gates | endpoints with validated twins elsewhere; `as string` casts as the only "validation"; query params straight into DB filters |
| 4 | AuthN/AuthZ & scoping | auth middleware chain + naming; role gates; tenant/org/location scoping (helper or hand-rolled per handler); the public-route allowlist | routes missing the chain; docblock says "X only" but no gate; scoping hand-rolled beside an existing helper |
| 5 | Errors & response envelope | central error middleware vs per-file helpers; success/error JSON shapes; status-code habits; what leaks in 500 bodies | multiple envelopes; raw `error.message` to clients; copy-pasted error helpers |
| 6 | Data access | `.lean()`/projection habits; populate patterns; transactions/sessions for multi-doc writes; where indexes are declared; bounded queries + clamped limits | populate-heavy reads without lean; unbounded `find({})` on growing collections; `limit` from query string unclamped; write-in-loop |
| 7 | Dates & timezones | the blessed wall-clock helper(s) and where the zone comes from | `setHours`/`getDay`/`new Date()` wall-clock math on the server clock beside the helper |
| 8 | Logging & observability | logger vs `console.*`; levels; hot-path logging; what must never be logged | per-request info logs in middleware; secrets/PII in logs |
| 9 | Naming & files | file suffixes/casing; model naming; test placement | twin files (`constraintController` vs `constraintsController` class); 3 test-placement conventions |
| 10 | Testing | framework; what ships with a feature (unit / integration / both); special suites (multi-timezone, zones, snapshots) | new endpoints with zero integration tests while siblings have them |

**Cross-cutting — the canonical-helper hunt (highest-value drift class).** For every non-trivial
helper in `utils/`/`services/` that answers a recurring question ("who can work at location L",
"does manager M cover location X", "start of day in the location's zone"), grep for hand-rolled
twins — code that re-derives the same answer without calling the helper. Twins drift; that is how
the same query returns different eligibility lists in two screens. Every such helper goes in the
registry; every twin is a 🔴 or 🟡 finding.

---

## Canon selection rules

Apply in order when drafting patterns.md:

1. **Dominant + sound → Canonical.** Majority variant with no evidence against it wins silently.
2. **Dominant but deprecated → `⚖ PROPOSED`.** If the newest files, the repo's own docs, or bug
   history point away from the majority, never crown the majority. Record both variants, state a
   recommendation with the evidence, and mark the entry `⚖ PROPOSED`. Establish mode resolves it
   with the user.
3. **Written docs outrank counts** (ARCHITECTURE.md, CLAUDE.md, docblocks that say "canonical") —
   but record doc-vs-reality contradictions as findings; a doc that claims "zero violations" while
   nine exist is itself drift.
4. **Newest-era signal.** If the last N months of files consistently do X, treat X as the intended
   direction even against a legacy majority (check `git log --diff-filter=A` dates).
5. **No signal → stay silent.** patterns.md explicitly lists undecided areas under `Silent`;
   architects fall back to repo sampling + their own stack defaults there. A silent entry is
   honest; a guessed canon is drift with a rubber stamp.

## Severity rubric (drift findings)

- 🔴 **Exploitable or wrong-answer now** — auth-coverage gaps, injection surfaces, hand-rolled
  twins of a correctness-critical helper, secrets fallbacks. These also feed Emergency Stop #3
  when found during a pipeline run.
- 🟡 **Decay** — envelope/style forks, unbounded queries, missing transactions on multi-doc
  writes, dead scaffolding, enforcement not running.
- 🟢 **Cosmetic** — naming wrinkles, file placement, log emoji.

---

## patterns.md — format

Hard rules:

- **≤150 lines.** It is injected verbatim into specialist prompts; brevity is a feature. If a
  section needs an essay, the essay belongs in the repo's docs with a one-line pointer here.
- Lives at `.autofeature/patterns.md`, committed. One per repo (paired `x-api`/`x-app` repos each
  get their own).
- Header: `status: DRAFT | ACTIVE`, date, repo. DRAFT = generated by audit, not yet approved —
  the pipeline treats DRAFT as advisory (check mode reports, never blocks) and ACTIVE as canon.
- Every dimension entry uses three labels: **Canonical** (1-3 lines, tiny snippet only when the
  shape is the point), **Banned** (variants to reject, each with a one-line why/migration note),
  **Silent** (explicitly undecided).
- Unresolved decisions carry `⚖ PROPOSED:` with the options and a recommendation.
- **Canonical-helper registry** — a table: Question → Helper (path) → Never.
- **Decisions log** — append-only dated one-liners. This is what keeps the file from going stale.

Template:

```markdown
# Coding patterns — [repo]
status: DRAFT | ACTIVE · updated: YYYY-MM-DD · maintained by /autofeature:patterns

Scope: repo-specific decisions only. Where this file is silent, match the repo and stack defaults.
Canonical overrides what you'd infer from sampling code — drifted majorities do not out-vote this file.

## Layering
Canonical: ...
Banned: ...

## Routes & handlers
...

## Validation
⚖ PROPOSED: ...

## Auth & scoping
...

## Errors & responses
...

## Data access
...

## Dates & timezones
...

## Logging
...

## Naming, files & tests
...

## Canonical helpers — delegate, never reimplement
| Question | Helper | Never |
|---|---|---|
| ... | `path` | ... |

## Decisions log
- YYYY-MM-DD: ...
```

## Audit report — format

Written to `.autofeature/patterns-audit-[YYYY-MM-DD].md`:

1. **Repo shape** — structure, language, framework versions, tooling configs and whether anything
   *runs* them (CI/hooks), convention docs, era spread.
2. **Census per dimension** — variants, counts, dominant, drift examples with `file:line`.
3. **Security smells** and **performance smells** tables (severity / finding / evidence). These are
   *sampled* pattern evidence, not a security audit — route anything exploitable to the user
   immediately and recommend `security-review` for depth.
4. **Dead scaffolding** — each item with the zero-usage evidence.
5. **Verdict** — does the repo have a canon; top 5 drift areas ranked by threat.
6. **Decisions needed** — the `⚖ PROPOSED` list, each with options + recommendation. This section
   is the establish-mode agenda.

---

## Audit procedure (audit mode)

1. **Surface map** — one Explore agent (haiku): src structure with file counts per layer,
   package.json framework/validation/logging deps, tooling configs (+ whether CI/husky exists),
   convention docs, era spread (`git log --diff-filter=A --format=%ad` on a sample per directory).
2. **Census fan-out** — general-purpose agents (sonnet), single parallel message, grouped so each
   holds a coherent slice:
   - **A** dimensions 1, 2, 9, 10 (structure, handlers, naming, tests)
   - **B** dimensions 3, 4 + security smells + the canonical-helper hunt
   - **C** dimension 5, 8 + dead-scaffolding hunt
   - **D** dimensions 6, 7 + performance smells
   Each agent reads this methodology file, samples ≥12 files spread across eras for its
   dimensions, and returns the census (variants/counts/dominant/drift `file:line`) under 900 words.
3. **Merge & draft** — the orchestrator dedupes, applies the canon selection rules, writes the
   audit report, then writes/updates the patterns file:
   - No patterns.md, or status DRAFT → (re)write the DRAFT.
   - Status ACTIVE → do NOT rewrite it; append a `## Proposed amendments` section to the audit
     report instead and leave the ACTIVE file untouched.

## Check procedure (check mode)

Diff-scoped and cheap — one sonnet agent, no edits, suitable for every pipeline run:

1. Scope = files changed on the current branch vs base (or staged changes if no branch diff).
2. For each changed file, check only: Canonical/Banned entries for the dimensions it touches, the
   canonical-helper registry (did the diff re-derive a registered question?), and the
   auth-coverage invariant (new routes carry the canonical middleware chain + role gate unless on
   the documented public allowlist).
3. Output: `Patterns check: CONFORMS` or a violations list — `file:line · canon rule · fix` —
   with 🔴/🟡/🟢 severity, plus any `⚖ PROPOSED` areas the diff touches (informational — the diff
   can't violate an undecided canon).
4. DRAFT patterns.md → report but mark everything advisory. Findings feed the same Fix-First
   triage queue as review findings.

## Establish procedure (establish mode)

Establish is **always interactive** — canon decisions are User Sovereignty territory. The pipeline
never runs establish; only a human invokes it.

1. Preconditions: an audit report + DRAFT patterns.md exist (else run audit first).
2. Resolve every `⚖ PROPOSED` with the user — one batched AskUserQuestion, each item carrying the
   recommendation and evidence from the audit.
3. Finalize patterns.md → `status: ACTIVE`, log the decisions in the Decisions log.
4. Enforcement menu (apply what the user approved; a canon nothing runs is a doc, not a ratchet):
   - **eslint ratchet extensions** — encode Banned variants that are expressible as
     `no-restricted-syntax` / `no-restricted-imports` / `no-restricted-properties` (match the
     repo's existing ratchet style; error message points at patterns.md).
   - **husky pre-push** (default) — lint + typecheck. Pre-push, not pre-commit: slow hooks get
     bypassed, and `--no-verify` habits kill the ratchet.
   - **CI workflow** (offer) — lint + typecheck + tests on PR. If the repo has no CI at all, say
     so plainly and recommend it; a hook only guards machines that installed it.
   - **Dead scaffolding removal** — one commit per item, each preceded by fresh zero-usage
     evidence (grep the import AND the symbol) pasted into the commit body.
   - **Doc pointer** — the repo's architecture/CLAUDE doc gets a one-liner pointing at
     patterns.md as the live canon.
5. Work on a branch (`patterns/establish-[date]`), bisectable commits, present the diff summary.
   Push/PR only with user approval.

## Fix procedure (fix mode)

Convergence work is code-semantics work — it goes through the pipeline, never applied inline:

- Take one drift area from the audit report (e.g. "converge all controllers on the canonical
  error style"), compose a feature request that names the canon entry, the file list from the
  audit, and `[skip-product-review]` (it is a convergence, not a product change).
- Hand off via `Skill({ skill: "autofeature:autofeature", args: "[composed request]" })`. Scope
  gate will usually classify single-layer; the specialists inherit patterns.md like any run.
- One drift area per run — convergence PRs must stay reviewable.

## Write-back protocol (pipeline Step 10)

When a run makes a convention decision patterns.md doesn't cover — a new canonical helper was
created, a new response case standardized, a first dependency of a given kind introduced — the
orchestrator appends to patterns.md **in the same PR**: a dated Decisions-log line plus the
relevant section entry.

Rules: **additions only.** Changing or removing an existing Canonical entry requires the user —
flag it `⚖` in the PR body instead of editing. If patterns.md is DRAFT, write-backs still apply
(the draft should track reality until establish freezes it).

## Fleet notes

The skill is repo-agnostic; only `.autofeature/patterns.md` differs per repo. Census dimensions
3-8 read naturally onto any Express/Mongoose API; for React / React Native / Swift repos the same
dimensions map to their equivalents (validation → form/input validation, data access → data
fetching/caching, auth → session/token handling, dates/timezones unchanged). Dimension 1-2
wording follows the stack's architect agent.
