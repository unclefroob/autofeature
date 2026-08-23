---
name: backend-api-standard
purpose: The portable backend API standard — the layering law, the layer contracts, the violation catalogue, and the verify gates. Read by /autofeature:backend-api-skills in every mode.
status: CUSTOM
---

# Backend API Standard

The authoritative definition of "the correct shape" for a backend API. It is stated **stack-neutrally** — the law is about where logic lives, not about which framework you imported. `stack-profiles.md` maps these layer names onto a concrete stack; `scaffold.md` turns them into files.

The reference implementation is Node + Express + TypeScript + Mongoose + zod + Jest (`ritchies-platform-api`), because that is the stack the standard was extracted from. Every rule below is written so it survives being pointed at Fastify + Prisma + Postgres + Vitest instead.

---

## The Iron Law

**Business logic lives in a layer that does not know HTTP exists.**

Everything else in this document is a consequence. A rule earns its place here only if breaking it puts domain logic somewhere that cannot be called without a request object.

The test of compliance is not "does it look layered" — it is:

> Can every business rule in this codebase be exercised by a unit test that never constructs a request, a response, or a router?

If the answer is no for a domain, that domain is non-compliant, whatever its directory names say.

---

## The four layers

```
route → controller → service → model
```

Each layer has a contract: a list of what it MAY do, and a list of what it MUST NOT. The MUST NOTs are what the audit greps for.

### 1. Route

Declares the HTTP surface and nothing else.

**MAY:** declare paths and methods; attach auth middleware; attach coarse capability/role guards; attach rate limits; bind a named controller handler; export the router.

**MUST NOT:** contain a handler body. A route file with an inline `async (req, res) => { … }` that does more than delegate is a controller in disguise.

One route file per domain. Registered in the app entrypoint with a single mount line.

### 2. Controller

The HTTP↔domain translator. Thin by construction — a reviewer should be able to read a whole controller and learn nothing about the business.

**MAY (and in this order):**
1. Parse and validate input into a typed DTO (`schema.parse(req.body)` — a declarative schema, not hand-rolled `if (!x) throw`).
2. Resolve the caller's access context, memoised on the request.
3. Call one or more service functions with plain arguments.
4. Shape the HTTP response — status code, body envelope, headers.
5. `try/catch → next(error)` so the central error handler owns the mapping.

**MUST NOT:** issue a database query; contain a business rule, a branch on domain state, or a multi-step domain workflow; reach across domains; format domain data beyond selecting what to return; construct its own error responses that bypass the central handler.

The tell that a controller has absorbed the service: it contains a query call (`.find(`, `.aggregate(`, `.create(`, `prisma.*`, `db.query(`, a raw SQL string) or a sequence of domain conditionals.

### 3. Service

Where the product actually lives. An **HTTP-agnostic function module**, not a class: exported `async function`s taking plain arguments (ids, validated DTOs, a resolved access object) and returning plain data.

**MAY:** query and mutate through models; orchestrate across entities; enforce visibility, scoping and domain invariants; serialise domain objects into the shapes the API returns; call other services; `throw` typed domain errors.

**MUST NOT:** import a request, response, router, or framework type; read `req.*`; set a status code; know that HTTP exists; be a class with injected framework state.

A service is a **function module** deliberately. Classes invite constructor-injected request state, which is how the HTTP boundary leaks back in.

### 4. Model

Schema, indexes, and data-integrity rules — the shape of a row/document.

**MAY:** declare fields, types, enums, defaults, timestamps; declare indexes; hide secrets from default selection; carry integrity-level hooks and small instance methods.

**MUST NOT:** carry cross-entity business rules or anything that needs to know why a write is happening. A hook that emails a user on save is business logic hiding in the schema.

---

## Reuse before extract

**Before writing logic, look for it.** Grep the service layer (and any grandfathered domain modules) for something that already does the job, and import it rather than re-implementing.

This is a hard rule, not a preference, because the failure mode is silent: two implementations of the same rule diverge on the first bugfix, and nothing fails a test.

**Extract shared helpers.** If a helper is used by — or would be used by — two or more domains, it belongs in its own module in the service layer, imported everywhere. Duplicated helper bodies across domains are a violation even when both copies are correct.

**Grandfathered domain modules.** A repo may have pre-existing service code organised as domain folders rather than `services/<domain>Service`. Keep importing from it; do NOT relocate it as part of an audit. Relocation is a behaviour risk with no behavioural payoff. New domains follow the standard shape.

---

## Cross-cutting contracts

### Errors

Errors funnel through **one central error handler**, registered LAST in the app entrypoint. Services `throw` typed domain errors; controllers `next(error)`; the handler maps error type → status code and response body. The error response shape is one shape, repo-wide.

A controller that builds its own error response bypasses the mapping and will drift from every other endpoint.

### Validation

