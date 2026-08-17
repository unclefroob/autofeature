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

**One wrinkle, found only by running this against real MA000004 data, not by reasoning about it.**
Several rows in a real table share one `clause_text` verbatim — 7 rows for cl 15.1's span, one per day;
3 for cl 15.4's overtime thresholds, one per employment type; several allowances and roster rules too.
Asking a blind reader to derive "the fields for THIS row" without saying which one is unanswerable — it
would flag nearly every one of those rows as a disagreement for no real defect at all, purely from a
one-in-several chance of guessing which row it was even being asked about. The fix isn't to tell the
reader which row — that leaks the exact fact being verified. It's to have it enumerate every row the
shared text describes, then match each shipped row to whichever derived entry agrees with it best. A
shipped row nothing resembles is real evidence on its own. See "Grouping" below.

## A verdict that claims behaviour must cite the code

The skeptic is briefed to argue the shipped row is wrong, and a skeptic arguing
well will reach for consequences: *this trigger fires once per shift and then
bills every hour of it, so it is a live overpayment*. That sentence was returned
at `confidence: high` against a row that was correct, and the mechanism it
described does not exist — the engine takes the quantity from the caller, and its
own comment says so.

An agent given a clause and a predicate has read neither the engine nor the
sibling award. It can say what the words mean. It cannot say what the code does
unless it opened the code.

**So a verdict asserting behaviour — "this overpays", "this never fires", "this
is unreachable" — must quote the file and line it read.** A behavioural claim with
no citation is an argument about a mechanism the agent imagined, and it is the
most persuasive kind of wrong answer this workflow produces.

This is also why `high` confidence blocks rather than auto-fixes. Three of the
first run's sixteen `high` findings were overturned by reading two files.

## Tables covered

Eight tables where judgement about **when** and **how much** lives, i.e. where a day, a time, a priority
or a threshold can be transposed: `rule_condition`, `rule_span`, `rule_overtime_threshold`,
`rule_junior_band`, `rule_allowance`, `rule_roster`, `rule_leave`, `rule_break_placement`.
`rule_break_entitlement` is the same shape and the same mechanism extends to it the same way — add a
schema entry when an award needs it verified; don't invent new machinery.

`rule_leave` and `rule_break_placement` are not optional extras. They're in the set because
`schema.sql`'s own comments record real production bugs in exactly these two: cl 29.3's casual carer's
leave was once converted from "48 hours of elapsed absence" into rostered days, producing a different
wrong number for every casual, and the break-placement modelling gap (cl 16.5) is in the README's own
account of what shipped late. A verification pass that skips the two tables that already burned this
service once is not a serious verification pass.

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

