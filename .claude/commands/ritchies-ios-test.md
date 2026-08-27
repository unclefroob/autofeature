---
name: ritchies-ios-test
description: |
  Build and test a Ritchies iOS change on a real simulator, from a machine that cannot compile Swift.
  This box has no Xcode. The only route to a compiler is a Claude session on the Mac, so this skill is
  mostly about driving that hand-off well: what to send, what to demand back first, and how to tell a
  session that is working from one that never got your message.
  Ends with an honest verdict — "built and simulator-verified", "compiled only", or "never compiled" —
  and never inflates one into another.
  Invoke as: /autofeature:ritchies-ios-test [<what changed / Trello #>]
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - ListAgents
  - SendMessage
  - AskUserQuestion
  - Skill
---

# Ritchies — iOS build & simulator test

`ritchies-mobile` cannot be compiled on the Linux box this skill usually runs from. There is no Xcode,
no `xcodebuild`, no simulator. Every claim about iOS therefore comes from somewhere else, and the
whole point of this skill is to make that somewhere else reliable and to keep the reporting honest.

**The cardinal rule:** never describe iOS work as tested unless a compiler and a simulator actually
ran. "It follows the same pattern as the file next to it" is not evidence. Say
`never compiled` and move on — an accurate gap is worth more than a confident guess.

---

## Step 0: Do you already have the answer?

If a Mac session has ALREADY built this exact commit and reported, skip to Step 5 and write the
verdict. Re-running a green build to feel thorough wastes ten minutes of someone else's machine.

---

## Step 1: Push first, always

The Mac pulls from GitHub. It cannot see your working tree. So:

1. Commit and push the branch.
2. Note the **exact SHA**. Every message you send about this work names that SHA.

A brief that says "the latest code" against a Mac that pulled ten minutes ago produces a green build
of the wrong commit, which is worse than no build at all.

---

## Step 2: Find a Mac — by machine, not by project

`ListAgents`, then pick a session **whose name identifies the machine**, e.g. one containing
`macbook`, `mac`, `mbp`.

**Do not pick a session because its name matches the project.** A session called
`Ritchies Specialist` is named after the work, not the hardware, and may be running on Linux, in the
cloud, or anywhere else. This has already cost a real session four unanswered messages and a wrong
report to the user. If no name clearly identifies a Mac, **ask the user which session has Xcode**
rather than guessing.

If the target has no history with this work, write the brief from scratch — repo, what changed, why.
Assume zero context.

---

## Step 3: The brief — demand the compile result on its own

Send ONE message containing:

- **Repo + SHA**, and that XcodeGen is in use, so there is no `.xcodeproj` in the tree and
  `xcodegen` must run before any build.
- **Every file added or changed**, by path, one line each.
- **The specific risks you could not check yourself**, named. Anything you already verified against
  the codebase, say so and say *don't change it* — otherwise a helpful session will "fix" working
  code into something off-pattern.
- **The order of work, and an explicit instruction not to batch**:

  > Send the compile result IMMEDIATELY — pass, or the exact error text. Don't fix anything yet,
  > don't move on.

- **Any runtime dependency.** New endpoints that are on `main` but NOT DEPLOYED will 404 against the
  dev backend. Say so, and say the app can be pointed elsewhere via the `api_base_url` UserDefaults
  key without a rebuild.
- **An explicit invitation to report blockage**: "a `blocked because X` reply is far more useful to
  me than silence."

The minimal ask, if the full brief is too much for them to take on:

```
git pull                                  # main @ <SHA>
xcodegen
xcodebuild -scheme Ritchies \
  -destination 'platform=iOS Simulator,name=iPhone 16 Pro' build
```

That one result unblocks everything else.

---

## Step 4: The simulator pass

Use `xcrun simctl`, not screen capture — the `all-ios-skills:ios-simulator` skill covers lifecycle,
install, launch, permissions and log streaming. Screenshots are for evidence, never for driving.

Ask for the checks that would actually catch a regression, phrased as observable outcomes rather
than as "check X works":

- The **specific user action from the ticket**, and what should now happen instead of the old
  behaviour. Name both.
- Anything **deliberately left alone** — it must still behave the old way. A viewer that swallows
  the types it cannot render is a worse bug than the one being fixed.
- Anything **verified only in unit tests on the API side**, confirmed end to end through the UI.
  Server-side behaviour proven only by a jest assertion is the highest-value thing a simulator run
  can retire.
- The **error path**, forced — kill the API mid-action. Expect an honest message and a retry that
  actually recovers, not a permanent spinner.

---

## Step 5: The verdict

Exactly one of these, and never a warmer one than the evidence supports:

| Verdict | Means |
|---|---|
| `built + simulator-verified` | Compiled, tests ran, and the flows were driven on a simulator. |
| `built + tests green` | Compiled and the unit tests ran. Nothing was driven. |
| `compile-checked only` | A type-check ran. Say what could not be caught — `swiftc -typecheck` is blind to the project file, so an added file that is not in the target still "passes". |
| `never compiled` | No compiler ran. The default until proven otherwise. |

Report which SHA the verdict applies to. A verdict silently inherited by later commits is how
"tested" ends up attached to code nobody built.

---

## Learnings this skill exists to encode

- **A project-named session is not a machine.** Match on hardware in the name, or ask.
- **Cross-session delivery can fail silently in both directions.** Reports sent may never arrive. If
  several messages go unanswered, suspect the channel rather than the recipient, tell the user
  plainly that nothing has been received, and offer the fallback: they can run the four commands
  themselves with a `!` prefix and the output lands directly in the conversation.
- **Never write down a peer's result you have not received.** If the user asks whether it passed and
  nothing has arrived, the answer is "nothing has reached me", not a plausible summary.
- **Check the house pattern before flagging a risk.** Actor-isolation on an `@Observable` store looks
  alarming and is usually already settled elsewhere in the repo — `ChatStore` is `@MainActor` +
  `@Observable`, held as `@State private var store = ChatStore()`, with `nonisolated static` pure
  helpers. Matching it is correct; "fixing" it is not.
- **XcodeGen means new files need no project edit** (`sources: - path: Ritchies`), but `xcodegen`
  must run before building.
- **A test filter matching the FILENAME can match zero tests.** The suite names live inside the file
  and often differ from it. Zero tests run reads exactly like zero failures.
- **UIKit is a real import.** A file building `UIScrollView`/`UIImageView` that compiles only because
  PDFKit dragged UIKit in is one refactor away from breaking. Import it explicitly.
- **Deployment is not a merge.** A client feature calling a new endpoint is untestable until the API
  is actually deployed. Probe it: an existing route unauthenticated returns **401**, a route that
  isn't there returns **404**. That is the cheapest deploy check there is.
