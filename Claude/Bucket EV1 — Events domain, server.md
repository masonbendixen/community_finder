---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Levi Kuhn (owner) · Mason Bendixen
Last Updated: 8/12/2026
Version: 0.1
tags:
---

# Bucket EV1 — Events domain, server

**Track:** EV — Events engine · **Size:** M · **Stream:** A · **Owner:** Levi
**Needs:** GD1 (the vocabulary values — locked lists in [[Bucket GD1 — Taxonomy & editorial foundations]] §1–§3, §5). **Feeds:** EV2 (public UI), EV3 (submissions), EV4 (jobs), EV5 (scanner), GD4a (source seeding) — and **Stream B blocks on Slice 1 of this bucket** (the shared vocabulary tables + `venues`), so land Slice 1 as your first PR and ping Mason when it merges.

# Context

The events engine's server half: schema → helpers → domain logic → endpoints. Implements [[Initial Project Implementation Outline]] Stream A bucket EV1 and **supersedes [[Setting up the project]] Phase 10** (per the outline's supersession rule), folding in what the brainstorm added: `series` grouping for recurring events, scene tags shared with the guide, `manage_events`, ingestion idempotency, and the approve/reject flow.

**Deliberate supersessions of the SUTP D4 sketch** (drafted before house conventions were pinned):

1. **Times are `int64_t` microseconds in `BIGINT` columns, suffixed `_us`** — not `timestamptz`. That's the honuware/knottyyoga house convention (DB clock via `SELECT now_us()`; column defaults via `kDatabaseInfoDefaultNow`). Event times are **real UTC instants**, rendered in **America/Los_Angeles** app-wide (single-city site; a per-community timezone becomes site meta when PL4 arrives).
2. **`event_categories` is renamed `categories`** (+ new `scene_tags`, `neighborhoods`) — the vocabularies are shared with the guide per Interface Contract 1; values come from GD1 verbatim. Assignments stay events-scoped (`event_category_assignments`, `event_scene_tag_assignments`).
3. **Recurrence stays one-row-per-occurrence (D4 item 5) plus a `series` grouping entity.** We are deliberately NOT adopting knottyyoga's derive-occurrences-on-the-fly machinery (`class_schedules`/slots) — community events are ingested/entered as concrete dated instances; `series` is a durable label + page ("T4T, 2nd Saturdays"), not a generator. If hand-entering weekly recurring events ever hurts, EV3/EV5 automate intake before we build a materializer.
4. **Approve/reject gets the knottyyoga review quartet** (`status` + `reviewed_by_person_id` + `reviewed_us` + `review_notes`) instead of a bare status column.

**Read before starting** (the models this doc cites throughout):

- CF repo conventions: `communityfinder/CLAUDE.md` (layering, anchors, testing).
- knottyyoga (`C:\Users\mason\source\repos\knottyyoga`) as the pattern reference: `CLAUDE.md` §"Adding a New Database Table" (the 11-step checklist); `src/db_schema/class_schedules.{h,cpp}` (table declaration); `src/sql_util/table_helpers/class_schedule_slots.{h,cpp}` (helper shape); `src/business_logic/scheduling/class_schedule_helper.h` (logic shape: request/result structs, error-code strings); `src/endpoints/get_calendar.cpp` + `get_class_detail.cpp` (endpoint + routing + query params); `src/endpoints/get_calendar_test.cpp` and `admin_close_classes_test.cpp` (test shape incl. the `ThreadPool::Shutdown()` rule); `src/database_helper/create_database.cpp` (admin-metadata registration idioms).
- honuware: `components/data/sql_util/schema/database_info.h` (the exact `AddTable`/`AddColumn*`/`AddUniqueConstraint`/`AddCheckConstraint` API).

# Scope

**In:** shared vocabulary tables (`categories`, `scene_tags`, `neighborhoods`) + seed · `venues` · `event_sources` · `series` · `events` + category/scene junctions · table helpers · `EventHelper` (ingest/review/visibility) · public read endpoints · admin ingest + review endpoints · `manage_events` permission + admin CRUD metadata · photo enablement for venues/events · test_helper commands · tests at every layer.

**Out (explicit non-goals):** the public UI, ICS feeds, JSON-LD (**EV2**) · public submission form (**EV3** — but `origin = 'user_submitted'` is reserved now) · archive job + scheduler (**EV4** — the `archived` status exists now, nothing sets it yet) · `scan_runs` + the scanner itself (**EV5**, per SUTP Q4b) · `venue_profiles` and all guide tables (**GD2**, Stream B — never edit their files) · seeding the ~40 real event sources (**GD4a** content-ops, via admin CRUD once this lands).

