---
name: award-gap-axes
purpose: The closure axes that make "are there gaps?" a query instead of a question. Read before mapping any award.
status: CUSTOM
---

# Gap axes — how an award mapping finishes

## The failure this file exists to prevent

MA000004 took 95 commits, and the gap waves in that history were not random.
Each wave arrived when a **new axis of enumeration** was introduced, and the
previous axis turned out to have been bounded by a word rather than by a count.
The commit titles say it plainly: *"Rebuild the coverage inventory from the award
text, and find what modelled hid"*, then *"Itemise the award to sub-clause level,
and fail the build when it is not"*, then *"Read the last 74, so no row carries a
status it did not earn"*, then *"Model the last of MA000004: nothing is
not_modelled"*.

Two concrete losses, both recorded in `test/coverage.test.ts`:

- **cl 15.2** (the late-trading span) sat inside clause 15 while clause 15 was
  marked `partial`. Every evening at a store trading past 6.00 pm on a weekend
  priced as overtime when it was ordinary time at a penalty.
- **cl 21.3** (time off instead of payment for overtime, a whole mechanism)
  sat inside clause 21 while clause 21 was marked `modelled` at clause level and
  never itemised. It appeared on no gap list at any point.

The lesson is in that file's own words: **a word cannot be trusted to bound a
list, a count can.**

So the discovery of the axes is what looped, not the modelling. The axes are now
known. A new award gets all of them on day one, and gap discovery is finite
because every axis is enumerated from a source outside the thing being checked.

## Three kinds of finding, and only one is a defect

Classify **every** finding the moment it is found. Perpetual fixing happens when
the second and third categories get triaged as the first.

| Kind | What it is | What it opens |
|---|---|---|
| **Unread** | a clause nobody opened, a published row no rule reaches, an allowance nobody implemented | Work. The axes below close this class completely. |
| **Inexpressible** | a mechanic the closed rule vocabulary has no field for | A ledger entry and a refusal. NOT a fix. Chasing these is an infinite backlog because there is always another one. |
| **Open reading** | the award is genuinely ambiguous and the market does not agree | A per-employer interpretation setting. See `docs/interpretation-settings.md` in the service. |

## The eight axes

Each axis enumerates a universe from **outside** the database, then requires
every member of it to carry a disposition. When all eight return empty, "are
there gaps" has a mechanical answer, and asking it again returns the same answer.

### 1. Every clause of the award has a coverage row

Universe: the award's table of contents, transcribed from the Commission's
consolidated PDF into the enumeration manifest (below). Not read out of the
database, because a clause forgotten in both would agree with itself.

Also checks the reverse, that no row claims a clause the award does not contain.

### 2. Every sub-clause exists, against a transcribed count

Universe: a per-clause count of parts, transcribed from the same PDF. This is
the axis that would have caught cl 21.3 and cl 15.2, and it is the single most
valuable one. Status is irrelevant here. A clause marked `modelled` must account
for every one of its parts exactly as a `partial` one must.

Related invariants, all already enforced:
- no row may carry a note beginning `Inherited from clause` (an inherited status
  is a guess wearing the shape of an answer)
- no row may be titled `Clause 5.9`, that means nobody found its heading
- every sub-clause row must carry `source = 'award_text'`

### 3. Every published penalty row is reachable, and every condition resolves

Universe: `fwc_penalty` rows for the award where `is_heading = false`.

Both directions. A condition naming no published row is a typo whose symptom is
a condition that never fires, quietly leaving every Saturday hour at the ordinary
rate. A published row no condition can reach is the omission the typo check
cannot see. `scripts/verify.ts` checks 1 and 4.

### 4. Every published allowance is implemented, and none is invented

Universe: `fwc_wage_allowance` and `fwc_expense_allowance` for the award.

The strongest completeness check in the service, because **both sides are data**.
One is mirrored from the Commission and never edited, the other is what the
engine actually pays. An allowance published and not implemented is a silent
underpayment every time it is earned.

### 5. Nothing claims to model a clause nobody read

Universe: `rule_coverage` rows for the award.

`status = 'modelled'` with `source <> 'award_text'` is banned. This is the
invariant that would have caught cl 11.3, which was recorded from recollection as
"casual conversion" when it is the 1.5 hour engagement for a secondary school
student, and while it was wrong the engine reported a breach on a roster the
award expressly permits.

Every row must also carry a note of real substance. `modelled` with no note is an
assertion nobody can check and `not_modelled` with no note is a shrug.

### 6. Every rule's text is the award's own words

Universe: every `rule_*` row that carries `clauses`.

`verify.ts` check 9 resolves each citation against `fwc_clause` and requires the
transcribed `clause_text` to appear verbatim in it. Statutory citations are
checked against the Act's text, held in the same table under award code `FWA`.

