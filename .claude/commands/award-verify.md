---
name: award-verify
description: |
  Semantic verification for an award already mapped into rosterio-compliance-service: checks that every predicate-bearing rule row (day, time, priority, threshold, kind) actually matches what its cited clause says, not just that it exists and is reachable.
  Closes the one gap the eight closure axes cannot: they prove a clause was read and a rule is reachable, never that it was read correctly. This does, by having two agents independently re-derive each row from the clause text alone, diffing against what shipped, and running a third adversarial pass only where they disagree.
  Standalone and re-runnable — use it after ingestion (award-map's Step 7.5 calls it) or on its own after a wage review or an award variation touches specific clauses.
  Invoke as:
    /autofeature:award-verify <AWARD_CODE>
    /autofeature:award-verify <AWARD_CODE> tables: rule_condition,rule_span
    /autofeature:award-verify <AWARD_CODE> clauses: 15,15A,21
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
  - Workflow
---

# AutoFeature — Award Semantic Verification

Checks correctness, not coverage. `/autofeature:award-map`'s eight closure axes prove every clause was
read and every rule is reachable; none of them prove a rule's `days_of_week`, `time_from`, `priority` or
`kind` actually says what the clause it cites says. This command answers that question the way payroll
systems answer "was this figure entered correctly": independently, twice, before trusting it.

Read `$AUTOFEATURE_HOME/awards/verify-workflow.md` in full before Step 2 — it has the Workflow script
verbatim and the reasoning for the design. Do not re-derive the mechanism from scratch; invoke that
script exactly as written.

## $AUTOFEATURE_HOME

```bash
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/awards" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

## Args

- `<AWARD_CODE>` — required, e.g. `MA000003`.
- `tables:` — optional comma list, restricts to those `rule_*` tables. Default: all eight the workflow
  covers (`rule_condition`, `rule_span`, `rule_overtime_threshold`, `rule_junior_band`, `rule_allowance`,
  `rule_roster`, `rule_leave`, `rule_break_placement`).
- `clauses:` — optional comma list of clause prefixes (e.g. `15,15A`), restricts to rows whose `clauses`
  column starts with one of them. Use this for a targeted re-check after an award variation, rather than
  re-running the whole award.

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

**There is deliberately no `psql` binary assumed on this machine** — `scripts/lib/psql.sh` is the
service's own answer to that, routing every query through a throwaway container (or the host network,
if `DATABASE_URL` points at a deployed database) rather than requiring a system install. Use its
`psql_query "<sql>"` for a read and `apply_sql "<path>"` for a file — never a bare `psql` invocation, it
will not exist to run.

Confirm the award has rules loaded (`psql_query "SELECT count(*) FROM rule_clause_group WHERE
instrument_id = '<CODE>'"` is non-zero). If it's zero, this is the wrong command — map it first with
`/autofeature:award-map`.

## Step 2: Pull every predicate-bearing row as one JSON array

Workflow scripts have no database access, so the rows are fetched here, in this command's own context,
and passed in as `args`. One query per table, `jsonb_build_object` shaped to match `db/schema.sql`
field-for-field — the diff in the workflow is only meaningful if the field names line up exactly.

Every branch filters `AND operative_to IS NULL`. An award can carry more than one row per clause once
it's been re-mapped after a variation, the historical one closed out and a current one taking over —
see "Re-authoring after a variation" in `award-drift.md`. This command verifies the **current** reading
by default; verifying a historical version is a deliberate, separate request, not the default, and if
asked for it, add `AND operative_from <= '<AS-OF DATE>' AND (operative_to IS NULL OR operative_to > '<AS-OF DATE>')`
in place of `operative_to IS NULL` instead.

Apply the `tables:`/`clauses:` filters as `WHERE` clauses on each branch (`clauses LIKE '15%' OR clauses
LIKE '15A%'` etc. for a `clauses:` filter; omit table branches not in `tables:`).

```bash
psql_query "
SELECT jsonb_agg(r) FROM (
  SELECT 'rule_condition' AS tbl, clauses AS clause, clause_text AS \"clauseText\",
         jsonb_build_object(
           'hour_type', hour_type, 'days_of_week', days_of_week, 'time_from', time_from,
           'time_to', time_to, 'public_holiday', public_holiday, 'ot_hours_from', ot_hours_from,
           'ot_hours_to', ot_hours_to, 'priority', priority, 'worker_type', worker_type,
           'whole_shift', whole_shift
         ) AS predicate
    FROM rule_condition
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL

  UNION ALL
  SELECT 'rule_span', clauses, clause_text,
         jsonb_build_object('day_of_week', day_of_week, 'worker_type', worker_type,
                             'time_from', time_from, 'time_to', time_to)
    FROM rule_span
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL

  UNION ALL
  SELECT 'rule_overtime_threshold', clauses, clause_text,
         jsonb_build_object('employment_type', employment_type, 'daily_hours', daily_hours,
                             'long_day_hours', long_day_hours, 'weekly_hours', weekly_hours)
    FROM rule_overtime_threshold
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL

  UNION ALL
  SELECT 'rule_junior_band', clauses, clause_text,
         jsonb_build_object('age_from', age_from, 'age_to', age_to,
                             'service_months_from', service_months_from,
                             'service_months_to', service_months_to)
    FROM rule_junior_band
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL

  UNION ALL
  SELECT 'rule_allowance', clauses, clause_text,
         jsonb_build_object('trigger', trigger, 'unit', unit, 'all_purpose', all_purpose)
    FROM rule_allowance
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL

  UNION ALL
  SELECT 'rule_roster', clauses, clause_text,
         jsonb_build_object('kind', kind, 'value', value, 'value_secondary', value_secondary,
                             'window_days', window_days, 'overtime_consequence', overtime_consequence)
    FROM rule_roster
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL

  UNION ALL
  SELECT 'rule_leave', clauses, clause_text,
         jsonb_build_object('source', source, 'paid', paid, 'applies_to', applies_to,
                             'worker_type', worker_type, 'accrual_method', accrual_method,
                             'weeks_per_year', weeks_per_year, 'days_per_occasion', days_per_occasion,
                             'hours_per_occasion', hours_per_occasion, 'excessive_weeks', excessive_weeks,
                             'cumulative', cumulative)
    FROM rule_leave
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL

  UNION ALL
  SELECT 'rule_break_placement', clauses, clause_text,
         jsonb_build_object('kind', kind, 'value', value)
    FROM rule_break_placement
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL
) r
" > /tmp/award-verify-rows.json
```

Read the file, confirm it parsed and the row count is sane (roughly the same order of magnitude as the
closure report's clause count for this award). A near-empty result usually means `award-text.sql` hasn't
been run yet — `clause_text` is null on everything, and there is nothing to check the words against.

## Step 3: Run the Workflow

Pass the script **inline** — the Workflow tool persists it to a file on its own and returns the path,
which is what you'd use to resume, not what you pass on a first run.

```
Workflow({
  script: "<the Workflow script from awards/verify-workflow.md, verbatim>",
  args: { award: "<CODE>", rows: <the parsed JSON array from Step 2> }
})
```

This is genuinely 2–3 agents per row (two blind readers always, a skeptic only where they disagree with
the shipped row), so cost scales with row count. For a first run on a whole award, say so before firing
it — this is real spend, not a cheap lint pass. For a `clauses:`-scoped re-check after a variation, it's
small enough to run without asking.

## Step 4: Report

Present, in this order:

1. **Confirmed defects** (`confidence: high`) — the clause, the shipped value, what the skeptic says it
   should be and why, quoting the clause text. These block: either fix the row and re-run this command
   scoped to that clause, or write the override and the reason into
   `.autofeature/awards/<CODE>-map.md`.
2. **Unresolved rows** — independent readers disagreed with the shipped row but the skeptic agent failed
   before returning a verdict. Re-run scoped to these clauses before treating anything else here as final;
   an unresolved disagreement is not a pass.
3. **Confirmed defects** (`medium`/`low`) and **cleared disagreements** — for the review pack, not a
   blocker.
4. **Coverage of this run**: rows checked, rows skipped for lack of a schema (name which tables), and —
   if this was a `clauses:`-scoped run — say plainly that the rest of the award was **not** re-checked
   this time, so the report isn't mistaken for a full re-verification.

If invoked standalone (not from `award-map`), append the findings to
`.autofeature/awards/<CODE>-review.md` under a `## Semantic verification — <date>` heading rather than
overwriting anything already there.

## What this is not

It is a second, independent reading, not a proof. Two agents can share a blind spot on an unusual
construction the way two people can. A `high`-confidence defect is strong enough to block; a clean run is
evidence worth having, not a guarantee — the review pack still needs the human sign-off `award-map.md`
calls for before anything ships.
