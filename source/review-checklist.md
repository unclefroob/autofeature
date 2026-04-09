---
gstack-source: gstack/review/checklist.md
extracted: 2026-04-09
note: Full extract. Unmodified. Rails/Python/Django specific references preserved.
status: ORIGINAL (unmodified)
---

# Pre-Landing Review Checklist

## Instructions

Review the `git diff origin/main` output for the issues listed below. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

**Two-pass review:**
- **Pass 1 (CRITICAL):** Run SQL & Data Safety, Race Conditions, LLM Output Trust Boundary, Shell Injection, and Enum Completeness first. Highest severity.
- **Pass 2 (INFORMATIONAL):** Run remaining categories below. Lower severity but still actioned.
- **Specialist categories (handled by parallel subagents, NOT this checklist):** Test Gaps, Dead Code, Magic Numbers, Conditional Side Effects, Performance & Bundle Impact, Crypto & Entropy. See `review/specialists/` for these.

All findings get action via Fix-First Review: obvious mechanical fixes are applied automatically,
genuinely ambiguous issues are batched into a single user question.

**Output format:**

```
Pre-Landing Review: N issues (X critical, Y informational)

**AUTO-FIXED:**
- [file:line] Problem → fix applied

**NEEDS INPUT:**
- [file:line] Problem description
  Recommended fix: suggested fix
```

If no issues found: `Pre-Landing Review: No issues found.`

Be terse. For each issue: one line describing the problem, one line with the fix. No preamble, no summaries, no "looks good overall."

---

## Review Categories

### Pass 1 — CRITICAL

#### SQL & Data Safety
- String interpolation in SQL (even if values are `.to_i`/`.to_f` — use parameterized queries (Rails: sanitize_sql_array/Arel; Node: prepared statements; Python: parameterized queries))
- TOCTOU races: check-then-set patterns that should be atomic `WHERE` + `update_all`
- Bypassing model validations for direct DB writes (Rails: update_column; Django: QuerySet.update(); Prisma: raw queries)
- N+1 queries: Missing eager loading (Rails: .includes(); SQLAlchemy: joinedload(); Prisma: include) for associations used in loops/views

#### Race Conditions & Concurrency
- Read-check-write without uniqueness constraint or catch duplicate key error and retry (e.g., `where(hash:).first` then `save!` without handling concurrent insert)
- find-or-create without unique DB index — concurrent calls can create duplicates
- Status transitions that don't use atomic `WHERE old_status = ? UPDATE SET new_status` — concurrent updates can skip or double-apply transitions
- Unsafe HTML rendering (Rails: .html_safe/raw(); React: dangerouslySetInnerHTML; Vue: v-html; Django: |safe/mark_safe) on user-controlled data (XSS)

#### LLM Output Trust Boundary
- LLM-generated values (emails, URLs, names) written to DB or passed to mailers without format validation. Add lightweight guards (`EMAIL_REGEXP`, `URI.parse`, `.strip`) before persisting.
- Structured tool output (arrays, hashes) accepted without type/shape checks before database writes.
- LLM-generated URLs fetched without allowlist — SSRF risk if URL points to internal network
- LLM output stored in knowledge bases or vector DBs without sanitization — stored prompt injection risk

#### Shell Injection (Python-specific)
- `subprocess.run()` / `subprocess.call()` / `subprocess.Popen()` with `shell=True` AND f-string/`.format()` interpolation in the command string — use argument arrays instead
- `os.system()` with variable interpolation — replace with `subprocess.run()` using argument arrays
- `eval()` / `exec()` on LLM-generated code without sandboxing

