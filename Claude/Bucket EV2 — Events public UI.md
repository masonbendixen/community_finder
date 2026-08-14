---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Levi Kuhn (owner) · Mason Bendixen
Last Updated: 8/12/2026
Version: 0.1
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

**In:** events client area (`/events/*`): upcoming list with filter bar + Tonight/This-Weekend presets, month calendar, event detail, series index + series pages · per-event add-to-calendar (single-event ICS) · per-category/scene ICS subscribe feeds (server endpoints + subscribe UI) · Event JSON-LD on detail pages · `CommunityAccess` events methods + full offline mock dataset · header "Events" nav link · vitest specs throughout · the C++ ICS endpoints with HTTP tests.

**Out (explicit non-goals):** public submission form + pending-queue UI polish (**EV3**) · admin review UI beyond what Phase 7's CRUD editor already gives (EV3 polishes it) · site-wide sitemap/robots/OG images (**RT4**) · home-page events module (site chrome is Stream C's; see OQ4) · guide venue pages (**GD3** — event detail shows venue name/address unlinked for now, see OQ3) · any new events *domain* capability (that's EV1; this bucket adds no tables and no domain rules).

# Design pin-downs

- **Time display:** event times are real UTC instants (`_us`), always rendered in **America/Los_Angeles** regardless of viewer timezone — one `events-date.utils.ts` module owns every conversion (Intl.DateTimeFormat with a fixed `timeZone`), tested across DST boundaries. Day-grouping keys are Pacific calendar days (`yyyy-mm-dd`).
- **View definitions (deep-linkable, URL-driven):**
  - **Tonight** = `/events?when=tonight` → events starting from now through 4:00 AM Pacific tomorrow (bar nights spill past midnight).
  - **This Weekend** = `/events?when=weekend` → upcoming Friday 00:00 through Sunday 24:00 Pacific (already-inside-the-weekend = the remainder of it).
  - **By Scene / By Neighborhood / By Category** = `?scene=bear`, `?neighborhood=capitol-hill`, `?category=drag` — passed straight to `/api/events/upcoming`'s filters; combinable with `when`.
- **Fetch strategy** (knottyyoga `CalendarService` pattern): fetch one window (list: the API's default 42 days; calendar: the visible month ± leading/trailing grid days), keep it in a service, re-filter client-side on chip toggles so filters are instant; a `rawItemCount` distinguishes "nothing on" from "your filters hid everything." Month navigation refetches per month (respecting EV1's 92-day cap).
- **ICS feeds:** one public feed endpoint, filterable — subscribing to "bear events" is a distribution hack, so the subscribe link must be visible wherever a filter is active. Feed window: now−1 day → now+60 days. Stable UIDs (`event-{id}@{uidDomain}` via `App::AppICalConfig()`), real-UTC events with `timezone = America/Los_Angeles`, `floatingLocal = false`. Null `ends_at_us` → 2-hour default duration (VEVENTs want a DTEND; note the default in the description? no — just apply it).
- **JSON-LD:** schema.org `Event` on the detail page only (MVP): `name`, `startDate`/`endDate` as ISO-8601 with the Pacific offset, `location` as `Place {name, address}` (or the `location_text`), `description`, `url` (canonical event URL), `eventStatus: EventScheduled`. `cost_text` is unstructured — map only the exact string "Free" → `offers.price: 0`; otherwise omit offers (never guess prices). Injected/removed by the detail component via a small tested builder; no SSR at MVP (Google renders JS; RT4 revisits if AI-crawler coverage needs static rendering).

# Layered work items

*(Server slice first — it's the lower layer; then access/types; then pages; content last.)*

### 1. Server — ICS endpoints (C++, `src/endpoints/`, EV1 conventions apply: anchors, tests via `handle_full` + `ThreadPool::Shutdown()`)

- [ ] 1.1 `src/business_logic/events/event_ical.{h,cpp}` — `BuildICalEventsFromPublicEvents(events, …) → std::vector<ICalEvent>` (mapping pin-downs above; venue name + address → `location`; description + event URL → `description`; `BuildEventUid(eventId, uidDomain)`). **Test:** `event_ical_test.cpp` — field mapping, 2-hour default DTEND, UID stability.
- [ ] 1.2 `src/endpoints/get_events_ics_feed.cpp` — `GET /api/events/feed.ics?category&scene&neighborhood` (all optional), anonymous. Reuses `EventHelper::GetUpcomingPublic` with the feed window, then `GenerateICalendar(vector, App::AppICalConfig())`. Headers per the knottyyoga feed: `Content-Type: text/calendar; charset=utf-8`, `X-PUBLISHED-TTL: PT1H`, `Cache-Control: max-age=3600`. **Test:** `get_events_ics_feed_test.cpp` — 200 + `BEGIN:VCALENDAR`; approved-only; category filter includes/excludes; pending never appears.
- [ ] 1.3 `src/endpoints/get_event_ics.cpp` — `GET /api/events/<int>/ical` (single VEVENT download; `Content-Disposition: attachment; filename="event-<id>.ics"`; 404 for non-approved to anonymous, mirroring the detail endpoint). **Test.**
- [ ] 1.4 Shared-file one-liners: volatile anchors ×2 TUs in `web_app.cpp`; CMakeLists source additions. **Gate:** docker suite green; floor unchanged or raised.

### 2. Client access layer (`ui/src/app/core/`)

- [ ] 2.1 `core/types/events.types.ts` — `PublicEvent`, `EventSeries`, `Category`, `SceneTag`, `Neighborhood` interfaces mirroring EV1's wire shapes (ids + `_us` numbers + slugs), plus `normalizeEvent(...)` statics (the `?? []` / number-coercion hygiene from `ServerAccessNetwork`).
- [ ] 2.2 `core/utils/events-date.utils.ts` — `usToDate`, `formatEventTime(us)` / `formatEventDateLine(us)` (Pacific-fixed), `pacificDayKey(us)`, `tonightRangeUs(now)`, `weekendRangeUs(now)`, `monthRangeUs(year, month)`. **Spec:** `events-date.utils.spec.ts` — Tonight's 4 AM boundary, weekend edges (each weekday + mid-weekend), DST spring/fall dates, day-key correctness for a 10 PM event UTC-vs-Pacific.
- [ ] 2.3 `CommunityAccess` gains (interface + `CommunityHttpAccess` + `CommunityMockAccess`): `getUpcomingEvents(query)` → `/api/events/upcoming` with query-string building; `getEvent(id)`; `getEventSeriesList()`; `getEventSeries(slug)`; `getCategories()` / `getSceneTags()` / `getNeighborhoods()` — HTTP impl reads the vocabularies through the framework's public generic read (`allowed_tables`, per EV1 §1.6), returns them typed + sorted by `sort_order`. **Spec:** http impl query-string cases (spy on HttpClient), mock parity.
- [ ] 2.4 **Mock demo dataset** (`CommunityMockAccess` + shared seed module): the GD1 vocabularies verbatim; ~4 venues across neighborhoods; 2 series; ~15 events **generated relative to `Date.now()`** (a couple tonight, a weekend cluster, a past one that must not render, one pending that must not appear) so the offline demo never goes stale. This is `ng serve` mock mode's whole events experience (and the screenshot/demo dataset the SUTP suggested-additions wanted). **Spec:** mock returns only approved + future-window events.

### 3. Events pages (`ui/src/app/pages/events/`, lazy `events.routes.ts`; standalone components; Material via the app's existing patterns)

- [ ] 3.1 Route scaffolding — `app.routes.ts` one-liner: `{ path: 'events', loadChildren: … }`; child routes: `''` (list), `calendar`, `series`, `series/:slug`, `:id` (**`:id` last** — the knottyyoga route-ordering hazard). Header gains the "Events" nav link (one-line shared-file edit, both logged-in and logged-out states).
- [ ] 3.2 `EventsListPage` (`/events`) — grouped-by-day upcoming list (day header "Friday, Aug 14" + rows: time, title, venue name, neighborhood chip, category/scene chips, photo thumb via `PhotoUrlBuilder('events', id, …)` with broken-image fallback); filter bar (category / scene / neighborhood selects from the vocab getters + **Tonight / This Weekend preset chips**); all filter state in query params (deep-linkable, back-button-friendly); "Subscribe to this view" ICS link reflecting active filters (`/api/events/feed.ics?...` + a copy-URL affordance); empty-state distinguishing no-events vs filtered-out. **Specs:** renders groups from mock; `when=tonight` filters correctly (fake clock); filter chips update the URL; subscribe link mirrors filters.
- [ ] 3.3 `EventsCalendarPage` (`/events/calendar`) — month grid (Pacific), day cells listing compact event chips (title, time), prev/next month nav (refetch per month), today highlighted, chip click → detail. Mobile: cells collapse to a stacked agenda list (simple CSS reflow, knottyyoga month-view as visual reference). **Specs:** correct day bucketing incl. a leading/trailing-grid-day event; month nav triggers a fetch with the right range.
- [ ] 3.4 `EventDetailPage` (`/events/:id`) — full field render (date line, time range, venue name + address or `location_text`, cost, description, source-of-truth link `url`, series link when `series_id`, category/scene chips, photo); **Add to calendar** button → `/api/events/{id}/ical`; loading / notFound / error states (knottyyoga detail-page flags pattern); sets `document.title`; injects/removes the JSON-LD script. **Specs:** renders from mock; 404 path shows notFound; add-to-calendar href correct.
- [ ] 3.5 `SeriesIndexPage` (`/events/series`) + `SeriesDetailPage` (`/events/series/:slug`) — index lists active series (title, cadence text, venue); detail = the durable page ("T4T — 2nd Saturdays at Kremwerk"): description, freshness footer per GD1 §7 (Verified {Month Year} from `last_verified_at_us`; "Ended" marker when status ≠ open), upcoming instances list reusing the list row component, subscribe link scoped to the series (see OQ2). **Specs:** both pages from mock; ended-series marker renders.
- [ ] 3.6 `core/utils/event-json-ld.ts` — `buildEventJsonLd(event, canonicalUrl)` per the pin-down. **Spec:** snapshot a full event and a minimal one (no venue, no end time, not "Free" → no offers).

### 4. Gates

- [ ] 4.1 `ng build -c local` + `ng build -c development` clean; events area lazy-chunked (initial bundle unchanged).
- [ ] 4.2 vitest green; record the new count (grows from 22).
- [ ] 4.3 Docker suite green for the server slice; floor per EV1 §7.2 policy.
- [ ] 4.4 **Manual browser loop = milestone M1 (Mason + Levi):** offline first (`ng serve`, mock): browse list/tonight/weekend/calendar/series/detail with zero backend. Then dev (`ng serve -c development` against :18081): admin-enter a venue + events → approve via review endpoint or CRUD status edit → they appear on list + calendar + detail → download the event ICS and subscribe to a category feed in a real calendar app (Google Calendar URL-subscribe or Apple Calendar) → JSON-LD validates in Google's Rich Results test (paste the rendered detail page HTML).

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **Google-Calendar quick-add link** next to the ICS download (a templated `calendar.google.com/calendar/render?action=TEMPLATE&…` URL — no API, pure link)? *Rec: yes, it's ~20 lines and most non-technical users click that before an .ics; keep ICS as the canonical path.*
	- Mason- Sure, if it is a better user experience, let's do it.
2. **Per-series ICS subscribe** requires a `series` filter on the feed endpoint (EV1's upcoming query already filters by `series_id` — the feed endpoint just passes it through). *Rec: include `series_id` on the feed endpoint in §1.2 while you're in the file.*
	- Mason- I'll go with your recommendation.
3. **Venue links on event pages:** detail pages show venue name/address **unlinked** until GD3 exists (venue guide URLs live on Stream B's `venue_profiles.slug`, which Stream A payloads don't know). When GD3 lands, linking is a small coordinated follow-up (Stream B pings; one-line-ish edit here). *Rec: as pinned — don't invent a cross-stream payload dependency for a link.*
	- Mason- Please add notes here and in GD3 to that whichever is done first and second 
4. **Home-page "coming up" strip:** the home page is site chrome (Stream C's file). *Rec: defer; after M1, propose a small `<app-upcoming-events-strip>` component that Stream C mounts with a one-line edit — EV2 exposes the component, C owns the placement.*
5. **`document.title`-only meta at MVP** (no `Meta` tags / OG images here — RT4 owns site-wide meta + the OG pipeline). *Rec: as pinned; titles are cheap and self-contained, everything else would collide with RT4.*
