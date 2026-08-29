---
name: ritchies-conventions
purpose: Per-repo conventions for the four Ritchies platform repos, fed to every architect by the /autofeature:ritchies command
status: CUSTOM
---

# Ritchies Platform — Cross-Repo Conventions

The Ritchies "Team App" is an **internal employee/staff portal** for Ritchies IGA (NOT a consumer storefront — ignore any storefront/ProductTile/Price language in stale READMEs). It ships as four repos kept in deliberate lockstep. Every feature that exists appears across all clients with the same domain model. The canonical reference feature in every repo is **Announcements** — copy its shape.

## The four repos (hard-wired paths)

| Role | Path | GitHub | Stack |
|------|------|--------|-------|
| API | `ritchies-platform-api` | `unclefroob/ritchies-platform-api` | Node ≥20 + Express 4 + TS (strict) + Mongoose 8 + JWT + zod |
| Web | `ritchies-platform-web` | `unclefroob/ritchies-platform-web` | React 19 + Vite 8 + TS + Tailwind + shadcn/ui + axios |
| iOS | `ritchies-mobile` | `unclefroob/ritchies-mobile` | SwiftUI, iOS 26, `@Observable`, XcodeGen, no SPM deps |
| Android | `ritchies-android` | `unclefroob/ritchies-android` | Kotlin 2.0 + Compose (Material3), OkHttp, kotlinx.serialization |

Sibling paths live under the same parent dir as the repo you're invoked in (both `/run/media/ryan/Files/dev/ritchies/` and `/home/ryan/dev/ritchies/` are the same tree). The generic `cross-repo-detect` CANNOT link these (suffix-stripping yields `ritchies-platform` for api/web but `ritchies` for mobile/android) — the command hard-wires them instead.

## Shared invariants (all clients)

- **Auth:** JWT access + refresh. Login identifier is an **8-digit employee number** (`^\d{8}$`), not email. Bearer token on every authorized request; **single-flight 401 refresh** against `POST /api/auth/refresh`, retry once, else clear tokens. Token storage: localStorage (web) / Keychain (iOS) / EncryptedSharedPreferences (Android).
- **API base:** all clients point at `https://ritchies-platform-api-dev.azurewebsites.net` by default (overridable). Endpoints are namespaced `/api/<domain>`. Responses are **plain JSON** (`{ … }`); errors are `{ "error": "…" }` (any "envelope" note in old comments is stale).
- **Authorization = four-pillar scoped RBAC** (Position + Location + Department + Group; additive union; server-side resolver). Granular **capabilities are the authority**; the **1-7 access level** is a default preset/label. Clients fetch `GET /api/me/permissions` → `{ capabilities, accessLevel, … }` and gate UI on capabilities (with level as a fallback/label). `User.permissionOverrides {grant, revoke}` are resolved live (revoke wins).
- **`main` is a deploy trigger, and what it triggers differs per repo.** `ritchies-platform-api` auto-deploys to the dev App Service on merge; `ritchies-platform-web` auto-deploys the Static Web App; **`ritchies-mobile` ships a TestFlight build** (`.github/workflows/ios-release.yml`, on push to `main`: a macOS runner runs xcodegen, archives, signs and uploads to App Store Connect). It runs NO tests — macOS minutes bill at ten times the Linux rate — so a merge releases whatever is on main ungated. Check with `gh workflow list` or `git ls-tree -r origin/main | grep github` after fetching; `ls` from the wrong directory reports "no such file" and reads exactly like "no pipeline". `ritchies-android` has nothing automatic today.

  So **do not merge iOS to `main` as a routine end-of-feature tidy-up.** Land it on the feature branch, open the PR, leave it open, and say plainly that merging is a release decision rather than a ship step. Merge only when the user has said they want a build cut.

