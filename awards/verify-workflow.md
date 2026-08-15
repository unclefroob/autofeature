---
status: CUSTOM
description: The Workflow script for semantic verification — independent re-derivation + adversarial refutation of every predicate-bearing rule row, checked against the award's own words rather than against itself.
---

# Semantic verification — the Workflow script

## What this closes, and what it doesn't

The eight closure axes in `gap-axes.md` prove a clause was read and a rule is reachable. None of them
prove the rule was **read correctly**. A row can cite the right clause, carry the award's own words
verbatim in `clause_text`, and still have `days_of_week = {6}` where the clause says Sunday, because a
transposition in eleven characters of SQL passes every closure check without difficulty. This is not a
ninth axis — it answers a different question (correctness given coverage, not coverage) and it is
answered probabilistically, not with a boolean, so it stays a separate gate rather than getting folded
into "closed."

The mechanism: every predicate-bearing row already carries two independent statements of the same
claim — `clause_text`, the award's verbatim words, and the structured predicate that was derived from
it. Nothing compares them. So two agents, each given only the clause text and never shown the shipped
row or each other's answer, independently re-derive the predicate. Where both agree with each other and
with what shipped, that is real evidence, because it rules out the one thing self-review can't: the same
misreading producing both the answer and its own check. Where they don't agree, a third agent — briefed
to argue the shipped row is wrong, not to referee — decides whether the disagreement is a real defect or
a benign difference in how the same fact got represented.

This raises confidence. It does not retire the risk. Independent agents can share a correlated blind
spot on an unusual legal construction, so a `high` confidence defect still goes to a human, it's just a
much shorter list than "read every row."

## Tables covered

The six tables where judgement about **when** and **how much** lives, i.e. where a day, a time, a
priority or a threshold can be transposed: `rule_condition`, `rule_span`, `rule_overtime_threshold`,
`rule_junior_band`, `rule_allowance`, `rule_roster`. `rule_break_entitlement`, `rule_break_placement` and
`rule_leave` are the same shape and the same mechanism extends to them the same way — add a schema entry
below when an award needs it verified; don't invent new machinery.

## Running it

Invoked from `.claude/commands/award-verify.md`, which queries the rows and supplies them as `args.rows`
— **Workflow scripts have no database access**, so the row-fetch happens in the command's own context,
not in the script.

```
Workflow({
  script: <the script below, verbatim>,
  args: {
    award: "<CODE>",
    rows: [ /* one entry per predicate-bearing row, shape below */ ]
  }
})
```

Each entry: `{ tbl, clause, clauseText, predicate: {...the table's own field names...} }`. The exact
query is in `award-verify.md` — it mirrors `db/schema.sql` field-for-field so the diff in Stage 2 is
comparing like against like.

### The Workflow script

