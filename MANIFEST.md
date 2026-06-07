# AutoFeature — File Manifest

Tracks the status of every file: original gstack extracts vs. adapted/custom files.

---

## Legend

| Status | Meaning |
|--------|---------|
| `ORIGINAL` | Direct extract from gstack. Content is unchanged from source. |
| `ADAPTED` | Based on a gstack source, but modified. See `changes:` frontmatter in the file. |
| `CUSTOM` | New file — no gstack source. Written from scratch. |

---

## Source Files (`source/`)

Original extracts from gstack. **Do not edit these** — they are the reference baseline. To change methodology, edit the corresponding `adapted/` file instead. When gstack releases updates, diff the new gstack version against these files to decide what to pull in.

| File | gstack Source | Status | Notes |
|------|--------------|--------|-------|
| `source/ethos.md` | `gstack/ETHOS.md` | `ORIGINAL` | Full extract. Core builder philosophy. |
| `source/office-hours.md` | `gstack/office-hours/SKILL.md.tmpl` | `ORIGINAL` | Partial — builder mode only. Startup diagnostic omitted. |
| `source/plan-eng-review.md` | `gstack/plan-eng-review/SKILL.md.tmpl` | `ORIGINAL` | Full extract. `{{PLACEHOLDERS}}` kept as markers. |
| `source/review.md` | `gstack/review/SKILL.md.tmpl` | `ORIGINAL` | Full extract. `{{PLACEHOLDERS}}` kept as markers. |
| `source/ship.md` | `gstack/ship/SKILL.md.tmpl` | `ORIGINAL` | Full extract. `{{PLACEHOLDERS}}` kept as markers. |
| `source/autoplan-principles.md` | `gstack/autoplan/SKILL.md.tmpl` | `ORIGINAL` | Partial — 6 decision principles and classification only. |
| `source/investigate.md` | `gstack/investigate/SKILL.md.tmpl` | `ORIGINAL` | Full extract. gstack freeze hook reference preserved. |
| `source/review-checklist.md` | `gstack/review/checklist.md` | `ORIGINAL` | Full extract. Unmodified. |
| `source/review-design-checklist.md` | `gstack/review/design-checklist.md` | `ORIGINAL` | Full extract. Unmodified. |
| `source/review-specialists-security.md` | `gstack/review/specialists/security.md` | `ORIGINAL` | Full extract. Unmodified. |
| `source/review-specialists-testing.md` | `gstack/review/specialists/testing.md` | `ORIGINAL` | Full extract. Unmodified. |

---

## Adapted Files (`adapted/`)

Working files used by the skill. Edit these freely — changes take effect immediately.

