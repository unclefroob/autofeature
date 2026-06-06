---
name: market-review
description: |
  Market & fundability review — is it useful, is there a gap in the market, and can you get funding? Frames the product/idea, runs a market analyst + a competitive-gap analyst + a VC analyst in parallel with live cited web research, stress-tests the thesis with an adversarial bear-case analyst, and synthesizes a managing-partner investment memo (TAM/SAM/SOM, competitive map, fundability verdict with stage/check/valuation, risks, next steps, investor one-pager).
  Decision-support, not financial advice or a funding guarantee.
  Invoke as:
    /autofeature:market-review <product or idea>
    /autofeature:market-review            → reviews the product/idea under discussion (or the current repo)
    /autofeature:market-review offline: <idea>  → skip web research (numbers flagged unsourced)
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
  - WebFetch
  - Agent
  - Workflow
---

# AutoFeature Market Review — Useful? Market Gap? Fundable?

An outside-in review of a product or idea: not its code, but whether it should exist as a business
and whether it can raise money. Runs four analysts (market demand + sizing, competitive gap, VC
fundability, and an adversarial bear case) over **live, cited web research**, and returns an
investment memo with a clear verdict.

For internal product quality use `/autofeature:product-review`; for a single feature's build advice
use `/autofeature:feature-review`. This command is the business/market lens.

## $AUTOFEATURE_HOME

```bash
# Files ship with the plugin — prefer its root; fall back to an explicit home or dev clone.
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

If `$AUTOFEATURE_HOME` doesn't exist, abort with:
`AutoFeature methodology repo missing. Expected at $AUTOFEATURE_HOME.`

---

## Step 1: Capture what's being reviewed

Determine the **product/idea under review** as `IDEA`, from:
- the text after `/autofeature:market-review`, if any,
- the **recent conversation** (what the user has been describing), and
- if `pwd` is a real project, the repo itself (the workflow's Frame phase reads the README/landing
  to ground it — you don't need to scan in main context).

Also capture as `NOTES` anything the user stated about **stage, traction, or constraints** (e.g.
"pre-revenue", "500 signups", "bootstrapped, want a seed round", target geography).

- If there's genuinely nothing to review (no args, no relevant conversation, not a product repo),
  ask one short question: *"What product or idea should I run the market review on?"* and stop.
- Echo what you're reviewing so the user can correct:
  > Market review: **[IDEA]** — [one-line]. Running market + competitive + VC + bear-case analysis…

**Web research mode:** default **on**. If the args contain `offline:` (or the user says "no web /
don't search"), set `webResearch = false` — the memo will flag all figures as unsourced.

---

## Step 2: Run the market-review workflow

Read `$AUTOFEATURE_HOME/adapted/market-review.md` and invoke the **Workflow** tool with the script it
contains, passing:

```
Workflow({
  script: <the script from market-review.md>,
  args: {
    repo:        "[pwd]",
    idea:        "[IDEA]",
    notes:       "[NOTES or '']",
    webResearch: true | false,
    maxVerify:   6            // optional — how many cited URLs the Verify phase re-fetches
  }
})
```

The workflow frames the idea, runs the three analysts in parallel (each citing sources), runs the
bear-case stress test, **re-fetches the cited URLs to confirm they actually support their figures**
(Verify phase — funding comps first, web only), and returns a synthesized investment memo.

Before invoking, give a one-line **cost preview** so the run isn't a surprise:
> Running ~6 agents + live web research + a short citation-verify pass — a few minutes. (`offline:` skips the web.)

Do **not** redo the research yourself.

---

## Step 3: Render the memo

Format the returned memo using the **Report output — Investment Memo** block in `market-review.md`.
Save the full memo (including the sources appendix) to `.autofeature/market-review-[YYYY-MM-DD].md`.

Surface the **citation trust signal** from `citationsChecked`: the header shows
`Sources cited: N · Re-verified: X ✓ / Y ⚠ / Z ✗`, and high-stakes figures (comps, top-down TAM)
carry inline `[verified ✓]` / `[unverified ⚠]` / `[stale]` tags. Keep unverified comps with their
loud tag rather than dropping them.

Always keep the closing caveat: estimates are decision-support, not verified fact — validate before
betting on them; not financial advice or a guarantee of funding.

Then offer:

```
A) Pressure-test a specific number / claim   ← re-run one analyst on that dimension
B) Turn the gaps into a build plan           ← hand off to /autofeature:feature-review or /autofeature
C) Save & done
```

- **A:** re-invoke the workflow's relevant analyst (or a focused web check) on the one claim.
- **B:** distill the most promising direction into a feature request and hand to
  `/autofeature:feature-review` (should we / how) or `/autofeature` (build it).
- **C:** print the saved memo path and exit.

This command never branches, writes product code, contacts investors, or ships — it researches and advises.

---

## File Reference

| File | Purpose |
|------|---------|
| `adapted/market-review.md` | Methodology + the Workflow script (frame → market ∥ gap ∥ VC → bear case → synthesize) |
| `agents/market-analyst.md` | Usefulness & demand + market sizing (TAM/SAM/SOM) |
| `agents/market-gap-analyst.md` | Competitive landscape + white space + defensibility |
| `agents/vc-analyst.md` | Fundability — venture-scale, comps, stage/check verdict |
| `agents/bear-case-analyst.md` | Adversarial skeptic — why it fails, stress-tests the numbers |