#### Enum & Value Completeness
When the diff introduces a new enum value, status string, tier name, or type constant:
- **Trace it through every consumer.** Read (don't just grep — READ) each file that switches on, filters by, or displays that value.
- **Check allowlists/filter arrays.** Search for arrays or lists containing sibling values.
- **Check `case`/`if-elsif` chains.** If existing code branches on the enum, does the new value fall through to a wrong default?

### Pass 2 — INFORMATIONAL

#### Async/Sync Mixing (Python-specific)
- Synchronous `subprocess.run()`, `open()`, `requests.get()` inside `async def` endpoints — blocks the event loop.
- `time.sleep()` inside async functions — use `asyncio.sleep()`
- Sync DB calls in async context without `run_in_executor()` wrapping

#### Column/Field Name Safety
- Verify column names in ORM queries against actual DB schema — wrong column names silently return empty results
- Check `.get()` calls on query results use the column name that was actually selected

#### Dead Code & Consistency
- Version mismatch between PR title and VERSION/CHANGELOG files
- CHANGELOG entries that describe changes inaccurately

#### LLM Prompt Issues
- 0-indexed lists in prompts (LLMs reliably return 1-indexed)
- Prompt text listing available tools/capabilities that don't match what's actually wired up
- Word/token limits stated in multiple places that could drift

#### Completeness Gaps
- Shortcut implementations where the complete version would cost <30 minutes CC time
- Test coverage gaps where adding the missing tests is a "lake" not an "ocean"
- Features implemented at 80-90% when 100% is achievable with modest additional code

#### Time Window Safety
- Date-key lookups that assume "today" covers 24h
- Mismatched time windows between related features

#### Type Coercion at Boundaries
- Values crossing language→JSON→JS boundaries where type could change (numeric vs string)
- Hash/digest inputs that don't normalize types before serialization

#### View/Frontend
- Inline `<style>` blocks in partials (re-parsed every render)
- O(n*m) lookups in views (`Array#find` in a loop instead of index-by hash)

#### Distribution & CI/CD Pipeline
- CI/CD workflow changes: verify build tool versions, artifact names/paths, secrets use env vars not hardcoded values
- New artifact types (CLI binary, library, package): verify a publish/release workflow exists
- Cross-platform builds: verify CI matrix covers all target OS/arch combinations
- Version tag format consistency: `v1.2.3` vs `1.2.3` must match across VERSION file, git tags, and publish scripts

**DO NOT flag:**
- Web services with existing auto-deploy pipelines
- Internal tools not distributed outside the team
- Test-only CI changes

---

## Severity Classification

```
CRITICAL (highest severity):      INFORMATIONAL (main agent):
├─ SQL & Data Safety              ├─ Async/Sync Mixing
├─ Race Conditions & Concurrency  ├─ Column/Field Name Safety
├─ LLM Output Trust Boundary      ├─ Dead Code (version only)
├─ Shell Injection                ├─ Completeness Gaps
└─ Enum & Value Completeness      ├─ Time Window Safety
                                   ├─ Type Coercion at Boundaries
                                   ├─ View/Frontend
                                   └─ Distribution & CI/CD Pipeline
```

---

## Fix-First Heuristic

```
AUTO-FIX (agent fixes without asking):     ASK (needs human judgment):
├─ Dead code / unused variables            ├─ Security (auth, XSS, injection)
├─ N+1 queries (missing eager loading)     ├─ Race conditions
├─ Stale comments contradicting code       ├─ Design decisions
├─ Magic numbers → named constants         ├─ Large fixes (>20 lines)
├─ Missing LLM output validation           ├─ Enum completeness
├─ Version/path mismatches                 ├─ Removing functionality
├─ Variables assigned but never read       └─ Anything changing user-visible behavior
└─ Inline styles, O(n*m) view lookups
```

**Rule of thumb:** If the fix is mechanical and a senior engineer would apply it
without discussion, it's AUTO-FIX. If reasonable engineers could disagree about
the fix, it's ASK.

---

## Suppressions — DO NOT flag these

- "X is redundant with Y" when the redundancy is harmless and aids readability
- "Add a comment explaining why this threshold/constant was chosen" — thresholds change during tuning
- "This assertion could be tighter" when the assertion already covers the behavior
- Consistency-only changes
- "Regex doesn't handle edge case X" when the input is constrained
- Eval threshold changes — tuned empirically
- Harmless no-ops
- ANYTHING already addressed in the diff you're reviewing — read the FULL diff before commenting
