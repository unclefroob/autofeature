---
name: award-drift
description: |
  On-demand check for whether a mapped award's wording has moved since it was last mapped or verified. Re-fetches the award text, diffs it clause-by-clause against what's stored, and names exactly which rule rows are affected.
  Manual trigger, not a cron — award variations are infrequent, so this is meant to be run when you have reason to suspect one landed, not on a schedule.
  Invoke as: /autofeature:award-drift <AWARD_CODE>
allowed-tools:
  - Bash
  - Read
  - Write
---

# AutoFeature — Award Drift Check

Answers one question: **has this award's own wording changed since we last checked it?** Rate changes
(Annual Wage Review) already flow through automatically on the next `npm run db:load` — that's the
lookup design working as intended, nothing to check here. This is about the award's *text* moving:
a new penalty condition, a widened span, a changed roster concession.

## Step 1: Resolve and snapshot what's currently stored

```bash
CWD=$(pwd); PARENT=$(dirname "$CWD")
for c in "$CWD" "$PARENT/rosterio-compliance-service" "$CWD/rosterio-compliance-service"; do
  [ -f "$c/db/schema.sql" ] && { SVC="$c"; break; }
done
cd "$SVC"
cp "data/raw/award-text-<CODE>.json" /tmp/award-text-<CODE>-before.json
```

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

If all three lists are empty and the operative date matches: **nothing moved.** Say so plainly and stop
— this is the expected, common result.

## Step 3: If anything moved, scope it

For every clause in `changed`/`added`/`removed`, find what cites it:

```sql
SELECT 'rule_condition' AS tbl, clauses FROM rule_condition WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%'
UNION ALL SELECT 'rule_span', clauses FROM rule_span WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%'
UNION ALL SELECT 'rule_roster', clauses FROM rule_roster WHERE instrument_id = '<CODE>' AND clauses LIKE '<clause>%'
-- repeat for rule_overtime_threshold, rule_junior_band, rule_allowance, rule_leave,
-- rule_break_entitlement, rule_break_placement, rule_evidence
```

## Step 4: Report — keep it short

```
<CODE> drift check — <today's date>

Operative date: unchanged | moved <old> -> <new>
Clauses changed: <list, or "none">
Clauses added:   <list, or "none">
Clauses removed: <list, or "none">

Affects N existing rule rows across these tables: <list>, OR "affects nothing currently modelled".

Next step: /autofeature:award-verify <CODE> clauses: <the changed clause list>
```

If a clause was **added** and nothing cites it yet, that's new obligation this service doesn't model at
all — flag it for `/autofeature:award-map`-style triage (expressible / needs-capability / refuse), not
for `award-verify`, which only re-checks rows that already exist.

Do not reload `award-text.sql` or run `npm run award:load` from this command. That's a decision for
whoever reads the diff, not something to do automatically on a wording change nobody has triaged yet.
