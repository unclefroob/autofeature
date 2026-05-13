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

Two sources of truth are combined:
1. **Codebase scan** (`feature-seo-audit.md`) — what's in the source files before deploy. Catches missing packages, wrong config, broken Netlify rules, semantic HTML issues.
2. **Live site analysis** (`claude-seo` skills) — what crawlers actually see after deploy. Catches JS rendering failures, missing meta on the rendered page, schema errors, Core Web Vitals.

Run both in parallel where possible for speed.

### 3a. Codebase scan (always)

Spawn as a subagent to keep findings out of main context:

```
Agent({
  description: "SEO codebase scan",
  subagent_type: "general-purpose",
  prompt: "Read and execute $AUTOFEATURE_HOME/adapted/feature-seo-audit.md — Steps 1 only (codebase scan, no live fetch).
  Repo: [pwd]
  Return the full structured findings list with severity tags. Under 600 words."
})
```

### 3b. Live site analysis (if SITE_URL available)

Run all three in a **single parallel message**:

```
[single message — all three Agent/Skill calls]

Skill({ skill: "claude-seo:seo-technical", args: SITE_URL })
  → JS rendering check, robots.txt, crawlability, Core Web Vitals, AI crawler config

Skill({ skill: "claude-seo:seo-audit", args: SITE_URL })
  → Full crawl-based audit: up to 500 pages, 15 specialist subagents, health score
  → Note: this is thorough and may take several minutes

Skill({ skill: "claude-seo:seo-page", args: SITE_URL })
  → Deep analysis of the homepage: on-page elements, content, meta, schema
```

If no SITE_URL: skip 3b. After the codebase scan, surface this message:
> ⚠️ No live URL — codebase scan only. For a complete audit (what crawlers actually see), re-run with `url: https://yourdomain.com`. The most impactful issues (React SPA rendering, meta tag rendering) require a live URL to confirm.

### 3c. Merge findings into unified report

Combine results from 3a and 3b. Deduplicate — if the same issue appears in both (e.g., "no sitemap"), merge into one finding. Mark source: `[code]` for codebase-only findings, `[live]` for crawl-only, `[both]` for confirmed in both.

Save full merged report to `.autofeature/seo-audit-[YYYY-MM-DD].md`.

### 3d. Show summary and offer next steps

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

**Step 4a: Pre-implementation enrichment (before seo-architect runs)**

If the fix involves **structured data / schema**, generate correct JSON-LD templates first:
```
Skill({ skill: "claude-seo:seo-schema", args: SITE_URL or "" })
```
Save the output as `SEO_SCHEMA_CONTEXT`. Pass it to seo-architect so it implements correct, validated JSON-LD rather than guessing structure.

If the fix involves **sitemap issues**, get the current sitemap state first:
```
Skill({ skill: "claude-seo:seo-sitemap", args: SITE_URL or "" })
```
Save as `SEO_SITEMAP_CONTEXT`. Pass to seo-architect.

**Step 7 override (Implementation):**
Spawn `seo-architect` as the primary implementer. If `react-architect` is also needed (e.g., adding `<Helmet>` to many existing page components), spawn both in a **single parallel message**:

```
[single message]

Agent({
  description: "SEO implementation",
  subagent_type: "general-purpose",
  prompt: "[seo-architect.md content]

  Mode: implement
  Feature Brief: [path]
  SEO Audit: [SEO_AUDIT_PATH or 'not available']
  Schema context: [SEO_SCHEMA_CONTEXT or 'not applicable']
  Sitemap context: [SEO_SITEMAP_CONTEXT or 'not applicable']
  Repo: [pwd]
  Issues to fix: [FIX_REQUEST]"
})

Agent({  ← only if component-level changes needed
  description: "React component SEO wiring",
  subagent_type: "general-purpose",
  prompt: "[react-architect.md content]

  Mode: implement
  Job: Add <Helmet> blocks to the page components listed in the Feature Brief.
  Follow the pattern established by seo-architect in this same run.
  Feature Brief: [path]
  Repo: [pwd]"
})
```

**Step 9 addition (Pre-Ship Review):**
After the standard review passes, add one SEO-specific check:
- Verify that every new/modified page component includes a `<Helmet>` (or equivalent) block
- Verify `robots.txt` includes `Sitemap:` directive
- Verify `_redirects` SPA fallback rule is still the last rule (must not have been displaced)

**Step 10 addition (PR body):**
Append to the PR body:

```markdown
## SEO Changes
[List of fixes applied — one line per issue resolved]

## Before/After
- SEO health score (seo-audit): [N/100 → expected improvement]
- Issues resolved: [C critical, H high, M medium]

## Verify after deploy
- [ ] Fetch page source — confirm <title> and meta description are populated
- [ ] https://[site]/sitemap.xml resolves
- [ ] https://[site]/robots.txt includes Sitemap: directive
- [ ] Paste URL in Facebook Sharing Debugger — confirm og:image loads
- [ ] Run /autofeature:seo url: https://[site] — re-audit to confirm fixes
```

---

## File Reference

### AutoFeature files
| File | Purpose |
|------|---------|
| `adapted/feature-seo-audit.md` | Codebase scan — checks source files for missing packages, broken config, semantic HTML issues |
| `agents/seo-architect.md` | React/Vite/Netlify SEO implementer — prerendering, meta tags, sitemap, Netlify config, structured data |

### claude-seo skills (live site analysis)
| Skill | When invoked | What it contributes |
|-------|-------------|---------------------|
| `claude-seo:seo-audit` | Audit mode with URL | Full crawl (500 pages), 15-specialist delegation, health score 0-100 |
| `claude-seo:seo-technical` | Audit mode with URL | JS rendering check, robots.txt, Core Web Vitals, AI crawler config |
| `claude-seo:seo-page` | Audit mode with URL | Deep homepage analysis — on-page elements, content quality, meta |
| `claude-seo:seo-schema` | Fix mode (schema work) | Detects + validates existing schema, generates correct JSON-LD templates |
| `claude-seo:seo-sitemap` | Fix mode (sitemap work) | Validates sitemap structure, generates missing pages list |