# Interface contracts this bucket must honor (from the outline)

1. **Slice 1 is shared foundation** — `categories`/`scene_tags`/`neighborhoods` columns and seed values come from GD1 §1–§3 **verbatim** (slugs are load-bearing; a slug rename needs a cross-stream ping). Seed-assertion tests make GD1 binding.
2. **`venues` is Stream A's table, exactly as specced here** — Stream B satellites onto `venues.id` via `venue_profiles` (GD2) and never edits Stream A files. Keep `venues` guide-free: no crowd/vibe, no adult flag, no slug (guide URL identity lives on the profile).
3. **Freshness convention (GD1 §5)** on the listable tables this bucket owns: `venues` and `series` carry `status` (GD1 enum) + `last_verified_at_us`. `events` does NOT — event rows are pipeline items with the workflow status (`pending/approved/rejected/archived`).
4. **Shared-file etiquette:** `web_app.cpp` anchors, CMakeLists source lists, `create_database.cpp` app-half — one-line additions, alphabetized, touch only your stream's lines.
5. **Namespaces:** Stream A owns `/api/events*`, `/api/event_series*`, `/api/admin/ingest_events`, `/api/admin/review_event`, and the `manage_events` permission.

# Design pin-downs

## Schema (all tables also get `created_us` default `now_us()`; mutable tables get `updated_us`)

**Slice 1 — shared vocabulary + venues:**

