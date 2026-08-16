---
name: award-close
description: |
  Drive an already-mapped award's `Pending:` backlog to zero — one command, resumable, ordered by which gaps under-pay rather than by clause number.
  For an award that has been read (its enumeration transcribed, its coverage ledger written) but not built. `award-map` maps an award nobody has read; this finishes one somebody has.
  Repairs the ledger until the backlog is countable, then closes it clause by clause, guarding every other award against regression on the way.
  Invoke as:
    /autofeature:award-close <AWARD_CODE>
    /autofeature:award-close <AWARD_CODE> clauses: 20,10,17
    /autofeature:award-close <AWARD_CODE> mode:automated
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Agent
---

# AutoFeature — Close an Award's Backlog

`award-map` answers "what does this award say". `award-audit` answers "does the
ledger say what it means". Neither builds anything. This one does: it takes an
award whose backlog is known and drives `Pending:` to zero.

## Missing information is a design task (always active)

"We don't hold that data" is the beginning of the work, not the end of it. Half
of what this command closes will be a clause waiting on a fact, and the answer is
never to leave it waiting. Name the channel — the `employee-facts` list a Staff
form fills in, or the `RequiredInput` prompts a finding raises — and where
neither fits, specify the record concretely enough to build.

`$AUTOFEATURE_HOME/awards/data-requirements.md` has both channels and their
shapes. It introduces no new mechanism, because the service has both already and
the product renders both.

## $AUTOFEATURE_HOME

Same resolution as `award-map`: the directory holding `awards/` alongside this
command. Read `$AUTOFEATURE_HOME/awards/gap-axes.md` and
`$AUTOFEATURE_HOME/awards/service-conventions.md` before Step 3.

## Args

- `<AWARD_CODE>` — required. Refuse without one.
- `clauses: 20,10,17` — close only these, in this order. Use it to take the
  under-payers first and stop, which is a legitimate finish rather than a partial
  one.
- `mode:checkpoint` (default) / `mode:automated`. Two hard checkpoints stop in
  both: the repaired backlog (Step 2) and the final closure (Step 4).

## Why this is not `award-map resume`

`resume` picks up a mapping mid-flight, on its own branch, with its state file
intact, and it assumes Steps 0 to 5 are behind it. Four things differ here and
each changes what the command must do:

- **The award has already shipped, or may have.** Every edit is against something
  an employer might be pricing against today. No-regression is a gate, not a
  closing formality.
- **The unit of work is one clause, not the award.** Closing six clauses and
  stopping is a result. A mapping run has no such stopping point.
- **The order is risk, not clause number.** During a first mapping everything
  gets built anyway so the order is arbitrary. Here it is the whole decision:
  three of one award's 114 residuals said they under-pay, and those three are
  worth more than the other 111 together.
- **The backlog may not be countable yet.** A `Pending:` that names no limb and
  no capture channel is not a task. Repairing that comes before building, and it
  changes the size of what follows in both directions.

---

## Step 0: Preflight

Resolve the service exactly as `award-map` Step 0 does — the same directory
probe, the same `scripts/lib/psql.sh` (there is no system `psql` on this
machine). Then:

1. `git status --porcelain` clean, or the user confirms the tree.
2. Database up and loaded; `npm run verify` and `npm test` both green. Record the
   test count.
3. **Snapshot every mapped award and confirm no diff:**

   ```bash
   for a in $(npx tsx -e 'import{awardsWithEnumerations}from"./src/awards/enumerations";console.log(awardsWithEnumerations().join(" "))'); do
     npm run snapshot -- "$a" -- --check
   done
   ```

   Anything moving here moved before this command started. Resolve it first —
   from this point on the snapshot is the only thing that distinguishes "I broke
   the other award" from "it was already like that".

4. The award has an enumeration (`src/awards/<CODE>/enumeration.ts`). Without one
   this is the wrong command; the award has not been read and `award-map` is.

Branch: `award/<code-lowercase>-close`.

---

## Step 1: The state, and the only ordering that matters

```bash
npm run closure -- <CODE>
```

Record it. Then triage every residual by **how it fails**, which is the ship
criterion from `award-map` Step 8 turned into a work order:

```sql
SELECT clause, left(note, 160) FROM rule_coverage
 WHERE instrument_id = '<CODE>' AND note LIKE 'Pending:%'
 ORDER BY clause;
```

Read each and sort it into one of three:

- **A — silently under-reports.** The rule does not fire, so a figure comes back
  complete-looking and low. Nobody notices, and it is somebody's wages.
- **B — errs toward over-reporting.** A breach flagged that the award permits,
  overtime counted generously. Arguable, never takes money off an employee.
- **C — refuses visibly.** Returns `unevaluated`, or the engine raises. The
  caller knows they did not get an answer.

**A first, always, and A is usually tiny.** On the award this command was written
for, exactly 3 of 114 were A and they shared one cause. B and C are ordinary
backlog and the clause order between them barely matters.

Most notes will not say which they are, because direction-of-error was not a
rule when they were written. Deciding is reading the clause and asking what the
engine does today when the rule is absent. Where you cannot tell, say so and
treat it as A — the conservative direction is to assume a gap under-pays.

---

## Step 2: Repair the ledger until the backlog is countable — HARD CHECKPOINT

Two passes. Neither builds anything; both change what "114 items" means.

### 2a. Every residual on a structured clause names its limb

```bash
npm run closure -- <CODE>     # axis 8 lists them
```

A clause whose own text runs `(a)`, `(b)`, `(c)` and whose note says only "needs
the agreed pattern" cannot be told from a clause with one limb of six open. The
limbs are where an under-payment hides, one level below where the sub-clause axis
stops looking.

Rewrite each note to say which limbs are modelled and which are not:

