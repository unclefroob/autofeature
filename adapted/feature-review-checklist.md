---
adapted-from: source/review-checklist.md, source/review-specialists-security.md, source/review-specialists-testing.md
changes: |
  - Removed Rails-specific references (update_column, html_safe, etc.) — kept patterns, updated for Node/React/RN/Swift/Kotlin
  - Removed Python async/sync mixing category (not relevant to our stacks)
  - Removed Django/Rails ORM-specific checks — replaced with Prisma/TypeORM/generic equivalents
  - Merged security specialist checklist inline (we don't have parallel subagents)
  - Merged testing specialist checklist inline
  - Removed gstack specialist subagent architecture (JSON output format not needed)
  - Kept Fix-First Heuristic and Suppressions unchanged (good methodology)
  - Added React Native and Swift and Kotlin specific checks
status: ADAPTED
---

# Pre-Ship Review Checklist

Review the diff for issues that unit tests don't catch. Two passes: critical first, then informational.

**Output format:**
```
Review: N issues (X critical, Y informational)

AUTO-FIXED:
- [file:line] Problem → fix applied

NEEDS INPUT:
- [file:line] Problem
  Fix: recommended fix
```

If no issues: `Review: Clean.`

Be terse. One line per problem. One line per fix. No preamble.

---

## Pass 1 — CRITICAL

### SQL & Data Safety
- String interpolation in SQL queries → use parameterized queries
  - Node/Prisma: use `${}` template in Prisma raw queries only with `Prisma.sql`; avoid raw string concat
  - Knex/TypeORM: use `?` placeholders or query builder methods
- TOCTOU races: check-then-set without atomic update
- Bypassing ORM validations via raw queries
- N+1 queries: loading associations inside loops without eager loading
  - Prisma: add `include` to the parent query
  - TypeORM: use `leftJoinAndSelect` or `relations` in find options

### Race Conditions & Concurrency
- Read-check-write without uniqueness constraint or optimistic locking
- Status transitions not atomic (no `WHERE status = 'old' UPDATE SET status = 'new'`)
- XSS via unsafe HTML rendering:
  - React: `dangerouslySetInnerHTML` with user content
  - React Native: `WebView` with user-controlled HTML
  - General: `.innerHTML =` with unsanitized data
- Node.js: concurrent async operations modifying shared state without synchronization

### Security (Auth & Input)
- Endpoints/handlers missing authentication middleware — check route definitions
- Authorization defaulting to "allow" instead of "deny"
- Direct object reference: user can access another user's resource by changing an ID (DORK)
- Role escalation: user can modify their own role/permissions
- User input accepted without validation at the controller/handler boundary
- File uploads without type/size/content validation
- Webhook payloads processed without signature verification
- SSRF via user-controlled URLs passed to fetch/axios/got

### Injection Vectors
- Command injection: `child_process.exec()` / `execSync()` with string interpolation → use `execFile()` with args array
- Path traversal: user-controlled file paths without sanitization (`path.join` doesn't protect against `../`)
- Template injection with user input in template strings passed to templating engines
- Header injection via user-controlled values in HTTP response headers

### Cryptographic Issues
- `Math.random()` for tokens, IDs, or secrets → use `crypto.randomBytes()` or `crypto.randomUUID()`
- MD5 or SHA1 for security-sensitive hashing → use SHA-256 or bcrypt/argon2 for passwords
- Non-constant-time comparison (`===`) on secrets or tokens → use `crypto.timingSafeEqual()`
- Hardcoded secrets, API keys, or passwords in source code

### Secrets Exposure
- API keys or passwords in source code (including comments)
- Secrets logged to `console.log` or application logs
- Sensitive data in error responses returned to the client
- PII stored in plaintext (should be encrypted at rest)

### Swift-specific Critical
- Force-unwrapping optionals that could be nil in production (`!`)
- Main thread violations: UIKit/SwiftUI updates off main thread
- Unsafe access to shared mutable state from multiple threads without synchronization
- Keychain data stored without `kSecAttrAccessibleWhenUnlocked` or stricter

### Kotlin-specific Critical
- `!!` force-unwraps that can throw NullPointerException on realistic inputs
- Network or disk I/O on the main thread (blocked main thread = ANR)
- Unsafe access to shared state in coroutines without Mutex or atomic operations

### Enum & Value Completeness
When diff adds a new enum value, status constant, or type literal:
- Trace it through every switch/when/match statement
- Check every array/set of sibling values for inclusion
- Check conditional chains — does the new value fall through to a wrong default?
This requires reading code OUTSIDE the diff.

---

## Pass 2 — INFORMATIONAL

### Node.js / TypeScript
- `any` type on values that cross trust boundaries (user input, external API responses)
- Non-null assertions (`!`) on values that could realistically be null/undefined
- Missing `await` on async calls that should be awaited (check return value is used)
- Error swallowed in `.catch(() => {})` or `catch (e) {}` with no logging or handling
- `console.log` left in production code paths

### React
- Missing `key` props on list items, or using array index as key for dynamic lists
- `useEffect` with missing or incorrect dependency array → stale closure
- State update after component unmount (memory leak) → needs cleanup function
- Expensive computation in render body without `useMemo`
- Event handlers re-created every render without `useCallback` causing child re-renders

### React Native
- All React checks above
- Missing Platform.OS check where platform behavior differs
- `ScrollView` used for long lists that should be `FlatList`
- Inline style objects creating new references on every render in list items
- Missing `removeEventListener` or subscription cleanup in `useEffect`

### Swift / iOS
- Retain cycles: closures that capture `self` strongly without `[weak self]`
- Async callbacks that update UI without dispatch to main queue
- Force-cast (`as!`) without type check
- Missing `guard let` / `if let` for values that can be nil in practice

### Kotlin / Android
- Coroutines launched in `GlobalScope` (leaks scope, not tied to lifecycle)
- Missing `lifecycleScope.launchWhenStarted` for UI work in Fragments
- `LiveData.observe()` without the correct `LifecycleOwner`
- Blocking calls (`runBlocking`) in coroutine context that should be non-blocking

### Completeness Gaps
- Error paths with no user feedback (blank screen on failure)
- API endpoints with no input validation
- Background jobs with no error handling or retry logic
- Features at 80-90% when 100% is achievable — tests are the cheapest lake to boil

### CI/CD & Distribution
- New artifact types without a publish/release workflow
- Hardcoded secrets in CI config (should use `${{ secrets.X }}`)
- Version tag format inconsistency (`v1.2.3` vs `1.2.3`) across VERSION, git tags, publish scripts

---

## Testing Pass

Apply these checks separately after the main passes:

### Missing Negative-Path Tests
- Error branches in try/catch with no test for the error path
- Guard clauses and early returns that are untested
- Auth/permission checks with no test for the "denied" case

### Missing Edge-Case Coverage
- Boundary values: `0`, `-1`, empty string `""`, empty array `[]`, `null`/`undefined`/`nil`
- Single-element collections (off-by-one on loops)
- Unicode characters in user-facing string inputs

### Test Isolation
- Tests sharing global/module-level mutable state
- `Date.now()` / `new Date()` in tests without mocking (non-deterministic)
- Tests making real network calls instead of using mocks/stubs

### Flakiness Risk
- `setTimeout`/`sleep` in tests (use fake timers)
- Assertions on insertion order of unordered data structures
- Tests that pass in isolation but fail when run with others (shared state)

### Coverage Gaps
- New public functions with no test coverage at all
- Changed functions where existing tests only cover the old code path

---

## Fix-First Heuristic

```
AUTO-FIX (apply without asking):          ASK (present to user):
├─ console.log in production paths        ├─ Security: auth, injection, XSS
├─ Missing await on fire-and-forget       ├─ Race conditions
├─ Missing key prop (use item.id)         ├─ Design decisions (which approach)
├─ Missing null check on obvious value    ├─ Large fixes (>20 lines)
├─ Stale comment contradicting code       ├─ Enum completeness gaps
├─ Magic number → named constant          ├─ Removing functionality
└─ Dead code / unused import              └─ Changes to user-visible behavior
```

**Rule:** Mechanical fix that any senior engineer would make without discussion → AUTO-FIX.
Reasonable engineers could disagree → ASK.

---

## Suppressions — Do NOT flag

- Redundancy that aids readability
- Consistency-only changes that don't affect behavior
- Anything already addressed in the diff being reviewed
- Third-party/vendor code in node_modules or vendor directories
- Patterns documented in DESIGN.md or CLAUDE.md as intentional
- Test assertion verbosity (multiple guards in one assertion is fine)
