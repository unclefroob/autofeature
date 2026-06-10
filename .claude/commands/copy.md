---
name: copy
description: |
  Copy audit and rewrite — make user-facing copy read human, not generated.
  Audit mode (default): scans marketing pages, in-app strings, emails/SMS, and meta tags for AI tells (staccato fragments, dash/colon/semicolon tics, punchline rhythm, marketing-bot vocabulary), produces prioritised findings with suggested rewrites.
  Fix mode: applies the rewrites in place — string edits only, never code semantics.
  Write mode: drafts new copy in the project's voice (reads .autofeature/voice.md when present).
  Invoke as:
    /autofeature:copy                          → audit current repo's user-facing copy
    /autofeature:copy path: src/pages/Landing.tsx → audit one file/directory
    /autofeature:copy fix                      → audit, then apply the rewrites
    /autofeature:copy fix: <scope>             → fix a specific surface/finding
    /autofeature:copy write: <what>            → draft new copy (e.g. "hero copy for pricing page")
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Agent
  - mcp__trello__addCard
  - mcp__trello__getLists
---

# AutoFeature Copy — Audit, Fix, Write

De-AI the copy. The methodology (tell catalogue, severity rubric, rewrite rules) lives in
`adapted/feature-copy-audit.md` — this command orchestrates it.

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

## Step 1: Parse Arguments and Select Mode

Arguments = everything after `/autofeature:copy` in the user's message.

**Mode detection:**
- Contains `write:` → **WRITE MODE** — extract the brief after `write:` as `WRITE_REQUEST`
- Contains `fix:` → **FIX MODE** — extract scope after `fix:` as `FIX_SCOPE` (a path, a surface name, or a prior audit path)
- Equals/contains bare `fix` → **AUDIT MODE** with `AUTO_FIX=true` (audit, then apply without re-asking which mode)
- Contains `path:` → extract as `SCAN_PATH`, proceed to **AUDIT MODE** scoped to it
- Empty or `audit` → **AUDIT MODE**, whole repo

**Voice file** (all modes): check in order `.autofeature/voice.md`, `VOICE.md`, `docs/voice.md`,
`docs/tone-of-voice.md`. If found, set `VOICE_FILE` and pass its path into every agent prompt below.

---

## Step 2 (AUDIT MODE): Run the Copy Audit

### 2a. Surface discovery

Spawn a lightweight scan to enumerate where copy lives — do **not** grep in main context:

```
Agent({
  description: "Copy surface map",
  subagent_type: "Explore",
  model: "haiku",   // enumerate-only — orchestrator/model-tiers.md
  prompt: "Read the 'Where copy lives' section of $AUTOFEATURE_HOME/adapted/feature-copy-audit.md
  and run its discovery commands in [pwd or SCAN_PATH].

  Return a bucketed list of files containing user-facing copy:
  1. marketing — landing/hero/pricing/about/FAQ pages, index.html
  2. onboarding — first-run screens, empty states, tours
  3. transactional — email/SMS/notification templates (client or API)
  4. meta — <title>, meta description, og:* tags, manifests
  5. in-app — components with button labels, toasts, dialogs, errors, placeholders
  6. modules — copy.ts/strings.ts/content.ts, default-locale i18n JSON

  Paths + one-line roles only. Skip tests, build output, node_modules, code comments, logs.
  Under 400 words."
})
```

### 2b. Audit fan-out

Group the discovered files into 2–4 buckets by surface (merge small buckets). Spawn one
auditor per bucket in a **single parallel message**:

```
[single message — one Agent call per bucket]

Agent({
  description: "Copy audit — [bucket name]",
  subagent_type: "general-purpose",
  model: "sonnet",   // audit judgment — orchestrator/model-tiers.md
  prompt: "Read $AUTOFEATURE_HOME/adapted/feature-copy-audit.md in full — the tell catalogue,
  severity rubric, and rewrite rules — and apply it to these files: [bucket file list].
  Voice file: [VOICE_FILE path, or 'none — default rules apply'].

  For every flagged string return: file:line · surface · the current string verbatim ·
  which tells it trips · severity (🔴/🟡/🟢 per the rubric) · a suggested rewrite that
  follows the rewrite rules (read-aloud test).

  Rules:
  - Flag strings tripping 2+ tells, or 1 tell repeated across the surface.
  - Every finding MUST include a suggested rewrite.
  - Preserve interpolation tokens, i18n keys, and strings the code matches on — if a string
    is both displayed and compared, mark it 'flag-only, do not auto-edit'.
  - Also return an 'exempt' list: regex hits you judged fine, with one-line reasons.

  Return the structured findings list. Under 800 words per bucket."
})
```

### 2c. Merge and report

