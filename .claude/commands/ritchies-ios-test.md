---
name: ritchies-ios-test
description: |
  Build and test a Ritchies iOS change on a real simulator, from a machine that cannot compile Swift.
  This box has no Xcode. The only route to a compiler is a Claude session on the Mac, so this skill is
  mostly about driving that hand-off well: what to send, what to demand back first, and how to tell a
  session that is working from one that never got your message.
  Drives the simulator through simctl + XCUITest, never by controlling the screen — pixel-clicking fails
  silently and reads as a pass.
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
- **How the simulator is to be driven**, stated in the brief rather than assumed: `xcrun simctl` for
  the device and XCUITest for the UI, and **no screen control** — no computer-use, no pointer, no
  clicking coordinates, no screenshot-then-click. See Step 4 for why, and say it plainly; a session
  told only "test it on the simulator" will often reach for the screen.
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

**Drive the simulator through its own tooling. Never by controlling the screen.**

State this in the brief, in as many words, because it is the single instruction most likely to be
quietly ignored — screen control feels like testing and produces confident-sounding reports that
mean nothing.

**Not allowed:** computer-use, moving a pointer, clicking pixel coordinates, or any
screenshot-look-then-click loop. Coordinates move with device size, dynamic type, locale and every
layout change; a click that lands on the wrong element fails **silently** and reads as a pass; and a
model interpreting pixels will cheerfully report a screen it has misread. None of it is repeatable
and none of it can ever run unattended.

**Two tools, two jobs:**

| Tool | Drives | Use it for |
|---|---|---|
| `xcrun simctl` | the **device** | boot/`bootstatus`, install, launch (`--console` for stdout), `openurl` for deep links, `privacy` grants, `push`, `status_bar` overrides, `io screenshot`, `log stream --predicate`, `get_app_container` for inspecting sandboxed files |
| **XCUITest** | the **app's UI** | the actual taps, typing and assertions — elements addressed by accessibility identifier, never by position |

The `all-ios-skills:ios-simulator` skill covers the `simctl` half in detail; hand it to the Mac
session rather than re-deriving the commands.

**Screenshots and `log stream` are EVIDENCE, captured to show what happened. They are never the
control mechanism.**

**If an element cannot be addressed, that is a finding, not a reason to fall back to coordinates.**
Add an `accessibilityIdentifier` (or a label) in the app and treat it as part of the change —
untestable UI is a defect worth fixing while you are in there, and every control this pipeline adds
should be reachable by name from the day it ships.

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

## Traps that have actually cost hours

Each of these was paid for once. None is guessable from the failure message.

### The iOS "Save Password?" dialog

After a successful sign-in, iOS may offer to save the password. It arrives in its
own window and covers the app, so **every element behind it reports as
non-hittable**. Dismiss it at the END of the sign-in helper, before anything else
touches the UI:

```swift
app.buttons["Sign in"].tap()

// iOS offers to save the password after a successful sign-in. It arrives in its
// own window and covers the app, which makes every element behind it report as
// non-hittable. Dismiss it before touching anything else.
let notNow = app.buttons["Not Now"]
if notNow.waitForExistence(timeout: 10) {
    notNow.tap()
}
```

**The `if` is required.** The dialog does not always appear — it depends on prior
simulator state — so an unconditional tap fails on the run where iOS decides not
to ask.

**The failure signature is the real trap, not the dialog.** You see
`Failed to not hittable: Button ... label: 'Operations'` on an element plainly on
screen, which reads as a z-order or scrim bug in your own views. One run was
spent patching `allowsHitTesting(false)` onto a drawer scrim chasing exactly
that, and it was the wrong fix. **If several unrelated elements all report
non-hittable at once, suspect a system window before you suspect your layout.**
`print(app.debugDescription)` shows the dialog; a query for one element does not.

### Do not poll `waitForExistence` in a hittability loop

`waitForExistence(timeout: 0.25)` returns INSTANTLY once the element exists, so a
loop around it busy-spins and starves the accessibility server — slowing the very
thing it is waiting for. Use a real sleep:

```swift
while Date() < deadline {
    if element.exists && element.isHittable { return true }
    Thread.sleep(forTimeInterval: 0.5)
}
```

### Invoke tests individually

Batching several `-only-testing:` flags into one `xcodebuild` makes them all fail
fast at the first navigation step, even on an idle machine. Cause unknown, not
guessed at. Individually each takes 22-28s and passes.

### A test filter matching the FILE name can match zero tests

The suite names inside a file often differ from the filename. Zero tests run
reads exactly like zero failures. Always report the actual count.

### Make failures say what IS on screen

A timeout on the way in and a genuinely wrong value fail the same assertion and
need opposite follow-up. Print the actual header, which states are visible, and
whether a spinner is up — not only that the expected string was absent.

Two corollaries worth copying:

- **Assert the control was live BEFORE the state you are testing.** In a
  stale-queue test, assert Approve IS offered before the row goes stale.
  Otherwise "no Approve button" reads identically whether the button was never
  enabled or was correctly withdrawn.
- **Guard destructive tests on their preconditions BEFORE the first write.** A
  test that proposes, writes, and only then fails to authenticate its second
  actor leaves real state behind and reports a UI fault. Check the precondition
  and skip, naming the missing config in the skip message.

### Loose on wording, strict on provenance

Pin the SHAPE of a message, not its exact text — a copy edit should not read as a
regression. But do assert it came from the server rather than from a generic
client string, because that is the property you actually care about.

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
- **Screen control is not testing.** Clicking coordinates fails silently when it lands on the wrong
  element, so a mis-driven pass and a real pass are indistinguishable in the report. `simctl` for the
  device, XCUITest for the UI, screenshots for evidence only — and if an element cannot be addressed
  by name, add an accessibility identifier rather than reaching for the pointer.
- **When waiting on a deploy, check something that should ALREADY work.** A watch
  that only polls the endpoint it wants cannot tell "not deployed yet" from
  "service is down" — both are non-200, and it will sit there reporting progress
  about a dead service. Poll login (or any known-good route) alongside the thing
  you are waiting for. This was found the hard way: a green deploy, a watch
  patiently waiting, and the whole API 503 for several minutes.
- **Deployment is not a merge.** A client feature calling a new endpoint is untestable until the API
  is actually deployed. Probe it: an existing route unauthenticated returns **401**, a route that
  isn't there returns **404**. That is the cheapest deploy check there is.