- **Check the branch before you reuse the name.** The client repos routinely already carry a `feature/<slug>` branch left over from an earlier run, pointing at a commit **older than `main`**. Verify with `git rev-list --count main..<branch>`; if it is `0` the branch is stale and `git branch -f <branch> main` is the fix. Never `git checkout` onto it with a dirty tree — you will get a merge conflict in a file the feature never touched, and the recovery is `git merge --abort` + `git reset --hard HEAD` + return to `main` + re-pop the stash.
- **Feature parity:** a new feature must consume **identical request/response shapes** across API, web, iOS, Android. Tile keys / capability strings must be **byte-identical** across clients (e.g. `"chat"`, `"announce"`, `"payForms"`). Route new contract shapes through `api-contract-broker`.

## Per-repo feature footprint (what a new feature "Foo" touches)

### API (`ritchies-platform-api`) — express-mongo-architect

**Required layering: route → controller → service → model.** This is a deliberate standard for new features, NOT "match whatever the nearest file does" — several older controllers (e.g. `announcementsController`) still inline business logic and must not be copied as-is. Every new API feature separates concerns:

1. `src/models/Foo.ts` — `export interface IFoo extends Document` + `new Schema<IFoo>({…}, { timestamps: true })` + `export const Foo = mongoose.model<IFoo>('Foo', schema)`. Enums for unions; `select: false` for secrets; instance methods on `schema.methods`; hooks via `schema.pre`.
2. `src/services/fooService.ts` — **the business logic lives here**, as an **HTTP-agnostic function module** matching the repo's existing service style (`chat/service.ts`, `auth/resolver.ts`, `auth/audit.ts`): exported `async function`s (NOT a class), taking plain args (ids, validated input DTOs, a resolved `EffectiveAccess`), returning plain data / domain objects, and `throw`ing typed errors that the central `errorHandler` maps. All Mongoose queries, cross-entity orchestration, visibility/scoping rules, and serialization go here. This layer must be unit-testable **without** supertest.
3. `src/controllers/fooController.ts` — **THIN.** Each named `async (req, res, next)` handler only: (a) `const input = fooCreateSchema.parse(req.body)` with an **inline zod** schema, (b) resolve access context (`req.access ??= await resolveAccess(user)`), (c) call `fooService.*`, (d) shape the HTTP response (`res.status(201).json(...)`), (e) `try/catch → next(error)`. No Mongoose queries, no business rules, no cross-entity logic in the controller. Import `'../types'` for augmented `req.user`/`req.access`.
4. `src/routes/foo.ts` — `const router = Router()`; `authenticate` + capability guards per route; `export default router`.
5. `src/app.ts` — add `import fooRoutes from './routes/foo'` + one `app.use('/api', fooRoutes)` line. **`errorHandler` must stay LAST.**
6. Tests — `src/services/__tests__/fooService.test.ts` (direct unit tests of the service against `mongodb-memory-server`, the cheap fast layer) **and** `src/controllers/__tests__/foo.routes.test.ts` (Jest + supertest, following `announcements.routes.test.ts`: seed access model, `bearer(user)` helper, 401/403/happy-path). The service split exists precisely so most logic is covered by fast service unit tests, with route tests asserting wiring + auth.

**Reuse services before writing new ones (check-before-building).** Before adding logic, grep `src/services/`, `src/auth/`, and `src/chat/` for something that already does it, and import it rather than re-implementing:
- `auth/resolver` → `resolveAccess`, `can`, `summarizeAccess` (four-pillar access; the ONLY way to compute access/capabilities).
- `auth/audit` → `recordAudit`, `commitCatalogChange` (any admin/catalog mutation must audit; catalog edits must go through `commitCatalogChange` to clear the live cache).
- `auth/authTokens`, `auth/teamMembership`, `auth/delegation`, `auth/passwordResetDelivery` (pluggable delivery seam).
- `chat/service` (`visibleGroups`, `visibleGroupIds`, `colorForUser`, `initialsOf`), `chat/membership` (`isMember`), `chat/reads`, `chat/hub` (`broadcast`).