Input is validated at the **controller boundary**, before any service call, with a declarative schema that produces a typed DTO. Reject early. A service receiving a validated DTO may trust its shape; it may not trust its semantics (a well-formed id can still be an id the caller cannot see).

### Auth vs authorization

Two different jobs at two different layers, and conflating them is the most common security-shaped violation:

- **Route-level guards** answer "may this caller reach this endpoint at all" — authentication, and coarse capabilities the caller either holds or does not.
- **Resource-scoped checks** answer "may this caller do this to *this record*". These belong in the **service**, after the resource is loaded. A route guard cannot know the resource, so a scoped rule enforced at the route is either wrong or accidentally correct.

Access resolution has exactly one implementation. Nothing re-derives permissions from a role name or an access level.

### Serialisation

The server authors the verdict; clients render it. If the answer to "can this user do X" requires domain knowledge, the service computes it and ships it in the response. Clients do not re-implement ladders, tiers, or precedence rules.

---

## The test contract

Two tiers, and the split is the reason the layering pays for itself:

| Tier | Target | Speed | What it proves |
|------|--------|-------|----------------|
| **Service unit tests** | service functions, called directly | fast | the business rules are right |
| **Route tests** | the HTTP surface, through the real router | slower | wiring, validation, auth, status codes |

Most coverage belongs in the fast tier. Route tests assert the endpoint is wired, guarded, and returns the documented shape — not every branch of the domain.

A domain with route tests and no service tests is under-tested even at 100% line coverage: it is paying integration-test latency for unit-test assertions.

---

## Violation catalogue

What an audit looks for. Severity is about blast radius, not effort.

| ID | Violation | Tell | Severity |
|----|-----------|------|----------|
| **V1** | Business logic in the controller | query calls or domain branching inside a handler | 🔴 |
| **V2** | No service module for a domain that has non-trivial logic | `controllers/x` exists, `services/x` does not | 🔴 |
| **V3** | Scoped authorization enforced at the route | a resource-specific rule in a route-level guard | 🔴 |
| **V4** | Service imports HTTP | request/response/router types in the service layer | 🔴 |
| **V5** | Duplicated helper across domains | same function body in 2+ files | 🟡 |
| **V6** | Handler body inline in the route file | route file longer than its path list | 🟡 |
| **V7** | Bypassed error handler | hand-built error responses in a controller | 🟡 |
| **V8** | Validation after the first data access | parse call below a query call | 🟡 |
| **V9** | Missing service unit tests | `services/x` exists, no unit test for it | 🟡 |
| **V10** | Business rule in a model hook | cross-entity work in a schema hook | 🟡 |
| **V11** | Re-derived permissions | role/level decoded outside the access resolver | 🔴 |
| **V12** | Unpropagated async errors | `async` handler with no `try/catch`/wrapper | 🟡 |
| **V13** | Service as a class with injected request state | constructor takes `req` or a framework object | 🟡 |
| **V14** | Inconsistent response envelope | one domain's success/error shape differs from the repo's | 🟢 |

🔴 = correctness or security risk. 🟡 = maintenance debt that compounds. 🟢 = consistency.

**A violation is not a bug.** If the audit surfaces a genuine defect (a rule that is wrong, not just misplaced), report it **separately** and do not fix it inside a structural change. Mixing the two makes the diff unreviewable and destroys the identical-behaviour proof.

---

## Verify gates

Whatever the mode, the same three gates decide whether the work is done, and each must be **run**, not assumed:

1. **Typecheck** clean.
2. **Tests** green — and for structural work, *identically* green (see below).
3. **Build** clean, because that is what CI gates on.

### Identical-green (structural work only)

The pre-existing route/integration suite is the behaviour contract. A structural change is proven behaviour-preserving when:

- every pre-existing test passes **unchanged**, and
- the count of pre-existing tests is the same before and after.

New service unit tests **add** to the total; that is expected and good. But **if a pre-existing route test had to be edited to pass, behaviour changed** — revert and reconsider. That is the whole safety net; editing the test to match the new behaviour cuts it.

**No green baseline, no structural work.** If the suite is red before you start, stop and report. Refactoring on top of failing tests destroys the only evidence you had.

---

## What this standard deliberately does not say

- **Directory names.** `services/fooService.ts`, `foo/service.ts`, and `domain/foo.ts` all satisfy the law. Match the repo; do not impose a rename.
- **Which framework, ORM, validator, or test runner.** See `stack-profiles.md`.
- **Whether to use a repository layer.** An extra layer between service and model is permitted, not required. It does not substitute for the service.
- **REST vs RPC vs GraphQL shape.** The law is about where logic lives, not about URL design.
- **Anything a repo's own `.autofeature/patterns.md` already decides.** Where a repo canon exists it **wins** on style, naming, and helper choice. This standard governs layering only, and the two should not conflict; if they do, surface it rather than silently picking one.
