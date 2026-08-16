---
name: award-drift
description: |
  On-demand check for whether a mapped award's wording has moved since it was last mapped or verified. Re-fetches the award text, diffs it clause-by-clause against what's stored, names exactly which rule rows are affected, and re-authors WITH HISTORY PRESERVED — a shift dated before the variation must still price against the reading that was actually in force then.
  Also flags when the FWC pay database itself may be stale, which the text diff alone cannot see.
  Manual trigger, not a cron — award variations are infrequent, so this is meant to be run when you have reason to suspect one landed, not on a schedule.
  Invoke as: /autofeature:award-drift <AWARD_CODE>
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
---

# AutoFeature — Award Drift Check

Answers one question: **has this award's own wording changed since we last checked it?** This is about
the award's *text* moving, a new penalty condition, a widened span, a changed roster concession, not
about dollar figures.

**Two things this checks, and they need different mechanisms because they come from different sources.**
The award's *text* is fetched live from the Fair Work library, so it can be diffed (Steps 1-2). The
Commission's *pay database* — the workbooks in `data/raw/` — has no fetch script at all; they're placed
by hand. A variation that changes wording is caught here. A variation that adds a wholly new priced
condition is only caught if someone separately re-downloads the workbooks, which this command cannot do
and will not silently assume happened — see Step 2b.

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


## Step 1: Resolve and snapshot what's currently stored

```bash
CWD=$(pwd); PARENT=$(dirname "$CWD")
for c in "$CWD" "$PARENT/rosterio-compliance-service" "$CWD/rosterio-compliance-service"; do
  [ -f "$c/db/schema.sql" ] && { SVC="$c"; break; }
done
cd "$SVC"
source scripts/lib/psql.sh
cp "data/raw/award-text-<CODE>.json" /tmp/award-text-<CODE>-before.json
```

Every SQL step below uses `psql_query "<sql>"` from that file, never a bare `psql` invocation — there is
deliberately no `psql` binary assumed on this machine; see `award-verify.md` Step 1 for why.

If that file doesn't exist, this award was never fetched — nothing to diff against, stop and say so.

## Step 2: Re-fetch and diff

```bash
npm run award:fetch -- <CODE>
```

This overwrites `data/raw/award-text-<CODE>.json` with today's version. Compare:

```bash
python3 - <<'PY'
import json
before = json.load(open("/tmp/award-text-<CODE>-before.json"))
after  = json.load(open("data/raw/award-text-<CODE>.json"))

if before["operativeFrom"] != after["operativeFrom"]:
    print(f"OPERATIVE DATE MOVED: {before['operativeFrom']} -> {after['operativeFrom']}")

b = {c["clause"]: c["text"] for c in before["clauses"]}
a = {c["clause"]: c["text"] for c in after["clauses"]}

added   = sorted(set(a) - set(b))
removed = sorted(set(b) - set(a))
changed = sorted(k for k in (set(a) & set(b)) if a[k] != b[k])

print(f"added: {added}")
print(f"removed: {removed}")
print(f"changed: {changed}")
PY
```

If all three lists are empty and the operative date matches: **nothing moved.** Still run Step 2b before
stopping — a clean text diff says nothing about whether the pay database is current.

## Step 2b: Flag possible pay-database staleness

There is no fetch script for the workbooks, so this can only surface the question, never answer it:

```bash
psql_query "SELECT award_code, version_number FROM fwc_award WHERE award_code = '<CODE>'"
```

Report the loaded version alongside a reminder to check
[the FWC Modern Awards Pay Database](https://www.fwc.gov.au/agreements-awards/awards/modern-awards-pay-database)
for a newer release, particularly if today's date is on or shortly after 1 July (the Annual Wage Review
takes effect then) or if Step 2 found a text change at all — a variation substantial enough to move the
award's wording is exactly the kind that tends to add a new priced row, not just reword an old one.

## Step 3: If anything moved, scope it

For every clause in `changed`/`added`/`removed`, find what cites it. Every branch matches the rule
tables `verify-workflow.md` covers, plus the ones it doesn't (`rule_break_entitlement`, `rule_evidence`)
so a clause touching those isn't silently missed just because semantic verification can't check it yet:

```bash
psql_query "
SELECT 'rule_condition' AS tbl, clauses FROM rule_condition WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_span', clauses FROM rule_span WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_overtime_threshold', clauses FROM rule_overtime_threshold WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_junior_band', clauses FROM rule_junior_band WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_allowance', clauses FROM rule_allowance WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_roster', clauses FROM rule_roster WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_leave', clauses FROM rule_leave WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_break_placement', clauses FROM rule_break_placement WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_break_entitlement', clauses FROM rule_break_entitlement WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
UNION ALL SELECT 'rule_evidence', clauses FROM rule_evidence WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%' AND operative_to IS NULL
"
```

`operative_to IS NULL` means "currently in force." Only the live row needs scoping here — a historical
row that's already closed out from a previous variation isn't affected by this one.

## Step 4: Report — keep it short

```
<CODE> drift check — <today's date>

Operative date: unchanged | moved <old> -> <new>
Clauses changed: <list, or "none">
Clauses added:   <list, or "none">
Clauses removed: <list, or "none">

Pay database: version N loaded — check the FWC database for a newer release (see Step 2b)

Affects N existing rule rows across these tables: <list>, OR "affects nothing currently modelled".

Next step: re-author with history preserved (Step 5), then /autofeature:award-verify <CODE> clauses: <the changed clause list>
```

If a clause was **added** and nothing cites it yet, that's new obligation this service doesn't model at
all — flag it for `/autofeature:award-map`-style triage (expressible / needs-capability / refuse), not
for Step 5, which only re-authors rows that already exist.

## Step 5: Re-author with history preserved

**Never delete a superseded row** — see "Re-authoring after a variation" in `service-conventions.md` for
why. In short: the engine already resolves rules against `operative_from`/`operative_to`, so a shift
dated before this variation is supposed to still price against the old reading, and a blind
delete-and-reinsert throws that away.

The fix lives in the **source SQL files**, not in a live UPDATE against the database, because
`rules-load.sh` reconstructs the full award from files on every `db:reset` — the files have to contain
the full history for a fresh database to have it. For each affected row, in the award's `.sql` file:

1. **Close the old row.** Find its `INSERT` and add a sibling `UPDATE` (or, if the file only ever inserts,
   change the existing row's `operative_to` from its default) so `operative_to` = the new award's
   `operativeFrom` date (the one Step 2 just fetched). Do not delete or edit the old row's predicate —
   it's the historical record of what was in force before.
2. **Insert the new row.** Same clause, corrected predicate, `operative_from` = the new `operativeFrom`
   date, transcribed `clause_text` from the freshly fetched text.
3. **A clause the variation added entirely** gets only step 2 — there's no prior row to close.
4. **A clause the variation removed** gets only step 1 — close it, insert nothing. Update its
   `rule_coverage` note to say so and when.

Then `npm run rules:load` (which deletes and reconstructs from the files — now correctly holding both
the old and new rows) and run `/autofeature:award-verify <CODE> clauses: <the affected list>`, scoped to
the **new** rows only (Step 2 of that command already filters `operative_to IS NULL`, so the closed
historical rows are left alone rather than re-litigated).

A note for whoever builds `scripts/closure.ts`: the current-vs-ever-read distinction in
`service-conventions.md` applies to the axes too. Get it backwards and a re-authored award reports either
false ambiguity or a false coverage gap.

After resolving drift, consider `/autofeature:award-audit <CODE>` — a variation
changes clauses, and a clause whose reading changed may carry a disposition that
no longer fits it.
