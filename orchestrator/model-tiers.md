---
status: CUSTOM
description: Model-tier (cost/quality) policy for the autofeature agent fleet. Every spawned Agent and every Workflow agent() phase inherits the session model unless given an explicit model:, so running on Opus puts the whole fan-out on Opus — the main cost driver. Floor is Sonnet (no Haiku). Each task has a base tier; escalation rules raise specific tasks to Opus per run based on scope, risk, and failures, recorded as a Model Plan in the Feature Brief. Read by autofeature.md and the review/market/seo/test/copy commands when fanning out.
---

# Model Tiers — cost/quality policy for the fleet

**Problem:** an `Agent({...})` spawn or a Workflow `agent()` call with **no `model:`** inherits the
session model. Launch `/autofeature` on Opus and *every* Explore, architect, reviewer, test-runner,
and analyst runs on Opus — the dominant cost. This file gives each task a base tier, and says when a
run should raise a task to Opus.

## How model selection works

- **Agent tool:** add `model: "sonnet" | "opus"` to the `Agent({...})` call.
- **Workflow:** add `model` to the `agent(prompt, { ...opts })` options.
- Omitted in either case → inherits the session / main-loop model. **Never omit it.**
- A subagent's `model:` overrides the session in **both** directions: it downgrades an Opus session for
  routine work *and* upgrades a Sonnet session for opus-tier steps. Two things always follow the
  session model:
  - **Skills** (`security-review`, `simplify`, `frontend-design`, the `seo` skills, `ios-simulator`) —
    invoked via the `Skill` tool, run in-context, not per-call overridable.
  - **The orchestrator's own loop** (prompt composition, triage, decisions).
  To economize *those*, run the command itself on Sonnet (`/model sonnet`). Opus-tier steps still
  self-elevate via their pins and the escalation rules below.

Use the generic aliases (`sonnet`/`opus`) — never pinned IDs — so they track the latest model in each
tier.

## Floor: Sonnet

**No agent runs below Sonnet.** `haiku` is not a valid tier in this fleet. Mechanical work (test-runner,
surface maps, citation re-fetch) runs on Sonnet: those outputs feed reviews and gates, and a missed
failure or surface costs more than the saving. If a request or override names `haiku`, treat it as
`sonnet`.

---

## Base tiers (profile: BALANCED)

| Tier | Tasks | Why |
|------|-------|-----|
| **sonnet** | `test-runner` (run + parse + ≤2KB summary); market-review **citation re-fetch + classify**; product-review / feature-test / copy-audit / patterns **surface maps**; deploy-verify **smoke run + parse**; scope-gate context Explore; autofeature **build-context Explore**; **Plan** (non-cross-repo); **all architects** design **and** implement (`express-mongo`, `react`, `react-native`, `swift`, `kotlin-compose`, `seo`); `mongo-data-modeler`; `api-contract-broker` (non-cross-repo); **code-review passes** (critical / info / testing / design / devex); feature-advice (scan + advisors + synthesis); product-review **lenses + claim-verify + synthesis**; market/gap/vc **analysts + framing**; copy-audit **audit passes** | Workhorse — enumeration, code comprehension, design, implementation, review, analysis. |
| **opus** | market-review **managing-partner memo**; **bear-case** adversarial analyst; copy **rewrites** (fix/write modes) | Fixed Opus pins — highest-judgment synthesis, adversarial quality gate, writing voice. Apply on every run regardless of scope. |

Base tiers are what standalone commands (`/autofeature:review`, `:market-review`, `:copy`, …) use.
`/autofeature` starts from them and applies the escalation rules.

---

## Escalation rules (dynamic — `/autofeature`)

Evaluated **once, right after the scope gate** (Step 3), from the Feature Brief (`## Context`,
`## Scope`, the feature request). Each rule that fires raises the named tasks from `sonnet` to `opus`
for this run only. Rules only raise — never below base tier.

| # | Rule | Fires when | Raise to opus |
|---|------|------------|---------------|
| **E1** | Cross-repo | Scope tier = `cross-repo` | **Plan**, **api-contract-broker** |
| **E2** | High-risk surface | The feature adds or changes: auth / sessions / tokens / refresh flow; permissions, roles, or tenant isolation; payments, billing, payroll or award/pay calculations; secrets or security headers (CORS/CSP) | **Plan**; the **architect in `design` mode** for each layer that owns the risky code; the **critical review pass** |
| **E3** | Destructive data change | A migration or backfill that rewrites, deletes or re-keys **existing** data (not just an additive field/index) | **mongo-data-modeler**; **Plan** |
| **E4** | Unclear design | Product review (Step 4.5) returned a 🔴 broken-flow finding folded into scope, **or** the Plan subagent surfaced a User Challenge that was resolved by changing the approach | the **architects in `design` mode** (runs after Plan, so E4 is re-evaluated when Step 5a returns) |

Not escalated by any rule: implementers (`implement` mode), test-runner, surface maps, the
informational / testing / design / devex review passes. Implementers follow a design that has already
been reviewed; they escalate only via **failure escalation** below.

`micro` scope: no fan-out, no rules apply — the plan is one line (`all sonnet`).

## Failure escalation (dynamic — any command)

When an agent's output fails its gate and the work is handed back to an agent for another attempt:

1. **First retry:** same task, same tier (Sonnet), failure evidence attached (test-runner summary,
   review finding, or the agent's own "blocked" note).
2. **Second retry:** same task on **opus**, with both failures attached.
3. A failure after the Opus retry → **Emergency Stop**, escalate to the user. Do not retry again.

"Fails its gate" means: an implementer's layer fails the verification gate after its fix, an agent
returns incomplete/blocked on its assigned job, or a claim-verify agent's verdict is contradicted by the
code. Record each escalation in the Model Plan (`E-fail: [task] → opus — [reason]`).

---

## Model Plan (written to the Feature Brief)

After the scope gate, the orchestrator appends:

```markdown
## Model Plan

**Profile:** [balanced | economy | quality | forced:<tier>]
**Rules fired:** [E1, E2 — or "none"]

| Task | Model | Why |
|------|-------|-----|
| Plan | opus | E2 — changes refresh-token flow |
| express-mongo-architect (design) | opus | E2 — owns auth middleware |
| react-architect (design) | sonnet | base |
| implementers | sonnet | base |
| critical review pass | opus | E2 |
| everything else | sonnet | base |

**Escalations during run:** [none — appended as they happen]
```

Every later spawn reads its `model:` from this table — don't re-judge per spawn. On resume, reuse the
plan from the brief. In CHECKPOINT mode, show it alongside the scope classification (the user can
change any row).

---

## Profiles and per-run override

A run may pass a fleet-level override in its args:

| Override | Effect |
|----------|--------|
| *(absent)* / `model: balanced` | Base tiers + escalation rules + failure escalation. |
| `model: economy` | Everything on Sonnet **including** the fixed Opus pins. Escalation rules **off**; failure escalation still applies (it's the only route to Opus). |
| `model: quality` | Base tiers, and every escalation rule treated as fired for all tasks it names (Plan, all architects in design mode, broker, data-modeler, critical review → opus). |
| `model: opus` / `model: sonnet` | Pin **every** spawned agent to that tier. No rules, no failure escalation. |
| `model: haiku` | Treated as `model: sonnet` (floor). |

The override changes only spawned-agent tiers; Skills and the orchestrator loop still follow the
session model.
