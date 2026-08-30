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

## Missing information is a design task (always active)

"We don't hold that data" is the beginning of the work, not the end of it. When a
clause cannot be modelled, a rule cannot be checked or a finding cannot be
answered because a fact is not captured, **design the capture**: which existing
channel carries it, under what key and type, who answers it and when.

This service already has two channels and the product renders both — the
`employee-facts` list a Staff form fills in, and the `RequiredInput` prompts a
finding raises. Check those first. Only where neither fits do you specify a new
record, and then specify it concretely enough to build: the model, the field, the
screen.

The one legitimate stop is a fact that cannot exist for anybody — a judgement, an
intention, something outside any system's reach. *Was this agreement genuine* is
that. *We never added the field* is not, and filing the second under the first is
how a finite backlog stops being finite.


## $AUTOFEATURE_HOME

```bash
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/awards" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

## Args

- `<AWARD_CODE>` — required, e.g. `MA000003`.
- `tables:` — optional comma list, restricts to those `rule_*` tables. Default: all nine the workflow
  covers (`rule_condition`, `rule_span`, `rule_overtime_threshold`, `rule_junior_band`, `rule_allowance`,
  `rule_roster`, `rule_leave`, `rule_break_placement`, `rule_break_entitlement`).

  **This list and `PREDICATE_TABLES` in `scripts/lib/verification.ts` have to hold the same tables.**
  They did not: `rule_break_entitlement` was in the service's count of predicate rows and in neither
  the query below nor the workflow's `SCHEMAS`, so nine rows across two awards were never read and no
  run said so. A table missing from the query is not reported as skipped — the workflow only skips
  what it was handed and could not type — so the coverage line reads clean while the table is
  invisible. Before adding a predicate-bearing table to the service, add it in both places.
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

  UNION ALL
  SELECT 'rule_break_entitlement', clauses, clause_text,
         jsonb_build_object('hours_from', hours_from, 'hours_from_exclusive', hours_from_exclusive,
                             'hours_to', hours_to, 'paid_rest_breaks', paid_rest_breaks,
                             'rest_minutes', rest_minutes, 'unpaid_meal_breaks', unpaid_meal_breaks,
                             'meal_minutes_min', meal_minutes_min, 'meal_minutes_max', meal_minutes_max)
    FROM rule_break_entitlement
   WHERE instrument_id = '<CODE>' AND clause_text IS NOT NULL AND operative_to IS NULL
) r
" > /tmp/award-verify-rows.json
```

### Step 2a: Reconcile the two counts. This is a gate, not a suggestion.

The query above filters `clause_text IS NOT NULL AND operative_to IS NULL`. The service's own count of
this award's predicate rows — the one `predicateFingerprint` hashes and `closure` quotes — filters
neither. So a row with no `clause_text` is **counted as a predicate and cannot be read by this command**,
and until now nothing said so: fifteen such rows sat inside a fingerprint of ninety-two while a run
verified seventy-seven, reported "77 rows", and was recorded as the award's verification. The advice
that used to live here — *find out why for every missing row* — was addressed to whoever was reading,
and nobody was.

So compute both numbers and refuse to proceed unless they reconcile. Do not read the count off
`npm run verified -- <CODE>`: its read form prints the row total only on a never-verified award, so the
one case where you most need the number is the case where it is missing. Ask the database the same
question `predicateFingerprint` asks.

```bash
COUNTED=$(psql_query "
SELECT count(*) FROM (
  SELECT 1
    FROM rule_condition
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_span
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_overtime_threshold
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_junior_band
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_allowance
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_roster
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_leave
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_break_placement
   WHERE instrument_id = '<CODE>'
  UNION ALL SELECT 1
    FROM rule_break_entitlement
   WHERE instrument_id = '<CODE>'
) t")

PULLED=$(jq 'length' /tmp/award-verify-rows.json)
```

Then name every row in the difference, one line each, with the reason it is not in the JSON. One branch
per table, on the same nine tables, differing only in the table name:

```bash
psql_query "
SELECT tbl || E'\t' || coalesce(nullif(clause, ''), '(no clause cited)') || E'\t' || reason
  FROM (
  SELECT 'rule_condition' AS tbl, clauses AS clause,
         CASE WHEN clause_text IS NULL
              THEN 'COUNTED BUT UNREADABLE — clause_text is null'
              ELSE 'superseded — operative_to is set' END AS reason
    FROM rule_condition
   WHERE instrument_id = '<CODE>' AND (clause_text IS NULL OR operative_to IS NOT NULL)
  UNION ALL
  -- ... the same branch for rule_span, rule_overtime_threshold, rule_junior_band,
  -- rule_allowance, rule_roster, rule_leave, rule_break_placement, rule_break_entitlement
) x ORDER BY 3 DESC, 1, 2"
```

**The gate, in three parts. Any one of them failing stops the run before the Workflow is invoked.**

1. `COUNTED = PULLED + <rows the query above returned>`. If numbers are still missing after every
   excluded row has been named, the shortfall is rows in a table the Step 2 query does not have a branch
   for — the `tables:` defect, which is invisible from inside the workflow because a table it was never
   handed cannot be reported as skipped. Add the branch, do not proceed.
