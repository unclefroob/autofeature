---
name: do
description: |
  Hand autofeature any task in plain words and let it triage — which kind of work it is, which repo it
  means, which command / skill / specialist should do it, and on which model.
  Builds go to the pipeline; questions like "have a look into the iOS app and see how the styles are"
  go to a read-only survey by that stack's architect on Sonnet; audits, tests, reviews and award work
  go to their own commands.
  Invoke as:
    /autofeature:do <task>
    /autofeature:do route-only <task>   → show the Route Card, run nothing
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
  - Workflow
  - Skill
  - TaskCreate
  - TaskUpdate
  - Monitor
  - mcp__trello__get_card_details
  - mcp__trello__get_card_checklists
  - mcp__trello__add_comment_to_card
---

# AutoFeature Do — triage any task

Pure router, like `review.md`: it decides where a request goes and dispatches it. The decision logic
lives in `orchestrator/task-router.md`, shared with `/autofeature` Step 0.75. Change routing there,
not here.

## $AUTOFEATURE_HOME

```bash
# Files ship with the plugin — prefer its root; fall back to an explicit home or dev clone.
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

If no candidate resolves, abort with: `AutoFeature methodology files not found. They ship with the plugin — reinstall it, or for a dev clone set AUTOFEATURE_HOME=/path/to/autofeature.`

## Step 1: Parse

`TASK` = everything after `/autofeature:do`. If it starts with `route-only`, set `ROUTE_ONLY` and strip
the token. If `TASK` is empty, take the task under discussion in this conversation; if there is none,
ask for it and stop.

## Step 2: Triage

Read and follow `$AUTOFEATURE_HOME/orchestrator/task-router.md` with `TASK` through Step 4, and emit
the Route Card.

`ROUTE_ONLY` → stop after the card. Also list the runner-up route when intent confidence was `low`.

## Step 3: Dispatch

Follow the router's Step 5. When the route is the `/autofeature` pipeline, read
`$AUTOFEATURE_HOME/.claude/commands/autofeature.md` and start at **Step 0.5** (Trello detection) with
`FEATURE_REQUEST = TASK`, skipping Step 0.75 — triage has already run.
