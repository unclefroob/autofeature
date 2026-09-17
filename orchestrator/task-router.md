---
name: task-router
purpose: Triage a free-form request to the right intent, target repo, command/skill/specialist and model tier — before anything assumes it is a feature build
status: CUSTOM
description: Read by /autofeature (Step 0.75) and /autofeature:do. Classifies the request's INTENT (build, fix, investigate, audit, test, review, award), resolves the TARGET (which repo and surface "the ios app" means), picks the ROUTE from a table of existing commands and specialists, and assigns a MODEL per orchestrator/model-tiers.md. Emits a Route Card, then dispatches. Investigate requests run the stack's architect in read-only survey mode rather than the build pipeline.
---

# Task Router

`/autofeature` was built for one kind of request: *build this*. Handed "have a look into the iOS app
and see how the styles are", the pipeline would ask automated-or-checkpoint, interrogate a feature
that does not exist, write a Feature Brief, and cut a branch for a question. The request needed a
read-only survey by someone who knows SwiftUI, on Sonnet, ending in a report.

This file decides, before any of that, **what kind of work the request is, where it lands, who does
it, and on which model**. It adds no methodology of its own — every route ends in a command,
methodology or agent that already exists.

---

## Step 1: Intent

Pick the **one** intent that best fits. Read the verb and the deliverable the user expects, not the
nouns — "look at the styles" and "restyle the app" share nouns and nothing else.

| Intent | The user expects back | Signals |
|--------|-----------------------|---------|
| `build` | A PR with new behaviour | add, build, implement, create, support, let users…; a Trello card |
| `fix` | A PR that corrects behaviour | broken, bug, crash, doesn't work, wrong, regression, error text |
| `investigate` | A **report** — how something is, today | have a look, see how, how does, what are, where is, explain, map out, understand, is there |
| `audit` | A report **against a standard**, ranked findings | audit, check against, is it consistent, what's wrong with, clean up (before any edits) |
| `test` | PASS/FAIL per flow on a running app | test, try it, drive, click through, does X still work on the simulator |
| `review` | An opinion — product, feature or market | is it worth, should we, gaps, fundable, market |
| `award` | Award-interpretation work in rosterio-compliance-service | modern award, MA0000xx code, clause, penalty rate, pay rules |

Rules:

- **Report before edits.** A request that both asks what's there and hints at change ("look at the
  styles and tidy them up") is `investigate`/`audit` first, with the build offered as the follow-up.
  Surveying first is cheap; an unasked-for refactor is not.
- **Compound requests** ("check the styles, then build a dark mode") become an ordered chain of
  routes, each with its own card. Never merge them into one pipeline run.
- **Confidence.** Record `high` when one intent clearly fits, `low` when two fit about equally.
  Low confidence is a question to the user (Step 5), not a coin flip.

---

## Step 2: Target

Resolve **which repo** and **which surface** the request means. "The iOS app" is a platform, not a
path.

1. **Explicit path or repo name** in the request → use it.
2. **CWD is a repo matching the platform** → use it.
3. **Otherwise, scan siblings** — the CWD's parent, or `~/dev` when the CWD is not a repo — and
   classify each by signature, not by name:

   ```bash
   ROOT=$( [ -d .git ] && dirname "$PWD" || echo "$HOME/dev" )
   for d in "$ROOT"/*/; do
     sig=""
     { [ -f "$d/project.yml" ] || ls "$d"/*.xcodeproj "$d"/*.xcworkspace >/dev/null 2>&1 || [ -f "$d/Package.swift" ]; } && sig="$sig ios"
     { [ -f "$d/build.gradle.kts" ] || [ -f "$d/app/build.gradle.kts" ] || [ -f "$d/build.gradle" ]; } && sig="$sig android"
     grep -q '"react-native"' "$d/package.json" 2>/dev/null && sig="$sig react-native"
     grep -qE '"(express|fastify|@nestjs/core|koa)"' "$d/package.json" 2>/dev/null && sig="$sig backend"
     grep -qE '"(react|next|vite)"' "$d/package.json" 2>/dev/null && ! grep -q '"react-native"' "$d/package.json" && sig="$sig web"
     [ -n "$sig" ] && echo "$(basename "$d"):$sig"
   done
   ```

   A React Native app also counts as "the iOS app" to a user; list it, marked `react-native`.

