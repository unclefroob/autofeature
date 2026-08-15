---
name: award-service-conventions
purpose: Authoritative context on rosterio-compliance-service for any award-mapping run
status: CUSTOM
---

# rosterio-compliance-service — conventions

Read this in full before touching an award. It is the shape of the thing being
extended, and almost every rule in it exists because something went wrong once.

## The one idea

Two layers, and the boundary between them is the whole design.

| Layer | What it is |
|---|---|
| `fwc_*` | The Commission's own extracts, mirrored one row to one row. Never hand-edited, never interpreted. If a figure here is wrong, it is wrong in the published data. |
| `rule_*` | Our reading. When a named condition applies, where the span of ordinary hours falls, what makes an allowance payable, what a roster may not do. This is the only place judgement lives, and it is per-award. |

Pricing is a **lookup, not a calculation**. The Commission has already done the
multiplication and publishes a dollar figure for every combination of
classification, employee type and named condition. The only thing left to get
wrong is which row to look up.

That has a direct consequence for mapping. **Never write a rate.** A rule names
a `penalty_description` verbatim and the engine fetches the Commission's figure
behind it. A rule that carries a number the Commission did not publish is a
defect even when the number is right, because next July it will be wrong and
nothing will say so.

## What is already award-agnostic

- **All 155 awards' rates are already loaded.** The five FWC workbooks are
  all-award extracts and `scripts/load.ts` creates an `instrument` row per award
  code. A new award needs no data import.
- **`scripts/rules-load.sh` loops over every directory under `src/awards/`.** A
  new `src/awards/<CODE>/` folder is picked up with no code change.
- **`scripts/verify.ts` runs across every mapped award**, not just retail.
- The engine itself contains six references to `MA000004`, only one of them a
  live query (a parental-leave lookup in `src/engine/parentalLeave.ts`).

## What is NOT yet award-agnostic

Four known hardcodes. Check each against the award being mapped and fix the ones
that bite, as part of the run.

| Where | What | Bites when |
|---|---|---|
| `src/data/rates.ts` (`ORDINARY_WEEK_HOURS`) | `38` hardcoded for turning a weekly rate into an hourly one | the award's ordinary week is not 38. Should read `rule_overtime_threshold.weekly_hours`. |
| `src/engine/employeeFacts.ts` | classification levels filtered with `classification_level ~ '^[0-9]+$'` | the award's levels are not plain integers (grades, pay points, named levels) |
| `test/golden/published-rates.test.ts` | `const AWARD = "MA000004"` plus a `shiftFor()` switch keyed on retail's condition names | always. The README's claim that this suite picks a new award up on its own is not true today. |
| `test/coverage.test.ts` | `const AWARD = "MA000004"`, `CLAUSES` and `SUBCLAUSE_COUNT` inline | always. See the enumeration manifest in `gap-axes.md`. |

## File layout for an award

Every award is a directory of SQL under `src/awards/<CODE>/`, applied in the
order `scripts/rules-load.sh` names. **The order is load-bearing.** Only
`rules.sql` is required; an award that does not have a part simply has no file.

```
rules.sql                  clause groups, conditions, spans, thresholds, junior bands
coverage.sql               one row per top-level clause
coverage-subclauses.sql    one row per sub-clause (depends on coverage.sql)
coverage-schedules.sql     schedules itemised
coverage-parents.sql       corrects parent notes against the children above
allowances.sql
roster.sql
shiftwork.sql
apprentices.sql
breaks.sql
break-placement.sql
leave.sql
pay-admin.sql
termination.sql
supported-wage.sql
evidence.sql
evidence-remaining.sql
evidence-procedural.sql
award-text.sql             replaces hand-typed clause_text with the award's own words
coverage-dispositions.sql  stamps Pending: / By design: on every remaining gap
```

`award-text.sql` and `coverage-dispositions.sql` go **last**, in that order.
Run them earlier and they classify rows that later files then rewrite.

## Doctrine

**A refusal is a 422 with a code, never a zero.** The single most damaging thing
the predecessor engine did was price an unpriceable shift at $0 and let it
through payroll. Where the award needs something that cannot be modelled
faithfully, the answer is a refusal with a reason, not an approximation.

**`published` versus `derived`.** A rate read straight from the Commission's
table is marked `published`. Anything the engine had to compute is `derived` and
must be justified by a clause and checked against a second answer key. Retail
needs exactly one derivation, casual overtime, because Part B publishes no
casual overtime table. `verify.ts` check six fails if a mapped clause has rows
with no published figure, so derivations cannot spread quietly.

**The roster-rule vocabulary is CLOSED.** `rule_roster.kind` has a `CHECK`
constraint listing every kind the engine implements. The schema comment states
why in terms worth repeating: the failure mode of every configurable award
engine is that an award needs something the format cannot express, a flag gets
added, and after thirty awards nobody understands the interaction. **Adding a
kind is a code change with a test, never a configuration change, and an award
needing a kind that does not exist is refused rather than approximated.**

The same closed-vocabulary rule applies to `rule_allowance.trigger` and `.unit`,
`rule_leave.accrual_method`, and `rule_roster.overtime_consequence`.

**Unevaluated is not compliant.** Every roster rule declares how much
surrounding roster it needs in `window_days`, and reports `unevaluated` when it
was not given enough. A rule that silently returns "fine" because it lacked the
data to say otherwise is the exact quiet wrongness the service exists to avoid.

**Nobody writes to the award layer.** An employer's variations live on their own
instrument, which chains out to the modern award at the root. That is what keeps
the annual wage review a data reload rather than a migration.

**Readings the award leaves open are settings, not gaps.** Where the award is
genuinely ambiguous and the market does not agree, the reading becomes a
per-employer setting recorded against their instrument, not a decision taken
here. `docs/interpretation-settings.md` holds the doctrine and the constraint
that only a rule which can NAME the breaching day may carry a pay consequence.

**Re-authoring after a variation never deletes a superseded row.** Every
`rule_*` table already carries `operative_from`/`operative_to`, and the engine's
query layer already resolves against them (`asOf`/`PointInTime` in
`src/data/rates.ts`, `src/data/instrument.ts`) — a shift dated before a
variation is meant to still price against the reading that was actually in
force then. `rules-load.sh`'s blanket `DELETE FROM rule_* WHERE instrument_id =
'<CODE>'` is correct and safe for a **first** mapping, where there's no history
yet to lose. It is not safe for a re-mapping: the fix is to close the old row's
`operative_to` in the source SQL file and add a new row from the new date,
never to edit the old row's predicate or remove it, because the file is
reconstructed from scratch on every `db:reset` and has to hold the full
history for a fresh database to have it. See `award-drift.md` Step 5 for the
full procedure. Any completeness check answering "is the CURRENT reading
correct" must filter `operative_to IS NULL`; any check answering "was this
clause ever read" must not, since a clause doesn't stop being read because its
reading later changed.

## Commands

```bash
npm run db:up                    # Postgres in podman
npm run db:load                  # the five workbooks
npm run award:fetch -- <CODE>    # the award's own words, from the FWO library
npm run award:load               # into fwc_clause
npm run rules:load               # apply every award's SQL, in order
npm run verify                   # our reading against the Commission's data
npm test                         # every test, against a real loaded database
```

Raw source files are **not committed** (they are the Commission's to distribute).
Their checksums go in `data/raw/CHECKSUMS.txt`, which is committed, and that file
is what pins which version of the award the rules were checked against. Adding an
award means adding its line there.
