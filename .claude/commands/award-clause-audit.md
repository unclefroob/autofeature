---
name: award-clause-audit
description: |
  Coverage verification for an award already mapped into rosterio-compliance-service, run INWARD: starts at the award as the Commission published it and asks, of every clause, what points at this.
  The direction nothing else has. `verify.ts` checks our rows reference real rates; `award-verify` checks our rows say what their clauses say; the test suite checks the engine behaves the way our rows say; the coverage ledger checks every clause has a row — and the row's status was self-asserted. All four run outward, from what we wrote toward the award, so a clause with NOTHING pointing at it is not a finding in any of them, it is not an input. Sixteen clauses of MA000004 had no implementation at all while all four reported clean.
  Enumerates every clause of the award, hands each auditor the verbatim clause text plus the ledger row, the rule rows and the test citations for its sections, and requires exactly one verdict per clause — tested / modelled_untested / covered_not_applicable / gap / mismodelled — then merges the verdicts into a deduped, prioritized work list.
  Standalone and re-runnable. Run it after `award-close` finishes an award, after a variation adds clauses, or whenever "are there gaps?" needs an answer that is a list rather than an opinion.
  Invoke as:
    /autofeature:award-clause-audit <AWARD_CODE>
    /autofeature:award-clause-audit <AWARD_CODE> sections: 15,16,17
    /autofeature:award-clause-audit <AWARD_CODE> group-size: 25
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
  - Workflow
---

# AutoFeature — Award Clause Audit

Asks what is **missing**, not whether what is there is right.

`/autofeature:award-verify` starts from a rule row and asks whether it matches its clause. This starts
from a clause and asks whether anything implements it. Both questions are worth asking and neither
answers the other: a row can be perfectly derived from a clause that the award never had, and a clause
can impose a real obligation that no row, no test and no ledger entry has ever mentioned. `award-verify`
cannot see the second case, and not because it is written carelessly — its own SQL only ever SELECTs
rows that exist, so a clause with no row is not a finding, it is not an input. A clean run of one is not
a clean run of both.

Read `$AUTOFEATURE_HOME/awards/clause-audit-workflow.md` in full before Step 5 — it has the Workflow
script verbatim and the reasoning for the two judgement rules that make the verdicts trustworthy. Do not
re-derive the mechanism from scratch; invoke that script exactly as written.

## The direction that had no command

Four verification layers reported clean while sixteen clauses of MA000004 had no implementation at all.
Not one of them was broken. Every one of them ran outward, from something we wrote toward the award:

- `verify.ts` — our rule rows reference real published rates.
- `/autofeature:award-verify` — our rule rows say what their clauses say.
- the test suite — the engine behaves the way our rows say.
- the coverage ledger — every clause has a **row**, and the row's `status` was written by a generator
  whose boilerplate note read *"Cited by this service's own rule data, so the engine acts on it"* — a
  claim about rule data made without reading rule data. For five clauses it was false.

A clause with nothing pointing at it is invisible to all four, and running them more often produces
nothing, because they share an enumeration: the rows we happen to have. The audit that found the sixteen
gaps worked by enumerating the **award** instead and asking, clause by clause, what points at this. That
is this command.

The award's own enumeration is `fwc_clause` — Layer A, mirrored from the Commission and never edited.
Not the coverage ledger, which is a transcription and is smaller: on MA000004 the award has 332 clauses
and the ledger has 221 rows, so 111 clauses are outside the ledger's own completeness check, which is a
check over the ledger's rows.

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

## Args

- `<AWARD_CODE>` — required, e.g. `MA000004`.
- `sections:` — optional comma list of top-level clause numbers or schedule letters (`15,16,B`),
  restricting the audit to those sections. Use it to re-audit what a variation touched rather than
  re-running the whole award. A scoped run must say in its report that the rest of the award was **not**
  looked at this time, or it reads as an audit of the award.
- `group-size:` — optional, the target number of clauses per auditor. Default 40. Lower it if a section
  group's file is large enough that an auditor starts skimming; the cost is linear in the number of
  groups and the failure it prevents is a `tested` verdict nobody checked.

## Step 1: Resolve the service and connect