// Defensive: some runs of this tool deliver `args` as the raw JSON text rather than a parsed
// object — confirmed on a live probe (`typeof args === 'string'`, holding the exact literal it
// was called with) even when the caller passed a real object, not a JSON-encoded string, exactly
// the failure mode the tool's own docs warn about. Parsing here rather than trusting `typeof`
// upstream is what kept a genuinely correct verification run from silently checking zero rows.
const ARGS = typeof args === 'string' ? JSON.parse(args) : args
const AWARD = ARGS.award
const ROWS  = ARGS.rows || []

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
  // Added with MA000003. A reader whose vocabulary cannot express the shipped
  // value will always disagree with it, so this list going stale manufactures
  // false findings — the opposite failure to the one this workflow exists for.
  // It is a copy of db/schema.sql's CHECK and has to move when that does.
  'sunday_overtime_minimum','cycle_ordinary_hours',
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
  rule_leave: {
    fields: 'source (nes|award — "the Act says" and "the award says" are different claims), ' +
      'paid (boolean), applies_to (array of full_time|part_time|casual), worker_type (null=any), ' +
      'accrual_method (progressive|upfront_annual|per_occasion), weeks_per_year (progressive/upfront only), ' +
      'days_per_occasion vs hours_per_occasion — THESE ARE NOT INTERCHANGEABLE. A period stated as ' +
      '"hours of absence" (elapsed time, e.g. cl 29.3\'s 48 hours) is hours_per_occasion; a period stated ' +
      'in DAYS is days_per_occasion. Converting one into the other by multiplying by rostered hours is ' +
      'exactly the defect that shipped once already. At most one of the two may be set. ' +
      'excessive_weeks (the balance, in weeks, at which cl 28.5-style excessive-leave provisions bite; ' +
      'null if the award has none), cumulative (does an unused balance carry over)',
    schema: {
      type: 'object', required: ['source', 'paid', 'accrualMethod', 'reasoning'],
      properties: {
        source: { type: 'string', enum: ['nes', 'award'] },
        paid: { type: 'boolean' },
        appliesTo: { type: 'array', items: { type: 'string', enum: ['full_time', 'part_time', 'casual'] } },
        workerType: { type: ['string', 'null'] },
        accrualMethod: { type: 'string', enum: ['progressive', 'upfront_annual', 'per_occasion'] },
        weeksPerYear: { type: ['number', 'null'] },
        daysPerOccasion: { type: ['number', 'null'] },
        hoursPerOccasion: { type: ['number', 'null'] },
        excessiveWeeks: { type: ['number', 'null'] },
        cumulative: { type: 'boolean' },
        reasoning: { type: 'string' },
      },
    },
  },
  rule_break_placement: {
    fields: 'kind (CLOSED vocabulary: no_break_in_first_hour, no_break_in_last_hour, no_combined_breaks, ' +
      'max_hours_before_meal, rest_breaks_split), value (hours or minutes — read which the clause states; ' +
      'no_break_in_first_hour/last_hour and max_hours_before_meal are HOURS, no_combined_breaks is MINUTES ' +
      'of contiguity, rest_breaks_split has no meaningful value beyond a placeholder)',
    schema: {
      type: 'object', required: ['kind', 'value', 'reasoning'],
      properties: {
        kind: { type: 'string', enum: ['no_break_in_first_hour', 'no_break_in_last_hour',
          'no_combined_breaks', 'max_hours_before_meal', 'rest_breaks_split'] },
        value: { type: 'number' },
        reasoning: { type: 'string' },
      },
    },
  },
}

// ---------- Grouping: rows that share one clause_text share one legal sentence ----------
//
// Checked against real MA000004 data before this shipped: a table-shaped clause — cl 15.1's
// span, cl 15.4's overtime thresholds, several allowances, roster rules, leave types — describes
// SEVERAL rows in one passage. rule_span alone has 7 rows citing IDENTICAL text for cl 15.1, one
// per day. Asking a blind reader to "derive the fields for THIS row" without saying which is
// unanswerable — it has a 1-in-7 chance of even naming the right day, which would flag every
// span row as a disagreement for no real defect at all.
//
// The fix is not to tell the reader which row it's checking — that would leak exactly the fact
// being verified. It's to ask it to enumerate EVERY row the shared text describes, then match
// each shipped row to whichever derived entry agrees with it best. A shipped row with no good
// match — nothing in what the reader derived resembles it — is real evidence on its own.
const groupKey = (row) => `${row.tbl}::${row.clauseText}`

function buildGroups(rows) {
  const groups = new Map()
  for (const row of rows) {
    const k = groupKey(row)
    if (!groups.has(k)) groups.set(k, { tbl: row.tbl, clause: row.clause, clauseText: row.clauseText, rows: [] })
    groups.get(k).rows.push(row)
  }
  return [...groups.values()]
}

// ---------- Stage 1 (per GROUP): two blind readers, in parallel ----------
function readerPromptSingle(group, spec) {
  return `You are reading ONE clause of the ${AWARD} modern award, cited as "${group.clause}". You have ` +
    `not seen how anyone else modelled it — derive the structured fields from the words alone.\n\n` +
    `CLAUSE TEXT (verbatim, the award's own words):\n"""\n${group.clauseText}\n"""\n\n` +
    `Fields to derive: ${spec.fields}\n\n` +
    `If the clause text alone cannot settle a field, say so honestly in "reasoning" and give your best ` +
    `reading rather than a guess dressed as certainty. Days of week: 0=Sunday..6=Saturday, matching the ` +
    `Fair Work convention. Times as 24-hour "HH:MM" (e.g. "21:00", never "9.00 pm" or "21:00:00"). Do ` +
    `not infer anything the text does not say.`
}

