---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/24/2026
Version: 0.2
tags:
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'm working on C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Splitting the server up into components.md. The code base that I am trying to split into components is at C:\Users\mason\source\repos\knottyyoga. I'm going to create a new project that will be a C++ web server using Crow very similar to the webserver at this path with an angular frontend also very similar to the angular webserver at this path. I will want to use the components that Splitting the server up into components is creating in this project. I probably should also componentize the angular frontend eventually but lets focus on the server for now.

Besides the components that we will reuse, I want a server pretty similar to the existing server with a testhelper, scheduled jobs process, gtest unit tests, a webserver, a database helper. I want a similar db schema directory, create_database.cpp, and rough layered architecture like the existing sever: database schema, table helpers / util code, business logic, and endpoints with Google tests to validate each component.

The basic idea of the website is that we will build out a gay community site. We will have a component that scans the Internet using AI at known locations and gathers events that are upcoming and places them into the database with images, description, location, date, etc. The site will then show upcoming events on the website and have a calendar. Eventually, we will expand functionality to let people post events with admin approval and let people search for events. We will also eventually do advertising.

For now, I want to focus on standing up a minimal server that we can start adding functionality to.

# Current State & Inherited Context (re-grounded 7/24/2026)

> Conventions: checkboxes checked off as implemented; numbered subsections inside each phase; lower layers first; tests for everything testable; open questions live in this document. **Division of labor (adopted from the tenancy project, 7/21/2026): Claude builds and runs all C++ suites in the Linux docker clients as the per-slice gate; Mason does occasional Windows/VS verification and ALL git writes.** CommunityFinder gets its own docker client in Phase 1 to join that workflow.

## What changed since v0.1 of this document (7/11 → 7/24)

- **The extraction is done.** [[Splitting the server up into components]] Phases 3–4 completed: the server components live standalone at **`github.com/honuware/server_components`** (checkout `C:\Users\mason\source\repos\server_components`), GitHub Actions CI green (Linux `gcc:14.2.0` + Postgres 13.1 service, test-count floor). knottyyoga consumes honuware purely via **CMake FetchContent pinned to a SHA** (currently `8df437d6d8533b92d68ee54205be613e51202f03`); its local `components/` tree is deleted; cross-repo co-dev runs on `FETCHCONTENT_SOURCE_DIR_HONUWARE`. The splitting plan's Phase 5 example server stays demoted — **CommunityFinder is the proving consumer**. Its Phase 4.4 (Conan packaging) remains deferred.
- **Multi-tenancy landed in honuware** ([[Converting the server to a multi tenant Saas architecture]] Phases 0–7 complete 7/21–7/23; only Phase 8, the AWS wiring, is open): `tenants` control DB (`honuware_control`) + `ControlDbTenantResolver`; **`FixedTenantResolver`** for zero-ceremony single-tenant consumers (its header comment names CommunityFinder as exactly this case); `TenantResolutionGuard` middleware + `X-Honuware-Site`; per-tenant secrets/mail/resources; `ProvisionTenant` + `--create_tenant` + `MigrateAllTenants`; multi-tenant scheduler fan-out; **`GET /api/site_info`** + knottyyoga's `SiteConfigService` pattern. The `HONUWARE_DB_*` and `HONUWARE_VERSION` renames are done (legacy `KNOTTYYOGA_*` fallbacks remain honored).
- **The frontend components shipped** ([[Componentizing the frontend]] Phases 1–4 complete): **`@honuware/ui@0.1.1`** published to public npm with provenance (repo `github.com/honuware/web_components`, checkout `C:\Users\mason\source\repos\honuware-web-components`), **Angular 21**, eight entry points. knottyyoga's `ui/` consumes the exact-pinned package. Frontend Phase 5 (showcase + "CommunityFinder consumption + docs") is open — this project *is* its 5.2.
- **Net effect:** v0.1's Phase 0 gate ("honuware exists as a consumable repo") is **SATISFIED**, and the old sequencing note ("start pre-tenancy, adopt `FixedTenantResolver` later via version bump") is obsolete in the good direction — tenancy is already there, and CommunityFinder starts on it in fixed mode.

## What the platform provides vs. what this app authors (server)

**Free from honuware** (7 targets: `honuware_foundation`, `honuware_data`, `honuware_services`, `honuware_square`, `honuware_platform`, `honuware_testing`, `honuware_tests`; build-enforced layer DAG):

- **Endpoints:** the generic CRUD + admin-metadata suite (`/api/add_item`, `/api/add_item_fetch_primary_key`, `/api/update_item`, `/api/delete_item/…`, `/api/get_table_rows/…`, `/api/get_filtered_table_rows`, `/api/get_row/…`, `/api/get_row_by_values`, `/api/get_rows_by_column/…`, `/api/get_db_schema`, `/api/get_fk_options`, `/api/resolve_fk_display`) = the free admin data editor backend; `GET /api/health` (`{status, db, version}`); `GET /api/site_info`. All self-anchored via `Endpoints::RegisterFrameworkEndpoints()`.
- **Middleware:** `CloudFrontOriginGuard → TenantResolutionGuard → CookieParser → CORSHandler → CsrfGuard → SecurityHeaders`. CSRF = `csrft` cookie + `X-CSRF-Token` header, with `/api/login`, `/api/register`, `/api/remember`, `/api/verify` bootstrap-exempt.
- **Auth business logic (not the HTTP endpoints — see below):** `Auth::Session` (`InitializeFromLogin` / `FromCookie` / `FromDeviceToken`, `IsAdmin`, `ActiveUserHasRole/Permission`), `AuthHelper` (Argon2id), `LoginGate` (lockout), `CookieManager(+Factory)`, `PersonHelper`, `PersonVerifyMail`, `QuickAccountHelper` + welcome mail, `TokenCleanupHelper`, service accounts (`EnsureSchedulerServiceAccount`, parameterized email/domain), `EndpointAuthHelper` (per-request façade: session, `RequirePermission`, tenant accessors, `GetService<T>`).
- **Services & data:** secrets with at-rest encryption (`config_secrets`), `Mail::TenantBranding` / `LoadSenderAddress` / `LoadTenantBranding`, mail via mailio, images business logic + resize, migration runner with namespaced framework/app streams, generic DB access + schema metadata + stored procedures.
- **Tenancy:** everything listed above, fixed and control modes.
- **Testing:** `GlobalDatabaseTestSupport` (create-once tables, per-test aborted transactions, DB name taken from the injected `DatabaseInfo`), `EndpointTestHelper` (installs a default fixed test tenant `test-site`; **fabricates authenticated sessions in-transaction, so permission-gated endpoints are testable before any login endpoint exists**), service doubles, matchers. The whole `honuware_tests` bag also runs inside the consumer's suite against the composed schema — free regression coverage.
- **32 framework tables:** `people` (**the identity table — not `users`**), `sessions`, `device_tokens`, `email_verifications`, `login_attempts`, `auth_events`, `roles`, `permissions`, `role_permissions`, `role_assignments`, `permission_implications`, `config_secrets`, `idempotency_keys`, `schema_migrations`, `allowed_tables`, `admin_alerts`, the 11 other `admin_*` metadata tables, and the photo set (`photo_instances`, `source_photos`, `scaled_photos`, `photo_support_tables`, `table_item_photos`); plus the separate control-plane `tenants` (+ its own `schema_migrations`) via `MakeControlDatabaseInfo`.