> `Pending:` cl 20.2 — the 38-hour and 11-hour limbs are modelled. The
> five-days-in-a-week limb and the three limbs keyed to rostered start and finish
> times need the agreed pattern.

**This pass moves items between the three categories, in both directions.** A
clause that looked like one task becomes one limb of six already done; a clause
that looked closed turns out to have an open limb that under-pays. Re-run the
Step 1 triage after it rather than trusting the order you started with.

### 2b. Every `Pending:` names what carries the fact

One sentence in the note: `employee-facts`, a named `RequiredInput` group, or
`neither`.

**`neither` is the answer that matters.** It means the product holds no record
the fact could attach to, so the clause needs a change in `shiftos-api` or
`shiftos-manager` and not in this service at all. Those are not closeable here.
Separate them, and say so at the checkpoint with the model, field and screen each
would need — a list somebody can schedule in the other repo, not a shrug.

### The checkpoint

```bash
npm run rules:load && npm run closure -- <CODE> && npm test
```

Axis 8 at zero means the award is ACCOUNTED and the backlog is complete. Present:

- the backlog **before and after** the repair, and what moved
- the A / B / C split, A itemised in full
- the `neither` list, as work for another repo
- the proposed clause order

Stop here in `mode:automated` too. This is where a person decides what is worth
building and in what order, and it is the last cheap moment to decide it.

---

## Step 3: Close, one clause at a time

For each clause in the agreed order. One clause, one commit, so an interrupted
run leaves the award in a state somebody can read.

Authoring rules are `award-map` Step 6's, unchanged, and are not restated here.
The ones that bite most often:

- cite the clause in `clauses`, and remember `rule_condition.clauses` is a
  **scope, not a citation** — a bare parent clause there makes every condition
  invisible and the award prices nothing
- transcribe a whole contiguous passage into `clause_text`, from the fetched
  text, never from memory. Where two awards disagree the ratio is 343 characters
  against 30, and the short ones produced twelve false findings in one
  verification run
- where a choice over- or under-reports, **over-report the breach and over-report
  the overtime**, and say so in the note

After each clause:

```bash
npm run rules:load && npm run closure -- <CODE> && npm test
```

`npm test` is in that chain deliberately. An earlier loop ran only
`rules:load && closure` and pushed a commit with five failing tests, because
closure was green and nobody re-read the suite count.

### The other awards may not move

`npm test` includes `test/snapshot.test.ts`, which re-prices every mapped award's
cross-product. A failure there is not this award's tests failing — it is this
award's work changing what a DIFFERENT award tells an employer.

That is the specific risk of this command over `award-map`: closing a gap usually
means touching something shared. A new condition kind, a widened vocabulary, a
column, a fix in the engine that another award's rules also reach.

Treat every changed line as a defect until shown otherwise. If the old answer was
genuinely wrong and this work fixed it, regenerate that award's fixture and say
in the commit which answers moved and why. **Never regenerate to make the suite
quiet** — the diff prints every combination with its before and after so it can
be read, and regenerating without reading launders the change into the baseline.

### Fanning out

Independent files can be authored in parallel on **sonnet** — allowances, leave,
termination and evidence do not interact. `rules.sql` and the coverage family
stay in the orchestrator's own context, because that is where the judgement is
spent.

**Any agent that writes needs `isolation: "worktree"`.** Four authoring agents
were once launched into one shared tree and each reported "another session is
editing this tree", which was its three siblings.

---

## Step 4: Confirm — HARD CHECKPOINT

```bash
npm run rules:load && npm run verify && npm run closure -- <CODE> && npm test && npm run typecheck
for a in <every other mapped award>; do npm run closure -- "$a"; done
```

Required:

- `<CODE>` **IMPLEMENTED**, or `Pending:` reduced to items that are all `neither`
  — work that belongs in another repo — with each one named
- every other award's closure **unchanged from Step 1**, verbatim
- the suite green with no fewer tests than the Step 0 baseline
- every snapshot clean, or every diff explained

**Closing a backlog invalidates the verification it had.** Every clause built in
Step 3 added predicate rows nobody has re-derived, so `closure`'s correctness
verdict will read `STALE` (or `NEVER VERIFIED`) even where it read VERIFIED
before this ran. That is the record working. Say it in the report and treat
`/autofeature:award-verify` as the natural next command, scoped to the clauses
this run touched if a full re-run is too much.

Then the report, which is a decision for a person and not a summary:

1. What this award now does that it did not.
2. What it still does not, and which category each remaining item is in.
3. Which other repo's work is now unblocked, or newly required.
4. **What was not checked**, quoting `closure`'s correctness line verbatim. This
   command builds and guards against regression. It does not re-derive a single
   predicate from the award's words — a rule closed here can be wired to the
   wrong day of the week and pass everything above.

---

## Prohibitions

- **Do not promote.** Merging is not deploying and this command does neither.
  `award-map` Step 9.5 is the gated path to the one live database, and it is
  gated because there is no staging tier.
- **Do not close a residual by re-labelling it.** Turning a `Pending:` into a
  `By design:` because it is hard is the failure this whole method exists to
  make visible. `By design:` is for a boundary where closing it would mean
  pretending to verify a judgement — never for a boundary you would rather not
  cross today.
- **Do not touch another award's rules** to make this one work. If a shared rule
  is wrong, that is a finding about both awards and a separate decision.
- **Do not skip Step 2** because the backlog "looks clear enough". It is the step
  that decides whether the number you are burning down is real.

## Resume

`git checkout award/<code-lowercase>-close`, re-run `npm run closure -- <CODE>`,
and continue from the first clause still `Pending:`. The closure run is
idempotent and is the source of truth about where the work actually is, which is
the point of having it. The state file is a convenience; the ledger is the state.
