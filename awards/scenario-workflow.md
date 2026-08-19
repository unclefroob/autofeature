# award-scenarios — the Workflow

The behavioural companion to `award-verify`. `award-verify` checks each rule row against the clause it
cites; this checks whether a real, whole employment situation comes out of the engine paid correctly —
which is a different question, because two rows each right against their clause can still combine into a
wrong total, and a penalty that should fire and silently does not returns a plausible lower number that
no per-row check questions.

The scenarios are priced deterministically by `npm run scenarios` before this runs. This workflow does
only the judgement, and it does it the way the rest of this family does: independently, twice, blind,
then adversarially where the readers and the engine disagree.

## What a scenario record carries

One object per scenario, from `npm run scenarios`:

```
{ id, title, situation, probes, expect,
  input:   { kind: "price" | "check-roster", ... the exact facts priced },
  clauses: [ { clause, heading, text } ],   // the award's own words
  engine:  { outcome: "priced" | "checked" | "refused", ... what the engine actually did } }
```

## Three disciplines this workflow exists to hold

**The blind readers never see the engine's answer.** Stage 1 is handed the situation, the input, and the
clause text — and nothing from `engine`. A reader shown the number it is meant to check will confirm it;
that is not a reading, it is a rubber stamp. The whole value of the pass is that the expected answer is
computed from the award before it meets the engine's.

**A reader reasons only from the clause text it was given.** A model's training-time memory of what a
retail award pays is exactly the thing under test, and it is often a year and a wage-review out of date.
So a reader that needs a rate the provided clauses do not state must say so — `missingClauseText` — and
must not fill the gap from memory. A confident answer built on a hallucinated penalty rate is worse than
an honest "I was not given enough to say".

**A reader states what SHOULD apply, including what the engine might have left out.** The dangerous
failure is not a wrong number, which stands out — it is a penalty that never fired, which returns a
smaller, plausible number and looks like a cheaper shift. So a reader lists the penalties or findings it
expects on its own account, and Stage 2 checks the engine against that list rather than only checking
the segments the engine chose to return.

## Running it

```
Workflow({
  script: <the script below, verbatim>,
  args: { award: "<CODE>", scenarios: <the array from `npm run scenarios`> }
})
```

Cost is two readers per scenario always, plus one adjudicator per scenario — three
agents per scenario, `3N` for N scenarios. A ten-scenario award is small enough to run without
asking; say so before a much larger set.

### The Workflow script

```js
export const meta = {
  name: 'award-scenarios',
  description: 'Behavioural check: price real situations, derive the expected pay from the award blind and twice, adjudicate disagreements adversarially',
  phases: [
    { title: 'Derive',     detail: 'two blind readers per scenario, from clause text and the input, never the engine answer' },
    { title: 'Adjudicate', detail: 'a skeptic sees both readings and the engine answer, and rules on every scenario' },
  ],
}

const ARGS = typeof args === 'string' ? JSON.parse(args) : args
const AWARD = ARGS.award
const SCENARIOS = ARGS.scenarios || []

const DERIVE_SCHEMA = {
  type: 'object',
  required: ['expectedOutcome', 'expected', 'mustAlsoApply', 'confidence', 'reasoning'],
  properties: {
    expectedOutcome: {
      type: 'string', enum: ['priced', 'checked', 'refused'],
      description: 'What the engine SHOULD do with this situation, before you have seen what it did.',
    },
    expected: {
      type: 'array',
      description:
        'For a priced shift, one entry per block of hours you would pay differently: the hours, the ' +
        'percentage of the base rate, and the clause that sets it. For a roster check, one entry per ' +
        'finding you would expect, with kind and clause. For a refusal, a single entry naming the reason.',
      items: {
        type: 'object',
        required: ['what', 'clause', 'arithmetic'],
        properties: {
          what: { type: 'string', description: 'e.g. "4 hours at 150%", or "min_engagement breach", or "refuse: not shiftwork"' },
          ratePercent: { type: 'string', description: 'e.g. "150" — omit for a roster finding or a refusal' },
          hours: { type: 'string' },
          clause: { type: 'string' },
          arithmetic: {
            type: 'string',
            description: 'How you got there, in words. A conclusion with no working is not an answer.',
          },
        },
      },
    },
    mustAlsoApply: {
      type: 'array', items: { type: 'string' },
      description:
        'Penalties, loadings or findings this situation attracts that you are NOT sure the engine will ' +
        'have produced — the things a wrong answer omits rather than gets wrong. Empty if none.',
    },
    missingClauseText: {
      type: 'array', items: { type: 'string' },
      description:
        'Clauses or figures you needed to answer and were NOT given the text for. Name them rather than ' +
        'supplying a rate from memory. Empty if the provided clauses were enough.',
    },
    confidence: { type: 'string', enum: ['high', 'medium', 'low'] },
    reasoning: { type: 'string' },
  },
}

const derivePrompt = (s, letter) => `You are reader ${letter}, pricing one employment situation under
${AWARD} from the award's own words. You have NOT been told what any engine produced, and you must not
guess at it — derive the answer yourself.

