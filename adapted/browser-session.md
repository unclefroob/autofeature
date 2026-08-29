---
status: CUSTOM
description: |
  Session establishment for browser-driven testing — how to get an authenticated browser WITHOUT
  typing a password. Claude in Chrome refuses password fields by design, so any methodology whose
  login step is "find the password input and fill it" now marks every auth-gated flow BLOCKED.
  This module replaces that step with a six-rung ladder (seed-and-mint → dev session endpoint →
  same-origin fetch login → token injection → storage-state replay → human-in-the-loop), a detection
  procedure that picks the rung from the repo and the running app rather than from a guess, and a
  two-signal verification gate that refuses to let an unauthenticated run report results.
  Shared by /autofeature:ui-test, /autofeature:test and /autofeature:award-ui.
---

# Browser Session — authenticate without typing a password

Claude in Chrome will not type into a password field. That is a guardrail, it is deliberate, and it
is not going to be argued out of the way. The consequence for browser testing is total: a login step
built on `form_input` into `#password` no longer completes, so every auth-gated flow behind it
resolves to **BLOCKED**, and a run that blocks on flow one has tested nothing.

**The fix is not to defeat the refusal. It is to stop needing it.**

A browser session is a cookie or a token. Typing a password into a form is one way to obtain one, and
it is the slowest, the flakiest and by far the most privileged way. It couples every test you own to
the login screen's markup, it re-runs your rate limiter on every flow, and it is the single step most
likely to break for a reason that has nothing to do with what you were testing. The other five ways
are all available, all faster, and none of them involve a keystroke.

> **The rule this module enforces:** the password never reaches the browser's input path. Where a
> fixture credential has to be exchanged for a session, the exchange happens over HTTP — from the
> shell, or from a same-origin `fetch` the page itself performs — never as typing into a field.

That distinction is the whole thing. `curl -d '{"password":"..."}'` against your own dev API with an
account your own seeder invented ten seconds ago is an ordinary HTTP request. Driving a human's real
password through a browser's keyboard is not, and this module does not do it at any rung.

---

## Preconditions — refuse the run if any of these fail

Check all three **before** rung 0. They are what make the rest of this file safe to run unattended.

| Precondition | How to check | On failure |
|---|---|---|
| **Non-production target** | Host is `localhost`/`127.0.0.1`/`*.local`, an explicit dev/staging/preview host, or a deploy-preview URL. Confirm the API agrees: `NODE_ENV`/`APP_ENV` is not `production`. | **STOP.** Say plainly that this module does not authenticate against production, and offer rung 5 instead. |
| **Fixture credentials only** | The account is one a seeder created, or a throwaway test account. Its password is a fixture constant, not a secret anyone relies on. | **STOP.** Drop to rung 5. |
| **Not a real person's password** | Including the user's own, and including one they volunteer. | **STOP.** Rung 5, always. |

A host that merely *looks* internal is not a check. `app.staging.example.com` resolving to the
production database has happened to everyone once. Where the API exposes a health or version endpoint
that names its environment, read it and believe it over the hostname.

If the user explicitly directs a run against production anyway, that is their call to make — but the
answer stays rung 5 (they log in themselves, in their own browser), because no rung below it should
ever hold a real credential.

---

## Detection — pick the rung from evidence, not from a guess

Do this once per target and cache the answer in `.autofeature/session-profile.json` (gitignored). It
is the difference between one probe and re-deriving auth on every flow.

**1. Find how the app stores a session.** Read the frontend, don't guess the key — a wrong
`localStorage` key is the most common silent failure in this whole file:

```bash
# Token in client storage? The key name is whatever THIS repo chose.
grep -rn --include=*.{ts,tsx,js,jsx,svelte,vue} -E \
  "(localStorage|sessionStorage)\.setItem\(\s*['\"]" src app 2>/dev/null | head -20

# Cookie-based? Find the name the server sets, and whether it is httpOnly.
grep -rn -E "(cookie-session|express-session|cookieParser|res\.cookie|setCookie|httpOnly|SameSite)" \
  src server api 2>/dev/null | head -20

# Where does login actually live, and what does it return?
grep -rn -E "(/auth/login|/login|signIn|authenticate)" --include=*.{ts,js} src server api routes 2>/dev/null | head -20
```

**2. Find the seeder and what it creates.** This is usually the fastest route to a working account,
and for self-seeded test data it is the only one that reproduces:

```bash
ls scripts/ prisma/ seeds/ 2>/dev/null | grep -iE "seed|fixture|factory"
grep -rn -iE "seed|fixture" package.json | head -20
```