| File | Adapted From | Status | Key Changes |
|------|-------------|--------|-------------|
| `adapted/feature-interrogation.md` | `source/office-hours.md` | `ADAPTED` | Builder mode only; no gstack bins; added stack detection; simplified design doc path |
| `adapted/feature-plan.md` | `source/plan-eng-review.md` | `ADAPTED` | No gstack bins; unit tests only; Node/React/RN/Swift/Kotlin checks; **added error mapping, shadow path testing, observability checklist** |
| `adapted/feature-investigate.md` | `source/investigate.md` | `ADAPTED` | No gstack freeze mechanism; added stack-specific patterns and debug commands; kept Iron Law + 5 phases |
| `adapted/feature-review.md` | `source/review.md` | `ADAPTED` | No Greptile; no adversarial step; no gstack bins; references feature-review-checklist.md and feature-design-check.md |
| `adapted/feature-review-checklist.md` | `source/review-checklist.md` + `source/review-specialists-security.md` + `source/review-specialists-testing.md` | `ADAPTED` | Rails/Python refs replaced with Node/React/RN/Swift/Kotlin; specialists merged inline; Fix-First heuristic unchanged |
| `adapted/feature-design-check.md` | `source/review-design-checklist.md` | `ADAPTED` | No gstack-diff-scope; added React Native specific checks; added Swift/SwiftUI checks; added missing UI states category |
| `adapted/feature-ship.md` | `source/ship.md` | `ADAPTED` | No gstack bins; no eval suites; no gstack metrics; detects test command dynamically; simplified CHANGELOG handling |
| `adapted/feature-deploy-verify.md` | *(none)* | `CUSTOM` | Post-ship deploy verification — Netlify branch deploy polling, Railway log health check, E2E smoke against preview URL, PR body update with preview links |
| `adapted/feature-seo-audit.md` | *(none)* | `CUSTOM` | SEO audit methodology — codebase scan (rendering strategy, meta tags, sitemap, robots.txt, structured data, Netlify config, semantic HTML), live site check via WebFetch, findings report with severity scoring and React/Vite/Netlify fix reference |
| `adapted/feature-product-review.md` | *(none)* | `CUSTOM` | CEO/PM/flow-walker product review — Workflow script (map product surface → 3 lenses in parallel → adversarially verify gap/flow claims against code → synthesize), severity rubric, report format, and report-then-offer-to-fix hand-off. Runs pre-build (Step 4.5) and standalone |
| `adapted/feature-advice.md` | *(none)* | `CUSTOM` | Scoped single-feature review — lean Workflow script (scan only the relevant code → product advisor ∥ build advisor → synthesize one opinionated recommendation). Opinion + build plan for the feature under discussion, NOT a whole-product audit |
| `adapted/market-review.md` | *(none)* | `CUSTOM` | Market & fundability review — Workflow script (frame → market ∥ gap ∥ VC analysts with cited live web research → adversarial bear-case stress test → managing-partner synthesis). Produces an investment memo: usefulness + TAM/SAM/SOM, competitive gap & moat, fundability verdict (stage/check/valuation/milestones), risks, next steps, investor one-pager, sources appendix |
| `adapted/feature-test.md` | *(none)* | `CUSTOM` | Live acceptance-testing methodology — derives its own test plan by reviewing what was built, then DRIVES the running app (web via Chrome MCP; iOS/Android via simulator/emulator + computer-use), logs in with given credentials, walks flows, reports per-flow PASS/FAIL/BLOCKED with screenshots + console/network errors, hands failures to `/autofeature`. Credential-redaction + evidence-discipline rules. Complements `agents/test-runner.md` (headless suites) |
| `adapted/feature-test-manifest.md` | *(none)* | `CUSTOM` | Test Manifest format — the re-runnable acceptance spec (header + setup + surfaces built + acceptance flows AF-N + out-of-scope) emitted at autofeature ship **Step 10.5**; the plan spine `/autofeature:test` consumes when present, or generates on-demand. Shared single source of truth for producer (ship) + consumer (test) |

---

## Skill Files (`.claude/commands/`)

The Claude Code custom command. No gstack source — written from scratch to orchestrate all adapted files.

| File | Status | Description |
|------|--------|-------------|
| `.claude/commands/autofeature.md` | `CUSTOM` | Main orchestration. Reads adapted files + agents/ + orchestrator/ at runtime. Supports automated + checkpoint modes. Pipeline: interrogate → scope-gate → product-review (pre-build) → plan → branch → implement (parallel) → test → review → ship → emit test manifest (Step 10.5). |
| `.claude/commands/fullrun.md` | `CUSTOM` | Extends autofeature with Railway + Netlify MCP deploy verification. Adds platform detection (Step 2), deploy polling + E2E smoke against real preview URLs (Step 10.5), and PR body preview URL injection. Does not modify autofeature.md. |
| `.claude/commands/seo.md` | `CUSTOM` | SEO audit + fix command. Audit mode: scans codebase + live site via WebFetch + Netlify MCP, scores against React/Vite/Netlify checklist, produces prioritised findings. Fix mode: implements improvements via autofeature pipeline using seo-architect specialist. Supports Trello card creation for audit findings. |
| `.claude/commands/product-review.md` | `CUSTOM` | Standalone CEO/PM/flow-walker product review. Runs the product-review Workflow (map → 3 lenses → verify → synthesize) over the whole product (`audit`), a proposed feature (`feature:`), or jumps straight to a fix (`fix:`). Reports prioritized gaps & broken flows, then offers to spin top fixes into autofeature runs or Trello cards. |
| `.claude/commands/feature-review.md` | `CUSTOM` | Standalone scoped review of the ONE feature under discussion. Captures the feature from conversation context, runs the lean feature-advice Workflow (scan → product advisor ∥ build advisor → synthesize), and returns an opinion + ordered build plan ending in a ready-to-run /autofeature prompt. Advice only — never branches or ships. |
| `.claude/commands/market-review.md` | `CUSTOM` | Standalone market & fundability review. Captures the product/idea from context (reads the repo README to ground it), runs the market-review Workflow (frame → market ∥ gap ∥ VC → bear case → **citation-verify** → synthesize) with live cited web research, and produces an investment memo answering is-it-useful / market-gap / can-I-get-funding. Saves to `.autofeature/market-review-[date].md`. Advisory only. |
| `.claude/commands/review.md` | `CUSTOM` | Unified entry for the review lenses — `/autofeature:review [product\|feature\|market\|scope] [args]` dispatches to the matching standalone command/methodology (no reimplementation). Single discoverable surface; the four standalones still work. |
| `.claude/commands/test.md` | `CUSTOM` | Standalone **live acceptance tester** — drives the running app (web via Chrome MCP; iOS sim / Android emulator via computer-use) against a URL + credentials, deriving its own test plan by reviewing what was built (newest Test Manifest + code + live surface). Reports per-flow PASS/FAIL/BLOCKED; offers to hand failures to `/autofeature [skip-product-review] fix:`. Reads `adapted/feature-test.md` + `adapted/feature-test-manifest.md`. Distinct from `agents/test-runner.md` (headless unit/e2e suites). |

