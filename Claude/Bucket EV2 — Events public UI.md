---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Levi Kuhn (owner) · Mason Bendixen
Last Updated: 10/28/2026
Version: 0.8
tags:
---

# Bucket EV2 — Events public UI

**Track:** EV — Events engine · **Size:** M · **Stream:** A · **Owner:** Levi
**Needs:** EV1 (endpoints + vocabularies live). **Feeds:** milestone M1 (events loop live locally), RT3 (newsletter pulls from these views), GD4a (sources entered → events visible here).

# Context

The events stream's public face plus its distribution surface. Implements [[Initial Project Implementation Outline]] Stream A bucket EV2 and **supersedes [[Setting up the project]] Phase 11**, folding in the brainstorm's real-usage views (*Tonight / This Weekend / By Scene / By Neighborhood*), the month calendar, series pages, per-event add-to-calendar, **and the distribution surface: per-category/scene ICS feeds + Event JSON-LD** (kept here so Stream A is fully self-contained; site-wide SEO stays in RT4).

**The phase gate is the manual loop** (milestone M1): enter an event via admin CRUD → approve → see it on the list, the calendar, the detail page, and in the ICS feed — no scanner needed.

This bucket is mostly Angular, **plus a thin C++ slice** (the ICS endpoints — same stream, same conventions as EV1). JSON-LD, sitemap-class SEO, and meta management have **no knottyyoga precedent** — the JSON-LD builder here is net-new; keep it small and tested.

**Read before starting:**

- [[Bucket EV1 — Events domain, server]] — the API shapes, the `_us`/UTC-instant time convention, the range-clamp rules.
- CF client as-built (SUTP Phases 3/5/7/9): `ui/src/app/app.config.ts` (providers, interceptors, initializers), `core/services/` (`CommunityAccess` trio + `SiteConfigService`), `core/mock/honuware-mock.ts` (mock seed incl. demo admin), environments (`local` = mock default, `serve:development` proxies to :18081), header user-chip pattern, vitest.
- knottyyoga UI as pattern reference (patterns, not code reuse — KY app code is not in `@honuware/ui`): `ui/src/app/pages/calendar/` (`calendar.service.ts` — fetch one wide window, re-filter client-side; month/week/day views), `pages/public/our-classes/` (the grouped-by-day list view-model: `ScheduleDayGroup { key, dayName, dateLabel, rows }`), `shared/utils/calendar-event-mapper.ts` (the two time encodings — CF uses only real-UTC-instants), `ServerAccessNetwork.ts` (query-string building, `{items}` unwrap, `normalizeXxx` statics, `withCredentials`).
- honuware ICS: `components/foundation/util/ical_generator.h` (`ICalConfig`/`ICalEvent`, multi-VEVENT `GenerateICalendar`, `FoldLine`); knottyyoga `src/endpoints/get_my_ical_feed.cpp` (serving headers) and CF's existing `src/business_logic/app_ical_config.h` (`App::AppICalConfig()` — prodId `-//CommunityFinder//Events//EN`).

# Scope

**In:** events client area (`/events/*`): upcoming list with filter bar + Tonight/This-Weekend presets, month calendar, event detail, series index + series pages · per-event add-to-calendar (single-event ICS) · per-category/scene ICS subscribe feeds (server endpoints + subscribe UI) · Event JSON-LD on detail pages · `CommunityAccess` events methods + full offline mock dataset · header "Events" nav link · **the bespoke `manage_events` event entry/edit admin page (§3.7 — decided in EV1 OQ2)** · **the `<app-upcoming-events-strip>` on the home page (§3.9 — decided in OQ4: ships with this bucket, order-independent with RT1)** · **full event-page metadata — titles, descriptions, OG/social tags (§3.10 — decided in OQ5, order-independent with RT4)** · vitest specs throughout · the C++ ICS endpoints with HTTP tests.

**Out (explicit non-goals):** public submission form + submitter-facing feedback flow (**EV3** — which also owns any review-queue polish beyond §3.7's editor) · site-wide sitemap/robots, branded OG-image design, and the prerender/static-rendering decision (**RT4** — event pages set their own full metadata per OQ5) · guide venue pages (**GD3** — event detail shows venue name/address unlinked for now, see OQ3) · any new events *domain* capability (that's EV1; this bucket adds no tables and no domain rules).

# Design pin-downs

- **Time display:** event times are real UTC instants (`_us`), always rendered in **America/Los_Angeles** regardless of viewer timezone — one `events-date.utils.ts` module owns every conversion (Intl.DateTimeFormat with a fixed `timeZone`), tested across DST boundaries. Day-grouping keys are Pacific calendar days (`yyyy-mm-dd`).
- **View definitions (deep-linkable, URL-driven):**
  - **Tonight** = `/events?when=tonight` → events starting from now through 4:00 AM Pacific tomorrow (bar nights spill past midnight).
  - **This Weekend** = `/events?when=weekend` → upcoming Friday 00:00 through Sunday 24:00 Pacific (already-inside-the-weekend = the remainder of it).
  - **By Scene / By Neighborhood / By Category** = `?scene=bear`, `?neighborhood=capitol-hill`, `?category=drag` — passed straight to `/api/events/upcoming`'s filters; combinable with `when`.