```bash
CWD=$(pwd); PARENT=$(dirname "$CWD")
for c in "$CWD" "$PARENT/rosterio-compliance-service" "$CWD/rosterio-compliance-service"; do
  [ -f "$c/db/schema.sql" ] && [ -d "$c/src/awards" ] && { SVC="$c"; break; }
done
[ -z "$SVC" ] && { echo "Run from rosterio-compliance-service or its parent."; exit 1; }
cd "$SVC"
source scripts/lib/psql.sh
mkdir -p /tmp/award-clause-audit
```

**There is deliberately no `psql` binary assumed on this machine** — `scripts/lib/psql.sh` is the
service's own answer to that, routing every query through a throwaway container (or the host network, if
`DATABASE_URL` points at a deployed database) rather than requiring a system install. Use its
`psql_query "<sql>"` for a read and `apply_sql "<path>"` for a file — never a bare `psql` invocation, it
will not exist to run.

Confirm the award's **text** is loaded, which is the enumeration this whole command runs on:

```bash
psql_query "SELECT count(DISTINCT clause) FROM fwc_clause WHERE award_code = '<CODE>'"
```

Zero means `npm run award:fetch` and `npm run award:load` have not run for this award. There is no
audit to do — not "no gaps", *no enumeration*, which is the same distinction the whole command is about.
Say so and stop.

## Step 2: Enumerate the award, and reconcile the ledger against it

Two counts, before anything is spent. They are deterministic, they take a second, and on a real award
they are already findings:

```bash
psql_query "
SELECT (SELECT count(DISTINCT clause) FROM fwc_clause WHERE award_code = '<CODE>')          AS clauses_in_award,
       (SELECT count(*) FROM rule_coverage WHERE instrument_id = '<CODE>')                  AS ledger_rows,
       (SELECT count(*) FROM (SELECT DISTINCT f.clause FROM fwc_clause f
          WHERE f.award_code = '<CODE>'
            AND NOT EXISTS (SELECT 1 FROM rule_coverage v
                             WHERE v.instrument_id = '<CODE>' AND v.clause = f.clause)) t)  AS clauses_with_no_ledger_row,
       (SELECT count(*) FROM rule_coverage v WHERE v.instrument_id = '<CODE>'
          AND NOT EXISTS (SELECT 1 FROM fwc_clause f
                           WHERE f.award_code = '<CODE>' AND f.clause = v.clause))          AS ledger_rows_with_no_clause
"
```

`clauses_with_no_ledger_row` is the ledger's blind spot stated as a number — clauses of the published
award its completeness check cannot report on, because that check counts its own rows. On MA000004 it is
111 of 332. `ledger_rows_with_no_clause` is the same error pointed the other way: a disposition about a
clause the award does not have, which usually means a renumbering landed and nobody noticed. Report both
in Step 6 whatever the auditors find.

Then pull the whole enumeration, one record per clause, with everything that points at it:

```bash
TABLES=$(psql_query "
  SELECT table_name FROM information_schema.columns
   WHERE table_schema = 'public' AND table_name LIKE 'rule\\_%'
     AND column_name = 'clauses' AND table_name <> 'rule_coverage'
   ORDER BY 1")

UNION=$(for t in $TABLES; do
  printf "    SELECT '%s' AS tbl, clauses, to_jsonb(x.*) - 'clause_text' AS row FROM %s x WHERE instrument_id = '<CODE>'\n    UNION ALL\n" "$t" "$t"
done | sed '$ d')

psql_query "
WITH latest AS (
  SELECT DISTINCT ON (clause) clause, heading, text
    FROM fwc_clause WHERE award_code = '<CODE>'
   ORDER BY clause, operative_from DESC
),
cited AS (
$UNION
),
pointed AS (
  SELECT regexp_replace(clauses, '\\(.*\$', '') AS clause,
         jsonb_agg(jsonb_build_object('tbl', tbl, 'citedAs', clauses, 'row', row)) AS rows
    FROM cited WHERE clauses IS NOT NULL AND clauses <> '' GROUP BY 1
)
SELECT jsonb_agg(c ORDER BY c.clause) FROM (
  SELECT l.clause, l.heading, l.text,
         CASE WHEN v.clause IS NULL THEN NULL
              ELSE jsonb_build_object('status', v.status, 'note', v.note, 'source', v.source) END AS coverage,
         coalesce(p.rows, '[]'::jsonb) AS rules
    FROM latest l
    LEFT JOIN rule_coverage v ON v.instrument_id = '<CODE>' AND v.clause = l.clause
    LEFT JOIN pointed p       ON p.clause = l.clause
) c" > /tmp/award-clause-audit/clauses.json
```

