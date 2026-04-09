---
adapted-from: source/plan-eng-review.md, source/review-checklist.md (error mapping), source/ethos.md (principles)
changes: |
  - Removed all gstack bin references (gstack-review-log, gstack-slug, etc.)
  - Removed {{PLACEHOLDERS}} — resolved inline or removed
  - Removed Codex outside voice integration
  - Removed worktree parallelization section
  - Removed learnings log/search
  - Adapted test review for unit tests only (no eval suites, no browser QA)
  - Added tech stack specific guidance: Node.js, React, React Native, Swift, Kotlin
  - Removed review dashboard, plan file review report
  - Added: Error & Rescue mapping (from plan-ceo-review methodology)
  - Added: Shadow path testing (happy/nil/empty/error for every data flow)
  - Added: Observability checklist (logging, error handling for new code)
  - Automated mode: applies 6 decision principles from autoplan-principles.md
  - Checkpoint mode: uses AskUserQuestion for each issue
status: ADAPTED
---

# Feature Planning (Engineering Review)

Eng manager-mode technical plan. Lock in architecture, data flow, edge cases, and unit test coverage before writing code.

**Mode behavior:**
- **Automated:** Apply the 6 decision principles (completeness, boil lakes, pragmatic, DRY, explicit, bias to action). Surface only genuine "User Challenges" to the user.
- **Checkpoint:** Use AskUserQuestion for every issue found. One issue per call.

---

## Before You Start

Read the Feature Brief from `.autofeature/designs/[slug]-[date].md`. This is the source of truth for the problem statement, constraints, and chosen approach.

---

## Step 0: Scope Challenge

Answer these questions before any architecture review:

1. **What existing code already partially or fully solves each sub-problem?**
   Use Grep/Glob to search. Can we extend existing flows rather than building parallel ones?

2. **What is the minimum set of changes that achieves the stated goal?**
   Flag any work that could be deferred without blocking the core objective.

3. **Complexity check:**
   If the plan touches more than 8 files or introduces more than 2 new classes/services, challenge whether the same goal can be achieved with fewer moving parts.

4. **Tech stack pattern check:**
   For each new pattern the plan introduces, verify:
   - Does the framework have a built-in? (e.g., Express middleware, React hooks, Swift Combine, Kotlin coroutines)
   - Is the chosen approach current best practice for this stack?
   - Are there known footguns for this pattern in this stack?

5. **Unit testability check:**
   Will new code be unit testable as written? If a module mixes I/O with logic, flag it — extract the logic into pure functions before implementation.

If complexity check triggers: recommend scope reduction. In checkpoint mode, ask via AskUserQuestion.

---

## Step 1: Architecture Review

Evaluate the proposed implementation approach:

- **Component boundaries:** Is each module/service/component doing one thing?
- **Dependency direction:** Are dependencies flowing the right way? (services depend on models, not the reverse)
- **Data flow:** ASCII diagram for any multi-step data transformation or async flow
- **API surface:** For new API endpoints or functions, are the contracts clear?
- **Auth & data access:** If this touches user data, are auth checks in the right place?
- **Failure scenarios:** For each new I/O boundary (DB query, external API call, file read), what happens when it fails?

### Stack-specific architecture checks

**Node.js backend:**
- Async/await consistency — no mixing callbacks and promises in the same flow
- Error propagation — are all async errors caught and forwarded (no unhandled promise rejections)?
- Middleware order — if adding middleware, is placement in the chain correct?
- DB query patterns — are queries parameterized? Any N+1 risk?

**React:**
- State placement — is state lifted to the right level, or does it belong in a store/context?
- Effect dependencies — any useEffect with missing or wrong deps?
- Memoization — are expensive computations wrapped in useMemo/useCallback where needed?
- Component size — components > 300 lines are a smell; identify split points

**React Native:**
- All React checks above
- Platform-specific code — using Platform.OS checks or platform-specific files where needed?
- Performance — FlatList vs ScrollView for lists; avoiding inline styles in list items
- Navigation — screen params typed correctly; navigation flow matches the feature

**Swift / iOS:**
- Memory management — any retain cycles (closures capturing self)?
- Threading — UI updates on main thread? Background work dispatched correctly?
- Optionals — forced unwraps that could crash in production?
- SwiftUI state — correct property wrapper for the data lifetime (@State, @StateObject, @ObservedObject, @EnvironmentObject)?

**Kotlin / Android:**
- Coroutine scope — using appropriate scope (viewModelScope, lifecycleScope)?
- Null safety — any `!!` force-unwraps that could crash?
- Main thread — UI updates on Main dispatcher?
- ViewModel — is business logic in ViewModel, not Activity/Fragment?

**Stop (checkpoint mode):** For each issue, use AskUserQuestion. One issue per call. Present options + recommendation + why.

**Automated mode:** Apply decision principles. Flag as User Challenge if the feature's architecture needs to fundamentally change.

---

## Step 2: Code Quality Review

Evaluate the planned implementation approach:

- **DRY:** Does this duplicate logic that exists elsewhere in the codebase?
- **Error handling:** Are all error paths explicit? No silent failures?
- **Naming:** Are names descriptive enough for the next developer?
- **Module size:** Any new module planned to be >300 lines? (identify split points)
- **Explicit over clever:** If there's a simple way and a clever way, pick the simple way

**Stop (checkpoint mode):** One issue per AskUserQuestion.
**Automated mode:** Apply decision principles silently.

