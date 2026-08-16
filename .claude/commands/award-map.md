---
name: award-map
description: |
  Map an Australian modern award into rosterio-compliance-service so it can be priced and roster-checked, with every rule citing a clause and mechanically checkable against the award's own words.
  Enumeration-first, deliberately: transcribe the award's clause list and per-clause sub-clause counts from the Commission's consolidated PDF, triage every clause against the closed rule vocabulary, stand up the eight closure axes to produce a FIXED work list, and only then author SQL against a list that can only shrink.
  Exists to kill the perpetual-gap loop. "Are there gaps?" becomes one command that returns a list, instead of a question an agent answers differently every time it is asked.
  Closure proves coverage, not correctness — Step 7.5 runs /autofeature:award-verify to independently re-derive every predicate-bearing row from its clause text and adversarially check it against what shipped, closing the one gap closure can't: a rule that's reachable and cites the right clause but got the day, the time or the priority wrong.
  A PR merging is not the award going live — Step 9.5 gates promotion to the one deployed database (no staging tier exists today) behind a re-run of verify, award-verify and closure against it, and explicit user sign-off.
  Invoke as: /autofeature:award-map <AWARD_CODE> [mode:checkpoint|automated] [resume]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebFetch
  - WebSearch
  - Agent
  - Workflow
  - Skill
  - TaskCreate
  - TaskUpdate
  - TaskList
  - Monitor
---

# AutoFeature — Award Mapping

Maps one modern award into `rosterio-compliance-service`. This is **not** the
feature pipeline. It borrows autofeature's branch/review/ship machinery at the
end and nothing else, because the work here is reading an award, not designing
software.

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

**Read all four of these in full before Step 1, and treat them as
authoritative:**

- `$AUTOFEATURE_HOME/awards/service-conventions.md` — the service being extended
- `$AUTOFEATURE_HOME/awards/gap-axes.md` — the eight axes, and why ACCOUNTED and
  IMPLEMENTED are two verdicts rather than one
- `$AUTOFEATURE_HOME/awards/data-requirements.md` — which existing channel each
  `Pending:` is waiting on, and which ones need a product change first
- `$AUTOFEATURE_HOME/awards/rule-tables.md` — what the vocabulary can and cannot say
- `$AUTOFEATURE_HOME/awards/verify-workflow.md` — semantic verification, run at Step 7.5

## Args

- `<AWARD_CODE>` — required, e.g. `MA000003`. Refuse to proceed without one.
- `mode:checkpoint` (default) — stop at every checkpoint below.
  `mode:automated` — run through, but the four **hard** checkpoints still stop:
  the transcription (Step 2), the expressiveness triage (Step 4), the human
  sign-off on the closure + semantic-verification report (Step 8), and
  promotion to the live database (Step 9.5). They are where judgement is
  exercised, where the mapping's claim to be trustworthy is earned, or where
  the action stops being reversible, and none of the four may be skipped.
- `resume` — pick up from the state file, see Resume.

## Iron Laws

1. **Never write a rate.** A rule names a `penalty_description` verbatim and the
   engine fetches the Commission's figure. A number the Commission did not
   publish is a defect even when it is right.
2. **Never approximate a closed vocabulary.** `rule_roster.kind`,
   `rule_allowance.trigger`, `rule_allowance.unit`, `rule_leave.accrual_method`
   and `rule_roster.overtime_consequence` are `CHECK`-constrained. An award
   needing a value that does not exist is **refused and written into the ledger**,
   never bent into the nearest existing one.
3. **Never mark a clause `modelled` from recollection.** `source` must be
   `award_text`, meaning somebody opened it.
4. **Classify every finding on the spot** as unread, inexpressible, or open
   reading (`gap-axes.md`). Only the first opens work.
5. **Done is the closure run, not an opinion.** No step reports completion on any
   other basis.

## Pipeline

```
0 preflight ─ 1 acquire ─ 2 transcribe [HARD CHECKPOINT] ─ 3 shape ─
4 triage [HARD CHECKPOINT] ─ 5 stand up closure → WORK LIST ─
6 author (burn down) ─ 7 scenario suite ─ 7.5 semantic verification ─
8 closure run + review pack [HARD CHECKPOINT] ─ 9 ship ─
9.5 promote [HARD CHECKPOINT]
```

The order matters and was arrived at the hard way. The closure axes cannot run
before Step 2, because two of them take the transcription as input. They must run
before Step 6, because their first output **is** the work list, and a mapping
that burns down a list fixed in advance cannot drift the way one that reads and
then audits does.

## State file

Everything is recorded in `.autofeature/awards/<CODE>-map.md` in the service repo
as it happens: decisions, refusals, transcription provenance, closure runs. On
`resume`, read it and continue from the last completed step.

---

