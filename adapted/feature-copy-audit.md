---
status: CUSTOM
description: Copy audit methodology — finds AI-sounding user-facing copy (staccato fragments, dash/colon/semicolon tics, punchline rhythm, marketing-bot vocabulary), scores by surface visibility, and rewrites in a plain human voice. Stack-aware string discovery for React / React Native / Express. Optional per-project voice file.
---

# Feature Copy Audit

Finds and fixes copy that reads like it was generated. The output is a prioritised findings
report (file:line, the current string, which tells it trips, a suggested rewrite) ready to
apply directly or feed into the autofeature pipeline.

**The audit covers user-facing copy only** — marketing pages, in-app UI strings, emails/SMS,
meta tags. It never flags code comments, log lines, commit messages, or test fixtures.

---

## The tells

Copy gets flagged when it trips one or more of these. A single tell in isolation can be
fine; flag when a string trips two or more, or when one surface (a hero, an email) trips
the same tell repeatedly. Rhythm tells are about *density* — one fragment on a page is
style, five is a template.

### 1. Punctuation tells

| Tell | Pattern | Example |
|------|---------|---------|
| Staccato full stops | 3+ consecutive sentences ≤4 words, or any one-word "sentence" run | "Simple. Fast. Done." |
| Dash pivot | Em/en dash used as a dramatic hinge mid-sentence | "Fast — really fast." / "One click — that's it." |
| Headline colon | Colon splitting a heading into label + payoff | "Pricing: simple and transparent" |
| Semicolon anywhere | Semicolons in UI or marketing copy at all | "Plan shifts; approve leave; export payroll" |
| Exclamation marks | Any, outside genuine celebration moments | "Get started today!" |

Regex starting points (catch candidates, then judge by eye — these overmatch on purpose):
```bash
# em/en dashes and semicolons in string literals / JSX text
grep -rnE '—|–| - [a-z]|;' src/ --include='*.tsx' --include='*.jsx' --include='*.ts' --include='*.js' | grep -vE '^\s*(//|\*|import|export type)' | head -40
# staccato runs: short sentence + short sentence inside one string
grep -rnE '"[A-Z][a-z]+( [a-z]+){0,2}\. [A-Z][a-z]+( [a-z]+){0,2}\.' src/ --include='*.tsx' --include='*.jsx' | head -20
```

### 2. Rhythm and structure tells

| Tell | Shape |
|------|-------|
| Punchline fragments | Verbless fragments deployed for punch: "No spreadsheets. No chaos. Just rosters." |
| Negation pivot | "Not just X — Y", "It's not about X, it's about Y", "X isn't Y. It's Z." |
| Mirror contrast | "Less admin. More selling." / "Work smarter, not harder." |
| Rule of three | Triplet lists in every heading and bullet — "plan, approve, and export" everywhere |
| Whether-or | "Whether you're a solo agent or a 50-person office…" |
| Rhetorical question header | "Ready to take control of your roster?" |
| Uniform cadence | Every sentence 8–12 words; no natural variance in length |

### 3. Vocabulary tells

Flag these words/phrases in user-facing strings (case-insensitive):

> seamless(ly), effortless(ly), supercharge, unlock, unleash, elevate, empower,
> streamline, game-changer, game-changing, revolutionize, transform(ative),
> robust, leverage, harness, all-in-one, hassle-free, say goodbye to, in seconds,
> at your fingertips, take … to the next level, we've got you covered, look no
> further, dive in, delve, journey, that's where … comes in, built for the way you work

```bash
grep -rniE 'seamless|effortless|supercharge|unlock|unleash|elevate|empower|streamlin|game.chang|revolution|leverage|harness|all.in.one|hassle.free|say goodbye|at your fingertips|next level|got you covered|look no further|delve|journey' src/ --include='*.tsx' --include='*.jsx' --include='*.ts' --include='*.json' | grep -vE '^\s*(//|\*)' | head -40
```

"Built for", "designed to", and "in seconds" are weaker signals — flag only when they
co-occur with another tell.

### 4. Formatting tells