Read the seeder for the fixture password constant and the roles it creates. Do not invent a password
and hope; a seeder that hashes `"password123"` is telling you the answer.

**3. Probe the live login endpoint from the shell** — one request, and it settles which rung applies:

```bash
curl -sS -i -X POST "$API/auth/login" \
  -H 'Content-Type: application/json' \
  -d '{"email":"<seeded>","password":"<fixture>"}' | sed -n '1,40p'
```

Read the response:

| What comes back | Rung |
|---|---|
| `Set-Cookie: …; HttpOnly` | **2** (same-origin fetch) — JS cannot write this cookie, the browser must receive it |
| `Set-Cookie` without `HttpOnly` | **2**, or **3** via `document.cookie` |
| A JSON body carrying a token, no cookie | **3** (inject into the storage key you found in step 1) |
| `302` to an external IdP | **5**, unless you add rung 1 |
| `404`/`405` | The route is elsewhere — find it in step 1's grep before assuming auth is broken |

**Record which signals prove a session is live** (step below, "Verify"), because you need them at
every rung.

---

## The ladder — take the highest rung the app supports

### Rung 0 — seed the account and mint its session in one step (best)

When the harness seeds its own data, the seeder already knows every user it created, and it runs with
direct database and signing-key access. Have it emit sessions alongside the fixtures:

```bash
npm run seed:test -- --emit-sessions > .autofeature/sessions.local.json
```

One object per role: `{ "manager": { "cookies": [...], "storage": {...} }, "employee": {...} }`.
Nothing is guessed, nothing is exchanged, no login endpoint is touched, and a run reproduces exactly
because the seed reproduces exactly. If the repo's seeder cannot do this yet, adding the flag is a
small change and it retires rungs 1–4 permanently — offer it.

### Rung 1 — a dev-only session endpoint

The standard Playwright/Cypress answer, and the right one when you control the API. One route that
takes a user id or role and returns a session, guarded so it cannot exist in production:

```js
// Mount ONLY in non-production. Guards ordered so the route is invisible unless deliberately enabled.
router.get('/__test/login', (req, res) => {
  if (process.env.NODE_ENV === 'production') return res.status(404).end();
  if (!process.env.TEST_LOGIN_SECRET) return res.status(404).end();
  if (req.query.secret !== process.env.TEST_LOGIN_SECRET) return res.status(404).end();

  const user = findFixtureUser(req.query.role);
  if (!user) return res.status(404).end();

  issueSession(res, user);              // the app's own session issuance — not a parallel copy
  res.redirect(req.query.next || '/');
});
```

`404` rather than `403` at every guard: a `403` advertises that the route exists. `TEST_LOGIN_SECRET`
unset means unset in production, so the route is dead there by default rather than by remembering.
Reuse the app's real `issueSession` — a second, subtly different session format is a bug generator.
Ship it with a test asserting it 404s under production config, or it will drift into being live.

Then the browser does nothing clever at all:

```
navigate → https://dev.example.test/__test/login?role=manager&secret=$TEST_LOGIN_SECRET&next=/roster
```

The server sets the cookie on a real navigation, so httpOnly, `Secure` and `SameSite` all behave
exactly as they do for a human. This rung is the most faithful to production of any below it.

### Rung 2 — same-origin fetch login (handles httpOnly)

No dev endpoint, and the session is an httpOnly cookie JS cannot write. Have **the page** perform the
login request: the response's `Set-Cookie` lands in the real browser jar, httpOnly intact, because a
genuine browser request received it.

```
1. navigate to the app's origin first — any URL on it, even a 404. The origin is what matters.
2. javascript_tool:
```
```js
const r = await fetch('/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',                    // ← without this the cookie is dropped
  body: JSON.stringify({ email: SEEDED_EMAIL, password: FIXTURE_PASSWORD })
});
({ status: r.status, body: (await r.text()).slice(0, 300) })
```
```
3. reload, so the app boots with the session instead of hydrating around its absence.
```

Three failure modes, all of which look like "login didn't work":

- **Wrong origin.** Running the fetch from `about:blank` or the previous site sets the cookie for the
  wrong host, or trips CORS. Navigate to the app first, every time.
- **Split origins.** Web on `app.example.test`, API on `api.example.test` — the fetch needs the
  absolute URL, the server needs `Access-Control-Allow-Credentials: true` with a specific origin (not
  `*`), and the cookie needs `SameSite=None; Secure`. If dev runs over plain HTTP, `Secure` cookies
  are dropped silently. Check the response headers before blaming the credential.
- **The body carried a token instead.** Status `200`, no cookie — that is rung 3, continue there.

### Rung 3 — inject the token

