---
description: |
  Drive an award's rules through the manager UI in a real browser: generate test cases from the award's own thresholds, seed a local stack, build each roster in the screen a manager actually uses, and check what the screen SHOWS against what the engine decided.
  The question no other award command asks. award-verify checks each rule against its clause, award-scenarios checks a whole situation's pay, closure proves every clause was read — all three stop at the engine. A finding the engine gets right and the screen never renders is invisible to every one of them, and it is the only kind of defect a manager experiences directly.
  Cases are GENERATED rather than written, seeded so they reproduce exactly, and drawn from each rule's own numbers so an edge case is the award's threshold, one minute under, and one minute over. Common cases too: a set where every case is red never checks that a lawful roster renders clean.
  Invoke as:
    /autofeature:award-ui <AWARD_CODE>
    /autofeature:award-ui <AWARD_CODE> seed: 4821
    /autofeature:award-ui <AWARD_CODE> count: 20 kinds: min_engagement,max_daily_hours
argument-hint: <AWARD_CODE> [seed: <n>] [count: <n>] [kinds: <rule,rule>]
---

# AutoFeature — Award UI Verification

Checks that the SCREEN shows what the engine decided. Not whether the award is
modelled correctly — `award-verify` and `award-scenarios` own that, and a UI case
that also tried to judge the award would be a worse version of both and would
fail for reasons that have nothing to do with the browser.

The gap this closes is narrow and real: every other award command stops at the
engine. A breach the engine raises and the roster never renders is invisible to
all of them, and it is the only kind of defect a manager meets in person.

## The cases are generated, and that is the point

`npm run ui-cases` in the compliance service samples the award's rules and walks
each across its own threshold. The numbers come from the rules themselves —
`min_engagement` carries 3 hours, `max_daily_hours` carries 9 with an 11 hour
concession, the break table carries its bands — so an edge case is the award's
own figure, one minute under it, and one minute over. Where a threshold is not
derivable from a rule's parameters it emits the common case only and says so,
because an invented edge fails against correct behaviour.

Every case's expectation is DERIVED by running the engine, never written by
hand. That is what makes the comparison meaningful: the screen is checked
against the engine's actual answer for that exact roster, so a disagreement is a
UI defect by construction.

Seeded throughout. `seed: 4821` reproduces a run byte-for-byte, and a report
that names its seed is one somebody else can re-run.

## Step 1: The stack, and refusing to guess about it

Four things have to be up. Check each and say which is missing rather than
failing in the browser twenty minutes later:

```bash
for p in 27019 5112 3200 3000; do
  (exec 3<>/dev/tcp/127.0.0.1/$p) 2>/dev/null && echo "  $p up" || echo "  $p DOWN"
done
```

- **27019** — Mongo, the `shiftos-mongo` container. `podman start shiftos-mongo`.
- **5112** — shiftos-api. `npm run dev` in shiftos-api.
- **3200** — the compliance service. `npm run dev` in rosterio-compliance-service.
  Needed twice over: without it the roster renders "Not checked" on everything
  and every case fails for one reason, which looks like a catastrophe and is a
  missing process — and the seeder in Step 2 asks it what the award needs before
  it can build anybody.
- **3000** — shiftos-manager. `npm start`. Slow to boot; wait for it.

The manager reads `REACT_APP_API_URL` from its `.env`, which points at 5112.

## Step 2: Seed a tenant, from the award itself

```bash
cd shiftos-api && npm run seed:award -- --award <CODE>
```

Works for any mapped award, because it does not know anything about any of them.
It asks `GET /v1/instruments/<CODE>/employee-facts` — the compliance service's
own answer to "what must an employee record carry for this award to price them",
which is what the product already renders the Staff form from — and builds a
cohort covering what it names. Somebody at every classification, somebody on
every employment type, a junior at each year from 16 to 21, a shiftworker, and
one employee carrying every optional fact while the rest carry none, so both
sides of each flag are reachable.

MA000004 gives 18 staff on retail levels; MA000003 gives 14 on fast-food ones,
including its "in charge of 2 or more people" split. Neither award is mentioned
anywhere in the seeder. That is the point: a fixture written for one award is a
copy of that award that can go stale, and this asks the award instead.

It prints a JSON manifest to stdout — the sign-in, the locations, and every
staff member with the profile they satisfy. Keep it: Step 4 resolves each case's
`profile` against it rather than hunting the Staff page.