SITUATION
${s.situation}

WHAT THIS IS PROBING
${s.probes}

THE EXACT FACTS
${JSON.stringify(s.input, null, 2)}

THE GOVERNING CLAUSES, verbatim from ${AWARD}:
${s.clauses.map((c) => `\n[${c.clause}] ${c.heading}\n${c.text}`).join('\n')}

Derive what the engine SHOULD do:
- If this is a shift to price, give the blocks of hours and the percentage each should be paid, with the
  clause and your arithmetic. Watch the boundaries — an hour that crosses 6pm, midnight, or the overtime
  threshold is two blocks, not one. Where penalties describe the same hours, say which one wins and why
  they do not stack.
- If this is a roster to check, give the findings you would expect, by kind and clause.
- If the situation is one the award says cannot be priced as asked, say "refused" and name the clause.

Reason ONLY from the clause text above. If you need a rate or a rule the text does not give you, put it
in missingClauseText and do not invent it. In mustAlsoApply, name anything this situation attracts that
an engine might silently leave out — the omission is the failure that hides.`

const ADJUDICATE_SCHEMA = {
  type: 'object',
  required: ['verdict', 'severity', 'explanation'],
  properties: {
    verdict: {
      type: 'string',
      enum: ['engine_defect', 'seed_wrong', 'ambiguous', 'engine_correct'],
      description:
        'engine_defect: the engine is wrong and the award says so. seed_wrong: the scenario is ' +
        'mis-specified and the engine is right about what it was actually asked. ambiguous: the award ' +
        'genuinely leaves this open, or the clause text given was not enough to decide. engine_correct: ' +
        'the readers erred and the engine is right.',
    },
    locus: {
      type: 'string',
      description: 'The specific segment, finding or figure in dispute. Omit for engine_correct.',
    },
    expectedVsActual: {
      type: 'string',
      description: 'What the award requires here versus what the engine did. Omit for engine_correct.',
    },
    severity: {
      type: 'string', enum: ['underpays', 'overpays', 'wrong_but_neutral', 'none'],
      description: 'Which direction the defect moves money. Underpayment is the one that reaches a payslip unnoticed.',
    },
    explanation: { type: 'string' },
  },
}

const adjudicatePrompt = (s, readerA, readerB) => `You are adjudicating one ${AWARD} scenario where an
independent reading disagrees with what the engine produced. Prefer to find the ENGINE wrong: your job
is to catch a defect, not to defend the code. But default to "ambiguous" over a forced verdict — a guess
dressed as a ruling is the failure this step exists to avoid.

SITUATION
${s.situation}

WHAT THIS IS PROBING
${s.probes}

THE GOVERNING CLAUSES, verbatim:
${s.clauses.map((c) => `\n[${c.clause}] ${c.heading}\n${c.text}`).join('\n')}

