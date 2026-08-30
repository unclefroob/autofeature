---
status: CUSTOM
description: The Workflow script for the clause audit — one verdict per clause of the award, derived from the clause's own words rather than from what the service says about it, then merged into a deduped work list.
---

# Clause audit — the Workflow script

## What this closes, and what it doesn't

Every other check this service has runs **outward**: from something we wrote, toward the award.
`verify.ts` checks our rule rows reference real published rates. `award-verify` checks our rule rows say
what their clauses say — but its own SQL only ever SELECTs rows that exist, so a clause with no row is
not a finding, it is not an input. The test suite checks the engine behaves the way our rows say. The
coverage ledger checks every clause has a **row**, and the row's `status` was self-asserted: written by a
generator whose boilerplate note read *"Cited by this service's own rule data, so the engine acts on
it"* — a claim ABOUT rule data, made without reading rule data. For five clauses it was simply false.

A clause with nothing pointing at it is invisible to all four. Sixteen clauses of MA000004 had no
implementation at all while four verification layers reported clean.

This runs **inward**. It starts at `fwc_clause` — the award as the Commission published it, the layer
this service mirrors and never edits — and asks, for every clause in it, *what points at this?* Nothing
else in the family asks that question, and no amount of running the others more often produces it,
because they are all downstream of the same enumeration: the rows we happen to have.

It does not replace them. It cannot tell you a row is wrong — it never re-derives a predicate — and a
`tested` verdict here means a test exists and asserts something, not that the assertion is right. The
two directions are complements and a clean run of either is not a clean run of both.

## A `not_applicable` verdict must cite the clause, never the ledger

`covered_not_applicable` is the verdict that hides gaps, because it is the one that ends the enquiry. It
is also the verdict the ledger hands out for free: 136 of MA000004's clauses carry `not_applicable`, and
the auditor is shown that status as context.

So the discipline is: **the ledger's own word is not evidence for agreeing with the ledger.** That is
circular in exactly the way the boilerplate note above was circular, and it is the specific mechanism
that produced five false `modelled` rows. An auditor may only return `covered_not_applicable` if it read
the clause text and can say, in its own words, what makes the clause unactionable — *this is the
definitions clause*, *this is the dispute-resolution procedure*, *this is a heading with no prose of its
own*. "The ledger says not_applicable" is not that sentence, and a verdict whose evidence is the ledger
is worth exactly as much as the generator that wrote it.

The inverse holds and is cheaper to check: a clause the ledger calls `not_applicable` whose heading
names money or rostering — rates, penalties, overtime, breaks, leave, rosters — is where a wrong
disposition would hurt most. The synthesis pass flags those rather than accepting them.

## A test citation is a hint, not a test

Each auditor is handed an index of which test files mention its clauses, built by grepping for `cl 15.7`
and its variants. That index is a **starting point for reading, not evidence**. The very first line it
returns for MA000004's cl 10 is a sentence inside a block comment in `award-text.test.ts` explaining
that cl 10 is a heading. Counting that as coverage would report a clause tested on the strength of prose
about it.

So an auditor claiming `tested` must open the file and name the `describe`/`it` that asserts the
behaviour — engine output, API response, or a property of the loaded data. A row existing is not an
assertion. This is the same rule `verify-workflow.md` holds for behavioural claims, pointed at coverage
instead of at mechanism: **an agent that did not open the file cannot say what the file does.**

## Every clause gets a verdict, and the script checks that it did

The reference implementation asked each auditor for "exactly N entries, no silent skips" in the prompt
and then trusted the answer. Asking politely and not checking is the failure this whole command exists
to correct, so the script does the arithmetic itself: it is handed the clause ids each group covers, and
it diffs them against the ids that came back. What is missing is returned as `unverdicted`, named clause
by clause.

The same applies to a whole auditor dying. `parallel()` resolves a failed thunk to `null`, and filtering
the nulls away — which the reference did — silently drops forty clauses and leaves a run that looks like
it covered the award. An unverdicted clause is not a `covered_not_applicable`; it is the exact state
this command was built to make visible, and it must survive into the output.

## What each auditor is handed

The clause text for a whole award is far too large to pass through `args` — MA000004's is hundreds of
kilobytes — so unlike `award-verify`, which passes its rows inline, this passes **file paths**. The
command writes one JSON file per group and the auditor reads it. Each file holds, for that group's
sections only:

```
{ award, sections: ["15","16","17"],
  clauses: [ { clause, heading, text,          // verbatim, from fwc_clause
               coverage: { status, note, source } | null,   // the ledger row, or null if there is none
               rules: [ { tbl, citedAs, row } ],            // every rule_* row citing this clause
               tests:  [ "test/golden/retail-shifts.test.ts:88: ..." ] } ] }
```

