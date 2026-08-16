---
name: award-data-requirements
purpose: What the product must capture before an award can be implemented, derived from the mapping rather than guessed by the frontend. Read at Step 6 and emitted at Step 8.
status: CUSTOM
---

# Data requirements — telling the product what the award needs

## The gap this closes

The service can already answer *"what can you use?"* — `employeeFacts.ts` derives
it from what is modelled, and its own header is candid about the limit:

> *"the depth of the answer follows the depth of the modelling, and for an award
> we have barely interpreted the honest answer is short."*

Nothing can answer the other question: **"what would you need?"** That direction
is not derivable from what is modelled, because the whole point is that it is
about what is *not*. It has to be recorded at the moment a `Pending:` is written,
by the person who just read the clause and knows exactly which fact was missing.

Without it, a mapping produces a backlog of 114 items in prose, and the team who
would have to build the capture has no list, no grain, no counts and no order.
That is what MA000003 shipped.

## The rule

**Every `Pending:` residual names the fact that blocks it.** A note reading
"not modelled yet" is not a disposition, it is a shrug wearing the shape of one.

Recorded in `rule_data_requirement`, one row per (clause, fact):

```
instrument_id   the award
clause          the clause that cannot be implemented without this
fact_key        stable key, e.g. 'agreed_pattern', 'rostered_start_finish'
grain           employer | employee | employment | shift | roster | leave_request
label           what a person would call it, for the capture UI
why             the sentence from the clause that needs it
blocks          what stays unavailable until it exists, in product terms
errs            which direction the omission errs — see below
```

`fact_key` is a **closed vocabulary per service**, not free text. Two clauses
needing the same fact must use the same key, or the product builds the same
capture twice and the count of remaining work is wrong in both directions.

## Grain is the field that decides who builds it

The most common way this goes wrong is recording a fact at the wrong level, so
the capture lands in the wrong screen and gets built twice or not at all.

| grain | captured where | example |
|---|---|---|
| `employer` | account settings | is this a small business employer (s121) |
| `employee` | the person's record | date of birth, first aid certificate |
| `employment` | the engagement | the agreed pattern of days and times (cl 10.3) |
| `shift` | the roster or the timesheet | rostered start and finish as distinct from worked |
| `roster` | the roster period | the change history a consultation clause measures |
| `leave_request` | the request | the reason a carer's leave absence is claimed for |

A fact recorded at `employee` grain that is really `employment` grain is the
mistake that produced a week of rework on Rosterio's shiftwork flags: two
employments of the same person can differ, and a field on the person cannot say
so.

## `errs` is not optional

Every requirement states which way its absence errs, in the `notes` style the
service already uses:

- **under-reports overtime** → under-pays → urgent, and say so
- **over-reports a breach** → an employer sees a warning the award does not
  support → annoying, not dangerous
- **cannot answer at all** → the engine refuses, which is safe but useless

Ordering the backlog by anything other than this produces a list sorted by how
easy each item looked.

## What it is FOR, and the trap in that

The output is a build list for the product team: the capture that has to exist
before the award can be implemented. It is deliberately in the award's language
and the product's, not SQL's.

**The trap:** a data requirement is not a promise that collecting the fact closes
the clause. It says the clause cannot be implemented *without* it. Some clauses
need the fact AND a judgement, and those are `By design:` even once the fact
exists — a genuine agreement under cl 5.2 stays a judgement no matter how much
of the agreement is captured. Recording such a clause as `Pending:` because a
field would help is how a backlog stops being finite.

## Emitting it

Step 8's review pack carries the requirements grouped by grain, with counts, so a
reader sees "this award needs four things at employment grain and two at shift
grain" rather than 114 lines of prose.

Serve it as `/v1/instruments/:id/data-requirements`, the mirror of
`/v1/instruments/:id/employee-facts`:

- **employee-facts** — what the engine can use today. Shrinks to nothing for an
  unmapped award, correctly.
- **data-requirements** — what it would need to finish. Largest for an unmapped
  award, and shrinks to nothing as the mapping completes.

The two together are the whole picture, and `data-requirements` reaching empty is
the same event as `Pending:` reaching zero. One number, two views.