- Title Case On Every Heading And Button (product copy should be sentence case unless the
  project's existing convention is demonstrably Title Case — check before flagging)
- Emoji in body copy or buttons
- **Bold** mid-sentence for emphasis in running prose

---

## Where copy lives (string discovery)

Scan these surfaces, in priority order. Skip `node_modules`, build output, and test files.

| Surface | Where to look | Visibility |
|---------|---------------|------------|
| Marketing pages | Landing/Hero/Pricing/FAQ/About components; `index.html` | Highest — first thing a prospect reads |
| Onboarding & empty states | First-run screens, zero-data states, tooltips/tours | High — every new user reads them |
| Transactional messages | Email/SMS templates (client or API side), notification strings | High — lands in someone's inbox with your name on it |
| Meta/SEO strings | `<title>`, meta description, og:* tags, manifest descriptions | High — shown on Google/social before the site |
| In-app UI | Button labels, headings, placeholders, helper text, toasts, dialogs, error messages users actually see | Medium |
| Copy/content modules | `copy.ts`, `strings.ts`, `content.ts`, i18n JSON (default locale) | Medium — audit the source locale only |
| Store/app metadata | App Store / Play Store description files if present in repo | High when present |

Discovery commands:
```bash
# dedicated copy/content modules and locales
ls src/**/copy.* src/**/strings.* src/**/content.* src/locales/* src/i18n/* 2>/dev/null
# email/sms templates
grep -rln 'subject\|sendEmail\|sendSMS\|nodemailer\|twilio' src/ --include='*.ts' --include='*.js' | head -10
# marketing surfaces
ls src/pages src/screens src/components 2>/dev/null | grep -iE 'landing|hero|pricing|home|about|faq|onboard|welcome|empty'
```

**API repos:** also scan user-visible response strings — validation messages and error
strings that the client renders verbatim. Skip internal error codes and logs.

---

## Severity rubric

Severity = surface visibility × tell density.

- 🔴 HIGH — marketing surface, onboarding, or outbound email/SMS tripping 2+ tells, or any
  surface that is pure template rhythm (staccato + negation pivot + vocab in one block)
- 🟡 MEDIUM — in-app UI copy (toasts, dialogs, empty states, helper text) tripping tells;
  meta/SEO strings with vocab tells
- 🟢 LOW — rarely-seen strings (admin-only screens, edge-case errors) with single tells;
  Title Case inconsistencies

---

## Rewrite rules

The goal is copy a person would say out loud to a customer — not blander copy.
Keep the meaning and the information; drop the performance.

1. **Real sentences.** Subject, verb. Turn fragment runs into one plain sentence:
   "No spreadsheets. No chaos. Just rosters." → "Build the roster here instead of in a spreadsheet."
2. **Say the concrete thing.** Replace abstract benefit-speak with what the feature
   actually does: "Streamline your workflow" → "Approve leave requests from your phone."
3. **Vary sentence length naturally.** A short sentence is fine when it follows a longer
   one. Three in a row is a tic.
4. **Punctuation diet.** Full stops and commas do almost all the work. No semicolons.
   Dashes only when a spoken aside genuinely needs one — at most one per screen.
   Colons only for true label:value pairs, never in headlines.
5. **Use contractions.** "You'll", "it's", "don't" — the absence of contractions is its
   own tell.
6. **Sentence case** for headings and buttons (unless the project convention is Title Case).
7. **Cut filler intensifiers**: truly, simply, just, incredibly, completely, absolutely.
8. **Kill the pivot constructions.** "Not just X — Y" becomes a direct claim about Y.
   Rhetorical questions become statements or get cut.
9. **Respect the medium.** Button labels stay short (verb-first, ≤3 words). Error messages
   say what happened and what to do next. Placeholders show an example, not an instruction.
10. **Never touch semantics.** Preserve interpolation tokens (`{name}`, `%s`, `${...}`),
    i18n keys, markup, and any string the code matches on (enum-like values, error codes).
    If a string is both displayed and compared, flag it instead of editing it.

**Read-aloud test:** before accepting a rewrite, read it out loud. If it sounds like a
keynote or a billboard, it fails. If it sounds like a competent colleague explaining the
product across a desk, it passes.

---

## Project voice file (optional)

Before auditing or writing, check for a project voice/tone file, in this order:

1. `.autofeature/voice.md`
2. `VOICE.md` / `docs/voice.md` / `docs/tone-of-voice.md`

If found, its tone guidance (formality, humour, audience, banned/preferred words) layers
on top of these rules — the de-AI tells above still apply regardless of voice. If the
voice file *mandates* something this audit flags (e.g. Title Case headings), the voice
file wins; note the exemption in the report.

If no voice file exists and the user has expressed voice preferences during a fix/write
run, offer to save them to `.autofeature/voice.md` so future runs stay consistent.

---

## Report format

Save to `.autofeature/copy-audit-[YYYY-MM-DD].md`:

```markdown
# Copy Audit — [project] — [date]

Voice file: [path | none found]
Surfaces scanned: [list]
Strings reviewed: [N] · Flagged: [N] (🔴 [n] · 🟡 [n] · 🟢 [n])

## 🔴 HIGH
### [file:line] — [surface, e.g. "Landing hero"]
> current: "No spreadsheets. No chaos. Just rosters — built for the way you work."
Tells: staccato fragments, dash pivot, vocab ("built for the way you work")
Suggested: "Build and share the week's roster without a spreadsheet."

[...]

## Patterns across the codebase
[2-4 sentences: the dominant tics, so future copy avoids them]

## Exempt
[strings flagged by regex but judged fine, or protected by the voice file — with reason]
```

Every finding must carry a suggested rewrite — the report should be directly applicable,
not just a list of complaints.