**Extract shared logic into a reusable service instead of duplicating.** If a helper is (or would be) used by two or more domains, put it in `src/services/` and import it everywhere. The backend-api-skills refactor has landed: `announcementsService.ts` and `src/services/format.ts` (`initialsOf`, `relativeTime`) now exist — import them, never re-inline or re-duplicate what they hold.

- **Grandfathered domains:** `src/auth/` and `src/chat/` are pre-existing domain-module services — keep importing from them (do not relocate them). New per-domain services follow `src/services/<domain>Service.ts`.
- **Auth/capability guards — two middlewares, different jobs.** `requireCapabilityHeld(key)` / `requireAnyCapabilityHeld(...keys)` gate a route on the caller merely HOLDING a capability (what tasks and calendar routes use); `requireCapability(key)` is for global caps with the same held semantics but an older shape. **Scoped** caps are checked in the service/handler via `resolveAccess`/`can` once the resource is loaded — a route-level guard cannot know the resource.
- **Boot-time OPTIONAL work must never be able to stop the server starting.** The
  API's entrypoint runs env-gated conveniences (`CATALOG_SYNC_ON_BOOT`,
  `COMPANY_LINKS_SEED_ON_BOOT`, `LEVEL_USERS_SEED_ON_BOOT`) before
  `server.listen()`, inside a `try` whose `catch` calls `process.exit(1)`. Run one
  of those bare and a throw from optional work kills required work — the whole
  API 503s, every route, not just the feature being seeded.

  This is not hypothetical: `LEVEL_USERS_SEED_ON_BOOT` on an environment with
  `NODE_ENV=production` hit the seeder's own refusal to use a default password
  there — a CORRECT guard — and took dev down. Wrap every such flag so the
  failure is loud in the log and the server still listens (`bootSeed` in
  `src/index.ts`). Note the company-links flag carried the identical fault for
  days and had simply never thrown; that is the version nobody finds by reading
  an incident report.

  Corollary for the flags themselves: **dev runs `NODE_ENV=production`.** A guard
  keyed on that fires there. Go THROUGH it (set `SEED_LEVEL_PASSWORD`) rather
  than around it.