**The rule tables are discovered, not listed.** `award-verify` hardcodes its nine, and that list going
stale is exactly how `rule_break_entitlement` came to be read by nothing and reported as skipped by
nothing — a table in no list is not skipped, it is invisible. A command whose entire purpose is finding
what nothing points at cannot afford its own version of that hole, so it asks the database which tables
carry a `clauses` column and unions whatever comes back. A rule table created tomorrow is in this audit
tomorrow.

Two details that matter and are not obvious:

- **The clause a row cites is stripped to its clause.** A row citing `15.7(b)` is coverage of clause
  `15.7`, which is the granularity `fwc_clause` numbers at, hence the `regexp_replace`. Without it every
  paragraph-level citation matches nothing and the audit reports the award unimplemented.
- **`clause_text` is dropped from every row.** The auditor is already holding the award's own words from
  `fwc_clause`, which is better evidence than a transcription of them, and carrying both doubles the
  file for nothing.

Read the file and confirm it parsed and that `jq length` equals `clauses_in_award`. It should, by
construction; check anyway, because that equality is the claim this command makes.

## Step 3: Index what the tests cite

```bash
jq -r '.[].clause' /tmp/award-clause-audit/clauses.json > /tmp/award-clause-audit/clause-list.txt

: > /tmp/award-clause-audit/cites.tsv
while read -r c; do
  e=$(printf '%s' "$c" | sed 's/[.[\*^$()+?{}|]/\\&/g')
  grep -rEn "cl(ause)? ${e}([^0-9A-Za-z.]|\$)" test/ src/ --include=*.ts 2>/dev/null \
    | sed "s|^|$c\t|" >> /tmp/award-clause-audit/cites.tsv
done < /tmp/award-clause-audit/clause-list.txt

grep -rEn '^\s*(describe|it)\(' test/ --include=*.test.ts \
  | sed -E 's/^([^:]+):([0-9]+): *(describe|it)\(\s*"?/\1\t\2\t\3\t/; s/",?\s*(async)?.*$//' \
  > /tmp/award-clause-audit/suites.tsv
```

`cl 15.7` and `clause 15.7` are the citation conventions this repo actually uses. The trailing
`([^0-9A-Za-z.]|$)` is load-bearing: without it `cl 15.1` matches `cl 15.10` and the index reports a
clause tested by a test about its neighbour.

**This index is a hint and the workflow tells every auditor so.** The first line it returns for
MA000004's cl 10 is a sentence inside a block comment in `award-text.test.ts` explaining that cl 10 is a
heading. A clause named in a comment is not a clause with a test, and a run that counted grep hits would
report the award far better covered than it is — which is the failure mode this command exists to stop,
reproduced inside the command. `suites.tsv` is what lets an auditor check: it holds every test file's
`describe`/`it` strings, so a citation can be resolved to an assertion or shown not to have one.

## Step 4: Partition, and write one file per group

Group by top-level section — clause `15.7` and clause `15` belong to one auditor, because judging a
sub-clause without its siblings is how a container heading gets read as an obligation. Pack sections
into groups up to `group-size:` clauses.

```bash
jq -R 'split("\t") | {clause: .[0], hit: (.[1:] | join(":"))}' /tmp/award-clause-audit/cites.tsv \
  | jq -s . > /tmp/award-clause-audit/cites.json

jq --argjson size 40 --slurpfile cites /tmp/award-clause-audit/cites.json '
  ($cites[0] | group_by(.clause) | map({key: .[0].clause, value: map(.hit)}) | from_entries) as $idx
  | map(. + {tests: ($idx[.clause] // []), section: (.clause | capture("^(?<s>[0-9]+|[A-Z])").s)})
  | group_by(.section)
  | reduce .[] as $sec ([]; if (.[-1] and ((.[-1].clauses | length) + ($sec | length)) <= $size)
        then .[0:-1] + [{sections: (.[-1].sections + [$sec[0].section]), clauses: (.[-1].clauses + $sec)}]
        else . + [{sections: [$sec[0].section], clauses: $sec}] end)
' /tmp/award-clause-audit/clauses.json > /tmp/award-clause-audit/groups.json

i=0
while read -r g; do
  i=$((i + 1))
  printf '%s' "$g" | jq --arg a '<CODE>' '{award: $a, sections: .sections, clauses: .clauses}' \
    > "/tmp/award-clause-audit/g$i.json"
done < <(jq -c '.[]' /tmp/award-clause-audit/groups.json)
```