```js
export const meta = {
  name: 'award-verify',
  description: 'Independent re-derivation + adversarial refutation of every predicate-bearing rule row against the award\'s own words',
  phases: [
    { title: 'Re-derive', detail: 'two blind readers per row, from clause_text alone' },
    { title: 'Diff',      detail: 'shipped vs both readers, field by field' },
    { title: 'Refute',    detail: 'a skeptic argues the shipped row is wrong, only on disagreement' },
  ],
}

const AWARD = args.award
const ROWS  = args.rows || []

// ---------- per-table field vocabulary, kept in lockstep with db/schema.sql ----------
// A reader is only useful if it picks from the SAME vocabulary the database uses. Days are
// 0=Sunday..6=Saturday throughout, matching rule_span.day_of_week and rule_condition.days_of_week.
const ROSTER_KINDS = [
  'min_engagement','max_daily_hours','meal_break_after','rest_between_shifts',
  'max_consecutive_days','max_days_per_week','consecutive_days_off','max_days_per_cycle',
  'ordinary_hours_continuous','transport_reimbursement','part_time_ordinary_ceiling',
  'sunday_days_off','shiftwork_public_holiday_avoidance','roster_change_notice',
  'no_mixed_shiftwork','roster_period_max','shift_hours_continuous','recall_minimum',
  'outside_agreed_pattern','excess_travel_costs','travelling_time','moving_expenses',
]

const SCHEMAS = {
  rule_condition: {
    fields: 'hour_type (ordinary|overtime), days_of_week (array of 0-6, 0=Sunday, null=any), ' +
      'time_from/time_to (local wall-clock, null=all day), public_holiday (true|false|null=either), ' +
      'ot_hours_from/ot_hours_to (hours INTO that day\'s overtime, not into the shift), ' +
      'priority (integer, highest wins), worker_type (standard|shiftworker|null=either), ' +
      'whole_shift (true = evaluate at shift START and apply to every hour)',
    schema: {
      type: 'object', required: ['hourType', 'reasoning'],
      properties: {
        hourType: { type: 'string', enum: ['ordinary', 'overtime'] },
        daysOfWeek: { type: ['array', 'null'], items: { type: 'integer', minimum: 0, maximum: 6 } },
        timeFrom: { type: ['string', 'null'] },
        timeTo: { type: ['string', 'null'] },
        publicHoliday: { type: ['boolean', 'null'] },
        otHoursFrom: { type: ['number', 'null'] },
        otHoursTo: { type: ['number', 'null'] },
        priority: { type: ['integer', 'null'] },
        workerType: { type: ['string', 'null'], enum: ['standard', 'shiftworker', null] },
        wholeShift: { type: 'boolean' },
        reasoning: { type: 'string' },
      },
    },
  },
  rule_span: {
    fields: 'day_of_week (0-6, 0=Sunday), worker_type (standard|shiftworker|null=either), ' +
      'time_from/time_to (the ordinary-hours window that day; a day with no row has NO ordinary hours)',
    schema: {
      type: 'object', required: ['dayOfWeek', 'timeFrom', 'timeTo', 'reasoning'],
      properties: {
        dayOfWeek: { type: 'integer', minimum: 0, maximum: 6 },
        workerType: { type: ['string', 'null'] },
        timeFrom: { type: 'string' },
        timeTo: { type: 'string' },
        reasoning: { type: 'string' },
      },
    },
  },
  rule_overtime_threshold: {
    fields: 'employment_type (full_time|part_time|casual), daily_hours, long_day_hours ' +
      '(the higher figure on ONE nominated day/week — only where the award grants this AND it needs ' +
      'a roster to apply, in which case it should be null, not guessed), weekly_hours',
    schema: {
      type: 'object', required: ['employmentType', 'dailyHours', 'weeklyHours', 'reasoning'],
      properties: {
        employmentType: { type: 'string', enum: ['full_time', 'part_time', 'casual'] },
        dailyHours: { type: 'number' },
        longDayHours: { type: ['number', 'null'] },
        weeklyHours: { type: 'number' },
        reasoning: { type: 'string' },
      },
    },
  },
  rule_junior_band: {
    fields: 'classification (verbatim published name), age_from/age_to (inclusive, null=unbounded), ' +
      'service_months_from/service_months_to (null unless the band turns on length of service too)',
    schema: {
      type: 'object', required: ['ageFrom', 'ageTo', 'reasoning'],
      properties: {
        ageFrom: { type: ['integer', 'null'] },
        ageTo: { type: ['integer', 'null'] },
        serviceMonthsFrom: { type: ['integer', 'null'] },
        serviceMonthsTo: { type: ['integer', 'null'] },
        reasoning: { type: 'string' },
      },
    },
  },
  rule_allowance: {
    fields: 'trigger (employee_attribute|location_attribute|shift_event|claim), ' +
      'unit (per_hour|per_shift|per_week|per_occasion|per_km), ' +
      'all_purpose (true only if the allowance is added to the BASE before penalties, not listed beside them)',
    schema: {
      type: 'object', required: ['trigger', 'unit', 'allPurpose', 'reasoning'],
      properties: {
        trigger: { type: 'string', enum: ['employee_attribute', 'location_attribute', 'shift_event', 'claim'] },
        unit: { type: 'string', enum: ['per_hour', 'per_shift', 'per_week', 'per_occasion', 'per_km'] },
        allPurpose: { type: 'boolean' },
        reasoning: { type: 'string' },
      },
    },
  },
  rule_roster: {
    fields: `kind (CLOSED vocabulary, pick exactly one: ${ROSTER_KINDS.join(', ')}), ` +
      'value/value_secondary (the threshold the clause states), window_days (surrounding roster needed), ' +
      'overtime_consequence (none|whole_day — whole_day ONLY if the clause can name the breaching day)',
    schema: {
      type: 'object', required: ['kind', 'value', 'windowDays', 'overtimeConsequence', 'reasoning'],
      properties: {
        kind: { type: 'string', enum: ROSTER_KINDS },
        value: { type: 'number' },
        valueSecondary: { type: ['number', 'null'] },
        windowDays: { type: 'integer' },
        overtimeConsequence: { type: 'string', enum: ['none', 'whole_day'] },
        reasoning: { type: 'string' },
      },
    },
  },
}

// ---------- Stage 1: two blind readers, in parallel, per row ----------
function readerPrompt(row) {
  const spec = SCHEMAS[row.tbl]
  return `You are reading ONE clause of the ${AWARD} modern award, cited as "${row.clause}". You have ` +
    `not seen how anyone else modelled it — derive the structured fields from the words alone.\n\n` +
    `CLAUSE TEXT (verbatim, the award's own words):\n"""\n${row.clauseText}\n"""\n\n` +
    `Fields to derive: ${spec.fields}\n\n` +
    `If the clause text alone cannot settle a field, say so honestly in "reasoning" and give your best ` +
    `reading rather than a guess dressed as certainty. Days of week: 0=Sunday..6=Saturday, matching the ` +
    `Fair Work convention. Do not infer anything the text does not say.`
}

