---
name: review
description: |
  One entry point for the autofeature review lenses — dispatches on the first token:
    /autofeature:review product [path]   → whole-product gaps & broken flows (CEO / PM / flow-walker)
    /autofeature:review feature <desc>   → opinion + build advice for one feature
    /autofeature:review market <idea>    → market demand, competitive gap & VC fundability
    /autofeature:review scope <desc>     → scope-tier classification (micro|single-layer|cross-stack|cross-repo)
  With no mode, it asks which lens. Each mode runs the exact same workflow as the matching standalone command.
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
  - mcp__trello__get_lists
  - mcp__trello__add_card_to_list
---

# AutoFeature Review — unified entry point

The four review lenses behind one command, so the product reads as **one builder with review modes**
rather than four co-equal commands. Pure router: it dispatches to the existing command/methodology
for the chosen mode — nothing is reimplemented, and the standalone commands still work unchanged.

## $AUTOFEATURE_HOME

```bash
# Files ship with the plugin — prefer its root; fall back to an explicit home or dev clone.
for _d in "$AUTOFEATURE_HOME" "${CLAUDE_PLUGIN_ROOT}" "$HOME/dev/autofeature"; do
  [ -n "$_d" ] && [ -d "$_d/adapted" ] && { AUTOFEATURE_HOME="$_d"; break; }
done
```

## Step 1: Parse the mode

The **first whitespace token** after `/autofeature:review` is `MODE`; everything after it is `REST`
(the arguments for that lens).

| MODE | Lens | Runs |
|------|------|------|
| `product` | Whole-product gaps & broken flows | `.claude/commands/product-review.md` (audit mode) |
| `feature` | Opinion + build advice for one feature | `.claude/commands/feature-review.md` |
| `market` | Market demand, competitive gap & VC fundability | `.claude/commands/market-review.md` |
| `scope` | Scope-tier classification only | `.claude/commands/scope.md` |

- If `MODE` is missing or not one of the four, **ask which lens** (list the four with one-line
  descriptions) and **stop** — do not guess.
- Accept obvious synonyms: `prod`→product, `feat`→feature, `mkt`/`funding`→market.

## Step 2: Dispatch

Read `$AUTOFEATURE_HOME/.claude/commands/<mode>-review.md` (or `scope.md` for `scope`) and follow it
exactly, treating `REST` as that command's arguments. For example:
- `/autofeature:review market an AI tool for SMB ops` → run `market-review.md` with idea = "an AI tool for SMB ops".
- `/autofeature:review feature` (no REST) → run `feature-review.md`, which captures the feature from
  conversation context (and STOPs to ask if none is found).
- `/autofeature:review product ./packages/api` → run `product-review.md` audit scoped to that path.

This command adds no behavior of its own beyond routing. To change a lens, edit its command +
methodology, not this file.

## Relationship to the standalone commands

`/autofeature:product-review`, `:feature-review`, `:market-review`, and `:scope` remain available and
behave identically — this is a convenience surface and the single discoverable name. (Roadmap: once
`/autofeature:review` is the documented path, the four standalones can become thin aliases to it.)