`< <(...)` rather than a pipe, because a `while` loop on the right of a pipe runs in a subshell and `i`
comes back out as 0 — every group written to `g0.json`, each overwriting the last, and eight ninths of
the award silently absent from the audit. The whole command is about absences that nothing reports; it
should not manufacture one in its own plumbing.

A section larger than `group-size:` on its own is not split — Schedule A's 42 clauses go to one auditor
at the default 40. That is deliberate: a schedule's classification definitions are judged against each
other or not at all.

Then build the `groups` argument the workflow takes — sections, the clause **ids** (small, and the
script checks the returned verdicts against them), and the file path:

```bash
jq -c 'to_entries | map({sections: .value.sections,
                         clauses: [.value.clauses[].clause],
                         file: ("/tmp/award-clause-audit/g" + ((.key + 1) | tostring) + ".json")})' \
  /tmp/award-clause-audit/groups.json
```

Confirm the clause ids across all groups, concatenated, equal `clause-list.txt` exactly. A partition
that drops a clause produces an audit that never knew the clause existed, and that is precisely the
state the sixteen gaps were in.

If `sections:` was given, filter `clauses.json` to those sections before this step and carry the fact
into the report.

## Step 5: Run the Workflow

Pass the script **inline** — the Workflow tool persists it to a file on its own and returns the path,
which is what you would use to resume, not what you pass on a first run.

```
Workflow({
  script: "<the Workflow script from awards/clause-audit-workflow.md, verbatim>",
  args: {
    award:   "<CODE>",
    service: "<$SVC, absolute>",
    suites:  "/tmp/award-clause-audit/suites.tsv",
    groups:  <the array from the last command in Step 4>
  }
})
```

Unlike `award-verify`, the clause text is **not** passed inline. A whole award's verbatim text is
hundreds of kilobytes and the auditors need to read files anyway to check citations, so they are handed
paths and read them.

Cost is **one auditor per group plus one synthesizer** — nine agents for a 332-clause award at the
default group size. That is small next to `award-verify`, which spends two readers per distinct clause
text and a skeptic on every disagreement. It is not small in wall time: each auditor reads dozens of
clauses in full and opens test files to confirm what the citation index only suggests, and that reading
is the work. Say the group count and the agent count before firing it. A `sections:`-scoped re-audit is
one or two agents and can be run without asking.

## Step 6: Report

Present, in this order:

1. **Unverdicted clauses first, before any count.** `unverdicted` and `lostGroups` from the workflow. A
   run that verdicted 290 of 332 clauses did not audit the award — it audited 290 clauses and does not
   know about 42, and each of those is in the same state the sixteen gaps were in: nothing pointing at
   it and nothing saying so. Re-run those sections scoped before quoting a total. Leading with the
   number that looks like success is how the previous four layers were read.
2. **Gaps** (`severity: high` in `defects`) — the clause, the obligation its text imposes, and what the
   service does instead of implementing it. Ordered by what **underpays**: a missing penalty reaches a
   payslip and nobody notices; a missing check refuses a lawful roster loudly and gets fixed the same
   day. These are the output.
3. **Mismodelled** — what is modelled contradicts the clause, both sides quoted. Cross-check any of
   these against `/autofeature:award-verify` scoped to that clause before acting: this command did not
   re-derive the predicate and is not the better instrument for deciding a row is wrong.
4. **Modelled but untested**, and the `test_plan` — the deduped work list. Say plainly that it is a set
   of proposals: an agent wrote each entry from a clause and a file listing, so the file it names may
   not be the right home and the assertion may need the award read again before it can be written.