## Step 0: Preflight

Resolve the service repo and prove the baseline is green **before** anything is
added to it. A mapping built on top of a red suite cannot be told from a mapping
that broke it.

```bash
CWD=$(pwd); PARENT=$(dirname "$CWD")
for c in "$CWD" "$PARENT/rosterio-compliance-service" "$CWD/rosterio-compliance-service"; do
  [ -f "$c/db/schema.sql" ] && [ -d "$c/src/awards" ] && { SVC="$c"; break; }
done
[ -z "$SVC" ] && { echo "Run from rosterio-compliance-service or its parent."; exit 1; }
cd "$SVC"
source scripts/lib/psql.sh
```

**There is deliberately no `psql` binary assumed on this machine.** `scripts/lib/psql.sh` is the
service's own answer — it routes every query through a throwaway container (or the host network, once
`DATABASE_URL` points at a deployed database) instead of requiring a system install. Use its
`psql_query "<sql>"` for a read and `apply_sql "<path>"` for a file, everywhere in this pipeline — a bare
`psql` invocation will not exist to run.

Then, in order, stopping on the first failure:

1. `git status --porcelain` is clean, or the user confirms the working tree.
2. Database up and loaded. Check it directly rather than via the service's
   `/health`, because nothing says the service is running during a mapping:
   `psql_query "SELECT count(*) FROM fwc_penalty"` returning thousands of rows
   means loaded. Otherwise `npm run db:up && npm run db:load`, then
   `npm run award:load && npm run nes:load && npm run rules:load` — and note that
   none of those `npm run` scripts source `.env` on their own (there is no
   dotenv here); `export $(grep -v '^#' .env | xargs)` or equivalent first, or
   `DATABASE_URL is not set` is the result.
3. `npm run verify` passes.
4. `npm test` passes. Record the test count in the state file. It is the baseline
   the final run is compared against.

4b. **Snapshot every award already mapped**, and commit the result before a line
   of the new award is written:

   ```bash
   for a in $(npx tsx -e 'import{awardsWithEnumerations}from"./src/awards/enumerations";console.log(awardsWithEnumerations().join(" "))'); do
     npm run snapshot -- "$a"
   done
   git status --porcelain test/fixtures/snapshot-*.json   # expect empty
   ```

   A snapshot is every priced answer an award gives across the cross-product of
   classification, employment type, age and shift shape — the combinations
   nobody wrote a test for. `test/snapshot.test.ts` compares against it on every
   run.

   Output that moves HERE, before any new work, is not the new award's doing. It
   means the committed snapshot was stale or the local database differs from the
   one it was written against, and either has to be resolved now, because from
   this point on the snapshot is the only thing that can tell "the new award
   changed the old one" from "it was already like that".

   Non-empty output at this step is a Step 0 failure like any other. Do not
   regenerate to make it quiet.

   **An award mapped before the snapshot existed has no honest baseline here.**
   Writing one now freezes whatever it currently answers, regressions included,
   and the file then reads as evidence when it is only a starting point. Say so
   in the state file the first time, or reconstruct the real baseline:

   ```bash
   git worktree add --detach /tmp/pre "<commit before the last mapping began>"
   psql_query "CREATE DATABASE award_baseline"          # a second, isolated database
   # in the worktree, with DATABASE_URL pointing at award_baseline:
   #   npm run db:schema && npm run db:load && npm run award:load \
   #     && npm run nes:load && npm run rules:load
   # then run the same matrix through the OLD engine and diff the two files
   ```

   `data/raw` is not committed, so symlink it from the working repo or the load
   finds nothing and the comparison is against an empty award.

   This is worth doing rather than assuming. Run against the award already in
   production it found no pricing regression across 1920 answers — and it also
   found that the harness itself was refusing every leave cell on a bad input,
   and that one engine's refusal code varied between identical calls. Both were
   invisible to a suite of a thousand passing tests.
5. The award exists in the extract:
   ```bash
   psql_query "SELECT award_code, name, version_number FROM fwc_award WHERE award_code = '<CODE>'"
   ```
   No row means the code is wrong or the workbooks are stale. Stop.
6. Nothing is already mapped:
   ```bash
   psql_query "SELECT count(*) FROM rule_clause_group WHERE instrument_id = '<CODE>'"
   ```
   Non-zero means this is a re-run. Switch to `resume` semantics rather than
   deleting anything.

If the baseline is red, **stop and report**. Do not map on top of failing tests.

With the baseline green, branch now, not at ship: `git checkout -b award/<code-lowercase>`.
Everything from Step 1 onward happens on the branch, so main never carries a
half-mapped award.

## Step 1: Acquire the award's own words

```bash
npm run award:fetch -- <CODE>
npm run award:load
```

