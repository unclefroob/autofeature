---
status: CUSTOM
description: Deploy verification step for autofeature fullrun. Uses Railway and Netlify MCPs to confirm preview deploys succeed, run E2E smoke tests against real URLs, and inject preview links into the PR body.
---

# Feature Deploy Verify

Post-ship verification that the feature is correctly deployed to preview/staging environments.
Reads `HAS_NETLIFY`, `NETLIFY_SITE_ID`, `HAS_RAILWAY`, `RAILWAY_PROJECT_ID` from the Feature Brief.

**When invoked:** After the PR is created (fullrun Step 10). Branch has been pushed.
**Outcome:** Preview URLs injected into PR body, E2E smoke run against real deployment.
**Hard stops:** Deploy failure, deploy crash loop, E2E smoke failure that is not an environment config issue.

---

## Pre-flight: MCP Availability Check

Before proceeding, confirm the required MCPs are reachable:

```bash
# Check Railway MCP is connected (will fail gracefully if not)
# Check Netlify MCP is connected (will fail gracefully if not)
```

If an MCP is not available and the corresponding platform was detected:
> ⚠️ [Railway/Netlify] MCP is not connected. Skipping [Railway/Netlify] deploy verification.
> To enable: run `claude mcp list` to check MCP status.

Degrade gracefully — skip the unavailable platform, continue with what's available.

---

## Step 1: Netlify Deploy Verification (if HAS_NETLIFY=true)

### 1a. Locate the branch deploy

When the feature branch was pushed in Step 10, Netlify automatically triggered a branch deploy.

Use the Netlify MCP to find the most recent deploy for this branch on `NETLIFY_SITE_ID`:
- List deploys for the site
- Filter to the feature branch name (from Step 6)
- Take the most recent entry

If no deploy found: wait 30 seconds and retry up to 5 times (Netlify webhook may not have fired yet).
If still not found after retries: warn and skip — branch deploys may be disabled for this site.

### 1b. Poll until the deploy settles

Poll every 30 seconds, max 10 minutes:

| Status | Action |
|--------|--------|
| `building` / `processing` / `enqueued` | Continue polling |
| `ready` | ✓ Extract `deploy_ssl_url` as `NETLIFY_PREVIEW_URL` |
| `error` / `failed` | ✗ **HARD STOP** — read `error_message`, present to user |
| `cancelled` | Warn + skip |

On timeout (10 min elapsed), ask:
> Netlify deploy is still in progress after 10 minutes.
> A) Wait another 5 minutes
> B) Skip Netlify verification and include partial results in PR
> C) Stop entirely

### 1c. Output

```
Netlify: [NETLIFY_SITE_NAME] → [NETLIFY_PREVIEW_URL] ✓
```

---

## Step 2: Railway Deploy Verification (if HAS_RAILWAY=true)

### 2a. Read service logs

Use Railway MCP `get-logs` for `RAILWAY_SERVICE_NAME` in `RAILWAY_PROJECT_ID`.
Request the last 80 lines, focusing on entries since the push timestamp.

### 2b. Assess health

Scan logs for signals:

**Healthy signals:**
- `Server listening on port`, `listening on`, `ready`, `started`
- `Connected to MongoDB`, `database connected`
- HTTP 200/201 responses in access logs

**Crash signals:**
- `Error:` at start of line, `UnhandledPromiseRejection`
- `ECONNREFUSED`, `EADDRINUSE`, `Cannot find module`
- Process exit with non-zero code
- Container restart / crash loop

| Observation | Assessment | Action |
|-------------|-----------|--------|
| Healthy signal present, no crashes | ✓ Healthy | Continue |
| No logs yet (cold start) | Wait 30s, retry once | |
| Crash signal found after push timestamp | ✗ **HARD STOP** | |
| Ambiguous (no clear signal) | Warn + continue | |

On crash: extract the error lines (first 5), present to user. Optionally invoke `feature-investigate.md` with the Railway log as context.

### 2c. Output

```
Railway: [RAILWAY_SERVICE_NAME] → healthy ✓
Railway: [RAILWAY_SERVICE_NAME] → ✗ [error summary]
```

---

## Step 3: E2E Smoke Test Against Netlify Preview (if HAS_E2E=true AND NETLIFY_PREVIEW_URL obtained)

Run the golden-path Playwright test against the real preview URL — not localhost.

```
Read $AUTOFEATURE_HOME/agents/test-runner.md
Agent({
  description: "E2E smoke test against Netlify preview",
  subagent_type: "general-purpose",
  model: "haiku",   // runs test-runner (mechanical) — orchestrator/model-tiers.md
  prompt: "[test-runner.md content]

  Job: Run the Playwright golden-path test against the Netlify preview URL.
  Override the Playwright baseURL to: [NETLIFY_PREVIEW_URL]
  Run only the golden path test file written during implementation (Step 7c).
  Mode: e2e.

  This is a real deployed environment — not localhost. Expect higher latency.
  Use waitForLoadState('networkidle') and generous timeouts (10s+ for navigation).
  Return: pass/fail, any errors (first 3 lines max), duration."
})
```

**On failure, classify the cause:**

1. **Environment config issue** (auth redirect, missing env var in Netlify, CORS from preview URL): warn, note "smoke test not applicable to this preview environment without additional config", set `E2E_SMOKE_RESULT=skipped (env config)`, continue.

2. **Real regression** (component broken, API error, JS exception): **HARD STOP** — this feature works locally but fails in the real deployment. Invoke `feature-investigate.md` with the Playwright error as context.

3. **Flake** (timeout on first run): retry once. If still failing, treat as environment config issue.

---

## Step 4: Update PR Body

After verification completes, update the PR to include preview URLs:

```bash
PR_NUM=$(gh pr view --json number -q .number)
CURRENT_BODY=$(gh pr view --json body -q .body)

PREVIEW_BLOCK="## Preview
- Frontend: ${NETLIFY_PREVIEW_URL:-n/a}
- API staging: ${RAILWAY_STAGING_URL:-n/a}
- E2E smoke: ${E2E_SMOKE_RESULT:-n/a}"

gh pr edit "$PR_NUM" --body "${CURRENT_BODY}

${PREVIEW_BLOCK}"
```

If `gh pr edit` fails: print the preview block and instruct the user to add it manually.

---

## Cross-Repo Coordination

For cross-repo runs (multiple repos in `.autofeature/coordination.md`):

Run verification in ship order:
1. **API repo first** → Railway verification (backend must be healthy before frontend smoke)
2. **Web repos** → Netlify verification + E2E smoke test against the now-verified API
3. Record each repo's result in `.autofeature/coordination.md`

Do NOT proceed to the next repo's verification if a previous one hits a hard stop.

---

## Output Block

```
=== Deploy Verification ===
Netlify:   [NETLIFY_SITE_NAME] → [NETLIFY_PREVIEW_URL] ✓ / ✗ [error] / n/a
Railway:   [RAILWAY_SERVICE_NAME] → healthy ✓ / ✗ [error] / n/a
E2E smoke: pass ✓ / fail ✗ / skipped (env config) / n/a
PR updated: yes / no
```