4. **Pick:**
   - Exactly one match → use it.
   - Several, and the conversation or CWD names a product (`ritchies`, `rosterio`, `shiftos`,
     `pathiq`) → the match with that prefix.
   - Several and nothing disambiguates → ask (Step 5), listing each candidate with its signature.
     Do not pick the most recently modified; that is how a survey runs on the wrong app.

Record `TARGET_REPO`, `STACK` (`swift` · `kotlin-compose` · `react-native` · `react` ·
`express-mongo`) and, when the request names one, `SURFACE` (styles, navigation, networking, a
named screen).

---

## Step 3: Route

First matching row wins. Every route is something that already exists — this table only chooses.

| Intent | Condition | Route |
|--------|-----------|-------|
| `build` | Target is a `ritchies-*` repo, or the feature spans the Ritchies API and its clients | `/autofeature:ritchies` |
| `build` | Needs deploy verification against Railway/Netlify | `/autofeature:fullrun` |
| `build` · `fix` | anything else | `/autofeature` pipeline (return to Step 1 of `autofeature.md`) |
| `fix` | Root cause unknown ("it crashes sometimes") | `adapted/feature-investigate.md` first, then `/autofeature` with the finding |
| `investigate` | Question about **coding conventions across the repo** ("how do we do errors / validation / logging") | `/autofeature:patterns` (audit) |
| `investigate` · `audit` | Backend API **shape** — layering, routes, controllers, services | `/autofeature:backend-api-skills audit` |
| `investigate` · `audit` | User-facing **copy / wording** | `/autofeature:copy` |
| `investigate` · `audit` | **SEO** | `/autofeature:seo` |
| `investigate` · `audit` | Whole-product **gaps or broken flows** | `/autofeature:review product` |
| `investigate` · `audit` | Anything else about one codebase — styles, theming, design system, navigation, state, networking, accessibility, performance, architecture | **Specialist survey** (Step 3a) |
| `test` | Ritchies iOS change | `/autofeature:ritchies-ios-test` |
| `test` | Web app that seeds its own fixtures | `/autofeature:ui-test` |
| `test` | anything else | `/autofeature:test` |
| `review` | one feature / whole product / market / scope | `/autofeature:review feature\|product\|market\|scope` |
| `award` | mapped already? → close / verify / audit / scenarios / ui / drift; not yet → map | the matching `/autofeature:award-*` (read each one's description; ask when two fit) |

### 3a. Specialist survey

For `investigate`/`audit` with no dedicated command. The specialist who would **build** in this
stack is the one who knows what good looks like in it, so the survey uses the same agent file,
read-only.

| STACK | Agent file |
|-------|-----------|
| `swift` | `agents/swift-architect.md` |
| `kotlin-compose` | `agents/kotlin-compose-architect.md` |
| `react-native` | `agents/react-native-architect.md` |
| `react` | `agents/react-architect.md` |
| `express-mongo` | `agents/express-mongo-architect.md` (+ `mongo-data-modeler.md` when the question is about the data) |

Several stacks in scope (e.g. "compare styles on iOS and Android") → one survey per stack, spawned in
parallel, then a short comparison written by the orchestrator.

Spawn the agent per `agents/README.md` (agent file inlined into a `general-purpose` prompt) with this
job block appended — it overrides the agent's `design`/`implement` sections:

```
Mode: survey — READ-ONLY. Do not edit, create, commit or branch. Do not run builds that write to
the repo. You are reporting what exists, not proposing a build.

Repo: [TARGET_REPO]
Question (verbatim): [user request]
Surface: [SURFACE or "infer from the question"]
Patterns file: [path + status, or none]
Ritchies conventions: [$AUTOFEATURE_HOME/ritchies/conventions.md — only for ritchies-* repos]

Deliver, under 1200 words:
1. Answer — 3–6 sentences answering the question as asked.
2. How it works today — the mechanism, with file:line for each claim. Name the source of truth
   (theme file, token enum, asset catalog, design-system module) or state that there isn't one.
3. Consistency — the canonical pattern, the variants, and a COUNT for each (e.g. "Color.brand used
   in 41 files; hardcoded Color(red:…) in 17"). Counts come from grep, not estimates; give the grep.
4. Findings — ranked 🔴 / 🟡 / 🟢, each with file:line and why it matters for a user or maintainer.
   Judge against your agent file's standards and the platform baseline you would build to.
5. Suggested next steps — at most 3, each a ready-to-run /autofeature… prompt. Do not do them.

Every claim cites a file. Anything you could not verify, mark "unverified" — do not round it up.
```

**Surface checklists** — add the matching lines to the job block so the survey covers the obvious
ground rather than whatever it opened first:

- **styles / theming / design system** — where colours, type, spacing, radii, shadows are defined
  (tokens, `Color`/`Font` extensions, asset catalog, `MaterialTheme`, tailwind config, `StyleSheet`);
  hardcoded literals vs token use, with counts; dark-mode handling; Dynamic Type / font scaling;
  reusable components and view modifiers vs one-off styling; platform design language (Liquid Glass
  on iOS 26+, Material 3 on Android) adopted, partial or absent; parity with the sibling platform
  when one exists.
- **navigation** — the router/stack type, deep links, modal vs push rules, back-stack handling.
- **networking / data** — client layer, auth header injection, error mapping, retries/timeouts,
  caching.
- **state** — state container, observation style, where side effects live.
- **accessibility** — labels/identifiers, contrast, hit targets, font scaling, VoiceOver/TalkBack
  order.
- **performance** — list virtualisation, image loading, main-thread work, re-render/recompose churn.

---

## Step 4: Model

Take tiers from `orchestrator/model-tiers.md` — floor Sonnet, no Haiku — and apply these for routed
work:

| Route | Model | Why |
|-------|-------|-----|
| Specialist survey, one stack | `sonnet` | Code comprehension and enumeration — the base-tier workhorse |
| Specialist survey, parallel stacks | `sonnet` each; the comparison is written in the orchestrator loop | No extra agent for a paragraph |
| Survey whose findings will **decide** a high-risk change (auth, permissions, payments, award/pay calculations — E2 surfaces) | `opus` | Same judgement bar as E2 design work |
| Routed to a command (`/autofeature:*`) | the command's own tiers — pass through any `model:` override | Every command already reads model-tiers.md |
| Routed to a Skill (`patterns`, `copy`, `seo`, …) run in-context | **session model** | Skills can't take `model:`. Say so on the card when the session is Opus and the work is sonnet-tier, so the user can `/model sonnet` first |
| A survey that fails its gate (no file citations, wrong repo, edited files) | failure escalation per model-tiers.md: retry on sonnet with the failure, then opus, then stop | |

A request-level override (`model: economy|balanced|quality|opus|sonnet`) wins over this table, exactly
as in model-tiers.md.

---

## Step 5: Route Card, confirm, dispatch

Emit before dispatching:

```markdown
## Route
**Intent:** investigate (high)
**Target:** ~/dev/ritchies-mobile — swift · surface: styles
**Route:** Specialist survey — agents/swift-architect.md, read-only
**Model:** sonnet — single-stack survey (base tier)
**Writes:** none — report only
**Next:** offered after the report (e.g. a `/autofeature` run to converge on the tokens)
```

**Ask** (one AskUserQuestion, all open points batched) when any of:
- intent confidence is `low`;
- the target is ambiguous (Step 2.4);
- the route **writes** and the request did not clearly ask for a change — e.g. it came in as a
  question but matched `build`.

Otherwise dispatch without asking. A read-only route with high confidence and a single target should
never stop for confirmation — the survey is cheaper than the question.

**Dispatch:**
- Command route → read `$AUTOFEATURE_HOME/.claude/commands/<name>.md` and follow it with the
  remaining request as its arguments (the same way `review.md` dispatches). Don't reimplement it.
- `build`/`fix` via `/autofeature` → continue at `autofeature.md` Step 1 with `FEATURE_REQUEST`
  unchanged.
- Specialist survey → spawn per 3a with `model:` from Step 4. Present the agent's report as returned,
  after checking it cited files and edited nothing (`git -C "$TARGET_REPO" status --porcelain` is
  unchanged from before the spawn). Save it to `$TARGET_REPO/.autofeature/surveys/[slug]-[date].md`
  only if `.autofeature/` already exists there; don't create the directory in someone's app repo for a
  question.

**After a report**, offer the suggested next steps as options (each a ready-to-run prompt). Picking
one re-enters this router with that prompt, so it's triaged like any other request.