async function deriveTwice(row) {
  const spec = SCHEMAS[row.tbl]
  if (!spec) return { row, skipped: true, reason: `no schema for ${row.tbl} yet` }
  const [a, b] = await parallel([
    () => agent(readerPrompt(row), { label: `read:${row.tbl}:${row.clause}:A`, phase: 'Re-derive', schema: spec.schema, model: 'sonnet' }),
    () => agent(readerPrompt(row), { label: `read:${row.tbl}:${row.clause}:B`, phase: 'Re-derive', schema: spec.schema, model: 'sonnet' }),
  ])
  // parallel() resolves a failed/skipped thunk to null rather than throwing —
  // a reader that died is not evidence of anything and must not be diffed
  // against, or the diff stage crashes dereferencing it.
  if (!a || !b) return { row, skipped: true, reason: 'a reader failed or was skipped' }
  return { row, a, b }
}

// ---------- Stage 2: plain diff, no agent — shipped vs each reader, three-way ----------
function normalize(v) {
  if (Array.isArray(v)) return JSON.stringify([...v].sort())
  if (v === undefined) return null
  return v
}

function diffField(field, shipped, a, b) {
  const s = normalize(shipped), av = normalize(a), bv = normalize(b)
  if (s === av && s === bv) return null
  return { field, shipped, readerA: a, readerB: b }
}

const FIELD_MAP = {
  rule_condition: [['hourType', 'hour_type'], ['daysOfWeek', 'days_of_week'], ['timeFrom', 'time_from'],
    ['timeTo', 'time_to'], ['publicHoliday', 'public_holiday'], ['otHoursFrom', 'ot_hours_from'],
    ['otHoursTo', 'ot_hours_to'], ['priority', 'priority'], ['workerType', 'worker_type'], ['wholeShift', 'whole_shift']],
  rule_span: [['dayOfWeek', 'day_of_week'], ['workerType', 'worker_type'], ['timeFrom', 'time_from'], ['timeTo', 'time_to']],
  rule_overtime_threshold: [['employmentType', 'employment_type'], ['dailyHours', 'daily_hours'],
    ['longDayHours', 'long_day_hours'], ['weeklyHours', 'weekly_hours']],
  rule_junior_band: [['ageFrom', 'age_from'], ['ageTo', 'age_to'],
    ['serviceMonthsFrom', 'service_months_from'], ['serviceMonthsTo', 'service_months_to']],
  rule_allowance: [['trigger', 'trigger'], ['unit', 'unit'], ['allPurpose', 'all_purpose']],
  rule_roster: [['kind', 'kind'], ['value', 'value'], ['valueSecondary', 'value_secondary'],
    ['windowDays', 'window_days'], ['overtimeConsequence', 'overtime_consequence']],
}

function diffRow(derived) {
  if (derived.skipped) return derived
  const { row, a, b } = derived
  const diffs = FIELD_MAP[row.tbl]
    .map(([readerKey, dbKey]) => diffField(dbKey, row.predicate[dbKey], a[readerKey], b[readerKey]))
    .filter(Boolean)
  return { row, a, b, diffs, flagged: diffs.length > 0 }
}