Then confirm what landed, because the fetcher has been wrong before (three
clauses once went missing when inline tags stopped being replaced by spaces, and
seven schedule roots vanished behind a regex that required whitespace):

```sql
SELECT count(*) AS clauses,
       count(*) FILTER (WHERE text = '') AS headings_only
  FROM fwc_clause WHERE award_code = '<CODE>';
SELECT clause, heading FROM fwc_clause
 WHERE award_code = '<CODE>' AND clause ~ '^[A-G]$' ORDER BY clause;
```

Every schedule the award has must appear. If schedules are missing, fix the
fetcher before going further, and say so in the state file.

Add the checksum line to `data/raw/CHECKSUMS.txt` with the retrieval date, the
source URL and the operative date the loader captured. **That line is what pins
which version of the award the rules were checked against**, and it is the only
part of the source material that gets committed.

`load-award-text.ts` refuses when the fetched text's operative date does not
match the version the rate extracts rest on. The library serves whatever is in
force TODAY, so a refusal means a variation has landed since the workbooks were
released, and the text would be from a version of the award the figures do not
describe. **Stop and report; never bypass the check.** Text from the wrong
version is worse than no text, because it reads as verified.

## Step 2: Transcribe the enumeration — HARD CHECKPOINT

The two axes that cannot be derived mechanically. This is the artefact every
later check depends on, so getting it wrong is the only way the closure run can
lie to you.

Source is the **Commission's consolidated PDF** for the award, not the database,
not the fetched text, and not the HTML the fetcher parsed. Fetch it and read the
table of contents, then read each clause heading to count its parts.

Write `src/awards/<CODE>/enumeration.ts` in the shape `gap-axes.md` specifies,
with a header comment naming the compilation, the retrieval date and the URL.

Creating that directory has a side effect: `rules-load.sh` treats an award
directory without `rules.sql` as an **error**, deliberately, so every
`npm run rules:load` between here and Step 6 would die. Write a stub
`rules.sql` alongside the enumeration — the `DELETE FROM rule_* WHERE
instrument_id = '<CODE>'` header inside a `BEGIN`/`COMMIT` and a comment saying
Step 6 replaces it. It inserts nothing, so nothing downstream can mistake it
for a mapping.

Cross-check the transcription two ways before presenting it:

```sql
-- Clauses the fetched text has that the transcription does not, and vice versa.
SELECT clause FROM fwc_clause WHERE award_code = '<CODE>' AND clause !~ '\.'
 ORDER BY clause;
```

A disagreement is information, not an error to smooth over. The PDF wins, and the
disagreement goes in the state file, because it usually means the fetcher lost
something.

**Stop and present to the user:** the clause list, the sub-clause counts, the
count of each, the source and compilation date, and any disagreement with the
fetched text. This is a checkpoint in `mode:automated` as well.

## Step 3: Read the shape of the extract

What the Commission actually publishes for this award, which is the universe axes
3 and 4 enumerate. Pure SQL, no judgement:

```sql
-- The clause tables that carry rates, and who each is headed for.
SELECT DISTINCT clauses, clause_description
  FROM fwc_penalty WHERE award_code = '<CODE>' ORDER BY clauses;

-- Every named condition, per clause.
SELECT clauses, penalty_description, count(*) AS rows
  FROM fwc_penalty WHERE award_code = '<CODE>' AND is_heading = false
 GROUP BY 1,2 ORDER BY 1,2;

-- Classifications, and whether juniors are published for every level.
SELECT employee_rate_type_code, classification, classification_level, count(*)
  FROM fwc_classification WHERE award_code = '<CODE>' GROUP BY 1,2,3 ORDER BY 1,3,2;

-- Allowances, both kinds.
SELECT 'wage' AS src, allowance FROM fwc_wage_allowance
 WHERE award_code = '<CODE>' AND is_heading = false
 UNION ALL SELECT 'expense', allowance FROM fwc_expense_allowance
 WHERE award_code = '<CODE>' AND is_heading = false ORDER BY 1,2;

-- Anything the Commission has NOT priced, which decides whether this award can
-- be mapped without deriving figures.
SELECT clauses, count(*) FROM fwc_penalty
 WHERE award_code = '<CODE>' AND is_heading = false
   AND penalty_calculated_value IS NULL GROUP BY 1;
```

Record the answers verbatim in the state file. Condition names are matched
**exactly** later, em dashes and curly apostrophes included, so they get copied
from here and never retyped.

Also check the four known hardcodes in `service-conventions.md` against what this
award needs, in particular whether its ordinary week is 38 and whether its
classification levels are plain integers.

## Step 4: Expressiveness triage — HARD CHECKPOINT

Walk **every** clause and sub-clause in the enumeration and assign each one:

