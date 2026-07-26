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

**Extract shared logic into a reusable service instead of duplicating.** If a helper is (or would be) used by two or more domains, put it in `src/services/` and import it everywhere. Known extract-candidates the architect should factor out when it touches them: `announcementsController`'s inlined `serialize` / `visibleTo` / `receiptsFor` → an `announcementsService`; and `initialsOf` / `relativeTime`, which are **already duplicated** between `announcementsController` and `chat/service.ts` → a shared `src/services/format.ts`. Do not add a third copy.

- **Grandfathered domains:** `src/auth/` and `src/chat/` are pre-existing domain-module services — keep importing from them (do not relocate them). New per-domain services follow `src/services/<domain>Service.ts`.
- **Auth/capability guards:** `authenticate` (per-route). `requireCapability(key)` / `requireAnyCapability` for **global** caps; **scoped** caps checked in the service/handler via `resolveAccess`/`can` (NOT `requireCapability`, which needs a resource and 403s scoped caps).
- CI gates every merge on **typecheck + jest + build** (Azure App Service, on push to `main`). A scaffolded feature MUST pass all three.

### Web (`ritchies-platform-web`) — react-architect
State = React Context (`AuthContext`) + local `useState`/`useEffect`. Data = **axios only** (no react-query). One `lib/<domain>.ts` = API client + types + domain helpers.
1. `src/lib/foo.ts` — async fns using the shared `api` axios instance (`api.get<{foo: Foo[]}>('/api/foo').then(r => r.data.foo)`), plus TS types and pure derived helpers. Components never call `api` directly.
2. `src/pages/app/FooPage.tsx` (or `pages/admin/…`) — default-export component; `const { capabilities } = useAppContext()`; local `useState`; `load()` + `useEffect(load, [])` with `.then/.catch/.finally`.
3. `src/App.tsx` — import the page, add `<Route path="foo" element={<FooPage/>}/>` under `/app` (or `/admin`); wrap in `<RequireCapability capability="foo.x">` if gated; add the capability key to `src/lib/access.ts`.
4. Reuse: `Button` (`@/components/ui/button`, brand `cva` variants), admin `components/admin/primitives.tsx` (`Panel`/`Field`/`TextInput`/`Table`), `<Can>` for conditional UI, `lucide-react` icons, `sonner` toasts via `apiErrorMessage()`. Small chips/badges are defined inline at the bottom of the page file.
- **Brand tokens:** `src/index.css` (`--brand-blue` = 231° HSL ≈ **`#002491`**) + `tailwind.config.js` (`brand.*`, `paper.*`, `ink.*`, fonts Inter/Fraunces). Note this differs from mobile — see the brand-colour caveat below.
- **No tests, no test script.** Match that (or introduce vitest only if the Brief explicitly asks). CI = Azure Static Web Apps, **build only** (no lint/test/typecheck gate).

### iOS (`ritchies-mobile`) — swift-architect
SwiftUI, iOS 26, **Observation** (`@Observable`, never `ObservableObject`). Organized by feature folder. `actor APIClient.shared`.
1. `Ritchies/Features/<Name>/<Name>View.swift` — `struct <Name>View: View` taking `level: AccessLevel`, `capabilities: Set<String>`, `onClose: () -> Void`; owns `@State private var store = <Name>Store()`; wrap in `NavigationStack`; `.task { await store.load() }`.
2. `…/<Name>Model.swift` — `@MainActor @Observable final class <Name>Store` with `private let api = APIClient.shared`, `load()` → `try await api.request("api/<name>")`; plus domain struct, `…DTO` wire types with `init(dto:)`, `…Response` envelopes, `enum Sample…` data.
3. Wire in `Features/Home/DashboardView.swift`: add a `DashboardFeature` case + a `.fullScreenCover(item:)` branch; add/gate a tile in `Features/Dashboard/AccessModel.swift` (`TileKey` + capability gate).
4. Add a Swift Testing decoding test in `RitchiesTests/` (unit = **Swift Testing** `@Test`/`#expect`; UI = XCTest). Sources are path-globbed — run `xcodegen generate` after adding files; no `project.yml` edit needed.
- **Theme:** `DesignSystem/Theme.swift` — `enum Theme` with `Colors`/`Spacing`/`Radius`. Brand blue = **`#0039A6`** (`Theme.Colors.brand`/`accent`). Light-only (`.preferredColorScheme(.light)`). No custom fonts bundled. "Liquid Glass" here = the frosted `GlassCard` (`.ultraThinMaterial`) in `Components.swift`; navigation is a level-driven grid + `fullScreenCover`, **no `TabView`**.
- No Xcode toolchain guarantee here → often "compile-checked only." Be honest.

### Android (`ritchies-android`) — kotlin-compose-architect
See `agents/kotlin-compose-architect.md` for the full grain. Summary: `object ApiClient` (OkHttp, string paths, no Retrofit); `remember { <Feature>Store(rememberCoroutineScope()) }` state holders (no Hilt/Koin/repository); `when`-driven nav wired into `DashboardScreen` (no Navigation-Compose); `@Serializable` DTO + domain + `toDomain()` co-located in one `<Feature>Model.kt`; screens take `(level, capabilities, onClose)`; theme `ui/theme/Theme.kt` (Brand **`#0039A6`**, light-only). Mirror the iOS slice field-for-field. Builds locally with SDK + JDK17 + gradle-8.9. No tests currently.

## Two cross-repo caveats to surface, not paper over

1. **Brand blue disagrees.** Web uses `#002491`; both mobile apps use `#0039A6`. Neither mobile app bundles the Fraunces/Inter fonts the design system specifies. If a feature touches brand chrome, flag the mismatch rather than silently picking one.
2. **Test coverage is asymmetric.** API (Jest, CI-gated) and iOS (Swift Testing) have tests; web and Android have none. Default: keep the API test convention, and match the no-test convention on web/Android unless the Brief asks otherwise. Don't quietly introduce a test framework into a repo that has none.

## Design-doc convention

Each repo uses `.autofeature/designs/<feature>-<YYYY-MM-DD>.md` for the Feature Brief and `.autofeature/checkpoints/<YYYYMMDD-HHMMSS>-impl.md` for impl checkpoints (the API repo already has six prior runs). Follow it in every repo the feature touches.

## Brand skill

`~/.claude/skills/ritchies-design/` is the source of truth for brand. For any new UI surface, prefer its primitives/tokens over designing fresh. Australian English throughout. Lucide stroke icons, no emoji as UI iconography.