- **Fetch strategy** (knottyyoga `CalendarService` pattern): fetch one window (list: the API's default 42 days; calendar: the visible month ± leading/trailing grid days), keep it in a service, re-filter client-side on chip toggles so filters are instant; a `rawItemCount` distinguishes "nothing on" from "your filters hid everything." Month navigation refetches per month (respecting EV1's 92-day cap).
- **ICS feeds:** one public feed endpoint, filterable — subscribing to "bear events" is a distribution hack, so the subscribe link must be visible wherever a filter is active. Feed window: now−1 day → now+60 days. Stable UIDs (`event-{id}@{uidDomain}` via `App::AppICalConfig()`), real-UTC events with `timezone = America/Los_Angeles`, `floatingLocal = false`. Null `ends_at_us` → 2-hour default duration (VEVENTs want a DTEND; note the default in the description? no — just apply it).
- **JSON-LD:** schema.org `Event` on the detail page only (MVP): `name`, `startDate`/`endDate` as ISO-8601 with the Pacific offset, `location` as `Place {name, address}` (or the `location_text`), `description`, `url` (canonical event URL), `eventStatus: EventScheduled`. `cost_text` is unstructured — map only the exact string "Free" → `offers.price: 0`; otherwise omit offers (never guess prices). Injected/removed by the detail component via a small tested builder; no SSR at MVP (Google renders JS; RT4 revisits if AI-crawler coverage needs static rendering).
- **Home-page strip (decided OQ4 — order-independent with RT1):** the upcoming-events strip is a headline deliverable and goes on the home page **as soon as this bucket completes** — not deferred to the restyle. `<app-upcoming-events-strip>` is fully self-contained (own fetch, quiet-empty on failure). If EV2 lands before RT1's home-page restyle, EV2 mounts it on the current home page directly — a sanctioned, announced exception to the shared-shell etiquette; if RT1 lands first, EV2 mounts it into the slot RT1 provides. The mirrored contract lives in the outline's RT1 blurb (RT1 has no bucket doc yet; it inherits the contract at draft time).
- **Page metadata (decided OQ5 — order-independent with RT4):** event pages set the **full** metadata set — `document.title`, meta description, canonical URL, and OG/social-card tags (`og:title/description/url`, `og:image` from the event's photo) — through one small shared meta helper. If RT4 lands first, these pages adopt RT4's site-wide convention instead; if EV2 lands first, RT4 adopts/extends this helper rather than redoing it (mirrored contract in the outline's RT4 blurb). Honest limit, recorded: link-preview scrapers (Discord/iMessage/Facebook) don't execute JavaScript, so client-injected OG tags won't produce rich preview cards until a prerender/static-rendering pass exists — that infrastructure call stays with RT4; Google/Bing render JS and see everything immediately, and the JSON-LD is unaffected.

# Layered work items

*(Server slice first — it's the lower layer; then access/types; then pages; content last.)*

### 1. Server — ICS endpoints (C++, `src/endpoints/`, EV1 conventions apply: anchors, tests via `handle_full` + `ThreadPool::Shutdown()`)

*(Built 10/28. 1.1–1.4 are green on both platforms: Windows 2084/2084, and the Linux docker gate 2084/2084 against the 2058 floor. That's 16 new tests. OQ6 decided 10/28: all five calls kept. 1.1–1.4 committed 10/28 as `44462bc`. 1.5, including OQ7's refresh hint, was built 10/28. honuware went out as `7cd3522`, and CF is re-pinned to it. The pinned build is green: Windows 2092/2092, and the strict Linux gate 2092/2092 (fresh clone, no override, floor 2058). Committed 10/28 as `a1c7b2e`. **§1 is complete.**)*

- [x] 1.1 `src/business_logic/events/event_ical.{h,cpp}` — `BuildICalEventsFromPublicEvents(events, …) → std::vector<ICalEvent>` (mapping pin-downs above; venue name + address → `location`; description + event URL → `description`; `BuildEventUid(eventId, uidDomain)`). **Test:** `event_ical_test.cpp` — field mapping, 2-hour default DTEND, UID stability.
	- *As built (10/28):*
		- **The mapping** (`BuildICalEvent`, and `BuildICalEventsFromPublicEvents` for a list). It's pure, with no database:
			- SUMMARY is the title.
			- DTSTART/DTEND are real UTC instants. With no end, the entry runs 2 hours (`kICalDefaultDurationUs`). The time zone is `America/Los_Angeles` and `floatingLocal` is false.
			- LOCATION is "venue, address" when there's a venue, or just the venue name if it has no address. Otherwise it's `location_text`.
			- DESCRIPTION is the description, a blank line, then the event's own `url` and its page on the site, each on its own line (OQ6 (b)).
			- UID is `event-{id}@{uidDomain}` (`BuildEventUid`). `BuildEventPageUrl(site, id)` gives `{site}/events/{id}` and tolerates a trailing slash.
		- **The venue address had to come from EV1** (OQ6 (a)). `PublicEvent` had only the venue's name. It now carries `venueAddress`, one line built by `FormatVenueAddress(street, city, state, zip)`, for example "1533 13th Ave, Seattle, WA 98122", skipping blank parts. The events JSON gains `venue_address` as text (not number-converted).
		- **Tests:**
			- `event_ical_test` (8): every field, the 2-hour default, the location fallbacks, the description's links, UID stability (an edited event keeps its UID and a different event gets another), the page URL, list order, and a pass through honuware's generator checking the escaped LOCATION, DESCRIPTION and UTC DTSTART/DTEND lines.
			- `event_helper_test` (+1): `FormatVenueAddressTest`. The test world's The Cuff now has an address, and the existing venue test checks it.
- [x] 1.2 `src/endpoints/get_events_ics_feed.cpp` — `GET /api/events/feed.ics?category&scene&neighborhood&series_id` (all optional; `series_id` decided in OQ2 — it just passes through to EV1's upcoming query), anonymous. Reuses `EventHelper::GetUpcomingPublic` with the feed window, then `GenerateICalendar(vector, App::AppICalConfig())`. Headers per the knottyyoga feed: `Content-Type: text/calendar; charset=utf-8`, `X-PUBLISHED-TTL: PT1H`, `Cache-Control: max-age=3600` *(superseded by OQ7: the hourly hint moved into the file with §1.5, and the `X-PUBLISHED-TTL` header is gone)*. **Test:** `get_events_ics_feed_test.cpp` — 200 + `BEGIN:VCALENDAR`; approved-only; category filter includes/excludes; pending never appears.
	- *As built (10/28):*
		- **The window lives in `EventHelper::GetFeedEvents(tx, filters)`**, which sets now − 1 day → now + 60 days (`kFeedLookBackDays`, `kFeedWindowDays`) and calls `GetUpcomingPublic` (OQ6 (d)). "Now" comes from the database, and endpoints hold no SQL. Two `static_assert`s keep the feed inside the list's clamp (one-day look-back, 92-day cap), so a later change to either can't silently shorten the feed.
		- **The endpoint** reads the four filters with the same parsing as `/api/events/upcoming`. It looks up the site address (`website_address_login`) for the page links and serves the knottyyoga headers. When nothing matches, it serves a valid empty calendar.
		- **Tests:**
			- `get_events_ics_feed_test` (3): headers and the VCALENDAR wrapper; approved only (pending and rejected absent); an event 50 days out is included (past the list's 42-day default); each filter includes and excludes, and no match gives an empty calendar; with a site address each entry links to its page, and without one there's no link.
			- `event_helper_test` (+1): `FeedEventsReachBackADayAndSixtyDaysOut`. It shows the window edges (12 hours ago in, 2 days ago out, 50 days in, 61 days out), that a caller's window is ignored, and that filters apply.
- [x] 1.3 `src/endpoints/get_event_ics.cpp` — `GET /api/events/<int>/ical` (single VEVENT download; `Content-Disposition: attachment; filename="event-<id>.ics"`; 404 for non-approved to anonymous, mirroring the detail endpoint). **Test.**
	- *As built (10/28):* `GetEventDetail`, then 404 unless the event is approved, for everyone including curators (OQ6 (c)). Then one VEVENT with the same mapping and page link as the feed. `get_event_ics_test` (3): the headers including `Content-Disposition`; exactly one entry, the right one; pending, rejected and missing are 404; a `manage_events` curator also gets a 404 for a pending event.
- [x] 1.4 Shared-file one-liners: volatile anchors ×2 TUs in `web_app.cpp`; CMakeLists source additions. **Gate:** docker suite green; floor unchanged or raised.
	- *Done 10/28:* the two anchors (`GetEventICal`, `GetEventsICalFeed`), plus CMakeLists entries in `src/endpoints/` and `src/business_logic/events/`. The Linux gate builds Release, which is where a missing anchor would show. It ran 2084 tests (floor 2058, unchanged until bucket exit per EV1 §7.2), and every ICS route answered.
	- *Found while testing:* the test suite's secrets helper starts out holding CF's own default settings, so `website_address_login` is already `http://localhost:4201/` in every test. A test that needs the link absent has to clear it.
- [x] 1.5 **Calendar name (follow-up from OQ6 (e); cross-repo; not part of the §1 commit; do before §4.4).** honuware's `ICalConfig` gains an optional calendar name, and `GenerateICalendar` writes it as `X-WR-CALNAME` when it's set. The feed then passes a name that reflects its filters, for example "CommunityFinder: bear events", or "CommunityFinder events" with no filter. Settle the exact wording when it's built. Bump-point flow: Mason pushes honuware, Claude re-pins and verifies the pinned build, Mason commits. **Tests:** honuware's generator test covers the line, both present and absent. `get_events_ics_feed_test` checks the name on a filtered feed and an unfiltered one.
	- *As built (10/28). Built against the local honuware tree, then re-pinned to honuware `7cd3522` "Calendar name":*
		- **honuware** (`components/foundation/util/ical_generator.{h,cpp}`):
			- `ICalConfig` gains `calendarName`.
			- When it's set, `GenerateICalendar` writes `X-WR-CALNAME:` as escaped text, right after `METHOD:PUBLISH`, ahead of any event.
			- When it's empty, nothing is written, so existing callers are unchanged. That includes knottyyoga's two-field config.
			- Tests (+2): `EmitsCalendarNameWhenSet` checks the escaping, the line's place in the header, and that an empty calendar is named too. `OmitsCalendarNameWhenEmpty`.
			- **The refresh hint (OQ7):**
				- `ICalConfig` also gains `refreshInterval`, an RFC 5545 duration such as `PT1H`.
				- When it's set, the generator writes `REFRESH-INTERVAL;VALUE=DURATION:` (RFC 7986) and `X-PUBLISHED-TTL:` (the form Outlook reads) after the name, ahead of any event. Empty writes neither.
				- Tests (+2): `EmitsRefreshIntervalWhenSet` and `OmitsRefreshIntervalWhenEmpty`.
		- **CF:**
			- The brand is `App::kICalCalendarBrand = "CommunityFinder"`, beside the PRODID in `app_ical_config.h`.
			- `EventHelper::GetFeedFilterNames(tx, filters)` turns the filters into display names: category, scene, neighborhood, then the series' title. A value that names nothing is shown as given.
			- `Events::BuildFeedCalendarName(brand, names)` builds the name.
			- The feed copies `App::AppICalConfig()` and sets the name and the refresh interval (`kFeedRefreshInterval = "PT1H"`, matching its `Cache-Control: max-age=3600`).
			- The feed no longer sends the `X-PUBLISHED-TTL` HTTP header (OQ7). `Cache-Control` stays.
			- The one-event download names no calendar, since it goes into the person's own calendar, and asks for no refresh, since it's read once.
		- **Wording settled:** "CommunityFinder events" with no filter, "CommunityFinder: Drag events" with one, and "CommunityFinder: Drag, Capitol Hill events" with several. Each filter shows its display name as curators entered it.
		- **CF tests (+4):**
			- `event_ical_test` +2: `FeedCalendarNameListsTheFilters`, `NamedFeedCarriesTheCalendarName`.
			- `event_helper_test` +1: `FeedFilterNamesAreDisplayNamesInQueryOrder`.
			- `get_events_ics_feed_test` +1: `CalendarIsNamedForItsFilters`. `ServesApprovedEventsAsACalendar` now checks both refresh lines in the file and that there's no `X-PUBLISHED-TTL` header.
			- `get_event_ics_test` also checks that the download carries no name and no refresh lines.
		- **Gates:** Windows 2092/2092 and the CF Linux gate 2092/2092 (floor 2058), both built against the local honuware tree, plus honuware's own Linux gate at 1904/1904 (floor 1700). That's 8 new tests: honuware +4, CF +4. For this, Mason's Windows `x64-Debug` build points at the local tree (`FETCHCONTENT_SOURCE_DIR_HONUWARE`), and the re-pin clears it.
		- **Bump-point (10/28):**
			1. Done: Mason pushed honuware as `7cd3522b52e38ddbb5c07476e1a0799a879e76ef`, which holds exactly the three tested files.
			2. Done: Claude re-pinned `GIT_TAG` in `server/communityfinder_server/CMakeLists.txt` (the pin's only home) from `920f18a` to `7cd3522`. Claude also cleared the Windows override (`FETCHCONTENT_SOURCE_DIR_HONUWARE` is empty again, and `_deps/honuware-src` is at `7cd3522`). The pinned build passed: Windows 2092/2092, and the strict Linux gate 2092/2092 (no local honuware, no build volume, a fresh FetchContent clone in Release).
			3. Done: Mason committed CF as `a1c7b2e` "Calendar name".
		- **knottyyoga stays on its current pin.** Mason decided on 10/28 to re-pin it later, together with more honuware changes. When it does re-pin, its feed can switch to `refreshInterval` (OQ7).

### 2. Client access layer (`ui/src/app/core/`)

*(Built 10/28. 2.1–2.4 are green: vitest 82/82, up from 27 before §2, with 55 new specs in 4 new spec files. `ng build -c local` and `-c development` are both clean. The suite also passes with the machine clock set to UTC and to Tokyo time, so nothing depends on the viewer's time zone. A live check against the dev server confirmed the wire shapes the normalizers expect. Committed 10/28 as `0189a0b`. OQ8 decided 10/28: all six calls kept. **§2 is complete.**)*

- [x] 2.1 `core/types/events.types.ts` — `PublicEvent`, `EventSeries`, `Category`, `SceneTag`, `Neighborhood` interfaces mirroring EV1's wire shapes (ids + `_us` numbers + slugs, and `venue_address`, which §1 added per OQ6 (a)), plus `normalizeEvent(...)` statics (the `?? []` / number-coercion hygiene from `ServerAccessNetwork`).
	- *As built (10/28):*
		- **Types:**
			- `PublicEvent`, `UpcomingEvents` (items plus the served window), `UpcomingEventsQuery`, `EventSeries`, `EventSeriesDetail`, `Category`, `SceneTag` and `Neighborhood`.
			- The status, origin, freshness, coverage and region values are unions.
			- Field names are the wire's snake_case (OQ8 (a)).
			- A curator's `status` / `review_notes` are optional fields that exist only when the server sent them.
		- **Normalizers** (exported functions, not class statics):
			- `normalizeEvent`, `normalizeUpcomingEvents`, `normalizeEventSeries`, `normalizeEventSeriesList` and `normalizeEventSeriesDetail`.
			- The server's `""` for "none" (a missing end, venue, series or verification time) becomes `null`.
			- Numbers are coerced, missing text becomes `''`, and missing slug lists become `[]`.
		- **Vocabularies:** `dataResultsToRecords` turns the generic table read's `{ sortedColumnNames, dataTable }` into one record per row. `categoriesFromRows` / `sceneTagsFromRows` / `neighborhoodsFromRows` keep the enabled entries (Postgres `"t"`), ordered by `sort_order`, then `id` (OQ8 (e)).
		- **`buildEventsQueryString(query)`** writes the set fields in a fixed order, encoded, and leaves out unset ones (0 counts as unset, as on the server). §3.2's subscribe link can reuse it, since the feed takes the same filter names.
		- **The server's window pins** are exported for the calendar page: 42 / 92 / 1 days and `US_PER_DAY`.
		- **`events.types.spec.ts`** covers a full event round-trip, `""` → null, defaults, number coercion with cost text kept as text, curator fields, the list and series unwrapping, the vocabulary rows (disabled entries dropped, order, a tie on `sort_order`, coverage, region), and the query string (empty, order, encoding).
		- **Live check:** a dev-server run confirmed `"t"`/`"f"` booleans and the `dataTable`/`sortedColumnNames` keys from `get_table_rows`, `""` for a missing end or venue, `venue_address` on events, and 404s for an unknown event or series.
- [x] 2.2 `core/utils/events-date.utils.ts` — `usToDate`, `formatEventTime(us)` / `formatEventDateLine(us)` (Pacific-fixed), `pacificDayKey(us)`, `tonightRangeUs(now)`, `weekendRangeUs(now)`, `monthRangeUs(year, month)`. **Spec:** `events-date.utils.spec.ts` — Tonight's 4 AM boundary, weekend edges (each weekday + mid-weekend), DST spring/fall dates, day-key correctness for a 10 PM event UTC-vs-Pacific.
	- *As built (10/28):*
		- **The listed functions**, plus `dateToUs`, `nowUs`, `pacificDateTime(us)`, `pacificDayOfWeek(us)` and `pacificWallClockToUs(y, m, d, h, min)`. That last one is the Pacific wall clock → `_us`, which §3.7's editor form needs too. Out-of-range days and months roll over, so "tomorrow" is `day + 1`.
		- **How it works:**
			- All zone math goes through `Intl.DateTimeFormat` with a fixed `America/Los_Angeles`. Nothing reads the browser's own zone.
			- Two passes settle the UTC offset across a DST change.
			- Months are 1–12.
		- **Formats:**
			- `formatEventTime` gives "10:00 PM" (OQ8 (f)).
			- `formatEventDateLine` gives "Friday, Aug 14".
			- `pacificDayKey` gives "2026-08-14".
			- Intl's narrow no-break space before AM/PM is replaced with a plain space.
		- **Tonight** runs from now to the next 4:00 AM Pacific. At 1 AM that's 4 AM today, not tomorrow (OQ8 (c)).
		- **The weekend** is Friday 00:00 through Monday 00:00 Pacific. From Friday through Sunday it starts at now.
		- **`events-date.utils.spec.ts`** covers:
			- The 2026 DST dates (Mar 8 and Nov 1, checked against Windows' zone rules): midnight and 4 AM on both changeover days, a 7-hour tonight in spring and a 9-hour one in fall, and the fall-back weekend (Fri Oct 30 PDT to Mon Nov 2 PST).
			- Tonight at 7 PM, 1:30 AM, 3:59 AM and 4:00 AM.
			- The weekend from every weekday, inside the weekend, at Monday 00:30, and across a year end.
			- The month range across DST and from December into January.
			- The 10 PM Friday event that is already Saturday in UTC.
- [x] 2.3 `CommunityAccess` gains (interface + `CommunityHttpAccess` + `CommunityMockAccess`): `getUpcomingEvents(query)` → `/api/events/upcoming` with query-string building; `getEvent(id)`; `getEventSeriesList()`; `getEventSeries(slug)`; `getCategories()` / `getSceneTags()` / `getNeighborhoods()` — HTTP impl reads the vocabularies through the framework's public generic read (`allowed_tables`, per EV1 §1.6), returns them typed + sorted by `sort_order`. **Spec:** http impl query-string cases (spy on HttpClient), mock parity.
	- *As built (10/28):*
		- **The interface** gains the seven methods, each commented with its route. `getEvent` and `getEventSeries` fail with a 404 `HttpErrorResponse` for what isn't public, so §3's pages can show "not found" from `status`.
		- **`CommunityHttpAccess`:**
			- Every reply passes through a normalizer.
			- Requests carry `withCredentials`, so a signed-in curator's preview works.
			- The vocabularies come from `GET /api/get_table_rows/<table>`, called with `HttpClient` directly rather than through the library's `HONUWARE_CRUD_ACCESS`. That proxy runs its calls one at a time, and public reads shouldn't queue behind sign-in traffic (OQ8 (b)).
		- **`CommunityMockAccess`** serves §2.4's dataset by the server's rules:
			- approved only, with the same window clamp as `ClampUpcomingWindow` (one day back, 42 by default, 92 at most, a backwards window empty)
			- the same filters, on the effective (inherited) slugs, in start-then-id order
			- the active series by title; any series by slug
			- 404s for anything pending or unknown
			- copies (`structuredClone`), so a page can't change the data
		- **Existing specs:** `home.spec.ts` and `site-config.service.spec.ts` stub only the two methods they use, now typed `Partial<CommunityAccess>`.
		- **`community-access.http.spec.ts`** uses `HttpTestingController` to check the exact URLs: no query, every filter, venue and series, encoded slugs. It also covers `withCredentials`, the normalized replies, a 404 passing through, and all three vocabulary tables.
		- **`community-access.mock.spec.ts`** runs on a pinned clock and covers the window, clamp and cap, each filter including inheritance, copies, `getEvent` (past approved served; pending and unknown 404), the series list and pages, the vocabulary counts, and mock parity (see 2.4).
- [x] 2.4 **Mock demo dataset** (`CommunityMockAccess` + shared seed module): the GD1 vocabularies verbatim; ~4 venues across neighborhoods; 2 series; ~15 events **generated relative to `Date.now()`** (a couple tonight, a weekend cluster, a past one that must not render, one pending that must not appear) so the offline demo never goes stale. This is `ng serve` mock mode's whole events experience (and the screenshot/demo dataset the SUTP suggested-additions wanted). **Spec:** mock returns only approved + future-window events.
	- *As built (10/28):* `core/mock/events-mock-data.ts`, with `buildEventsMockData(now)`. It's pure, so specs pin the clock.
		- **Vocabularies:** GD1's 12 categories, 11 scene tags and 25 neighborhoods, copied from the server seed (`create_app_tables.cpp`) with the same slugs, names, coverage, regions and 10-step sort order.
		- **Venues:** four made-up ones in Capitol Hill, Ballard, Columbia City and Bellevue (OQ8 (d)).
		- **Series:** three.
			- Bear Happy Hour: open, weekly, bar / bear.
			- Sober Saturday Hike: seasonal, no venue, never verified.
			- A **closed** Drag Brunch Club. That's one more than planned, so §3.5's "Ended" marker has something to show.
		- **Events:** 17.
			- Two fall inside tonight's window whenever "now" is.
			- Four make up the weekend cluster.
			- The next three Bear Happy Hours have no tags of their own, so they inherit the series'.
			- Two hikes have no venue and no end time.
			- Four are spread over the coming weeks. One of them is 50 days out, past the list's 42, so it shows only on the calendar.
			- One past event and one pending event never show.
		- **The mix** covers free / paid / no cost, with and without an end, and one link-out scene each for sapphic and trans.
		- **Mock parity specs:**
			- Normalizing any mock event or series changes nothing, so the pages see identical values offline and live.
			- Every slug a mock event uses exists in the vocabularies.
			- From 6 PM on each day of a week, the list has two or more events tonight and nothing past or pending.

### 3. Events pages (`ui/src/app/pages/events/`, lazy `events.routes.ts`; standalone components; Material via the app's existing patterns)

*(Built 10/28. 3.1–3.7, 3.9 and 3.10 are green: vitest 173/173, up from 82 after §2, with 91 new tests in 15 new spec files. The suite also passes with the machine clock set to UTC and to Tokyo time. `ng build -c local`, `-c development` and `-c production` all build, and the events pages load as their own chunk (129 kB, 43 kB optimized). 3.8 waits for GD3. Nothing was checked against a live server, since none was running; that's §4.4's manual loop. OQ9 decided 10/28: all twelve calls kept, with (a)'s honuware fix done now rather than later. That fix is built against the local honuware tree, and the editor's stopgap is gone. It's green on Windows 2093/2093, the CF Linux gate 2093/2093 (floor 2058) and honuware's Linux gate 1905/1905, with one new test. vitest stays 173. Remaining: Mason pushes honuware, Claude re-pins CF and checks the pinned build, then Mason commits §3 with the re-pin.)*

- [x] 3.1 Route scaffolding — `app.routes.ts` one-liner: `{ path: 'events', loadChildren: … }`; child routes: `''` (list), `calendar`, `series`, `series/:slug`, `:id` (**`:id` last** — the knottyyoga route-ordering hazard). Header gains the "Events" nav link (one-line shared-file edit, both logged-in and logged-out states).
	- *As built (10/28):* `pages/events/events.routes.ts` is lazy-loaded from `app.routes.ts`, with `:id` last and a comment saying why. The header's **Events** link sits beside the site name, so it shows whether or not anyone is signed in. `header.spec.ts` checks it.
- [x] 3.2 `EventsListPage` (`/events`) — grouped-by-day upcoming list (day header "Friday, Aug 14" + rows: time, title, venue name, neighborhood chip, category/scene chips, photo thumb via `PhotoUrlBuilder('events', id, …)` with broken-image fallback); filter bar (category / scene / neighborhood selects from the vocab getters + **Tonight / This Weekend preset chips**); all filter state in query params (deep-linkable, back-button-friendly); "Subscribe to this view" ICS link reflecting active filters (`/api/events/feed.ics?...` + a copy-URL affordance); empty-state distinguishing no-events vs filtered-out. **Specs:** renders groups from mock; `when=tonight` filters correctly (fake clock); filter chips update the URL; subscribe link mirrors filters.
	- *As built (10/28):*
		- **Files:** `events-list.*`, plus `events-view.ts` (filters ↔ URL, the in-browser filter, day groups, tag chips) and two pieces the series page reuses: `event-row.*` (one event's row) and `subscribe-link.*`.
		- **Data:** one fetch of the API's default 42 days. Every filter then applies in the browser, so changing one is instant (OQ9 (b)).
		- **URL:** `when` (`tonight` / `weekend`), `category`, `scene`, `neighborhood`. An unknown `when` is ignored. Pressing the active preset again turns it off.
		- **Empty states:** "Nothing's on the calendar yet" when there are no events at all; "No events match these filters", with Clear filters, when the filters hid them; and a "couldn't be loaded" message.
		- **Subscribe:** "Get Drag events in your calendar", with Subscribe and Copy link (OQ9 (h)). It carries the category, scene and neighborhood, but not Tonight / This weekend, because the feed's dates are fixed (OQ6 (d)).
		- **Rows:** the time (with the end), title, venue, a neighborhood chip, category and scene chips (each a link to that filter), and a photo thumbnail that hides when there's no photo (OQ9 (f)).
		- **Page tags:** the title reads like "Drag, Capitol Hill events tonight". The canonical address keeps the category, scene and neighborhood and drops Tonight / This weekend, which mean a different day tomorrow.
		- **Specs:**
			- `events-view.spec.ts`: Tonight keeps 1 AM and drops 5 AM; the weekend; each filter and combinations; day groups; chip labels.
			- `events-list.spec.ts`, against the offline data on a pinned clock: day groups, `?when=tonight`, the pressed preset, a scene filter with its subscribe link, title and canonical address, the URL after each click, both empty states, a failed load.
- [x] 3.3 `EventsCalendarPage` (`/events/calendar`) — month grid (Pacific), day cells listing compact event chips (title, time), prev/next month nav (refetch per month), today highlighted, chip click → detail. Mobile: cells collapse to a stacked agenda list (simple CSS reflow, knottyyoga month-view as visual reference). **Specs:** correct day bucketing incl. a leading/trailing-grid-day event; month nav triggers a fetch with the right range.
	- *As built (10/28):*
		- **Files:** `events-calendar.*` and `calendar-grid.ts` (the Sunday-first month grid in Seattle time, the `?month=2026-09` parameter, and the range to fetch).
		- Each month fetches its whole grid, so the days before and after the month show their events too.
		- The page doesn't go back before the current month, and the current month has no `?month=` (OQ9 (d)).
		- On a phone the grid becomes a list of just the days that have events.
		- **Specs:**
			- `calendar-grid.spec.ts`: August 2026 is six weeks, Jul 26 to Sep 5; February 2026 is exactly four; today; the fetch range across the November clock change; the month parameter; the year wrap; bucketing by Seattle day.
			- `events-calendar.spec.ts`: August from the offline data with the right fetch range; today and outside-month days marked; an Oct 1 event on September's grid; next and previous in the URL; an old month showing this one; an empty month.
- [x] 3.4 `EventDetailPage` (`/events/:id`) — full field render (date line, time range, venue name + address or `location_text`, cost, description, source-of-truth link `url`, series link when `series_id`, category/scene chips, photo); **Add to calendar** = the `/api/events/{id}/ical` download **plus a Google-Calendar quick-add link** (templated `calendar.google.com/calendar/render?action=TEMPLATE&…` URL, no API — decided OQ1; ICS stays the canonical path); **hide Add to calendar when a curator previews an event that isn't approved** (`status` ≠ `approved`, since the download is approved-only — OQ6 (c)); loading / notFound / error states (knottyyoga detail-page flags pattern); sets `document.title`; injects/removes the JSON-LD script. **Specs:** renders from mock; 404 path shows notFound; add-to-calendar href correct; Google link encodes title/times/location correctly (incl. an event with no end time); a pending preview shows no Add to calendar.
	- *As built (10/28):*
		- `event-detail.*`, with everything listed. The time range reads "9:00 PM – 2:00 AM" for a night out and "10:00 AM – Sun, Aug 16, 6:00 PM" for a multi-day event (OQ9 (c)).
		- **Add to calendar:** the download and the Google Calendar link. An event with no end gets two hours in both.
		- **A curator's preview** of an event that isn't approved shows a banner, and no calendar links or search-engine data. Signed-in curators also get **Edit this event** (OQ9 (i)).
		- **The photo** shows, and goes into the page's sharing tags, only after the server confirms the event has one (OQ9 (f)).
		- §3.8's spot is marked with a comment.
		- **Specs:**
			- `event-detail.spec.ts`: the full page from the offline data, with its calendar links and title; a series event's link and inherited tags; the photo case; an unknown event and a non-numeric id; the curator preview.
			- `event-links.spec.ts`: the addresses, the location line, and the Google link with and without an end.
- [x] 3.5 `SeriesIndexPage` (`/events/series`) + `SeriesDetailPage` (`/events/series/:slug`) — index lists active series (title, cadence text, venue); detail = the durable page ("T4T — 2nd Saturdays at Kremwerk"): description, freshness footer per GD1 §7 (Verified {Month Year} from `last_verified_at_us`; "Ended" marker when status ≠ open), upcoming instances list reusing the list row component, subscribe link scoped to the series (`feed.ics?series_id=…` — decided OQ2). **Specs:** both pages from mock; ended-series marker renders.
	- *As built (10/28):*
		- `series-index.*` and `series-detail.*`, plus `shared/components/freshness-footer/`, which GD3's guide pages can reuse.
		- The footer shows "Verified July 2026", or "Not yet verified", and a marker: **Ended** for a closed series, **Moved**, or **Seasonal** (OQ9 (e)).
		- An ended or moved series keeps its page, with a banner, no upcoming dates and no subscribe link.
		- **Specs:** `series-index.spec.ts`; `series-detail.spec.ts` (a running series' dates, feed and footer; the ended one; the seasonal one; not found); `freshness-footer.spec.ts`.
- [x] 3.6 `core/utils/event-json-ld.ts` — `buildEventJsonLd(event, canonicalUrl)` per the pin-down. **Spec:** snapshot a full event and a minimal one (no venue, no end time, not "Free" → no offers).
	- *As built (10/28):* `buildEventJsonLd(event, canonicalUrl, imageUrl?)`, with three additions to the pin (OQ9 (g)). `event-json-ld.spec.ts` covers a full event, a minimal one, the "Free" rule, and no location.
- [x] 3.7 `ManageEventsPage` (`/admin/events`, `pages/admin/manage-events.*` — **decided in EV1 OQ2:** event entry/editing gets a bespoke page instead of leaning on the generic CRUD editor, whose date control can't express times-of-day). A `manage_events`-gated page calling `HONUWARE_CRUD_ACCESS` + the review endpoint directly (the `ManageRolesPage`/`ManageUsersPage` precedent — no new server surface): pending-queue list (`getFilteredTableRows('events', …, [{status: 'pending'}])`) with approve/reject → `POST /api/admin/review_event`; create/edit form — title/description/url/cost/location, **date+time picker composing Pacific wall-clock input to `_us`** (reuse `events-date.utils`), venue picker via `getFkOptions`, series picker, category + scene-tag checkboxes writing the junction tables (leave them empty to inherit the series' tags, per EV1's inheritance pin). One-line shared-file edits: `admin.routes.ts` route + an `AdminDashboard` tile. *Note: `/admin/*` sits behind `AdminGuard` (admin role) — fine while every curator is an admin; revisit the guard when a non-admin `manage_events` curator role becomes real.* **Specs:** form composes correct `_us` from Pacific datetimes (incl. DST edges); approve calls the review endpoint and drops the row from the queue; junction writes on save; empty tag selection leaves no event-level assignments.
	- *As built (10/28):*
		- **Files:**
			- `pages/admin/manage-events.*`.
			- `manage-events-form.ts`, the form's rules, tested on their own.
			- `core/events-admin-access/`, the review call, with an offline version, wired in `app.config.ts`.
			- The `/admin/events` route and an **Events** tile on the dashboard.
		- **Queue:** pending events by start time, each with a line like "Thu, Aug 13 · 8:00 PM · Driftwood Hall", and Approve, Reject (which asks for an optional note) and Edit (OQ9 (j)).
		- **Form:**
			- Find an event by title, start a new one, or arrive from the queue or an event page (`?edit=<id>`).
			- Times are typed in Seattle time. An end time alone that's earlier than the start means the next morning.
			- Venue and series pickers.
			- Category and scene checkboxes. Left empty on a series event, the event keeps the series' tags.
		- **Saving:** a new event is saved as pending, recording who entered it (OQ9 (j)). An edit sends only what changed. Tag changes add and delete only the rows for the boxes shown.
		- **Emptying a text field** (the description, location, website or cost) saves as a blank. Until 10/28 the server couldn't store that, and the form refused the save. honuware's update now keeps a blank for a text column that can't be `NULL`, so the form's check is gone (OQ9 (a)).
		- **Offline:** the mock's table store now holds the events tables (`core/mock/events-crud-mock.ts`), and the demo admin has `manage_events` (OQ9 (k)).
		- **Specs:**
			- `manage-events-form.spec.ts`: Seattle time to instants in summer and winter, both 2026 clock-change weekends, next-morning and month-end endings, a multi-day event, the error messages, loading a row, the insert and edit rows, an emptied field sent as a blank, the tag changes.
			- `manage-events.spec.ts`, against the offline store: the queue; approve; reject with and without a note; `?edit=` loading Seattle times and tags; a new event's row and tags; a series event with no tags of its own; an edit sending only the cost and swapping a tag; emptying the description; a missing start.
			- `events-crud-mock.spec.ts`: the offline rows, with no tag rows for events that inherit their series'.
- [ ] 3.8 **Cross-stream venue-link-up (per OQ3, mirrored in GD3 §3.4):** whichever of EV2/GD3 lands **second** wires the detail page's venue name to `/guide/venues/by-id/{venue_id}` — a one-line edit in `EventDetailPage`, called out in that landing PR's checklist so it can't be forgotten.
	- *Not done here (10/28):* GD3 hasn't landed, so GD3 lands second and wires the link (GD3 §3.4). A comment marks the spot in `event-detail.html`.
- [x] 3.9 `<app-upcoming-events-strip>` (`shared/components/upcoming-events-strip/` — **decided OQ4**): self-contained compact strip — the next ~5 upcoming events (own `CommunityAccess` fetch; quiet-empty on error or no events), title/time/venue rows, "See all events →" link to `/events`. **Mount it on the home page:** if RT1's restyle hasn't landed, edit `pages/home/` directly (an announced one-block addition — the OQ4-sanctioned exception to the shared-shell etiquette; ping John-Michael in the PR); if RT1 landed first, mount into the slot it provides (mirrored contract in the outline's RT1 blurb). **Specs:** renders rows from mock; empty and error states stay quiet; the home page hosts it.
	- *As built (10/28):*
		- The strip shows the next five events ("Fri, Aug 14 · 10:00 PM", the title, the place) and "See all events →". It renders nothing at all when there's nothing to show or the fetch fails.
		- RT1 hasn't landed, so it's mounted on the current home page, below the welcome card. That's OQ4's exception: **Mason, ping John-Michael in the PR** (OQ9 (l)).
		- **Specs:** `upcoming-events-strip.spec.ts` (five rows, empty, error) and `home.spec.ts` (the home page hosts it).
- [x] 3.10 `core/utils/page-meta.ts` (**decided OQ5**): a small meta helper over Angular's `Meta`/`Title` — `setEventPageMeta(event, canonicalUrl)` sets title, description (trimmed from the event description), canonical, `og:title` / `og:description` / `og:url`, and `og:image` (absolute scaled-photo URL when the event has a photo), plus list/calendar/series-page variants. Applied by every §3 page (replacing the bare `document.title` calls in 3.2–3.5); tags overwritten on each navigation so nothing stale lingers. Built to be adopted site-wide by RT4 (mirrored contract in the outline's RT4 blurb). **Specs:** tag set for a full event vs a photo-less event; navigating between pages overwrites stale tags.
	- *As built (10/28):*
		- `PageMeta` (`core/utils/page-meta.ts`) has `set(...)` for any page and `setEventPage(event, photo?)` for an event. Every §3 page uses it.
		- It sets the title ("Leather Night — Fri, Aug 14 · CommunityFinder"), the description, the canonical address, `og:title` / `description` / `url` / `type` / `site_name`, `og:image` with a large `twitter:card` when there's a photo, and the JSON-LD script.
		- Every navigation first clears the last page's tags, so nothing stale carries over (OQ9 (g)).
		- **Spec:** `page-meta.spec.ts` covers an event with and without a photo, a preview with no JSON-LD, one page's tags replacing another's, a navigation clearing them, a `</script>` title kept harmless, and description trimming.

### 4. Gates

- [ ] 4.1 `ng build -c local` + `ng build -c development` clean; events area lazy-chunked (initial bundle unchanged).
	- *After §3 (10/28):* all three builds succeed, and the events pages load as their own chunk. The initial bundle grew about 12 kB, for the home strip and the header link. The production build's warning that the initial bundle is over its 500 kB budget was already there before §3: 857 kB then, 870 kB now.
- [ ] 4.2 vitest green; record the new count (grows from 22). *(It was already 27 when §2 started, 82 after §2, and 173 after §3.)*
- [ ] 4.3 Docker suite green for the server slice; floor per EV1 §7.2 policy.
- [ ] 4.4 **Manual browser loop = milestone M1 (Mason + Levi):** offline first (`ng serve`, mock): browse list/tonight/weekend/calendar/series/detail with zero backend. Then dev (`ng serve -c development` against :18081): admin-enter a venue + events via §3.7's editor (or raw admin CRUD) → approve in the editor's queue → they appear on list + calendar + detail **and in the home-page strip** → devtools confirms the meta/OG tags on a detail page → download the event ICS and subscribe to a category feed in a real calendar app (Google Calendar URL-subscribe or Apple Calendar; after §1.5, so the subscription shows its name) → JSON-LD validates in Google's Rich Results test (paste the rendered detail page HTML).

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **Google-Calendar quick-add link** next to the ICS download (a templated `calendar.google.com/calendar/render?action=TEMPLATE&…` URL — no API, pure link)? *Rec: yes, it's ~20 lines and most non-technical users click that before an .ics; keep ICS as the canonical path.*
	- Mason- Sure, if it is a better user experience, let's do it.
	- Claude- Decided (8/14) — folded into §3.4: Add-to-calendar now offers both the ICS download and the Google-Calendar quick-add link (ICS stays the canonical path; the Google link is the one-click path most people actually use).
2. **Per-series ICS subscribe** requires a `series` filter on the feed endpoint (EV1's upcoming query already filters by `series_id` — the feed endpoint just passes it through). *Rec: include `series_id` on the feed endpoint in §1.2 while you're in the file.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (8/14) — folded in: `series_id` added to the feed endpoint (§1.2), and the series page's subscribe link uses it (§3.5).
3. **Venue links on event pages:** detail pages show venue name/address **unlinked** until GD3 exists (venue guide URLs live on Stream B's `venue_profiles.slug`, which Stream A payloads don't know). When GD3 lands, linking is a small coordinated follow-up (Stream B pings; one-line-ish edit here). *Rec: as pinned — don't invent a cross-stream payload dependency for a link.*
	- Mason- Please add notes here and in GD3 to that whichever is done first and second makes sure that this functionality is added.
	- Claude- Done (8/14) — pinned as **mirrored checkbox work items**, not just notes: §3.8 here and GD3 §3.4 both say whichever bucket lands *second* wires the event-detail venue name to `/guide/venues/by-id/{venue_id}` and calls it out in that landing PR's checklist so it can't slip.
4. **Should EV2 put an "upcoming events" strip on the home page?** *(Rewritten 8/14 per Mason's ask — original said "the home page is site chrome".)* The home page (`pages/home/`), header, and footer are the site's shared shell. Under the outline's Interface Contracts those files belong to **Stream C** (John-Michael's launch/brand stream, which restyles them in RT1) — and the parallel-work etiquette says one stream doesn't redesign a page another stream owns. An events strip on the home page is clearly wanted eventually; the question is whether this bucket should reach into Stream C's home-page file to add it now. *Rec: not now — after M1, EV2 exposes a self-contained `<app-upcoming-events-strip>` component, and Stream C mounts it with a one-line edit when they restyle the home page: the events stream owns the component, Stream C owns the placement.*
	- Mason- "the home page is site chrome"- This makes no sense. Please rewrite and elaborate.
	- Claude- Rewritten above (8/14) — "site chrome" was shorthand for the shared shell (home page + header + footer) that Stream C owns. The recommendation stands as rewritten: defer, then component-built-by-A, mounted-by-C. **"Agreed" locks it.**
	- Mason- Let's put it on the homepage now if this lands first or use his placeholder if his work lands first. Regardless, let's modify both documents to be aware of each other and allow either item to be done first. This is one of the biggest deliverables and I'd like it on the home page as soon as the work is completed restyling be damned.
	- Claude- Decided + folded in (8/14), overriding the defer rec: the strip is now work item **§3.9** and goes on the home page **as soon as this bucket completes**, whichever side lands first — EV2 mounts it directly on the current home page if first (recorded as a sanctioned, announced exception to the shared-shell etiquette), or into RT1's provided slot if the restyle landed first. Scope, design pins, and gate 4.4 updated. The mirror lives in **the outline's RT1 blurb** (RT1 has no bucket doc yet — it inherits the contract when John-Michael's doc gets drafted), so neither side can land without honoring it.
5. **How much SEO / link-preview metadata should event pages set?** *(Rewritten 8/14 per Mason's ask.)* Context: a page can set several kinds of metadata — the browser-tab/search-result title (`document.title`), a description meta tag, and Open Graph/social-card tags (`og:title`, `og:image`, … — what makes a pasted link in Discord/iMessage/Slack render as a rich preview card with an image). Bucket **RT4** owns the site-wide versions of all of this: sitemap, robots.txt, the OG-image infrastructure (built on the photo pipeline), and one consistent meta convention for every page on the site. The question for *this* bucket: what do event pages do before RT4 lands? The proposal: EV2 pages set **only `document.title`** (e.g. "T4T — Sat Aug 15") — a few lines per page, no infrastructure needed, and it immediately improves browser tabs, history, and plain search listings. Description and social-card tags **wait for RT4**, so the site gets one convention and one OG-image pipeline instead of an events-only one-off that RT4 would then have to rework. (Event *structured data* is separate and already in this bucket: the JSON-LD of §3.6.) *Rec: titles now; description/OG tags with RT4.*
	- Mason- You are being very sparse with these descriptions. Please rewrite this to make more sense. 
	- Claude- Rewritten above (8/14) with the terms spelled out. The actual call being asked: **event pages set browser-tab titles only for now, and the richer description/social-card tags arrive with RT4's site-wide pass.** "Agreed" locks it.
	- Mason- This metadata landing is critical for the success of the site as SEO is important. Let's land all of the metadata if this feature lands first. Like question 4, let's update both documents to be aware of each other and to make sure that this is implemented correctly no matter which order the items are completed in. I don't want to add a dependency here when I think these can be done in any order.
	- Claude- Decided + folded in (8/14), overriding the titles-only rec: new work item **§3.10** lands the full metadata set (title, description, canonical, OG tags incl. the event photo) through one shared `page-meta` helper, applied by every events page — order-independent with RT4: if RT4 lands first these pages adopt its convention, if EV2 lands first RT4 adopts/extends this helper site-wide (mirror in **the outline's RT4 blurb**, inherited by RT4's future doc). One honest limit recorded in the design pin so it doesn't surprise anyone later: social link-preview scrapers (Discord/iMessage/Facebook) don't run JavaScript, so the rich preview cards need RT4's prerender/static-rendering pass before they show up there — Google and Bing render JS and see all of it immediately, and the JSON-LD is unaffected. Neither bucket blocks on the other.
6. **§1 (the calendar endpoints) as built — five calls** *(added 10/28)*. Each is already in the code and tested. "Agreed" locks all five, or name the ones to change. Background: §1 added two addresses. `/api/events/feed.ics` is a calendar people *subscribe* to: their calendar app re-reads it every hour, so new events just appear. `/api/events/<id>/ical` is a one-event file behind an "Add to calendar" button.
	- **(a) Event responses now include the venue's address.** The calendar entry's location needed "The Cuff, 1533 13th Ave, Seattle, WA 98122", but EV1's event data only had the venue's *name*. Every event the server sends (the upcoming list, the event page, series pages) now also has `venue_address`: one line built from the venue's street, city, state and zip, skipping any that are blank. §3.4's event page and §3.6's search-engine data need it too. It's an addition only; nothing existing changed. *Rec: keep.*
	- **(b) Each calendar entry links back to its CommunityFinder page.** The spec said "description + event URL". As built, the entry's notes hold the event's description, then its own link (the venue's or organizer's page), then its page on our site: `…/events/<id>`. Someone who subscribed to "bear events" sees an entry in Google Calendar, and one tap brings them back to the full page. That return trip is most of the point of the feed. The site address comes from the `website_address_login` setting, which honuware already uses as "the site" for links in emails. It's `http://localhost:4201/` in dev. Despite its name, `website_address` is the API server (`:18081`), so it isn't the one to use. If the setting is blank, the page link is left out. One thing to remember: the production value of `website_address_login` isn't decided yet (`app_secret_values.cpp` has localhost in both branches), and it must be the public site's address before launch. *Rec: keep.*
	- **(c) Only approved events can be downloaded, even by curators.** The event page lets a `manage_events` curator preview a pending event. The calendar download doesn't; anything not approved is "not found" for everyone. A calendar file of an event that isn't public yet has no use, and the feed is public-only as well. The one effect elsewhere: when §3.4's page shows a curator a pending event, it should hide "Add to calendar". It can tell from the `status` field curators get. *Rec: keep.*
	- **(d) The feed's dates are fixed on the server, and it takes the four filters the spec names.** It always covers a day ago through 60 days out. Any dates in the request are ignored, since a subscription should keep rolling forward on its own. The filters are category, scene, neighborhood and series, matching the list page. The list page also takes `venue_id`, but the feed doesn't yet. That would be the "subscribe to this venue" link GD3's venue pages may want. *Rec: keep the four; add `venue_id` when GD3 asks for it. It's a one-line change.*
	- **(e) Subscribed calendars will be named after the web address for now.** A calendar app labels a subscription with a name the file provides (the `X-WR-CALNAME` line). honuware's calendar writer has no way to set one, so a subscription will likely show as something like "feed.ics" instead of "CommunityFinder: bear events". The proper fix is a small honuware change: a calendar-name setting the feed passes in. That means a cross-repo change and a re-pin, so it isn't done here. *Rec: log it as a honuware follow-up and do it before §4.4's manual loop, where you'll subscribe in a real calendar app and see the name. The other option is to patch the line in on CF's side now, which works but goes around honuware.*
	- Mason- This all sounds fine to me.
	- Claude- Decided (10/28): all five are kept as built, and §1 is ready to commit. Follow-ups folded in:
		- (b): the production value of `website_address_login` rides on the domain decision that's already open, [[Setting up the project]] OQ9, which feeds the `website_address` production values. Nothing new to track. Set it when the domain is settled.
		- (c): §3.4 now hides Add to calendar for a curator's pending preview, with a spec.
		- (d): a note in [[Bucket GD3 — Guide public UI]] §3.4 says the feed takes no `venue_id` yet, so a "subscribe to this venue" link there means asking for the one-line server change.
		- (e): logged as work item **§1.5** (the honuware calendar name plus a re-pin), due before §4.4. §4.4 points to it.
7. **The feed's "check back every hour" hint is in the wrong place** *(added 10/28, found while building §1.5; please answer before pushing honuware)*.
	- **What's wrong.** OQ6's background said a subscriber's calendar app re-reads the feed every hour. The feed asks for that with `X-PUBLISHED-TTL: PT1H`, copied from knottyyoga's feed, which sends it as an HTTP header. As far as I know, calendar apps only look for that hint inside the calendar file itself, not in the HTTP headers. So the header does nothing, and each app falls back to its own schedule.
	- **The correct form.** Inside the file there are two spellings:
		- `REFRESH-INTERVAL;VALUE=DURATION:PT1H` is the standard one (RFC 7986).
		- `X-PUBLISHED-TTL:PT1H` is the older one, which Outlook reads.
		- Feeds commonly write both.
	- **What it can't fix.** Google Calendar ignores both and refreshes subscriptions on its own timetable, which people report as anywhere from several hours to a day. No feed can change that.
	- **The fix.** It belongs in the same honuware change as §1.5. `ICalConfig` gains an optional refresh interval, written as both lines, and the feed passes `PT1H`. Doing it before you push honuware keeps it to one re-pin. knottyyoga's feed has the same gap. It would get the fix with a one-line change whenever it next re-pins.
	- *Rec: do it now, in the §1.5 honuware change. Also drop the HTTP header that does nothing, and keep `Cache-Control`. If you'd rather not, push honuware as is, and this becomes a later follow-up with its own re-pin.*
	- Mason- Let's do it now.
	- Claude- Decided and built (10/28), as recommended, in the §1.5 honuware change:
		- honuware `ICalConfig::refreshInterval` writes both refresh lines into the file.
		- The feed passes `PT1H` and no longer sends the `X-PUBLISHED-TTL` header. `Cache-Control: max-age=3600` stays.
		- This corrects OQ6's background line: Outlook, and probably Apple Calendar, can now actually re-read the feed hourly. Google Calendar still refreshes on its own schedule.
		- knottyyoga's feed (`get_my_ical_feed.cpp`) still sends the header. It can switch to `refreshInterval` with a one-line change when it next re-pins. That's knottyyoga's call and isn't tracked here.
8. **§2 (the client access layer) as built — six calls** *(added 10/28)*. Each one is already in the code and tested. "Agreed" locks all six, or name the ones to change. Background: §2 is the layer between the pages (§3) and the server. It covers how event data is typed, how dates are worked out in Seattle time, how the app fetches events, and the fake data `ng serve` shows offline. None of it is visible yet; §3 builds the pages on top of it.
	- **(a) Event fields keep the server's names.** In the client code an event's start time is `starts_at_us`, exactly as the server sends it, not a renamed `startsAtUs`. That's knottyyoga's convention and CF's existing one (`display_name` on the site info). A field reads the same in the code, in the browser's network tab and in the server's code, so there's nothing to translate. *Rec: keep.*
	- **(b) The category, scene and neighborhood lists are fetched directly.** honuware's UI library has a shared helper for table reads, but it runs calls one at a time, queued with sign-in and admin traffic. The events pages call the same server address directly, so loading a filter bar never waits behind a login check. Nothing on the server changed. *Rec: keep.*
	- **(c) After midnight, "Tonight" still means last night.** The spec said Tonight runs "through 4:00 AM tomorrow". Read literally, at 1 AM on Saturday that would mean through 4 AM Sunday, a whole extra day. As built, Tonight always ends at the next 4 AM. At 9 PM Friday it runs to 4 AM Saturday. At 1 AM Saturday it runs to 4 AM Saturday, three hours, which is what someone still out that night means. From 4 AM on, Tonight is the coming night. *Rec: keep.*
	- **(d) The offline demo uses made-up venues.** The fake events `ng serve` shows are at invented places: The Lantern Room, Driftwood Hall, Columbia City Social Club and Eastside Commons. The neighborhoods and street names are plausible. Using real Seattle bars would put made-up events under real businesses' names in screenshots and demos. There's also a third demo series, an ended one, so §3.5's "Ended" marker has something to show; the plan said two. *Rec: keep.*
	- **(e) Turned-off categories, scenes and neighborhoods are hidden.** A curator can turn an entry off in the admin editor (the `enabled` box) instead of deleting it. The server's public table read returns every row, so the client drops the turned-off ones and sorts the rest by their sort order. That matches what the server's own event filters already do. *Rec: keep.*
	- **(f) Times show as "10:00 PM".** The event pages will show "10:00 PM" and "9:30 AM", not "10 PM". The fuller form lines up in lists and is what the formatter gives by default. Dropping ":00" is a one-line change if you'd rather. It's worth deciding before §3 builds the list and detail pages on it. *Rec: keep "10:00 PM".*
	- Mason- I'll go with your recommendations.
	- Claude- Decided (10/28): all six are kept as built, and nothing changes. §2 is complete.
9. **§3 (the events pages) as built — twelve calls** *(added 10/28)*. Each one is already in the code and tested. "Agreed" locks all twelve, or name the ones to change. Only (a) needs follow-up work, and that work is in honuware.
	- **(a) The editor can't empty a text field yet. The fix belongs in honuware.**
		- **What happens.** When an edit is saved, honuware's shared update turns any blank field into "no value" (`NULL`). An event's description, location, website and cost must always hold text, even if it's blank, so the database refuses the save. Adding or changing text works. Only clearing a field fails.
		- **For now** the editor catches it before saving and says "Description can't be emptied yet…".
		- **The fix** is a small honuware change: the update keeps a blank as blank for columns that can't be `NULL`.
		- *Rec: make that change with the next batch of honuware work (knottyyoga's re-pin is waiting for a batch anyway), then drop the editor's check.*
	- **(b) The list filters in the browser.** It fetches the next 42 days once, then applies Tonight, This weekend and the category, scene and neighborhood filters itself. A filter change is instant and needs no new request. This is the plan's fetch-strategy pin. *Rec: keep. Send the filters to the server only if the list ever gets too big to fetch at once.*
	- **(c) How times read.**
		- "9:00 PM – 2:00 AM": an end by 6 AM the next morning reads as the same night, with no date.
		- A later end names its day: "10:00 AM – Sun, Aug 16, 6:00 PM".
		- With no end it's just "9:00 PM".
		- *Rec: keep.*
	- **(d) The calendar starts at this month.** The server only serves events from a day ago onward, so earlier months would always be empty. The previous-month button stops at the current month, and a link to an old month shows this one instead. The month is in the address (`/events/calendar?month=2026-09`), so a month can be shared. *Rec: keep.*
	- **(e) A series' footer marker depends on its status.**
		- The plan said to show "Ended" whenever a series isn't open. But a seasonal series isn't open either, and it hasn't ended.
		- As built, closed shows **Ended**, moved shows **Moved** and seasonal shows **Seasonal**. Open and unverified show no marker.
		- Only closed and moved series get the "has ended" banner and lose their subscribe link.
		- *Rec: keep.*
	- **(f) Photos.**
		- A list row asks for the event's thumbnail and hides it if there's none, with no extra request.
		- The event page asks the server first whether there's a photo. Only then does it show the photo and put it in the sharing tags, so a shared link never points at a missing image.
		- *Rec: keep.*
	- **(g) Page tags and search-engine data.**
		- The canonical address and `og:url` use the address the page is being viewed at. honuware's `website_address` setting is the API server, not the site.
		- Every navigation clears the last page's tags first.
		- The JSON-LD adds three things to the plan's list: that the event is in person, the event's photo, and the currency (USD) on the "Free" offer.
		- "Free" matches any capitalization.
		- A curator's preview of an unapproved event gets no JSON-LD.
		- *Rec: keep.*
	- **(h) Subscribing offers two buttons.** **Subscribe** opens the feed in a calendar app such as Apple Calendar or Outlook. **Copy link** is for Google Calendar, which can only subscribe from a pasted address. A one-line hint says where to paste it. *Rec: keep.*
	- **(i) Curators get a shortcut on the public pages.** Someone with `manage_events` sees **Edit this event** on an event page, which opens it in the editor. The editor adds no check of its own. `/admin` already requires the admin role, and the server checks `manage_events` on every save, as §3.7's note says. *Rec: keep.*
	- **(j) The editor's review flow.**
		- A curator's new event is saved as pending, recording who entered it. It goes public once it's approved in the queue, like every other event. That costs curators one extra click and keeps a single path to "public".
		- Reject asks for its note in the browser's own pop-up box. EV3 owns any queue polish.
		- *Rec: keep both.*
	- **(k) Offline, the editor and the public pages use separate data.** In `ng serve`'s offline mode the editor works on its own copy of the demo events. Approving one there doesn't make it appear on the public pages. Against the real server they're the same data. *Rec: keep. It only affects the offline demo.*
	- **(l) The home-page strip sits below the welcome card.** It shows the next five events and a "See all events" link, and disappears when there's nothing to show. This is OQ4's exception to editing a Stream C page, so **ping John-Michael in the PR**. *Rec: keep. RT1's restyle moves it into its own slot.*
	- Mason- I'll go with your recommendations except let's make the honuware change now.
	- Claude- Decided (10/28): (b)–(l) are kept as built. (a)'s honuware change is built now:
		- **honuware:** `DatabaseRESTHelper::UpdateTableValue` still turns a blank into `NULL`, except for a text column that can't be `NULL`. That column keeps the blank, since a blank is the only way to clear it. Number and date columns are unchanged: a blank can't be cast to them, so it still becomes `NULL`.
		- **honuware test:** new `UpdateTableValueEmptyStringKeptForNotNullText`. It clears `people.last_name` (a required text column) and `locked_until` (an optional number) in one save. The name ends up blank and the number ends up `NULL`. The existing blank-to-`NULL` test still passes.
		- **CF:** the editor's "can't be emptied yet" check is gone, and its two specs now check that an emptied field saves as a blank.
		- **Who else it affects:** only saves through the generic update (`/api/update_item`, the admin editor) that clear a required text column. Those always failed before, so nothing that worked changes. No honuware or knottyyoga test relied on the old failure. knottyyoga gets this whenever it next re-pins.
		- **Order:** the CF editor change needs the new honuware. Commit §3 together with the re-pin, so the editor never offers a save the pinned server refuses.

# Change Log

- **10/28/2026 (v0.8)** — **OQ8 and OQ9 decided.**
	- OQ8: all six §2 calls kept. §2 is complete.
	- OQ9: (b)–(l) kept. (a) was done now: honuware's update keeps a blank for a required text column, with a new honuware test, and the editor's stopgap check is removed. Windows 2093/2093, CF Linux 2093/2093, honuware Linux 1905/1905.
	- §3.7's note is updated, and §3's status line lists what's left: the honuware push, the CF re-pin, and the commit.

- **10/28/2026 (v0.7)** — **§3 built (3.1–3.7, 3.9, 3.10).**
	- vitest 173/173: 91 new tests in 15 new spec files, up from 82. It also passes with the machine set to UTC and to Tokyo time.
	- `ng build -c local`, `-c development` and `-c production` all build. The events pages load as their own chunk (129 kB, 43 kB optimized).
	- New: the five events pages, the event editor, the home strip, the freshness footer, the JSON-LD builder, the page-tag helper, the review call, and the offline table data for the editor.
	- 3.8 waits for GD3. Nothing was checked against a live server, since none was running; that's §4.4.
	- **OQ9**: twelve as-built calls awaiting "Agreed". (a) needs a honuware follow-up. OQ8 is still waiting for an answer.
	- §2's status line records `0189a0b`.

- **10/28/2026 (v0.6)** — **§2 built (2.1–2.4).**
	- vitest 82/82: 55 new specs in 4 new files, up from 27. `ng build -c local` / `-c development` are clean, and the suite also passes with the machine set to UTC and Tokyo time.
	- New: `core/types/events.types.ts` (types, normalizers, vocabulary rows, the query string), `core/utils/events-date.utils.ts` (Pacific dates and times, tonight, weekend, month), `core/mock/events-mock-data.ts` (the offline dataset), and `CommunityAccess`'s seven events methods in the interface, the HTTP access and the mock.
	- Wire shapes were confirmed against the live dev server.
	- **OQ8**: six as-built calls awaiting "Agreed".
	- §1's status line records `a1c7b2e`, and §1 is complete.

- **10/28/2026 (v0.5.2)** — **§1.5 done.**
	- honuware was pushed as `7cd3522` "Calendar name", and CF is re-pinned from `920f18a`.
	- The pinned build is green: Windows 2092/2092, and the strict Linux gate 2092/2092 (fresh clone, floor 2058).
	- The Windows build no longer points at the local tree.
	- knottyyoga's re-pin is deferred until there are more honuware changes.
	- Remaining: Mason commits CF.
- **10/28/2026 (v0.5.1)** — **OQ7 decided ("do it now") and built** into the §1.5 honuware change.
	- `ICalConfig::refreshInterval` writes `REFRESH-INTERVAL` and `X-PUBLISHED-TTL` into the file.
	- The feed passes `PT1H` and drops the HTTP header that did nothing.
	- Tests: honuware +2 and CF assertions. Windows 2092/2092, CF Linux 2092/2092, honuware Linux 1904/1904.
	- 1.5 now waits only on the bump-point.
- **10/28/2026 (v0.5)** — **§1.5 built** against the local honuware tree.
	- honuware `ICalConfig::calendarName` → `X-WR-CALNAME`.
	- CF names the feed from its filters' display names: "CommunityFinder: Drag events".
	- Tests: +6 (honuware +2, CF +4). Windows 2090/2090, CF Linux 2090/2090, honuware Linux 1902/1902.
	- Waiting on OQ7, new and found while building: the hourly-refresh hint is sent as an HTTP header that calendar apps don't read. The rec is to fix it inside the file in the same honuware change, before the push and re-pin.
	- §1's status line records `44462bc`.

- **10/28/2026 (v0.4.1)** — **OQ6 decided: all five §1 calls kept**, and §1 (1.1–1.4) is ready to commit. Folded in: new **§1.5**, the honuware calendar name (`X-WR-CALNAME`), a cross-repo follow-up due before §4.4, which now points to it; §3.4 hides Add to calendar for a pending preview (+ spec); §2.1 names `venue_address`. The production site address rides on [[Setting up the project]] OQ9. A `venue_id` feed-filter note was mirrored into GD3 §3.4.

- **10/28/2026 (v0.4)** — **§1 built (1.1–1.4)**, green on Windows (2084/2084) and the Linux Docker gate (2084/2084, floor 2058), with 16 new tests.
	- New: `business_logic/events/event_ical.{h,cpp}` (events → calendar entries), `EventHelper::GetFeedEvents` (the feed's fixed window), the endpoints `GET /api/events/feed.ics` and `GET /api/events/<int>/ical`, and their two anchors.
	- EV1's shape grew: `PublicEvent::venueAddress` / JSON `venue_address`, via `FormatVenueAddress`.
	- Doc: as-built notes under §1.1–1.4, and **OQ6**, five as-built calls awaiting "Agreed".
- **8/14/2026 (v0.3)** — **OQ4 + OQ5 decided (both overriding the defer recs); all five EV2 open questions now closed.** OQ4: the home-page `<app-upcoming-events-strip>` ships with this bucket as **§3.9**, order-independent with RT1 — mount directly on the current home page if EV2 lands first (sanctioned etiquette exception), or into RT1's slot if the restyle lands first. OQ5: **§3.10** lands full event-page metadata (title/description/canonical/OG tags via a shared `page-meta` helper superseding the bare `document.title` calls), order-independent with RT4; the no-JS-scraper prerender caveat is recorded in the design pin. Scope, design pins, and gate 4.4 updated; mirrored contracts added to the outline's RT1/RT4 blurbs (their future bucket docs inherit them).
- **8/14/2026 (v0.2)** — EV1-OQ2 ripple folded in: **§3.7 bespoke `manage_events` event editor** (scope updated; EV3 keeps the submitter-facing work). Mason's EV2 OQ answers folded: **OQ1 decided** — Google-Calendar quick-add beside the ICS download (§3.4) · **OQ2 decided** — `series_id` on the ICS feed (§1.2, §3.5) · **OQ3 done** — cross-stream venue-link-up pinned as mirrored work items (§3.8 here, GD3 §3.4; whichever bucket lands second wires the link) · **OQ4 + OQ5 rewritten in plain language** per Mason's asks (home-page strip ownership; metadata scope) — both still await a verdict.
- **8/12/2026 (v0.1)** — Initial draft.