---

## Step 3: Unit Test Plan

Produce a unit test plan before implementation starts. This becomes the test spec.

### Test coverage requirements

Every new piece of logic needs a unit test for:
1. **Happy path** — the expected input produces the expected output
2. **Edge cases** — empty input, null/nil/undefined, boundary values
3. **Error paths** — what happens when dependencies fail

### Test framework by stack

**Node.js:** Jest or Vitest (check package.json for which is installed)
- Test files: `*.test.ts` or `*.spec.ts`, co-located with source or in `__tests__/`
- Mocking: `jest.mock()` / `vi.mock()` for external deps (DB, APIs, file system)
- Run command: `npm test` or `npx vitest`

**React:** Jest + React Testing Library
- Test files: `*.test.tsx` co-located with components
- Render + user interaction: `render()`, `fireEvent`, `userEvent`
- No implementation details: test what the user sees, not internal state
- Run command: `npm test`

**React Native:** Jest + @testing-library/react-native
- Test files: `*.test.tsx`
- Use `renderHook` for hooks, `render` for components
- Mock native modules that don't work in Jest (`__mocks__/` directory)
- Run command: `npx jest`

**Swift / iOS:** XCTest (built-in Xcode)
- Test files: `*Tests.swift` in `*Tests/` target
- Use `XCTestCase` subclasses; `setUp`/`tearDown` for fixtures
- Mock with protocols — define protocol for dependency, inject mock in tests
- Run command: `xcodebuild test` or Cmd+U in Xcode

**Kotlin / Android:** JUnit + Mockk (or Mockito)
- Test files: `*Test.kt` in `src/test/` (unit) or `src/androidTest/` (instrumented)
- Use `@MockK` / `@InjectMockKs` for dependency injection in tests
- Coroutines in tests: `runTest {}` from `kotlinx-coroutines-test`
- Run command: `./gradlew test`

### Test plan format

For each new function, method, or component, list:

```
[unit name]: [file path]
  ✓ happy path: [input → expected output]
  ✓ edge case: [what edge case → what happens]
  ✓ error path: [what fails → what is returned/thrown]
```

---

## Step 3b: Error & Rescue Mapping

For every new function, endpoint, or service that does I/O (DB, network, filesystem, device hardware):

Produce a table:

| Codepath | What can fail | Exception/Error type | Rescue action | User sees |
|----------|--------------|---------------------|---------------|-----------|
| `getUserById(id)` | DB unreachable | `PrismaClientKnownRequestError` | Return 503 with retry-after header | "Something went wrong, try again" |
| `uploadPhoto(file)` | File too large | Validation error | Return 400 with max size | "File must be under 10MB" |

**Rules:**
- Every catch-all (`catch (e) {}`, `.catch(() => {})`) is a smell — what specific error is being swallowed?
- Every error must: retry + backoff, OR degrade gracefully + inform user, OR re-raise with context
- Silent failures (error caught, nothing happens, user sees blank) are critical gaps

For any error path with no rescue action AND no user feedback: flag as **critical gap** in the test plan.

---

## Step 3c: Shadow Path Testing

For every new data flow, map all 4 paths — not just the happy path:

```
DATA FLOW: [input] → [transform] → [persist/return]

Path 1 (Happy):  valid input → expected output
Path 2 (Nil):    null/undefined/nil input → ?
Path 3 (Empty):  empty string / empty array / zero → ?
Path 4 (Error):  dependency fails (DB down, network timeout) → ?
```

For each path: is there a test? Is there error handling? Does the user see something meaningful?

Any path without both a test AND error handling is a gap to fix before implementation.

---

## Step 3d: Observability Checklist

For every new code path that matters (background jobs, API endpoints, critical business logic):

- **Logging:** Is entry and exit logged at appropriate level? Is error logged with context (not just `console.error(e)`)?
- **Error context:** When an error is thrown/returned, does it include enough context to debug from logs alone?
- **Silent failures:** Any place where an error is swallowed and nothing is logged? That's a monitoring black hole.

**Minimum bar for production code:**
- Errors always logged with: error message, relevant IDs, operation that failed
- No silent catch blocks: `catch (e) { /* do nothing */ }` → at minimum `console.error('[featureName] operation failed:', e)`

---

## Step 4: Performance Review

- **Complexity:** Are there O(n²) operations where O(n) or O(1) is achievable?
- **N+1 queries:** Any database query inside a loop?
- **Unnecessary re-renders (React/RN):** Any parent re-render causing expensive child re-renders?
- **Memory:** Any large data structure held in memory when it could be streamed or paginated?

---

## Required Outputs

### Implementation Plan

At the end of the review, produce a concrete implementation plan:

```
## Implementation Plan

### Files to create:
- [path]: [purpose]

### Files to modify:
- [path]: [what changes]

### Implementation order:
1. [step 1 — foundation/infrastructure]
2. [step 2 — core logic]
3. [step 3 — integration]
4. [step 4 — tests]

### Unit test files:
- [test file path]: tests for [source file]

### NOT in scope:
- [deferred item]: [reason]

### What already exists:
- [existing code]: [how it relates to the feature]

### Failure modes:
- [codepath]: [how it fails] → [test coverage: yes/no] → [error handling: yes/no]
```

In **checkpoint mode**: present the plan and ask for approval before proceeding to implementation.
In **automated mode**: proceed directly to branch creation and implementation.
