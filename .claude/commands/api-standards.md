---
name: api-standards
description: |
  Alias — superseded by /autofeature:backend-api-skills. Forwards there with the Ritchies API profile pre-selected and, with no argument, in fix mode (the behaviour-preserving route → controller → service → model refactor this command used to own).
  Kept so existing muscle memory and any scripted invocations keep working. New work should call /autofeature:backend-api-skills directly.
  Invoke as: /autofeature:api-standards [audit | <domain> | all] [mode:checkpoint|automated]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Agent
  - Workflow
  - Skill
  - TaskCreate
  - TaskUpdate
  - Monitor
---

# AutoFeature — API Standards (alias)

This command has been **superseded by `/autofeature:backend-api-skills`**, which does what this did and also creates new backend code to the same standard, on any detected stack rather than only Node + Express + Mongoose.

## What to do

Read `$AUTOFEATURE_HOME/.claude/commands/backend-api-skills.md` and run it, with these defaults applied:

1. **Target:** prefer `ritchies-platform-api` — whether invoked from it or from a sibling directory.

   ```bash
   CWD=$(pwd); PARENT=$(dirname "$CWD")
   for c in "$CWD" "$PARENT/ritchies-platform-api" "$CWD/ritchies-platform-api"; do
     [ -f "$c/src/app.ts" ] && grep -q '"express"' "$c/package.json" 2>/dev/null && { API="$c"; break; }
   done
   [ -z "$API" ] && { echo "Run from ritchies-platform-api or its parent dir."; exit 1; }
   ```

2. **Profile:** Ritchies. Read `$AUTOFEATURE_HOME/ritchies/conventions.md` (API section) **in addition to** `backend/api-standard.md` — it adds the repo-specific rules (four-pillar RBAC via `auth/resolver`, `auth/audit` on admin mutations, `catalog:sync` as a deploy step, `auth/` + `chat/` grandfathered as domain modules, CI gating on typecheck + jest + build).

3. **Argument mapping** — this command's old grammar, translated:

   | Typed here | Runs there |
   |------------|------------|
   | `audit` | `audit` |
   | `<domain>` | `fix <domain>` |
   | `all` | `fix all` |
   | *(nothing)* | `audit`, then offer the fix |
   | `mode:checkpoint` / `mode:automated` | passed through unchanged |

Mention the supersession once, in a single line, then get on with the work. Do not make the user re-invoke anything.