function readerPromptGroup(group, spec) {
  return `You are reading ONE clause of the ${AWARD} modern award, cited as "${group.clause}". This ` +
    `clause describes MORE THAN ONE structured rule in a single passage — a table, a list of ` +
    `employment types, a list of days, several named things of the same kind. Nobody is telling you ` +
    `how many there are or which one to focus on. Enumerate EVERY distinct instance the text supports, ` +
    `as an array — no more than the text justifies, no fewer. You have not seen how anyone else ` +
    `modelled this.\n\n` +
    `CLAUSE TEXT (verbatim, the award's own words):\n"""\n${group.clauseText}\n"""\n\n` +
    `Fields to derive PER INSTANCE: ${spec.fields}\n\n` +
    `If nothing in the text distinguishes between instances along some dimension — a general rule ` +
    `stated once rather than spelled out per day or per type — that means the SAME values repeat across ` +
    `every instance of that dimension. Describe every instance the record needs (all seven days if the ` +
    `dimension is day of week, all three employment types if it is that), not fewer just because the ` +
    `text was written generally rather than as a table.\n\n` +
    `Days of week: 0=Sunday..6=Saturday. Times as 24-hour "HH:MM" (e.g. "21:00", never "9.00 pm" or ` +
    `"21:00:00"). Do not infer anything the text does not say.`
}

async function deriveGroup(group) {
  const spec = SCHEMAS[group.tbl]
  if (!spec) return { group, skipped: true, reason: `no schema for ${group.tbl} yet` }
  const many = group.rows.length > 1
  const schema = many
    ? { type: 'object', required: ['items'],
        properties: { items: { type: 'array', items: spec.schema }, reasoning: { type: 'string' } } }
    : spec.schema
  const prompt = many ? readerPromptGroup(group, spec) : readerPromptSingle(group, spec)
  const [a, b] = await parallel([
    () => agent(prompt, { label: `read:${group.tbl}:${group.clause}:A`, phase: 'Re-derive', schema, model: 'sonnet' }),
    () => agent(prompt, { label: `read:${group.tbl}:${group.clause}:B`, phase: 'Re-derive', schema, model: 'sonnet' }),
  ])
  // parallel() resolves a failed/skipped thunk to null rather than throwing — a reader that died
  // is not evidence of anything and must not be diffed against, or the next stage crashes.
  if (!a || !b) return { group, skipped: true, reason: 'a reader failed or was skipped' }
  return { group, arrA: many ? a.items : [a], arrB: many ? b.items : [b] }
}

// ---------- Stage 2 (per ROW, looked up by group): match, then diff, three-way ----------
//
// Postgres serialises `time` as "07:00:00"; a model asked for "the correct time" writes "07:00",
// "7:00 am", "7am" — anything a human would. Plain string equality would flag every time field on
// every row as a disagreement for a formatting difference, not a substantive one, which defeats
// the entire point (found on a dry run against real data before this shipped). Parse anything
// recognisable to minutes-since-midnight; fall back to string equality only for what doesn't parse.
const TIME_RE = /^(\d{1,2})(?::(\d{2}))?(?::\d{2})?\s*(am|pm)?$/i
function toMinutes(v) {
  if (typeof v !== 'string') return null
  const m = v.trim().match(TIME_RE)
  if (!m) return null
  let h = parseInt(m[1], 10)
  const min = m[2] ? parseInt(m[2], 10) : 0
  const suffix = m[3] ? m[3].toLowerCase() : null
  if (suffix === 'pm' && h < 12) h += 12
  if (suffix === 'am' && h === 12) h = 0
  // 24:00 is Postgres's own end-of-day boundary (a shift running to local midnight), distinct
  // from 00:00 — found on the same dry run, when a genuinely correct shiftworker span ("24:00:00"
  // shipped, "24:00" derived) failed to match purely because 24 is one past a normal clock hour.
  if (h === 24 && min === 0) return 24 * 60
  if (h > 23 || min > 59) return null
  return h * 60 + min
}

