---
name: award-rule-tables
purpose: What the rule_* vocabulary can and cannot express. The reference for the expressiveness triage.
status: CUSTOM
---

# The rule vocabulary — what it can say

Use this to triage every pay-affecting or roster-affecting clause of a new award
into one of three buckets.

- **Expressible** — an existing table can say it. Write the row.
- **Needs a capability** — the concept is sound but no table has a field for it.
  Costs a schema change, an engine change and a test. Scope it explicitly or
  refuse it.
- **Refuse** — modelling it faithfully is not possible, or the award defers to
  something outside the system. Write the refusal, cite the clause, move on.

A concept the data model has no field for has **no interpretation, right or
wrong, and will never produce a finding**. That is why the ledger of what cannot
be expressed matters as much as the rules that can.

---

## Pricing

### `rule_clause_group` — which published table covers whom

Keys on employment type (`full_time` | `part_time` | `casual`), worker type
(`standard` | `shiftworker`), hour type (`ordinary` | `overtime`) and employee
kind (`adult` | `junior` | `apprentice` | `adult_apprentice`), with an optional
award-specific `variant`.

Exactly one clause may answer any given employee-and-hour question within one
instrument at one moment, enforced by a unique index.

`rate_adjustment_pp` adds percentage points to the published rate and marks the
result `derived`. It exists solely because General Retail publishes no casual
overtime table. **Check whether the new award publishes one before reaching for
it.** MA000003 publishes casual overtime at A.2.2, so it should not need this at
all, and `verify.ts` check 6 will complain if a mapped clause has rows with no
published figure.

Cannot express: a rate that depends on anything other than these five facts.

### `rule_condition` — when a named condition applies

The only judgement in the system. Predicates available:

| Field | Says |
|---|---|
| `days_of_week` | which days, 0 = Sunday, null = any |
| `time_from` / `time_to` | a local wall-clock window, half-open, `from` after `to` reads as crossing midnight |
| `public_holiday` | must be, must not be, or either |
| `ot_hours_from` / `ot_hours_to` | a band in hours **into that day's overtime**, not into the shift |
| `worker_type` | shiftworkers, everybody else, or either |
| `whole_shift` | evaluate against the shift START and apply to every hour, so a loading earned at 6.00 pm carries past midnight rather than reverting when the clock does |
| `requires_attribute` / `excludes_attribute` | a fact about the employee, where two windows would otherwise both match |
| `clauses` | restrict to one clause's table, for when the Commission's own strings differ in case between two tables |
| `priority` | highest wins, which is how a public holiday outranks the Sunday it falls on |

`penalty_description` is matched **verbatim** against `fwc_penalty`. Names carry
em dashes and curly apostrophes. Copy them from the extract, never retype them.

Cannot express: a condition that depends on the previous shift, on cumulative
hours other than the day's overtime, on the roster pattern, or on anything the
engine would have to look outside the shift to know.

### `rule_span` — the ordinary-hours span, per day

One row per day of week, optionally per worker type. **An hour outside it is
overtime however short the shift**, which is the other way an hour becomes
overtime and has nothing to do with how long somebody worked.

Absence of a row for a day means no ordinary hours may be worked that day, which
is a defensible reading of an absent row and a terrible thing to arrive at by
accident. `verify.ts` check 7 fails an award with fewer than seven days.

Cannot express: a span that varies by store type or trading hours without the
employer asserting it. Retail's cl 15.2 late-trading limbs are handled by
`src/engine/spanLimbs.ts` off an employer assertion, because nothing in the
service knows a newsagency from a hardware store.

### `rule_overtime_threshold` — where ordinary hours end

`daily_hours`, `long_day_hours` (the higher figure allowed on one nominated day
a week), `weekly_hours`, per employment type.

Where the award allows a concession that needs a roster to apply, do not model
it. Not modelling it **over**-reports overtime, and under-reporting overtime is
under-paying. State the choice in the note.

### `rule_junior_band` — turning an age into a published classification name

Maps an age, and optionally a length of service, to the verbatim classification
string the extract uses. Verified against `fwc_classification` at load.

Refuse rather than fall back where the Commission publishes junior rows for only
some levels. Falling back to the adult rate invents a rate.

---

## Roster

### `rule_roster` — what a roster may not do