- **A new capability key is a deploy step, not just code.** Adding one to the catalog means `npm run catalog:sync` must run against every environment before any preset holds it — a feature that "doesn't work for level 1" after deploy is usually this.
- **The server authors the verdict, clients render it.** Access ladders (chat create tiers), admin-role provenance (`adminSource`), and "can this selection be created" answers are computed server-side and shipped in envelopes (`can`, `create`, `admin`, preview verdicts with server-authored reason strings). Clients never decode an access level or re-implement a ladder — the chat create ladder is deliberately non-monotonic and three clients must say the same thing.
- **Auth flows are enumeration-safe.** `forgot-password` answers 200 with the same copy whether or not the account exists; keep that property in any new auth-adjacent surface and mirror the server's copy in clients.
- **A policy readout and the policy enforcement must read the same store.** Any envelope that reports a limit, an allow-list or a toggle state to clients has to compute it from the persisted rules the enforcement path reads — never from module constants that happen to match them. `chatAdminLimits()` shipped a hardcoded `25 MB / "Any file type"` to the L7 admin screen while `checkUpload` was already reading the stored per-category rules, so the screen stated a number the server did not enforce. Making the readout `async` and cascading `await` up through the envelope builder is the correct cost, not a reason to keep the constant.
- **The transport ceiling must be ≥ the largest value an admin can set.** Multer's `fileSize` aborts the request *before any handler runs*, so a route-level bound below the admin-settable maximum turns a legal upload into an opaque 413 instead of the real 422 and its message. `MAX_ATTACHMENT_BYTES` is the outer bound only; the meaningful limits are the stored per-category ones. Pin it with a test (`expect(MAX_ATTACHMENT_BYTES).toBeGreaterThanOrEqual(MAX_SETTABLE_BYTES)`).
- **Server-authored enum values never join the client-submittable enum.** `blocked_word` is a reason only the auto-filter may write, so it stays out of `REPORT_REASONS` and out of the clients' `ChatReportReason` — otherwise the report endpoint can be used to forge an auto-flag. Clients render its label in their own `reasonLabel` map. Apply the same split to any status or reason the server alone is allowed to author.
- **Derive on read rather than back-fill an immutable log**, and take the paging cursor from **rows read, not rows kept** — a filtered page that yields zero entries must still advance the cursor, or "load older" stalls forever on a quiet category.
- **A moderation snapshot is frozen.** Actions that delete or edit the underlying record (chat's Remove) must leave the report's embedded `messageSnapshot` untouched — what the reviewer is reading is evidence, and it is a separate subdocument precisely so the action cannot blank it.
- **Singleton settings documents upsert atomically** — `findOneAndUpdate(…, { upsert: true, new: true })`, never find-then-create — or two admins saving at once race a duplicate.
- CI gates every merge on **typecheck + jest + build** (Azure App Service, on push to `main`). A scaffolded feature MUST pass all three.

### Web (`ritchies-platform-web`) — react-architect
State = React Context (`AuthContext`) + local `useState`/`useEffect`. Data = **axios only** (no react-query). One `lib/<domain>.ts` = API client + types + domain helpers.
1. `src/lib/foo.ts` — async fns using the shared `api` axios instance (`api.get<{foo: Foo[]}>('/api/foo').then(r => r.data.foo)`), plus TS types and pure derived helpers. Components never call `api` directly.
2. `src/pages/app/FooPage.tsx` (or `pages/admin/…`) — default-export component; `const { capabilities } = useAppContext()`; local `useState`; `load()` + `useEffect(load, [])` with `.then/.catch/.finally`.
3. `src/App.tsx` — import the page, add `<Route path="foo" element={<FooPage/>}/>` under `/app` (or `/admin`); wrap in `<RequireCapability capability="foo.x">` if gated; add the capability key to `src/lib/access.ts`.
4. Reuse: `Button` (`@/components/ui/button`, brand `cva` variants), admin `components/admin/primitives.tsx` (`Panel`/`Field`/`TextInput`/`Table`), `<Can>` for conditional UI, `lucide-react` icons, `sonner` toasts via `apiErrorMessage()`. Small chips/badges are defined inline at the bottom of the page file.
- **Brand tokens:** `src/index.css` holds the CSS custom properties (`--brand-blue` = **`#0039A6`**, PMS 286 — sourced from the Ritchies Corporate Style Guide V15, same as both mobile apps) + `tailwind.config.js` (`brand.*`, `paper.*`, `ink.*`). **Accent colours come from `src/lib/brand.ts`** (`BRAND.*`, guide-sourced, each with a citation comment) and the foreground on any accent is `onBrand(color)` — derived from WCAG contrast, never assumed white; half the guide's palette fails white text. Avatars use `personColor()`. Brand marks are components in `src/components/Logo.tsx` (`Wordmark`, `RMark`, `FriendliestBadge`) — never `<img>` a deleted asset name; `src/lib/brandAssets.test.ts` enforces the invariants and will fail the build of anyone who reintroduces a clipped badge or the wrong blue.
- **The settled-result idiom is lint-enforced.** `react-hooks/set-state-in-effect` (an ERROR here) rejects synchronous `setState` inside effects. Loading is DERIVED: store `{query…, data}` results, compute `const settled = result && result.key === currentKey ? result : null`, `loading = settled === null`, `failed = settled?.data === null`. Copy `ChatSearchPage`/`CalendarPage`. Resets belong in event handlers, never at the top of an effect.
- **Tests exist: vitest** (`npm test`), node environment, `src/**/*.test.ts` — pure-logic tests in `src/lib/` (see `announcements.test.ts`, `brandAssets.test.ts`). New domain logic in `lib/` should get one. `npm run typecheck` and `npm run lint` (0 errors expected) both work. CI = Azure Static Web Apps, **build only** — the other gates run locally, so run them yourself before shipping.

### iOS (`ritchies-mobile`) — swift-architect
SwiftUI, iOS 26, **Observation** (`@Observable`, never `ObservableObject`). Organized by feature folder. `actor APIClient.shared`.
1. `Ritchies/Features/<Name>/<Name>View.swift` — `struct <Name>View: View` taking `level: AccessLevel`, `capabilities: Set<String>`, `onClose: () -> Void`; owns `@State private var store = <Name>Store()`; wrap in `NavigationStack`; `.task { await store.load() }`.
2. `…/<Name>Model.swift` — `@MainActor @Observable final class <Name>Store` with `private let api = APIClient.shared`, `load()` → `try await api.request("api/<name>")`; plus domain struct, `…DTO` wire types with `init(dto:)`, `…Response` envelopes, `enum Sample…` data.
3. Wire in `Features/Home/DashboardView.swift`: add a `DashboardFeature` case + an INLINE `case .foo:` branch in the feature switch (modules render inline with a `.move(edge: .trailing)` transition — NOT `fullScreenCover`; a comment in the file explains why), route it in `open(_:)`, **and add the key to `DashboardView.wiredTiles`** — that set lives beside the router on purpose, and everything outside it renders muted with a SOON chip. Add/gate the tile in `Features/Dashboard/AccessModel.swift` (`TileKey` + capability gate).
4. Add a Swift Testing decoding test in `RitchiesTests/` (unit = **Swift Testing** `@Test`/`#expect`; UI = XCTest). Sources are path-globbed — run `xcodegen generate` after adding files; no `project.yml` edit needed.
- **Theme:** `DesignSystem/Theme.swift` — `enum Theme` with `Colors`/`Spacing`/`Radius`. Brand blue = **`#0039A6`** (`Theme.Colors.brand`). Accents come from `Theme.Colors.Accent` (guide-sourced, cited per line) and the foreground on any accent is `Theme.Colors.on(_:)` (WCAG-derived — never hardcode `.white` on an accent). Header gradients use `Theme.Colors.headerGradientTop/Bottom` (brand → #00266E), the same pair web and Android render. Light-only (`.preferredColorScheme(.light)`). No custom fonts bundled. "Liquid Glass" here = the frosted `GlassCard` (`.ultraThinMaterial`) in `Components.swift`; navigation is a level-driven grid with inline feature rendering, **no `TabView`**.
- List screens are `.refreshable`; failure states show the message AND a visible "Try again" button (a gesture is not a discoverable retry). `App/Connectivity.swift` provides the offline banner; `Features/Account/AccountView.swift` is the account surface.
- **Non-JSON response bodies go through `APIClient.requestRaw(_:method:query:body:authorized:retryOn401:) -> Data`** (added for chat export downloads). It mirrors `request`'s single-flight 401 refresh and error-body handling and only skips the decode — do not hand-roll a `URLSession` call for a CSV or file body and lose the refresh.
- **Never `async let` two calls against a `@MainActor` store.** Both hop to the same actor anyway so there is no concurrency to win, and the capture is non-`Sendable`. Sequential `await`s.
- No Xcode toolchain guarantee here → often "compile-checked only." Be honest: **this repo has never been compiled on this machine in its history**, so a PR must say "compile-checked only, never built" rather than anything that implies a build or a simulator run happened.

### Android (`ritchies-android`) — kotlin-compose-architect
See `agents/kotlin-compose-architect.md` for the full grain. Summary: `object ApiClient` (OkHttp, string paths, no Retrofit); `remember { <Feature>Store(rememberCoroutineScope()) }` state holders (no Hilt/Koin/repository); `when`-driven nav wired into `DashboardScreen` (no Navigation-Compose) — **and add the `TileKey` to `WiredTiles` beside the router**, or the tile renders muted with a SOON chip; `@Serializable` DTO + domain + `toDomain()` co-located in one `<Feature>Model.kt`; stores expose `isLoading` + `loadFailed`; screens take `(level, capabilities, onClose)`.
- **Shared chrome in `ui/common/`** — use it, never re-grow it inline: `AppChrome.kt` (root snackbar via `LocalAppSnackbar` + offline banner), `Refresh.kt` (`RefreshableContent` = pull-to-refresh with snackbar on failure — wrap every list screen, `onRefresh` returns success), `AttachmentSources.kt` (the one chooser + EXIF-aware transcode for camera/gallery/files — do not copy it into a feature), `Polling.kt`.
- **Theme:** `ui/theme/Theme.kt` — Brand **`#0039A6`**, accents from the `Accent` object (guide-sourced, cited per line), foreground via `onAccent()` (WCAG-derived), header gradient `HeaderTop`/`HeaderBottom` (brand → #00266E). Light-only.
- **Tests + CI exist**: JUnit4 local unit tests in `app/src/test/` run by `gradle testDebugUnitTest`, gated in `.github/workflows/android-build.yml`. `FragmentVersionTest` guards the androidx.fragment version floor — **never remove the `constraints { implementation(libs.androidx.fragment) }` block in `app/build.gradle.kts`**; fragment ≤1.2.x crashes the app at launch (16-bit request-code guard vs ActivityResult codes).
- **A `@Composable` never returns a value and never mutates state during composition.** A menu/picker helper takes `onSelect: (T) -> Unit`; one written to return `T?` and clear its own state mid-composition compiles cleanly and then misbehaves at runtime.
- **`ApiClient.requestRawBytes(path, method, bodyJson, authorized): ByteArray`** is the byte-body counterpart of iOS's `requestRaw` (exports, downloads). Sharing a file written from it also needs a matching `<cache-path>` entry in `res/xml/file_paths.xml` — FileProvider throws at runtime if the directory is not declared.
- Builds locally with SDK + JDK17 (`~/.gradle/jdks/eclipse_adoptium-17-amd64-linux.2`) + gradle-8.9 (`~/tools/gradle-8.9`); machine-default JDK 25 breaks the Kotlin compiler.

## Brand standard (aligned 2026-08-20 — do not regress it)

The authority is the **Ritchies Corporate Branding & Style Guide V15 (Jul 2024)**, public at
<https://www.ritchies.com.au/application/files/1817/5824/7810/Ritchies_Style_Guide_V15_Jul2479.pdf>.
The historic web/mobile blue split is RESOLVED: every client uses **`#0039A6`** (PMS 286). Rules a feature must not break:

- **Marks.** The logo is the all-caps RITCHIES **wordmark, supplied as artwork, never typeset** (guide p9 forbids typing it, lower case, outlines, stretching). Small/square chrome uses the **R mark** (white R on brand blue, guide p46). The **friendliest-team badge is a culture asset, not a logo**: sign-in heroes and splashes only, ≥56px, and the artwork is a full-bleed alpha disc — never clip it round or put a ring/backing behind it. Web components: `Wordmark`/`RMark`/`FriendliestBadge` in `components/Logo.tsx`; iOS `RitchiesWordmark` imageset; Android `ritchies_wordmark.xml` drawable.
- **The badge already contains the word RITCHIES**, so it must never share a surface with the wordmark — showing both says the brand name twice, and *that*, not its size, is what makes it read as out of place. If a screen adopts the badge, the wordmark comes off that screen. Two further constraints follow from the artwork itself: it is a black disc, so on brand blue it reads as a hole punched in the header, and below ~56px its brush script stops resolving into words.
- **Reading a colour off the guide PDF is a trap.** The PDF's artwork renders through a CMYK separation, so the R mark measures `#1A4295` on screen; the guide's own p16 **screen** spec (`#0039A6`) is the authority and wins. Verified 2026-08-24 — web `ritchies-r-mark.svg`, the iOS `AppIcon`, web's favicon/apple-touch-icon and Android's adaptive icon are all compliant; the Android foreground's R sits inside the circular mask's safe radius, and `ic_notification.xml` is correctly the R glyph **alone with no blue square** (a notification icon is a tinted mask, so a filled square renders as a solid blob).
- **Accents are sourced, not invented.** Each client carries the same guide-sourced palette (web `lib/brand.ts` BRAND, iOS `Theme.Colors.Accent`, Android `Accent`) with per-line citations. Never eyeball a new hex; if a colour is missing, take it from the guide (pp16-18 spec'd RGB first, p54 Commitments artwork second) and add it to all three with the citation.
- **Foreground is computed, never assumed.** `onBrand()` / `Theme.Colors.on()` / `onAccent()` pick white or ink by WCAG contrast. Half the guide's palette fails white text; hardcoding `.white` on an accent is a bug.
- **A feature header is FOUR things, and a design export gives you none of them.** The **gradient** is brand → `#00266E` (web `from-brand to-brand-800`, iOS `headerGradientTop/Bottom`, Android `HeaderTop/HeaderBottom`). The **navigation controls** are the shared glass ones (iOS `FloatingGlassButton`, Android `ChatIconButton`), never a hand-rolled button on a `white.opacity(0.15)` plate. The **extent** runs behind the status bar: iOS `.ignoresSafeArea(edges: .top)` with the header's own `.padding(.top, 56)`, Android `WindowInsets.statusBars` added to the header padding. The **behaviour**, on iOS only, is that the header SCROLLS AWAY and hands its title to a pinned glass capsule — `CollapsingHeader.compactOpacity(scrollY:)` fading the large 26pt title out, `CollapsedTitle` fading in, both reading the same curve so they can never drift apart. Android pins every header and has no collapsing pattern at all, so parity here means *not* porting it.
- **One ScrollView per screen, and the header goes inside it.** The collapse above is structural, not decorative: a pinned header above content that scrolls underneath can never leave, and a vertical scroll nested in a vertical scroll gives the inner one the whole gesture and the outer one none. So the screen owns ONE `ScrollView { VStack { header; content } }` with `.onScrollGeometryChange` feeding `scrollY`, `.overlay(alignment: .top) { floatingBar }`, `.headerOverscrollCap(...)` and `.refreshable` — and every list it renders, including ones inside child views like an admin panel, is a bare `LazyVStack`. Two things break when a view moves inside that scroll: **`Spacer()` stops centring** (there is no free space to expand into, so a Spacer pair collapses to nothing — use `.frame(maxWidth: .infinity, minHeight: 240)` on loading and empty states), and **anything in the header scrolls off**, so a control pinned in the floating bar that targets a field in the header must `scrollTo` it first. Copy an EXISTING feature (Chat is the reference), never the mock.
- **A flat overscroll cap forces a VERTICAL header gradient.** iOS fills the area above a scrolling header with `headerOverscrollCap(_:)`, which paints ONE FLAT colour. A flat block can only match a top edge that is a single colour, so a `.topLeading → .bottomTrailing` gradient — whose top edge is a horizontal ramp already well toward `#00266E` on the right — meets it at a visible seam the moment anyone pulls down. Header gradients are therefore `startPoint: .top, endPoint: .bottom` on iOS, and the cap colour MUST be the gradient's own first stop (check it: Announcements and the chat admin panel use their own colour pairs, not the tokens). This is iOS-only: Android pins its headers and Compose overscroll is a stretch rather than a fill, and web has no cap.
- **A missing modifier has no line in a diff.** Company Links shipped with all four wrong and each one was invisible to code review and to the compiler: it read as complete, because absence has no diff. Every one was caught by a person or a screenshot looking at the rendered screen. Any change to a screen's chrome gets rendered and looked at before it is called done — diffing it against a reference file is not the same check.
- **Honesty rules.** Tiles/nav rows that don't route render muted with a SOON tag, driven by the wiring set that lives BESIDE each client's router (web `!!tile.to`, iOS `DashboardView.wiredTiles`, Android `WiredTiles`) — wire the router and the set together. **Never put an invented number in a tile sub or badge**; counts come from the API or don't appear.
- **Type: Avenir Next on every client** (decision 2026-08-20, sourced from Impact Connect's licensed files). Web self-hosts four woff2 weights (`src/assets/fonts/`, zero third-party font requests; `brandAssets.test.ts` fails any serif or Fraunces). Android bundles TTFs in `res/font/` and delivers the family through `RitchiesTypography` on `MaterialTheme` — per-call `fontSize`/`fontWeight` keep working and pick the right file. iOS bundles nothing (Apple ships the family): use **`Theme.font(size, weight)`** for ALL text, never `.font(.system(size:))` — but SF Symbols DO stay on `.system` fonts (a symbol takes its stroke weight from the font; Avenir carries no symbol strokes), and monospaced-digit surfaces (device pairing codes) stay `.system(design: .monospaced)` with a comment.
- **Residual mismatch to flag, not fix silently:** the mobile splash radial `#1257CE` is consistent across both apps but not guide-sourced. If a feature touches it, surface the decision.

## UX state standard (all clients)

Every data-driven screen distinguishes **loading, failed, and genuinely empty** — three states that must never share one blank pane — and a failure offers a visible **Try again** control (pull-to-refresh alone is not a discoverable retry). **A failure must never masquerade as an empty result** (web encodes this as `data: null` = failed vs `[]` = empty). List screens refresh: `.refreshable` on iOS, `RefreshableContent` on Android. Transient errors on Android go to `LocalAppSnackbar`; web uses sonner. All three clients already mount a passive offline banner at root (web `OfflineBanner`, iOS `ConnectivityMonitor`, Android `AppChrome`) — do not add per-screen connectivity handling.

- **Pin the strings that answer a question.** An empty state, a disabled-control
  explanation, a "someone else does this" note — copy that does real work looks
  like filler to anyone who did not have the question it answers, so it is the
  copy most likely to be deleted as noise by a later tidy-up. Assert it in a
  test. The Company Links empty queue is the worked example: "Nothing awaiting
  approval" plus "Proposed changes appear here for a second administrator" is
  what stops a phone that cannot author from reading as a dead end, and it was
  confirmed to land by someone opening that screen cold. One sentence carrying
  the whole maker-checker split is exactly the sentence that looks removable.

## Test conventions (all four repos have tests — keep each green)

| Repo | Framework | Run | CI-gated |
|------|-----------|-----|----------|
| API | Jest (service unit + supertest routes) | `npm test` | yes (typecheck + jest + build) |
| Web | Vitest, node env, `src/**/*.test.ts` | `npm test` (+ `typecheck`, `lint` — 0 errors) | no (SWA builds only) — run locally |
| iOS | Swift Testing (unit), XCTest (UI) | needs a Mac | no |
| Android | JUnit4 in `app/src/test/` | `gradle testDebugUnitTest` | yes (`android-build.yml`) |

New API logic gets a service unit test; new web `lib/` logic gets a vitest file; guard-style invariant tests (`brandAssets.test.ts`, `FragmentVersionTest`) protect deliberate decisions — extend them rather than deleting them when they fail.

## Design-doc convention

Each repo uses `.autofeature/designs/<feature>-<YYYY-MM-DD>.md` for the Feature Brief and `.autofeature/checkpoints/<YYYYMMDD-HHMMSS>-impl.md` for impl checkpoints (the API repo already has six prior runs). Follow it in every repo the feature touches.

## Brand skill

`~/.claude/skills/ritchies-design/` is the working design system (tokens, extracted guide artwork in `assets/`, the Commitments palette in `assets/commitments/`); where it and the corporate style guide disagree, the guide wins. For any new UI surface, prefer its primitives/tokens over designing fresh. Australian English throughout. Lucide stroke icons (web) / SF Symbols (iOS) / Material icons (Android), no emoji as UI iconography.