WHAT THE ENGINE ACTUALLY DID
${JSON.stringify(s.engine, null, 2)}

READER A DERIVED
${JSON.stringify(readerA, null, 2)}

READER B DERIVED
${JSON.stringify(readerB, null, 2)}

Rule on it:
- engine_defect if the engine's answer contradicts the clause text — say exactly which segment or finding,
  what the award requires instead, and which way the money moves.
- Pay special attention to anything in a reader's mustAlsoApply that the engine's output does not contain:
  a penalty that never fired is a silent underpayment and is the hardest defect to see.
- seed_wrong if the readers priced a different situation than the one the engine was actually handed —
  the scenario's facts and its prose disagree, or it asks for something the input does not encode.
- ambiguous if the award leaves this genuinely open, or the clause text provided was not enough to decide.
  This is a real answer, not a cop-out.
- engine_correct only if you can show the readers made the error and the engine follows the clause.`

// ---- the pass ----
// Two readers, then always an adjudicator. An earlier version tried to skip
// adjudication where the readers "agreed" with the engine, but agreement is
// exactly what cannot be judged in code here: the readers derive free-text
// segments, and a shape-match — both say "priced", the engine says "priced" —
// hides a wrong rate on a segment underneath. Adjudicating every scenario costs
// one extra agent each and closes that hole. The adjudicator returns
// engine_correct where the readers erred, so nothing is lost but the shortcut.
const results = await pipeline(
  SCENARIOS,

  // Stage 1: two blind readers. Neither is handed `engine`.
  (s) =>
    parallel([
      () => agent(derivePrompt(s, 'A'), { label: `derive:${s.id}:A`, phase: 'Derive', schema: DERIVE_SCHEMA }),
      () => agent(derivePrompt(s, 'B'), { label: `derive:${s.id}:B`, phase: 'Derive', schema: DERIVE_SCHEMA }),
    ]).then((readings) => ({ s, readerA: readings[0], readerB: readings[1] })),

  // Stage 2: adjudicate. Sees both readings AND the engine output — the first
  // point in the pass where the two meet.
  async ({ s, readerA, readerB }) => {
    if (!readerA && !readerB) {
      return { id: s.id, title: s.title, verdict: 'ambiguous', severity: 'none',
               explanation: 'Both readers failed to return; re-run this scenario before trusting it.' }
    }
    const ruling = await agent(adjudicatePrompt(s, readerA, readerB), {
      label: `adjudicate:${s.id}`, phase: 'Adjudicate', schema: ADJUDICATE_SCHEMA,
    })
    return { id: s.id, title: s.title, ...(ruling || { verdict: 'ambiguous', severity: 'none',
             explanation: 'The adjudicator did not return; re-run this scenario before trusting it.' }) }
  },
)

const clean = results.filter(Boolean)
return {
  award: AWARD,
  scenarios: clean.length,
  defects: clean.filter((r) => r.verdict === 'engine_defect'),
  ambiguous: clean.filter((r) => r.verdict === 'ambiguous'),
  seedWrong: clean.filter((r) => r.verdict === 'seed_wrong'),
  confirmed: clean.filter((r) => r.verdict === 'engine_correct'),
}
```

## Reading the result

`defects` block: the engine contradicts the award on a real situation. Each names the segment or finding,
what the award requires instead, and the direction the money moves — `underpays` is the one that reaches
a payslip. Fix the rule or the engine, then re-run the scenario.

`ambiguous`: the award genuinely leaves it open, or the clause text handed to the readers was not enough
to decide. Either is a real finding — the first is a note for the review pack, the second means the
scenario should carry more clauses next time.

`seedWrong`: the test is mis-specified. Fix the scenario in `test/scenarios/<AWARD>.ts`, not the engine.

`confirmed`: both blind readers derived the engine's answer independently and neither expected a penalty
it omitted. This is evidence the situation is handled, not a proof the award is — it is the handful of
situations somebody thought to write, and the coverage line in the report must say so.
