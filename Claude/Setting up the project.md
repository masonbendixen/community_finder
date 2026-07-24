---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/11/2026
Version: 0.1
tags:
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'm working on C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Splitting the server up into components.md. The code base that I am trying to split into components is at C:\Users\mason\source\repos\knottyyoga. I'm going to create a new project that will be a C++ web server using Crow very similar to the webserver at this path with an angular frontend also very similar to the angular webserver at this path. I will want to use the components that Splitting the server up into components is creating in this project. I probably should also componentize the angular frontend eventually but lets focus on the server for now.

Besides the components that we will reuse, I want a server pretty similar to the existing server with a testhelper, scheduled jobs process, gtest unit tests, a webserver, a database helper. I want a similar db schema directory, create_database.cpp, and rough layered architecture like the existing sever: database schema, table helpers / util code, business logic, and endpoints with Google tests to validate each component.

The basic idea of the website is that we will build out a gay community site. We will have a component that scans the Internet using AI at known locations and gathers events that are upcoming and places them into the database with images, description, location, date, etc. The site will then show upcoming events on the website and have a calendar. Eventually, we will expand functionality to let people post events with admin approval and let people search for events. We will also eventually do advertising.

For now, I want to focus on standing up a minimal server that we can start adding functionality to.

# Mason Update
- Please update the document based on the notes in this section and then remove this section.
- Please use the code base at: C:\Users\mason\source\repos\knottyyoga as a reference for a project using the server and client components as well as a lot of infrastructure.
- Please update the document based on: C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Converting the server to a multi tenant Saas architecture.md
- Please update the document based on: C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Splitting the server up into components.md
	- Please use C:\Users\mason\source\repos\server_components - as the code base used to componentize the server components that we should incorporate into this project.
	- Please update this document to use these server components to add to this project these features. Use the knottyyoga and server_components code bases as examples of how to add:
		- Account creation with email verification
		- Login with persistent login with session and device tokens
		- A user portal
		- User photo support
		- The ability to view account information and change first name, last name, email, photo
		- Show their name and a photo in the upper right corner when logged in like the knottyyoga app
- Please update the document based on: C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Componentizing the frontend.md
	- Please use C:\Users\mason\source\repos\honuware-web-components - as the code base used to componentize the front end angular components that we should incorporate into this project.
	- Please update this document to use these web_components code bases as examples of how to add:
		- Creation of a server access to call the server for AJAX Json calls
		- A local only test mode that mocks the server access with a local test implementation of server access
		- The various UI pathways to do the server components user scenarios listed above
		- Support for the CRUD style database editor for users with admin permissions
		- The ability to fetch the roles and permissions that this logged in user has
- Please update the document based on: C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Converting the server to a multi tenant Saas architecture.md
	- Please add anything into this document that needs to be done to support a multi tennant architecture. I would like to have multiple client front ends leveraging the same server to target different communities.
	- Let's be sure to extract the Name of the site and various meta aspects as fields in the database that get used to adapt in the front end.
- Let's list the work that needs to be done to stand up a very basic version of the site. Please help think of things that need to be done.
	- I'd like an MVP and then steps to flesh things out a bit.
	- Start with a basic web server that takes a few endpoint calls, runs gtest, incorporates conan, builds in Visual Studio with CMakeSettings.json, and incorporates the server components.
	- Add a basic client that 

# Current State & Inherited Context

> Conventions carried over from [[Splitting the server up into components]]: checkboxes on work items (checked off as implemented), numbered subsections inside each phase, lower layers first, tests for everything testable, you build/run tests and handle git yourself, and open questions live in this document rather than at the prompt.

## Where the honuware extraction stands (as of 7/11/2026)

The components this project will consume do not exist as a consumable artifact yet. Status of the componentization plan (in the Knotty Yoga vault):

- **Done (Phases 1–2.6):** all coupling broken in-place; seven CMake targets exist *inside* the knottyyoga repo (`honuware_foundation`, `honuware_data`, `honuware_services`, `honuware_square`, `honuware_platform`, `honuware_testing`, plus the brand-free `knotty_yoga_scheduler` engine). `knotty_yoga_core` is now the app library. Branch `test_target_split`.
- **Not done:** 2.7 (layering enforcement), 3.1 (physical move to `components/` + per-component include roots), 3.2 (bootstrap-seam audit), 4.x (the actual extraction to a public GitHub `honuware` repo with standalone build + CI), 5.x (example server).
- **Consequence:** CommunityFinder cannot consume honuware via FetchContent until splitting-plan Phase 4.1/4.2 lands — the components share knottyyoga's `src/` include root and top-level CMakeLists and are not standalone-buildable. This drives decision D1 below.