`coverage: null` — a clause of the published award with no ledger row at all — is not an edge case worth
a footnote. On MA000004 it is 111 of 332 clauses, because the ledger was written against a transcribed
clause list and the award has more clauses than the transcription did. Those clauses are invisible to
the ledger's own completeness check, which is a check over its own rows.

## Running it

Invoked from `.claude/commands/award-clause-audit.md`, which does the enumeration, the grepping and the
partitioning — **Workflow scripts have no database access and no shell**, so all of that happens in the
command's own context.

```
Workflow({
  script: <the script below, verbatim>,
  args: {
    award:   "<CODE>",
    service: "<absolute path to rosterio-compliance-service>",
    suites:  "/tmp/award-clause-audit/suites.tsv",
    groups:  [ { sections: ["15","16"], clauses: ["15","15.1",...], file: "/tmp/award-clause-audit/g3.json" } ]
  }
})
```

Cost is **one auditor per group plus one synthesizer** — for a 332-clause award partitioned at roughly
forty clauses a group, nine agents. That is cheap next to `award-verify`, and it is not cheap in wall
time: each auditor reads dozens of clauses and opens test files to check citations, which is the work
and must not be tuned away. A cheaper model that takes the citation index at its word returns a longer
`tested` list and a worse audit.

### The Workflow script

