---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen (owner)
Last Updated: 10/29/2026
Version: 0.5
tags:
---

# Bucket GD2 — Directory data layer, server

**Track:** GD — Guide & content · **Size:** M · **Stream:** B · **Owner:** Mason
**Needs:** GD1 (locked vocabularies + listing-type field matrix + freshness convention) **and EV1 Slice 1 merged** (the shared `categories`/`scene_tags`/`neighborhoods` tables and `venues` — this bucket FKs into them; per OQ8, Mason builds EV1 first and GD2 starts once Slice 1 merges). **Feeds:** GD3 (guide UI), GD4b (registry seeding), GD8 (Freeze directory data), CM1/CM2 (flags + recommendations attach to these tables), EV5 (the verifier consumes the freshness columns generically).

# Context

The moat's skeleton: the guide/directory schema → helpers → domain logic → public read endpoints. Implements [[Initial Project Implementation Outline]] Stream B bucket GD2 and the brainstorm's Phase D.1, under two Interface Contracts:

- **Contract 2 (venues satellite):** Stream A owns `venues`; **all guide data about a venue lives in a satellite `venue_profiles` table FK'd to `venues.id`** — Stream B never edits Stream A's files. Guide pages join the two.
- **Contract 3 (freshness):** every listable table here carries `status` + `last_verified_at_us` (+ `volatility_tier`) exactly as defined in [[Bucket GD1 — Taxonomy & editorial foundations]] §5, and **every public payload surfaces them** — the "Verified {Month Year}" stamp and the closure graveyard are rendered from these fields, and EV5's verifier later consumes them generically.

House conventions are identical to EV1's (same layer stack, same `_us` time convention, same test rules) — **read [[Bucket EV1 — Events domain, server]]'s "Read before starting" list**; the model files cited there (knottyyoga `class_schedules.{h,cpp}`, `class_schedule_slots.{h,cpp}`, `class_schedule_helper.h`, `get_calendar.cpp`, `create_database.cpp` idioms) are this bucket's models too. Additional models for this bucket: knottyyoga `src/db_schema/blog_posts.h` + `src/business_logic/blog/blog_helper.h` (**the publish-gating predicate** guide_pages copies) and `src/endpoints/blog_posts.cpp` (published-vs-preview serving).

Also inherited from EV1's pin-downs: **deletes cascade** (honuware FKs are `ON DELETE CASCADE` — close, don't delete), and the **three-place rule** for every new table (`make_app_tables.cpp` + `CreateTables` + `PopulateTables`).

# Scope

**In:** `venue_profiles` (satellite) · `organizations` · `places` · `service_listings` · `guide_pages` (the CMS-lite layer) · scene-tag junctions for all four entity kinds · `guide_links` ("where this scene talks") · table helpers · `GuideHelper` (scene/neighborhood/directory assembly, graveyard, publish predicate) · public read endpoints under `/api/guide/*` · `manage_guide` permission + admin CRUD metadata · photo enablement (organizations, places, guide_pages) · test_helper commands · tests at every layer.

**Out (explicit non-goals):** all UI (**GD3**) · real registry content — the ~35 venues, ~80 orgs, health facts (**GD4b**, content-ops via admin CRUD once this lands) · editorial content itself (**Content Pool GD5–GD10**, publishes through `guide_pages`) · community flags/claims/recommendations (**CM1/CM2** — but the schema they attach to is being laid here, see OQ6) · any edit to Stream A's files (`venues`, vocab tables — FK references only) · events anything (Stream A).

# Design pin-downs

## Schema (per GD1 §4's field matrix; every table gets `created_us`; mutable tables get `updated_us`; every CHECK enum uses GD1's values verbatim)

