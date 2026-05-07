# AutoFeature Agent Fleet

Specialist subagent prompts spawned by the autofeature orchestrator. Each file is a self-contained prompt — the orchestrator reads it and passes it inline to a `general-purpose` agent (no install step required).

## Roster

| Agent | Stack / Domain | Spawned when |
|-------|----------------|--------------|
| [`express-mongo-architect`](express-mongo-architect.md) | Express.js + Mongoose backend | Backend changes (routes, controllers, models, middleware) |
| [`react-architect`](react-architect.md) | React web (Vite/Next/CRA/Remix) | Web frontend changes |
| [`react-native-architect`](react-native-architect.md) | React Native (Expo or bare) | Mobile changes |
| [`mongo-data-modeler`](mongo-data-modeler.md) | MongoDB schema + queries + indexes | New collection, schema change, index proposal, migration, or query plan concern |
| [`api-contract-broker`](api-contract-broker.md) | Cross-repo coordination | Backend + frontend(s) touched in same run |
| [`test-runner`](test-runner.md) | Test execution + summary | Verify gate (Step 6) — keeps test logs out of orchestrator context |

## Invocation pattern

The orchestrator does NOT install these as `subagent_type`s. Instead:

```
1. Read the agent file at $AUTOFEATURE_HOME/agents/<name>.md
2. Compose the spawn prompt:
   - Agent file content (the role definition)
   - The specific job for this run (mode, paths, what to deliver)
3. Invoke Agent with subagent_type="general-purpose" and the composed prompt
4. For parallel fan-out, send multiple Agent calls in a single message
```

Why this pattern:
- Files stay portable — no Claude config dir lock-in
- Edit a file → next run picks up the change, no reinstall
- The fleet works on any machine that has this repo cloned

## Calling multiple in parallel

For cross-stack or cross-repo features, spawn architects concurrently:

```
[single message containing:]
Agent({ description: "Backend design", prompt: "[express-mongo-architect.md] + job spec" })
Agent({ description: "Mobile design", prompt: "[react-native-architect.md] + job spec" })
Agent({ description: "Data model review", prompt: "[mongo-data-modeler.md] + job spec" })
```

After all three return, run `api-contract-broker` to reconcile their plans. The broker is sequential (it consumes their outputs).

## Handing off between agents

Agents communicate by writing to the Feature Brief, not by direct messaging. Each agent appends a clearly-headed section. The orchestrator reads the brief between phases.

Sections an agent may write:
- `## Backend Plan (express-mongo-architect)`
- `## Frontend Plan (react-architect)` or `## Mobile Plan (react-native-architect)`
- `## Data Model Review (mongo-data-modeler)`
- `## API Contract (api-contract-broker)`
- `## Scope` (orchestrator, via scope-gate)
- `## Test Run: [unit|e2e|integration]` (test-runner — these can be transient, not always appended)

## Adding a new agent

1. Create `<name>.md` here following the format of existing agents (frontmatter + role + process + cheat-sheet + escalation rules).
2. Update this README's roster table.
3. Wire it into the orchestrator (`~/.claude-account1/commands/autofeature.md`) at the appropriate phase.
4. Add a row to `MANIFEST.md` at the project root.

## Calling outside autofeature

These agents are also useful standalone. To invoke one:

```
Read $AUTOFEATURE_HOME/agents/express-mongo-architect.md
Compose: "<file content>\n\nJob: Design the backend for [feature]. Repo: [path]. Mode: design."
Agent(general-purpose, that prompt)
```

Or paste the content directly into a chat as a system-prompt-style preamble.