| | |
|---|---|
| `expressible` | which table and which fields say it |
| `needs_capability` | what field or `kind` is missing, and what building it costs |
| `refuse` | why it cannot be modelled faithfully, and what the API returns instead |
| `not_applicable` | it names the instrument, defines terms, or is otherwise nothing to act on |
| `evidence` | an obligation the employer must DO rather than pay, so an evidence contract |
| `open_reading` | genuinely ambiguous, so a per-employer interpretation setting |

Fan this out with `Agent` over clause families rather than doing it in one
context. One agent per family, each given `rule-tables.md`, the fetched text for
its clauses, and the Step 3 shape. Model per `orchestrator/model-tiers.md`, with
this step pinned to **opus**, because it is the judgement the whole run rests on
and it is where the closed vocabularies get respected or quietly bent.

Every entry cites its clause and quotes the sentence it turns on.

Every `open_reading` entry gets a destination, not just a label: it is written
into `docs/interpretation-settings.md` following that file's own rules, in
particular that **only a rule which can name the breaching day may carry a pay
consequence**, and the setting defaults to the conservative reading.

### A capability is not built until it can reach production

Every approved `needs_capability` item is a schema change, and a schema change
that exists only in `db/schema.sql` reaches a FRESH database and never a
populated one — `schema-load.sh` finds the schema present and skips. Mapping the
second award added five columns and tables this way; three had no migration, the
whole test suite passed against a schema production did not have, and the first
priced shift in production would have raised `column "classification_levels" does
not exist`.

So an approved capability owes three things, not one: the column, a file in
`db/migrations/`, and a passing `test/schema-drift.test.ts`. Say so at the
checkpoint, because the cost quoted for a capability is wrong without them.

### The second award finds bugs in the first — expect it, and ship them separately

The most valuable output of a second mapping is not the second award. It is the
defects a different award SHAPE exposes in code that only ever had to serve one:

- a Sunday penalty split by classification, which retail does not have, found
  `rule_condition` with no classification predicate — **every level 2 and 3
  employee's Sunday priced 25 points light**, silently, on the highest-volume
  penalty day
- a casual-majority workforce found s123 excluding casuals from notice but not
  redundancy, so a nine-year casual showed nil notice beside sixteen weeks
- fetching a second award's NES text found a parser embedding page headers in
  seven sections of the Act already being cited as verified

None of those is `needs_capability` in the vocabulary sense and none belongs in
this award's branch. **They fix an award already in production, so they ship
first and separately**, on their own merits and their own review. Holding a live
underpayment behind an unfinished mapping is the wrong trade.

Record them in the triage as a fourth list — *defects in shared code this award
revealed* — and get a decision on each before Step 5.

**Stop and present to the user:** the triage table, and separately and
prominently the `needs_capability` and `refuse` lists, which together are the
ledger of what this award will not be able to say. Ask which `needs_capability`
items are in scope for this run. Everything not approved becomes a refusal with a
`By design:` or `Pending:` disposition. This is a checkpoint in `mode:automated`
as well.

## Step 5: Stand up the closure axes — the work list

Now, and not before, build the thing that decides when this is finished.

**5a.** Parameterise `test/coverage.test.ts` by award. It currently hardcodes
`AWARD`, `CLAUSES` and `SUBCLAUSE_COUNT`. Move retail's inline `CLAUSES` and
`SUBCLAUSE_COUNT` into `src/awards/MA000004/enumeration.ts` — **preserving the
comments that state their provenance and why they are hardcoded**, since those
comments are the reasoning this whole command is built on — then have the test
read `enumeration.ts` per award and run its describes for each award directory
found under `src/awards/`. Keep every existing assertion. The MA000004 run must
stay identical.

**5b.** Parameterise `test/golden/published-rates.test.ts`. It hardcodes
`AWARD = "MA000004"` and a `shiftFor()` switch keyed on retail's condition names.
Move the condition-to-test-shift mapping into the award directory so each award
supplies its own, and have the suite walk every mapped award. This is the
answer-key suite, and the README's claim that it picks a new award up on its own
is not true until this is done.