5. **The two ledger reconciliation numbers from Step 2** — clauses with no ledger row, ledger rows with
   no clause. Report them even on a clean audit; they are facts about the ledger, not opinions of an
   agent, and they are the cheapest evidence in the run.
6. **Coverage of this run** — clauses audited, of how many in the award, and the counts whole:
   `143 tested / 136 not-applicable / 37 modelled-untested / 16 gaps / 0 mismodelled`. Quote all five.
   "16 gaps" alone is read either as a catastrophe or as a rounding error depending on the reader's
   mood; the other four numbers are what make it a measurement. If this was a `sections:`-scoped run,
   say plainly that the rest of the award was **not** looked at.

Append the findings to `.autofeature/awards/<CODE>-review.md` under a `## Clause audit — <date>` heading
rather than overwriting what is already there.

## Step 7: Record the run — which there is currently nowhere to do

`award-verify` ends by calling `npm run verified -- <CODE>`, and that step exists because a verification
whose result lived only in a markdown file was invisible to `npm run closure`, which printed
`IMPLEMENTED` identically for a verified award and a never-verified one. The same argument applies here
with the same force, and **this command must not reuse that record**:

- `verification_run` records rows checked, defects found and defects real over `PREDICATE_TABLES`. This
  command checks no predicate and reads no `PREDICATE_TABLES` row for correctness. Writing 332 clauses
  into `--rows` would make `closure` print `correctness VERIFIED — 332 rows` for an award whose
  predicates nobody re-derived. That is the same lie the record was built to stop, wearing the record's
  own uniform.
- The staleness key is wrong in both directions. The predicate fingerprint hashes rule rows, so a
  variation that **adds a clause** to `fwc_clause` — the thing that most obviously invalidates a clause
  audit — does not move it at all, and an unrelated edit to a rule row moves it when the audit is still
  perfectly good.

So: do **not** call `npm run verified` from this command. Report the run, write it to the review file,
and say in the report, in one sentence, that this audit is recorded nowhere the tooling can see. That
sentence is the finding, and it should be uncomfortable to write, because it is the identical shape of
failure `verified.ts` was written to end.

What closing it properly would take, concretely, so this is a specification rather than a complaint:

- a `clause_audit_run` table alongside `verification_run` — `instrument_id`, `audited_at`,
  `award_fingerprint`, `git_commit`, `sections_audited`, `clauses_in_award`, `clauses_verdicted`,
  `gaps`, `mismodelled`, `report_path`, `note`;
- `award_fingerprint` hashed over `(fwc_clause.clause, fwc_clause.text)` for the award **plus**
  `(rule_coverage.clause, rule_coverage.status)` — the enumeration and the dispositions, which are what
  this command reads. Deliberately not `rule_coverage.note`, for the reason `predicateFingerprint`
  excludes `clause_text`: lengthening a note is what an audit asks for, and hashing it would mark an
  award stale for having acted on the report;
- `clauses_verdicted < clauses_in_award` recorded rather than rounded, so a partial run can never read
  as a whole one;
- a `scripts/clause-audit.ts` and an `"audited"` npm script with `verified.ts`'s read/write split — no
  arguments reads the state and must never write, arguments write it — and a third `closure` verdict
  line beside `coverage` and `correctness`.

Until that exists, say so. Do not invent a command to call.

## What this is not

It is an enumeration and a reading, not a proof. Auditors can misread a clause the way a person can, and
the verdict most likely to be wrong is the one that ends the enquiry: `covered_not_applicable`. The
workflow makes an auditor say what in the clause text makes it unactionable, and makes the synthesizer
flag any such verdict on a clause about money or rostering, precisely because the ledger's own word for
it was wrong five times.

It also does not check that anything is **correct**. A `tested` verdict means a test exists and asserts
something, not that the assertion is right; a `modelled` clause with a transposed day is `tested` here
and a defect in `/autofeature:award-verify`. The two run in opposite directions over the same award and
neither subsumes the other — this one asks *what is missing*, that one asks *is this row right* — so a
clean run of one is not a clean run of both, and `/autofeature:award-scenarios` still asks the third
question neither can, which is whether a real week's pay comes out right.
