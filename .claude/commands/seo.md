---
name: seo
description: |
  SEO audit and implementation for React/Vite/Netlify projects.
  Audit mode (default): scans codebase + live site, scores against a React/Vite/Netlify SEO checklist, produces prioritised findings.
  Fix mode: implements chosen SEO improvements through the autofeature pipeline using the SEO architect specialist.
  Stack-aware: React SPA prerendering, react-helmet-async, vite-plugin-sitemap, Netlify _redirects/_headers, JSON-LD structured data.
  Invoke as:
    /autofeature:seo                          → audit current directory
    /autofeature:seo url: https://example.com → audit + check live site
    /autofeature:seo fix: <description>       → implement specific SEO improvement
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
  - WebFetch
  - Agent
  - Skill
  - TaskCreate
  - TaskUpdate
  - mcp__netlify__list_sites
  - mcp__netlify__get_site
  - mcp__netlify__list_site_deploys
  - mcp__trello__add_card_to_list
  - mcp__trello__get_lists
---

# AutoFeature SEO — Audit and Fix

## $AUTOFEATURE_HOME

```bash
AUTOFEATURE_HOME="${AUTOFEATURE_HOME:-$HOME/dev/autofeature}"
```

---

## Step 1: Parse Arguments and Select Mode

Arguments = everything after `/autofeature:seo` in the user's message.

**Mode detection:**
- Contains `fix:` → **FIX MODE** — extract the fix description after `fix:` as `FIX_REQUEST`
- Contains `url:` → extract URL as `SITE_URL`, proceed to **AUDIT MODE**
- Empty or contains `audit` → **AUDIT MODE**

---

## Step 2: Project Detection

```bash
# Confirm this is a React/Vite project
cat package.json 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); deps={**d.get('dependencies',{}), **d.get('devDependencies',{})}; print('REACT' if 'react' in deps else 'NOT_REACT')"

# Get repo remote URL for Netlify site matching
git remote get-url origin 2>/dev/null
```

If not a React project: warn that this command is optimised for React/Vite/Netlify but will still attempt a generic audit.

**Netlify site detection:**
Use `mcp__netlify__list_sites` to match the repo remote URL against `build_settings.repo_url`.
If matched: set `HAS_NETLIFY=true`, `NETLIFY_SITE_ID`, `NETLIFY_SITE_URL`.
If no `SITE_URL` was provided in args and `HAS_NETLIFY=true`: set `SITE_URL = NETLIFY_SITE_URL`.

---

## Step 3 (AUDIT MODE): Run the SEO Audit

Read and follow `$AUTOFEATURE_HOME/adapted/feature-seo-audit.md`.

Pass:
- `SITE_URL` (live URL if available)
- `HAS_NETLIFY`, `NETLIFY_SITE_ID`

Save the full audit output to `.autofeature/seo-audit-[YYYY-MM-DD].md`.

After the audit report is shown, ask:

```
Found [N] SEO issues ([C] critical, [H] high, [M] medium).

A) Fix all critical + high issues now  ← recommended if C > 0
B) Fix a specific issue                 ← pick from numbered list
C) Create a Trello card for this work
D) Audit only — no changes
```

**If A selected:**
Set `FIX_REQUEST = "Fix all critical and high SEO issues found in audit at [SEO_AUDIT_PATH]"` and proceed to Fix Mode (Step 4).

**If B selected:**
Show numbered list of critical + high findings. User selects one or more. Set `FIX_REQUEST = [selected issues]` and proceed to Step 4.

**If C selected:**
Proceed to Step 3c (Trello card creation).

**If D selected:**
Print the audit path and exit.

### Step 3c: Create Trello Card (if selected)

Ask: "Which Trello list should I add this to?" — call `mcp__trello__get_lists` to show available lists on the active board.

Card format:
```
Title: SEO improvements — [project name]

Description:
## Audit Summary
[Date] audit of [SITE_URL or project name]

🔴 Critical: [N issues — brief list]
🟡 High: [N issues — brief list]
🟢 Medium: [N issues — brief list]

## Recommended Action
[Top 2-3 fixes with effort estimates]

Full audit: .autofeature/seo-audit-[date].md
```

Call `mcp__trello__add_card_to_list` with the formatted card. Confirm success. Exit.

---

## Step 4 (FIX MODE): Implement SEO Improvements

Run the full autofeature pipeline (`$AUTOFEATURE_HOME/.claude/commands/autofeature.md`) with:

**Feature request:** `FIX_REQUEST` (from fix: arg or from audit selection)

**Additional context injected into Step 2 (Feature Interrogation):**
- SEO audit findings (if available at `SEO_AUDIT_PATH`) — include in Feature Brief under `## SEO Audit`
- Force `STACK = REACT` (SEO work is always frontend)
- Skip `mongo-data-modeler` and `express-mongo-architect` — SEO is frontend only

**Step 5 override (Technical Planning):**
When spawning the Plan subagent, include: "This is an SEO improvement task. Consult `$AUTOFEATURE_HOME/agents/seo-architect.md` for stack-specific patterns."

**Step 7 override (Implementation):**
Instead of (or in addition to) `react-architect`, spawn `seo-architect`:

```
Read $AUTOFEATURE_HOME/agents/seo-architect.md
Agent({
  description: "SEO implementation",
  subagent_type: "general-purpose",
  prompt: "[seo-architect.md content]

  Mode: implement
  Feature Brief: [path]
  SEO Audit: [SEO_AUDIT_PATH or 'not available']
  Repo: [pwd]
  Issues to fix: [FIX_REQUEST]"
})
```

If the fix also requires component changes (e.g., adding `<Helmet>` to existing pages), spawn `react-architect` in parallel.

**Step 9 addition (Pre-Ship Review):**
After the standard review passes, add one SEO-specific check:
- Verify that every new/modified page component includes a `<Helmet>` (or equivalent) block
- Verify `robots.txt` includes `Sitemap:` directive
- Verify `_redirects` SPA fallback rule is still the last rule (must not have been displaced)

**Step 10 addition (PR body):**
Append to the PR body:

```markdown
## SEO Changes
[List of fixes applied from audit]

## Verify after deploy
- [ ] Fetch page source — confirm <title> and meta description are populated
- [ ] https://[site]/sitemap.xml resolves
- [ ] https://[site]/robots.txt includes Sitemap: directive  
- [ ] Paste URL in Facebook Sharing Debugger — confirm og:image loads
- [ ] Run Lighthouse SEO audit — target score 90+
```

---

## File Reference

| File | Purpose |
|------|---------|
| `adapted/feature-seo-audit.md` | SEO audit methodology — codebase scan + live site check + findings report |
| `agents/seo-architect.md` | SEO specialist — implements prerendering, meta tags, sitemap, Netlify config, structured data |