**5c.** Write `scripts/closure.ts` and the `closure` npm script. It takes an award
code, runs all eight axes from `gap-axes.md` against the live database, prints the
table in that file's format, and exits non-zero when any axis is open. Axes 3, 4,
6 and 7 already exist as queries in `scripts/verify.ts` and `test/reachable.test.ts`
— reuse them, do not re-implement them. Axis 7 is service-global rather than
award-scoped (an engine's route either exists or it does not); run it as-is under
every award's report rather than trying to filter it per award.

**5c-bis.** Any test that assumes this service has exactly ONE award will now
fail, and those are not in the two suites above. `api.test.ts` asserted
`/v1/awards` returns a list of length 1; a second priceable award broke it,
correctly. Fix them by asserting the SET rather than the count, so the test does
not need editing again the next time an award is mapped.

**5d.** Run it. `npm run closure -- <CODE>`.

It will report nearly everything open, because nothing is modelled yet. **That
output is the work list.** Save it to the state file as the opening balance. From
here the number can only go down, and a step that does not move it is a step that
did not do anything.

Also run `npm run closure -- MA000004` and confirm it is closed. If the
parameterisation broke retail, fix that before writing a line of the new award.

## Step 6: Author, burning the list down

One file at a time, in `scripts/rules-load.sh` order (`service-conventions.md`
has the list). After each file:

```bash
npm run rules:load && npm run closure -- <CODE> && npm test
```

and record the new balance.

**`npm test` is in that chain deliberately.** An earlier loop ran only
`rules:load && closure`, and a mapping run pushed a commit with five failing
tests because closure was green and nobody re-read the suite count. Closure
measures the ledger; the suite measures everything the ledger does not, including
every existing test that quietly assumed this service had one award in it.

Rules for every row, without exception:

- cite the clause in `clauses`
- transcribe the award's own sentence into `clause_text`, from the fetched text,
  never from memory
- put the reasoning in `note`, including anything deliberately not modelled and
  which direction that errs in
- where a choice over-reports or under-reports, **choose to over-report a breach
  and over-report overtime**, and say so, because under-reporting overtime is
  under-paying
- `penalty_description` and allowance names copied verbatim from Step 3

Coverage files carry `source = 'award_text'` on every row somebody opened, a
substantive note on every row, and no inherited statuses. The dispositions file
goes last and stamps `Pending:` or `By design:` on every residual.

**Every `Pending:` says which existing channel will carry the fact it is waiting
on** — `employee-facts`, a `RequiredInput` group, or neither. One sentence in the
note. See `$AUTOFEATURE_HOME/awards/data-requirements.md`; it introduces no new
mechanism, because the service already has both channels and the product already
renders both.

`neither` is the answer that matters: it means the product holds no record the
fact could attach to, so the clause needs a product change and not just
modelling. MA000003 has two — cl 20.2(b)–(d) needs rostered start and finish
times `Shift` does not carry, and cl 29 needs a roster change history the service
is never given.

A `Pending:` with no such line is not a backlog item, and a backlog nobody can
total is what let one award ship 114 of them behind eight green zeroes.

A note is also not substantive if it is short. `verify.ts` check 9 asks whether
the stored `clause_text` appears in the award's text, which a seven-character
fragment satisfies trivially — `"below 0"`, `"further"`, `"in excess of"` all
passed. Transcribe a whole contiguous passage that carries the rule's meaning on
its own, not the few words that happen to be unique. Where the two awards
mapped so far disagree, the ratio is 343 characters average against 30.

`award-text.sql` then replaces the hand-typed `clause_text` with the award's own
words out of `fwc_clause`, which is the mechanised version of the check a human
would do by eye.

Fan out authoring across independent files where it is safe (allowances, leave,
termination and evidence do not interact), on **sonnet**. Keep `rules.sql` and the
coverage family in the orchestrator's own context, because they are where the
judgement from Step 4 gets spent.

**Agents that WRITE need `isolation: "worktree"`.** Four authoring agents were
once launched into one shared tree and each reported "another session is editing
this tree" — which was its three siblings. Nothing was lost that time, but that
was luck rather than design, and the cost of a worktree is a few hundred
milliseconds against a mapping measured in hours.

## Step 7: The scenario suite

The closure axes cannot catch a clause read, cited and transcribed correctly but
wired to the wrong rule — Step 7.5 goes after most of that. What Step 7.5 can't
see is a bug that only shows up when several rules interact over one real shift:
a boundary crossed, a priority resolving the way two individually-correct rules
did not anticipate together. That's what a hand-written scenario still earns its
place doing, and it's bounded by the clause count rather than open-ended.

Write `test/golden/<award-slug>-shifts.test.ts` in the shape of
`retail-shifts.test.ts`, with **a clause cited on every expectation**. Cover at
minimum: each named condition in its own right; a shift crossing a penalty
boundary; a shift crossing local midnight; a shift running outside the span; a
public holiday beating the day it falls on; a public holiday in a state where the
same instant is a different date; each employment type; the junior bands at their
edges; and every roster rule reporting `unevaluated` when given too little
context.

Add the second answer key if the Fair Work Ombudsman publishes a pay guide for
this award. `scripts/extract-pay-guide.ts` exists. Two independent keys that have
to agree with each other as well as with the engine is worth considerably more
than one.

## Step 7.5: Semantic verification

The eight closure axes prove a clause was read and a rule is reachable. None of
them prove a rule's `days_of_week`, `time_from`, `priority` or `kind` says what
the clause it cites actually says — that's a different question, answered
probabilistically rather than with a boolean, which is why it is its own step
and not a ninth axis.

Run `/autofeature:award-verify <CODE>` (`$AUTOFEATURE_HOME/awards/verify-workflow.md`
has the mechanism: two agents independently re-derive each predicate-bearing row
from its clause text alone, blind to what shipped and to each other, and a third
argues the shipped row is wrong only where they disagree).

Any `confidence: high` finding is a hard stop here, in checkpoint mode and in
automated mode alike: fix the row and re-run scoped to that clause, or record
the override and the reasoning for it in the state file — an override is a
judgement call and belongs beside the others from Step 4, not made silently.
`medium`/`low` findings and the cleared disagreements carry forward into the
review pack rather than blocking on their own.

## Step 8: Closure run and review pack — HARD CHECKPOINT

```bash
npm run rules:load && npm run verify && npm run closure -- <CODE> \
  && npm run closure -- MA000004 && npm test && npm run typecheck
```

Every axis zero, retail still accounted, the suite green and no fewer tests than
the Step 0 baseline. **Anything else is not done**, and the report says which
axis is open rather than describing the run as nearly finished.

### The already-shipped awards must price exactly what they priced at Step 0

`npm test` includes `test/snapshot.test.ts`, which re-prices the cross-product
for every mapped award and compares it against the fixtures committed at Step
0.4b. A failure there is not the new award's tests failing — it is the new award
having changed what an award **already in production** tells an employer.

That is the failure mode the rest of this pipeline cannot see. Every other check
here is scoped to the award being mapped: the axes enumerate its clauses, the
scenario suite prices its shifts, `award-verify` re-derives its predicates. None
of them look at the other award, and the other award's own tests only cover the
combinations somebody thought to write down. A mapping run touches shared
things — the engine, the schema, the closed vocabularies, the loaders — and the
combination that moves is, by construction, the one nobody asserted.

**Treat every changed line as a defect until shown otherwise.** For each one,
name the shared change that caused it and say whether the old answer or the new
one is right:

- **The old answer was wrong and the new award's work fixed it.** Legitimate.
  Regenerate the fixture, and say so in the commit and in the review pack — a
  silently corrected figure is a figure somebody was paid on.
- **A rate rise or a variation landed in the extract.** Legitimate, and it should
  have moved at Step 0.4b rather than here. Resolve why it did not.
- **Anything else.** A regression in a live award. Fix it before the checkpoint.

The one thing never permitted is regenerating the fixture to make the suite
quiet. The diff prints every changed combination with its before and after
precisely so it can be read rather than counted, and a snapshot regenerated
without reading it is worse than no snapshot, because it launders the change
into the baseline.

**`closure` now exits 1 while any `Pending:` remains, so the chain above stops
here on an unimplemented award. That is the gate working, not a failure of the
command.**

IMPLEMENTED is the outcome. ACCOUNTED is a step — it tells you the backlog you
are holding is complete rather than a floor, which is worth knowing and is not a
result. `gap-axes.md` has the reasoning.

Shipping an award that is accounted but not implemented is a legitimate decision
— retail ran in production for months at `Pending: 1`. It is a **decision**,
taken by a person, stated in one plain sentence at the top of the review pack:
what this award does, what it does not do yet, and how many clauses are waiting.
It is never the default and never silent.

### The count is the wrong criterion

`Pending: 1` shipped safely and `Pending: 114` obviously should not, but the
number is not what separates them. Two awards could sit at 50 apiece with one
safe and the other dangerous.

What separates them is **how each gap fails**:

- **It refuses visibly.** The rule returns `unevaluated`, or the engine raises
  rather than answering. The caller knows they did not get an answer. Safe at any
  count — this is the whole reason the service refuses instead of returning zero.
- **It errs toward over-reporting.** A breach flagged that the award permits,
  overtime counted generously. Annoying, arguable, and it never takes money off
  an employee.
- **It silently under-reports.** The rule simply does not fire, so the figure
  comes back complete-looking and low. **Nobody notices, and it is somebody's
  wages.**

**Any `Pending:` in the third category blocks the ship, at any count. Nothing in
the first two does.** That is the rule, and it is why one award shipped at 1 and
another must not at 114 — retail's single pending item is about which NES
entitlements to surface, and it cannot underpay anyone.

**Apply it per row, and describe the result per clause.** The row count is what
the gate reads; the clause count is what a person can act on. In the mapping that
produced this rule, 114 pending rows fell across 27 clauses — Schedule C alone
contributing ten of them for one supported-wage implementation — so "114" was
never a workload and saying it as though it were is the same overstatement as
"closed", pointed the other way.

Run it against the ledger. Exactly three of those 114 said they under-pay:

```
cl 10.9   time beyond the agreed hours is overtime. UNDER-reported without
          the agreed pattern, which under-pays
cl 20     cl 20.2's remaining limbs ...
cl 20.2   the limbs keyed to rostered start and finish ...
```

Three rows, in two clauses, sharing one cause — the agreed pattern and the
rostered times. That is a shippable scope, and it was invisible while the
conversation was about the number 114.

The sentence a decision can be made from is *"27 areas unbuilt, 2 of which can
silently under-pay, both waiting on the same product change"* — not a count.

### A `Pending:` that does not state its direction is a blocker

Step 6 requires every note to say which way its omission errs. Of the 114 above,
**3 stated it and 111 did not** — so the criterion could only be applied to 3% of
the backlog, and the rest had to be read one by one.

An unstated direction is not "probably fine". It is a gap nobody has classified,
and it blocks until somebody does. This is cheap to fix at authoring time and
expensive to reconstruct months later.

Then emit `.autofeature/awards/<CODE>-review.md`, for a person who knows the
award and not SQL:

- the closure table, all eight axes at zero
- **every rule row, with its clause reference and the award's own words beside
  it**, grouped by clause, so the whole reading can be reviewed by reading one
  column
- the Step 7.5 semantic-verification summary — rows checked, any table skipped
  for lack of a schema, every confirmed defect and how it was resolved, every
  cleared disagreement
- **both verdicts, in the first screenful** — accounted yes/no, implemented
  yes/no, and if not implemented the `Pending:` count and one plain sentence
  saying what the award does and does not do yet. A pack that opens with a
  green table and buries a 69% backlog in a tally is read as a stronger claim
  than the work supports
- the coverage tally, and every `Pending:` and `By design:` residual with its
  reason
- **the `Pending:` residuals grouped by what carries them, with counts** —
  how many need modelling alone (`employee-facts` or a `RequiredInput` group)
  against how many need a product change first (`neither`). The last number
  goes first: it is the only part of the backlog that cannot be burned down
  inside the compliance service. See `awards/data-requirements.md`
- the refusal ledger from Step 4, meaning what this award cannot say and why
- what is `derived` rather than `published`, with the clause that justifies it
- source provenance: award text URL, retrieval date, compilation date, checksums,
  and the workbook release
- the assumptions, in the `notes` style the service already uses, each stating
  which direction it errs in

**Stop and present the review pack to the user.** This is a hard checkpoint in
`mode:automated` too — closure and a passing suite are evidence a mapping is
mechanically sound, not a claim that it's fit to be relied on. Say so plainly in
the presentation: this raises confidence, it is not a compliance guarantee, and
a business relying on it should have a qualified person, an award-interpretation
lawyer or a Fair Work consultant, review the pack before it's treated as
authoritative. Do not proceed to Step 9 without the user's go-ahead.

## Step 9: Ship

Follow `$AUTOFEATURE_HOME/adapted/feature-ship.md`. The branch
`award/<code-lowercase>` already exists from preflight; commits in the repo's
voice (they explain what was found, not what was typed), PR body carrying the
closure table and the refusal ledger. Attach the review pack.

Do not merge. An award mapping nobody read is worth less than no mapping at all.

## After it ships

The method keeps learning after an award is mapped, so a mapping is never
permanently verified. Two commands re-open it deliberately:

- **`/autofeature:award-audit <CODE>`** — read-only, against the principles as
  they stand today rather than when the award was mapped. Run it when a principle
  changes, and on every previously-shipped award, not only the new one.
- **`/autofeature:award-drift <CODE>`** — when the Commission publishes new
  figures or a variation.

Neither edits anything. Both produce a decision for a person.

## Step 9.5: Promote — HARD CHECKPOINT

**Merging the PR does not put the award anywhere.** Code and data deploy
separately: `gcloud run deploy` ships the engine, which carries zero award
data by design; the rules just merged to main live only in the git repo until
someone loads them into a real database. And **there is one deployed
database, not a staging and a production one** — `docs/deploy.md` describes a
single Cloud SQL instance (`rosterio-award`) behind the one Cloud Run service
that `shiftos-api` calls for real payroll. The first time this award's rules
get loaded remotely, it is directly into what real customers price shifts
against. Treat this step with that weight — it is a hard checkpoint in
`mode:automated` too, and nobody proceeds past it without the user's explicit
go-ahead.

This is the minimum-viable version of this step: it makes the existing,
real promotion mechanism an explicit, gated part of the pipeline, rather than
an assumption left for whoever merges the PR to rediscover. It does not add a
staging tier — that's a real infra decision (a second Cloud SQL instance, a
second Cloud Run service) for the user to make separately, not something this
command provisions on its own.

