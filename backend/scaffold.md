---
name: backend-scaffold
purpose: Blueprints for create mode — the day-one backend skeleton, and the correctly-shaped domain slice added to an existing backend.
status: CUSTOM
---

# Backend Scaffold Blueprints

Two blueprints, both producing code that satisfies `api-standard.md` from the first commit. Which one runs is decided by whether an app entrypoint already exists.

The point of scaffolding to the standard is negative: **it is cheaper to never create the debt than to run an audit later.** Nothing here is aspirational — if a blueprint item can't be delivered, say so rather than shipping a skeleton that quietly omits it.

---

## Blueprint A — Greenfield skeleton

Run when there is no app entrypoint. The skeleton is deliberately small; it is a spine, not a starter kit.

### The spine (all of it, or explain the omission)

1. **Entrypoint + app assembly, separated.** The app object (middleware, routes, error handler) is built in one module and *exported*; a separate entrypoint binds the port. Fusing them makes the app untestable in-process — the route-test tier depends on this split.
2. **Central error handler, registered LAST.** Maps typed domain errors → status codes, everything unrecognised → 500 with no internal detail leaked. One response shape.
3. **Typed domain error classes.** At minimum: not-found, forbidden, conflict, validation. Services throw these; nothing else decides a status code.
4. **Auth seam.** Authentication middleware that populates a typed caller on the request, and an access resolver that is the *single* implementation of "what may this caller do". Even if the first version is trivial, the seam must exist — V11 violations are born the day a second place decodes a role.
5. **Request-type augmentation.** The typed caller and resolved access declared once, centrally, so no handler casts.
6. **Config module.** Environment read and validated in exactly one place, failing loudly at boot on a missing required var. Never `process.env.X` scattered through services.
7. **Health endpoint.** Liveness, and a readiness check that actually touches the data layer.
8. **Test harness, proven.** An in-memory or containerised test database, both test tiers wired, and **at least one passing test of each tier** — a harness that has never run green is not a harness.
9. **The four gate commands** in `scripts`: `typecheck`, `test`, `build`, `lint`.
10. **First domain slice** via Blueprint B, so the skeleton ships with a worked example of its own standard.

### Deliberately NOT in the spine

Logging framework choice, rate limiting, OpenAPI generation, containerisation, CI config, a message queue. Each is a real decision with real trade-offs; adding them unasked is how a scaffold becomes a framework nobody chose. Offer them as a follow-up list.

---

## Blueprint B — Domain slice

Run to add one correctly-shaped domain to an existing backend (greenfield or long-lived). Files, in dependency order:

1. **Model** — schema, field types, enums for unions, timestamps, indexes declared on the schema, secrets excluded from default selection.
2. **Service** — the domain's function module. Exported `async function`s over plain args; all queries, all rules, all cross-entity orchestration, all serialisation. Throws typed errors. Imports nothing HTTP-shaped.
3. **Controller** — one thin handler per endpoint: parse → resolve access → call service → shape response → `try/catch → next`.
4. **Route** — paths, auth, coarse guards, handler bindings. No handler bodies.
5. **Mount** — one line in the app assembly. Error handler stays last.
6. **Service unit tests** — the domain's rules, called directly. This is where the coverage lives.
7. **Route tests** — wiring, validation rejection, unauthenticated, unauthorised, happy path, and the documented response shape.

### Before writing any of it

Run the **reuse pass**. Grep the service layer and any grandfathered domain modules for logic that already exists — access resolution, audit recording, formatting, membership, visibility. Import it. A new domain that re-implements the access resolver is a 🔴 on arrival.

Then check the **shared-helper** question: does this domain need a helper a second domain already has, or would want? If yes, it goes in a shared service module now, not duplicated with a TODO.

### Definition of done

All three gates run and green (`typecheck`, `test`, `build`), with the new service tests visible in the count. No route test outside this domain changed. If the repo has a capability catalogue, registry, or permission list, the new keys are registered there — **a capability that exists only in code is a deploy-time outage**, and in most deployments that registration is a separate step that must be run per environment.

---

## Sequencing when both run

Greenfield: spine first, verified green with its own trivial tests, *then* the first domain slice. Do not scaffold six domains before running anything — a broken spine multiplied by six domains is six broken domains and no signal about which layer is wrong.