```js
export const meta = {
  name: 'award-clause-audit',
  description: 'One verdict per clause of the award, derived inward from the clause text; merged into a deduped, prioritized work list',
  phases: [
    { title: 'Audit',      detail: 'one auditor per group of award sections, every clause verdicted' },
    { title: 'Synthesize', detail: 'merge verdicts, dedupe the proposed work, prioritize by what underpays' },
  ],
}

// Defensive, and for the same reason verify-workflow.md is: some runs of this tool deliver `args`
// as the raw JSON text rather than a parsed object, even when the caller passed a real object.
// Parsing here is what keeps a genuinely complete audit from silently auditing zero clauses.
const ARGS   = typeof args === 'string' ? JSON.parse(args) : args
const AWARD  = ARGS.award
const SVC    = ARGS.service
const SUITES = ARGS.suites
const GROUPS = ARGS.groups || []

const VERDICTS = ['tested', 'modelled_untested', 'covered_not_applicable', 'gap', 'mismodelled']

const CLAUSE_SCHEMA = {
  type: 'object',
  required: ['clauses'],
  properties: {
    clauses: {
      type: 'array',
      items: {
        type: 'object',
        required: ['clause', 'category', 'evidence'],
        properties: {
          clause: { type: 'string' },
          title: { type: 'string' },
          category: { type: 'string', enum: VERDICTS },
          evidence: {
            type: 'string',
            description: 'file:line, or the test file + the exact describe/it, proving the verdict. ' +
              'For covered_not_applicable: what IN THE CLAUSE TEXT makes it unactionable, in your own words.',
          },
          proposed_test: {
            type: ['string', 'null'],
            description: 'for modelled_untested/gap: one concrete test — which file, what input, what assertion',
          },
        },
      },
    },
  },
}

const SYNTH_SCHEMA = {
  type: 'object',
  required: ['counts', 'defects', 'test_plan'],
  properties: {
    counts: { type: 'object', additionalProperties: { type: 'number' } },
    defects: {
      type: 'array',
      items: {
        type: 'object', required: ['clause', 'summary'],
        properties: {
          clause: { type: 'string' },
          summary: { type: 'string' },
          severity: { type: 'string', enum: ['high', 'medium', 'low'] },
        },
      },
    },
    test_plan: {
      type: 'array',
      items: {
        type: 'object', required: ['file', 'clauses', 'description'],
        properties: {
          file: { type: 'string' },
          clauses: { type: 'array', items: { type: 'string' } },
          description: { type: 'string' },
          priority: { type: 'string', enum: ['high', 'medium', 'low'] },
        },
      },
    },
    summary: { type: 'string' },
  },
}

const auditorPrompt = (g) => `You are auditing the ${AWARD} modern award against its implementation in ` +
  `rosterio-compliance-service, for CLAUSE-LEVEL COVERAGE. Your sections: ${g.sections.join(', ')} ` +
  `(${g.clauses.length} clauses).\n\n` +
  `Read ${g.file}. It holds, for your sections only:\n` +
  `- clauses: every clause id, heading, and its VERBATIM award text, mirrored from the Commission's own ` +
  `publication into fwc_clause\n` +
  `- coverage: the service's rule_coverage ledger row for that clause (status: modelled | partial | ` +
  `not_modelled | not_applicable, with a note), or null where the ledger has no row for it at all\n` +
  `- rules: every rule_* row citing that clause, with its columns. A row whose operative_to is not null ` +
  `is a CLOSED historical reading and is not current coverage.\n` +
  `- tests: which test files textually mention the clause — a HINT, not proof.\n\n` +
  `Also read ${SUITES} — every test file in the repo with its describe/it strings.\n\n` +
  `The service repo is at ${SVC}. You may and should grep and read its test/ and src/ directories to ` +
  `check claims. You are expected to open files, not to reason from the index.\n\n` +
  `For EVERY clause in your file — no silent skips, output exactly ${g.clauses.length} entries, one per ` +
  `clause id — assign exactly one category:\n\n` +
  `- tested: the obligation is modelled AND at least one test ASSERTS the behaviour — engine output, an ` +
  `API response, or a property of the loaded data. A row existing is not an assertion. Evidence: the ` +
  `test file and the specific describe/it, which you opened.\n` +
  `- modelled_untested: modelled (rule rows exist, the engine acts on it) but nothing exercises the ` +
  `behaviour. Propose ONE concrete test: the target file (an existing golden file where it fits, else a ` +
  `new one), the input situation, the assertion.\n` +
  `- covered_not_applicable: the clause text genuinely contains nothing a rostering or payroll system ` +
  `can act on — definitions, dispute resolution, consultation procedure, a heading carrying no prose.\n` +
  `- gap: the clause imposes an obligation the service should model and does not.\n` +
  `- mismodelled: what is modelled CONTRADICTS the clause text. Quote both sides.\n\n` +
  `Judging rules, each of which exists because ignoring it produced a wrong answer before:\n\n` +
  `1. THE LEDGER IS NOT EVIDENCE FOR ITSELF. You are shown rule_coverage.status as context, and it is a ` +
  `disposition somebody wrote, not a fact about the code. Several of its notes assert that the engine ` +
  `acts on a clause without anyone having checked that it does, and for five clauses that was false. ` +
  `Never return covered_not_applicable because the ledger says not_applicable. Return it only if you ` +
  `read the clause text and can say WHAT ABOUT THE TEXT makes it unactionable. If the text carries a ` +
  `real obligation, the verdict is gap or mismodelled no matter what the ledger says — and a clause ` +
  `with coverage: null is not thereby out of scope; it is a clause the ledger never saw.\n\n` +
  `2. A CITATION IS NOT A TEST. The tests index is grep output. A clause named in a block comment, in a ` +
  `test that is about something else, or in a note string, is not a clause with a test. Open the file ` +
  `and find the assertion, or the verdict is not 'tested'.\n\n` +
  `3. A parent heading whose children you have judged individually is covered_not_applicable with ` +
  `evidence "container heading, no prose of its own" — judge the leaf clauses, not the heading.\n\n` +
  `4. Schedule rate rows: the published-rates and pay-guide suites price every fwc_penalty and ` +
  `fwc_classification row to the cent, so a schedule rate backed by those suites is 'tested'. Say which ` +
  `suite rather than demanding a bespoke test.\n\n` +
  `5. Do NOT propose tests for judgement-only facts — whether an agreement was genuine, whether a ` +
  `direction was reasonable. But a MISSING CAPTURE FIELD is a gap, not a judgement: say which field or ` +
  `channel would carry the fact (the employee-facts list, a RequiredInput a finding raises, or a named ` +
  `new record).\n\n` +
  `Return via StructuredOutput: {clauses: [{clause, title, category, evidence, proposed_test}]}.`

phase('Audit')
const results = await parallel(GROUPS.map((g) => () =>
  agent(auditorPrompt(g), { label: `audit:${g.sections.join(',')}`, phase: 'Audit', schema: CLAUSE_SCHEMA })
))

// A group whose auditor died resolves to null. Filtering the nulls away — which the first
// implementation did — drops every clause in that group and leaves a run that reads as complete.
// Losing a group silently is the precise defect this command exists to expose, so it is accounted
// for by name instead.
const lostGroups = GROUPS.filter((_, i) => !results[i]).map((g) => g.sections.join(','))

// One verdict per clause, checked rather than requested. Duplicates keep the FIRST and are
// reported, because a second verdict on the same clause means the auditor changed its mind
// without saying so and both readings deserve a look.
const seen = new Map()
const duplicates = []
for (const r of results) {
  if (!r) continue
  for (const c of r.clauses || []) {
    if (seen.has(c.clause)) { duplicates.push(c.clause); continue }
    seen.set(c.clause, c)
  }
}