**`kind` is a CLOSED vocabulary with a `CHECK` constraint.** An award needing a
kind that does not exist is refused rather than approximated. Adding one is a
code change with a test, never a configuration change.

Kinds implemented today:

```
min_engagement            max_daily_hours            meal_break_after
rest_between_shifts       max_consecutive_days       max_days_per_week
consecutive_days_off      max_days_per_cycle         ordinary_hours_continuous
transport_reimbursement   part_time_ordinary_ceiling sunday_days_off
shiftwork_public_holiday_avoidance                   roster_change_notice
no_mixed_shiftwork        roster_period_max          shift_hours_continuous
recall_minimum            outside_agreed_pattern     excess_travel_costs
travelling_time           moving_expenses
```

Every rule declares `window_days`, the surrounding roster it needs either side.
Given less, the engine reports `unevaluated`, never compliant.

`overtime_consequence` is `none` or `whole_day` and nothing else. There is
deliberately no `excess_hours`, because the daily ordinary-hours cap already
converts its excess through `rule_overtime_threshold` and a second path to the
same conversion would double-count it. **Only a rule that can NAME the breaching
day may carry a pay consequence.**

Common concepts with **no kind today**, which a new award may need. Each is a
capability decision, not a configuration one:

- broken shifts and split shifts (a day worked in two or more separate periods)
- sleepovers and on-call or standby periods
- 24-hour care and other whole-of-day engagements
- annualised salary arrangements and their reconciliation
- rostered days off as a work-cycle mechanism rather than as leave
- time off in lieu ledgers, where the award allows accrual instead of payment
- minimum breaks between roster cycles rather than between shifts

### `rule_break_entitlement` — what a shift of a given length is OWED

Bands over hours worked, inclusive at both ends, resolving to the higher band at
a boundary. Carries `hours_from_exclusive` for the awards whose table does not
read the same way at every boundary.

Distinct from the roster rules on purpose. The roster rules say what a roster may
not do; this says what an employee is owed, and a shift can satisfy every
prohibition while still shorting somebody a rest break.

### `rule_break_placement` — where the breaks GO

For awards that forbid a break in the first or last hour, forbid combining a rest
break with a meal break, cap hours worked before one, or want a rest break in
each half of the shift. Needs break TIMES, and a caller sending only a total gets
these back `unevaluated`.

---

## Allowances

### `rule_allowance` — what makes an allowance payable

The amounts stay in the Commission's tables. This says what triggers one.

`trigger` is closed: `employee_attribute`, `location_attribute`, `shift_event`,
`claim`. `unit` is closed: `per_hour`, `per_shift`, `per_week`, `per_occasion`,
`per_km`.

**Nothing is inferred.** Whether somebody holds a first aid certificate is a fact
about them that the service has no way to know and no business guessing.

`all_purpose` marks an allowance that is added to the base **before** penalties
rather than listed beside them. Retail has none. Where an award does have them,
they change the rate every penalty is calculated on, and listing one beside the
rates would understate every penalty on the shift. **The service refuses
all-purpose allowances today.** An award with them needs that refusal honoured
or the capability built deliberately, and it is the single most consequential
capability gap in the vocabulary.

---

## Leave, termination, pay administration

`rule_leave` carries `source` (`nes` or `award`) because "the Act says" and "the
award says" are different claims, and a citation that blurs them is worse than
none. `accrual_method` is closed: `progressive`, `upfront_annual`,
`per_occasion`.

Note `days_per_occasion` versus `hours_per_occasion`. They are not convertible
and treating them as though they were is a live underpayment. A "period of up to
48 hours" is elapsed time, two whole calendar days of being absent, not two days
of rostered hours.

`rule_notice_period`, `rule_notice_uplift`, `rule_redundancy_pay` and
`rule_job_search` cover termination, mostly from the NES with the award's uplift
where it has one. `rule_pay_cycle` and `rule_superannuation` cover pay
administration. `rule_supported_wage` covers the Supported Wage System.

## Procedural obligations

`rule_evidence` and `declaration` hold the clauses that are things an employer
must **do** rather than pay. Coverage, individual flexibility arrangements,
consultation, dispute resolution, record-keeping. Modelled as an evidence
contract with named conditions that must each be asserted, kept with an author
and a date, and reported as gaps rather than computed.

This is the right destination for any clause where the honest answer is that the
employer knows and the service cannot. It is a `By design:` disposition, not a
`Pending:` one.