---

## Specialist Agents (`agents/`)

Self-contained subagent prompts. Spawned via the `Agent` tool (subagent_type=`general-purpose`) by reading the file and passing its content inline. No install step — files stay portable.

| File | Status | Description |
|------|--------|-------------|
| `agents/express-mongo-architect.md` | `CUSTOM` | Backend specialist: Express routes/controllers/middleware, Mongoose models, validation, integration tests |
| `agents/react-architect.md` | `CUSTOM` | Web frontend specialist: components, hooks, routing, forms, React Query, RTL tests |
| `agents/react-native-architect.md` | `CUSTOM` | Mobile specialist: screens, navigation, Platform.OS, permissions, native modules, lists |
| `agents/mongo-data-modeler.md` | `CUSTOM` | Schema design, index strategy, query plan review, migration plan |
| `agents/api-contract-broker.md` | `CUSTOM` | Cross-repo contract reconciliation when backend + frontend(s) are touched in one run |
| `agents/seo-architect.md` | `CUSTOM` | SEO specialist — implements prerendering (vite-plugin-prerender/vite-ssg), react-helmet-async per-route meta tags, sitemap, robots.txt, Netlify canonical redirects/_headers, JSON-LD structured data; flags OG image, copy, and dynamic route decisions to user |
| `agents/product-strategist.md` | `CUSTOM` | Product reviewer personas — CEO lens (value/differentiation/coherence/opportunity cost), PM lens (job-to-be-done, missing capabilities, edge/empty states), flow-walker lens (traces journeys through the code for dead-ends/broken round-trips). Severity rubric + structured output contract; operational prompts live in feature-product-review.md |
| `agents/market-analyst.md` | `CUSTOM` | Market-review persona — usefulness & demand (painkiller-vs-vitamin, demand signals) and market sizing (bottom-up + top-down TAM/SAM/SOM) with cited web research and confidence flags |
| `agents/market-gap-analyst.md` | `CUSTOM` | Market-review persona — competitive landscape (direct/substitute/adjacent), white space, the wedge, moat & copyability, why-now; sourced. Flags "no competitors" and feature-not-a-company traps |
| `agents/vc-analyst.md` | `CUSTOM` | Market-review persona — fundability: venture-scale test, business model/unit economics sketch, stage traction bar, sourced comparable raises, partner objections, verdict (stage/check/valuation/milestones, or alt funding if not VC) |
| `agents/bear-case-analyst.md` | `CUSTOM` | Market-review persona — adversarial skeptic. Builds the case to pass, stress-tests the other analysts' claims (inflated TAMs, graveyards, illusory moats, cherry-picked comps), names kill-shots, writes an 18-month pre-mortem. The quality gate against happy-path market analysis |
| `agents/test-runner.md` | `CUSTOM` | Test execution proxy — keeps multi-MB test logs out of orchestrator context, returns <2KB summary |
| `agents/README.md` | `CUSTOM` | Roster, invocation pattern, parallel fan-out usage |

---

## Orchestrator Helpers (`orchestrator/`)

Decision logic the orchestrator reads at runtime. Edit to change scope/coordination/skill behavior without touching the main command file.

| File | Status | Description |
|------|--------|-------------|
| `orchestrator/scope-gate.md` | `CUSTOM` | Classifies feature into micro / single-layer / cross-stack / cross-repo before fan-out |
| `orchestrator/cross-repo-detect.md` | `CUSTOM` | Finds sibling `*-api`/`*-mobile`/`*-desktop`/`*-cms`/`*-website` repos under `~/dev/` |
| `orchestrator/skill-wiring.md` | `CUSTOM` | When to invoke Plan / Explore subagents and security-review / simplify / frontend-design skills |
| `orchestrator/trello-scope.md` | `CUSTOM` | Trello card fetch (via MCP), technical scope generation, and comment posting. Invoked when a `trello.com/c/` URL is detected in the feature request. |
| `orchestrator/model-tiers.md` | `CUSTOM` | Model-tier (cost/quality) policy — pins each spawned agent / Workflow phase to haiku/sonnet/opus so the fan-out doesn't all run on the session model. Active profile BALANCED; the single editable knob (incl. economy/quality profiles + per-run `model:` override). Referenced by every command that fans out. |