It **drops and rebuilds** the tenant each run, deliberately, so a run never
depends on the run before it. Confirm `MONGODB_URI` points at the local 27019
before running it, and never point it anywhere else.

No shifts are seeded. The roster is what the UI run is about to build, and a
pre-seeded one collides with it — leftover rosters fail generated cases on
consecutive-days and weekly-hours rules that have nothing to do with them.

### When the richer fixture is the better choice

For MA000004 specifically, `scripts/seed-supermarket.ts` is 650 lines written by
somebody who knew the award: deliberate pairs either side of a boundary, a
Melbourne Cup public holiday, real qualifications, a closed pay period. It is
better than the generic seeder for that award and does not replace it. Use it
when a case needs any of that; use `seed:award` when you need an award the
supermarket seed has never heard of, which is every award but one.

## Step 3: Generate

```bash
cd rosterio-compliance-service
npm run ui-cases -- --award <CODE> --seed <n> --count <n> > /tmp/ui-cases.json
```

Read it before driving anything. Each case carries the rule, the clause, whether
it is `common` or `edge`, the shifts to build, and `expect.mustShow` — every
finding the engine raises, with its verdict and clause. A case whose
`expect.surface` is `clean` is as important as one that breaches: it is the only
thing that checks a lawful roster does not raise a false alarm, which is the
defect that erodes trust in the screen fastest.

## Step 4: Drive the browser

Chrome MCP. For each case, in the Roster page:

1. Pick the case's employee. `profile` names what the case needs
   (`full_time`, an age, `shiftworker`) rather than a person, because the people
   belong to whichever tenant is seeded — resolve it against the Staff page once
   and reuse the mapping.
2. Add Shift for the case's date, set the times and break, save.
3. Read the award chip on the shift card and the findings behind it.
4. Compare against `expect`.

**There is not one `data-testid` in this application.** Selectors are visible
text and roles, which is closer to what a manager sees and more brittle than an
attribute — so a case that cannot find its control is a SELECTOR failure, not a
compliance one, and must be reported as such. Never let a missed click read as a
missing finding.

The text to match, from `AwardFindings.jsx`:

- the chip reads `N issue` / `N issues` for breaches, `N warning` / `N warnings`,
  `N to answer` where the engine wants a figure only a person holds, and
  `Not checked`
- each finding line reads `cl {clause} {message}`, or `Agreed {message}` where it
  comes from the employee's own agreement rather than the award

So a case expecting `{ kind: "max_daily_hours", verdict: "warning", clause: "15.4" }`
should find a warning chip and a line beginning `cl 15.4`.

## Step 5: Clean up, every time

Delete the shifts the run created before reporting. A failed run that leaves
shifts behind poisons the next one: the second run's cases collide with the
first's rosters and start failing on consecutive-days and weekly-hours rules
that have nothing to do with them. Re-seeding also works and is slower.

## Step 6: Report

1. **UI defects** — the engine raised a finding and the screen did not show it,
   or showed a finding the engine did not raise. For each: the case id, the seed,
   the rule and clause, what `expect.mustShow` said, and what was on screen.
   These are the output. A missing breach is worse than a spurious one: a manager
   who sees nothing publishes the roster.
2. **Selector failures** — a control could not be found or driven. Not compliance
   findings. Fixing them usually means adding a `data-testid`, which is worth
   saying out loud since the app currently has none.
3. **Passed** — screen agreed with engine, split by `common` and `edge`, because
   passing every common case and no edge case is a different result from passing
   both.
4. **Coverage** — the seed, the count, and which rules were sampled. Say plainly
   which rules were NOT sampled this run: the generator picks from the award's
   rules, so a twelve-case run over twenty rules leaves most of them untouched,
   and a report that does not say so reads as the award having been UI-tested.

Append to `.autofeature/awards/<CODE>-review.md` under `## UI verification — <date>`.

## What this is not

It is a browser driving a seeded tenant, so it inherits everything about that
tenant: one employer, one award, two locations, and a cohort built to cover the
award's declared facts rather than to look like a real workforce. Nobody works a
pattern, nobody has history, and no two employees are related. It does not prove
the screen is right for an employer it has never seen.

And it cannot tell you the award is modelled correctly — by design. If a case
disagrees with the engine, the engine is the reference and the screen is what is
wrong. Where you doubt the engine itself, that is `/autofeature:award-verify` and
`/autofeature:award-scenarios`, and a clean run here says nothing about either.