Combine bucket results. Deduplicate identical strings that appear in multiple files (one
finding, all locations listed). Write the report per the format in
`adapted/feature-copy-audit.md` to `.autofeature/copy-audit-[YYYY-MM-DD].md` and set
`COPY_AUDIT_PATH`.

Show the user a summary: counts by severity, the 3–5 worst offenders with current →
suggested side by side, and the "patterns across the codebase" paragraph.

### 2d. Offer next steps

Skip this step if `AUTO_FIX=true` — go straight to Step 3 with `FIX_SCOPE = all 🔴 + 🟡 findings`.

```
Found [N] AI-sounding strings (🔴 [n] high, 🟡 [n] medium, 🟢 [n] low).

A) Rewrite all high + medium now        ← recommended
B) Rewrite a specific surface           ← pick from list
C) Create a Trello card for this work
D) Audit only — no changes
```

**A** → `FIX_SCOPE = all 🔴 + 🟡 findings`, proceed to Step 3.
**B** → show surfaces with finding counts; user picks; proceed to Step 3.
**C** → call `mcp__trello__getLists`, ask which list, create a card titled
`Copy de-AI pass — [project]` with the severity summary + `COPY_AUDIT_PATH`. Exit.
**D** → print `COPY_AUDIT_PATH` and exit.

---

## Step 3 (FIX MODE): Apply the Rewrites

Copy fixes are micro-tier (string literals only) — apply directly, no autofeature pipeline.

If entered via `fix:` without a prior audit this session, first run Step 2 scoped to
`FIX_SCOPE`, then continue here.

### 3a. Branch

If the working tree is clean and on a default branch, create `copy/de-ai-[YYYY-MM-DD]`.
If the user is already on a feature branch or has uncommitted work, stay put and say so.

### 3b. Rewrite fan-out

One agent per surface bucket from the findings, **single parallel message**:

```
Agent({
  description: "Copy rewrite — [bucket name]",
  subagent_type: "general-purpose",
  model: "opus",   // writing quality is the whole point — orchestrator/model-tiers.md (borderline: drop to sonnet for in-app/low-visibility buckets)
  prompt: "Read the rewrite rules in $AUTOFEATURE_HOME/adapted/feature-copy-audit.md.
  Voice file: [VOICE_FILE path or 'none'].
  Apply these findings from [COPY_AUDIT_PATH]: [bucket's findings, with file:line + suggested rewrites].

  For each: open the file, locate the string, replace it. You may improve on the suggested
  rewrite if you can do better under the rules — but:
  - Edit string literals / JSX text nodes ONLY. Never change identifiers, keys, logic, props order, or markup structure.
  - Preserve interpolation tokens ({name}, %s, ${...}) exactly.
  - Skip anything marked 'flag-only' — list it back instead.
  - Keep button labels ≤3 words, verb-first.

  Return: list of edits made (file:line, old → new) and anything skipped with reason."
})
```

### 3c. Verify and summarize

- Re-run the punctuation/vocab grep candidates from the methodology over the edited files —
  confirm the flagged tells are gone (or consciously exempted).
- Run the project's typecheck/build if one exists (`npm run typecheck || npm run build`) —
  string edits shouldn't break it, but interpolation mistakes would.
- Show the user a diff summary (`git diff --stat` + the 5 most visible before/afters) and
  the skipped/flag-only list. Do **not** commit — leave the diff for the user to review,
  and point them at `/autofeature [skip-product-review] <…>` or their normal flow to ship.

---

## Step 4 (WRITE MODE): Draft New Copy

1. **Context**: read `VOICE_FILE` if present; skim the README and the surface the copy is
   for (the component/page named in `WRITE_REQUEST`, plus 1–2 neighbouring surfaces so the
   register matches the rest of the product). If the brief is too thin to know what the
   product/feature actually does, ask one clarifying question — concrete claims need facts.
2. **Draft**: produce 2–3 options per string, each following the rewrite rules in
   `adapted/feature-copy-audit.md` (real sentences, concrete claims, punctuation diet,
   read-aloud test). Label the register of each option (plainest / warmer / tightest).
3. **Deliver**: show options side by side. If the user picks, apply the edit (same
   constraints as Step 3b). If no voice file exists, offer to save the chosen register to
   `.autofeature/voice.md` so future audits and writes stay consistent.

---

## File Reference

| File | Purpose |
|------|---------|
| `adapted/feature-copy-audit.md` | Methodology — AI-tell catalogue, string discovery, severity rubric, rewrite rules, voice-file protocol, report format |
| `.autofeature/voice.md` (per-project, optional) | Project tone of voice — layers on top of the de-AI rules |
| `.autofeature/copy-audit-[date].md` (output) | Findings report with suggested rewrites |