```bash
# From docs/deploy.md. Requires gcloud auth and access to the `rosterio` project.
cloud-sql-proxy --token "$(gcloud auth print-access-token)" \
  --port 55433 rosterio:australia-southeast1:rosterio-award &

export DATABASE_URL="postgresql://award:$(gcloud secrets versions access latest \
  --secret=compliance-database-url --project=rosterio | sed -E 's#.*://award:([^@]*)@.*#\1#')@127.0.0.1:55433/award"
```

With `DATABASE_URL` pointing at the proxied connection, `scripts/lib/psql.sh`
and every `npm run` script route to it automatically — nothing else changes.

**Order is load-bearing: data first, image second.** A new column is ADDITIVE, so
the image already running tolerates it; the new image cannot run without it.
Reversed, the service raises `column ... does not exist` on every request between
the two steps.

1. **Apply the migrations, by hand, first.** Nothing applies them for you:
   `schema-load.sh` sees `instrument` and exits 0 with "Nothing to build", and
   `rules:load` only touches `rule_*` rows. Every capability approved at Step 4
   has a file in `db/migrations/` and this is where it lands.

   ```bash
   for f in db/migrations/*.sql; do
     echo "-- $f"
     podman run --rm -i --network host docker.io/library/postgres:16-alpine \
       psql -v ON_ERROR_STOP=1 "$DATABASE_URL" < "$f"
   done
   ```

   Skipping this is the near-miss that produced the rule: five schema changes,
   three with no migration, a green suite locally, and a production database that
   would have failed on the first priced shift.

