---
name: fullrun
description: |
  Build an entire feature end-to-end including deploy verification against real infrastructure.
  Extends the standard autofeature pipeline with Railway and Netlify MCP integration.
  After ship: waits for preview deploys to complete, runs E2E smoke tests against real preview URLs, injects preview links into the PR body.
  Pipeline: interrogate → scope-gate → plan → branch → implement (parallel specialists) → test → review → ship → deploy-verify.
  Requires Railway MCP and/or Netlify MCP connected in Claude Code settings.
  Invoke as: /autofeature:fullrun [mode:] <feature description>
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
  - Agent
  - Skill
  - TaskCreate
  - TaskUpdate
  - Monitor
  - mcp__trello__get_card_details
  - mcp__trello__get_card_checklists
  - mcp__trello__add_comment_to_card
  - mcp__railway__check_railway_status
  - mcp__railway__list_projects
  - mcp__railway__list_services
  - mcp__railway__get_logs
  - mcp__railway__deploy
  - mcp__railway__list_variables
  - mcp__netlify__list_sites
  - mcp__netlify__get_site
  - mcp__netlify__list_site_deploys
  - mcp__netlify__get_site_deploy
  - mcp__netlify__update_site
---

# AutoFeature FullRun — With Deploy Verification

## What this command does differently from `/autofeature`

Runs the **identical pipeline** as `autofeature.md` with three additions:

1. **Step 2 Extension** — detect Netlify site and Railway service linked to this repo
2. **Step 10.5** — wait for preview deploys, run E2E smoke against real URLs, update PR with preview links
3. **Task 10** added to the pipeline task list

**Base pipeline:** Read and execute ALL steps from `$AUTOFEATURE_HOME/.claude/commands/autofeature.md`. Then apply the additions below at the indicated injection points.

---

## $AUTOFEATURE_HOME

Same resolution as autofeature:

```bash
AUTOFEATURE_HOME="${AUTOFEATURE_HOME:-$HOME/dev/autofeature}"
```

---

## Pipeline

```
resume? → [trello?] → interrogate → scope-gate → plan → branch → cross-repo-coord →
  implement (parallel specialists) → verify (test-runner) → review (parallel + skills) → ship →
  [NEW] deploy-verify → cleanup checkpoint
```

---

## Injection 1: Step 2 Extension — Platform Detection

After the standard Step 2 context gathering completes (after `HAS_E2E` is recorded in the Feature Brief), also run:

### Netlify Detection

```bash
# Check for Netlify config files
cat netlify.toml 2>/dev/null
ls _redirects public/_redirects 2>/dev/null
cat package.json 2>/dev/null | grep -i netlify
```

Then use the Netlify MCP to match this repo to a Netlify site:
- List all Netlify sites via MCP
- Match by `build_settings.repo_url` against `git remote get-url origin`
- If match found: record `HAS_NETLIFY=true`, `NETLIFY_SITE_ID`, `NETLIFY_SITE_NAME`

### Railway Detection

```bash
# Check for Railway config files
ls railway.json railway.toml .railway/ 2>/dev/null
cat railway.json 2>/dev/null
```

Then use the Railway MCP to match this repo to a service:
- Call `mcp__railway__list_projects` to list available Railway projects
- Match by project name, service name, or connected repo against the current repo name
- If match found: record `HAS_RAILWAY=true`, `RAILWAY_PROJECT_ID`, `RAILWAY_SERVICE_NAME`

### Record in Feature Brief

Append to the `## Context` section:

```
HAS_NETLIFY = true | false
NETLIFY_SITE_ID = [id or "n/a"]
NETLIFY_SITE_NAME = [name or "n/a"]
HAS_RAILWAY = true | false
RAILWAY_PROJECT_ID = [id or "n/a"]
RAILWAY_SERVICE_NAME = [name or "n/a"]
```

**If neither Netlify nor Railway detected:**
> ⚠️ No Netlify site or Railway service matched this repo. Step 10.5 will be skipped — this run will behave identically to standard `/autofeature`.
Continue normally.

---

## Injection 2: Task List Addition (Step 1)

When creating the pipeline tasks via TaskCreate, add one additional task:

```
Task 10: Deploy Verification → "Verifying preview deploys (Netlify + Railway)"
```

Set `addBlockedBy: [Task 9]` so it blocks on Ship.

---

## Injection 3: PR Body Addition (Step 10)

When constructing the PR body in Step 10, append a `## Preview` section as a placeholder:

```markdown
## Preview
- Frontend: deploying…
- API staging: n/a
- E2E smoke: pending
```

This will be updated with real URLs after Step 10.5 completes.

---

## Step 10.5: Deploy Verification

**Runs after:** Step 10 (Ship — PR created, branch pushed)
**Runs before:** Step 11 (Cleanup Checkpoint)

Read and follow `$AUTOFEATURE_HOME/adapted/feature-deploy-verify.md`.

Pass it the following context:
- `HAS_NETLIFY`, `NETLIFY_SITE_ID`, `NETLIFY_SITE_NAME`
- `HAS_RAILWAY`, `RAILWAY_PROJECT_ID`, `RAILWAY_SERVICE_NAME`
- `HAS_E2E`, `E2E_TOOL`
- Current branch name
- PR number (from Step 10 output)
- `AUTOFEATURE_HOME` path (for test-runner.md reference)

---

## Final Output Addition

Append to the standard `=== AutoFeature Complete ===` block:

```
Deploy verification:
  Netlify:   [NETLIFY_SITE_NAME] → [NETLIFY_PREVIEW_URL] ✓ / ✗ / n/a
  Railway:   [RAILWAY_SERVICE_NAME] → healthy ✓ / ✗ [error] / n/a
  E2E smoke: pass ✓ / fail ✗ / skipped / n/a
```

---

## File Reference

Inherits all file references from `autofeature.md`, plus:

### New Methodology (`$AUTOFEATURE_HOME/adapted/`)

| File | Purpose |
|------|---------|
| `feature-deploy-verify.md` | Deploy verification — Netlify polling, Railway log check, E2E smoke against preview URL, PR body update |