- **`venue_profiles`** — `venue_id` (FK → `venues.id`, **`AddUniqueConstraint` → 1:1 satellite**), `slug` (unique — the guide's URL identity for a venue; `venues` deliberately has no slug), `crowd_vibe` (the editorial heart: which crowd, what vibe, when to go), `price_band` (nullable free text, `$` – `$$$`), `adult` (bool, default false — drives the 18+ gate), `closure_note` (nullable — "Closed 2021, lease lost; owners opened X" for graveyard entries), `last_verified_at_us` (nullable), `volatility_tier` (default `'quarterly'`). No `status` and no `name` — `venues.status`/`venues.name` are authoritative (Stream A's freshness applies there; the profile's own `last_verified_at_us` tracks the *guide facts*, per GD1 §4).
- **`organizations`** — `name`, `slug` (unique), `org_kind` (CHECK: `sports-league` · `chorus-performance` · `club-hobby` · `recovery` · `faith` · `professional` · `community-support` · `social-meetup`), `description`, `url`, `meeting_cadence_text` ("Mondays 6:30 PM, Cal Anderson"), `neighborhood_id` (nullable FK), `status` (GD1 enum, default `'open'`), `closure_note` (nullable), `last_verified_at_us`, `volatility_tier` (default `'annual'`).
- **`places`** — `name`, `slug` (unique), `place_kind` (CHECK: `park-beach` · `landmark` · `historical`), `description`, `address_text`, `neighborhood_id` (nullable FK), `adult` (bool), `status`, `last_verified_at_us`, `volatility_tier` (default `'quarterly'`; Denny Blaine runs `'monthly'` while litigation is live — a data value, not schema).
- **`service_listings`** — `name`, `slug` (unique), `service_kind` (CHECK: `sti-testing` · `prep` · `hiv-care` · `primary-care` · `mental-health` · `gender-affirming` · `dental` · `everyday-service`), `description`, `url`, `phone`, `address_text`, `neighborhood_id` (nullable FK), `is_health` (bool — the MHMDA rendering flag: when CM2 arrives, recommendations on `is_health` rows render as unattributed aggregates only), `status`, `last_verified_at_us`, `volatility_tier` (default `'semiannual'`).
- **`guide_pages`** — `slug` (unique), `title`, `section` (CHECK: `scenes` · `health` · `services` · `new-to-seattle` · `freeze` · `new-to` · `adult`), `body_markdown`, `adult` (bool), **`draft` (bool, default true) + `publish_at_us` (nullable)** — the knottyyoga blog publish predicate: **public ⇔ `NOT draft AND publish_at_us IS NOT NULL AND publish_at_us <= now_us()`** — plus `status` (GD1 enum; `'unverified'` = facts need a check) and `last_verified_at_us`, `volatility_tier` (default per section: `adult` → `'monthly'`, `health` → `'semiannual'`, else `'annual'`).
- **Scene-tag junctions** (unique pair each): `venue_profile_scene_tag_assignments` (FK → venue_profiles), `organization_scene_tag_assignments`, `place_scene_tag_assignments`, `guide_page_scene_tag_assignments` — all FK → `scene_tags` on the tag side.
- **`guide_links`** ("where this scene talks", GD1 §4's field set) — `label`, `url`, `platform` (CHECK: `discord` · `bluesky` · `instagram` · `facebook-group` · `meetup` · `website` · `other`), `sort_order`, and **exactly one parent** among four nullable FKs — `venue_profile_id`, `organization_id`, `place_id`, `scene_tag_id` — enforced with `AddCheckConstraint(kGuideLinksTable, "guide_links_exactly_one_parent", "num_nonnulls(venue_profile_id, organization_id, place_id, scene_tag_id) = 1")` (one table, real FK constraints, one admin surface — beats four link tables or an unconstrained polymorphic pair).

**No indexes at MVP** — guide tables hold hundreds of rows, not events-table volumes; the house rule (index per named query, added when a query needs it) says wait.

**FK order in `make_app_tables.cpp`:** after EV1's block — venue_profiles, organizations, places, service_listings, guide_pages, the four junctions, guide_links.

## Behavior pins

- **Graveyard = a query, not a table:** closed things stay in their tables with `status='closed'` + `closure_note`; the graveyard endpoint returns closed venues (joined with profiles) + closed orgs. Nothing is ever deleted to "clean up" (cascade + history both forbid it).
- **Draft safety:** `guide_pages` (and all guide tables) are **NOT added to the public `allowed_tables`** — public reads go through the bespoke shaped endpoints only, so an unpublished draft can never leak through generic `get_table_rows`. (EV1 added only the four vocabulary tables + venues to the public list; that stands.)
- **Preview:** a caller holding `manage_guide` gets unpublished pages from the page endpoint (the knottyyoga author-preview pattern); anonymous gets 404.
- **Page-slug conventions** (so GD3 and the Pool can rely on them): a scene's intro editorial = `guide_pages` row with `section='scenes'` and `slug == the scene tag's slug`; a neighborhood's rundown = `section='new-to-seattle'`, `slug == the neighborhood's slug`. Everything else is free-form.
- **Page body format (OQ5):** `body_markdown` is markdown. Raw HTML inside it is allowed and gets filtered client-side by Angular's sanitizer — layout tags and `class` survive; scripts, iframes, inline `style`, and event handlers are stripped (no embeds at MVP). Links to other guide pages are plain relative markdown links into GD3's routes (`/guide/venues/<slug>`, `/guide/orgs/<slug>`, …), keyed by slug; no custom shortcode syntax. The server stores and serves the body verbatim — no server-side rendering or sanitizing.
- **Verification act:** setting `last_verified_at_us` is an explicit human action (GD1 §5) — via the admin editor or the `verify-listing` test_helper command; routine edits don't bump it.

# Layered work items

*(Numbered; lower layers first; tests named per item. Prerequisite: EV1 Slice 1 merged.)*

### 1. db_schema (`src/db_schema/`, one pair per table — model: knottyyoga `class_schedules.{h,cpp}`)

*(Built 10/29 on `9792dc0` plus the uncommitted EV2 4.3 floor change. 1.1–1.9 are green: Windows 2107/2107 and Linux 2107/2107 (floor 2083), with 14 new tests. The test harness builds all ten tables at startup, so the DDL is valid on Postgres, including `num_nonnulls`. The live seed passed 10/29; see 6.2. OQ9's six calls were agreed, and (e)'s EV1 follow-up is done.)*

- [x] 1.1 `venue_profiles.{h,cpp}` — incl. the unique `venue_id` (satellite 1:1).
	- *As built (10/29):* the columns as pinned. `venue_id` is a required FK with its own UNIQUE; `slug` is required and unique. `crowd_vibe` defaults to blank; `price_band` and `closure_note` are empty (`NULL`) until there's something to say. `adult` defaults false; the tier defaults to `quarterly`. No `name` or `status`.
- [x] 1.2 `organizations.{h,cpp}`.
	- *As built (10/29):* as pinned. `org_kind` is required, with no default, and checked against GD1's eight kinds. The freshness columns default to `open` and `annual`.
- [x] 1.3 `places.{h,cpp}`.
	- *As built (10/29):* as pinned. `place_kind` is required and checked against GD1's three kinds; `adult` defaults false; the tier defaults to `quarterly`. A place in the news, like Denny Blaine, is set to `monthly` as data.
- [x] 1.4 `service_listings.{h,cpp}`.
	- *As built (10/29):* as pinned. `service_kind` is required and checked against GD1's eight kinds. The tier defaults to `semiannual`. `is_health` defaults to **true** (OQ9 (b)).
- [x] 1.5 `guide_pages.{h,cpp}`.
	- *As built (10/29):* as pinned. `section` is required and checked against GD1's seven sections; `slug` is unique across all sections. `draft` defaults true and `publish_at_us` is empty, so a new page is never public. The tier defaults to `annual` (OQ9 (a)).
	- One rule the plan didn't have: a page in the `adult` section must have `adult` set (`guide_pages_adult_section_check`, OQ9 (c)).
	- `manage_guide`'s name lives in `guide_pages.h`, as `manage_events` lives in `events.h`.
- [x] 1.6 The four `*_scene_tag_assignments.{h,cpp}` pairs.
	- *As built (10/29):* EV1's shape: an id, the owner, the scene tag, `created_us`, and a UNIQUE pair. Both sides are required FKs.
- [x] 1.7 `guide_links.{h,cpp}` — incl. the `num_nonnulls` CHECK.
	- *As built (10/29):* `label`, `url` and `platform` are required, with `platform` checked against GD1's seven. `sort_order` defaults to 0. The four parent FKs are optional, and `guide_links_exactly_one_parent` requires exactly one, as pinned.
- [x] 1.8 `make_app_tables.cpp`: register in the FK order above (below EV1's block — separate, clearly-commented Stream B section per the shared-file etiquette). **Test:** extend `make_database_info_test.cpp` — all ten tables present; `venue_profiles.venue_id` unique; the CHECK constraints exist.
	- *As built (10/29):* a "Stream B" block after EV1's, in the pinned order. `db_schema/CMakeLists.txt` lists the 20 new files, alphabetized.
	- **Tests** (`make_database_info_test.cpp`, 7 new): the composed table list now ends with the ten; the profile is a 1:1 satellite of venues; every guide entity's columns, freshness CHECKs, default tier and unique slug; GD1's kinds and sections verbatim, each required with no default; the flags' defaults and the 18+ section rule; pages start as drafts with no publish time; the four assignment tables; the links' exactly-one-parent CHECK and four FKs. EV1's assignment-table loop moved into a shared helper the guide test reuses.
- [x] 1.9 `create_database.cpp` app half — *since EV1 Slice 1 (10/26) this lives in `src/business_logic/create_app_tables.cpp`, not `create_database.cpp`: the three-place rule is `MakeAppTables` + `AppTableCreationOrder` + `PopulateAppTables`, and the seed test is `create_app_tables_test.cpp`; spell the freshness CHECKs with `db_schema/freshness.h` (`kFreshnessStatusSqlValues`, `kVolatilityTierSqlValues`). Since EV1 Slice 2 (10/27), `create_app_tables.cpp` also holds **`PopulateAppPermissions`** (where `manage_events` lives; `create_database.cpp`'s empty `PopulateAppRolesAndPermissions` stub is gone) and **`PopulatePhotoSupportTables`**, so `manage_guide` and the guide's photo rows go there too. EV1 maps its tables from an explicit `kEventsDomainTables` list, so GD2 adds its own list instead of looping over `AppTableCreationOrder`. Seed tests call `SeedAsRecreateDatabaseDoes` (framework seed, then app seed), because the permission grant looks up the framework's Administrator role by name.* — (three-place rule: `CreateTables` gains the ten tables in FK order; no index call needed): admin metadata — all ten in `admin_top_level_tables` (junctions + `guide_links` also in `admin_nested_tables`), `admin_column_data_info` for every editable column (labels + hints; `*_us` as date; `body_markdown` as long text via `rows`), table/column friendly names, display templates (`venue_profiles` → `{slug}`, `organizations`/`places`/`service_listings` → `{name}`, `guide_pages` → `{title}`) · `PopulateAppRolesAndPermissions` gains **`manage_guide`** — permission row, granted to `Administrator` **by name**, `admin_table_permissions` rows for the ten guide tables **plus `venues`** (guide curators verify venue facts — that's the freshness workflow; see OQ3) · `PopulatePhotoSupportTables` += `organizations`, `places`, `guide_pages` · **no public `allowed_tables` additions** (draft safety). **Tests:** extend `create_app_tables_test.cpp` — `manage_guide` exists + admin holds it; the table-permission map covers the ten + venues; photo rows present; guide tables absent from `allowed_tables`.
	- *As built (10/29):* all three places carry the ten in the same order. Admin top-level registration loops over `AppTableCreationOrder`, so the ten are registered automatically.
	- **Admin editor:**
		- Every non-id column has a label and a hint.
		- The long fields (crowd & vibe, descriptions, closure notes) get multi-line boxes, and Body gets a tall one (20 rows).
		- Times are date pickers; Created and Updated are read-only.
		- The fixed lists (kinds, sections, status, cadence) are text fields whose hints list the allowed values, as in EV1. The database CHECK rejects anything else. See OQ9 (f).
		- Display templates are as pinned, plus `{venue_profile_id} — {scene_tag_id}` (and its siblings) for the assignment tables and `{label}` for links.
		- The four assignment tables and `guide_links` are nested as pinned, plus `venue_profiles` under `venues` (OQ9 (d)).
	- **Seed changes:**
		- A data-info row can now carry extra settings: the box height, and a checkbox's starting state (OQ9 (e)). Per (e), EV1's four Enabled boxes got the same starting state.
		- Permission setup is one shared `AddAppPermission` that both `manage_events` and `manage_guide` use. EV1's grant is unchanged, and its tests still pass.
	- **`manage_guide`:** granted to Administrator by name and mapped to `kManageGuideTables`, the ten plus `venues`. `venues` now has two permission rows (OQ3).
	- **Photos and public reads:** photo rows for organizations, places, and guide pages. `allowed_tables` is unchanged.
	- **Tests** (`create_app_tables_test.cpp`, 7 new):
		- A catch-all: every column of every app table, EV1's included, has a friendly name, and every non-id column has data info, so a future column can't ship unlabeled.
		- A second catch-all: every app checkbox starts the way its column defaults. Exactly six default to checked: EV1's four Enabled, Draft, and Health Care.
		- The `manage_guide` grant and its table map.
		- The guide is off the public list.
		- Photo rows exist for the three and are absent for profiles and services.
		- The nesting.
		- The editor settings: templates, the tall body, and the date pickers.

### 2. table_helpers (`src/sql_util/table_helpers/` — EV1 §3 conventions)

*Since EV1 Slice 3 (10/27), this layer exists. Add GD2's files to `src/sql_util/table_helpers/CMakeLists.txt`, alphabetized, as one-line additions. Models to copy, all in that folder:*
- *Single-table helpers: `venues.{h,cpp}` and `series.{h,cpp}`. `GetActiveSeries` is the freshness-status filter, with statuses passed as parameters from `db_schema/freshness.h`'s new `kFreshnessStatus*` constants.*
- *Junction helpers: the four EV1 assignment helpers. Batch `GetForXs(ids)` reads bind one `= ANY($1::bigint[])` parameter via the shared `BigintArrayLiteral`, and each row carries the joined tag's `scene_tag_slug`.*
- *Tests: insert with `updated_us = 0` so the bump is observable, and make any constraint violation the test's last statement, because it aborts the transaction.*

*(Built 10/29 on top of the uncommitted §1. 2.1–2.6 are green: Windows 2160/2160 and Linux 2160/2160 (floor 2083), with 53 new tests. No schema or seed change, so no live seed re-run is needed. One shared-file line: `db_schema/freshness.h` gained the five cadence names. OQ10 has four as-built calls awaiting "Agreed".)*

- [x] 2.1 `venue_profiles.{h,cpp}` — `AddProfile`, `GetProfile(id)`, `GetProfileByVenueId`, `GetProfileBySlug`, `GetProfiles`, `UpdateProfile`. **Test** incl. the 1:1 uniqueness violation.
	- *As built (10/29):* as listed. `GetProfiles` is ordered by slug, since a profile has no name of its own. A profile has no status either, so a closed venue's profile is in the list too; that's what the graveyard reads.
	- **Tests** (6): add and read by id, venue and slug, with the defaults; misses are empty; the slug order; an update bumps `updated_us` and can clear a column; a second profile for the same venue throws; a duplicate slug throws.
- [x] 2.2 `organizations.{h,cpp}` — `AddOrganization`, `GetOrganization(id)`, `GetOrganizationBySlug`, `GetOrganizationsByKind`, `GetOrganizationsByNeighborhood`, `GetOrganizations(includeClosed)`, `UpdateOrganization`. **Test.**
	- *As built (10/29):* as listed, ordered by name. Leaving out closed rows also leaves out moved ones (OQ10 (a)). The kind and neighborhood reads return every status (OQ10 (b)).
	- **Tests** (7): add and read, with the freshness defaults; misses; the kind filter; the neighborhood filter (countywide groups are in none); the closed and moved filter both ways; the update; a duplicate slug throws.
- [x] 2.3 `places.{h,cpp}`, `service_listings.{h,cpp}` — same shape (`GetServiceListingsByKind` for services). **Tests.**
	- *As built (10/29):* both have the organizations shape: by id, slug, kind and neighborhood, the full list with or without closed rows, and the update.
	- Per OQ9 (a), a service listing created with Health Care unchecked and no cadence of its own re-checks annually. Updates never change the cadence (OQ10 (c)).
	- **Tests** (8 each): the organizations set. Places adds Denny Blaine as an 18+ place set to monthly, as data, and the quarterly default. Services adds the cadence rule: health stays semiannual, everyday goes annual, and a cadence passed in wins.
- [x] 2.4 `guide_pages.{h,cpp}` — `AddPage`, `GetPage(id)`, `GetPageBySlug` (any state), `GetPublishedPageBySlug`, `GetPublishedPagesBySection` (title/slug/verified fields only — list shape), `UpdatePage`. The publish predicate lives in the SQL here (knottyyoga `blog_posts` model). **Test:** draft / future-dated / published visibility matrix.
	- *As built (10/29):* as listed. The publish rule is spelled out in both published reads, as in knottyyoga's blog. The section list is ordered by title and has no body (OQ10 (d)).
	- Per OQ9 (a), a new page with no cadence of its own gets monthly in the adult section and semiannual in health; others keep the annual default.
	- **Tests** (7):
		- A new page is a draft with no publish time: the preview read finds it and the public read doesn't.
		- Misses are empty.
		- **The visibility matrix:** only "not a draft + publish time + time arrived" is public. A dated draft, an undated non-draft and a future-dated page each stay private, and the preview path still finds all three.
		- The section list's shape and order.
		- The section cadence rule.
		- Publishing by update: clear Draft and set the time.
		- A duplicate slug throws across sections.
- [x] 2.5 The four junction helpers — `Assign`, `Remove`, `GetForX`, `GetForXs(ids)` (batch), `GetXIdsForSceneTag`. **Tests** (one representative + the unique-pair violation).
	- *As built (10/29):* EV1's event-assignment shape for all four: `GetForVenueProfile` / `GetForVenueProfiles` / `GetVenueProfileIdsForSceneTag`, and likewise for organizations, places and guide pages. Every row carries `scene_tag_slug`. The page tags say nothing about publish state; callers filter pages through `GuidePages`.
	- **Tests:** venue profiles is the representative (5: display order, the batch, ids for a tag, remove, the duplicate pair). The other three get one round trip through all five methods plus the duplicate pair (2 each), 11 in all.
- [x] 2.6 `guide_links.{h,cpp}` — `AddLink`, `GetLinksForVenueProfile/Organization/Place/SceneTag`, `DeleteLink`. **Test** incl. the exactly-one-parent CHECK rejection.
	- *As built (10/29):* as listed, ordered by sort order and then id. There's no update method: links are edited in the admin editor.
	- **Tests** (6): each parent gets only its own links; the sort order; delete removes only that link; no parent throws; two parents throws; a platform outside GD1's list throws.
- *`CMakeLists.txt`:* the 30 new files, alphabetized, in both lists.

### 3. business_logic (`src/business_logic/guide/`)

- [ ] 3.1 `guide_helper.{h,cpp}` — `GuideHelper(DatabaseHelper)`:
  - `GetSceneOverview` (scene tags + per-scene entity counts + coverage posture), `GetSceneDetail(slug)` (open venues joined with profiles, orgs, places, the scene's links, the intro page by slug convention).
  - `GetVenuesDirectory(filters: scene, neighborhood, includeClosed)` (join venues ⋈ venue_profiles), `GetVenueProfileBySlug` (full: venue fields + profile + tags + links).
  - `GetOrganizationsDirectory(filters: kind, scene, neighborhood)`, `GetOrganizationBySlug` (+tags +links).
  - `GetPlaces`, `GetPlaceBySlug` · `GetServices(kind?)`.
  - `GetNeighborhoodsOverview` (+counts), `GetNeighborhoodDetail(slug)` (venues/orgs/places there + rundown page by slug convention).
  - `GetGraveyard` (closed venues ⋈ profiles + closed orgs, `closure_note` surfaced).
  - `GetPublishedGuidePage(slug)` / `GetGuidePageAnyState(slug)` (the preview path) + `IsPublished` free predicate · `GetPublishedPagesBySection(section)`.
  - **Every returned item carries `status`, `last_verified_at_us`, and (where set) `closure_note` + `adult`** — Contract 3's surfacing requirement, asserted in tests.
  - **Tests** (`guide_helper_test.cpp`): scene assembly (tagged entities in, untagged out; closed venues excluded from scene detail but present in graveyard); venue join includes both venue-row and profile-row fields; publish matrix; slug conventions resolve; neighborhood assembly; freshness fields present on every item.
- [ ] 3.2 `guide_key_value_table.{h,cpp}` — domain→wire conversions for each shape. **Test:** field presence (incl. freshness + adult) per shape.

### 4. endpoints (`src/endpoints/`, all anonymous GETs under `/api/guide/*` — EV1 §5 conventions: PascalCase methods, volatile anchors, `handle_full` tests with `ThreadPool::Shutdown()`)

- [ ] 4.1 `get_guide_scenes.cpp` — `GET /api/guide/scenes`. **Test.**
- [ ] 4.2 `get_guide_scene_detail.cpp` — `GET /api/guide/scenes/<string>`. **Test:** entities + links + intro page; 404 unknown slug.
- [ ] 4.3 `get_guide_venues.cpp` (`GET /api/guide/venues?scene&neighborhood&include_closed`) + `get_guide_venue_detail.cpp` (`GET /api/guide/venues/<string>`). **Tests:** filters; closed venue serves its page (graveyard entry) with `status` surfaced; freshness fields present.
- [ ] 4.4 `get_guide_orgs.cpp` (`?kind&scene&neighborhood`) + `get_guide_org_detail.cpp`. **Tests.**
- [ ] 4.5 `get_guide_places.cpp` + `get_guide_place_detail.cpp` (`adult` flag surfaced — the client gates rendering). **Tests.**
- [ ] 4.6 `get_guide_services.cpp` — `GET /api/guide/services?kind`. **Test:** `is_health` surfaced.
- [ ] 4.7 `get_guide_neighborhoods.cpp` + `get_guide_neighborhood_detail.cpp`. **Tests.**
- [ ] 4.8 `get_guide_graveyard.cpp` — `GET /api/guide/graveyard`. **Test:** closed-only, `closure_note` present.
- [ ] 4.9 `get_guide_page.cpp` — `GET /api/guide/pages/<string>`: published → 200 for everyone; unpublished → 404 anonymous, 200 for `manage_guide` (preview). Also `GET /api/guide/pages?section=` (published list — powers GD3's section indexes). **Tests:** the visibility matrix over HTTP + `GrantPermissionToPerson(…, "manage_guide")` for preview.
- [ ] 4.10 Shared-file one-liners: volatile anchors for the TUs in `web_app.cpp` (own-stream lines, alphabetized); CMakeLists additions; `main.cpp` `PublicPhotoTables` += `organizations`, `places`, `guide_pages`.

### 5. test_helper commands (`src/test_helper/commands/guide_commands.{h,cpp}`)

- [ ] 5.1 `create-demo-guide` (a venue+profile, a closed venue+profile with closure note, two orgs of different kinds, Denny Blaine as an adult place, one published + one draft page — prints ids) · `publish-page <slug>` · `verify-listing <table> <id>` (sets `last_verified_at_us = now` — the verification act) · `list-graveyard`.

### 6. Gates

- [ ] 6.1 Docker suite green per slice; **raise the floor** (EV1 §7.2 policy — both `build_and_test.sh` and CI once PL1 lands).
- [ ] 6.2 **Live seed gate (Mason):** `--recreate_database` exit 0; psql spot-check: `manage_guide` granted; ten tables in admin metadata; guide tables absent from `allowed_tables`; photo support rows.
	- *Passed 10/29 after §1 (Claude ran it on Mason's go-ahead; the dev database was rebuilt):*
		- `--recreate_database` exited 0.
		- The ten tables exist, all empty.
		- `manage_guide` is granted to admin and maps to 11 tables; `venues` carries both `manage_events` and `manage_guide`.
		- All ten tables have a top-level registration, a table name, a display template, and labels on every column. The six nested tables are nested.
		- No guide table is on `allowed_tables`.
		- Photo support covers organizations, places, and guide pages, plus venues, events, and the framework's people.
		- Exactly six checkboxes start checked.
		- All 16 guide CHECK constraints are in the live catalog, and `venue_id` is UNIQUE.
		- EV1's vocabularies are still 12 / 11 / 25.
	- Left open for the bucket exit: re-run it then if §2–§5 change the seed (4.10's photo-table line is in `main.cpp`, not the seed).
- [ ] 6.3 **Manual loop:** via the admin CRUD editor create a venue (Stream A's table) + its profile + scene tags + a link → `curl /api/guide/venues/<slug>` shows the joined shape with freshness fields → set the venue `status='closed'` + a closure note → it appears in `/api/guide/graveyard` → create a draft page, confirm 404, publish (draft=false + publish_at_us) → 200.

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **Page-slug conventions** (scene intro = tag slug under `section='scenes'`; neighborhood rundown = neighborhood slug under `'new-to-seattle'`). *Rec: as pinned — zero extra columns, and the Pool can pre-write pages that snap into place.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (10/26) — the slug conventions stand as pinned (Behavior pins); no body change.
2. **`guide_links` as one table with the `num_nonnulls` CHECK** vs. four per-parent link tables. *Rec: one table — real FKs, one admin surface, one helper.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (10/26) — one `guide_links` table with the `num_nonnulls` CHECK, as pinned (schema, §1.7, §2.6); no body change.
3. **`admin_table_permissions`: map `venues` → `manage_guide` too**, so non-admin guide curators can edit venue facts (address, status, verification) through the generic editor. Both `manage_events` and `manage_guide` then reach `venues` — the table supports multiple grants. *Rec: yes; verifying venue facts is exactly the guide curator's job.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (10/26) — `venues` maps to both `manage_events` and `manage_guide` (§1.9); no body change.
4. **Guide tables stay out of public `allowed_tables`** (bespoke endpoints only — draft safety + shaped payloads). *Rec: as pinned.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (10/26) — guide tables stay out of public `allowed_tables` (Behavior pins, §1.9); no body change.
5. **`body_markdown` format:** markdown (rendered client-side with the knottyyoga blog's ngx-markdown pattern), not HTML. *Rec: markdown — safer default, nicer diffs, editors are us.*
	- Mason- Does the markdown engine support embedded HTML? Is there any extensibility to reference other pages on the site?
	- Claude- Answered (10/26). Both answers come from the knottyyoga wiring GD3 copies: ngx-markdown 21 over `marked`, plain `provideMarkdown()`.
		- **Embedded HTML: yes, filtered.** `marked` passes raw HTML through untouched; ngx-markdown then runs the output through Angular's `DomSanitizer` (`SecurityContext.HTML` — the default, and [[Bucket GD3 — Guide public UI]] already pins "default sanitization stays on"). **Survives:** structural and inline tags — `div`, `span`, `section`, `details`/`summary`, `figure`/`figcaption`, the `table` family, `img`, `video`/`audio`, `mark`, `small`, `sub`/`sup`, … — with `class`, `title`, `target`, and `aria-*` attributes. **Stripped:** `<script>`, `<style>`, `<iframe>` (so no embedded maps or YouTube), forms/inputs/buttons, `style="…"` and `id` attributes, `on*` handlers, `javascript:` URLs. Upshot: editors can reach for HTML where markdown has no syntax (a collapsible `<details>`, a captioned `<figure>`, a `class` hook GD3 styles); styling comes from classes, never inline styles. If we later want embeds, that's a whitelisted-embed extension in GD3, not turning the sanitizer off.
		- **Linking to other site pages: yes, with plain markdown links — no extension needed.** `[The Cuff](/guide/venues/the-cuff)` renders as an ordinary `<a href>` into GD3's routes (`/guide/venues/:slug`, `/guide/orgs/:slug`, `/guide/places/:slug`, `/guide/scenes/:slug`, `/guide/neighborhoods/:slug`, `/guide/pages/:slug`). Slugs are the stable identity (OQ6), so links survive name changes. Two caveats, both GD3's: such a link does a full page reload unless the article component hands same-origin clicks to the Angular router (a small directive), and nothing flags a link to a slug that doesn't exist.
		- **Extensibility beyond that:** `marked` has an extension API (custom tokenizer + renderer), exposed through `provideMarkdown({ markedExtensions: [...] })` — e.g. a shortcode that renders a link titled with the venue's *current* name, or a whitelisted embed. *Rec: not at MVP* — plain links cover the need with zero custom syntax and render identically everywhere. If we ever add a shortcode, avoid `[[…]]`: it collides with Obsidian wiki-links if Pool content is drafted in the vault.
		- **GD2 impact: none either way** — the server stores and serves `body_markdown` verbatim; rendering and sanitizing are GD3's. *Rec: markdown as pinned + plain relative links; on "Agreed" the same-origin-link directive gets added to GD3 §3.9.* "Agreed" locks it.
		- Mason- Agreed.
		- Claude- Decided (10/26) — markdown with sanitizer-filtered HTML and plain relative links, no shortcodes; pinned in Behavior pins ("Page body format"). The same-origin-link directive is now [[Bucket GD3 — Guide public UI]] §2.6, applied in §3.9.
6. **CM1/CM2 forward-compatibility:** nothing extra now — flags/recommendations will FK to these tables later; the only shape decision they need from us is stable ids + slugs, which we have. *Rec: confirm nothing speculative gets added (no empty `recommendation_count` columns).*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (10/26) — nothing speculative for CM1/CM2; stable ids + slugs are the whole forward contract. No body change.
7. **Org multi-neighborhood:** organizations get ONE nullable `neighborhood_id` (null = countywide/varies). Leagues that rotate venues stay null. *Rec: as pinned; a junction table for multi-neighborhood orgs is speculative until a real page needs it.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (10/26) — one nullable `neighborhood_id` on `organizations` (null = countywide/varies); no body change.
8. **Start the EV1-independent `guide_pages` slice before EV1 Slice 1 merges?** *(Added 10/26.)* As of 10/26, EV1 Slice 1 hasn't merged (EV1 §1.7 is open; communityfinder `master` has no `venues` or vocabulary tables). Everything in this bucket FKs into those tables **except `guide_pages`**, which can go end-to-end now: schema (1.5), its `make_app_tables` + `create_database` share (1.8/1.9 — incl. the `manage_guide` permission, `guide_pages` admin metadata, and its photo row), helper + publish predicate (2.4), `GuideHelper`'s page methods (3.1's publish/preview bullet), `get_guide_page.cpp` (4.9), and `publish-page` (5.1). What waits for Slice 1: `venue_profiles`, `organizations`/`places`/`service_listings` (their `neighborhood_id` FKs), all four junctions (incl. `guide_page_scene_tag_assignments`), `guide_links`, and the `venues` → `manage_guide` permission row. *Rec: yes, unless Slice 1 is landing within days — it's a self-contained vertical slice (schema → endpoint, with the publish/preview matrix) and its shared-file edits sit in the Stream B section, so merging with Slice 1 is mechanical. Don't start orgs/services early by dropping their `neighborhood_id` FK — that buys little and adds a redo.*
	- Mason- I can switch over to EV1 first.
	- Claude- Decided (10/26) — no early slice. Mason builds EV1 first; GD2 starts in plan order (§1 onward) once EV1 Slice 1 merges. The Needs line is updated; no work-item changes.
9. **§1 as built — six calls the plan didn't spell out.** *(Added 10/29.)* Each is built as recommended, so "Agreed" covers all six. Answer per letter if you want any changed.
	- **(a) Re-check cadence that depends on another field.** The plan says a guide page's cadence depends on its section: adult pages monthly, health pages semiannual, everything else annual. GD1 also puts everyday services at annual and health services at semiannual. A database default can't look at another column, so the defaults are the common case: pages default to annual and services to semiannual. The §2 helpers will set the right cadence when they create a row. The catch: rows typed into the admin editor skip the helpers, so there the editor sets it by hand, and the field's hint states the rule. Getting it wrong only means a page is re-checked less often than it should be; nothing breaks or leaks. *Rec: as built. If it slips in practice, a later fix can have EV5's verifier treat adult and health pages as monthly and semiannual whatever the column says.*
	- **(b) "Health care" starts checked on a new service listing.** The plan gave no default. Seven of the eight service kinds are health care, and the flag decides whether a person can ever be shown recommending the listing (the MHMDA rule). Leaving it checked by mistake only hides names on a dry cleaner's recommendations. Leaving it unchecked by mistake could show a person's name on an HIV clinic's. *Rec: starts checked, as built.*
	- **(c) An 18+ rule for guide pages.** A page in the `adult` section must also have its 18+ box checked, or the database refuses to save it. Without the rule, a page could be filed under adult and still render without the 18+ gate. *Rec: keep.*
	- **(d) Venue profiles show inside the venue's admin page.** The plan nested only the assignment tables and links. Nesting the profile under its venue too means opening a venue in the admin editor shows its guide profile right there. Profiles also stay reachable from the main table list. *Rec: keep.*
	- **(e) Checkboxes that start checked now look checked.** In the admin editor's "new row" form, a checkbox whose database default is true used to show unchecked. The row still saved as true if you left it alone, because an untouched box is left out of the insert. So the box showed the opposite of what would happen. The guide's two such boxes, Draft and Health Care, now start checked. EV1 has four such boxes that still show unchecked, all named Enabled: categories, scene tags, neighborhoods, and event sources. *Rec: give EV1's four the same one-line setting now, as a tiny separate change outside GD2's slice (it touches EV1's rows). It's cosmetic, so it can also wait.*
	- **(f) Fixed lists are typed, not picked.** Kinds, sections, status, and cadence are text fields whose hints list the allowed values, and the database rejects a typo. This matches EV1. The honuware admin editor can show these as dropdowns instead (knottyyoga uses this). Changing to dropdowns is purely seed rows, with no schema change. *Rec: keep text for now, and switch both buckets' lists to dropdowns in one small later slice, before GD4b's content entry starts, since that's when people start typing these by hand.*
	  - Mason- All of these sound good.
	- Claude- Decided (10/29) — all six stand as built.
		- **(e) done now:** EV1's four Enabled boxes (categories, scene tags, neighborhoods, event sources) start checked too. A new catch-all test makes every checkbox's starting state match its column's default, for any future column as well.
		- **(f) logged for later:** the dropdown switch is a do-first item in GD4's blurb in [[Initial Project Implementation Outline]] (v0.4.3), so it happens before registry entry begins.
		- (a)–(d) need no further work. §2's helpers will apply (a)'s section and service cadence rules when they create rows.
10. **§2 as built — four calls the plan didn't spell out.** *(Added 10/29.)* Each is built as recommended, so "Agreed" covers all four. Answer per letter if you want any changed.
	- **(a) "Leave out closed" also leaves out moved.** The full organization, place and service lists can drop closed rows. They drop `moved` rows too, the same way EV1's series index does, because GD1 treats a moved listing as retired and pointing at its successor. Seasonal and unverified rows stay in. *Rec: keep. The graveyard reads closed rows on purpose, so nothing disappears from the site.*
	- **(b) The kind and neighborhood lookups return every status.** "Organizations of this kind" and "places in this neighborhood" include closed ones, as EV1's venues-by-neighborhood does. §3's `GuideHelper` decides what each page shows, so the directory hides closed rows and the graveyard shows them, using the same lookups. *Rec: keep. The alternative puts the same filter in two layers.*
	- **(c) The cadence rules apply only when a row is created.** A service listing's cadence follows its Health Care box when the listing is created. Unchecking the box later doesn't change the cadence, and moving a guide page to another section doesn't either. An editor who changes one should change the other. The admin hints state the rule but don't spell out that step. *Rec: keep. Silently overwriting a cadence someone chose would be worse than leaving it.*
	- **(d) A section's page list is alphabetical.** The list GD3's section indexes use is ordered by title, not newest first like knottyyoga's blog, since the guide is a reference rather than a feed. Besides the title, slug and freshness fields the plan named, each entry also carries the 18+ flag (so GD3 can mark gated pages) and the publish time. *Rec: keep.*

# Change Log

- **10/29/2026 (v0.5)** — **§2 (table_helpers) built.** 2.1–2.6 are marked complete with as-built notes: ten helper classes (30 files) and 53 new tests. Windows and Linux are both 2160/2160 (floor 2083). `db_schema/freshness.h` gained the five cadence names. **OQ10 added:** four as-built calls awaiting "Agreed".

- **10/29/2026 (v0.4.1)** — **OQ9 decided** (all six agreed). (e)'s EV1 follow-up is done: EV1's four Enabled boxes start checked, and a catch-all test covers every checkbox (14 new tests in §1; Windows and Linux both 2107/2107). (f) went to GD4's blurb in the outline. **6.2's live seed passed** after §1: every spot-check is green (details under 6.2). 6.2 stays open for a re-run at bucket exit if the seed changes.
- **10/29/2026 (v0.4)** — **§1 (db_schema) built.** 1.1–1.9 are marked complete with as-built notes. The ten tables are in the schema, the creation order, and the seed: admin metadata, `manage_guide`, and photo rows. There are 13 new tests; Windows and Linux are both 2106/2106 (floor 2083). Not yet live-seeded (6.2). **OQ9 added:** six as-built calls awaiting "Agreed".
- **10/26/2026 (v0.3)** — **All eight open questions are now closed.** OQ5 decided — markdown with sanitizer-filtered HTML and plain relative links, no shortcodes; new "Page body format" behavior pin (server impact: none); the same-origin-link directive went to [[Bucket GD3 — Guide public UI]] §2.6/§3.9 · OQ8 decided — no early `guide_pages` slice; Mason builds EV1 first, and GD2 starts in plan order once EV1 Slice 1 merges (Needs line updated).
- **10/26/2026 (v0.2)** — Mason's OQ answers folded in: **OQ1–4, OQ6, OQ7 decided** as recommended — the body already matched every outcome, so no work-item changes · **OQ5 answered** (embedded HTML passes through Angular's sanitizer allow-list; site links are plain relative markdown links into GD3's routes; custom `marked` extensions are possible but not recommended at MVP) — awaiting "Agreed" · **OQ8 added** — start the EV1-independent `guide_pages` slice now? · Fixed a rendering break: `price_band`'s `$`–`$$$` was being parsed by Obsidian as LaTeX math; now inline code.
- **8/12/2026 (v0.1)** — Initial draft.