const expected = GROUPS.flatMap((g) => g.clauses)
const unverdicted = expected.filter((c) => !seen.has(c))
const unexpected = [...seen.keys()].filter((c) => !expected.includes(c))
const all = expected.filter((c) => seen.has(c)).map((c) => seen.get(c))

const counts = VERDICTS.reduce((acc, v) => ({ ...acc, [v]: all.filter((c) => c.category === v).length }), {})

log(`${all.length}/${expected.length} clauses verdicted across ${results.filter(Boolean).length}/${GROUPS.length} auditors. ` +
  `${JSON.stringify(counts)}`)
if (lostGroups.length) log(`AUDITOR FAILED for section group(s): ${lostGroups.join(' | ')} — their clauses are unverdicted, not clean.`)
if (unverdicted.length) log(`${unverdicted.length} clause(s) got NO verdict: ${unverdicted.join(', ')}`)
if (unexpected.length) log(`${unexpected.length} verdict(s) name a clause that is not in the award's enumeration: ${unexpected.join(', ')}`)
if (duplicates.length) log(`${duplicates.length} clause(s) were verdicted twice; the first was kept: ${duplicates.join(', ')}`)

phase('Synthesize')
const synthesis = await agent(`You are synthesizing a clause-level coverage audit of ${AWARD} as ` +
  `implemented in rosterio-compliance-service. Here are ${all.length} per-clause verdicts ` +
  `(categories: ${VERDICTS.join(' / ')}):\n\n${JSON.stringify(all)}\n\n` +
  `Produce:\n` +
  `1. counts — verdicts per category.\n` +
  `2. defects — every 'mismodelled' and every 'gap', deduped, each with a one-line summary and a ` +
  `severity. HIGH = it affects pay or refuses a lawful roster; medium = wrong metadata or citation; ` +
  `low = cosmetic. Order by what UNDERPAYS first: a missing penalty reaches a payslip and nobody ` +
  `notices, which is worse than a missing check that stops a roster loudly.\n` +
  `3. test_plan — merge the proposed_test entries of every modelled_untested and gap clause into a ` +
  `DEDUPED, ordered work list: group clauses belonging in the same test file into one entry (file, ` +
  `clauses[], description of the situations and assertions, priority). Prefer extending existing golden ` +
  `files in ${SVC}/test/golden/ over new files. High priority = pay-affecting behaviour with no test; ` +
  `low = record-keeping and consultation process rules.\n` +
  `4. summary — five sentences a maintainer can act on.\n\n` +
  `Sanity-check the verdicts as you merge, do not just tally them:\n` +
  `- A clause marked covered_not_applicable whose title or evidence involves money or rostering — ` +
  `rates, penalties, loadings, overtime, breaks, leave, rosters, allowances — goes in defects as ` +
  `'verify manually', not into the not-applicable count. That verdict is the one that ends an enquiry, ` +
  `so a doubtful one costs more than a doubtful gap.\n` +
  `- Evidence that cites the coverage ledger rather than the clause text or a file is not evidence. ` +
  `Flag the clause instead of accepting the verdict.\n` +
  `- A 'tested' verdict whose evidence names a file but no describe/it has not been checked. Flag it.\n\n` +
  `Return via StructuredOutput.`,
  { label: 'synthesize', phase: 'Synthesize', schema: SYNTH_SCHEMA, effort: 'high' })

return {
  award: AWARD,
  clausesExpected: expected.length,
  clausesVerdicted: all.length,
  counts,
  unverdicted,
  unexpected,
  duplicates,
  lostGroups,
  verdicts: all,
  synthesis,
}
```

## Reading the result

`unverdicted` and `lostGroups` come first, before any count. A run that verdicted 290 of 332 clauses did
not audit the award; it audited 290 clauses and does not know about 42, and every one of those is in the
same state the sixteen gaps were in — nothing pointing at it, nothing saying so. Re-run those sections
before reporting a total.

`defects` at `high` severity are the output: a clause of the award with an obligation and no
implementation. They are not defects in the sense `award-verify` means — nothing here has been shown to
be *wrong* — they are absences, and an absence is what the four existing layers structurally cannot see.

`test_plan` is a work list, not a verdict. Its entries are proposals an agent wrote from a clause and a
file listing; the file it names may not be the right home and the assertion it describes may need the
award read again before it can be written. Treat it as the shape of the work rather than as instructions.

`counts` is the number worth quoting, and it must be quoted whole. "16 gaps" on its own is a smaller
statement than it looks; `143 tested / 136 not-applicable / 37 modelled-untested / 16 gaps / 0
mismodelled` says both what is missing and how much of the award was already fine, and the second half
is what stops the first from being read as a catastrophe or as a rounding error.
