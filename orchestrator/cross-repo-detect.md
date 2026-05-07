---
name: cross-repo-detect
purpose: Find sibling repos that should be coordinated with the current one
status: CUSTOM
---

# Cross-Repo Detection

The orchestrator runs this **during Step 2 (interrogation)** when the feature looks like it might span repos, or always when in a repo with a recognizable suffix.

## Why

The user typically organizes paired projects under `~/dev/`:
- `<base>-api` — Express + Mongo backend
- `<base>-mobile` — React Native
- `<base>-app` — React Native (alternate naming)
- `<base>-desktop` — likely Electron or a web app
- `<base>-cms` — admin frontend
- `<base>-website` — marketing or public site

A feature like "add user profile photo upload" almost always touches the API + at least one frontend. Detecting siblings up front lets us coordinate branches and run `api-contract-broker`.

## Algorithm

```bash
CWD=$(pwd)
REPO_NAME=$(basename "$CWD")
PARENT=$(dirname "$CWD")

# Strip the trailing role suffix to get the base
# Suffixes recognized (in match order — longer first):
#   -website, -desktop, -mobile, -admin, -cms, -app, -api
BASE=$(echo "$REPO_NAME" | sed -E 's/-(website|desktop|mobile|admin|cms|app|api)$//')

# If no suffix matched, REPO_NAME == BASE → no siblings to look for
if [ "$REPO_NAME" = "$BASE" ]; then
  echo "no-suffix"
  exit 0
fi

# List sibling directories matching ${BASE}-*
ls -d "$PARENT/${BASE}-"* 2>/dev/null
```

Filter the result to:
- Directories that are git repositories (contain `.git`)
- Not the current repo
- Each one's git branch and clean/dirty status

## Output

```markdown
## Sibling Repos Detected

**Current repo:** `[name]` ([role])
**Base name:** `[base]`

**Siblings:**
| Repo | Role | Path | Current branch | Dirty? |
|------|------|------|----------------|--------|
| foo-mobile | mobile | ~/dev/foo-mobile | main | clean |
| foo-desktop | desktop | ~/dev/foo-desktop | feature/old-thing | DIRTY |
| foo-cms | cms | ~/dev/foo-cms | main | clean |
```

## Coordination prompt

If siblings are found, ask the user (CHECKPOINT) or apply heuristic (AUTOMATED):

> Siblings detected: **[list]**
>
> The feature description suggests changes in: [auto-inferred list — e.g. "api + mobile"]
>
> A) Yes, coordinate — create matching `feature/[slug]` branches in each, run api-contract-broker, ship coordinated PRs
> B) Just this repo — I'll handle siblings separately
> C) Custom — pick which siblings to include

In AUTOMATED mode, default behavior:
- If feature description names a sibling explicitly ("update mobile too", "and the desktop") → include it
- If feature is in `*-api` and the words "endpoint", "API", "backend" suggest backend-only → DON'T auto-pull frontends in (user can re-run autofeature in mobile separately)
- If feature is in `*-api` and the words involve UI ("user can see", "screen for", "page for") → pull the frontends in
- When unclear, pull in the frontends marked CLEAN; skip DIRTY ones with a note

## Safety rules

- **Never modify a dirty sibling.** Surface it and skip. The user has unfinished work there.
- **Never check out a sibling's branch silently.** Always create a fresh `feature/[slug]` from the sibling's main/master.
- **One ship at a time.** When shipping coordinated PRs, ship the API first, confirm green, then mobile/desktop. Order matters because frontend depends on the API being deployed.

## Files emitted into the run state

For coordinated runs, write to `.autofeature/coordination.md` in the primary repo:

```markdown
## Coordinated Run

**Primary repo:** [name]
**Sibling repos:** [list with paths]
**Branch in all:** feature/[slug]
**Ship order:** api → cms → website → mobile → desktop

**Per-repo status:**
- [repo]: [phase — pending | designed | implemented | tested | shipped]
```

The orchestrator updates this after each phase per repo so a resume picks up correctly.