The API returns a token and the client stores it. Mint it out-of-band (the `curl` from detection, or
a repo signing script), then write it to **the key the frontend actually reads** — from detection
step 1, never guessed:

```js
localStorage.setItem('auth_token', TOKEN);   // ← the real key, verbatim from the grep
```

Reload afterwards. Most SPAs read the token once at boot; setting it on a live page leaves the app
rendering its logged-out shell over a perfectly good session, and the run then reports a broken
product. For non-httpOnly cookies the equivalent is `document.cookie = 'name=…; path=/'` — the same
reload rule applies.

### Rung 4 — storage-state replay

Capture a known-good session once, replay it per run and per role. This is the pragmatic rung when
auth is expensive, rate-limited, or involves a step nothing can automate.

```js
// capture, on an authenticated page
({ storage: {...localStorage}, cookies: document.cookie })
```

Write to a **gitignored** `.autofeature/session-state.local.json`, one entry per role. Re-inject on a
fresh tab, then reload. Sessions expire, so treat a failed verification here as "recapture", not as
"the app is broken" — and say which it was in the report.

### Rung 5 — the human's own session

Two shapes, both legitimate, both honest:

- **Reuse what's already there.** Claude in Chrome drives the user's real Chrome. If they are already
  logged in to the dev app, there is nothing to establish — check for a live session before doing any
  work at all. This is frequently the correct answer and it costs nothing.
- **Pause and ask.** For third-party SSO, MFA, magic links, or any production target: open the login
  page, ask the user to authenticate in that tab, wait for them to confirm, then verify and carry on.

Rung 5 is not a failure. It is the only correct rung for real credentials, and a run that uses it is
worth strictly more than one that reports everything BLOCKED.

---

## Verify — two independent signals, before any flow runs

The dangerous outcome is not a session that fails to establish. It is one that *appears* to. An SPA
renders its shell before auth resolves, an API happily returns `200` with an anonymous body, and a
public marketing route looks identical either way. Assert **identity**, not the absence of a login
form, and require both signals:

**1. The API agrees you are somebody specific.**

```js
const r = await fetch('/api/me', { credentials: 'include' });
({ status: r.status, body: (await r.text()).slice(0, 200) })
```

`200` is not enough — the body must name the user you intended. Wrong user is worse than no user: the
run completes, every assertion passes, and the results describe an account nobody was testing.

**2. The DOM agrees.** `read_page` on an authenticated route and find a logged-in marker that cannot
render anonymously — the user's name, a sign-out control, a tenant switcher. "No login form present"
is not a marker; plenty of apps render their shell before redirecting.

If either signal fails, **stop**. Do not walk the flows. An unauthenticated run produces a report in
which every flow failed for the same wrong reason, which reads exactly like a catastrophically broken
product and costs an afternoon to disbelieve. Report `BLOCKED (session not established at rung N)`
with the rung, the probe response and what you expected — that is a one-line fix for whoever reads
it, where "everything is red" is not.

---

## Multiple roles

Establish per role; never assume a role change followed a URL. Re-run the ladder per role and
re-verify — the identity assertion above is what catches a manager-scoped test silently running as an
employee, which otherwise surfaces as a permissions bug that isn't real. One tab per role is cleaner
than switching in place, and lets a cross-role flow (a manager approves what an employee submitted)
run without re-establishing anything.

## Hygiene

- Fixture credentials are still credentials: redact to `••••` in every surface — report, screenshots,
  console, error messages. A screenshot of a token in `localStorage` is a leaked token.
- Session files (`.autofeature/sessions.local.json`, `session-state.local.json`, `session-profile.json`)
  must be gitignored. Verify with `git check-ignore` before writing; if not ignored, **STOP** and
  offer to add them. Never write a session blob to a tracked path.
- Never write the session-establishment code into the app's own test suite as a shortcut around real
  auth coverage. This is how tests get driven, not a substitute for testing that login works — keep
  at least one flow that exercises the real login screen with the real fixture user, and let it be
  the one flow that reports BLOCKED where the password field cannot be typed.
- Close tabs you opened. Leave the user's own tabs alone.

## What this module returns to its caller

```
rung:      0–5, which one succeeded
identity:  the user the API confirmed (never the credential)
roles:     established role → tab id
verified:  both signals passed | BLOCKED + reason
notes:     anything the caller should report (recaptured expired state, fell back a rung, …)
```

A caller that receives `BLOCKED` reports every auth-gated flow as `BLOCKED (session)` with that
reason attached, and does not drive them. A caller that receives a rung below the one the app could
support should say so — "fell back to rung 3 because no dev endpoint exists" is how rung 1 eventually
gets built.