---

## How to Make Changes

**To tweak a methodology (e.g., change how planning works):**
→ Edit the relevant `adapted/` file

**To change the orchestration pipeline:**
→ Edit `.claude/commands/autofeature.md`

**To change a specialist's behavior (e.g., how the backend architect designs routes):**
→ Edit the relevant `agents/*.md` file

**To change scope classification or cross-repo detection rules:**
→ Edit the relevant `orchestrator/*.md` file

**To add a new specialist agent:**
1. Create `agents/<name>.md` following the format of existing agents
2. Add it to `agents/README.md` roster
3. Wire it into `.claude/commands/autofeature.md` at the appropriate phase
4. Add a row to this MANIFEST

**To add a new tech stack:**
→ Add stack-specific sections to each `adapted/` file that currently has stack sections; create a new `agents/<stack>-architect.md` if a specialist is warranted

**To sync with upstream gstack changes:**
1. Re-extract the relevant `source/` file from the new gstack version
2. Diff it against the current `source/` file
3. Apply relevant changes to the corresponding `adapted/` file manually
4. Update the `extracted:` date in the source file's frontmatter
5. Update this MANIFEST if anything changed structurally

---

## What Was Removed from gstack Sources

Intentionally excluded from adapted files:

| Feature | Reason |
|---------|--------|
| gstack bin references (`gstack-slug`, `gstack-review-log`, etc.) | Requires gstack installed |
| `{{PLACEHOLDERS}}` (PREAMBLE, BROWSE_SETUP, etc.) | Requires gstack template compiler |
| Greptile PR review integration | External service dependency |
| Eval suites (LLM-as-judge) | gstack-specific test infrastructure |
| Review dashboard | Requires gstack state files |
| Adversarial review (specialist subagents) | gstack multi-agent infrastructure |
| Learnings search/log (`~/.gstack/` state) | gstack state directory |
| Document-release auto-invoke | Separate gstack skill dependency |
| Worktree parallelization | gstack parallel agent infrastructure |
| VERSION file management (4-digit format) | gstack versioning convention |
| gstack metrics logging | gstack analytics |
| Startup diagnostic mode (office-hours) | Not needed for feature building |
| Codex outside voice | Requires OpenAI Codex CLI |
| Design review phases (`/plan-design-review`) | Out of scope |
| Browser QA (`/qa`) | Unit tests only |
| Plan-CEO review (scope modes, 11 sections) | Too heavy for this use case — error mapping and shadow paths extracted instead |
| gstack freeze/hook mechanism | Requires gstack hooks in settings.json |
| Retro / metrics tracking | Separate concern |
| Document release | Separate concern |

---

## What Was Added Beyond gstack Sources

Additions not in any gstack source:

