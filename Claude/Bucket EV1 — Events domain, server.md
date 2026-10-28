---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Levi Kuhn (owner) · Mason Bendixen
Last Updated: 10/28/2026
Version: 0.9
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

**Out (explicit non-goals):** the public UI, ICS feeds, JSON-LD, and the bespoke `manage_events` event-editor admin page (**EV2** — the editor was decided in OQ2; EV1's server surface already supports it via generic CRUD + the review endpoint) · public submission form (**EV3** — but `origin = 'user_submitted'` is reserved now) · archive job + scheduler (**EV4** — the `archived` status exists now, nothing sets it yet) · `scan_runs` + the scanner itself (**EV5**, per SUTP Q4b) · `venue_profiles` and all guide tables (**GD2**, Stream B — never edit their files) · seeding the ~40 real event sources (**GD4a** content-ops, via admin CRUD once this lands).

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
- **`neighborhoods`** — `slug` (unique), `display_name`, `region` (string, CHECK in GD1 §3's five regions), `sort_order`, `enabled`. Seed: GD1 §3's twenty-five *(the draft said twenty-four; GD1 §3 lists 25 — 10 core, 5 south, 1 north, 6 Eastside, 3 south-king — and the seed follows GD1 verbatim; see OQ7)*.
- **`venues`** — `name`, `description`, `address`, `city`, `state`, `zip`, `url`, `latitude`/`longitude` (nullable `DB_TYPE_DOUBLE` — write-only at MVP, no map view yet), `neighborhood_id` (nullable FK → neighborhoods), `status` (string, CHECK in GD1 §5 enum, default `'open'`), `last_verified_at_us` (nullable BIGINT), `volatility_tier` (CHECK in GD1 §5 tiers, default `'monthly'` — GD1 §5's venue default), `created_us`, `updated_us`. *(Slice 1 as built, 10/26: `lat`/`lng` spelled out per CLAUDE.md's no-abbreviations rule, and `volatility_tier` added per GD1 §5 — see OQ7.)*

**Slice 1 as built — conventions the later slices and GD2 inherit:** slugs are `NOT NULL` + a table-level `UNIQUE` (honuware's `AddColumnUnique` emits `UNIQUE` alone, which still admits NULL); optional text columns are `NOT NULL DEFAULT ''`; CHECK constraints are named `<table>_<column>_check`; the GD1 §5 enum value lists live once in `db_schema/freshness.h` (`kFreshnessStatusSqlValues`, `kVolatilityTierSqlValues`) so both streams' CHECKs spell them identically; seeded `sort_order`s step by 10.

**Slice 2 — events core:**

- **`event_sources`** — `name`, `url`, `kind` (CHECK `IN ('ics_feed','venue_site','ticketing_org','aggregator','editorial_monitor','submission','social')` — merges D4's list with brainstorm C.2's intake types), `venue_id` (nullable FK), `enabled` (default true), `scan_hints` (text, for EV5), `volatility_tier` (CHECK in GD1 §5 tiers — the per-source scan cadence EV5 consumes), `last_scanned_at_us` (nullable), `notes`.
- **`series`** — `title`, `slug` (unique), `description`, `cadence_text` ("2nd Saturdays"), `url`, `venue_id` (nullable FK), `status` (GD1 enum, default `'open'`), `last_verified_at_us` (nullable), timestamps.
- **`events`** — `title`, `description`, `starts_at_us` (NOT NULL), `ends_at_us` (nullable), `venue_id` (nullable FK) + `location_text` (for venue-less events), `url`, `cost_text`, `source_id` (nullable FK → event_sources), `series_id` (nullable FK → series), `external_key`, `origin` (CHECK `IN ('scanned','manual','user_submitted')`), `status` (CHECK `IN ('pending','approved','rejected','archived')`, default `'pending'`), `created_by_person_id` (nullable person id, **not an FK** — decided OQ8 (e)), `reviewed_by_person_id` (nullable person id, not an FK), `reviewed_us` (nullable), `review_notes`, timestamps.
  - **`AddNamedUniqueConstraint` on `(source_id, external_key)`** — the ingestion idempotency anchor (Postgres allows repeated NULL `source_id`, so manual events don't collide).
  - `CreateEventsIndexes`: `(status, starts_at_us)` (the upcoming query), `(series_id)`, `(venue_id)` — raw `CREATE INDEX IF NOT EXISTS` DDL with a comment naming the query each serves, knottyyoga-style.
- **`event_category_assignments`** — `event_id` FK, `category_id` FK, `AddUniqueConstraint(event_id, category_id)`.
- **`event_scene_tag_assignments`** — `event_id` FK, `scene_tag_id` FK, unique pair.
- **`series_category_assignments`** — `series_id` FK, `category_id` FK, unique pair *(decided OQ5)*.
- **`series_scene_tag_assignments`** — `series_id` FK, `scene_tag_id` FK, unique pair *(decided OQ5)*.

**Slice 2 as built (10/27)**, following Slice 1's conventions. Each call below is listed in OQ8.
- `series` gained `volatility_tier` (default `'monthly'`), per OQ7(a) and GD1 §5.
- `event_sources.volatility_tier` defaults to `'weekly'`.
- `events.origin` defaults to `'manual'`.
- `events.external_key` is **nullable**, unlike the house `''` rule for optional text.
- The idempotency constraint is named `events_source_id_external_key_key`, matching how Postgres names the other unique constraints.
- `CreateEventsIndexes` builds `events_status_starts_at_idx`, `events_series_idx` and `events_venue_idx`.
- The assignment tables carry `created_us` only (they are never updated), and use the plain unnamed `UNIQUE` pair.
- `kPermissionManageEvents` sits in `db_schema/events.h`, beside the table it gates (knottyyoga's `author_blog` pattern).
- `events.created_by_person_id` and `reviewed_by_person_id` are plain nullable BIGINTs, not FKs, so deleting an account never deletes events (OQ8 (e), option 2). Re-linking them waits on honuware ledger item **H11** (`ON DELETE SET NULL`).

No image columns anywhere — photos ride the framework pipeline (`table_item_photos`), enabled in §1.6/§2.6 below.

## Behavior pins

- **Deletes cascade.** honuware FK constraints are emitted `ON DELETE CASCADE`, always — deleting a venue deletes its events; deleting a series deletes its instances; deleting a category deletes its assignments. Operational rule everywhere in this domain: **close/archive, don't delete** (that's also what the closure graveyard and freshness convention want); the admin editor's delete is for data-entry mistakes only.
- **Visibility rule:** public = `status == 'approved'`. (`archived` is its own status, so "approved ∧ ¬archived" collapses to the equality; EV4 owns the transition to `archived`.)
- **Category/scene-tag inheritance (decided OQ5):** series carry their own category and scene-tag assignments; instances inherit them. An event's own assignments **override per axis** — if the event has any category assignments of its own, those *replace* the series' categories for that event (and scene tags override independently the same way); with none of its own, the series' set applies. Everything public-facing operates on the **effective** (post-inheritance) sets: `GetUpcomingPublic`'s payloads and its category/scene filters both resolve own-else-series in `EventHelper`, in memory after the range query — at MVP volumes that beats encoding the inheritance in SQL. Ingest items carrying `category_slugs`/`scene_tag_slugs` set event-level assignments (i.e., overrides); items without them leave inheritance in effect.
- **Ingest upsert (SUTP Q4c, adopted-by-default):** match on `(source_id, external_key)`. New → insert as `pending`/`scanned`. Existing → **minor** field changes (title, description, url, cost_text, location_text, series/category hints) update silently keeping current status; **major** changes (`starts_at_us`, `ends_at_us`, `venue_id`) update AND revert an `approved` event to `pending` with a review note ("date/venue changed by re-scan"). Unchanged → no-op, reported as such. Never resurrect a `rejected` event (re-ingesting a rejected key reports `rejected_kept` and changes nothing) — that's what makes reject meaningful against a weekly scanner.
- **Review:** `pending → approved | rejected` (also allowed from the other of the two, for corrections); sets the reviewer quartet. `archived` is terminal except back to `approved` (un-archive; rare, admin CRUD can do it raw).
- **Range clamp on public queries:** default window now → +42 days (knottyyoga's calendar default); hard cap 92 days per request; `from_us` may reach back 1 day (time-zone slack). EV2's calendar pages through windows.

# Layered work items

*(Numbered; lower layers first; every item names its tests. Slice 1 = §1; land it, ping Stream B, then proceed.)*

### 1. Slice 1 — db_schema: shared vocabulary + venues

*(Slice 1 built 10/26 — 1.1–1.6 green on both platforms: Windows 1930/1930, Linux docker gate 1930/1930, floor 1700 untouched (raised at bucket exit per §7.2). Live seed check passed 10/27 (see §7.3's note). **Complete — merged 10/27 as `43b8437`.**)*

- [x] 1.1 `src/db_schema/categories.{h,cpp}` — constants + `MakeCategoriesTable(DatabaseInfo)` per the schema pin-down. Model: knottyyoga `src/db_schema/class_schedules.{h,cpp}`.
- [x] 1.2 `src/db_schema/scene_tags.{h,cpp}` — incl. the `coverage` CHECK.
- [x] 1.3 `src/db_schema/neighborhoods.{h,cpp}` — incl. the `region` CHECK.
- [x] 1.4 `src/db_schema/venues.{h,cpp}` — incl. `MakeVenuesTable` + (empty-ok) `CreateVenuesIndexes`. *(Plus the shared `src/db_schema/freshness.h` — GD1 §5's status + tier value lists, for GD2 to reuse.)*
- [x] 1.5 `make_app_tables.cpp`: register in FK order — categories, scene_tags, neighborhoods, venues. **Test:** update `make_database_info_test.cpp` — **its `AppStreamIsEmpty` and `ComposedSchemaIsExactlyTheFrameworkTableSet` tests exist precisely to be replaced by the first app table** (they fail the moment 1.5 lands); rewrite them to assert the app table set, plus the four tables' expected columns.
- [x] 1.6 `create_database.cpp` app half (follow knottyyoga's `create_database.cpp` idioms + CF's existing `PopulateAppRolesAndPermissions` scaffold). **The three-place rule** — a new table touches all three: `CreateTables` (live-DB DDL execution in FK order; the test harness builds from `MakeDatabaseInfo` instead, so forgetting this only breaks the live recreate), `PopulateTables`, and `make_app_tables.cpp` (done in 1.5). Then: `PopulateVocabularies(...)` seeding GD1 §1–§3 **verbatim** (slug, display_name, coverage/region, sort_order; seed lambdas capture `databaseHelper`/`tableName` **by value** — the documented `MakeAddRowLambda` dangling-`string_view` trap) · admin metadata for all four tables — `admin_top_level_tables`, `admin_column_data_info` (labels/hints/types; `*_us` columns as date type, readonly for `created_us`/`updated_us`), `admin_table_friendly_names`, `admin_table_display_template` (`categories`/`scene_tags`/`neighborhoods` → `{display_name}`, `venues` → `{name}`) · `allowed_tables` += categories, scene_tags, neighborhoods, venues (public read via the generic endpoints — this is how EV2/GD3 fetch the vocabularies; no bespoke list endpoints). **Remember the #1 trap:** every table goes in `admin_top_level_tables` even when also nested. **Tests:** `create_database_app_seed_test.cpp` (new) — run the app populate functions in a harness transaction (the honuware `create_framework_tables_test.cpp` pattern) and assert: 12 categories, 11 scene tags (3 `link_out`), 24 neighborhoods, exact slugs for a spot-checked handful, vocab tables present in `allowed_tables`.
  - *As built (10/26):* `create_database.cpp` compiles into the `communityfinder_database_helper` **executable** with its populate functions in an anonymous namespace, so no test could reach them. The app halves therefore moved into `communityfinder_core` as **`src/business_logic/create_app_tables.{h,cpp}`** — `CreateAppTables` / `AppTableCreationOrder` / `PopulateAppTables`, the exact mirror of honuware's `create_framework_tables` — and `create_database.cpp` now just composes them (framework halves first, then app). The **three-place rule** for an app table is now `make_app_tables.cpp` (`MakeAppTables`) + `create_app_tables.cpp` (`AppTableCreationOrder` and `PopulateAppTables`). The test is **`create_app_tables_test.cpp`**: creation order == `MakeAppTables` order (the guard honuware added after `site_assets`); the **full** ordered slug lists for all three vocabularies copied by hand from GD1 (stronger than a spot-check — drift on either side fails), with coverage per tag and region per neighborhood; `allowed_tables`; every app table top-level + friendly-named + display-templated + column-described; `created_us`/`updated_us` read-only everywhere. Plus `crud_test.cpp` `AnonymousCanReadTheSeededVocabularies` — a signed-out `GET /api/get_table_rows/<table>` returns all 12 / 11 / 25 rows (and venues), proving OQ1's Option A end to end. Admin metadata also gained `admin_column_friendly_names` (list-view headers — knottyyoga checklist step 8).
- [x] 1.7 **Merge Slice 1 + ping Mason (Stream B unblocked).** Note in the PR: vocabulary values are GD1's; slug renames need a cross-stream ping. *(Merged 10/27 as `43b8437` "Slice 1 — db_schema: shared vocabulary + venues".)*

### 2. Slice 2 — db_schema: events core

*(Slice 2 built 10/27 — 2.1–2.6 green on both platforms: Windows 1940/1940, Linux docker gate 1940/1940 (floor 1700). Live seed check passed 10/27 (see §7.3's note). OQ8 decided and (e) carried out (the person columns are plain ids, not FKs). **Complete — committed 10/27 as `5858eeb`.**)*

- [x] 2.1 `src/db_schema/event_sources.{h,cpp}`.
- [x] 2.2 `src/db_schema/series.{h,cpp}`.
- [x] 2.3 `src/db_schema/events.{h,cpp}` — incl. the named unique constraint + `CreateEventsIndexes`.
- [x] 2.4 `src/db_schema/event_category_assignments.{h,cpp}` + `event_scene_tag_assignments.{h,cpp}` + `series_category_assignments.{h,cpp}` + `series_scene_tag_assignments.{h,cpp}` *(the OQ5 pair rides here)*.
- [x] 2.5 `make_app_tables.cpp` order: … event_sources, series, events, then all four junctions. **Test:** composition test again.
- [x] 2.6 `create_database.cpp` (three-place rule again: `CreateTables` gains the seven tables in FK order **and the `CreateEventsIndexes` call** — indexes only exist on the live DB): admin metadata for the seven tables (junctions also in `admin_nested_tables`, under `events` and `series` respectively) · display templates (`events` → `{title}`, `series` → `{title}`, `event_sources` → `{name}`) · `PopulateAppRolesAndPermissions` gains **`manage_events`** — permission row, granted to the framework `Administrator` role **by name**, and `admin_table_permissions` rows mapping all eleven events-domain tables → `manage_events` (so a future non-admin curator role works through the generic CRUD editor) · `PopulatePhotoSupportTables` app half += `venues`, `events`. **Tests:** extend the seed test — `manage_events` exists + admin holds it; photo support rows present; junction tables in top-level + nested metadata. *(After Slice 1: the seed test is `create_app_tables_test.cpp`, and only code in `create_app_tables.cpp` is reachable from it — so `manage_events`, its `admin_table_permissions` rows, and the photo-support rows go into `PopulateAppTables` (or a function it calls), not into `create_database.cpp`'s anonymous namespace. `PopulateAppRolesAndPermissions` there is still the empty stub.)*
  - *As built (10/27):* all of it is in `create_app_tables.cpp`:
    - **`PopulateAppPermissions`:** the `manage_events` row (not pricing-eligible), granted to `Administrator` by name, plus `admin_table_permissions` for the eleven tables in an explicit `kEventsDomainTables` list. It is not a loop over `AppTableCreationOrder`, which will also hold GD2's tables.
    - **`PopulatePhotoSupportTables`:** `venues` and `events`.
    - **Nested tables:** the four assignment tables, inside `PopulateAdminMetadata`.
    - `create_database.cpp`'s empty `PopulateAppRolesAndPermissions` stub is **removed**; its pattern comment moved onto `PopulateAppPermissions`.
    - **None of the seven new tables is in `allowed_tables`.** Pending and rejected events, review notes and scan hints must not be publicly readable; the public reads approved events through §5's endpoints.

    The grant looks up the framework's Administrator role, so the seed tests now run `PopulateFrameworkTables` first (`SeedAsRecreateDatabaseDoes`), exactly as `--recreate_database` does. Tests:
    - **`create_app_tables_test.cpp`** gained:
      - `NestsTheAssignmentTablesUnderTheirParents`;
      - `GrantsManageEventsToTheAdministratorRole`, which also asserts the eleven-table map;
      - `EnablesPhotosForVenuesAndEvents`;
      - `KeepsTheEventPipelineOffThePublicList`.
    - **`make_database_info_test.cpp`** gained five schema tests: columns, CHECKs, defaults, foreign keys, the named unique constraint, and the assignment tables' unique pairs.
    - **`crud_test.cpp`** gained `AnonymousCannotReadTheEventPipeline`, so a signed-out `get_table_rows` on events, series, sources or assignments is refused.
    - Behavior tests for the unique constraint (a duplicate `(source_id, external_key)` fails; hand-entered events don't collide) stay in §3.5 as specced.

### 3. table_helpers (`src/sql_util/table_helpers/`, one class per table — model: knottyyoga `class_schedule_slots.{h,cpp}`)

**This directory doesn't exist yet — this bucket creates the layer:** new `add_subdirectory` line in `src/CMakeLists.txt` + a `CMakeLists.txt` using the house `target_sources(communityfinder_core PRIVATE …)` / `target_sources(communityfinder_test_cases PUBLIC …)` split (see `src/db_schema/CMakeLists.txt` for the shape).

Convention: ctor takes `DatabaseHelper`; methods take `Transaction&`; `KeyValueTable`/`KeyValueTableArray` at boundaries; `Add*` via `DbCrud::AddRowToTableFetchInt64PrimaryKey`; `Update*` bumps `updated_us`; SQL as file-local `constexpr std::string_view kSql*`; deterministic `ORDER BY … , id ASC` tiebreakers.

*(Slice 3 built 10/27 — 3.1–3.6 green on both platforms: Windows 1998/1998, Linux docker gate 1998/1998 (floor 1700); 58 new tests. OQ9 decided (all five kept as built). **Complete — committed 10/27 as `de0a219`.**)*

- [x] 3.1 `categories.{h,cpp}`, `scene_tags.{h,cpp}`, `neighborhoods.{h,cpp}` — `Add*` (tests/admin), `GetAll` (enabled, ordered by sort_order), `GetBySlug`. **Tests:** one `*_test.cpp` each (insert → fetch by slug → ordering).
- [x] 3.2 `venues.{h,cpp}` — `AddVenue`, `GetVenue`, `GetVenuesByIds` (batch, for event-list assembly), `GetVenuesByNeighborhood`, `UpdateVenue`, `DeleteVenue`. **Test:** `venues_test.cpp`.
- [x] 3.3 `event_sources.{h,cpp}` — `AddSource`, `GetSource`, `GetEnabledSources`, `UpdateSource`, `TouchLastScanned(id, nowUs)`. **Test.**
- [x] 3.4 `series.{h,cpp}` — `AddSeries`, `GetSeries`, `GetSeriesBySlug`, `GetActiveSeries`, `UpdateSeries`. **Test.**
- [x] 3.5 `events.{h,cpp}` — `AddEvent`, `GetEvent`, `GetEventBySourceAndExternalKey`, `GetEventsInRange(fromUs, toUs, status)` (single-table, `(status, starts_at_us)`-index-shaped), `GetEventsBySeries`, `GetPendingEvents`, `UpdateEvent`. **Test:** `events_test.cpp` — incl. the unique-constraint violation path and NULL-source manual events not colliding.
- [x] 3.6 `event_category_assignments.{h,cpp}` + `event_scene_tag_assignments.{h,cpp}` — `Assign`, `Remove`, `GetForEvent`, `GetForEvents(ids)` (batch), `GetEventIdsForCategory` / `GetEventIdsForSceneTag` — plus `series_category_assignments.{h,cpp}` + `series_scene_tag_assignments.{h,cpp}` — `Assign`, `Remove`, `GetForSeries`, `GetForSeriesIds(ids)` (batch, for effective-set resolution). **Tests** (incl. the unique-pair violations).
  - *As built (10/27):*
    - **Layer and model.** `src/sql_util/table_helpers/` now exists, wired in with `add_subdirectory(sql_util/table_helpers)` and the house `PRIVATE` / `PUBLIC` source split. It has one class per table in namespace `TableHelpers`, in knottyyoga's `class_schedule_slots` shape. No file or class name collides with honuware's helpers under the same include prefix.
    - **Writes.** `Add*` / `Assign` go through `DbCrud::AddRowToTableFetchInt64PrimaryKey`. `Update*` sets `updated_us = now_us()`, and `"NULL"` is an allowed value, so an update can clear a nullable column (knottyyoga's idiom).
    - **Reads.** SQL lives in file-local `kSql*` constants with `…, id ASC` tiebreakers. Status names are passed as parameters from the schema constants, never spelled in the SQL.
    - **Batch reads** (`GetVenuesByIds`, `GetForEvents`, `GetForSeriesIds`) bind one `= ANY($1::bigint[])` parameter, built by the new shared `bigint_array_literal.{h,cpp}` (knottyyoga's co-instructors pattern). Empty input returns empty without a query.
    - **Assignment reads join their vocabulary.** Each row carries `category_slug` or `scene_tag_slug`, ordered by the vocabulary's `sort_order` (OQ9 (b)).
    - **`db_schema/freshness.h`** gained the five `kFreshnessStatus*` value constants beside the SQL list (`GetActiveSeries` and the tests use them). GD2 can use them too.
    - **Tests:** 58 across 12 `*_test.cpp` files.
      - Every helper has add/get, a not-found case, ordering with tiebreaks, and update.
      - **Bump tests** insert with `updated_us = 0`, because an insert and the update after it can land in the same microsecond. *(Corrected in Slice 4: this note first said `now_us()` is fixed for the whole transaction. It isn't. It reads the wall clock (`clock_timestamp()`). The tests were right; only the reason was wrong.)*
      - Each unique violation (slugs, the four pairs, `(source_id, external_key)`) and the `kind` CHECK is the last statement of its test, because the violation aborts the transaction.
      - `events_test` covers both §3.5 behaviors: a duplicate `(source_id, external_key)` throws, and hand-entered events don't collide, whether they have no source or a source with no key.
      - `venues_test` pins the cascade behavior: deleting a venue deletes its events.

### 4. business_logic (`src/business_logic/events/` — model: knottyyoga `class_schedule_helper.h`: request/result structs, error-code strings, no exceptions)

*(Slice 4 complete — committed as `a245baf` (10/28). Built 10/27: 4.1–4.2 green on both platforms, Windows 2026/2026 and Linux docker gate 2026/2026 (floor 1700), with 28 new tests. OQ10 decided 10/28 (all seven kept as built).)*

- [x] 4.1 `event_helper.{h,cpp}` — `EventHelper(DatabaseHelper)` owning the table helpers it needs.
  - `IngestEvents(Transaction&, const std::vector<IngestItem>&) → IngestResult` implementing the upsert pin-down (per-item outcome: `created` / `updated` / `unchanged` / `reverted_to_pending` / `rejected_kept`; batch is one transaction).
  - `ReviewEvent(Transaction&, eventId, ReviewAction, reviewerPersonId, notes) → ReviewResult` (transition validation per the pin-down; stamps the quartet with `now_us()`).
  - `GetUpcomingPublic(Transaction&, const UpcomingQuery&) → std::vector<PublicEvent>` — clamps the range, filters approved, resolves venue (batch), **effective** category + scene-tag slugs (own-else-series, per the inheritance pin), series title; optional filters: category slug, scene slug, neighborhood slug, venue_id, series_id — category/scene filters match against the effective sets.
  - **Tests** (`event_helper_test.cpp`): re-ingesting an identical batch → all `unchanged` (idempotency) · minor change updates silently, status untouched · date change on an approved event → `pending` + note · rejected events stay rejected on re-ingest · review transitions (incl. invalid ones → error codes) · visibility: pending/rejected/archived never in `GetUpcomingPublic` · range clamp caps at 92 days · filters by category/scene/neighborhood · **inheritance:** an instance with no assignments of its own matches its series' category/scene filters and carries the series' slugs; an instance with its own category assignments overrides them (a series-only category no longer matches it, while its scene tags still inherit).
- [x] 4.2 `events_key_value_table.{h,cpp}` — domain→wire: `PublicEventToKeyValueTable` (id, title, description, starts_at_us, ends_at_us, venue_id, venue_name, neighborhood slug, location_text, url, cost_text, effective category + scene-tag slugs, series_id, series_slug, series_title, origin; **never** the reviewer quartet on public shapes), plus array forms. **Test:** field-presence + redaction of review fields.
  - *As built (10/27):*
    - **Files.** `src/business_logic/events/` holds `event_helper.{h,cpp}`, `events_key_value_table.{h,cpp}`, their two tests and a `CMakeLists.txt` (house `PRIVATE`/`PUBLIC` split). It is wired in with `add_subdirectory(events)` in `business_logic/CMakeLists.txt`. Namespace `Events`.
    - **Shapes.** `IngestItem` uses `std::optional` for every optional field, because "absent" and "empty" mean different things (OQ10 (a)). Every result has `ok` and `errorCode`. `IngestResult` adds `errorItemIndex` plus one `IngestItemResult` per item: the event id, the outcome, and any slugs that were ignored. `PublicEvent` has no review or author fields, so a public response can't leak them.
    - **Error codes.** Ingest: `SOURCE_NOT_FOUND`, `MISSING_EXTERNAL_KEY`, `MISSING_TITLE`, `MISSING_START`, `END_BEFORE_START`, `VENUE_NOT_FOUND`. Review: `EVENT_NOT_FOUND`, `INVALID_TRANSITION`.
    - **Ingest outcomes.** The spec's five, plus `archived_kept` (OQ10 (d)).
    - **Ingest runs in two passes.** The first checks every item and looks up its slugs. The second writes. One bad item fails the batch before anything is written, and `errorItemIndex` says which.
    - **Ingest writes.**
      - Every change goes through `UpdateEvent`, so a change that only touches categories or scene tags still bumps `updated_us`.
      - Each item's own assignments are adjusted by difference: only the ones that changed are added or removed.
      - A major change on an approved event:
        - sets status `pending`;
        - sets the note "date/venue changed by re-scan";
        - keeps who reviewed it and when.
      - A major change on a pending event just updates it.
    - **The window.** `ClampUpcomingWindow(now, from, to)` is a free function, so Slice 5 can echo the window it served. It is tested without a database. Constants: `kUpcomingDefaultWindowDays` 42, `kUpcomingMaxWindowDays` 92, `kUpcomingLookBackDays` 1. A backwards request returns an empty window.
    - **`GetUpcomingPublic`** makes 9 queries at most, however many events match:
      1. the clock (`now_us()`);
      2. the range query;
      3. venues, by id;
      4. neighborhoods (the enabled list);
      5. series, by id;
      6. through 9. own and series categories and scene tags.

      Venue and series filters run before the lookups, so those only cover events that can still be returned. The neighborhood, category and scene filters run after the inherited sets are worked out. Start order is kept.
    - **One helper added below:** `TableHelpers::Series::GetSeriesByIds` (a batch read, ordered by id, with an empty-input early return), with a test.
    - **Key-value table.**
      - The 17 public fields only.
      - A missing end, venue or series is an empty string, so the client reads it as null.
      - Slug lists are comma-joined (OQ10 (f)).
    - **Tests (28 new):**
      - `event_helper_test` (19). It covers every spec'd case plus:
        - venue changes;
        - a major change on a pending event;
        - archived events;
        - "absent keeps a curator's edit";
        - stated-empty values clearing;
        - ignored and reported slugs;
        - all-or-nothing batches;
        - one case per ingest error code;
        - review of archived and unknown events;
        - venue, neighborhood and series resolution.
      - `ClampUpcomingWindowTest` (4).
      - `events_key_value_table_test` (4): every field, the redaction list, empty strings for missing values, array order.
      - `series_test` (+1).
    - **Found while testing:** `now_us()` is `clock_timestamp()`, the wall clock, not the transaction start. So a `now_us()` stamp is at or after one read earlier in the same transaction, never equal to it. CLAUDE.md's testing bullet and four table-helper test comments that said otherwise are corrected.

### 5. endpoints (`src/endpoints/` — model: knottyyoga `get_calendar.cpp`; PascalCase `crow::HTTPMethod` aliases; every TU = handler + `SetupRouting` + exported `Endpoints::X`; **volatile anchor in `web_app.cpp` or it 404s in Release**)

*(Slice 5 built 10/28 — 5.1–5.6 green on both platforms: Windows 2058/2058, Linux docker gate 2058/2058 (floor 1700); 32 new tests. OQ11 decided 10/28 (all six kept as built). **Complete — committed 10/28 as `b57ce19`.**)*

- [x] 5.1 `get_upcoming_events.cpp` — `GET /api/events/upcoming?from_us&to_us&category&scene&neighborhood&venue_id&series_id` (all optional; `ParseIntParam` fallback pattern). Anonymous. Response `{"items":[…]}`. The KVT's comma-joined `category_slugs` / `scene_tag_slugs` go out as real JSON arrays, nested here at the endpoint as knottyyoga's `get_calendar.cpp` does for tags (OQ10 (f)); §5.3's slug lists do the same. **Test** (`get_upcoming_events_test.cpp`, HTTP via `handle_full` with the file-local request helper that calls `ThreadPool::GetInstance().Shutdown()` before returning): approved-only; each filter; window default + cap; find items by id, never index.
- [x] 5.2 `get_event_detail.cpp` — `GET /api/events/<int>`. Anonymous sees `approved` (404 otherwise, to keep the URL low-information); a caller holding `manage_events` sees any status (the reviewer's preview — knottyyoga blog's author-preview pattern). **Test:** both behaviors.
- [x] 5.3 `get_event_series.cpp` — `GET /api/event_series` (active list, each with its own category/scene-tag slugs) and `GET /api/event_series/<string>` (by slug: series fields + its category/scene-tag slugs + its upcoming approved instances). Separate top-level path avoids the `/api/events/<int>` route ambiguity. **Test:** incl. series slugs present in both shapes.
- [x] 5.4 `admin_ingest_events.cpp` — `POST /api/admin/ingest_events`, `endpointAuthHelper.RequirePermission(manage_events)`; body `{"items":[{source_id, external_key, title, starts_at_us, ends_at_us?, description?, url?, cost_text?, location_text?, venue_id?, series_slug?, category_slugs?[], scene_tag_slugs?[]}]}`; response per-item outcomes + counts. This is the scanner's (EV5) and any feed importer's single door. **Test:** ingest twice over HTTP → second pass all `unchanged`; anonymous → 401, logged-in-without-permission → 403 (grant the positive case via `EndpointTestHelper::GrantPermissionToPerson(tx, personId, "manage_events")`); malformed item → 400 `ValidationError` with the item index.
- [x] 5.5 `admin_review_event.cpp` — `POST /api/admin/review_event/<int>` body `{"action":"approve"|"reject","notes":…}`, `RequirePermission(manage_events)`. **Test:** approve → appears in `/api/events/upcoming`; reject → doesn't; quartet stamped; bad transition → 400 with error code.
- [x] 5.6 Shared-file one-liners (alphabetized, own-stream lines only): volatile anchors for the five TUs in `endpoints/web_app.cpp`; sources added to the CMakeLists lists; `main.cpp` registers **`PublicPhotoTables` = {events, venues}** via `WebApp::SetService` (anonymous `get_scaled_photo` for event/venue images — the seam CF deliberately left unwired in Phase 2). **Test:** `get_scaled_photo` on a venue photo succeeds anonymously (extend the seed/HTTP tests; upload via the framework endpoint as an admin first). *Pin note: anonymous photo **reads** (this seam) exist at the pinned honuware SHA `3b1c3dc`; photo **uploads by non-admin curators** would need `RequireTableWriteAccess`, which exists only at honuware HEAD `c258a8c` — admin uploads work today, and a re-pin is a separate cross-repo bump-point if curator uploads are ever wanted.*
  - *As built (10/28):*
    - **Routes.** All anonymous unless noted:
      - `GET /api/events/upcoming` → `{"items", "from_us", "to_us"}` (OQ11 (a)).
      - `GET /api/events/<int>` → the event object. A `manage_events` holder also gets `status` and `review_notes` (OQ11 (b)).
      - `GET /api/event_series` → `{"items"}`.
      - `GET /api/event_series/<string>` → `{"series", "upcoming"}` (OQ11 (c)).
      - `POST /api/admin/ingest_events` and `POST /api/admin/review_event/<int>`, both behind `RequirePermission(manage_events)`, which is checked before the body is read.
    - **Files.** One TU per route family: `get_upcoming_events`, `get_event_detail`, `get_event_series` (both series routes; one anchor pulls in the whole TU), `admin_ingest_events`, `admin_review_event`. Five volatile anchors in `web_app.cpp`.
    - **`events_json.{h,cpp}`** turns the key-value tables into JSON for all three read endpoints. The slug lists become real arrays (OQ10 (f)). *Found while building:* the framework's converter turns any all-digit value into a number, so a cost of "10" or a title like "1999" would have reached the browser as a number. The free-text fields are put back as strings.
    - **business_logic additions.** `GetUpcomingPublic` now returns `UpcomingEvents` (the clamped window plus the events) instead of a bare list, so the endpoint can echo the window. New on `EventHelper`: `GetEventDetail`, `GetActiveSeries`, `GetSeriesDetail`. They share its batched lookups, factored into private `AssemblePublicEvents` / `AssemblePublicSeries`. New in the KVT layer: `PublicSeriesToKeyValueTable` (13 fields).
    - **Photos.** `main.cpp`'s setting moved into `Endpoints::RegisterAppServices(webApp)` (`register_app_services.{h,cpp}`), so the test checks the same setup the server runs (OQ11 (f)).
    - **Tests (32 new):**
      - One test file per endpoint, sharing plain helpers in `events_endpoint_test_support.{h,cpp}` (not a fixture):
        - `get_upcoming_events_test` (4)
        - `get_event_detail_test` (3)
        - `get_event_series_test` (3)
        - `admin_ingest_events_test` (6)
        - `admin_review_event_test` (5)
      - Also `events_json_test` (4), `register_app_services_test` (2), `event_helper_test` (+3) and `events_key_value_table_test` (+2).
      - Results are compared as id sets, never by position.
      - The 403 tests first grant `manage_events` to another user, so the permission exists and the caller simply lacks it.
    - **Windows gotcha.** In `register_app_services_test.cpp`, the boost gil/jpeg headers must come *after* the app headers. Otherwise libjpeg defines `INT32` before `<windows.h>` does, and MSVC fails with C2371.

### 6. test_helper simulation commands (`src/test_helper/commands/` — the SUTP Q10 standing convention)

*(Slice 6 built 10/28 — 6.1 green on both platforms: Windows 2068/2068, Linux docker gate 2068/2068 (floor 1700); 10 new tests. The test_helper exe also builds on Linux, and on Windows `help` and `list-upcoming` ran against the dev database. Remaining: OQ12 and the commit.)*

- [x] 6.1 `events_commands.{h,cpp}`: `create-venue <name>` · `create-event <title> --starts-in <hours> [--venue <id>] [--status pending|approved] [--past]` · `ingest-demo` (a canned 6-item batch exercising created/updated/reverted) · `list-upcoming`. Registered in the command registry; each command prints ids it created.
  - *As built (10/28):*
    - **Commands** (category "Events", alias in brackets). Flags are `--key=value` (OQ12 (a)):
      - `create-venue [cv] --name="The Cuff" [--neighborhood=capitol-hill]`
      - `create-event [ce] --title="Bear Night" --starts_in_hours=5 [--venue_id=N] [--status=pending|approved] [--past]`
      - `ingest-demo [demo] [--changed] [--approve]` (OQ12 (b))
      - `list-upcoming [lu] [--category=…] [--scene=…] [--neighborhood=…]`
    - **What each does.**
      - `create-event` makes a hand-entered event: no feed, origin `manual`, created by the logged-in person. `--status=approved` goes through `EventHelper::ReviewEvent`, so the review fields are stamped (OQ12 (c)).
      - `ingest-demo` files its six events under an ICS source named "test_helper demo feed". It creates that source the first time and reuses it after. Each run prints a table of key, event id, outcome, status and ignored slugs, then the counts.
      - `list-upcoming` is `GetUpcomingPublic`, so it shows exactly what `/api/events/upcoming` serves.
    - **Wiring.**
      - `RegisterEventsCommands` is called from `RegisterAllCommands`.
      - `main.cpp` has a one-shot absl flag for each new argument. For example: `communityfinder_test_helper --command=create-event --title="Bear Night" --starts_in_hours=5 --status=approved`.
      - The REPL's `ParseCommandLine` now honors double quotes, so a value can contain spaces.
    - **Tests (10 new).** `command_registry.cpp` and the events commands also build into `communityfinder_test_cases` (OQ12 (e)):
      - `command_registry_test` (4): quoted values, bare flags, an empty quoted value, a blank line.
      - `events_commands_test` (6): each command is registered with its alias; `ingest-demo` twice leaves everything unchanged, and `--changed` makes one event go back to pending and two stay approved with their edits; approving needs a logged-in person; `create-event` timing, approval and `--past`; bad input is refused with nothing written; `create-venue` with a neighborhood, then an event at that venue.
      - The tests check what ends up in the database, not the printed text.
    - **Windows gotchas.**
      - `<windows.h>` (from `console_colors.h`) defines `CreateEvent` as a macro, so the command functions are named `RunCreateEvent` and so on.
      - In a gtest `TEST` body, a free function named `Run` is hidden by the inherited `testing::Test::Run`, so the test helper is `RunCommand`.

### 7. Gates (per slice and at bucket exit)

- [ ] 7.1 **Docker suite green per slice** (`server/docker/build_and_test.sh`), Linux, against the pinned honuware SHA (no co-dev override expected — this bucket is CF-only).
- [ ] 7.2 **Raise the test-count floor** in `build_and_test.sh` at bucket exit to (new actual − ~10) — the floor is the Release dead-strip alarm; leaving it at the old value defeats it.
- [x] 7.3 **Live seed gate (Mason):** `--recreate_database` (destructive-gated) exits 0; psql spot-check: 12 categories / 11 scene tags / 25 neighborhoods, `manage_events` granted to Administrator, events tables in the admin metadata + `allowed_tables` as specced. (The suite never runs the seed — this live pass is the only seed gate.)
	- *Slice 1 portion passed 10/27* (Claude ran it at Mason's request with the Windows x64-Debug `communityfinder_database_helper.exe`, env from `launch_defaults.local.json`): exit 0; psql shows 12 categories, 11 scene tags (3 `link_out`: `qtpoc`, `sapphic`, `trans`), 25 neighborhoods across all 5 regions, 0 venues. All three CHECK constraints are present, as is the `venues.neighborhood_id` FK. All four tables are in `allowed_tables`, `admin_top_level_tables`, friendly names and display templates. `mail_app_password` is seeded. The `manage_events` and events-table items wait for Slice 2, so this box stays open until bucket exit.
	- *Slice 2 portion passed 10/27* (same exe and env, re-run on the Slice 2 working tree): exit 0. The seven new tables are present and empty. All the CHECK constraints (source kind, both volatility tiers, series status, event origin and status) and UNIQUE constraints (series slug, `events_source_id_external_key_key`, the four assignment pairs) are present, along with the three `events` indexes. Every FK is `ON DELETE CASCADE`, including the two person FKs (OQ8 (e)). *(Those two were dropped after this check, per OQ8. That change is DDL only and doesn't touch the seed. The schema test pins it, and the next `--recreate_database` will show it.)* `manage_events` (not pricing-eligible) is granted to `admin`, and `admin_table_permissions` maps it to all 11 events-domain tables. All 11 are in `admin_top_level_tables` with friendly names and display templates, and the 4 assignment tables are also in `admin_nested_tables`. `photo_support_tables` = events, people, venues. `allowed_tables` = categories, neighborhoods, permission_implications, scene_tags, venues, so none of the event pipeline is public. Column data info has starts/ends as `date`, and the reviewer, creator and timestamp columns are read-only. A rolled-back insert test accepted two hand-entered events (pending, manual, NULL key). It rejected status `published`, source kind `rss`, and a second insert of the same feed item. The new source defaulted to `weekly`. **Everything §7.3 specifies now passes, so the box is checked.** Re-run it if a later slice changes the seed.
- [ ] 7.4 **Manual loop (Mason or Levi, Windows or Linux):** enter a venue + event via the admin CRUD editor (`/admin/tables`; if the generic date control fights you on times-of-day, use the `create-event` test_helper command — the bespoke event editor arrives in EV2 per OQ2) → `POST /api/admin/review_event` approve (or edit status via CRUD) → `curl /api/events/upcoming` shows it; re-run `ingest-demo` → idempotent.
	- *The test_helper half, as built (Slice 6).* In the REPL (`communityfinder_test_helper --repl`), which logs you in automatically:
		1. `create-venue --name="The Cuff" --neighborhood=capitol-hill` prints the venue id.
		2. `create-event --title="Bear Night" --starts_in_hours=5 --venue_id=<id> --status=approved` prints the event id.
		3. `list-upcoming`, or `curl http://localhost:18081/api/events/upcoming`, shows it.
		4. `ingest-demo --approve` reports 6 created, all approved.
		5. `ingest-demo` reports 6 unchanged (the idempotency check).
		6. `ingest-demo --changed` reports 3 unchanged, 2 updated and 1 `reverted_to_pending`. That one is Leather Night, now pending with the re-scan note.
	- These write to the dev database. The demo events are prefixed "Demo:", and the admin editor's delete clears them.

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **How should the website fetch the three vocabulary lists (categories / scene tags / neighborhoods)?** *(Expanded 8/14, round 2, with the alternatives spelled out.)* The vocabularies are the three small lookup tables Slice 1 creates and seeds: the category list (bar, drag, pride, …), the scene tags (bear, leather, sober, …), and the neighborhoods (capitol-hill, bellevue, …). The public pages need those lists in the browser to build their filter dropdowns and chips (EV2's events pages now, GD3's guide pages later). This question is only about **how the Angular app gets those lists from the server** — two workable options:
	- **Option A — reuse the framework's generic read endpoints (recommended).** honuware already ships generic table-read endpoints (`GET /api/get_table_rows/<table_name>` and friends — the same machinery the admin data editor uses). Which tables they may serve is controlled by *data*, not code: any table listed in the `allowed_tables` DB table is readable through them by anyone, logged-out visitors included. So §1.6 simply seeds `categories`/`scene_tags`/`neighborhoods` (+ `venues`) into `allowed_tables`, and the client reads the lists through endpoints that already exist. *Pros:* zero new server code and zero new endpoint tests; admin edits to the lists show up immediately; one mechanism instead of two. *Cons:* the response comes back in the generic rows-and-columns wire shape, so the client converts it to typed objects and sorts by `sort_order` itself (a few lines in the access layer); the payload carries every column rather than a tailored shape (harmless here — these tables hold nothing but public display data); it makes those tables world-readable, which is a per-table decision to make consciously (these lists exist precisely to be public).
	- **Option B — write dedicated list endpoints** (`GET /api/categories`, `/api/scene_tags`, `/api/neighborhoods`), each returning a purpose-built, pre-sorted JSON array. *Pros:* tailored payloads with exactly the fields we choose; no reliance on the generic mechanism for public pages. *Cons:* three new endpoint files + volatile anchors + HTTP tests to write and maintain, for what are small, admin-editable lookup lists; duplicates a capability the framework already provides; every future vocabulary means writing another endpoint.
	Either way, the client hides the choice behind typed access methods (`getCategories()` etc.), so switching later is a client-internal change — starting with A and adding B if a shaped payload is ever actually wanted breaks nothing. *Rec: Option A.*
	- Mason- This sentence makes no sense. What does "vocabulary reads ride" mean? Please rephrase this with a better description.
	- Claude- Rewritten above (8/14). Plain version of the jargon: "ride X" meant "are served by the existing generic endpoints instead of new bespoke ones." The actual decision is still open — **"agreed" locks the no-new-endpoints approach.**
	- Mason- Can you explain the vocabulary thing again? You are asking me a question but I'm not sure that you have explained what the alternatives are that I'm choosing from. Can you explain what you are asking and what the pros and cons are?
	- Claude- Expanded above (8/14, round 2) into an explicit **Option A vs Option B** with pros and cons. Short version: the site's filter UIs need the category/scene/neighborhood lists from the server. **A** = no new code — mark the tables publicly readable in `allowed_tables` and read them through the framework's existing generic table-read endpoints (client does a small convert-and-sort). **B** = write three dedicated list endpoints with tailored payloads (more code to build, anchor, and test, for small lookup lists). Recommendation stays **A**; B can be added later at any point without breaking the client. **Pick A or B ("agreed" = A).**
	- Mason- Option A sounds fine.
	- Claude- **Decided (8/14) — Option A.** The doc already implements it (§1.6 seeds the vocabulary tables + `venues` into `allowed_tables`; the client reads them through the existing generic endpoints and sorts by `sort_order`), so no body change was needed. If a shaped payload is ever wanted, Option B's endpoints can be added later without breaking the client.
2. **Datetime entry in the generic admin editor:** `admin_column_data_info`'s input types are text/number/bool/date only — if the date control is date-only (no time-of-day) for `starts_at_us`, hand-entering event times via generic CRUD is clunky. *Rec: verify during §2.6; if clunky, don't extend honuware now — the `ingest-demo`/`create-event` test_helper commands cover dev, EV3's submit form becomes the real human intake, and EV2 may add a small `manage_events`-gated "quick add" form if the manual loop needs it before EV3.*
	- Mason- We should have a bespoke UI for editing these entries so we won't be relying on the generic CRUD editor to do this. That said, I think we should extend the generic CRUD editor to support date only items with no time as a first class scenario (which I know will probably affect server components and web components).
	- Claude- Decided + folded in (8/14), two halves. **(a) Bespoke editor:** a `manage_events`-gated event entry/edit page is now an EV2 work item ([[Bucket EV2 — Events public UI]] §3.7) — pending-queue list, create/edit form with a real date+time picker (Pacific wall-clock in, `_us` out), venue FK picker, category/scene checkboxes, approve/reject buttons. It calls the generic CRUD access + `/api/admin/review_event` directly (the `ManageRolesPage`/`ManageUsersPage` precedent), so **nothing changes server-side in this bucket**; §7.4's manual loop notes the interim path. **(b) Generic-editor date support:** recorded as honuware cross-repo ledger item **H10** in [[Setting up the project]] (server_components: honest first-class date-only semantics for `_us` columns in `admin_column_data_info`; web_components: the matching `@honuware/ui` date control) — it runs the co-dev loop on its own bump-point and gates nothing in this bucket. **One clarification wanted:** I read "date only items with no time as a first class scenario" as *date-without-time fields done properly* (edited as a bare date, stored as UTC-midnight `_us` — the natural unit for `last_verified_at_us`, a publish date, etc.), with times-of-day staying the bespoke editor's job. If you also want a full **date+time** control in the generic editor, say so and H10 grows to include it.
	- Claude- Closed (8/14): with all questions called answered and no objection raised, the **date-only** reading stands adopted-by-default — H10 stays scoped to first-class date-without-time columns ([[Setting up the project]]'s ledger row updated to match), and times-of-day remain the bespoke editor's job (EV2 §3.7). Flag anytime and a date+time control joins H10's scope.
	- Claude- *Checked during §2.6 (10/27), as this question asked.* The generic editor's `date` control in the current `@honuware/ui` (`hw-simple-date`) shows **a date picker and a time picker** by default (time in 20-minute steps) and writes microseconds for BIGINT columns. So `starts_at_us` and `ends_at_us` are seeded as `date`, and entering an event through the generic editor keeps its time of day. The bespoke editor (EV2 §3.7) and H10 stand as decided.
3. **Re-scan update policy — which field changes force an event back through review?** *(Rewritten 8/14 per Mason's ask — original wording was the shorthand "minor vs major ingest fields".)* Context: after an event is approved and publicly visible, its source page gets scanned again (EV5's weekly scanner, or any repeat call to `POST /api/admin/ingest_events`). When a re-ingested item matches an event we already have — same `(source_id, external_key)` — but some field values differ, the server has to decide what to do with the differences. The policy already decided in SUTP Q4c: **minor** changes (description, url, cost_text, location_text) are absorbed silently — our stored copy updates, and an approved event stays approved and public; **major** changes (`starts_at_us`, `ends_at_us`, `venue_id` — the when-and-where) also update our copy, **but knock an approved event back to `pending` with a review note**, so a human confirms the change before it's public again (a silently-moved date could strand someone at the wrong bar on the wrong night). The one call SUTP didn't make, which this question asks: **is a *title* change minor or major?** *Rec: minor — an already-vetted event that gets retitled ("Bear Night" → "Bear Night: Pride Edition") isn't worth forcing back through the review queue.*
	- Mason- This isn't descriptive enough for me to understand what you are talking about. Please elaborate.
	- Claude- Elaborated above (8/14) — the original packed the whole re-scan policy into two words. Everything except the title call is already decided (SUTP Q4c, adopted-by-default); the only ruling needed here: **title change = minor (silent update, stays approved) or major (forces re-review)?** Rec stands: minor.
	- Mason- Yeah, that sounds like a minor change that we should allow the auto update for.
	- Claude- Decided (8/14) — **title = minor**: a re-scan that changes only the title updates it silently and the event stays approved; only date/venue changes force re-review. The ingest behavior pin already listed title in the minor set, so no body change was needed — this locks it.
4. **Series URL namespace `/api/event_series`** (not `/api/events/series`) to avoid colliding with `/api/events/<int>`. Client-facing page URLs (EV2) stay `/events/series/:slug` — only the API path differs. *Rec: as pinned.*
	- Mason- Sure, that sounds fine.
	- Claude- Decided (8/14) — `/api/event_series` stands; the doc already pinned it, no change needed.
5. **`categories` on series:** series carry no direct category/scene assignments at MVP — their instances carry them, and the series page derives chips from its instances. *Rec: as pinned; add `series_scene_tag_assignments` later only if a series page needs tags with zero upcoming instances.*
	 - Mason- Since we are creating this from scratch, let's put this on both the series and allow instances to override.
	- Claude- Decided + folded in (8/14): series now carry their own assignments — new `series_category_assignments` + `series_scene_tag_assignments` tables (schema pin-downs, work items 2.4 / 2.6 / 3.6) — and a new **inheritance behavior pin**: instances inherit the series' sets, and an instance's own assignments **override per axis** (an event with its own category assignments replaces the series' categories for that event; scene tags override independently the same way; no merging). Filters and public payloads use the effective sets (§4.1/§4.2), and series payloads carry their own slugs (§5.3). **One semantics check on "allow instances to override":** I pinned it as *replace-when-present*, not union — if you'd rather an instance's tags be **added to** the series' set instead of replacing it, say "merge" and I'll flip the pin (replace is what lets a one-off special edition shed a series tag that doesn't apply to it).
	- Claude- Closed (8/14): with all questions called answered and no objection raised, **replace-when-present** stands adopted-by-default — an instance with its own assignments on an axis replaces the series' set for that axis; an instance without them inherits. Flipping to merge stays a one-line pin change if a real case ever wants it.
6. **GET-with-query-params for the public list endpoints** follows the knottyyoga app precedent (`/api/calendar?from_us=…`); the honuware *framework* endpoints use POST-JSON-bodies instead. GETs keep the feeds linkable/cacheable. *Rec: as pinned.*
	- Mason- I'm fine with GET but we might need to switch to post eventually if this grows too much.
	- Claude- Agreed + noted (8/14): GET stays. If the filter surface ever outgrows query strings, we add a **POST sibling endpoint alongside the GET** rather than mutating it — subscribe/feed links in the wild keep working.
7. **Slice 1 as built — four calls made during implementation** *(added 10/26)*. Each is already in the body; "Agreed" locks all four, or name the one to undo.
	- **(a) `venues.volatility_tier`** (CHECK in GD1 §5's tiers, default `'monthly'`). The draft schema omitted it, but GD1 §5 — the authoritative freshness definition — says *every* listable table carries it, with venues defaulting to `monthly`, and EV5's verifier consumes it generically. *Rec: keep; and `series` gets the same column in Slice 2 (GD1 §5 default `monthly`).*
	- **(b) `latitude`/`longitude`** instead of `lat`/`lng` (CLAUDE.md: full names, no abbreviations). Nothing reads them yet. *Rec: keep.*
	- **(c) 25 neighborhoods, not 24.** GD1 §3 lists 25 slugs; the draft's "twenty-four" was a miscount, and the seed follows GD1 verbatim. GD1 gives slugs only, so the display names are the common names (incl. "SODO", "U District") — editable in the admin CRUD editor. *Rec: GD1 wins. To drop one, edit GD1 §3 first; the seed test then fails until the seed matches.*
	- **(d) The app seed moved out of `create_database.cpp`** into `communityfinder_core` as `business_logic/create_app_tables.{h,cpp}` (mirroring honuware's `create_framework_tables`), because a test cannot reach code inside the database_helper executable's anonymous namespace. The seed test is `create_app_tables_test.cpp`, not `create_database_app_seed_test.cpp`. *Rec: keep — it's the only way §1.6's seed test can exist; Slice 2's `manage_events` + photo rows go there too.*
	- *Related, not a call:* GD1 itself is still `Status: Draft` with its lock boxes (§1.1–1.3) unchecked and its OQs unanswered. The seed implements GD1 **as currently written**; if review changes a value, edit GD1, the seed (`create_app_tables.cpp`), and its test together.
8. **Slice 2 as built — four calls, plus one problem to decide** *(added 10/27)*. "Agreed" locks (a)–(d) and takes (e)'s recommendation; or name the ones to change.
	- **(a) `series.volatility_tier`**, default `'monthly'`. This is OQ7(a)'s recommendation carried out, and GD1 §5's default for a series. *Rec: keep.*
	- **(b) `event_sources.volatility_tier` defaults to `'weekly'`.** The draft gave no default. GD1 §5 puts recurring-event lineups in the weekly tier, and the scanner (EV5) re-checks sources weekly. *Rec: keep.*
	- **(c) `events.origin` defaults to `'manual'`.** The draft gave no default. A row added in the admin editor is hand-entered. The ingest sets `scanned`, and EV3's form will set `user_submitted`. *Rec: keep.*
	- **(d) `events.external_key` is nullable**, not the house `NOT NULL DEFAULT ''`. With `''`, two hand-entered events credited to the same source would collide on the `(source_id, external_key)` uniqueness rule. With NULL they never do, and NULL plainly means "not from an ingest". *Rec: keep.*
	- **(e) Problem: deleting a person deletes their events.** `created_by_person_id` and `reviewed_by_person_id` link to `people` as specced, and honuware deletes linked rows along with the parent on every link (`ON DELETE CASCADE`). So removing an account deletes every event that person entered or reviewed, approved ones included. An account can be removed through the admin editor's delete on a `people` row, or through honuware's `PersonHelper::DeletePerson`. The close-don't-delete rule covers events, venues and series, but not people. Options:
		- **(1) Keep the links** and never delete a curator's account; disable it instead. No code, but it relies on discipline.
		- **(2) Drop the two links**, storing plain nullable person ids. Nothing cascades. The admin editor shows a number instead of a name, and a deleted person leaves an id that points nowhere.
		- **(3) Teach honuware an `ON DELETE SET NULL` link for record-keeping columns.** This is the right fix long term, but it's a cross-repo change with its own re-pin.

		*Rec: (2) now.* It's a two-line schema change, and there is no real data yet. Record (3) as a honuware ledger item in case the names are missed.
		- Mason- Agreed. All this sounds good to me.
		- Claude- **Decided (10/27): (a)–(d) locked; (e) option (2) carried out.** `events.created_by_person_id` and `reviewed_by_person_id` are now plain nullable BIGINTs with no link to `people`. Deleting an account leaves its events alone, and the admin editor shows the person's id. `EventsCarryTheReviewWorkflowNotTheFreshnessColumns` now asserts that neither column is a foreign key. Option (3) is recorded as honuware ledger item **H11** in [[Setting up the project]]. When it lands, the two links can come back as `SET NULL`. Windows 1940/1940 and Linux 1940/1940 after the change.
9. **Slice 3 as built — five calls** *(added 10/27)*. Each is already in the code. "Agreed" locks all five, or name the ones to change.
	- **(a) What "active series" means.** The spec names `GetActiveSeries` (the series index EV2 lists) without defining "active". As built, it means every status except `closed` and `moved`. `seasonal` series (dormant off-season) and `unverified` ones (listed but not yet checked, per GD1 §5) stay in the index. A closed series still has its own page; it only drops off the list. *Rec: keep.*
	- **(b) Category and scene-tag reads return slugs.** The four assignment helpers' reads also look up each category's or tag's slug and return it on the row (`category_slug` / `scene_tag_slug`), in the vocabulary's display order. The public payload and the filters both work in slugs, so EventHelper doesn't need a second lookup. *Rec: keep.*
	- **(c) "Events in a date range" means "events that start in it".** `GetEventsInRange(from, to)` returns events whose start time is at or after `from` and before `to`. An event that started before `from` and is still going isn't included. For example, with `from` at midnight, a party that started at 10pm the night before won't show. The calendar lists events by start time, and the range rule already lets `from` reach back a day, so this matches. If EV2 later wants a "happening now" view, EventHelper can move `from` earlier; the database query doesn't change. *Rec: keep.*
	- **(d) Looking up a disabled category or tag by slug still finds it.** `GetAll` (the filter lists) skips disabled rows, but `GetBySlug` returns a row either way. So the helper layer doesn't decide what an ingest naming a retired category means; Slice 4's EventHelper does. *Rec: keep here. Slice 4 will propose that ingest ignores disabled slugs and reports them.*
	- **(e) Recording a scan doesn't count as an edit.** `TouchLastScanned` sets `last_scanned_at_us` and leaves `updated_us` alone. A weekly scan then doesn't make every source look recently edited, and `updated_us` keeps meaning "a curator changed this source". *Rec: keep.*
	- Mason- I'll got with your recommendation.
	- Claude- **Decided (10/27): all five locked as built.** No code changes were needed. For (d), Slice 4's EventHelper will propose that ingest ignores disabled slugs and reports them.
10. **Slice 4 as built — seven calls** *(added 10/27)*. Each is already in the code and tested. "Agreed" locks all seven, or name the ones to change. Background: an *ingest* is a batch of events from a feed (EV5's scanner, or any importer) arriving through one door. Each item is matched against what we already have by its source plus the feed's own id for it.
	- **(a) A field the feed leaves out means "no change", not "blank it".** A feed that never mentions a description won't erase the one a curator typed in by hand. A field the feed *does* send wins, even if it's empty. So an empty category list removes the event's own categories (it goes back to using its series' categories), and an empty series name unlinks it from its series. New events get the database's defaults for anything left out. *Rec: keep. It's the only way a hand-edit survives the weekly scan.*
	- **(b) A new event with no venue gets its source's venue.** A source like "The Cuff's own website" is tied to The Cuff, so its events happen there unless the item says otherwise. This applies only when the event is first created. On a later scan, a missing venue keeps whatever the event has. *Rec: keep.*
	- **(c) Bad names are skipped and reported, but bad ids stop the batch.** A category, scene tag or series name we don't recognise, or one a curator has turned off, is dropped from that item. The result lists it so the scanner can log it for a curator, and the rest of the event goes in. An unknown source or venue *id* fails the whole batch instead, because that's a bug in the scanner, not something a renamed category caused. Either way the check runs before anything is written, so a failed batch leaves no half-written events. *Rec: keep. Renaming or retiring a category shouldn't stop the weekly scan.*
	- **(d) A re-scan leaves archived events alone** and reports `archived_kept`, just as it reports `rejected_kept` for rejected ones. The spec only named the rejected case. An archived event is past and settled (EV4), so a feed still listing it shouldn't reopen it. *Rec: keep.*
	- **(e) Reviewing an event into the state it's already in is an error.** Approving an approved event returns `INVALID_TRANSITION` (along with its current status) rather than quietly succeeding. That way two curators approving at once can't both think they did it, and the second click doesn't overwrite the first reviewer's name and notes. Archived events can't be reviewed at all; un-archiving stays a raw admin edit, as the behavior pin says. *Rec: keep.*
	- **(f) In the public list, an event's categories and scene tags travel as one comma-separated string** (`"dance,drag"`), because the layer below the endpoints is a flat list of name/value pairs. That's knottyyoga's convention. Slice 5's endpoint will turn them into real JSON arrays before they reach the browser, as knottyyoga's calendar does for its tags. *Rec: keep, and do the array conversion in §5.1.*
	- **(g) A turned-off neighborhood disappears from listings. A turned-off category doesn't.** Events at a venue in a disabled neighborhood show no neighborhood, and the neighborhood filter won't find them. A category or scene tag that's disabled *after* events were tagged with it still shows on those events, and still filters. Disabling only stops new feed items from using it (c). The difference comes from where each list is read. *Rec: leave as is for now. Retiring a category is rare, and a curator can remove it from the affected events. If you'd rather a disabled category vanish everywhere at once, say so; it's a small change in `GetUpcomingPublic`.*
	- Mason- Agreed
	- Claude- **Decided (10/28): all seven locked as built.** No code changes were needed. (f)'s array conversion is now noted on §5.1, and it applies to §5.3's slug lists too.
11. **Slice 5 as built — six calls** *(added 10/28)*. Each is already in the code and tested. "Agreed" locks all six, or name the ones to change.
	- **(a) The upcoming list says which dates it covers.** The response is `{"items": […], "from_us": …, "to_us": …}`, not just `{"items": […]}` as the spec said. The server trims what it's asked for: no start date means "now", no end date means six weeks out, and anything over 92 days is cut to 92. So a page that asks for a whole year gets three months back. It needs the real end date to say "showing through Jan 28" or to know where the next page starts. *Rec: keep.*
	- **(b) On an event's page, "hidden" looks the same as "doesn't exist".** A pending or rejected event is "not found" (404) for everyone without `manage_events`, including someone who is signed in. They don't get "not allowed" (403), because that would tell them the event exists. A curator holding `manage_events` sees any event, plus two extra fields, `status` and `review_notes`, so the preview can say "pending". Who reviewed it and when stay out of the response. *Rec: keep.*
	- **(c) A series page is one call:** `{"series": {…}, "upcoming": […]}`. `upcoming` is the series' approved events over the default six weeks. A closed series still has its page, with an empty `upcoming` list (OQ9 (a)). Each series shows its `status` and `last_verified_at_us`, so the page can say "seasonal" or "last checked in May". It doesn't show `volatility_tier`, which only tells the verifier (EV5) how often to re-check. *Rec: keep.*
	- **(d) The ingest door checks only that each field has the right kind of value:** a number where a number goes, text where text goes, and a list of text for the slug lists. A wrong kind fails the batch with a 400 that says which item (`item_index`). A *missing* title, start time and so on is still the business layer's call, with its own code (`MISSING_TITLE` …, Slice 4). That 400 also carries `item_index`, plus `error_code`. Nothing is written in either case. The permission check comes first, so someone without access learns nothing from the validation messages. *Rec: keep.*
	- **(e) A review that can't happen says why.** Approving an event that's already approved is a 400 carrying `error_code: INVALID_TRANSITION` and the event's current `status` (OQ10 (e)). The review screen can then say "already approved by someone else" and refresh. An unknown event is a 404. An `action` other than approve or reject is a 400 naming the `action` field. *Rec: keep.*
	- **(f) The photo test, and where the photo setting lives.** §5.6 said the test should upload a venue photo through the framework's upload endpoint as an admin. Instead, the test attaches the photo directly with the framework's image helper, as knottyyoga's photo test does. The upload endpoint is honuware's and already tested there. What's new here is only *who may read* the photo. The test checks that venue and event photos load for a signed-out visitor, and that a person's photo still needs a login. To let the test check the server's real setting rather than a copy, `main.cpp`'s one-line photo setting moved into `Endpoints::RegisterAppServices`, which `main.cpp` calls. *Rec: keep both.*
	- Mason- I'll go with your recommendations.
	- Claude- **Decided (10/28): all six locked as built.** No code changes were needed.
12. **Slice 6 as built — five calls** *(added 10/28)*. Each is already in the code and tested. "Agreed" locks all five, or name the ones to change. Background: the test_helper is the developer command tool (`communityfinder_test_helper`). It has a dashboard, a REPL where you type commands, and a one-shot mode (`--command=…`).
	- **(a) Every value is a named flag.** The spec wrote `create-venue <name>` and `--starts-in <hours>`. As built, every value is given as `--key=value`: `create-venue --name="The Cuff"` and `create-event --title="Bear Night" --starts_in_hours=5`. There are two reasons. The REPL only understands `--key=value`. And one-shot mode's flags can't contain hyphens, so the names use underscores to stay the same in both modes. The REPL used to cut a value off at its first space. Quotes now keep a value with spaces together. *Rec: keep.*
	- **(b) `ingest-demo` has two switches.** The spec asked for one canned batch that shows events being created, updated and sent back to review. §7.4 also asks for re-running it to change nothing. One fixed batch can't do both, so:
		- a plain `ingest-demo` always sends the same six events;
		- `ingest-demo --changed` sends them with three edits: a new description, a new title, and a start an hour later;
		- `--approve` (on either) approves whatever is still pending, so the "back to review" case has an approved event to act on.

		The start times count from next Monday, so the batch is identical every time it runs within a week. The anchor moves at Monday 00:00 UTC, which is Sunday at 4 or 5pm in Seattle. A run just after that sees every date move and reports updates. *Rec: keep.*
	- **(c) Approving through the helper records you as the reviewer.** `create-event --status=approved` and `ingest-demo --approve` go through the same approval step as the review endpoint, so the reviewer and time are stamped. That means someone must be logged in. The helper logs in as you when it starts; if it can't, these refuse rather than record a nobody. `create-event` also records you as the creator. *Rec: keep.*
	- **(d) Two small additions the spec didn't list.** `create-venue` takes an optional `--neighborhood=<slug>`, and `list-upcoming` takes the same category, scene and neighborhood filters as the endpoint. Without these, the neighborhood filter and series inheritance couldn't be checked from the helper. *Rec: keep.*
	- **(e) The commands are tested.** The test_helper is its own program, and no test has ever run any of its code. Phase 16 of the VS 2026 doc notes the same gap. So its command parser and the events commands now also build into the test suite, and each command runs against the test database (10 new tests). The REPL, dashboard and startup code stay out. Times print in UTC, not Seattle time, because the helper has no time-zone support. *Rec: keep both.*
	- Mason- I'll go with your recommendations

# Change Log

- **10/28/2026 (v0.9)** — **Slice 5 complete** (committed as `b57ce19`). **Slice 6 built (6.1)**, green on Windows (2068/2068) and the Linux Docker gate (2068/2068), with 10 new tests. The test_helper exe builds on both platforms. On Windows, `help` and `list-upcoming` ran against the dev database.
	- New: `src/test_helper/commands/events_commands.{h,cpp}` (`create-venue`, `create-event`, `ingest-demo`, `list-upcoming`), quote support in the REPL parser, one-shot flags in `main.cpp`, and the parser and commands built into the test bag.
	- Doc: an as-built note under §6.1, the exact commands under §7.4, and **OQ12**, five as-built calls awaiting "Agreed".
- **10/28/2026 (v0.8.1)** — **OQ11 decided** (Mason went with the recommendations). All six Slice 5 calls are kept as built, with no code changes. Remaining for Slice 5: the commit.

- **10/28/2026 (v0.8)** — **Slice 4 complete** (committed as `a245baf`). **Slice 5 built (5.1–5.6)**, green on Windows (2058/2058) and the Linux Docker gate (2058/2058), with 32 new tests.
	- New: the five events endpoints, `events_json` (key-value tables to JSON, keeping text as text), `register_app_services` (the anonymous photo seam), and `EventHelper`'s detail and series reads.
	- Doc: an as-built note under §5.6 and **OQ11**, six as-built calls awaiting "Agreed".
- **10/28/2026 (v0.7.1)** — **OQ10 decided** (Mason: "Agreed"). All seven Slice 4 calls are kept as built, with no code changes. §5.1 now carries OQ10 (f)'s JSON-array note. Remaining for Slice 4: the commit.
- **10/27/2026 (v0.7)** — **Slice 3 complete** (committed as `de0a219`). **Slice 4 built (4.1–4.2)**, green on Windows (2026/2026) and the Linux Docker gate (2026/2026), with 28 new tests.
	- New: `src/business_logic/events/`, with `EventHelper` (ingest, review, public upcoming list), `ClampUpcomingWindow`, and the public key-value-table conversion. Also `Series::GetSeriesByIds` in table_helpers.
	- Fix: `now_us()` turned out to read the wall clock, not the transaction's start time. CLAUDE.md, four test comments and the §3.6 note are corrected. No behavior changed.
	- Doc: an as-built note under §4.2 and **OQ10**, seven as-built calls awaiting "Agreed".
- **10/27/2026 (v0.6.1)** — **OQ9 decided** (Mason went with the recommendations). All five Slice 3 calls are kept as built, with no code changes. Remaining for Slice 3: the commit.
- **10/27/2026 (v0.6)** — **Slice 2 complete** (committed as `5858eeb`). **Slice 3 built (3.1–3.6)**, green on Windows (1998/1998) and the Linux Docker gate (1998/1998), with 58 new tests.
	- New: the `src/sql_util/table_helpers/` layer, with one class per events-domain table (eleven), the shared `bigint_array_literal` helper for batch reads, and the five `kFreshnessStatus*` constants in `db_schema/freshness.h`.
	- Doc: an as-built note under §3.6 and **OQ9**, five as-built calls awaiting "Agreed". GD2 §2 now points at the layer.
	- Remaining for Slice 3: OQ9 and the commit.
- **10/27/2026 (v0.5.2)** — **OQ8 decided** (Mason: "Agreed"). (a)–(d) are locked, and (e) option (2) is carried out: `events.created_by_person_id` and `reviewed_by_person_id` are plain nullable ids, not FKs. The change is in `db_schema/events.{h,cpp}`, and the schema test now asserts they are not FKs. Green on Windows (1940/1940) and the Linux Docker gate (1940/1940). The schema line, the "Slice 2 as built" note and §7.3's note are updated. Option (3) is now honuware ledger item **H11**. Remaining for Slice 2: the commit.
- **10/27/2026 (v0.5.1)** — The §7.3 live seed check passed for Slice 2 on a fresh `--recreate_database` of the dev database. It covered the new tables, constraints and indexes, the `manage_events` grant and table mapping, the admin, nested and photo rows, and nothing from the event pipeline in `allowed_tables`. A rolled-back insert test also confirmed the defaults and that the constraints reject bad rows. Note added under §7.3, and **7.3 checked**. Remaining for Slice 2: OQ8 and the commit.
- **10/27/2026 (v0.5)** — **Slice 1 complete** (1.7 checked: merged as `43b8437`). **Slice 2 built (2.1–2.6)**, green on Windows (1940/1940) and the Linux Docker gate (1940/1940).
	- New: `event_sources`, `series`, `events` and the four assignment tables. `manage_events` (granted to Administrator, mapped to all eleven events-domain tables), the nested-table rows and photo support for venues and events, all in `create_app_tables.cpp`. `create_database.cpp`'s empty permission stub was removed.
	- Tests: five schema tests, four seed tests and one HTTP test (signed-out reads of the event pipeline are refused). The seed tests now run the framework seed first.
	- Doc: a "Slice 2 as built" note under the schema, an as-built note under §2.6, OQ2's date-control check, and **OQ8**: four as-built calls plus the person-delete problem, (e).
	- GD2 §1.9 now points at `PopulateAppPermissions` and `PopulatePhotoSupportTables`.
	- Remaining for Slice 2: OQ8, the live seed check (§7.3) and the commit.
- **10/27/2026 (v0.4.1)** — The §7.3 live seed check passed for the Slice 1 portion, on a live `--recreate_database` of the `communityfinder` dev database: 12 categories, 11 scene tags, 25 neighborhoods, constraints and registrations present. A note was added under §7.3, and the box stays open for Slice 2's items. Remaining for Slice 1: the merge (1.7).
- **10/26/2026 (v0.4)** — **Slice 1 built (1.1–1.6)**, green on Windows (1930/1930) and the Linux docker gate (1930/1930). New: `categories` / `scene_tags` / `neighborhoods` / `venues` schema + the shared `db_schema/freshness.h`; the app's DDL list + seed in `business_logic/create_app_tables.{h,cpp}` (composed by `create_database.cpp`); tests in `make_database_info_test.cpp` (rewritten — the two empty-app-stream tests retired as planned), `create_app_tables_test.cpp`, and `crud_test.cpp` (anonymous vocabulary reads). **OQ7 added** — four as-built calls awaiting "Agreed": `venues.volatility_tier`, `latitude`/`longitude`, 25 neighborhoods (GD1 verbatim; the "24" was a miscount), and the seed's move into `communityfinder_core`. §2.6 annotated (Slice 2's permission + photo seed goes in `PopulateAppTables`); §7.3's count corrected to 25. Remaining for Slice 1: the live seed gate (§7.3) and the merge (1.7).
- **8/14/2026 (v0.3)** — **All six open questions are now closed.** OQ1 decided — **Option A**: the vocabulary lists are read through the framework's generic public table-reads via `allowed_tables` (already how the doc was specced — no body change) · OQ2's scope check closed adopted-by-default — **H10 = date-only** (SUTP's ledger row updated); times-of-day stay in EV2 §3.7's bespoke editor · OQ5's semantics check closed adopted-by-default — instance override = **replace-when-present** · OQ3/OQ4/OQ6 closed in earlier rounds. No work-item changes this round — the body already matched every outcome. The spec is settled; the outline's "EV1 doc" box (= drafted **+ approved**) awaits Mason's sign-off.
- **8/14/2026 (v0.2.1)** — Answer round 2: **OQ3 decided** — title changes are **minor** (silent auto-update on re-scan; the ingest behavior pin already classed title minor, so no body change — this locks it) · **OQ1 expanded** into an explicit Option A (generic public table-reads via `allowed_tables`, no new code) vs Option B (three dedicated list endpoints) with pros/cons — awaiting the pick · still pending from round 1: OQ2's H10 scope check (date-only vs also a date+time control) and OQ5's override-vs-merge semantics check.
- **8/14/2026 (v0.2)** — Mason's OQ answers folded in: **OQ5 decided** — series-level category/scene-tag assignments with per-axis instance override (two new junction tables; the inheritance behavior pin; effective-set filters + payloads in §4.1/§4.2/§5.3; events-domain table count 9 → 11 in §2.6) · **OQ2 decided** — bespoke `manage_events` event editor added to EV2 as §3.7, and first-class date-only generic-editor support recorded as honuware cross-repo ledger item **H10** in [[Setting up the project]] (clarification pending: does H10 also include a date+time control?); §7.4's manual loop notes the interim entry path · **OQ4 + OQ6 decided** — `/api/event_series` and GET-with-query-params stand (a POST sibling gets added only if the filter surface outgrows URLs) · **OQ1 + OQ3 rewritten in plain language** per Mason's asks — the no-new-vocabulary-endpoints approach and the title-minor-or-major ruling still await verdicts.
- **8/12/2026 (v0.1)** — Initial draft from the outline + SUTP Phase 10, grounded in the communityfinder / knottyyoga / honuware code scans.
