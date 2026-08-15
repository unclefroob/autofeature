---
name: award-map
description: |
  Map an Australian modern award into rosterio-compliance-service so it can be priced and roster-checked, with every rule citing a clause and mechanically checkable against the award's own words.
  Enumeration-first, deliberately: transcribe the award's clause list and per-clause sub-clause counts from the Commission's consolidated PDF, triage every clause against the closed rule vocabulary, stand up the eight closure axes to produce a FIXED work list, and only then author SQL against a list that can only shrink.
  Exists to kill the perpetual-gap loop. "Are there gaps?" becomes one command that returns a list, instead of a question an agent answers differently every time it is asked.
  Closure proves coverage, not correctness — Step 7.5 runs /autofeature:award-verify to independently re-derive every predicate-bearing row from its clause text and adversarially check it against what shipped, closing the one gap closure can't: a rule that's reachable and cites the right clause but got the day, the time or the priority wrong.
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

## $AUTOFEATURE_HOME

```bash
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/awards" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

**Read all four of these in full before Step 1, and treat them as
authoritative:**

- `$AUTOFEATURE_HOME/awards/service-conventions.md` — the service being extended
- `$AUTOFEATURE_HOME/awards/gap-axes.md` — the eight axes and the definition of done
- `$AUTOFEATURE_HOME/awards/rule-tables.md` — what the vocabulary can and cannot say
- `$AUTOFEATURE_HOME/awards/verify-workflow.md` — semantic verification, run at Step 7.5

## Args

- `<AWARD_CODE>` — required, e.g. `MA000003`. Refuse to proceed without one.
- `mode:checkpoint` (default) — stop at every checkpoint below.
  `mode:automated` — run through, but the three **hard** checkpoints still stop:
  the transcription (Step 2), the expressiveness triage (Step 4), and the
  human sign-off on the closure + semantic-verification report (Step 8). They
  are where judgement is exercised or where the mapping's whole claim to be
  trustworthy is either earned or not, and none of the three may be skipped.
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
8 closure run + review pack [HARD CHECKPOINT] ─ 9 ship
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

**5d.** Run it. `npm run closure -- <CODE>`.

It will report nearly everything open, because nothing is modelled yet. **That
output is the work list.** Save it to the state file as the opening balance. From
here the number can only go down, and a step that does not move it is a step that
did not do anything.

Also run `npm run closure -- MA000004` and confirm it is closed. If the
parameterisation broke retail, fix that before writing a line of the new award.

## Step 6: Author, burning the list down

One file at a time, in `scripts/rules-load.sh` order (`service-conventions.md`
has the list). After each file: `npm run rules:load && npm run closure -- <CODE>`,
and record the new balance.

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

`award-text.sql` then replaces the hand-typed `clause_text` with the award's own
words out of `fwc_clause`, which is the mechanised version of the check a human
would do by eye.

Fan out authoring across independent files where it is safe (allowances, leave,
termination and evidence do not interact), on **sonnet**. Keep `rules.sql` and the
coverage family in the orchestrator's own context, because they are where the
judgement from Step 4 gets spent.

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

Every axis zero, retail still closed, the suite green and no fewer tests than the
Step 0 baseline. **Anything else is not done**, and the report says which axis is
open rather than describing the run as nearly finished.

Then emit `.autofeature/awards/<CODE>-review.md`, for a person who knows the
award and not SQL:

- the closure table, all eight axes at zero
- **every rule row, with its clause reference and the award's own words beside
  it**, grouped by clause, so the whole reading can be reviewed by reading one
  column
- the Step 7.5 semantic-verification summary — rows checked, any table skipped
  for lack of a schema, every confirmed defect and how it was resolved, every
  cleared disagreement
- the coverage tally, and every `Pending:` and `By design:` residual with its
  reason
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

## Resume

`git checkout award/<code-lowercase>` first — resume picks up work, not a fresh
branch. Then read `.autofeature/awards/<CODE>-map.md`, find the last completed
step, re-run `npm run closure -- <CODE>` to re-establish the balance, and
continue. The closure run is idempotent and is the source of truth about where
the mapping actually is, which is the point of having it.