| Addition | File | Purpose |
|----------|------|---------|
| Railway + Netlify deploy verification | `adapted/feature-deploy-verify.md` + `.claude/commands/fullrun.md` | Post-ship preview deploy polling, Railway log health check, E2E smoke against real Netlify preview URLs, PR body enrichment with preview links |
| CEO/PM/flow-walker product review (pre-build) | `adapted/feature-product-review.md` + `agents/product-strategist.md` + `.claude/commands/product-review.md` | Multi-agent **Workflow** that maps the product, fans out three product lenses, adversarially verifies broken-flow/gap claims against the code, and synthesizes a prioritized report — finds product gaps before building. Runs pre-build in /autofeature (Step 4.5) and standalone via /autofeature:product-review |
| Scoped single-feature review (opinion + build advice) | `adapted/feature-advice.md` + `.claude/commands/feature-review.md` | Lean **Workflow** (scan → product advisor ∥ build advisor → synthesize) that reviews only the feature under discussion and returns an opinionated recommendation + build plan. Standalone via /autofeature:feature-review — the "should we / how?" step before /autofeature builds it |
| Market & fundability review | `adapted/market-review.md` + `agents/market-analyst.md` + `agents/market-gap-analyst.md` + `agents/vc-analyst.md` + `agents/bear-case-analyst.md` + `.claude/commands/market-review.md` | **Workflow** with live cited web research + adversarial bear case → managing-partner investment memo. Answers is-it-useful / market-gap / can-I-get-funding. Standalone via /autofeature:market-review — the business/market lens before product & build |
| Citation-verify hardening (market-review) | `adapted/market-review.md` + `.claude/commands/market-review.md` | A **Verify phase** (Phase 3.5) re-fetches each cited URL with WebFetch and classifies it confirmed/unconfirmed/dead (+stale); the memo downgrades + tags `⚠ unverified` figures and shows a `Sources cited: N · X✓/Y⚠/Z✗` trust line. Closes the hallucinated-comp risk that sourcing + the bear pass alone didn't. Funding-comps-first, cap via `maxVerify` (default 6), skipped offline |
| Fail-fast on empty subject (feature-review) | `adapted/feature-advice.md` + `.claude/commands/feature-review.md` | Command STOPs and asks when no feature is resolved; the Workflow backstops with an early return that also catches the literal truthy string `'undefined'` and defaults `repo` to cwd — so the lens never spends agents emitting generic advice dressed as scoped advice |
| Unified review surface | `.claude/commands/review.md` | `/autofeature:review [product\|feature\|market\|scope]` routes to the existing lenses — one discoverable name instead of four co-equal commands (addresses the product-review "command sprawl" finding); standalones retained as aliases |
| Plugin-root path resolution | all `.claude/commands/*.md` `$AUTOFEATURE_HOME` blocks | Commands resolve methodology/agent files from `$AUTOFEATURE_HOME` → `$CLAUDE_PLUGIN_ROOT` → `$HOME/dev/autofeature` (first that contains `adapted/`). The whole repo ships as the plugin (marketplace `source: "./"`), so an installed plugin finds its bundled files; verify with a real `/plugin install` |
| Checkpoint/resume capability | `.claude/commands/autofeature.md` | Resume interrupted feature builds |
| React Native specific design checks | `adapted/feature-design-check.md` | RN has different design concerns than web |
| Swift/SwiftUI design checks | `adapted/feature-design-check.md` | iOS-specific accessibility + layout patterns |
| Missing UI states category | `adapted/feature-design-check.md` | Loading/empty/error states are common omission |
| Stack-specific debug commands | `adapted/feature-investigate.md` | Cache clearing, simulator reset, Metro bundler |
| Methodology reference table | `.claude/commands/autofeature.md` | Single place to see which file does what |
| Stack-specialist agent fleet | `agents/*.md` | Parallel design + implementation across backend/web/mobile |
| Scope gate (micro/single/cross-stack/cross-repo tiers) | `orchestrator/scope-gate.md` | Avoid spawning 5 agents for a 1-file change |
| Cross-repo coordination | `orchestrator/cross-repo-detect.md` | Detect sibling `*-api`/`*-mobile` repos and ship coordinated PRs |
| Test-runner subagent | `agents/test-runner.md` | Keep multi-MB test logs out of orchestrator context |
| Plan + Explore subagent delegation | `orchestrator/skill-wiring.md` | Outsource planning and context-gather to built-in subagent_types |
| security-review + simplify skill wiring | `orchestrator/skill-wiring.md` | Reuse existing skills inside the pipeline |
| frontend-design skill wiring | `orchestrator/skill-wiring.md` | Polished UI for new components, not generic AI aesthetics |
| Live acceptance testing (drive the real app) | `.claude/commands/test.md` + `adapted/feature-test.md` | `/autofeature:test` — opens the running app (web via Chrome MCP; iOS sim / Android emulator via computer-use), logs in with a URL + credentials, **derives its own test plan by reviewing what was built**, walks the whole product or a subset of flows, and reports per-flow PASS/FAIL/BLOCKED with screenshots + console/network errors. Complements (does not replace) the headless `test-runner` |
| "What was built" Test Manifest | `adapted/feature-test-manifest.md` + `autofeature.md` Step 10.5 | autofeature documents exactly what each run built as a structured, re-runnable acceptance spec at `.autofeature/tests/[slug]-[date].md` — the precise hand-off from building to testing, consumed by `/autofeature:test` |
| Model-tier cost control | `orchestrator/model-tiers.md` + `model:` pins at every spawn site (`autofeature.md`, `feature-advice.md`, `feature-product-review.md`, `market-review.md`, `scope.md`, `product-review.md`, `seo.md`, `feature-test.md`, `feature-deploy-verify.md`, `skill-wiring.md`) | Stops the whole fan-out inheriting an expensive session model. Each `Agent()`/Workflow `agent()` is pinned to the cheapest capable tier (BALANCED: Sonnet workhorse, Haiku for mechanical test-runner/scans/maps, Opus only for the bear-case gate, market memo, and cross-repo planning). One editable policy file; per-run `model:` override |
