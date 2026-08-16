---
name: award-data-requirements
purpose: Every `Pending:` names which existing channel will carry the fact it is waiting on. Read at Step 6, reported at Step 8. Introduces no new mechanism.
status: CUSTOM
---

# What a `Pending:` is waiting on

## Read this first: there is nothing to build

The service already has two channels for asking a human for a fact, both derived
from what is modelled, both refusing to ask for anything the engine cannot then
use, and both already rendered by the product:

| channel | shape | asked | rendered by |
|---|---|---|---|
| `employee-facts` | `{key, label, type, help, required, options?, clause?, onlyWhen?}` | once, about a person | `Staff.js` → `employment.awardFacts` |
| `RequiredInput` | `{group, key, label, type, help, required}` | in the moment, on a finding | `RequiredInputs.jsx` via `AwardFindings.jsx` |

`RequiredInput` already carries `group`, and the vocabulary is already in use:
`apprenticeship`, `leave`, `otherWorkplace`, `pattern`, `recall`, `rosterPeriod`,
`townshipTransfer`, `transport`.

**A mapping does not add a third channel, a table, or an endpoint.** An earlier
draft of this file specified all three, and it was wrong on its face: the facts a
`Pending:` is waiting on are precisely the ones the engine cannot use yet, so a
runtime endpoint serving them would have no consumer by construction.
`employeeFacts.ts`'s own header says as much — *"It never asks for something the
engine cannot then use. Requesting an apprenticeship year for an award whose
apprentices we cannot price would be theatre."*

## The rule

**Every `Pending:` says which channel will carry its fact once the clause is
modelled, and under what key.** One line, in the note, after the disposition:

```
Pending: <what is missing and why the clause needs it>.
Carried by: employee-facts `expectedRetirementAge`
Carried by: RequiredInput group `pattern`, key `agreedPatternId`
Carried by: neither — <the record the product does not hold>
```

That is the whole addition. It costs a sentence per residual and turns a backlog
of prose into something sortable, countable, and assignable to the team that owns
the channel.

## The three answers, and only the third is a product decision

**`employee-facts`** — a standing fact about the person or the engagement.
Modelling the clause is the whole job; the fact appears in the Staff form the
moment the engine can use it, with no interface change anywhere. This is the
cheap case and most `Pending:` items are it.

**`RequiredInput`** — a fact about this shift, this roster period, this leave
request. Modelling the clause is again the whole job: the rule returns
`unevaluated` with the input attached rather than passing, and the existing
component renders the prompt. Pick the `group` from the list above, or say plainly
that a new group is needed and why the existing eight do not fit.

**`neither`** — the fact cannot be asked for at all, because the product does not
hold the record it would attach to. **This is the only answer that implies work
outside the compliance service**, and it is the reason the sentence is worth
writing.

Two of MA000003's stand out, and both are already known:

- cl 20.2(b)–(d) measure hours against the **rostered** start and finish, as
  distinct from the hours worked. `Shift` carries no rostered times. No prompt can
  ask for this, because there is nowhere to put the answer.
- cl 29 measures a **roster change**, which needs the roster's change history.
  The compliance service is never given one.

Neither is a capture gap. Both are model gaps, and both under-report overtime or
a breach until they are closed — which is why they are `Pending:` and not
`By design:`.

## What Step 8 reports

The review pack groups the `Pending:` residuals by their answer and counts them:

```
carried by employee-facts     N   modelling only
carried by RequiredInput      N   modelling only, group: ...
carried by neither            N   needs a product change first — list them
```

The third number is the one a reader should see first. It is the only part of the
backlog that cannot be burned down inside the compliance service, and an award
whose remaining work is all in the first two categories is much closer to
implemented than a count of 114 suggests.

## The trap

A `Pending:` is not a promise that capturing the fact closes the clause. Some
clauses need the fact **and** a judgement, and those stay `By design:` even once
the fact exists — a genuine agreement under cl 5.2 remains a judgement no matter
how much of the agreement is stored. Recording such a clause as `Pending:`
because a field would help is how a finite backlog stops being finite.