## The knottyyoga repo shape we are replicating

Monorepo `knottyyoga/` = `ui/` (Angular 19) + `server/knottyyoga_server/` (CMake 3.24 + Conan 2, C++20) + `database_server/` (Postgres 13.1 docker-compose on host port 5400). The server project builds five executables: `knottyyoga_the_server` (Crow API server, port 18080 dev), `knottyyoga_database_helper` (`--recreate_database` / `--migrate`), `knottyyoga_helper` (scheduled-jobs process), `knottyyoga_test_helper` (ftxui/replxx dev REPL), `knottyyoga_tests` (single gtest exe). The Crow server is **API-only** (`/api/*`); the SPA is served separately (dev: `ng serve` + proxy; prod: S3/CloudFront). Tests run against a real, already-running Postgres; the harness creates all tables once per run and every test rolls back its transaction.

Conan third-party set (16 packages): abseil, boost 1.86, crowcpp-crow 1.3.2, date, gtest 1.12.1, libcurl, libjpeg (transitive), libpng, libpqxx 7.10.5, libsodium, libtiff, mailio, openssl 3.5.2, zlib, plus ftxui + replxx (test_helper TUI only). The `conanfile.py`'s `init()` generates `ConanLibImports.cmake` defining the `${XXX_LIB}` variables.

## What a brand-new consumer app must implement

The extraction left exactly **five app-side composition roots** plus four small config headers. This is the core of Phase 2:

| Seam | knottyyoga file it mirrors |
|---|---|
| Server composition root (DB name, secrets bootstrap, mail, cookies, WebApp, `SetService<T>`, endpoint registration, `ServerConfig::Initialize`) | `src/main.cpp` |
| Schema composition: `MakeAppTables()` + `MakeDatabaseInfo()` = framework tables ++ app tables | `src/db_schema/make_app_tables.*`, `make_database_info.*` |
| Seed: `CreateAndPopulateDatabases()` — DDL order, allowed_tables, admin metadata, roles/permissions, first admin users, config-secret defaults, service accounts | `src/database_helper/create_database.cpp` |
| Endpoint anchor table: `web_app.cpp` with one `g_*` pointer per endpoint TU + `Endpoints::RegisterAllEndpoints()` | `src/endpoints/web_app.cpp` |
| Scheduler catalog + flags main | `src/scheduler/knottyyoga_job_catalog.*`, `scheduler/main.cpp` |
| Config headers: `App::kDatabaseName`; app secret keys; `App::FillInAppSecretDefaults(...)` (also supplies brand values for framework keys — sender name/address, subjects, website URL); ical branding (optional) | `src/business_logic/app_{database_config,secret_keys,secret_values,ical_config}.*` |
| Test main: `RegisterAllEndpoints()` + `GlobalDatabaseTestSupport::Initialize(MakeDatabaseInfo(kTestDatabaseName))` | `test/src/main.cpp` |
| App migrations stream (initially empty) | `src/business_logic/migration/app_migrations.*` |

What the platform gives for free once those exist: full auth (register/login/verify/sessions/remember/RBAC), quick accounts, the generic CRUD + admin-metadata endpoint suite (`get_db_schema`, `add_item`, `update_item`, `delete_item`, `get_fk_options`, …) = a free admin data editor, the photo pipeline (upload/scale/serve, linkable to any table via `table_item_photos`), config-secrets with at-rest encryption, mail, admin alerts, CSRF/CORS/security middleware, migration runner, and the whole test harness.

## The Angular frontend (for the later frontend phase)

Angular 19 standalone-components app. Cleanly reusable nearly as-is: the schema-driven admin CRUD (`pages/admin/**` + `controls/**` + `DatabaseSchemaService`/`TableManagementService` + the schema types) — driven entirely by the backend's `/api/get_db_schema` metadata; the auth stack (pages, guards, `AuthService`, CSRF + error interceptors); the `ServerAccess` token/proxy/mock HTTP layer (trim the ~250-method interface); shell/header/footer/toast/error services. Brand-specific bits are concentrated: logo SVG + header constants, `_variables.scss`/`_theme.scss` colors, package name, Square payment service, and the yoga feature areas (which we simply don't copy).

## Decisions inherited from the other plans (not re-litigated here)