// field-scoped deliberately: only the columns known to hold a clock time get parsed as one. A
// purely-numeric enum value elsewhere must never be silently reinterpreted as a time in disguise.
const TIME_FIELDS = new Set(['time_from', 'time_to'])
function normalize(v, field) {
  if (Array.isArray(v)) return JSON.stringify([...v].sort())
  if (v === undefined) return null
  if (TIME_FIELDS.has(field) && typeof v === 'string') {
    const mins = toMinutes(v)
    if (mins !== null) return `t:${mins}`
  }
  return v
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
  rule_leave: [['source', 'source'], ['paid', 'paid'], ['appliesTo', 'applies_to'],
    ['workerType', 'worker_type'], ['accrualMethod', 'accrual_method'], ['weeksPerYear', 'weeks_per_year'],
    ['daysPerOccasion', 'days_per_occasion'], ['hoursPerOccasion', 'hours_per_occasion'],
    ['excessiveWeeks', 'excessive_weeks'], ['cumulative', 'cumulative']],
  rule_break_placement: [['kind', 'kind'], ['value', 'value']],
}

// How many of a candidate's fields disagree with the shipped row — the matching score. Matching
// on field agreement rather than on a given identity is what keeps this blind: the candidate that
// looks most like the shipped row wins, and if NOTHING looks like it, that is itself the finding.
function mismatchCount(tbl, shippedPredicate, candidate) {
  return FIELD_MAP[tbl].filter(([readerKey, dbKey]) =>
    normalize(shippedPredicate[dbKey], dbKey) !== normalize(candidate[readerKey], dbKey)
  ).length
}

function bestMatch(tbl, shippedPredicate, candidates) {
  if (!candidates || candidates.length === 0) return null
  return candidates.reduce((best, c) =>
    mismatchCount(tbl, shippedPredicate, c) < mismatchCount(tbl, shippedPredicate, best) ? c : best
  )
}

function diffField(field, shipped, a, b) {
  const s = normalize(shipped, field), av = normalize(a, field), bv = normalize(b, field)
  if (s === av && s === bv) return null
  return { field, shipped, readerA: a, readerB: b }
}

function diffRow(row, groupResultsByKey) {
  const groupResult = groupResultsByKey.get(groupKey(row))
  if (!groupResult || groupResult.skipped) return { row, skipped: true, reason: groupResult ? groupResult.reason : 'no group result' }

  const matchedA = bestMatch(row.tbl, row.predicate, groupResult.arrA)
  const matchedB = bestMatch(row.tbl, row.predicate, groupResult.arrB)
  if (!matchedA || !matchedB) {
    return { row, flagged: true,
      diffs: [{ field: '*', shipped: row.predicate, readerA: matchedA, readerB: matchedB,
        note: 'a reader returned nothing resembling this row at all' }] }
  }

  const diffs = FIELD_MAP[row.tbl]
    .map(([readerKey, dbKey]) => diffField(dbKey, row.predicate[dbKey], matchedA[readerKey], matchedB[readerKey]))
    .filter(Boolean)
  return { row, a: matchedA, b: matchedB, diffs, flagged: diffs.length > 0 }
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

// One derivation per GROUP (a shared clause_text is read once, not once per row that cites it —
// asking the same table-shaped clause 7 times would either waste 6 calls or, worse, get 7
// different arrays and make the matching arbitrary). One diff+refute per original ROW.
const groups = buildGroups(ROWS)
const groupResults = await pipeline(groups, deriveGroup)
const groupResultsByKey = new Map(groupResults.map((r, i) => [groupKey(groups[i].rows[0]), r]))

const results = await pipeline(ROWS, (row) => diffRow(row, groupResultsByKey), refute)

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