2. `npm run db:remote` — schema (skips, correctly), migrations, workbooks, award
   text, NES text, then every award's rules.

   **Not `rules:load` alone.** `award-text.sql` replaces each rule's hand-typed
   `clause_text` with the award's own words out of `fwc_clause`, and `fwc_clause`
   is populated by `award:load`. A remote database that has never seen this award
   has no text for it, so `rules:load` on its own leaves every transcription
   unverified and `verify` reports a category it does not report locally.

   Each award's file deletes only its own instrument's rows (per
   `service-conventions.md`'s versioning doctrine), so this does not disturb any
   other award or any employer's `rate_override`/`declaration` data.
2. `npm run verify` — must be clean against the remote database, same as
   local. A pass locally is not evidence of a pass remotely; the workbooks or
   award text loaded there may not be what was tested against.
3. `/autofeature:award-verify <CODE>` — re-run against the remote database.
   Not optional and not assumed-same-as-local: this is the same "don't trust,
   check" principle applied to the promotion boundary itself.
4. `npm run closure -- MA000004` (and any other previously-mapped award) —
   confirm nothing else regressed. The remote database may be on a different
   workbook release than local.

5. **Deploy the image**, last, per `docs/deploy.md`. Any capability from Step 4
   is engine code, and until this runs the deployed service is the previous one —
   reading new rows with old logic, which is exactly the direction that is safe
   and exactly why the order is this way round.

6. **Smoke-test the deployed service against the new award**, with payloads taken
   from `test/api.test.ts` rather than written from memory. Routes are `/v1/...`
   and the bodies are flat; a smoke test that 404s tells you nothing about the
   deploy. Price one shift on the new award and one on the award already in
   production — the second is the regression check that matters.

**Stop and present the remote verify + closure + award-verify results to the
user before this is called done.** Only after they confirm does this award
count as live. Record the promotion — who, when, which git commit — in
`.autofeature/awards/<CODE>-map.md`.

---

## Prohibitions

- No rate, penalty percentage or dollar figure written by hand anywhere.
- No `rule_roster.kind`, `trigger`, `unit` or `accrual_method` outside the
  existing `CHECK` lists without a deliberate, approved schema and engine change.
- No `status = 'modelled'` with `source <> 'award_text'`.
- No `partial` without its sub-clauses itemised.
- No residual without a `Pending:` or `By design:` disposition.
- No editing MA000004's rules to make a shared check pass. If a shared check
  needs to change, it changes for a reason that is written down.
- No reporting completion on any basis other than a green closure run.
- No shipping past a `confidence: high` semantic-verification defect without
  either fixing the row or writing the override and its reasoning into the
  state file.
- No presenting a green closure + verification run to the user as a compliance
  guarantee. It is evidence a mapping is mechanically sound. Say what it is.
- No considering a mapping "done" because the PR merged. It is done when
  Step 9.5 has run against the actual deployed database and the user has
  confirmed it — there is no staging tier today, so this is the only
  rehearsal an award gets before it prices a real shift.

## Resume

`git checkout award/<code-lowercase>` first — resume picks up work, not a fresh
branch. Then read `.autofeature/awards/<CODE>-map.md`, find the last completed
step, re-run `npm run closure -- <CODE>` to re-establish the balance, and
continue. The closure run is idempotent and is the source of truth about where
the mapping actually is, which is the point of having it.
