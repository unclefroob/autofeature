---
description: |
  Behavioural verification for an award already mapped into rosterio-compliance-service: prices real, whole employment situations and reasons — from the award's own words, independently and twice — about whether the pay comes out right.
  The one gap the other four commands cannot reach. `award-verify` checks each rule row against its clause; `award-audit` reads what was written about the award; closure proves every clause was read. None of them can catch a rule INTERACTION — two rows each correct against their clause, combining into a wrong total — or a penalty that should fire and silently does not, which returns a plausible, lower number that no per-row check questions.
  The scenarios are authored, grounded in the award's clauses, and carry NO expected figure: the expected pay is DERIVED at run time by reasoners reading the clause text, blind to what the engine produced, then weighed adversarially against the engine's answer. The reasoning is the check.
  Standalone and re-runnable. Use it after `award-close` finishes an award, after a wage review, or whenever you want to ask "does a real week's pay actually come out right" rather than "is each rule row correct".
  Invoke as:
    /autofeature:award-scenarios <AWARD_CODE>
    /autofeature:award-scenarios <AWARD_CODE> only: sunday-is-public-holiday,weekly-overtime-on-saturday
argument-hint: <AWARD_CODE> [only: <scenario-id,...>]
---

# AutoFeature — Award Behavioural Verification

Checks that a real situation is PAID correctly, not that each rule is correct. `/autofeature:award-verify`
re-derives every predicate row from its clause and diffs it against what shipped — a row-level check. This
is a system-level one: it prices a whole employment situation, the way a payroll officer would describe it,
and asks whether the money the engine returns is what the award actually requires.

The two catch different things, and neither subsumes the other. A rule can be perfect against its clause
and still combine wrongly with the rule beside it; a penalty can be missing entirely, in which case there
is no row for `award-verify` to check and the engine simply returns a smaller number. A clean run of one
is not a clean run of both.

Read `$AUTOFEATURE_HOME/awards/scenario-workflow.md` in full before Step 3 — it has the Workflow script
verbatim and the reasoning for the three disciplines that make the judgement trustworthy. Do not
re-derive the mechanism; invoke that script as written.

## The scenarios are content, and they are grounded

The situations live in `test/scenarios/<AWARD>.ts` in the service, version-controlled, human-readable, and
authored against the award as LOADED rather than from anyone's memory of what a retail or hospitality award
pays — the spans, penalties, overtime thresholds, break bands and shiftwork windows are all read out of
the mapped award first. Each scenario states the situation in prose, the clauses that govern it, and the
one interaction it probes. It carries no expected number: supplying one would defeat the point, which is
that the expected pay is derived from the award at run time and only then compared.

Adding a scenario is editing that file. A good one targets a place two rules meet — a junior casual after
6pm, a Sunday that is a public holiday, overtime that lands on a Saturday, a shiftworker crossing midnight
— rather than a single rule in isolation, which `award-verify` already covers.

## Missing information is a design task (always active)

If a scenario cannot be judged because a fact is not captured — the reasoner needs to know whether an
employee regularly works Sundays, or whether a cold chamber was below zero, and nothing carries it — that
is not a stop. It is the same design task the other award commands hold: name the channel that should
carry the fact (the `employee-facts` list, a `RequiredInput` a finding raises, or a named new record) and
specify it concretely. The one legitimate stop is a fact no system could hold — a judgement, an intention.

## Step 1: Resolve the service and connect

```bash
CWD=$(pwd); PARENT=$(dirname "$CWD")
for c in "$CWD" "$PARENT/rosterio-compliance-service" "$CWD/rosterio-compliance-service"; do
  [ -f "$c/db/schema.sql" ] && [ -d "$c/src/awards" ] && { SVC="$c"; break; }
done
[ -z "$SVC" ] && { echo "Run from rosterio-compliance-service or its parent."; exit 1; }
cd "$SVC"
source scripts/lib/psql.sh
```

Confirm the award has rules loaded (`psql_query "SELECT count(*) FROM rule_clause_group WHERE
instrument_id = '<CODE>'"` is non-zero). Zero means map it first with `/autofeature:award-map` — there is
nothing to price against.

Confirm scenarios exist for it: `test/scenarios/<CODE>.ts`. If none, this award has no behavioural suite
yet; author one against its clauses before running, or say plainly that there is nothing to check.

## Step 2: Price every scenario, deterministically

```bash
npm run scenarios > /tmp/award-scenarios.json
```

This runs each scenario through the engine in-process, gathers the verbatim clause text for the clauses it
cites, and emits one self-contained record per scenario — the situation, the exact input, the clause text,
and what the engine actually did, a refusal included. It is the deterministic half: no judgement, just the
engine's answer and the award's words, both of which have exactly one value.

Read the file and confirm it parsed and every scenario is present. A scenario whose `engine.outcome` does
not match its `expect` — a `price` that refused, a `refused` that priced — is already a finding worth
noting before the workflow runs, because it is a disagreement the shapes alone reveal.

If invoked with `only:`, filter the array to those ids before Step 3, and say in the report that the rest
were not run this time.

## Step 3: Run the Workflow

Pass the script inline, exactly as in `scenario-workflow.md`.

```
Workflow({
  script: "<the Workflow script from awards/scenario-workflow.md, verbatim>",
  args: { award: "<CODE>", scenarios: <the parsed array from Step 2, or the only: subset> }
})
```

Two blind readers plus one adjudicator per scenario — three agents each. A ten-scenario award
is small; a much larger set is real spend, so say so first.

## Step 4: Report

In this order:

1. **Engine defects** — the engine contradicts the award on a real situation. For each: the scenario, the
   segment or finding in dispute, what the award requires instead, the clause, and the DIRECTION —
   `underpays` blocks hardest, because it reaches a payslip unnoticed. These are the output. Fix the rule
   or the engine and re-run scoped to that scenario with `only:`.
2. **Ambiguous** — the award genuinely leaves it open, or the clause text the readers were given was not
   enough to decide. The first is a note for the review pack; the second means the scenario should carry
   more clauses. Neither is a pass and neither is a defect.
3. **Seed-wrong** — the scenario is mis-specified: fix `test/scenarios/<CODE>.ts`, not the engine.
4. **Confirmed** — both blind readers independently derived the engine's answer and neither expected a
   penalty it omitted. Evidence, not proof.
5. **Coverage of this run** — how many scenarios, of how much of the award's surface. Say plainly that N
   scenarios verified is not the award verified: it is the handful of situations somebody thought to write,
   and the interactions nobody wrote a scenario for are exactly the ones this cannot see.

Append the findings to `.autofeature/awards/<CODE>-review.md` under a `## Behavioural verification —
<date>` heading rather than overwriting what is there.

## What this is not

It is a reading of real situations, not a proof. Two blind readers can share a blind spot on an unusual
interaction the way two people can, and the scenarios are authored, so the award's untested corners are
untested here too. An `engine_defect` with the clause quoted is strong enough to act on; a clean run is
evidence worth having, weighed alongside `award-verify`'s row-level pass and the human sign-off the review
pack calls for — not a substitute for either.
