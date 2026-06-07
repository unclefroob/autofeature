# Product Review — AutoFeature plugin (2026-06-06)

Run: `/autofeature:product-review what you just built` · Workflow `wf_599c08fa-d98` · 12 agents
Mode: **product** (whole-plugin audit) · Mapped: 25 surfaces, 7 journeys · Verified: high-sev claims checked against code, 1 refuted

## Bottom line

AutoFeature's advisory and specialist content is strong, but the core "build a feature from one
prompt" loop is **broken at step 0** for anyone who isn't the author on the author's machine: both
documented install paths leave the flagship command hard-aborting because it reads ~18 files out of
a hardcoded `$AUTOFEATURE_HOME` (`$HOME/dev/autofeature`) that the package never ships. Fix
install/packaging first — nothing else matters until a new user can run `/autofeature`. Two more
confirmed flow-breaks follow: the Trello scoping journey calls MCP tools the command never permits
(empty name intersection), and the primary repo gets no dirty-tree guard before branch creation
while siblings do. Separately, the product advertises 5 stacks but only 3 are buildable end-to-end.

Counts: 🔴 2 critical · 🟡 3 high · 🟢 6 medium · ⚪ 1 low (14 confirmed, 1 refuted)

---

## 🔴 CRITICAL

1. **Install path is broken — flagship command hard-aborts on every new install.** ✓verified
   `$AUTOFEATURE_HOME` defaults to `$HOME/dev/autofeature` and the command reads ~18 files from it,
   but the plugin only ships `commands/` and the README install is `cp` one file.
   *Where:* `plugin.json`; `README.md:7,13-23`; `autofeature.md:46,55` + ~18 runtime reads.
   *Fix:* ship `adapted/agents/orchestrator/source` under the plugin and resolve relative to the
   plugin root — OR change README to `git clone` + explicit `export AUTOFEATURE_HOME=…`.

2. **Trello scoping calls MCP tools the command never permits (empty name intersection).** ✓verified
   `trello-scope.md` calls `get_card` / `get_acceptance_criteria` / `add_comment`; allowed-tools
   permits `get_card_details` / `get_card_checklists` / `add_comment_to_card`. Zero overlap → the
   `trello.com/c/` entry point silently drops acceptance criteria or errors.
   *Where:* `trello-scope.md:9,41,61,156` vs `autofeature.md:24-26`.
   *Fix:* reconcile to the canonical `*_details` / `*_checklists` / `add_comment_to_card` names.

## 🟡 HIGH

3. **Swift build journey dead-ends — `swift-architect` (29KB) is wired into zero commands.** ✓verified
   Advertised in README/roster but no `STACK` value, no fan-out row, no scope-gate path.
   *Where:* `swift-architect.md` + `agents/README.md:12` + `README.md:7` vs `autofeature.md:198`,
   Step 5b/7b fan-out, `scope-gate.md`.
   *Fix:* wire it in (STACK enum + `*.xcodeproj`/`Package.swift` detection + fan-out rows + scope path) **or** cut it.

4. **Product claims 5 stacks (incl. Kotlin) but only 3 are buildable.** Swift unwired; no Kotlin agent exists.
   *Where:* `README.md:7`; skill description vs `agents/`.
   *Fix:* list only Node/React/React Native as supported; move Swift/Kotlin to a roadmap section.

5. **No dirty-working-tree guard on the primary repo before branch creation — only siblings are protected.** ✓verified
   `git checkout $BASE; git pull; git checkout -b` with no preceding `git status --porcelain`.
   *Where:* `autofeature.md` Step 6 (392-397) vs sibling guard `cross-repo-detect.md:86` + Emergency Stop #7.
   *Fix:* clean-tree preflight → AskUserQuestion (stash/commit/abort); promote to an Emergency Stop.

## 🟢 MEDIUM

6. **Seven overlapping commands diffuse the product identity.** *(about the new review suite)*
   The four advisory commands each open by redirecting to the others.
   *Where:* `feature-review.md:5-6,104-108`; `product-review.md:37-40`; `market-review.md:30-32`; `scope.md`.
   *Fix:* say what AutoFeature IS in one line; demote reviews to subordinate modes or a single
   `/autofeature:review [product|feature|market|scope]`. Prioritize the build path over more review lenses.

7. **No cost/time guardrail on a pipeline that fans out ~12 agents + nested workflows.** *(partly the new Step 4.5)*
   *Fix:* a cost/scope preview before an automated run, and/or a budget cap that downshifts fan-out.

8. **Resume offers SHIPPED and other-branch checkpoints as resumable** — prose says "this branch", code filters neither.
   *Where:* `autofeature.md` Step 0 (70-83), Step 11 (657-665).
   *Fix:* filter candidates by `branch:` == current and exclude `status: SHIPPED`; show status per entry.

9. **`claude-seo` live-crawl/schema fixes depend on a plugin no manifest declares.**
   *Where:* `seo.md:102-109,193-199` vs `.claude-plugin/*` (no dependency).
   *Fix:* declare the companion plugin + add an availability check that warns instead of silently degrading.

10. **Aborting after branch creation leaves an orphaned `feature/[slug]` branch with no teardown.**
    *Where:* abort options at `autofeature.md:213,298,383` all precede Step 6 (387).
    *Fix:* on abort/terminal stop at/after Step 6, detect the branch and offer cleanup.

11. **No way to see in-flight/past runs; `.autofeature/` artifacts accumulate with no list/clean.**
    *Fix:* a `/autofeature:status` command listing checkpoints (by branch+status), latest brief, reports, with archive/clear.

## ⚪ LOW

12. **No git-remote preflight** — a repo with no `origin` silently defaults base to `main` and only fails at push.
    *Fix:* warn at Step 0/6 if `git remote get-url origin` fails (build local-only or abort).

---

## Recommended next features (workflow output)

1. 🔴 Make the plugin self-contained (ship methodology dirs under the plugin; resolve from plugin root).
2. 🔴 Reconcile all Trello MCP tool names to the canonical set.
3. 🟡 Wire `swift-architect` into the pipeline (STACK + detection + fan-out + scope path).
4. 🟡 Primary-repo clean-tree preflight + Emergency Stop.
5. 🟡 Align stack advertising with reality (README + skill description).
6. 🟢 Add `/autofeature:status` (closes visibility + resume-filter + accumulation at once).

> ⚠ Findings grounded in code and adversarially verified (1 false positive dropped). Line numbers
> reflect the repo at review time. This is product review, not engineering/security review.