2. **No row may carry `COUNTED BUT UNREADABLE`.** This is the fifteen. A predicate row with no
   `clause_text` is one the service counts and this command structurally cannot check — there are no
   words to check the fields against — so it is not an exclusion, it is a hole in the verification with
   a row inside it. Print every one, with its table and clause, and stop. The fix is upstream:
   `npm run award:load` if the award text was never loaded, or writing the transcription into the row if
   the passage was never transcribed. Then re-run Step 2.
3. `superseded` rows are a **legitimate** exclusion and still get named. An award re-mapped after a
   variation carries a closed historical reading alongside the current one (see "Re-authoring after a
   variation" in `award-drift.md`), and this command verifies the current reading by default. Naming
   them is what keeps "legitimate" from quietly absorbing case 2.

Both numbers go in the report and in the `--note` in Step 5 — `verified 77 of 92 predicate rows, 15
superseded` — so that "verified 77 rows" can never be read as "verified the award". When `tables:` or
`clauses:` narrowed the run, scope the `COUNTED` query the same way, or the gate compares a filtered
pull against an unfiltered count and fails on its own filter.

One thing this gate cannot see: a predicate-bearing table in **neither** the query above nor
`PREDICATE_TABLES`. Both sides of the comparison would be blind to it in exactly the same way and the
numbers would agree. That is `verification-scope.test.ts`'s job — it parses the table names out of this
file's own SQL and diffs them against `PREDICATE_TABLES` — and it is the reason the branches above are
written one table per line rather than folded into something shorter.

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

Cost scales with **distinct `(table, clause_text)` groups, not row count** — two blind readers per group
always, a skeptic only where a shipped row disagrees. Rows sharing one clause_text (a span table, an
overtime-threshold table) collapse to one group, so this is often far cheaper than "two per row" implies.
For a first run on a whole award, say so before firing
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
4. **Coverage of this run**: rows checked **against the service's own count** — the `PULLED` of
   `COUNTED` from Step 2a, with every excluded row named and its reason — rows skipped for lack of a
   schema (name which tables), and — if this was a `clauses:`-scoped run — say plainly that the rest of
   the award was **not** re-checked this time, so the report isn't mistaken for a full re-verification.

If invoked standalone (not from `award-map`), append the findings to
`.autofeature/awards/<CODE>-review.md` under a `## Semantic verification — <date>` heading rather than
overwriting anything already there.

## Step 5: Record the run, which is not optional

```bash
npm run verified -- <CODE> \
  --rows <checked> --found <total defects> --real <confirmed after the skeptic pass> \
  --tables <all | the tables: list you were given> \
  [--clauses <the clauses: list you were given>] \
  --report .autofeature/awards/<CODE>-review.md \
  --note "<one sentence a person can act on>"
```

**This is the step that makes the verification visible to everything else.**
Without it the result lives in a markdown file in a working directory, and
`npm run closure` — the command everyone actually reads — has no way to know
whether an award has ever been checked. It printed `IMPLEMENTED` identically for
a verified award and a never-verified one, and `IMPLEMENTED` gets read as "done".

That is the same failure three times over: `closed` meant *read*, `accounted`
meant *the backlog is complete*, `implemented` means *nothing is unbuilt*. Each
was taken for "done" by the next person to look, and each time the fix was to
stop asking a word to carry what a record should. This is the record.

`closure` now prints two verdicts, coverage and correctness, and the second is
read from what this step writes:

```
  coverage      IMPLEMENTED — every clause read, every published rate
                reachable, every rule quoting the award ...

  correctness   VERIFIED 2026-08-16 — every predicate row, 38 rows,
                17 finding(s) of which 2 real.
```

Three rules for writing it honestly:

- **`--found` and `--real` are both required**, because the gap between them is
  the useful number. One run reported 17 and 5 survived a second pass, of which
  2 were real — the other twelve were short transcriptions rather than wrong
  rules, and a record showing only "2" would hide that the run's first output
  was mostly noise about its own inputs.
- **`--tables` and `--clauses` record the SCOPE.** A run over `rule_roster`
  alone is a real result and must never read as a whole-award verification. The
  verdict quotes the scope back, so "verified" always says verified of what.
- **Record it after the defects are fixed, not before.** The record fingerprints
  the predicate rows as they stand when it is written; writing it first and then
  fixing two rows leaves a record that describes rules nobody checked.

### Staleness looks after itself

The record hashes the predicate rows themselves — the days, times, thresholds,
priorities and kinds this command actually read. Change one and the verdict
becomes `STALE` on its own:

```
  correctness   STALE — verified 2026-08-16 over every predicate row (38 rows),
                but the predicate rows have CHANGED since.
```

A commit hash was the obvious alternative and is the wrong one: it moves when
anything in the repo moves, so every award would read stale after every unrelated
change, and an alarm that always fires is one nobody reads.

`clause_text` and `note` are deliberately **outside** the hash. Lengthening a
transcription is exactly what this command asks for — twelve of one run's
seventeen findings vanished when the passages were lengthened — so hashing them
would mark an award stale for having acted on the report.

## What this is not

It is a second, independent reading, not a proof. Two agents can share a blind spot on an unusual
construction the way two people can. A `high`-confidence defect is strong enough to block; a clean run is
evidence worth having, not a guarantee — the review pack still needs the human sign-off `award-map.md`
calls for before anything ships.

`/autofeature:award-audit` is the complement: this command re-derives predicates
from the award's words, that one reads what was written *about* the award — the
dispositions, the transcriptions, the backlog. A clean run of either is not a
clean run of both.