- **`categories`** — `slug` (unique), `display_name`, `description`, `sort_order` (BIGINT default), `enabled` (bool default true). Seed: GD1 §1's twelve.
- **`scene_tags`** — same shape + `coverage` (string, CHECK `IN ('full','link_out')`). Seed: GD1 §2's eleven with postures.
- **`neighborhoods`** — `slug` (unique), `display_name`, `region` (string, CHECK in GD1 §3's five regions), `sort_order`, `enabled`. Seed: GD1 §3's twenty-four.
- **`venues`** — `name`, `description`, `address`, `city`, `state`, `zip`, `url`, `lat`/`lng` (nullable `DB_TYPE_DOUBLE` — write-only at MVP, no map view yet), `neighborhood_id` (nullable FK → neighborhoods), `status` (string, CHECK in GD1 §5 enum, default `'open'`), `last_verified_at_us` (nullable BIGINT), `created_us`, `updated_us`.

**Slice 2 — events core:**

- **`event_sources`** — `name`, `url`, `kind` (CHECK `IN ('ics_feed','venue_site','ticketing_org','aggregator','editorial_monitor','submission','social')` — merges D4's list with brainstorm C.2's intake types), `venue_id` (nullable FK), `enabled` (default true), `scan_hints` (text, for EV5), `volatility_tier` (CHECK in GD1 §5 tiers — the per-source scan cadence EV5 consumes), `last_scanned_at_us` (nullable), `notes`.
- **`series`** — `title`, `slug` (unique), `description`, `cadence_text` ("2nd Saturdays"), `url`, `venue_id` (nullable FK), `status` (GD1 enum, default `'open'`), `last_verified_at_us` (nullable), timestamps.
- **`events`** — `title`, `description`, `starts_at_us` (NOT NULL), `ends_at_us` (nullable), `venue_id` (nullable FK) + `location_text` (for venue-less events), `url`, `cost_text`, `source_id` (nullable FK → event_sources), `series_id` (nullable FK → series), `external_key`, `origin` (CHECK `IN ('scanned','manual','user_submitted')`), `status` (CHECK `IN ('pending','approved','rejected','archived')`, default `'pending'`), `created_by_person_id` (nullable FK → people), `reviewed_by_person_id` (nullable FK → people), `reviewed_us` (nullable), `review_notes`, timestamps.
  - **`AddNamedUniqueConstraint` on `(source_id, external_key)`** — the ingestion idempotency anchor (Postgres allows repeated NULL `source_id`, so manual events don't collide).
  - `CreateEventsIndexes`: `(status, starts_at_us)` (the upcoming query), `(series_id)`, `(venue_id)` — raw `CREATE INDEX IF NOT EXISTS` DDL with a comment naming the query each serves, knottyyoga-style.
- **`event_category_assignments`** — `event_id` FK, `category_id` FK, `AddUniqueConstraint(event_id, category_id)`.
- **`event_scene_tag_assignments`** — `event_id` FK, `scene_tag_id` FK, unique pair.

No image columns anywhere — photos ride the framework pipeline (`table_item_photos`), enabled in §1.6/§2.6 below.

## Behavior pins

- **Deletes cascade.** honuware FK constraints are emitted `ON DELETE CASCADE`, always — deleting a venue deletes its events; deleting a series deletes its instances; deleting a category deletes its assignments. Operational rule everywhere in this domain: **close/archive, don't delete** (that's also what the closure graveyard and freshness convention want); the admin editor's delete is for data-entry mistakes only.
- **Visibility rule:** public = `status == 'approved'`. (`archived` is its own status, so "approved ∧ ¬archived" collapses to the equality; EV4 owns the transition to `archived`.)
- **Ingest upsert (SUTP Q4c, adopted-by-default):** match on `(source_id, external_key)`. New → insert as `pending`/`scanned`. Existing → **minor** field changes (title, description, url, cost_text, location_text, series/category hints) update silently keeping current status; **major** changes (`starts_at_us`, `ends_at_us`, `venue_id`) update AND revert an `approved` event to `pending` with a review note ("date/venue changed by re-scan"). Unchanged → no-op, reported as such. Never resurrect a `rejected` event (re-ingesting a rejected key reports `rejected_kept` and changes nothing) — that's what makes reject meaningful against a weekly scanner.
- **Review:** `pending → approved | rejected` (also allowed from the other of the two, for corrections); sets the reviewer quartet. `archived` is terminal except back to `approved` (un-archive; rare, admin CRUD can do it raw).
- **Range clamp on public queries:** default window now → +42 days (knottyyoga's calendar default); hard cap 92 days per request; `from_us` may reach back 1 day (time-zone slack). EV2's calendar pages through windows.

# Layered work items

*(Numbered; lower layers first; every item names its tests. Slice 1 = §1; land it, ping Stream B, then proceed.)*

### 1. Slice 1 — db_schema: shared vocabulary + venues

- [ ] 1.1 `src/db_schema/categories.{h,cpp}` — constants + `MakeCategoriesTable(DatabaseInfo)` per the schema pin-down. Model: knottyyoga `src/db_schema/class_schedules.{h,cpp}`.
- [ ] 1.2 `src/db_schema/scene_tags.{h,cpp}` — incl. the `coverage` CHECK.
- [ ] 1.3 `src/db_schema/neighborhoods.{h,cpp}` — incl. the `region` CHECK.
- [ ] 1.4 `src/db_schema/venues.{h,cpp}` — incl. `MakeVenuesTable` + (empty-ok) `CreateVenuesIndexes`.
- [ ] 1.5 `make_app_tables.cpp`: register in FK order — categories, scene_tags, neighborhoods, venues. **Test:** update `make_database_info_test.cpp` — **its `AppStreamIsEmpty` and `ComposedSchemaIsExactlyTheFrameworkTableSet` tests exist precisely to be replaced by the first app table** (they fail the moment 1.5 lands); rewrite them to assert the app table set, plus the four tables' expected columns.
- [ ] 1.6 `create_database.cpp` app half (follow knottyyoga's `create_database.cpp` idioms + CF's existing `PopulateAppRolesAndPermissions` scaffold). **The three-place rule** — a new table touches all three: `CreateTables` (live-DB DDL execution in FK order; the test harness builds from `MakeDatabaseInfo` instead, so forgetting this only breaks the live recreate), `PopulateTables`, and `make_app_tables.cpp` (done in 1.5). Then: `PopulateVocabularies(...)` seeding GD1 §1–§3 **verbatim** (slug, display_name, coverage/region, sort_order; seed lambdas capture `databaseHelper`/`tableName` **by value** — the documented `MakeAddRowLambda` dangling-`string_view` trap) · admin metadata for all four tables — `admin_top_level_tables`, `admin_column_data_info` (labels/hints/types; `*_us` columns as date type, readonly for `created_us`/`updated_us`), `admin_table_friendly_names`, `admin_table_display_template` (`categories`/`scene_tags`/`neighborhoods` → `{display_name}`, `venues` → `{name}`) · `allowed_tables` += categories, scene_tags, neighborhoods, venues (public read via the generic endpoints — this is how EV2/GD3 fetch the vocabularies; no bespoke list endpoints). **Remember the #1 trap:** every table goes in `admin_top_level_tables` even when also nested. **Tests:** `create_database_app_seed_test.cpp` (new) — run the app populate functions in a harness transaction (the honuware `create_framework_tables_test.cpp` pattern) and assert: 12 categories, 11 scene tags (3 `link_out`), 24 neighborhoods, exact slugs for a spot-checked handful, vocab tables present in `allowed_tables`.
- [ ] 1.7 **Merge Slice 1 + ping Mason (Stream B unblocked).** Note in the PR: vocabulary values are GD1's; slug renames need a cross-stream ping.

### 2. Slice 2 — db_schema: events core

- [ ] 2.1 `src/db_schema/event_sources.{h,cpp}`.
- [ ] 2.2 `src/db_schema/series.{h,cpp}`.
- [ ] 2.3 `src/db_schema/events.{h,cpp}` — incl. the named unique constraint + `CreateEventsIndexes`.
- [ ] 2.4 `src/db_schema/event_category_assignments.{h,cpp}` + `event_scene_tag_assignments.{h,cpp}`.
- [ ] 2.5 `make_app_tables.cpp` order: … event_sources, series, events, junctions. **Test:** composition test again.
- [ ] 2.6 `create_database.cpp` (three-place rule again: `CreateTables` gains the five tables in FK order **and the `CreateEventsIndexes` call** — indexes only exist on the live DB): admin metadata for the five tables (junctions also in `admin_nested_tables` under `events`) · display templates (`events` → `{title}`, `series` → `{title}`, `event_sources` → `{name}`) · `PopulateAppRolesAndPermissions` gains **`manage_events`** — permission row, granted to the framework `Administrator` role **by name**, and `admin_table_permissions` rows mapping all nine events-domain tables → `manage_events` (so a future non-admin curator role works through the generic CRUD editor) · `PopulatePhotoSupportTables` app half += `venues`, `events`. **Tests:** extend the seed test — `manage_events` exists + admin holds it; photo support rows present; junction tables in top-level + nested metadata.

### 3. table_helpers (`src/sql_util/table_helpers/`, one class per table — model: knottyyoga `class_schedule_slots.{h,cpp}`)

**This directory doesn't exist yet — this bucket creates the layer:** new `add_subdirectory` line in `src/CMakeLists.txt` + a `CMakeLists.txt` using the house `target_sources(communityfinder_core PRIVATE …)` / `target_sources(communityfinder_test_cases PUBLIC …)` split (see `src/db_schema/CMakeLists.txt` for the shape).

Convention: ctor takes `DatabaseHelper`; methods take `Transaction&`; `KeyValueTable`/`KeyValueTableArray` at boundaries; `Add*` via `DbCrud::AddRowToTableFetchInt64PrimaryKey`; `Update*` bumps `updated_us`; SQL as file-local `constexpr std::string_view kSql*`; deterministic `ORDER BY … , id ASC` tiebreakers.

- [ ] 3.1 `categories.{h,cpp}`, `scene_tags.{h,cpp}`, `neighborhoods.{h,cpp}` — `Add*` (tests/admin), `GetAll` (enabled, ordered by sort_order), `GetBySlug`. **Tests:** one `*_test.cpp` each (insert → fetch by slug → ordering).
- [ ] 3.2 `venues.{h,cpp}` — `AddVenue`, `GetVenue`, `GetVenuesByIds` (batch, for event-list assembly), `GetVenuesByNeighborhood`, `UpdateVenue`, `DeleteVenue`. **Test:** `venues_test.cpp`.
- [ ] 3.3 `event_sources.{h,cpp}` — `AddSource`, `GetSource`, `GetEnabledSources`, `UpdateSource`, `TouchLastScanned(id, nowUs)`. **Test.**
- [ ] 3.4 `series.{h,cpp}` — `AddSeries`, `GetSeries`, `GetSeriesBySlug`, `GetActiveSeries`, `UpdateSeries`. **Test.**
- [ ] 3.5 `events.{h,cpp}` — `AddEvent`, `GetEvent`, `GetEventBySourceAndExternalKey`, `GetEventsInRange(fromUs, toUs, status)` (single-table, `(status, starts_at_us)`-index-shaped), `GetEventsBySeries`, `GetPendingEvents`, `UpdateEvent`. **Test:** `events_test.cpp` — incl. the unique-constraint violation path and NULL-source manual events not colliding.
- [ ] 3.6 `event_category_assignments.{h,cpp}` + `event_scene_tag_assignments.{h,cpp}` — `Assign`, `Remove`, `GetForEvent`, `GetForEvents(ids)` (batch), `GetEventIdsForCategory` / `GetEventIdsForSceneTag`. **Tests.**

### 4. business_logic (`src/business_logic/events/` — model: knottyyoga `class_schedule_helper.h`: request/result structs, error-code strings, no exceptions)

- [ ] 4.1 `event_helper.{h,cpp}` — `EventHelper(DatabaseHelper)` owning the table helpers it needs.
  - `IngestEvents(Transaction&, const std::vector<IngestItem>&) → IngestResult` implementing the upsert pin-down (per-item outcome: `created` / `updated` / `unchanged` / `reverted_to_pending` / `rejected_kept`; batch is one transaction).
  - `ReviewEvent(Transaction&, eventId, ReviewAction, reviewerPersonId, notes) → ReviewResult` (transition validation per the pin-down; stamps the quartet with `now_us()`).
  - `GetUpcomingPublic(Transaction&, const UpcomingQuery&) → std::vector<PublicEvent>` — clamps the range, filters approved, resolves venue (batch), category slugs, scene-tag slugs, series title; optional filters: category slug, scene slug, neighborhood slug, venue_id, series_id.
  - **Tests** (`event_helper_test.cpp`): re-ingesting an identical batch → all `unchanged` (idempotency) · minor change updates silently, status untouched · date change on an approved event → `pending` + note · rejected events stay rejected on re-ingest · review transitions (incl. invalid ones → error codes) · visibility: pending/rejected/archived never in `GetUpcomingPublic` · range clamp caps at 92 days · filters by category/scene/neighborhood.
- [ ] 4.2 `events_key_value_table.{h,cpp}` — domain→wire: `PublicEventToKeyValueTable` (id, title, description, starts_at_us, ends_at_us, venue_id, venue_name, neighborhood slug, location_text, url, cost_text, series_id, series_slug, series_title, origin; **never** the reviewer quartet on public shapes), plus array forms. **Test:** field-presence + redaction of review fields.

### 5. endpoints (`src/endpoints/` — model: knottyyoga `get_calendar.cpp`; PascalCase `crow::HTTPMethod` aliases; every TU = handler + `SetupRouting` + exported `Endpoints::X`; **volatile anchor in `web_app.cpp` or it 404s in Release**)

- [ ] 5.1 `get_upcoming_events.cpp` — `GET /api/events/upcoming?from_us&to_us&category&scene&neighborhood&venue_id&series_id` (all optional; `ParseIntParam` fallback pattern). Anonymous. Response `{"items":[…]}`. **Test** (`get_upcoming_events_test.cpp`, HTTP via `handle_full` with the file-local request helper that calls `ThreadPool::GetInstance().Shutdown()` before returning): approved-only; each filter; window default + cap; find items by id, never index.
- [ ] 5.2 `get_event_detail.cpp` — `GET /api/events/<int>`. Anonymous sees `approved` (404 otherwise, to keep the URL low-information); a caller holding `manage_events` sees any status (the reviewer's preview — knottyyoga blog's author-preview pattern). **Test:** both behaviors.
- [ ] 5.3 `get_event_series.cpp` — `GET /api/event_series` (active list) and `GET /api/event_series/<string>` (by slug: series fields + its upcoming approved instances). Separate top-level path avoids the `/api/events/<int>` route ambiguity. **Test.**
- [ ] 5.4 `admin_ingest_events.cpp` — `POST /api/admin/ingest_events`, `endpointAuthHelper.RequirePermission(manage_events)`; body `{"items":[{source_id, external_key, title, starts_at_us, ends_at_us?, description?, url?, cost_text?, location_text?, venue_id?, series_slug?, category_slugs?[], scene_tag_slugs?[]}]}`; response per-item outcomes + counts. This is the scanner's (EV5) and any feed importer's single door. **Test:** ingest twice over HTTP → second pass all `unchanged`; anonymous → 401, logged-in-without-permission → 403 (grant the positive case via `EndpointTestHelper::GrantPermissionToPerson(tx, personId, "manage_events")`); malformed item → 400 `ValidationError` with the item index.
- [ ] 5.5 `admin_review_event.cpp` — `POST /api/admin/review_event/<int>` body `{"action":"approve"|"reject","notes":…}`, `RequirePermission(manage_events)`. **Test:** approve → appears in `/api/events/upcoming`; reject → doesn't; quartet stamped; bad transition → 400 with error code.
- [ ] 5.6 Shared-file one-liners (alphabetized, own-stream lines only): volatile anchors for the five TUs in `endpoints/web_app.cpp`; sources added to the CMakeLists lists; `main.cpp` registers **`PublicPhotoTables` = {events, venues}** via `WebApp::SetService` (anonymous `get_scaled_photo` for event/venue images — the seam CF deliberately left unwired in Phase 2). **Test:** `get_scaled_photo` on a venue photo succeeds anonymously (extend the seed/HTTP tests; upload via the framework endpoint as an admin first). *Pin note: anonymous photo **reads** (this seam) exist at the pinned honuware SHA `3b1c3dc`; photo **uploads by non-admin curators** would need `RequireTableWriteAccess`, which exists only at honuware HEAD `c258a8c` — admin uploads work today, and a re-pin is a separate cross-repo bump-point if curator uploads are ever wanted.*

### 6. test_helper simulation commands (`src/test_helper/commands/` — the SUTP Q10 standing convention)

- [ ] 6.1 `events_commands.{h,cpp}`: `create-venue <name>` · `create-event <title> --starts-in <hours> [--venue <id>] [--status pending|approved] [--past]` · `ingest-demo` (a canned 6-item batch exercising created/updated/reverted) · `list-upcoming`. Registered in the command registry; each command prints ids it created.

### 7. Gates (per slice and at bucket exit)

- [ ] 7.1 **Docker suite green per slice** (`server/docker/build_and_test.sh`), Linux, against the pinned honuware SHA (no co-dev override expected — this bucket is CF-only).
- [ ] 7.2 **Raise the test-count floor** in `build_and_test.sh` at bucket exit to (new actual − ~10) — the floor is the Release dead-strip alarm; leaving it at the old value defeats it.
- [ ] 7.3 **Live seed gate (Mason):** `--recreate_database` (destructive-gated) exits 0; psql spot-check: 12 categories / 11 scene tags / 24 neighborhoods, `manage_events` granted to Administrator, events tables in the admin metadata + `allowed_tables` as specced. (The suite never runs the seed — this live pass is the only seed gate.)
- [ ] 7.4 **Manual loop (Mason or Levi, Windows or Linux):** enter a venue + event via the admin CRUD editor (`/admin/tables`) → `POST /api/admin/review_event` approve (or edit status via CRUD) → `curl /api/events/upcoming` shows it; re-run `ingest-demo` → idempotent.

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **Vocabulary reads ride `allowed_tables` + generic `get_table_rows`** (no bespoke `/api/categories` endpoints). Cheapest thing that works; EV2/GD3 sort by `sort_order` client-side. *Rec: yes; a bespoke endpoint can wrap it later without breaking clients if payload shaping is ever needed.*
	- Mason- This sentence makes no sense. What does "vocabulary reads ride" mean? Please rephrase this with a better description.
2. **Datetime entry in the generic admin editor:** `admin_column_data_info`'s input types are text/number/bool/date only — if the date control is date-only (no time-of-day) for `starts_at_us`, hand-entering event times via generic CRUD is clunky. *Rec: verify during §2.6; if clunky, don't extend honuware now — the `ingest-demo`/`create-event` test_helper commands cover dev, EV3's submit form becomes the real human intake, and EV2 may add a small `manage_events`-gated "quick add" form if the manual loop needs it before EV3.*
	- Mason- We should have a bespoke UI for editing these entries so we won't be relying on the generic CRUD editor to do this. That said, I think we should extend the generic CRUD editor to support date only items with no time as a first class scenario (which I know will probably affect server components and web components).
3. **"Minor vs major" ingest fields:** title is classed **minor** (silent update) — only date/venue changes revert approval, per SUTP Q4c's wording. *Rec: as pinned; a retitle that survives review isn't worth re-review friction.*
4. **Series URL namespace `/api/event_series`** (not `/api/events/series`) to avoid colliding with `/api/events/<int>`. Client-facing page URLs (EV2) stay `/events/series/:slug` — only the API path differs. *Rec: as pinned.*
5. **`categories` on series:** series carry no direct category/scene assignments at MVP — their instances carry them, and the series page derives chips from its instances. *Rec: as pinned; add `series_scene_tag_assignments` later only if a series page needs tags with zero upcoming instances.*
6. **GET-with-query-params for the public list endpoints** follows the knottyyoga app precedent (`/api/calendar?from_us=…`); the honuware *framework* endpoints use POST-JSON-bodies instead. GETs keep the feeds linkable/cacheable. *Rec: as pinned.*
