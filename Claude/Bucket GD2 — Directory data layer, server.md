---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen (owner)
Last Updated: 8/12/2026
Version: 0.1
tags:
---

# Bucket GD2 — Directory data layer, server

**Track:** GD — Guide & content · **Size:** M · **Stream:** B · **Owner:** Mason
**Needs:** GD1 (locked vocabularies + listing-type field matrix + freshness convention) **and EV1 Slice 1 merged** (the shared `categories`/`scene_tags`/`neighborhoods` tables and `venues` — this bucket FKs into them; Levi pings when it lands). **Feeds:** GD3 (guide UI), GD4b (registry seeding), GD8 (Freeze directory data), CM1/CM2 (flags + recommendations attach to these tables), EV5 (the verifier consumes the freshness columns generically).

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

- **`venue_profiles`** — `venue_id` (FK → `venues.id`, **`AddUniqueConstraint` → 1:1 satellite**), `slug` (unique — the guide's URL identity for a venue; `venues` deliberately has no slug), `crowd_vibe` (the editorial heart: which crowd, what vibe, when to go), `price_band` (nullable free text, "$"–"$$$"), `adult` (bool, default false — drives the 18+ gate), `closure_note` (nullable — "Closed 2021, lease lost; owners opened X" for graveyard entries), `last_verified_at_us` (nullable), `volatility_tier` (default `'quarterly'`). No `status` and no `name` — `venues.status`/`venues.name` are authoritative (Stream A's freshness applies there; the profile's own `last_verified_at_us` tracks the *guide facts*, per GD1 §4).
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
- **Verification act:** setting `last_verified_at_us` is an explicit human action (GD1 §5) — via the admin editor or the `verify-listing` test_helper command; routine edits don't bump it.

# Layered work items

*(Numbered; lower layers first; tests named per item. Prerequisite: EV1 Slice 1 merged.)*

### 1. db_schema (`src/db_schema/`, one pair per table — model: knottyyoga `class_schedules.{h,cpp}`)

- [ ] 1.1 `venue_profiles.{h,cpp}` — incl. the unique `venue_id` (satellite 1:1).
- [ ] 1.2 `organizations.{h,cpp}`.
- [ ] 1.3 `places.{h,cpp}`.
- [ ] 1.4 `service_listings.{h,cpp}`.
- [ ] 1.5 `guide_pages.{h,cpp}`.
- [ ] 1.6 The four `*_scene_tag_assignments.{h,cpp}` pairs.
- [ ] 1.7 `guide_links.{h,cpp}` — incl. the `num_nonnulls` CHECK.
- [ ] 1.8 `make_app_tables.cpp`: register in the FK order above (below EV1's block — separate, clearly-commented Stream B section per the shared-file etiquette). **Test:** extend `make_database_info_test.cpp` — all ten tables present; `venue_profiles.venue_id` unique; the CHECK constraints exist.
- [ ] 1.9 `create_database.cpp` app half (three-place rule: `CreateTables` gains the ten tables in FK order; no index call needed): admin metadata — all ten in `admin_top_level_tables` (junctions + `guide_links` also in `admin_nested_tables`), `admin_column_data_info` for every editable column (labels + hints; `*_us` as date; `body_markdown` as long text via `rows`), table/column friendly names, display templates (`venue_profiles` → `{slug}`, `organizations`/`places`/`service_listings` → `{name}`, `guide_pages` → `{title}`) · `PopulateAppRolesAndPermissions` gains **`manage_guide`** — permission row, granted to `Administrator` **by name**, `admin_table_permissions` rows for the ten guide tables **plus `venues`** (guide curators verify venue facts — that's the freshness workflow; see OQ3) · `PopulatePhotoSupportTables` += `organizations`, `places`, `guide_pages` · **no public `allowed_tables` additions** (draft safety). **Tests:** extend `create_database_app_seed_test.cpp` — `manage_guide` exists + admin holds it; the table-permission map covers the ten + venues; photo rows present; guide tables absent from `allowed_tables`.

### 2. table_helpers (`src/sql_util/table_helpers/` — EV1 §3 conventions)

- [ ] 2.1 `venue_profiles.{h,cpp}` — `AddProfile`, `GetProfile(id)`, `GetProfileByVenueId`, `GetProfileBySlug`, `GetProfiles`, `UpdateProfile`. **Test** incl. the 1:1 uniqueness violation.
- [ ] 2.2 `organizations.{h,cpp}` — `AddOrganization`, `GetOrganization(id)`, `GetOrganizationBySlug`, `GetOrganizationsByKind`, `GetOrganizationsByNeighborhood`, `GetOrganizations(includeClosed)`, `UpdateOrganization`. **Test.**
- [ ] 2.3 `places.{h,cpp}`, `service_listings.{h,cpp}` — same shape (`GetServiceListingsByKind` for services). **Tests.**
- [ ] 2.4 `guide_pages.{h,cpp}` — `AddPage`, `GetPage(id)`, `GetPageBySlug` (any state), `GetPublishedPageBySlug`, `GetPublishedPagesBySection` (title/slug/verified fields only — list shape), `UpdatePage`. The publish predicate lives in the SQL here (knottyyoga `blog_posts` model). **Test:** draft / future-dated / published visibility matrix.
- [ ] 2.5 The four junction helpers — `Assign`, `Remove`, `GetForX`, `GetForXs(ids)` (batch), `GetXIdsForSceneTag`. **Tests** (one representative + the unique-pair violation).
- [ ] 2.6 `guide_links.{h,cpp}` — `AddLink`, `GetLinksForVenueProfile/Organization/Place/SceneTag`, `DeleteLink`. **Test** incl. the exactly-one-parent CHECK rejection.

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
- [ ] 6.3 **Manual loop:** via the admin CRUD editor create a venue (Stream A's table) + its profile + scene tags + a link → `curl /api/guide/venues/<slug>` shows the joined shape with freshness fields → set the venue `status='closed'` + a closure note → it appears in `/api/guide/graveyard` → create a draft page, confirm 404, publish (draft=false + publish_at_us) → 200.

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **Page-slug conventions** (scene intro = tag slug under `section='scenes'`; neighborhood rundown = neighborhood slug under `'new-to-seattle'`). *Rec: as pinned — zero extra columns, and the Pool can pre-write pages that snap into place.*
2. **`guide_links` as one table with the `num_nonnulls` CHECK** vs. four per-parent link tables. *Rec: one table — real FKs, one admin surface, one helper.*
3. **`admin_table_permissions`: map `venues` → `manage_guide` too**, so non-admin guide curators can edit venue facts (address, status, verification) through the generic editor. Both `manage_events` and `manage_guide` then reach `venues` — the table supports multiple grants. *Rec: yes; verifying venue facts is exactly the guide curator's job.*
4. **Guide tables stay out of public `allowed_tables`** (bespoke endpoints only — draft safety + shaped payloads). *Rec: as pinned.*
5. **`body_markdown` format:** markdown (rendered client-side with the knottyyoga blog's ngx-markdown pattern), not HTML. *Rec: markdown — safer default, nicer diffs, editors are us.*
6. **CM1/CM2 forward-compatibility:** nothing extra now — flags/recommendations will FK to these tables later; the only shape decision they need from us is stable ids + slugs, which we have. *Rec: confirm nothing speculative gets added (no empty `recommendation_count` columns).*
7. **Org multi-neighborhood:** organizations get ONE nullable `neighborhood_id` (null = countywide/varies). Leagues that rotate venues stay null. *Rec: as pinned; a junction table for multi-neighborhood orgs is speculative until a real page needs it.*
