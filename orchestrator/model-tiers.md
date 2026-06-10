---
status: CUSTOM
description: Model-tier (cost/quality) policy for the autofeature agent fleet. Every spawned Agent and every Workflow agent() phase inherits the session model unless given an explicit model:, so running on Opus puts the whole fan-out on Opus — the main cost driver. This file pins each task to the cheapest tier that does it well (active profile: BALANCED) and is the single editable knob; the spawn sites carry the resulting model: value. Read by autofeature.md and the review/market/seo/test/copy commands when fanning out.
---

# Model Tiers — cost/quality policy for the fleet

**Problem:** an `Agent({...})` spawn or a Workflow `agent()` call with **no `model:`** inherits the
session model. Launch `/autofeature` on Opus and *every* Explore, architect, reviewer, test-runner,
and analyst runs on Opus — the dominant cost. This file pins each task to the cheapest tier that does
it well, so cost no longer depends on which model the session happens to be on.

## How model selection works

- **Agent tool:** add `model: "haiku" | "sonnet" | "opus"` to the `Agent({...})` call.
- **Workflow:** add `model` to the `agent(prompt, { ...opts })` options.
- Omitted in either case → inherits the session / main-loop model.
- A subagent's `model:` overrides the session in **both** directions: it downgrades an Opus session for
  cheap work *and* upgrades a Sonnet session for the few opus-tier steps. So with these pins the
  pipeline costs roughly the same regardless of which model you launch on — **except** the two things
  that always follow the session model:
  - **Skills** (`security-review`, `simplify`, `frontend-design`, the `seo` skills, `ios-simulator`) —
    invoked via the `Skill` tool, run in-context, not per-call overridable.
  - **The orchestrator's own loop** (prompt composition, triage, decisions).
  To economize *those*, run the command itself on Sonnet (`/model sonnet`). The deep-reasoning steps
  are still elevated to Opus via the per-task pins below, so this is the recommended way to run.

Use the generic aliases (`haiku`/`sonnet`/`opus`) — never pinned IDs — so they track the latest model
in each tier.

---

## Active profile: BALANCED

| Tier | Tasks | Why |
|------|-------|-----|
| **haiku** | `test-runner` (run + parse + ≤2KB summary); market-review **citation re-fetch + classify**; product-review & feature-test **surface maps**; copy-audit **surface map**; deploy-verify **smoke-result parse**; scope-gate context Explore | Mechanical — run / fetch / parse / enumerate. No design judgment. |
| **sonnet** | autofeature **build-context Explore**; **Plan** (single-layer / cross-stack); **all architects** design **and** implement (`express-mongo`, `react`, `react-native`, `swift`, `seo`); `mongo-data-modeler`; `api-contract-broker`; **code-review passes** (critical / info / testing / design / devex); feature-advice (scan + product/build advisors + synthesis); product-review **lenses + claim-verify + synthesis**; market/gap/vc **analysts + framing**; copy-audit **audit passes**; deploy-verify **E2E smoke run** | Workhorse — code comprehension, design, implementation, review, analysis. Sonnet 4.6 is a strong coder/reader. |
| **opus** | market-review **managing-partner memo**; **bear-case** adversarial analyst; **Plan when scope = `cross-repo`**; copy **rewrites** (fix/write modes) | Highest-judgment synthesis · adversarial quality gate · hardest multi-repo planning · writing voice. The only places Opus earns its cost. |

> Borderline calls (worth knowing if you re-tune): **surface maps** sit on haiku — they only enumerate
> routes/endpoints/screens — but they feed the reviews, so bumping them to sonnet is the first move if a
> review feels shallow. **Plan** and **api-contract-broker** sit on sonnet — bump to opus for unusually
> large or risky changes.

---

## Other profiles (one-edit shifts)

- **Economy (max savings):** move build-context Explore + every scan/map to **haiku**; move the three
  **opus** tasks to **sonnet**. Opus is then never used unless opted in per run.
- **Quality-preserving (smaller cut):** keep only `test-runner` + citation re-fetch on **haiku**, all
  Explore/scan/map on **sonnet**; everything else (architects, Plan, reviews, analysts, syntheses)
  stays on **opus**.

To switch profiles: edit the `model:` values at the spawn sites to match the chosen column. This file
is the reference for what those values should be.

---

## Per-run override

A command may accept a fleet-level override in its args and apply it for that run only:

- `model: economy` | `model: balanced` | `model: quality` → use that whole profile's column.
- `model: opus` / `model: sonnet` / `model: haiku` → pin **every** spawned agent to that one tier
  (blunt but simple — e.g. `model: sonnet` for an all-Sonnet run).

Absent → **BALANCED**. The override changes only spawned-agent tiers; Skills and the orchestrator loop
still follow the session model (see above).