**Correction to v0.1 — NOT framework (app-authored today, in knottyyoga's `src/endpoints/`):** the account/user/photo HTTP endpoints — `/api/register`, `/api/verify` (+ account activation), `/api/login`, `/api/logout`, `/api/me`, `/api/remember`, `/api/get_user_info`, `/api/set_user_info`, `/api/update_user_password`, photo upload / user-photo / `get_scaled_photo` / delete / has — are thin app-side shims over the framework business logic. `@honuware/ui`'s `AuthHttpAccess`/`PhotoHttpAccess` call **exactly these routes**, and the framework CSRF guard already allow-lists them, so they are framework-shaped in everything but location. Work item **H8** proposes extracting them into `honuware_platform`; the fallback is copying knottyyoga's files. Also app-side: the scheduler **engine** (`knotty_yoga_scheduler`, foundation-only, comment-marked "future honuware_scheduler" — **H9**), the admin-alerts digest endpoint, and all of `create_database.cpp`'s CREATE+POPULATE execution (**H2**, still a 3,018-line monolith in knottyyoga).

## The consumer checklist (what CommunityFinder writes — knottyyoga as-built reference)

| Seam | Reference in knottyyoga (`server/knottyyoga_server/`) |
|---|---|
| **Server composition root**: `PORT` env, at-rest secrets bootstrap (`HONUWARE_SECRET_KEY` + `MigrateSecretsToEncrypted`), `MailHelper`, `CookieManagerFactory`, column redactions, **tenant mode switch** (`HONUWARE_TENANT_MODE`; default fixed → `FixedTenantResolver` + `TenantResourceRegistry` with an app factory), `RegisterAllEndpoints()` + `RoutingBase::AddRoutes`, `ServerConfig::Initialize/ValidateProdEnvironment` | `src/main.cpp` (223 lines; CF = this minus Square + `SetServerBanner`) |
| **Endpoint anchor table**: `Endpoints::RegisterAllEndpoints()` → call platform's `RegisterFrameworkEndpoints()` + anchor app endpoints. **Anchors MUST use the volatile-store pattern** — plain unused `g_*` pointers are dead-stripped at `-O2` and every route 404s in Release (proven in the extraction shakeout) | `src/endpoints/web_app.cpp` (515 lines; knottyyoga still enumerates framework endpoints itself because it predates `RegisterFrameworkEndpoints` — CF calls the platform function instead) |
| **App config headers**: `App::kDatabaseName`; app secret key names; `App::FillInAppSecretDefaults(+String)` (brand values for framework keys: `mail_sender_name/address`, activation + admin-alert subjects, `website_address(_login)` dev/prod, `site_logo_url`); `App::kServiceAccountEmailDomain` + `kSchedulerServiceAccountEmail`; iCal identity | `src/business_logic/app_database_config.h`, `app_secret_keys.h`, `app_secret_values.{h,cpp}`, `app_service_account_config.h`, `app_ical_config.h` |
| **Schema composition**: `MakeAppTables()`; `MakeDatabaseInfo(name)` = `MakeFrameworkTables` ++ app | `src/db_schema/make_app_tables.*`, `make_database_info.*` |
| **Seed**: `--recreate_database` (gated by `HONUWARE_ALLOW_DESTRUCTIVE`), `--migrate`, `--create_tenant`; `CreateAndPopulateDatabases()` — after **H2** this composes platform's `CreateFrameworkTables`/`PopulateFrameworkTables` + a small app half | `src/database_helper/main.cpp`, `create_database.cpp` (the pre-split monolith H2 fixes; `CreateSchemaAndPopulate(databaseHelper, databaseInfo)` already exists as the extracted by-database callable) |
| **Migrations**: empty app stream + framework++app composition (separate id namespaces `honuware/`, `app/`) | `src/business_logic/migration/app_migrations.*`, `all_migrations.*` |
| **App endpoint auth helper** (typed app services; trivial for CF at MVP) | `src/endpoints/app_endpoint_auth_helper.*` |
| **Tenant resources factory**: CF has no per-tenant extras → thin factory over `Tenancy::PopulateBaseTenantResources` (knottyyoga's adds Square) | `src/business_logic/app_tenant_resources.*` |
| **Test main**: app-side `kTestDatabaseName = "test_communityfinder"` (**do NOT reuse the harness's lingering legacy default `test_knottyyoga`**), `RegisterAllEndpoints()`, `Secrets::Test::RegisterAppSecretDefaults(App::FillInAppSecretDefaultsString)`, `Initialize(MakeDatabaseInfo(kTestDatabaseName))` with `MakeTenantsTable` composed on top (the tenancy tests in `honuware_tests` need it) | `test/src/main.cpp` (46 lines) |
| **Build**: `conanfile.py` with honuware's package set (below) + `init()`-generated `ConanLibImports.cmake`; `conan_provider.cmake`; `CMakeSettings.json` (x64-Debug Ninja + `FETCHCONTENT_SOURCE_DIR_HONUWARE` cache var for co-dev); top-level `CMakeLists.txt`: `include(ConanLibImports)` → `FetchContent(honuware @ SHA)` → app-superset `cmake/honuware_layering.cmake` → targets; `CMAKE_LINK_LIBRARIES_ONLY_TARGETS ON`; the `if(POLICY CMP0167)` guard | `CMakeLists.txt`, `conanfile.py`, `CMakeSettings.json` |

**Conan set** (honuware's, verbatim — no shared list exists yet, see H6): abseil 20220623.1, boost 1.86.0, crowcpp-crow 1.3.2, date 3.0.4, gtest 1.12.1, libcurl 7.86.0, libjpeg 9e, libpng 1.6.40, libpqxx 7.10.5, libsodium 1.0.20, libtiff 4.6.0, mailio 0.25.3, openssl 3.5.2, zlib 1.3.1. (`ftxui`/`replxx` only if the test_helper TUI happens — Q10.)

## @honuware/ui as-built (client)

`@honuware/ui@0.1.1`, **Angular 21** (peers `^21.0.0`; note Angular 22 waits on Node ≥ 22.22.3 per knottyyoga), selector prefix `hw-`, eight entry points, exact-pin install (`npm install @honuware/ui@0.1.1 --save-exact`); cross-repo co-dev via the commented tsconfig `paths` override at the sibling checkout.

- **`/access`** — the trimmed seams: `CrudAccess` (~13 methods), `AuthAccess` (9: register/verify/login/logout/me/remember/getUserInfo/setUserInfo/updateUserPassword), `PhotoAccess` (4: uploadPhoto/uploadUserPhoto/deletePhoto/hasPhoto); DI tokens `HONUWARE_{CRUD,AUTH,PHOTO}_ACCESS` (default → `HonuwareAccessProxy`, which **serializes requests** — the server allows one transaction per session) + `HONUWARE_API_BASE` (default `/api`); `provideHonuwareAccess({mode:'http'|'mock'})`; HTTP impls (`withCredentials: true`); `PhotoUrlBuilder` (`/api/get_scaled_photo/{table}/{id}/{w}/{h}`); `CsrfInterceptor` (double-submit `csrft` → `X-CSRF-Token`); `ErrorService` + `ProblemDetails`; all schema/admin types.
- **`/auth`** — `AuthService` (`authData$` with roles/permissions/isAdmin/mustChangePassword), **`tryTokenLoginInitializer`** (silent login: `me()` → on 401 `remember()` → `me()` — the session-token + device-token persistent login), guards (`AuthGuard`, `AdminGuard`, `NoAuthGuard`, …), `ErrorInterceptor`, pages `hw-login` / `hw-register` / `hw-verify`, and the **`AUTH_ROUTES` token** (defaults are knottyyoga-flavored — CF overrides paths + returnUrl allowlist).
- **`/crud`** — `DatabaseSchemaService` (driven wholly by `/api/get_db_schema`, refreshes on auth change), `TableViewPage/TableEditPage/TableNewPage` + `TableViewControl` (pagination, sort, FK display resolution, child-table drilldown, photo thumbnails, delete-confirm), **`CRUD_EDITOR_ROUTES` token** (default basePath `/admin/tables`). *(Note: v0.1 mentioned a `TableManagementService` — it doesn't exist; `DatabaseSchemaService` is the service.)*
- **`/controls`** — text/long-text/bool/enum/date field controls, `FkPickerComponent`, `CompositeControlComponent`. **`/photos`** — `PhotoUploadComponent` (`userMode` avatar path, `deferUpload`, client-side resize). **`/foundation`** — `ToastService`, `ConfirmDialogComponent`, animations, date utils. **`/square`** — skipped for CF. **`/testing`** — `MockCrudAccess` (schema-driven in-memory store), `MockAuthAccess` (session + remember-token simulator incl. `expireSession()`), `MockPhotoAccess`, `provideHonuwareAccessMock()`.
- **Gaps CF builds app-side** (knottyyoga reference in parens): header + logged-in **user chip** (`shared/components/header/` + `HeaderService`), **`SiteConfigService`** + a `getSiteInfo` access method (`core/services/site-config.service.ts` — the runtime-branding seam), account/profile + change-password pages (`pages/account/…`; the `mustChangePassword` flow needs the page), admin dashboard/menu (`pages/admin/dashboard/`), an **Angular Material theme** (`mat.theme` — the library renders unstyled without one), the CF domain access superset (events calls), and permission-gated menus (read `authData`; no `*hasPermission` directive exists).
- **knottyyoga wiring reference** (`ui/src/app/app.config.ts`): `provideHttpClient(withInterceptorsFromDi())`; `CsrfInterceptor` + `ErrorInterceptor` (`HTTP_INTERCEPTORS`, multi); APP_INITIALIZERs = `tryTokenLoginInitializer` + `SiteConfigService.load()`; access tokens `useExisting` its legacy proxy (CF just uses the library default instead). Its `angular.json` **`local` configuration swaps the network access for the mock, and plain `ng serve` defaults to `local`** — zero-backend UI development; `serve:development` proxies `/api` to the server.

## Dev environment facts (corrected)

- **Shared Postgres 13.1 container** `knotty-postgres-docker` on docker network `knotty-net`, **host port 5432** (compose maps `5432:5432` — knottyyoga's CLAUDE.md/SERVER.md still say 5400; the compose file is authoritative). In-network alias `postgresql` is the framework's Linux default host, so docker clients need no DB env vars. Databases coexist happily: `knottyyoga`, `test_knottyyoga`, `honuware_test`, `docker`, tenant DBs — CF adds `communityfinder` + `test_communityfinder` (+ per-community DBs in Phase 14). Test DBs are DROP/CREATEd per run — **never run the same suite from Windows and Linux at once**.
- **Ports:** knottyyoga server 18080 / ui 4200 → CommunityFinder server **18081** / ui **4201** so both stacks run side-by-side (Q6).
- **The cross-repo loop** (tenancy plan Phase 0, applies verbatim to CF's honuware work): edit both trees with the `/honuware` mount + `FETCHCONTENT_SOURCE_DIR_HONUWARE`; per-slice gate = both docker suites green; at each **⇑ bump point** [Mason] pushes honuware → CI green → re-pin `GIT_TAG` → [Claude] verifies the pinned SHA on a clean volume (no override) → [Mason] commits both repos.

# Key Decisions

## D1 — Sequencing (RESOLVED)

The v0.1 gate is satisfied; S3 happened: CommunityFinder is the real second consumer and now **drives the remaining cross-repo items** (ledger below) through the established co-dev loop. Standing rule: **extract, don't copy** — when CF needs something generic that lives app-side in knottyyoga (account endpoints, scheduler engine, seed halves), the default is moving it into honuware with knottyyoga's suite as the regression gate; copies are a tracked fallback only.

## D2 — Repo layout, hosting, naming (Q1/Q2 still open)

Monorepo `communityfinder/{ui/, server/, database_server/}` (flattened — no `server/<name>_server/` nesting). GitHub. Working name **CommunityFinder**: repo `communityfinder`, libs `communityfinder_core` + `communityfinder_test_cases`, exes `communityfinder_server`, `communityfinder_database_helper`, `communityfinder_helper`, `communityfinder_tests`; DB `communityfinder` / `test_communityfinder`; service account `scheduler@communityfinder.local`. Rename before Phase 5 stays cheap (~an hour, mechanical).

## D3 — Components consumed

Server: `communityfinder_core` PUBLIC-links `honuware_{foundation,data,services,platform}`; the test exe adds `honuware_testing` + `honuware_tests` + `communityfinder_test_cases`. **Skip `honuware_square`** (it still builds — `honuware_testing` links it — but nothing app-side uses it and it costs nothing extra; one-line link to add back for payments/ads). Client: `@honuware/ui` `foundation/access/auth/controls/photos/crud/testing`; skip `/square`.

## D4 — MVP events domain schema (unchanged, one addition)

1. **`venues`** — id BIGSERIAL PK, name, address/city/state/zip, url, description, lat/lng (nullable), status.
2. **`event_sources`** — the "known locations to scan": name, url, venue_id (nullable FK), kind (venue site / ticketing / social / aggregator), enabled, scan_hints, last_scanned_at, notes.
3. **`events`** — title, description, starts_at (timestamptz), ends_at (nullable), venue_id (nullable FK) + location_text, url, cost_text, source_id (nullable FK), external_key, origin (scanned/manual/user_submitted), status (pending/approved/rejected/archived), created_by (nullable FK people), created_at/updated_at. **`UNIQUE (source_id, external_key)`** is the ingestion idempotency anchor. Display template `{title} @ {starts_at}`. Images via framework photo tables + `table_item_photos` — no image columns.
4. **`event_categories`** + assignments — Q4a. **`scan_runs`** — deferred to Phase 13 (Q4b).
5. Recurrence: one row per occurrence; no rules at MVP.

**New:** under D8 each community is its own database (Model C), so the app schema stays completely tenancy-free — no `tenant_id` columns anywhere, exactly like knottyyoga's app tables.

## D5 — AI event scanner (decide at Phase 13; seam built in Phase 10)

Unchanged options: **A** in-process C++ scheduled job (raw HTTPS to the Claude API, structured outputs); **B** separate Python/TS worker on the official SDK logging in as the scheduler service account and POSTing `/api/admin/ingest_events` (my lean — prompt iteration speed); **C** Anthropic Managed-Agents scheduled deployment (post-deploy hosting option). Whatever wins talks to the one idempotent ingestion endpoint.

## D6 — Frontend timing (SUPERSEDED)

The client lands **immediately after the server skeleton** (Phase 3), per your MVP ladder — `@honuware/ui` makes it cheap, and mock mode means UI work never blocks on the server.

## D7 — Dev environment

Share the existing Postgres container (**host port 5432**, network `knotty-net`); no new compose project (a `database_server/README.md` pointer instead). Server dev port **18081**; `ng serve --port 4201` when both UIs run. CF gets its own Linux docker client (Phase 1.2) mirroring `server_components/docker/` + knottyyoga's `docker_project/`: bind-mounted source, honuware co-dev mount, `build_and_test.sh` with a test-count floor.

## D8 — Multi-community architecture (NEW — your multi-tenant ask)

**Communities-as-tenants on the honuware tenancy stack (Model C, database-per-community):**

- **Dev + launch:** fixed mode — `FixedTenantResolver`, zero new env vars, no control DB; site key defaults from app config (`HONUWARE_FIXED_SITE_KEY` optional). Fixed mode serves headerless requests and rejects a contradicting site header (400).
- **Multi-community:** `HONUWARE_TENANT_MODE=control` + `honuware_control`'s `tenants` rows (`--create_tenant` per community) + **one CloudFront distribution per community** injecting `X-Honuware-Site` — one server process, one database per community, **one shared SPA bundle**.
- **Site name & meta as DB fields (your ask):** per-community values live in the `tenants` row (`display_name`) + that community's `config_secrets` (framework keys: `mail_sender_name/address`, `website_address(_login)`, `site_logo_url`; app keys added in Phase 14: tagline, about, contact email, social links, city/region — final list Q8). Exposed via the framework **`GET /api/site_info`** (`display_name`, `website_url`, `logo_url`) plus an app **`GET /api/site_meta`** for the richer set; the SPA's `SiteConfigService` fetches at boot (APP_INITIALIZER) and falls back to bundled defaults.
- **CF adopts `SiteConfigService` + `/api/site_info` from day one** — a deliberate deviation from the tenancy doc's "CommunityFinder runs static branding" note, which predates your multi-community requirement. In fixed mode the endpoint simply serves the single community's values headerless, so the frontend is runtime-branded from the start and adding community #2 needs zero frontend work.
- Events/users are per-community for free (own DB); the app schema never learns about tenancy.

## D9 — Client server-access + mock strategy (NEW)

- Use the library seams as-is: the **default `HonuwareAccessProxy`** (CF has no legacy 243-method interface to bridge, unlike knottyyoga). Override only `AUTH_ROUTES` + `CRUD_EDITOR_ROUTES` for CF paths.
- CF domain access = a small **`CommunityAccess`** trio (interface + HTTP impl + mock) following the library's seam pattern: `getHealth()`, `getSiteInfo()`/`getSiteMeta()`, then the events queries.
- **Local-only test mode (your ask):** a `local` build configuration — and **plain `ng serve` defaults to it**, knottyyoga-style — that swaps in `provideHonuwareAccessMock(...)` + `MockCommunityAccess` via a DI/environment flag (cleaner than knottyyoga's file-replacement mechanics, same effect: the whole app runs with zero backend). Unit tests use the same mocks.

# Cross-Repo Work Ledger (honuware items this project owns or watches)

v0.1's hand-off items reconciled + new ones. Every **[hw]** item runs in the co-dev loop with knottyyoga's full suite as the regression gate, lands in honuware CI, then a re-pin.

| # | Item | Status |
|---|---|---|
| H1 | `RegisterFrameworkEndpoints()` in platform | **DONE** (`components/platform/endpoints/register_framework_endpoints.*`, volatile anchor pattern) |
| H2 | **`create_database` framework/app split**: platform gains `CreateFrameworkTables(...)` (framework DDL + indexes, FK order) + `PopulateFrameworkTables(...)` (framework tables' admin metadata, base roles/permissions, `allowed_tables` entries, framework + registered-app secret defaults, scheduler service account); consumer composes instead of copying ~3,000 seed lines | **OPEN — owned here** (re-scoped out of tenancy 5.1 on 7/23; `CreateSchemaAndPopulate(databaseHelper, databaseInfo)` exists as the extracted by-database callable to build on; delicate: `PopulateConfigSecrets/Roles/Permissions/AllowedTables` intermingle framework+app seed; **gate: knottyyoga `--recreate_database` yields an identical DB**) → Phase 0.1 |
| H3 | `kTestDatabaseName` out of the harness | **DONE in effect** — the harness resolves the DB name from the injected `DatabaseInfo`; the legacy `"test_knottyyoga"` default constant lingers (cosmetic cleanup, Phase 0.4) |
| H4 | Brand env vars parameterized | **DONE** — `HONUWARE_DB_*`, `HONUWARE_VERSION`, `HONUWARE_LOG_DEST/ALLOW_DESTRUCTIVE/ORIGIN_SECRET/TRUST_PROXY/DEV_CORS_ORIGIN/SECRET_KEY`, all with legacy fallback; service-account identity fully app-supplied |
| H5 | Secrets test-defaults hook | **DONE** (`Secrets::Test::RegisterAppSecretDefaults`) |
| H6 | Shared Conan requirements list | **OPEN (optional)** — versions live independently in each `conanfile.py`; drift is convention-guarded only → Phase 0.4 |
| H7 | Consumer docs in the honuware repo | **PARTIAL** — README FetchContent section + CLAUDE.md exist; no new-consumer checklist, no example server. CF writes the checklist from its own bootstrap → Phase 0.4 (+ the frontend plan's 5.2 quickstart) |
| H8 | **NEW — extract the generic account/user/photo endpoints into platform**: register, verify (+ account activation), login, logout, me, remember, get_user_info, set_user_info, update_user_password, photo upload / user-photo / get_scaled_photo / delete / has (candidate: the admin-alerts digest endpoint too) — thin TUs over framework logic, anchored in `RegisterFrameworkEndpoints`; knottyyoga deletes its copies | **OPEN — proposed, owned here** (Q3). The UI package already binds to these exact routes and the CSRF guard already allow-lists them. Fallback: copy knottyyoga's files into CF and leave H8 open (tracked divergence) → Phase 0.2 |
| H9 | **NEW — extract the scheduler engine → `honuware_scheduler`** (`components/scheduler/`, foundation-only side target like square): `scheduled_job` / `api_client` / `job_scheduler` / `scheduler` incl. the header-aware target key + login headers (multi-tenant-ready); knottyyoga's `knotty_yoga_scheduler` becomes a consumer (catalog + main stay app-side) | **OPEN — owned here**, needed by Phase 12 (the engine is explicitly "lift-ready") → Phase 0.3 |

# Target Architecture

## Repo layout

```
communityfinder/
├── CLAUDE.md                    conventions + planning-dir override → this vault (Phase 1.1)
├── ui/                          Angular 21 SPA
│   ├── package.json             @honuware/ui@0.1.1 exact-pinned
│   ├── angular.json             production / development / local(mock); serve default = local
│   ├── src/proxy.conf.json      /api → http://127.0.0.1:18081
│   └── src/app/
│       ├── app.config.ts        interceptors, initializers (silent login, SiteConfig), route/token overrides
│       ├── core/services/       site-config.service.ts, community-access/ (interface + http + mock)
│       ├── shared/components/   header (user chip + admin dropdown), footer
│       └── pages/               home, events/, account/, admin/, auth routes onto hw-* pages
├── server/
│   ├── conanfile.py             honuware's 14-package set (init() → ConanLibImports.cmake)
│   ├── conan_provider.cmake · CMakeSettings.json · CMakeUserPresets.json
│   ├── CMakeLists.txt           project, C++20, per-OS flags, CMP0167 guard, FetchContent(honuware @ SHA)
│   ├── cmake/honuware_layering.cmake   app-superset DAG
│   ├── docker/                  Linux build/test client (build_and_test.sh w/ count floor, co-dev mount)
│   ├── certs/cacert.pem
│   ├── src/
│   │   ├── main.cpp
│   │   ├── business_logic/      app_*.h/cpp seams · events/ · migration/
│   │   ├── db_schema/           venues, event_sources, events, … + make_app_tables, make_database_info
│   │   ├── sql_util/table_helpers/   Venues, EventSources, Events
│   │   ├── endpoints/           web_app.cpp anchors · app_endpoint_auth_helper · events endpoints
│   │   ├── database_helper/     main.cpp + create_database.*
│   │   └── scheduler/           communityfinder_job_catalog.* + main.cpp        (Phase 12)
│   └── test/                    CMakeLists.txt + src/main.cpp (tests live beside sources)
└── database_server/README.md    pointer to the shared knotty-net container (host 5432)
```

## CMake targets

```
FetchContent honuware @ pinned SHA → honuware_{foundation,data,services,square,platform,testing,tests}
communityfinder_core            app lib; PUBLIC-links foundation/data/services/platform
communityfinder_test_cases      app test bag (self-registering gtest TEST()s beside sources)
communityfinder_server          src/main.cpp → core
communityfinder_database_helper → core
communityfinder_helper          scheduler main + catalog → core + honuware_scheduler   (after H9)
communityfinder_tests           test/src/main.cpp → core + honuware_testing + communityfinder_test_cases + honuware_tests
```

## Environment variables (all framework names; defaults work in dev)

`PORT` (18081 dev) · `HONUWARE_DB_HOST/_PORT/_USER/_PASSWORD/_NAME/_SSLMODE/_SSLROOTCERT` · `HONUWARE_LOG_DEST` · `HONUWARE_ALLOW_DESTRUCTIVE` · `HONUWARE_SECRET_KEY` (prod) · `HONUWARE_ORIGIN_SECRET` (prod) · `HONUWARE_TRUST_PROXY` (prod) · `HONUWARE_DEV_CORS_ORIGIN` (dev alternative to the proxy) · `HONUWARE_VERSION` · `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` · `HONUWARE_TENANT_MODE` (unset = fixed; `control` for multi-community) · `HONUWARE_FIXED_SITE_KEY` (optional) · `HONUWARE_CONTROL_DB_NAME` (control mode, default `honuware_control`).

# Phased Implementation Plan

**MVP = Phases 1–3** (a booting platform server + a client that talks to it, both test-gated). Phases 4–9 are your build-out ladder, 10–13 the product, 14–15 scale-out and ship. Phase 0 items are cross-repo and interleave: **0.1 gates 2.5**, **0.2 gates Phase 4** (unless the copy fallback is taken), **0.3 gates Phase 12**, 0.4 trails.

## Phase 0 — Cross-repo prerequisites in honuware (the ledger's open items)

### 0.1 `create_database` framework/app split (H2) **[hw + knottyyoga gate]**
- [ ] Platform gains `CreateFrameworkTables(...)` + `PopulateFrameworkTables(...)` extracted from knottyyoga's `create_database.cpp` (framework DDL + indexes FK-ordered; framework tables' admin metadata; base roles/permissions incl. Administrator/User + `admin_portal`; framework `allowed_tables` entries; framework + registered-app secret defaults; scheduler service account via the parameterized `EnsureSchedulerServiceAccount`). Home: `components/platform/business_logic/` (beside `migration/`) with the DDL half near `db_schema/`.
- [ ] knottyyoga's `CreateAndPopulateDatabases()`/`CreateSchemaAndPopulate()` recomposed on the halves. **Gate: `--recreate_database` produces an identical database** (full suite against a recreated DB); both docker suites green; CI; push; re-pin.
- [ ] Framework-only stand-up test in honuware (its own test main + CI already exercise most of this).

### 0.2 Generic account/user/photo endpoints → platform (H8) **[hw + knottyyoga gate]** — or the copy fallback (Q3)
- [ ] Move/adapt into `components/platform/endpoints/`: register, verify, account activation, login, logout, me, remember, get_user_info, set_user_info, update_user_password, photo upload / user-photo / get_scaled_photo / delete / has (+ their HTTP-driven `_test.cpp`); anchor in `RegisterFrameworkEndpoints`; sweep residual brand strings through `TenantBranding`/secrets.
- [ ] knottyyoga deletes its copies; full-suite gate; push; re-pin. *(If deferred: CF copies these files in Phase 4 and H8 stays open — record the divergence here.)*

### 0.3 Scheduler engine → `honuware_scheduler` (H9) **[hw + knottyyoga gate]** — needed by Phase 12
- [ ] `components/scheduler/` foundation-only side target; move `scheduled_job` / `api_client` / `job_scheduler` / `scheduler` + tests; knottyyoga's `knotty_yoga_scheduler` consumes it (catalog + main stay app-side). Gate + re-pin.

### 0.4 Docs & hygiene (H6/H7, trailing)
- [ ] honuware README **new-consumer checklist** written from this bootstrap's experience; optional shared Conan requirements mechanism; legacy `kTestDatabaseName` default removed; the logged CMake-comment brand scrub.

## Phase 1 — Repo & dev-environment bootstrap

### 1.1 Repository
- [ ] Create the GitHub repo (name Q1, visibility/org Q2) with the monorepo skeleton, `.gitignore` (VS/CMake/Conan/node artifacts, `out/`, `build/`, `dist/`, `ConanLibImports.cmake`), **`.gitattributes` pinning LF for `*.sh`/`Dockerfile`/`*.yml` and CRLF for `*.cmd`** (the honuware clone lesson — CRLF shebangs break Linux), README stub.
- [ ] Root `CLAUDE.md` adapted from knottyyoga's: runtime layering (endpoints → business_logic → services → table_helpers → db_schema) + component layer order; thin-endpoint / KeyValueTable-at-boundaries rules; testing conventions (no fixtures, pre-created tables, HTTP via `handle_full`, **`ThreadPool::Shutdown()` before the next DB read**, never assume collection order); **the volatile endpoint-anchor convention** (the `-O2` dead-strip trap); Crow `HTTPMethod` PascalCase gotcha; `FormatString` + `NormalizeCrLf` mail rules; naming conventions; the env-var table above; the FetchContent/co-dev section; **planning-directory override → `C:\Users\mason\Documents\Obsidian\CommunityFinder\Claude\`** and the division-of-labor block (Claude: Linux docker builds/tests + read-only git; Mason: Windows spot-checks + all git writes).

### 1.2 Dev database + docker client
- [ ] `database_server/README.md` documenting the shared container (knotty-net, host 5432, alias `postgresql`, user/pass docker) — no new compose project (Q6).
- [ ] `server/docker/` Linux client mirroring `server_components/docker/`: `build_container.cmd`, `load_container.cmd <network>`, `build_and_test.sh` (conan + cmake + build + run with a test-count floor; `HONUWARE_SRC_DIR` co-dev mount override). Claude drives it non-interactively (the `docker run --rm … bash docker/build_and_test.sh` pattern from the tenancy plan's Phase 0).
- [ ] Verify `communityfinder` + `test_communityfinder` create and coexist alongside the existing databases.

## Phase 2 — Server skeleton boots (your MVP item 1)

### 2.1 Build system
- [ ] `conanfile.py` (the 14-package set; `init()` generates `ConanLibImports.cmake`; class `CommunityFinderRecipe`), `conan_provider.cmake`, `CMakeSettings.json` (x64-Debug Ninja; `FETCHCONTENT_SOURCE_DIR_HONUWARE` cache var), `CMakeUserPresets.json`.
- [ ] Top-level `CMakeLists.txt`: `project(communityfinder)`, C++20, per-OS flag blocks, `if(POLICY CMP0167)` guard, `CMAKE_LINK_LIBRARIES_ONLY_TARGETS ON`, `include(ConanLibImports.cmake)` → `FetchContent_Declare(honuware … GIT_TAG <current SHA>)` → app-superset `honuware_layering.cmake` → the target set from *Target Architecture*. First clean configure+build: [Claude] docker client, [Mason] Windows/VS.

### 2.2 App config seams
- [ ] `app_database_config.h` (`App::kDatabaseName = "communityfinder"`), `app_secret_keys.h` (empty scaffold + NOTE), `app_secret_values.{h,cpp}` (`FillInAppSecretDefaults(+String)`: sender name/address, activation + admin-alert subjects, `website_address` dev `http://localhost:18081/` / prod TBD (Q1/Q9), `website_address_login` dev `http://localhost:4201/`, `site_logo_url` ""), `app_service_account_config.h` (`@communityfinder.local`, `scheduler@communityfinder.local`), `app_ical_config.h` (`-//CommunityFinder//Events//EN`, UID domain). Tests mirroring knottyyoga's `app_secret_values_test.cpp`.

### 2.3 Schema & migration composition
- [ ] `make_app_tables.{h,cpp}` (empty body), `make_database_info.{h,cpp}` (`MakeFrameworkTables` ++ app), `app_migrations.{h,cpp}` (empty stream) + `all_migrations.{h,cpp}` (framework ++ app, separate id namespaces). Composition test: composed `DatabaseInfo` == framework set exactly; framework precedes app.

### 2.4 Endpoints anchor + composition root
- [ ] `endpoints/web_app.cpp`: `RegisterAllEndpoints()` = `Endpoints::RegisterFrameworkEndpoints()` + an (empty for now) app anchor block **using the volatile-store pattern**; minimal `app_endpoint_auth_helper`.
- [ ] `src/main.cpp`: `PORT` default 18081, production DB helper for `App::kDatabaseName`, secrets-at-rest bootstrap (+ `MigrateSecretsToEncrypted`), `MailHelper`, `CookieManagerFactory`, column redactions, `WebApp`, **fixed-mode tenancy wiring** (`FixedTenantResolver` + `TenantResourceRegistry` with a thin base-resources factory over `Tenancy::PopulateBaseTenantResources`), registration + `AddRoutes`, `ServerConfig::Initialize/ValidateProdEnvironment`. (knottyyoga's `main.cpp` minus Square and the server banner.)

### 2.5 database_helper + seed *(needs 0.1)*
- [ ] `database_helper/main.cpp`: `--recreate_database` (`HONUWARE_ALLOW_DESTRUCTIVE`-gated), `--migrate` (leave `--create_tenant` wiring to Phase 14).
- [ ] `create_database.cpp`: `CreateAndPopulateDatabases()` composing platform's `CreateFrameworkTables`/`PopulateFrameworkTables` + the app half (no app tables yet; seed first admin user(s) — Q1, scheduler service account, secrets defaults).

### 2.6 Test infrastructure + smoke tests
- [ ] `test/src/main.cpp`: app-side `kTestDatabaseName = "test_communityfinder"`, `RegisterAllEndpoints()`, `Secrets::Test::RegisterAppSecretDefaults(App::FillInAppSecretDefaultsString)`, `Initialize(MakeDatabaseInfo(kTestDatabaseName))` with `MakeTenantsTable` composed on top.
- [ ] Smoke tests over HTTP (harness-fabricated sessions — no login endpoint needed yet): `/api/health` 200 with version; generic CRUD add/get/delete on a framework table under an admin session; non-admin 403; CSRF guard behavior; `/api/site_info` returns the seeded branding headerless.
- **Phase gate:** server boots; full suite (**`honuware_tests` bag + smoke tests**) green in the docker client and on Windows.

## Phase 3 — Angular client skeleton with mock mode (your MVP item 2)

### 3.1 Workspace
- [ ] `ng new` (Angular 21, standalone, scss, routing) in `ui/`; `npm install @honuware/ui@0.1.1 --save-exact`; Angular Material + `mat.theme` setup (library requirement); eslint + karma headless.

### 3.2 Access + config wiring
- [ ] `app.config.ts`: `provideHttpClient(withInterceptorsFromDi())`, `CsrfInterceptor` + `ErrorInterceptor`, `tryTokenLoginInitializer`, `AUTH_ROUTES` + `CRUD_EDITOR_ROUTES` overrides (CF paths + returnUrl allowlist), library-default access providers (`HonuwareAccessProxy`; `HONUWARE_API_BASE` default `/api`).
- [ ] `CommunityAccess` (interface + HTTP + mock): `getHealth()`, `getSiteInfo()`. `SiteConfigService` ported from knottyyoga's pattern (APP_INITIALIZER, merge non-empty `site_info` fields over `DEFAULT_SITE_CONFIG`, never rejects) + spec.
- [ ] Environments dev/prod/local; **`local` configuration → `provideHonuwareAccessMock(...)` + `MockCommunityAccess`** (DI swap on an environment flag); **`ng serve` default = `local`** (zero backend); `serve:development` uses `proxy.conf.json` → `http://127.0.0.1:18081`; document `--port 4201` for side-by-side runs.

### 3.3 Shell + the dummy call
- [ ] Minimal shell: header (site name from `SiteConfigService`; login/register placeholders), footer, home page showing the **live `/api/health` payload + the community display name** — the "simple dummy call that gets shown".
- [ ] Unit tests: `SiteConfigService`, `CommunityAccess` mock, home component (mock providers).
- **Phase gate:** `ng serve` (mock) fully works offline; `ng serve -c development` renders health + site name from the running server; ui tests green.

## Phase 4 — Account creation & user info endpoints, server (your item 3; *needs 0.2 or the copy fallback*)

- [ ] The account/user/photo endpoints live in CF's build (via platform after 0.2 — then this phase is wiring + seeding — or as tracked copies): register / verify / account activation / login / logout / me / remember / get_user_info / set_user_info / update_user_password (+ the photo endpoints ahead of Phase 5).
- [ ] Mail path: dev uses the test/log mail path or real SMTP via `config_secrets`; activation links built from `website_address_login`.
- [ ] HTTP tests: register → verify → login → me; **remember-token silent re-login** (`device_tokens` — your persistent-login item); `set_user_info` roundtrip; `update_user_password` incl. `must_change_password`; `LoginGate` lockout sanity; CSRF enforced on POSTs (bootstrap exemptions honored); **`get_user_info` returns roles + permissions** (your "fetch roles and permissions" item, server side).
- **Phase gate:** the full auth loop green over HTTP in the suite.

## Phase 5 — Account creation & user profile UI (your item 4)

- [ ] Routes `/login` `/register` `/verify` → `hw-login`/`hw-register`/`hw-verify`; guards wired (`NoAuthGuard` on auth pages; `AuthGuard` elsewhere).
- [ ] **Header user state — name + photo in the upper right like knottyyoga:** user chip with avatar via `PhotoUrlBuilder.scaledPhotoUrl('people', personId, 300, 300)` + dropdown (My account, Admin — permission-gated, Logout); modeled on knottyyoga's `header` + `HeaderService`.
- [ ] **User portal:** account page to view/edit first name, last name, email (`AuthService.setUserInfo`); change-password page (incl. the `mustChangePassword` redirect flow); **photo upload** via `hw-photo-upload` `userMode`.
- [ ] Component specs with `provideHonuwareAccessMock` (`MockAuthAccess` covers silent-login + `expireSession`); full mock-mode parity (register/login/portal all work offline).
- **Phase gate:** signup → verify → login → edit profile → photo → logout, in the browser against the dev server AND in mock mode.

## Phase 6 — Generic CRUD endpoints, server (your item 5)

- [ ] Confirm `PopulateFrameworkTables` seeded `allowed_tables` + admin metadata + `admin_table_permissions` for the framework tables an admin should edit (`people`, `roles`, `permissions`, `role_assignments`, `role_permissions`, `permission_implications`, `admin_alerts`; `config_secrets` exposure is Q8c).
- [ ] App-side HTTP tests: admin session can `add_item`/`get_row`/`update_item`/`delete_item` on a representative table; non-admin 403; column redactions hide `password_hash`; `get_db_schema` shape sanity.
- **Phase gate:** the generic CRUD suite green under permission gating.

## Phase 7 — Admin CRUD editor in the client (your item 6)

- [ ] `/admin` dashboard page (root-tables list from `DatabaseSchemaService`, friendly names) + `CRUD_EDITOR_ROUTES` mounting `TableViewPage`/`TableEditPage`/`TableNewPage` under `/admin/tables`; `AdminGuard` (+ `admin_portal` permission from `authData`).
- [ ] **The admin dropdown** in the header, shown only for admin/`admin_portal` — your "admin dropdown with all the CRUD support" item.
- [ ] Specs with `MockCrudAccess` (schema-driven); mock mode ships a small fake schema so the editor works offline.
- **Phase gate:** edit `people`/`roles` rows from the browser; FK pickers + display templates resolve.

## Phase 8 — Roles & permissions management, server (your item 7)

- [ ] Verify-and-test: `roles`/`permissions`/`role_permissions`/`role_assignments`/`permission_implications` manageable via the generic CRUD under the right permission; assignment changes reflect in `get_user_info` (implications resolved); non-admin cannot self-assign (escalation guard tests).
- [ ] Define + seed CF's permission catalog beyond the framework base: `manage_events` (Phase 10), `manage_site_meta` (Phase 14) — in the `create_database` app half.
- **Phase gate:** assign/revoke a role via endpoints; permissions propagate to sessions.

## Phase 9 — Admin portal for roles & permissions (your item 8)

- [ ] The CRUD editor over `roles`/`permissions`/`role_assignments` **is** the admin portal mechanism (knottyyoga parity — no dedicated role endpoints exist anywhere); ensure display templates make `role_assignments` human-readable (person email + role name); document the assign-role flow.
- [ ] Optional stretch: a bespoke "User roles" page (pick person → toggle roles) only if the raw editor feels clunky.
- **Phase gate:** grant Administrator to a second user through the UI; their next login shows the admin dropdown.

## Phase 10 — Events domain, server (the product starts)

*(D4 schema. knottyyoga's CLAUDE.md "Adding a New Database Table" registration checklist — db_schema pair, make_database_info, CreateTables, admin_top_level/nested_tables, admin_table_permissions, column data info, friendly names, display templates, CMakeLists — applies to every table here; forgetting the allowed/nested-tables step is the classic CRUD-endpoint 403.)*

### 10.1 db_schema + seed
- [ ] `venues`, `event_sources`, `events` (+ `event_categories` pair per Q4a) with `Create*Indexes` (events: `(status, starts_at)`, `UNIQUE(source_id, external_key)`); `make_app_tables` FK order venues → event_sources → events; `create_database` app half: DDL order, allowed_tables, admin metadata, `manage_events` permission + role wiring, dev seed data. Schema tests per house pattern.

### 10.2 table_helpers
- [ ] `Venues`, `EventSources`, `Events` (ctor takes `DatabaseHelper`, methods take `Transaction&`, `KeyValueTable` at boundaries): `GetEventsInRange(from, to, status)`, `GetBySourceAndExternalKey`, `GetPendingEvents`. Tests each.

### 10.3 business_logic/events
- [ ] `EventHelper`: `IngestEvents(batch)` — per-item upsert on `(source_id, external_key)` (insert as `pending`+`scanned`; re-scan policy per Q4c); `Approve/Reject`; visibility rule (public = approved ∧ ¬archived). Tests: idempotent re-ingestion, transitions, visibility.

### 10.4 endpoints
- [ ] Public `GET /api/events/upcoming?from&to` (approved only, range-clamped, venue joined) + `GET /api/event?id`; admin `POST /api/admin/ingest_events` (`manage_events`-gated batch; per-item created/updated/unchanged). Anchors (volatile) + HTTP tests; one end-to-end event-photo test via `table_item_photos`.
- **Phase gate:** a CRUD-entered event → approved (status edit) → appears in `/api/events/upcoming`; ingestion idempotent; suite green.

## Phase 11 — Public events UI

- [ ] `CommunityAccess` events methods (+ mocks with seeded fake events — the offline demo); home = upcoming list grouped by day (venue + photo); a simple month calendar view; event detail page; category filter if Q4a. Specs throughout.
- **Phase gate:** the full manual loop in the browser: enter (admin CRUD) → approve → see it on the public page and calendar — no scanner needed.

## Phase 12 — Scheduled-jobs process (*needs 0.3/H9*)

- [ ] `scheduler/communityfinder_job_catalog.{h,cpp}`: `BuildCommunityFinderJobs(target, intervals)` — `archive_past_events`, `notify_admin_alerts`, + a **disabled-by-default** `run_event_scan` placeholder; validation + password resolution mirroring knottyyoga's catalog; catalog tests.
- [ ] The admin endpoints behind them (`archive_past_events` logic + the admin-alerts digest — per the H8 note these may ride the extraction or be app-authored) + tests.
- [ ] `scheduler/main.cpp` (ABSL flags; `server_url` default `http://localhost:18081`; `SCHEDULER_SERVICE_ACCOUNT_PASSWORD`), linking `honuware_scheduler`.
- **Phase gate:** `communityfinder_helper` authenticates as the service account and runs the archive job on its interval.

## Phase 13 — AI event scanner (design on arrival; Q5)

Sketch unchanged: per-source scan → Claude (web fetch/search tools + structured outputs producing `[{title, starts_at, ends_at?, description, url, external_key, …}]`) → `POST /api/admin/ingest_events` → events land `pending` → admin approves in the CRUD UI. At this point add: `scan_runs` audit table, `event_sources.scan_hints` usage, `run_event_scan` job enablement. Follow-ups: cross-source dedup, image capture, geo. Architecture decision A/B/C (D5) made here.

## Phase 14 — Multi-community enablement (D8)

- [ ] Local proof: `--create_tenant` a second community (site_key, db_name, display_name) → run the server with `HONUWARE_TENANT_MODE=control` → requests with each `X-Honuware-Site`: isolated events/users, distinct `/api/site_info` branding. (honuware's tenancy tests gate the mechanics; this is CF-level verification.)
- [ ] **Site meta as DB fields** (your ask, beyond site_info's three): app secret keys (`site_tagline`, `site_about_html`, `site_contact_email`, `site_social_links` JSON, `site_city_label`, … — final list Q8b) + defaults in `FillInAppSecretDefaults`; **`GET /api/site_meta`** app endpoint (public, cacheable, tenant-resolved) + tests; `SiteConfigService` merges site_info + site_meta; per-community editing story (Q8c).
- [ ] `database_helper` `--create_tenant` wired (ProvisionTenant + the composed create/populate callable) + control-mode `--migrate` (all active communities).
- [ ] Scheduler control mode: catalog × active communities with per-community `X-Honuware-Site` headers (knottyyoga's `BuildMultiTenant…` pattern; `--tenant_refresh_interval`).
- **Phase gate:** two communities served by one local server process and one SPA bundle, with distinct branding and fully isolated data.

## Phase 15 — CI & deployment

- [ ] GitHub Actions: server job cloned from `server_components/.github/workflows/ci.yml` (gcc:14.2.0 container, postgres:13.1 service, Conan cache, test-count floor, `HONUWARE_DB_*`); ui job (npm ci, lint, headless tests, prod build). Branch protection when the first collaborator lands (Q11).
- [ ] `package/`: Dockerfile + `build_linux_release.sh` adapted from knottyyoga (git present in the builder for the FetchContent clone; one image, all binaries, one `HONUWARE_VERSION`).
- [ ] AWS per knottyyoga's `Deploying to AWS.md` conventions: EC2/ECS + RDS, S3 + CloudFront **per community** with `X-Honuware-Site` + shared `X-Origin-Secret` origin headers, ACM/Route 53; `server.env` with `HONUWARE_*` names; fixed mode while single-community, control mode when community #2 onboards (Q9).

# Open Questions

Please answer inline (house style). Recommendations included so you can just say "agreed".

1. **Product/brand name + seed admins.** Is **CommunityFinder** the real name or a working title? It drives repo/DB/target names, mail sender, website secrets, service-account domain, UI branding, and the eventual domain. And which admin email(s) should `create_database` seed (knottyyoga seeds masonbendixen@gmail.com)? *(Rec: proceed with CommunityFinder as the working name; renaming before Phase 5 is ~an hour of mechanical work.)*
	- Mason- CommunityFinder works. Can you seed masonbendixen@gmail.com (Mason Bendixen), ljkuhn33@gmail.com (Levi Kuhn), mr.calebault@gmail.com (Caleb Ault). Please make all administrators.
2. **GitHub visibility + org.** Private personal repo (my rec — this is the product app, unlike the public honuware repos; friends can be invited as collaborators), or public / under the `honuware` org?
	- Mason- My friends working on this really want to be able to list this on resumes. The knottyyoga app is pretty proprietary but I'm okay with world read access to this code base.
3. **H8 — extract vs copy the account/user/photo endpoints.** My rec: extract into `honuware_platform` in Phase 0.2 — `@honuware/ui` already binds to those exact routes, every future consumer needs them, and the CSRF guard already knows their paths. Fallback: copy knottyyoga's files into CF to unblock Phase 4 and leave H8 open.
	- Mason- I'll go with your recommendation.
4. **Events schema scope.** (a) `event_categories` now? *(rec: yes — trivial, and the UI filter wants it early)*; (b) `scan_runs` deferred to Phase 13? *(rec: yes)*; (c) re-ingestion policy when an approved event's source page changes *(rec: update minor fields silently; revert to `pending` only when date/venue changes)*.
	- Mason- Yes, I very much want to be able to categorize events. I want things like bar, pride, volunteer, sports, theatre, protest, etc.
5. **Scanner architecture (D5).** No decision needed until Phase 13 — but a standing preference between in-process C++ (A), a separate Python/TS worker on the official SDK (B, my lean), or an Anthropic Managed-Agents scheduled deployment (C) shapes small things earlier (how much scan config lives in `event_sources`).
	- Mason- I think it should be a separate process. I'm on the fence about C++ versus Python. I'm currently taking a ML class doing the examples in Python and know a lot of the support for AI / ML is Python based but I'm more comfortable with C++ and it is higher performance. I think by planning a separate process regardless, it let's us buffer a bit on making that decision.
6. **Dev environment.** Confirm: share the knottyyoga Postgres container (host port **5432** — compose is authoritative; the 5400 in knottyyoga's docs is stale), CF server on **18081**, `ng serve --port 4201` for side-by-side. *(Rec: yes to all.)*
	- Mason- I'll go with your recommendation.
7. **Public self-registration at launch.** The endpoints exist regardless; do we show Register in the UI from day one, or hide it (effectively invitation-only) until user-submitted events arrive? *(Rec: hide the link initially; flip it on with the user-submission feature.)*
8. **Multi-community details (D8).** (a) Confirm communities-as-tenants (Model C, one CloudFront distribution per community, one shared bundle). (b) Day-one site-meta field list *(rec: display name, logo, tagline, about, contact email, social links; add city/region when community #2 exists)*. (c) How per-community meta gets edited *(rec: a small "Site settings" admin page in Phase 14 — generic CRUD over `config_secrets` is awkward with at-rest encryption; until then, values are seeded/set via the helper)*.
9. **Deploy timing + domain.** CF can deploy single-community fixed-mode any time after Phase 11; control mode only when community #2 onboards. When do you want the domain name bought (it feeds `website_address` prod values and SES setup)?
10. **test_helper TUI.** Include a `communityfinder_test_helper` (ftxui/replxx REPL) from the start, or defer? *(Rec: defer — the admin CRUD editor covers manual data needs; keeps two Conan deps out.)*
11. **Collaborators.** Will the friends work in the CF repo itself (affects Q2, branch protection at Phase 15, and a LICENSE/contribution note), or only on honuware?
12. **First job catalog scope.** Beyond `archive_past_events` + `notify_admin_alerts`, should Phase 12 mirror any of knottyyoga's generic jobs (e.g. token cleanup via `TokenCleanupHelper`)? *(Rec: yes — adopt the generic hygiene jobs from knottyyoga's catalog wholesale.)*

# Suggested Additions (things a public community site will want that weren't on your list)

- **Legal + trust:** privacy policy / terms pages; cookie note. Needed before open registration.
- **Email deliverability:** SES (or provider) setup + SPF/DKIM on the domain before real verification mails; until then dev uses the test mail path.
- **iCal export** for events — nearly free: foundation's `ICalGenerator` + `app_ical_config.h`; "Add to calendar" on the event page.
- **SEO/social cards** for event pages: meta + `og:image` from the scaled-photo pipeline; sitemap + robots.txt. (No SSR needed at MVP.)
- **Admin alerts wired early** (framework `admin_alerts` + the digest job) — your ops eye once anything runs unattended.
- **Rate limiting posture:** `LoginGate` covers auth; consider per-IP limits on the public events endpoints + ingest before deploy.
- **Content moderation plan** for user-submitted events (ties to the framework quick-accounts feature when that phase comes).
- **Demo/seed dataset** for mock mode and dev (doubles as screenshots/marketing material).
- **Ops:** RDS backup/restore runbook; favicon/logo asset (drives `site_logo_url`); privacy-friendly analytics.

# Change Log

- **7/24/2026 (v0.2)** — Re-grounded to as-built reality per the (now-removed) Mason Update section: extraction complete (`server_components` live + CI green; knottyyoga on FetchContent @ `8df437d`), tenancy Phases 0–7 landed in honuware (`FixedTenantResolver`/control mode, `/api/site_info`, `HONUWARE_DB_*`), `@honuware/ui@0.1.1` published (Angular 21). Corrected v0.1 assumptions: account/user/photo HTTP endpoints are app-authored (new H8 proposes extraction), the scheduler engine was never extracted (new H9), the `create_database` split is owned here (H2, re-scoped out of tenancy 7/23), DB host port is 5432. Replaced the phase plan with the MVP ladder (Phases 0–15 with checkboxes), added D8 multi-community + D9 client access/mock strategy, rebuilt Open Questions, added Suggested Additions.
- **7/11/2026 (v0.1)** — Initial plan against the pre-extraction componentization state.