// ---------- Stage 3: adversarial skeptic, ONLY on flagged rows ----------
async function refute(diffed) {
  if (diffed.skipped || !diffed.flagged) return { ...diffed, verdict: null }
  const { row, diffs } = diffed
  const prompt = `You are checking one rule against the ${AWARD} award. Two independent readers, given ` +
    `only the clause text, derived at least one field differently from what was shipped. Your job is to ` +
    `argue the SHIPPED row is wrong — find the real defect if there is one. Only report no defect if you ` +
    `genuinely cannot construct the case; a benign difference in how the same fact is represented is not ` +
    `a defect, but a wrong day, a wrong boundary, or a swapped priority is.\n\n` +
    `CLAUSE "${row.clause}" TEXT:\n"""\n${row.clauseText}\n"""\n\n` +
    `SHIPPED predicate: ${JSON.stringify(row.predicate)}\n\n` +
    `FIELDS WHERE THE INDEPENDENT READERS DISAGREED WITH IT:\n${JSON.stringify(diffs, null, 2)}\n\n` +
    `Decide field by field. Cite the exact words of the clause that settle each one.`
  const verdict = await agent(prompt, {
    label: `refute:${row.tbl}:${row.clause}`, phase: 'Refute', model: 'opus',
    schema: {
      type: 'object', required: ['isDefect', 'confidence', 'explanation'],
      properties: {
        isDefect: { type: 'boolean' },
        confidence: { type: 'string', enum: ['low', 'medium', 'high'] },
        explanation: { type: 'string' },
        correctedFields: { type: 'object' },
      },
    },
  })
  return { ...diffed, verdict }
}

const results = await pipeline(ROWS, deriveTwice, diffRow, refute)

const skipped = results.filter((r) => r && r.skipped)
if (skipped.length) log(`${skipped.length} row(s) had no schema yet and were not verified: ${[...new Set(skipped.map(s => s.reason))].join('; ')}`)

const clean = results.filter((r) => r && !r.skipped && !r.flagged)
// flagged but the skeptic agent itself failed/skipped — NOT the same as "cleared".
// A row that disagreed and then got no verdict at all is unresolved, not fine,
// and must not silently disappear from the tally the way a `null` would.
const unresolved = results.filter((r) => r && r.flagged && !r.verdict)
const cleared = results.filter((r) => r && r.flagged && r.verdict && !r.verdict.isDefect)
const defects = results.filter((r) => r && r.flagged && r.verdict && r.verdict.isDefect)

log(`${clean.length} rows agreed across both independent readers and the shipped row. ` +
  `${cleared.length} disagreed but the skeptic found no real defect. ${defects.length} confirmed. ` +
  `${unresolved.length} disagreed and got no verdict — the skeptic agent failed and these need a re-run.`)

return {
  award: AWARD,
  checked: results.filter((r) => r && !r.skipped).length,
  skippedNoSchema: skipped.length,
  clean: clean.length,
  cleared: cleared.map((r) => ({ tbl: r.row.tbl, clause: r.row.clause, note: r.verdict.explanation })),
  unresolved: unresolved.map((r) => ({ tbl: r.row.tbl, clause: r.row.clause, diffs: r.diffs })),
  defects: defects.map((r) => ({
    tbl: r.row.tbl, clause: r.row.clause, clauseText: r.row.clauseText,
    diffs: r.diffs, confidence: r.verdict.confidence, explanation: r.verdict.explanation,
    correctedFields: r.verdict.correctedFields,
  })),
}
```

## Reading the result

`defects` with `confidence: "high"` are a hard stop — fix the row or write down, in the state file, why
the skeptic is wrong (that override is itself worth a second pair of eyes before ship). `medium`/`low`
go in the review pack for the human sign-off gate; they don't block on their own, because a gate that
blocks on every low-confidence flag gets ignored the first time it's wrong. `cleared` is worth keeping
in the pack too — it's the record of what disagreement turned out to be benign, which is useful the next
time the same reasoning shows up on a different clause. `unresolved` is a real disagreement between
independent readers that never got a verdict because the skeptic agent itself failed — re-run just those
rows before treating the pass as clean; an unresolved disagreement is closer in spirit to a defect than
to a clean row, and must never be counted as either.