- honuware: name, Apache-2.0, public GitHub repo, one repo/one version/layered targets, FetchContent SHA-pinned consumption (P1) with `FETCHCONTENT_SOURCE_DIR_HONUWARE` for local co-dev, Conan registry later, Linux-only CI, fresh history.
- Global sequencing: componentize → extract → **tenancy** → deploy. CommunityFinder starts on pre-tenancy honuware as a plain single-tenant server (exactly like knottyyoga today); when the tenancy work lands in honuware, single-tenant consumers adopt `FixedTenantResolver` (no control DB, no CloudFront site header) via a version bump — the tenancy plan explicitly designs for this.
- New sites live on GitHub (splitting plan Q2) — public vs private for *this* repo is Q2 below.

# Key Decisions & Options

## D1 — Sequencing against the honuware extraction

- **S1 — Wait for the full splitting plan (through Phase 5) before starting.** Cleanest, but blocks this project for weeks and the Phase 5 example server duplicates what CommunityFinder itself will prove.
- **S2 — Copy the component sources into the new repo now, reconcile later.** Fast start; recreates exactly the fork-divergence problem the whole componentization exists to prevent. Rejected.
- **S3 (recommended) — CommunityFinder is the real second consumer and drives the extraction.** Finish splitting-plan Phases 3.1/3.2/4.1/4.2 first (that plan tracks the work; this plan's Phase 0 is the gate), then bootstrap CommunityFinder against the honuware GitHub repo, using the `FETCHCONTENT_SOURCE_DIR_HONUWARE` local-checkout override while the two evolve together. Splitting-plan Phase 2.7 (enforcement hardening) does **not** block us. The Phase 5 `examples/hello_server` gets demoted: CommunityFinder is a stronger "no yoga leaked in" regression test than any example (Q3).
- Work that does not depend on honuware (repo bootstrap, schema design, admin/branding decisions, frontend planning) proceeds in parallel before the gate.

## D2 — Repo layout, hosting, naming

- **Layout (recommended): monorepo mirroring knottyyoga**, slightly flattened: `communityfinder/{ui/, server/, database_server/}` (no extra `server/<name>_server/` nesting — that was an artifact of the sibling `docker_project/`). One clone gives a contributor everything; proven layout; the CLAUDE.md conventions port directly.
- **Hosting: GitHub** (decided by the splitting plan for new sites). **Public vs private is Q2** — the splitting plan assumed friends' sites are public for portfolio value, but this is your own product idea; my recommendation is private-first with the option to open later (opening later is trivial; un-publishing is not).
- **Naming:** working name **CommunityFinder** (repo `communityfinder`, targets `communityfinder_*`, DB `communityfinder`, test DB `test_communityfinder`, service account `scheduler@communityfinder.local`). The real product/brand name is Q1 — it only touches the app-side seam files and secrets defaults, so renaming later is cheap, but choosing before Phase 2 avoids churn.

## D3 — Which honuware components to consume

Link `honuware_foundation`, `honuware_data`, `honuware_services`, `honuware_platform`, `honuware_testing`, and the scheduler engine. **Skip `honuware_square`** until payments/advertising become real (it's a one-line link to add back). Conan set: knottyyoga's minus nothing critical — mail (mailio/openssl) is needed for verification emails, images (png/tiff/zlib) for event photos, libsodium for secrets/auth, abseil for flags. ftxui/replxx only if/when we build the test_helper TUI (Q10).

## D4 — MVP events domain schema

Options ranged from ultra-minimal (one `events` table) to rich (organizers, recurrence rules, tags, per-source scrape configs). Recommended middle, respecting FK layering (referenced tables first):

1. **`venues`** — id (BIGSERIAL PK), name, address/city/state/zip, url, description, lat/lng (nullable), status. Events can also carry freeform location text for one-offs.
2. **`event_sources`** — the "known locations to scan": id, name, url, venue_id (nullable FK), kind (venue site / ticketing page / social / aggregator), enabled, scan_hints (text the scanner prompt can use), last_scanned_at, notes.
3. **`events`** — id, title, description, starts_at (timestamptz), ends_at (nullable), venue_id (nullable FK) + location_text, url, cost_text, source_id (nullable FK), external_key (source-scoped dedup key), origin (scanned / manual / user_submitted), status (pending / approved / rejected / archived), created_by (nullable FK people), created_at/updated_at. `UNIQUE (source_id, external_key)` is the ingestion idempotency anchor. Images via the framework photo tables + `table_item_photos` — no image columns needed.
4. **`event_categories`** + **`event_category_assignments`** — optional at MVP (Q4). Cheap to add now, cheap to add later.
5. **`scan_runs`** — scanner audit trail (source_id, started/finished, status, events_found/ingested, error). Only meaningful once the scanner exists; I recommend deferring to Phase 6 (Q4).

**Recurrence:** store one row per occurrence; no recurrence rules in MVP (the scanner naturally produces occurrences; a weekly bar night is N rows). Revisit only if manual entry of repeating events becomes painful.

## D5 — AI event scanner architecture (decide later, design the seam now)

The scanner is deliberately **not** in the minimal-server scope, but the ingestion seam is (Phase 3): a permission-gated, idempotent `POST /api/admin/ingest_events` batch endpoint + the `(source_id, external_key)` upsert in `EventHelper`. Whatever form the scanner takes, it talks to that one endpoint. Options for later (Q5):

- **A — In-process C++ scheduled job.** Scheduler POSTs `/api/admin/run_event_scan`; the endpoint iterates enabled `event_sources`, fetches pages via `Http::HttpClient`, calls the Claude API over raw HTTPS (there is no official C++ SDK) — Messages API with `web_fetch`/`web_search` server tools + structured outputs (`output_config.format` with a JSON schema for the extracted event list; model `claude-opus-4-8`) — then upserts. Pros: one deployable, reuses the whole stack. Cons: hand-rolled HTTP/JSON against the API, slow prompt iteration, long-running scans inside request handlers.
- **B — Separate worker process** (Python or TypeScript, official Anthropic SDK or Claude Agent SDK) that logs in with the scheduler service account and POSTs to the ingestion endpoint. Pros: real SDK, fast prompt iteration, agentic scraping much easier. Cons: a second runtime to run/deploy.
- **C — Managed Agents scheduled deployment** (Anthropic-hosted cron agent hitting the ingestion endpoint). Pros: zero scanner infra, built-in scheduling. Cons: beta surface, and the server must be internet-reachable — post-deploy only.

My lean: **B** for development velocity (with C as the eventual production hosting for it), keeping A viable for simple, well-structured sources. No commitment needed until Phase 6.

## D6 — Frontend scope and timing

Per your Overview, server first. Recommended: frontend is **Phase 5**, sized minimal — shell + auth + the reused admin CRUD + one public "upcoming events" page with a simple calendar. Rationale for not deferring it entirely: the admin CRUD UI is the *manual event entry* tool, which makes the site useful (and testable end-to-end) before the scanner exists. It can be pulled earlier if you want manual entry sooner (Q6).

## D7 — Dev environment

- **Postgres (recommended): share the existing `database_server` container** (port 5400) — `CreateAndPopulateDatabases()` only touches its own named databases, so `communityfinder`/`test_communityfinder` coexist safely with the knottyyoga DBs, and there is zero new setup. Alternative: a copied compose project on its own port for full isolation (Q7). Note Postgres 13.1 is aging; upgrading is a deploy-time question, not a dev blocker.
- **Ports:** server dev port **18081** (knottyyoga owns 18080) so both stacks run side-by-side; `ui/src/proxy.conf.json` targets 18081; `ng serve --port 4201` when both UIs run at once.

# Hand-off Requirements to the Componentization Plan

Bootstrapping a genuinely fresh consumer exposes gaps the in-repo extraction couldn't see. These belong in the splitting plan (Phases 3.2/4.1 — most would otherwise be forced by its Phase 5 example server, which CommunityFinder replaces). Listed here so they can be folded into that document; each is `⇦ communityfinder`:

1. **Framework endpoint registration must move into platform.** Today the app-authored `web_app.cpp` anchors *every* endpoint TU, framework ones included — a new consumer would have to enumerate ~50 framework `g_*` pointers by hand and silently lose endpoints when honuware adds one. Platform should own `Endpoints::RegisterFrameworkEndpoints()` (anchoring its own endpoint TUs); the app's `RegisterAllEndpoints()` calls it plus its own anchors.
2. **Split `create_database.cpp` into framework/app halves**, mirroring `MakeFrameworkTables`/`MakeAppTables`: platform exposes `CreateFrameworkTables(transaction, dbInfo)` (DDL + indexes in FK order) and `PopulateFrameworkTables(...)` (admin metadata for the ~33 framework tables, base roles/permissions — Administrator/User + `admin_portal` etc., allowed_tables entries for framework tables, framework config-secret defaults). The consumer's `CreateAndPopulateDatabases()` composes instead of copying ~3,000 lines of seed code it doesn't own.
3. **`kTestDatabaseName` moves out of `honuware_testing`.** `global_database_test_support.h` hardcodes `"test_knottyyoga"`; the harness already takes the composed `DatabaseInfo` (which carries the name), so the constant belongs in each app's test main.
4. **Residual brand literals in framework code:** `business_logic/auth/service_account.h` (`scheduler@knottyyoga.local`, `@knottyyoga.local`, role wiring) → parameterized/app-supplied; `endpoints/health.cpp` reads `KNOTTYYOGA_VERSION` → `HONUWARE_VERSION` (legacy fallback); the `KNOTTYYOGA_DB_NAME` / `KNOTTYYOGA_DB_*` connection env vars → `HONUWARE_DB_*` via the existing `GetEnvWithFallback` pattern.
5. **The `secrets_helper_test_util.cpp → business_logic/app_secret_values.h` reverse dependency** (documented transitional wrinkle from splitting-plan 1.3/2.6) must dissolve at extraction: the testing lib exposes a register-defaults hook; the app's test main injects `App::FillInAppSecretDefaults`.
6. **The shared third-party Conan requirements list** (already part of the P1 decision) needs its concrete mechanism: honuware ships one requirements list its own `conanfile.py` and every consumer's `conanfile.py` import, so versions can't drift.
7. **Consumer-facing docs in the honuware repo:** the new-consumer checklist (the table above), layer rules, testing conventions, and the Windows/Crow gotchas (`crow::HTTPMethod` PascalCase, MSVC dead-stripping of unanchored endpoint TUs, `ThreadPool::Shutdown()` after `handle_full()`).

# Target Architecture

## Repo layout

```
communityfinder/
├── CLAUDE.md                  adapted conventions (layering, testing, env vars, gotchas)
├── ui/                        Angular 19 SPA (Phase 5)
├── server/
│   ├── conanfile.py           subset deps + honuware shared requirements list
│   ├── conan_provider.cmake   Windows/VS Conan integration (as in knottyyoga)
│   ├── CMakeLists.txt         project, C++20, per-OS flags, FetchContent(honuware @ SHA)
│   ├── CMakeSettings.json     VS x64-Debug (Ninja) config
│   ├── src/
│   │   ├── main.cpp                       server composition root
│   │   ├── business_logic/
│   │   │   ├── app_database_config.h      App::kDatabaseName
│   │   │   ├── app_secret_keys.h          app secret key names (initially ~empty)
│   │   │   ├── app_secret_values.{h,cpp}  App::FillInAppSecretDefaults (brand values)
│   │   │   ├── events/                    EventHelper (ingestion/approval/publish)
│   │   │   └── migration/app_migrations.{h,cpp}
│   │   ├── db_schema/                     venues, event_sources, events, make_app_tables, make_database_info
│   │   ├── sql_util/table_helpers/        Venues, EventSources, Events
│   │   ├── endpoints/                     web_app.cpp anchor table + events endpoints
│   │   ├── database_helper/               main.cpp + create_database.cpp
│   │   └── scheduler/                     communityfinder_job_catalog + main.cpp
│   ├── test/                              src/main.cpp + kTestDatabaseName; app tests live next to code
│   ├── package/                           Dockerfile + build_linux_release.sh (Phase 7)
│   └── certs/cacert.pem
└── database_server/           pointer to shared container OR own compose (per Q7)
```

## CMake targets and links

```
FetchContent: honuware @ pinned SHA  →  honuware_{foundation,data,services,platform,testing} + scheduler engine
communityfinder_core        PUBLIC-links foundation/data/services/platform  (app library)
communityfinder_server      = src/main.cpp                     → core
communityfinder_database_helper                                → core (+ABSL)
communityfinder_helper      = scheduler main + catalog         → core + scheduler engine (+ABSL)
communityfinder_tests       = test/src/main.cpp                → core + honuware_testing (+GTest)
```

Same per-OS flag blocks as knottyyoga (MSVC `/std:c++20 /EHsc`, `_WIN32_WINNT`; Linux gcc, `-lgssapi_krb5` last). `FETCHCONTENT_SOURCE_DIR_HONUWARE` documented for cross-repo co-development.

# Phased Implementation Plan

## Phase 0 — Gate: honuware exists as a consumable repo

The work lives in [[Splitting the server up into components]] (its Phases 3.1, 3.2, 4.1, 4.2 plus the hand-off items above); this phase just states what must be true before our Phase 2:

- [ ] 0.1 honuware repo on GitHub builds **standalone** (own conanfile/CMake/test exe; component tests green against a Postgres container).
- [ ] 0.2 knottyyoga consumes it via FetchContent at a pinned SHA and its full suite is green (proves the seam from the first consumer's side).
- [ ] 0.3 Hand-off items 1–6 above landed (item 7 docs can trail slightly).

Phases 1 (and the design halves of 3/5) do not wait on this gate.

## Phase 1 — Repo & dev-environment bootstrap (no honuware dependency)

### 1.1 Repository
- [ ] Create the GitHub repo (name per Q1, visibility per Q2) with `.gitignore` (VS/CMake/Conan/node artifacts, `out/`, `build/`, `dist/`), README stub, and the `ui/ server/ database_server/` skeleton.
- [ ] Write the root `CLAUDE.md`, adapted from knottyyoga's: runtime layering (endpoints → business_logic → services → table_helpers → db_schema), component layer order, testing conventions (no fixtures, pre-created tables, HTTP via `handle_full`, `ThreadPool::Shutdown()` race), naming conventions, env-var table (`PORT`, `HONUWARE_*` set), Crow `HTTPMethod` PascalCase + MSVC dead-strip gotchas, and the planning-directory override pointing at this vault.

### 1.2 Development database
- [ ] Per Q7: either document "uses the knottyyoga `database_server` container on port 5400" or copy the compose project onto its own port. Verify `communityfinder` + `test_communityfinder` databases can be created alongside the existing ones.

## Phase 2 — Server skeleton (boots, auth works, admin CRUD works)

Everything here is the minimal consumer checklist made real. Lower layers first.

### 2.1 Build system
- [ ] `server/conanfile.py` (D3 subset; imports honuware's shared requirements list) with the `init()`-generates-`ConanLibImports.cmake` pattern; `conan_provider.cmake` + `CMakeSettings.json` for VS; top-level `CMakeLists.txt` with FetchContent(honuware @ SHA), the target set from *Target Architecture*, and the knottyyoga per-OS flag blocks. Verify a clean configure+build on Windows (you) — this is the first true test of honuware's consumability; file anything broken back to the splitting plan.

### 2.2 App config seams (foundation of everything above)
- [ ] `business_logic/app_database_config.h` (`App::kDatabaseName`), `app_secret_keys.h` (empty scaffold + NOTE), `app_secret_values.{h,cpp}` (`FillInAppSecretDefaults`: sender name/address, activation-mail subject, admin-alerts subject, website address/login URLs — debug vs prod per `_DEBUG`, brand values per Q1). Tests mirroring knottyyoga's `app_secret_values_test.cpp` (defaults present, no overlap with framework defaults).

### 2.3 Schema & migration composition
- [ ] `db_schema/make_app_tables.{h,cpp}` (empty body for now) + `make_database_info.{h,cpp}` (framework ++ app). `business_logic/migration/app_migrations.{h,cpp}` (empty stream) + the `all_migrations` composition. Composition test: composed `DatabaseInfo` = framework set exactly, framework precedes app.

### 2.4 Endpoints anchor + server composition root
- [ ] `endpoints/web_app.cpp`: `RegisterFrameworkEndpoints()` (hand-off item 1) + `Endpoints::RegisterAllEndpoints()`; no app endpoints yet; no `AppEndpointAuthHelper` (no app services — Square stays out per D3).
- [ ] `src/main.cpp`: port from env (default 18081), production DB helper for `App::kDatabaseName`, secrets at-rest bootstrap sequence, `MailHelper`, `CookieManagerFactory`, column redactions, `WebApp`, registration, `ServerConfig::Initialize/ValidateProdEnvironment`. (Copy of knottyyoga's minus the Square block.)

### 2.5 database_helper + seed
- [ ] `database_helper/main.cpp` (`--recreate_database` gated by `HONUWARE_ALLOW_DESTRUCTIVE`, `--migrate`).
- [ ] `create_database.cpp`: `CreateAndPopulateDatabases()` composing platform's framework halves (hand-off item 2) + app half (empty at this phase); seed first admin user(s) (Q1: which emails?), scheduler service account, framework + app secret defaults.

### 2.6 Test infrastructure + smoke tests
- [ ] `test/src/main.cpp` (anchor endpoints, `Initialize(MakeDatabaseInfo(kTestDatabaseName))`), app-side `kTestDatabaseName = "test_communityfinder"` (hand-off item 3), secrets-defaults injection hook (hand-off item 5).
- [ ] Smoke tests (all over HTTP per convention): `/api/health` 200; register→verify→login→`/api/me` round-trip; generic CRUD on a framework table via `add_item`/`get_row`/`delete_item`; photo upload/get; CSRF guard behavior.

**Phase gate:** server boots; login as seeded admin works; `get_db_schema` returns the framework schema; test suite green. From here every phase adds app functionality on a working platform.

## Phase 3 — Events domain (db_schema → table_helpers → business logic → endpoints)

### 3.1 db_schema
- [ ] `venues.{h,cpp}`, `event_sources.{h,cpp}`, `events.{h,cpp}` (+ categories pair if Q4 says now) per the D4 schema, with `Create*Indexes` (events: `(status, starts_at)`, `(source_id, external_key)` unique). Wire into `make_app_tables.cpp` (FK order: venues → event_sources → events) and `create_database` DDL order.
- [ ] Seed/admin wiring in `create_database.cpp`: allowed_tables entries, admin metadata (top-level tables, column data info, friendly names, display templates — an event displays as `{title} @ {starts_at}`), `manage_events` permission + role wiring, a few dev venues/sources/sample events.
- [ ] Schema tests per house pattern.

### 3.2 table_helpers
- [ ] `Venues`, `EventSources`, `Events` helpers (ctor takes `DatabaseHelper`, methods take `Transaction&`, `KeyValueTable` at boundaries). Events queries: `GetEventsInRange(from, to, status)`, `GetBySourceAndExternalKey`, `GetPendingEvents`. Tests for each.

### 3.3 business_logic/events
- [ ] `EventHelper`: `IngestEvents(batch)` — per-item upsert keyed on `(source_id, external_key)` (insert as `pending`+`scanned`; update policy for re-scanned events that were already approved — recommend: update fields, keep status, flag if material change — refine at Q4); `Approve/Reject(eventId)`; visibility rule (public = approved ∧ not archived). Tests: idempotent re-ingestion, status transitions, visibility.

### 3.4 endpoints
- [ ] Public: `GET /api/events/upcoming?from&to` (approved only, range-clamped, venue joined) and `GET /api/event?id`. Admin: `POST /api/admin/ingest_events` (permission-gated batch, returns per-item created/updated/unchanged). Approval itself rides the generic CRUD/admin UI (status column edit) — dedicated approve endpoints only if a workflow needs them later. Each endpoint: `.h/.cpp` + anchor in `web_app.cpp` + HTTP-driven `_test.cpp`.
- [ ] Verify event photos work end-to-end through the framework photo endpoints + `table_item_photos` (one test).

**Phase gate:** an event entered via generic CRUD (or curl) appears in `/api/events/upcoming`; ingestion is idempotent; suite green.

## Phase 4 — Scheduled-jobs process

### 4.1 Catalog + main
- [ ] `scheduler/communityfinder_job_catalog.{h,cpp}`: `JobTarget`/`JobIntervals`, `BuildCommunityFinderJobs(target, intervals)` with the initial set — `archive_past_events` (moves long-past events to `archived`), `notify_admin_alerts` (verify the digest endpoint is framework; it should be, per splitting-plan Q12), and a **disabled-by-default** `run_event_scan` placeholder for Phase 6. `Validate…` + password resolution mirroring knottyyoga's. Catalog tests.
- [ ] `scheduler/main.cpp` with ABSL flags (server_url default `http://localhost:18081`, service-account creds, per-job intervals), linking the honuware scheduler engine (foundation-only).
- [ ] The `archive_past_events` admin endpoint + business logic + tests (it's the first scheduled job with a body).

**Phase gate:** `communityfinder_helper` authenticates against the local server and runs the archive job on its interval.

## Phase 5 — Minimal Angular frontend

### 5.1 Skeleton
- [ ] `ui/` scaffold copied from knottyyoga: `main.ts`/`app.config.ts` (APP_INITIALIZER silent login, interceptor order), `angular.json` (dev/prod/local configs), tsconfig path aliases, `proxy.conf.json` → 18081, environments (no Square IDs). Trim `ServerAccess` interface + network/proxy/mock trio to auth + admin + events.

### 5.2 Auth + shell
- [ ] Auth pages (login/register/verify), guards, `AuthService`, CSRF + error interceptors; header/footer/page-not-found with CommunityFinder branding (Q1): new logo asset, `_variables.scss` palette, package name.

### 5.3 Admin CRUD (the reused prize)
- [ ] `pages/admin/**`, `controls/**`, `DatabaseSchemaService`/`TableManagementService`, schema types — pointed at the same endpoint contract. Manual event/venue/source entry works here.

### 5.4 Public events UI
- [ ] Home + upcoming-events list (grouped by day, venue + photo) + a simple month calendar view + event detail. Adapt knottyyoga's `calendar/` feature area if it fits; otherwise a minimal new one.

**Phase gate:** enter an event in the admin UI, approve it, see it on the public page and calendar — full manual loop, no scanner needed.

## Phase 6 — AI event scanner (future; seam already built)

Not scoped in detail yet — decide architecture at Q5 when we get here. Sketch: per-source scan → fetch/interpret page(s) with Claude (`claude-opus-4-8`, web_fetch/web_search server tools where applicable, structured outputs producing `[{title, starts_at, ends_at?, description, url, external_key, …}]`) → `POST /api/admin/ingest_events` → events land `pending` → admin approves in the CRUD UI. Add `scan_runs` audit table + `run_event_scan` job enablement + per-source prompt hints (`event_sources.scan_hints`) at that point. Follow-ups: dedup across sources, image capture, geo.

## Phase 7 — CI & deployment (future)

- [ ] GitHub Actions: Linux build + full test run with a Postgres service container (the honuware repo's CI from splitting-plan 4.3 is the template).
- [ ] `package/` Dockerfile + `build_linux_release.sh` adapted (FetchContent needs git+network in the builder stage, per the splitting plan's deployment invariant).
- [ ] AWS deploy (CloudFront + S3 + EC2 + RDS mirroring knottyyoga's setup) — timing relative to the tenancy work is Q9.

# Open Questions

Please answer inline (house style). Recommendations included so you can just say "agreed".

1. **Product/brand name.** Is **CommunityFinder** the actual name, or a working title? It drives: repo name, `App::kDatabaseName`, mail sender name ("… <what?>"), website address secrets, service-account domain, UI branding, and eventually the domain name. Also: which admin email(s) should `create_database` seed? (Recommendation: proceed with CommunityFinder as the working name everywhere; a rename before Phase 5 is ~an hour of mechanical work.)

2. **GitHub visibility.** Public (matches the splitting-plan assumption for new sites, portfolio value for friends) or **private** (my recommendation — this is your product idea and its business logic; you can open it later, and friends can be invited as collaborators either way)?

3. **Sequencing (D1/S3).** Confirm: drive the honuware extraction to its Phase 4.2 as the gate, CommunityFinder becomes the real second consumer, and the splitting plan's Phase 5 `hello_server` example is demoted to optional/na. Also confirm I should fold the **hand-off requirements** section above into the splitting-plan document (as its plan did with the tenancy hand-offs).

4. **MVP schema scope (D4).** (a) `event_categories` now or later? (My lean: now — trivial, and the UI filter will want it early.) (b) `scan_runs` deferred to Phase 6? (My lean: yes.) (c) Re-ingestion policy when a scanned event was already admin-approved and the source page changed: silently update fields, or flag for re-review? (My lean: update minor fields, revert to `pending` only when date/venue changes.)

5. **Scanner architecture (D5)** — no decision needed until Phase 6, but if you already have a preference between in-process C++ (A), a separate Python/TS worker (B, my lean), or an Anthropic Managed-Agents scheduled deployment (C), it shapes small things earlier (e.g., how much scan config lives in `event_sources`).

6. **Frontend timing (D6).** Phase 5 as placed, or pull it earlier (right after Phase 3) so manual event entry gets a UI sooner? Server-only phases 2–4 are still curl/test-driven either way.

7. **Dev environment (D7).** Share the knottyyoga Postgres container (port 5400, my recommendation) or clone `database_server/` onto its own port? And confirm dev port 18081 for the server.

8. **Public registration at MVP.** The platform gives register/verify for free. Leave self-registration open from day one, or admin-only accounts until user-submitted events (with quick accounts) arrive? (My lean: endpoints stay registered — they're framework — but the UI hides registration until we want it; effectively invitation-only.)

9. **Deploy timing.** knottyyoga's sequence puts tenancy before its first deploy. CommunityFinder could deploy earlier as a plain single-tenant server (pre-tenancy honuware) and adopt `FixedTenantResolver` in a later version bump — or wait for tenancy to land in honuware. Any preference? (No work depends on this until Phase 7.)

10. **test_helper TUI.** Include a `communityfinder_test_helper` (ftxui/replxx REPL like knottyyoga's) from the start, or defer until the domain has enough commands to justify it? (My lean: defer; the generic admin CRUD covers most manual data needs early on.)

11. **Collaborators.** Will the friends work on CommunityFinder itself (affects Q2, repo permissions, and whether a LICENSE/contribution note is needed), or only on honuware and their own sites?