The first run of this check on retail found cl 4.7 did not exist, cl 35's three
obligations cited clause 34's numbering, and cl 5.1's transcription had dropped
the phrase that limits when an individual flexibility arrangement is available.

### 7. Every modelled engine is reachable from a route

Universe: the engine modules under `src/engine/`.

Nine of them were written, unit-tested, recorded as `modelled` in the coverage
ledger, and had no route. Dead code behind a green suite, invisible from both
directions at once, because a unit test proves the arithmetic and says nothing
about whether a caller can get at it. `test/reachable.test.ts`.

### 8. Every residual carries a disposition

Universe: every `rule_coverage` row not marked `modelled`.

Each one must begin with one of two words:

- **`Pending:`** solvable later, usually by collecting a fact the product does
  not yet hold. This is the backlog, and it is finite and listed.
- **`By design:`** correctly ends at an assertion, an evidence contract, or the
  boundary of what any rostering system can know. Closing one would mean
  pretending to verify a judgement or a fact held outside the system, which is
  worse than the gap.

A new residual is classified at the moment it is written, never left for an audit
to find.

## The enumeration manifest

Axes 1 and 2 are the only ones whose universe cannot be derived mechanically.
They come from a human reading the Commission's consolidated PDF. That
transcription is the artefact everything else depends on, so it is committed,
reviewable, and pinned to a compilation date.

Create `src/awards/<CODE>/enumeration.ts`:

```ts
/**
 * <CODE> — the award's own table of contents.
 *
 * Transcribed from the Fair Work Commission's consolidated PDF,
 * compilation of <DATE>, retrieved <DATE> from <URL>.
 *
 * Hardcoded ON PURPOSE and held outside the database. A clause forgotten in
 * both the coverage table and this list would agree with itself and every
 * check would pass.
 */
export const CLAUSES: string[] = [ /* "1", "2", ... "A", "B", ... */ ];

/**
 * How many parts each clause has.
 *
 * A word cannot be trusted to bound a list. A count can. This is the axis that
 * would have caught cl 21.3 in MA000004, where a clause marked `modelled` hid
 * a whole mechanism.
 */
export const SUBCLAUSE_COUNT: Record<string, number> = { /* "1": 3, ... */ };
```

`test/coverage.test.ts` and `scripts/closure.ts` both read this file. The
invariant that the list comes from outside the database is preserved, because a
committed TypeScript file is outside the database.

## One command, per award

The eight axes exist today but are split across two places, four inside
`scripts/verify.ts` (which already takes every award) and four inside
`test/coverage.test.ts` (hardcoded to MA000004 and only reachable via
`npm test`). There is no single thing anyone can run that answers "are there gaps
in this award" and gives the same answer to whoever asks.

**That split is why the question kept having a new answer.** Ask an agent and it
goes looking, and looking finds whatever it happens to look at.

So an award-mapping run stands up `scripts/closure.ts`, which takes an award
code, runs all eight, and prints one table:

```
$ npm run closure -- MA000003

  axis                                       open
  1  clauses without a coverage row              0
  2  sub-clauses missing against the count       0
  3  published rows no condition reaches         0
  4  published allowances not implemented        0
  5  rows claiming modelled without award_text   0
  6  rules whose text is not the award's words   0
  7  modelled engines with no route              0
  8  residuals with no disposition               0

  MA000003: closed.
```

Non-zero anywhere means it prints the list. Zero everywhere is the definition of
done, and it is the only definition of done a mapping run is permitted to use.

## What the axes cannot catch, and what does

They cannot catch a clause that was read, cited, and transcribed correctly but
wired to the wrong rule — a `Saturday` condition sitting on `days_of_week = {0}`
passes every axis here, because every axis checks *existence and reachability*,
never *does the predicate say what the clause says*. That is a different
question, and it does not get a ninth axis, because it can't be answered with a
boolean the way the eight above can.

Two things answer it instead, and they are deliberately not the same mechanism:

- **`awards/verify-workflow.md`** (run as Step 7.5 of `award-map.md`, or
  standalone via `/autofeature:award-verify`) — independent re-derivation of
  every predicate-bearing row from its clause text alone, diffed against what
  shipped, with a third adversarial pass only where two blind readers disagree
  with it. This is the closest thing to a mechanical check the correctness
  question gets, and it covers every row, not a sample.
- **The hand-checked scenario suite** (Step 7) — for the class of bug the row-by-
  row check can't see at all: several individually-correct rules interacting
  wrongly over one real shift, a boundary, a priority resolving in a way nobody
  anticipated.

Both raise confidence. Neither is a proof, and the closure table should never be
presented as one — see the sign-off gate at Step 8 of `award-map.md`.
