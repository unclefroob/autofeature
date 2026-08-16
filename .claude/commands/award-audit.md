---
name: award-audit
description: |
  Read-only audit of an award ALREADY mapped, against the principles the method holds today. Writes nothing, changes nothing, ships nothing.
  Exists because awards are mapped over months and the method keeps learning. Every award shipped before a principle existed is unaudited against it, and "it passed at the time" is not the same as "it passes".
  Invoke as:
    /autofeature:award-audit <AWARD_CODE>
    /autofeature:award-audit all
allowed-tools:
  - Bash
  - Read
  - Grep
---

# AutoFeature — Award Audit

Checks a mapped award against the principles as they stand **now**, not as they
stood when it was mapped. It never edits. Its output is a list of findings and a
recommendation, and acting on any of them is a separate, deliberate decision.

## Missing information is a design task (always active)

"We don't hold that data" is the beginning of the work, not the end of it. This
command's most valuable finding is a residual that stopped at the absence of a
fact instead of designing its capture — so hold the principle while reading, and
do not repeat the error in the report by writing "needs more data" about a
finding.

## Why this exists

An award is mapped once and the method improves for years afterwards. Three
changes in one week made every previously-shipped award unaudited:

- **ACCOUNTED is not IMPLEMENTED** (v1.16.0). "Closed" was printed on eight zero
  axes and read as "done". It meant the award had been *read*.
- **Every `Pending:` names what carries it** (v1.16.1).
- **Missing information is a design task** (v1.17.0). A `Pending:` owes a design,
  not an observation.

None of those existed when MA000004 shipped. It is not defective for that — it
is simply unmeasured against them, and the only way to know is to look.

## Step 1: The two verdicts

```bash
npm run closure -- <CODE>
```

Report ACCOUNTED and IMPLEMENTED separately, and if the award is accounted but
not implemented, lead with the `Pending:` count. Never report the first in words
a reader could hear as the second.

## Step 2: The mechanical checks

Each returns a precise answer with no judgement in it. Run them all; report the
count for every one, including zero, so a clean check is distinguishable from one
that did not run.

```bash
source scripts/lib/psql.sh
```

**A `Pending:` with no capture design.** The finding this command exists for.

```sql
SELECT clause, left(note, 120) FROM rule_coverage
 WHERE instrument_id = '<CODE>' AND note LIKE 'Pending:%'
   AND note NOT ILIKE '%carried by:%'
 ORDER BY clause;
```

**A transcription too short to be verified against.** `verify.ts` check 9 asks
whether the stored text *appears in* the award's text, which a fragment satisfies
trivially. 60 characters is the bar; the award mapped most carefully has a
minimum of 96 and the one mapped fastest had 25 rows under 40.

```sql
SELECT * FROM (
  SELECT 'rule_condition' t, clauses, clause_text FROM rule_condition
   WHERE instrument_id='<CODE>' AND clause_text IS NOT NULL
  UNION ALL SELECT 'rule_span', clauses, clause_text FROM rule_span
   WHERE instrument_id='<CODE>' AND clause_text IS NOT NULL
  UNION ALL SELECT 'rule_overtime_threshold', clauses, clause_text FROM rule_overtime_threshold
   WHERE instrument_id='<CODE>' AND clause_text IS NOT NULL
  UNION ALL SELECT 'rule_allowance', clauses, clause_text FROM rule_allowance
   WHERE instrument_id='<CODE>' AND clause_text IS NOT NULL
  UNION ALL SELECT 'rule_junior_band', clauses, clause_text FROM rule_junior_band
   WHERE instrument_id='<CODE>' AND clause_text IS NOT NULL
) x WHERE length(clause_text) < 60 ORDER BY length(clause_text);
```

**A note too short to say anything.** Under 40 characters. `coverage.test.ts`
already bans under 20; this is the softer bar, and a row near it is worth reading.

```sql
SELECT clause, status, note FROM rule_coverage
 WHERE instrument_id = '<CODE>' AND length(coalesce(note,'')) < 40 ORDER BY clause;
```

**An axis examining an empty universe.** `closure` prints these; repeat them here,
because a zero over nothing is the failure mode this whole method keeps finding
and it must never be reported as a pass.

## Step 3: The candidate finders — SQL narrows, a person reads

These do **not** produce findings on their own. They produce a short list to
read, and the reading is the finding. Say so in the report rather than passing a
regex hit off as a defect.

**A `By design:` whose reason is a fact somebody holds.** `By design:` is for a
judgement no system can make. If the reason is that a *fact* lives somewhere else,
it is probably a `Pending:` with an obvious capture — asking the employer to
assert it, with an evidence contract behind the assertion.

```sql
SELECT clause, left(note, 140) FROM rule_coverage
 WHERE instrument_id = '<CODE>' AND note LIKE 'By design:%'
   AND note ~* 'held by|does not hold|outside this service|another system'
 ORDER BY clause;
```

Read each. *"Turns on whether the employee is eligible for a disability support
pension, which is a fact held by Services Australia"* is the shape to look for:
the fact exists, somebody knows it, and nothing stops the employer asserting it.

**A note carrying several dispositions at once.** Over roughly 700 characters, or
containing more than one ALL-CAPS section marker. One clause, one disposition —
a note that says "this part is modelled, this part is the product's job, this part
is out of scope" is three findings wearing one row, and none of the three can be
counted.

```sql
SELECT clause, status, length(note) FROM rule_coverage
 WHERE instrument_id = '<CODE>' AND status <> 'modelled' AND length(note) > 700
 ORDER BY length(note) DESC;
```

**A note that could have been written about any award.** No clause number, no
figure, no term from this award's own vocabulary. Read the shortest twenty.

### What NOT to flag

A `partial` row **should** say which part is modelled and which is not. A regex
for "is modelled" on a non-modelled row hits that correct pattern far more often
than a real mismatch — it was tried, it returned eleven hits on one award and
every one was right. Do not add it back.

## Step 4: Report

In this order:

1. **The two verdicts**, and the `Pending:` count if not implemented.
2. **`Pending:` without a capture design** — the count, then the clauses. This is
   the backlog that cannot be scheduled, and it is the point of the audit.
3. **Mechanical findings**, each with its count including zero.
4. **Candidates read**, with your reading of each and a recommendation. Where you
   did not reach a view, say that rather than guessing.
5. **What was not checked.** Name it. This audit reads the ledger and the
   transcriptions; it does not re-derive a single predicate from the award's
   words — that is `award-verify`, and a clean audit is not a clean verification.

Do not edit anything. Do not open a branch. The output is a decision for a person:
which findings are worth a pass, and when.

## What this is not

It is not a correctness check. Every finding here is about whether the mapping
*says what it means* — whether a backlog item is schedulable, whether a
transcription can be checked, whether a disposition is the right one. A rule can
pass every check in this file and still be wired to the wrong day of the week.

`award-verify` answers that, and the two are complements: this one reads what was
written about the award, that one re-derives the award from its own text.
