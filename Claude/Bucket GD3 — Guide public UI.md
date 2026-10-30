---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen (owner)
Last Updated: 10/29/2026
Version: 0.5
tags:
---

# Bucket GD3 — Guide public UI

**Track:** GD — Guide & content · **Size:** M · **Stream:** B · **Owner:** Mason
**Needs:** GD2 (the `/api/guide/*` endpoints). **Feeds:** GD4b (seeded content becomes visible), the Content Pool (GD5–GD10 publish through these pages), GD8-flow (onramp UI mounts on this area), CM1 (flag buttons attach to these pages), milestone M2 (guide browsable locally).

# Context

The guide's public face: scene landings, venue/org profiles, neighborhood pages, freshness rendering ("Verified {Month Year}" + closed markers), the 18+ interstitial, link-out cards, and guide-article rendering. Implements [[Initial Project Implementation Outline]] Stream B bucket GD3 and the brainstorm's Phase D.2, consuming [[Bucket GD1 — Taxonomy & editorial foundations]] §7's page templates as the render contracts and [[Bucket GD2 — Directory data layer, server]]'s payloads.

**Read before starting:**

- GD1 §6–§7 (voice + templates — the templates are the spec for these components) and §5 (what the freshness stamp means).
- CF client as-built: `ui/src/app/core/community-access/` — **the 4-file access-seam pattern** (`*.ts` interface + `InjectionToken` · `*.http.ts` · `*.mock.ts` · `*.providers.ts`) this bucket's `guide-access/` copies; `app.config.ts` (provider order is load-bearing — new providers slot in beside `provideCommunityAccess`); `pages/admin/admin.routes.ts` (lazy child-routes shape, `export default routes`); `pages/home/home.spec.ts` (the spec pattern: object-literal access mock + `provideNoopAnimations`).
- knottyyoga UI patterns (reference, not reuse): `pages/public/public.routes.ts` (**route-ordering hazard: literal segments before `:slug`**, redirects for old URLs), `pages/public/class-detail/` (loading/notFound/error flags; **independent sub-fetches never take each other down**), `pages/blog/` + `provideMarkdown()` route-scoped (the markdown rendering model for guide articles).
- EV2's `shared/components/freshness-footer` — GD3 reuses it as the footer on every page (OQ8). The footer formats its own date. Where the guide shows the "Verified" stamp inline (cards, the section index), it uses its own small formatter, `core/utils/guide-date.utils.ts` (OQ5). Nothing comes from Stream A's `events-date` utilities.

# Scope

**In:** `GuideAccess` seam + full offline mock dataset · guide client area (`/guide/*`): guide home, scene landings (full + link-out postures), venue directory + venue profiles (incl. graveyard rendering), org directory (the Freeze-directory data view) + org profiles, places, services list, neighborhoods index + pages, guide-article pages with markdown rendering, section indexes · shared link-out-card + scene-chips components (the freshness footer is EV2's, reused — OQ8) · the 18+ interstitial · "upcoming events at this venue/scene" sub-fetches (via Stream A's public API, from Stream B-owned code) · header "Guide" nav link · vitest specs throughout.

**Out (explicit non-goals):** all editorial content (**Pool GD5–GD10** — this bucket ships with mock/demo content and empty-state handling) · registry data entry (**GD4b**) · the "pick your onramp" interactive flow (**GD8-flow** — it mounts at `/guide/onramp` later) · report-stale flags, claims (**CM1**) · site-wide SEO/meta/OG (**RT4** — pages set `document.title` only) · event pages (**EV2**) · admin editing UI (the Phase 7 CRUD editor already covers guide tables via GD2's metadata).

# Design pin-downs

- **Freshness rendering (GD1 §5/§7):** every entity/article page ends with EV2's shared freshness footer (`shared/components/freshness-footer`, reused per OQ8): "Verified {Month} {Year}" from `last_verified_at_us` (absent → "Not yet verified" in the same quiet style — honesty over polish), plus a "Closed" / "Moved" / "Seasonal" marker when `status` says so (series-style "Ended" wording stays Stream A's concern, through the footer's `closedLabel` input). A closed venue or org also shows a "Closed" banner at the top of its page with the `closure_note` beside it, since the footer has no slot for the note (OQ8). Closed pages render fully — the graveyard is history, not an error state.
- **18+ gate (Q3b):** a data-driven interstitial, not age verification (WA has no AV law as of mid-2026; the brainstorm's posture). Mechanics: entities/pages carry `adult` from the server; an `AdultGate` wrapper component checks `sessionStorage['cf_adult_ok']` — unset → full-viewport interstitial ("18+ section · I'm 18 or older / Take me back"), accept → set flag + render, decline → `router.navigate('/guide')`. **sessionStorage, not cookies/localStorage** (MHMDA-flavored minimalism: nothing persistent, nothing sent to the server). The `/guide/sections/adult` index route also mounts the gate.
- **Link-out cards (GD1 §4/§7):** platform icon + label, outbound `target="_blank" rel="noopener noreferrer"`; used for "where this scene talks" on profiles AND as the whole body of `link_out`-posture scene landings ("we point you to this scene's own curators, with credit").
- **Cross-stream reads happen in Stream B files:** venue/scene pages show "coming up" by calling Stream A's public `/api/events/upcoming?venue_id=…` / `?scene=…` from `GuideAccess` (own HTTP method, own mock) — no import of Stream A client code, no payload coupling. Sub-fetch failure or an empty result renders as a quiet empty state, never an error (the knottyyoga isolation pattern) — so this bucket works even before EV1's endpoints exist.
- **Venue-page identity:** URLs use `venue_profiles.slug` (`/guide/venues/:slug`). A resolver route `/guide/venues/by-id/:venueId` looks the slug up from the directory listing client-side and redirects — this is what lets Stream A's event pages link to venue pages later with zero payload change (EV2 OQ3's follow-up: after this lands, ping Levi; linking is a one-line edit on their side). Every directory item carries `venue_id` for this. **The lookup must ask for `include_closed=true`** ([[Bucket GD2 — Directory data layer, server]] OQ11 (c)): the directory hides closed and moved venues by default, and an old event at a venue that has since closed should still land on its graveyard page rather than a not-found.
- **Markdown:** `guide_pages.body_markdown` renders via ngx-markdown with `provideMarkdown()` **route-scoped on the guide routes** (knottyyoga's blog keeps `marked` out of the initial bundle this way). Content is trusted-editor-authored; default sanitization stays on. Per [[Bucket GD2 — Directory data layer, server]] OQ5 (10/26): bodies may contain raw HTML (filtered by that sanitizer — no embeds at MVP), and they link to other guide pages with plain relative markdown links (`/guide/venues/<slug>`, …). Those render as ordinary `<a href>`s, so the rendered body carries the in-app-links directive (§2.6) to turn same-origin clicks into router navigations instead of full reloads.

# Layered work items

*(Access seam first, then shared components, then pages; specs named per item.)*

### 1. Access seam (`ui/src/app/core/guide-access/` — mirror `community-access/`'s 4 files)

*(Built 10/29.)*
- vitest is 226/226 in 32 files, up from 173 in 29: 53 new tests in 3 new spec files.
- The suite also passes with the machine set to UTC and to Tokyo time.
- `ng build -c local`, `-c development` and `-c production` all build.
- The initial bundle grew by 23 kB. That goes away when §3.1 moves the provider into the guide routes (OQ7 (d)).
- **§1 is complete.**

- [x] 1.1 `guide-access.ts` — `GUIDE_ACCESS` token + interface: `getScenes()`, `getScene(slug)`, `getVenues(filters?)`, `getVenue(slug)`, `getOrgs(filters?)`, `getOrg(slug)`, `getPlaces()`, `getPlace(slug)`, `getServices(kind?)`, `getNeighborhoods()`, `getNeighborhood(slug)`, `getGraveyard()`, `getGuidePage(slug)`, `getGuidePagesBySection(section)`, `getVenueEvents(venueId)` + `getSceneEvents(sceneSlug)` (the Stream A public-API reads); types file with `normalizeX` statics (`?? []`, number coercion, freshness fields on every entity type).
	- *As built (10/29):* all 16 reads, as listed. The organization pair is spelled out as `getOrganizations` / `getOrganization` (OQ7 (a)).
		- **Types:** `core/types/guide.types.ts`, beside EV2's `events.types.ts` and in its style: exported `normalizeX` functions. They mirror GD2's wire shapes field for field (checked against `guide_key_value_table.cpp` and `guide_json.cpp`).
			- Every listing type extends a shared `GuideFreshness` (`status`, `last_verified_at_us`).
			- A guide venue carries both raw verified dates as well as the combined one.
			- A missing intro or rundown page stays `null`.
		- **The event reads** come back as the guide's own small `GuideEvent`, only the fields a "coming up" list shows, not Stream A's event type (OQ7 (b)).
- [x] 1.2 `guide-access.http.ts` — HTTP impl over `/api/guide/*` (+ `/api/events/upcoming` for the two event reads), `HONUWARE_API_BASE`, `withCredentials: true`, query-string building. **Spec:** URL/query construction per method (HttpClient spy).
	- *As built (10/29):* GD2's routes as built.
		- `getOrganizations` asks for `/api/guide/orgs`.
		- Filters go in a fixed order. `include_closed` is sent only when it's on, as `true`.
		- Slugs are URL-encoded.
		- A 404 passes through for the page to show "not found".
		- **Specs** (25, `guide-access.http.spec.ts`): every method's URL, method and credentials (one table-driven test per method); the filters; slug encoding; reading a scene landing, the graveyard and a venue's events; the 404. Plus 10 for the normalizers in `guide.types.spec.ts`: all-digit text stays text, `""` becomes null, missing fields are defaulted, a missing intro page stays null, and the query-string builder.
- [x] 1.3 `guide-access.mock.ts` — the offline demo dataset (relative dates where dated): GD1's scene tags with postures; 5 venues with profiles across neighborhoods (one **closed** with a closure note, one **adult**); 6 orgs across kinds (league, chorus, recovery, professional…); Denny Blaine as an adult place; 6 services (mixed `is_health`); 3 published pages (a scene intro, a neighborhood rundown, an adult-section page) + 1 draft that must never surface; links on several entities; demo venue events. **Spec:** draft never returned; graveyard returns exactly the closed entities.
	- *As built (10/29):* the dataset is in `core/mock/guide-mock-data.ts`, beside the events mock. `GuideMockAccess` serves it by GD2's rules:
		- Closed and moved listings are left out of every list but the graveyard. The graveyard has closed listings only.
		- `include_closed` brings closed venues back.
		- Scene and neighborhood counts equal what their pages list.
		- Only published pages show, as of the moment you ask.
		- Events come from the 42-day window, in start order.
		- An unknown slug is a 404, and a missing section is a 400.
		- The data is a bit larger than the list above (OQ7 (c)):
			- 6 venues.
			- 9 organizations, including one closed, one moved and one 18+.
			- 2 places.
			- 6 services.
			- 5 published pages, plus a draft and one scheduled for next week.
			- 10 events.
		- Everything is made up apart from GD1's vocabularies and the name "Denny Blaine Park". Its listing says nothing about the park beyond being 18+.
		- **Specs** (18, `guide-access.mock.spec.ts`):
			- The draft and the scheduled page never show, and the scheduled one does once its week has passed.
			- The graveyard is exactly the closed venue and the closed organization, not the moved one.
			- Retired listings are out of the directories, but their own pages still load.
			- Every filter works. Every scene's and neighborhood's counts match its page. The link-out postures and the slug conventions hold.
			- Each unknown slug is a 404.
			- The 18+ layer is there.
			- Event reads stay within the window.
			- The mock hands out copies.
			- Parity: running the normalizers over the mock data changes nothing.
			- Every venue's stamp is the older of its two dates.
- [x] 1.4 `guide-access.providers.ts` — `provideGuideAccess(useMock)`; wire into `app.config.ts` beside `provideCommunityAccess` (one-line shared-file edit).
	- *As built (10/29):* as planned, one provider line and its import in `app.config.ts`, directly after `provideCommunityAccess`. This is what costs the initial bundle its 23 kB. §3.1 moves it into the guide routes (OQ7 (d)).

### 2. Shared components (`ui/src/app/shared/components/`)

*(Built 10/29.)*
- vitest is 251/251 in 39 files, up from 226 in 32: 25 new tests in 7 new spec files.
- The suite also passes with the machine set to UTC and to Tokyo time.
- `ng build -c local`, `-c development` and `-c production` all build.
- The production initial bundle is unchanged at 893 kB. No page uses these pieces yet, so none of them are in any bundle until §3.
- **§2 is complete.** OQ9 lists the calls made along the way.
- **Helpers the components share**, beyond the plan:
	- `core/utils/guide-date.utils.ts` is OQ5's small formatter, after all. It gives "Verified {Month Year}" or "Not yet verified" for a card, dated in Seattle time like the footer (OQ9 (d)).
	- `core/utils/guide-labels.ts` holds the display names for organization, place and service kinds, and link platforms. They're written so that a kind added later without a name fails to compile.
	- `core/services/guide-vocabularies.service.ts` gives the scene and neighborhood display names that chips and cards show. It's fetched once per visit, and a failed fetch falls back to showing slugs. It copies EV2's `EventVocabulariesService`, but reads through the guide's own seam (OQ9 (c)).

- [x] 2.1 `freshness-footer/` — **reuse EV2's; nothing to build** (OQ8, 10/29).
	- Its inputs are `status`, `lastVerifiedAtUs` and `closedLabel` (default "Closed"). The guide's `GuideFreshnessStatus` is the same list of statuses as its own, so a page passes a listing's `status` and `last_verified_at_us` straight in.
	- The closure note isn't the footer's job. Pages show it beside the top-of-page "Closed" banner (§3.4, §3.5, §3.8).
	- **Spec:** none here. EV2's `freshness-footer.spec.ts` covers the footer, and the page specs check the banner and note.
- [x] 2.2 `link-out-card/` — input: the GD1 link shape; platform icon mapping; outbound rel attributes. **Spec:** href + rel.
	- *As built (10/29):* the card shows the link's label and always the platform's name ("Discord", "Facebook group", …, "Link" for `other`). It opens in a new tab with `rel="noopener noreferrer"`.
		- Material Icons has no brand logos, so each platform gets the nearest plain icon, like a speech bubble for Discord or a camera for Instagram (OQ9 (e)).
		- **Specs** (3): href, target and rel; label and platform name; every platform has a name and an icon.
- [x] 2.3 `scene-chips/` — chips from scene-tag slugs → `routerLink /guide/scenes/:slug`. **Spec.**
	- *As built (10/29):* inputs `slugs` and `scenes` (the scene list, for display names). A slug that isn't in the list shows as itself, and no tags renders nothing.
		- The chips look like the events pages' tag chips, but with their own styles rather than Stream A's global `.event-chip` class.
		- **Specs** (2): links and names, with the slug standing in for a name; nothing when there are no tags.
- [x] 2.4 `adult-gate/` — the interstitial wrapper per the pin-down (content-projected body renders only past the gate). **Spec:** blocked without the flag; accept sets sessionStorage + renders; decline navigates away.
	- *As built (10/29):* the 18+ gate as pinned.
		- It uses `sessionStorage['cf_adult_ok']`. "Take me back" goes to `/guide`.
		- Its note says the answer is kept only until the tab closes and isn't sent anywhere.
		- Its `adult` input defaults to on. A page passes its listing's flag, so the same wrapper lets non-adult content straight through.
		- OQ4's comment (noindex is RT4's call) is in the component.
		- **Two changes from the plan** (OQ9 (a), (b)):
			- The gated body goes inside an `<ng-template>`, so nothing in it is created before a yes. Its photos don't load and its fetches don't run.
			- The question fills the page under the header rather than covering the whole screen.
		- If the browser blocks storage, a yes still opens the page; it just isn't remembered.
		- **Specs** (6): asks first, without creating the body; a yes shows the body and is stored for this tab only, not in localStorage; no second ask in the same tab; a no goes to `/guide` without the body; non-adult content passes straight through; blocked storage.
- [x] 2.5 `entity-card/` (venue/org/place/service card: name, chips, neighborhood, one-line description, freshness stamp inline, closed badge) — the list-item workhorse. **Spec.**
	- *As built (10/29):* the card takes one shape for every listing type, built by `venueCardItem` / `organizationCardItem` / `placeCardItem` / `serviceCardItem`. It also takes the vocabularies, for names.
		- It shows the name (linked to the listing's page), a "Closed", "Moved" or "Seasonal" badge (the footer's markers), an "18+" badge, kind and neighborhood ("Sports league · Capitol Hill"), the description cut to one line, scene chips, and "Verified {Month Year}".
		- A closed or moved listing is faded and shows its closure note. That's what the graveyard and the "include closed" directory need. An open listing never shows its note.
		- A service has no page of its own, so its name links out to its site in a new tab, or is plain text if it has none (OQ9 (e)).
		- **Specs** (5): a venue; a closed organization; an open listing hides its note; a seasonal, never-verified, 18+ place; a service linking out, and one without a site.
- [x] 2.6 `ui/src/app/shared/directives/in-app-links/` — attribute directive for a rendered-markdown host: listens for clicks on descendant `<a>`s and, for a same-origin href with a plain left-click (no modifier keys, no `target`, no `download`), calls `preventDefault()` + `router.navigateByUrl(path + query + hash)`. External, `mailto:`, `tel:`, and modified clicks fall through to the browser. **Spec:** relative link → router navigation, no reload; external link untouched; ctrl/meta-click untouched.
	- *As built (10/29):* `appInAppLinks`, as planned.
		- It also leaves alone middle-clicks, shift and alt clicks, `#anchor` jumps on the same page, and the server's own `/api/…` addresses (a calendar feed, say), which aren't app pages.
		- The app's root comes from `<base href>` rather than being assumed to be `/`.
		- **Specs** (5): an app link navigates in-app, with its query and anchor; a click on something inside a link; other sites, mail and phone links; modified and middle clicks; new-tab, download, same-page and `/api` links.

### 3. Guide pages (`ui/src/app/pages/guide/`, lazy `guide.routes.ts`; `app.routes.ts` one-liner `{ path: 'guide', loadChildren: … }`; header gains "Guide" — one-line shared-file edits)

- [ ] 3.1 Route scaffolding — child routes in hazard-safe order (literals before `:slug`): `''` home · `venues` · `venues/by-id/:venueId` · `venues/:slug` · `orgs` · `orgs/:slug` · `places/:slug` · `services` · `neighborhoods` · `neighborhoods/:slug` · `graveyard` · `scenes/:slug` · `sections/:section` · `pages/:slug`; `provideMarkdown()` in the route providers. **Move `provideGuideAccess(environment.useMock)` out of `app.config.ts` into those same route providers** (OQ7 (d)), so the seam and its demo data load only in the guide chunk. `GuideVocabulariesService` (§2) goes in the same providers; it isn't app-wide, since it reads through the seam. Every page sets `document.title` and carries loading / notFound / error flags (knottyyoga detail-page pattern).
- [ ] 3.2 `GuideHomePage` (`/guide`) — the hub per GD1 §7: scene grid (full-coverage scenes as tiles; `link_out` scenes as a credited "sister scenes" card row), section links (Health · Services · New to Seattle · Beat the Freeze · 18+), neighborhoods + graveyard links. **Spec:** postures split correctly from mock.
- [ ] 3.3 `SceneLandingPage` (`/guide/scenes/:slug`) — GD1 §7's scene template: intro article (page-slug convention), venues, orgs, places, links, "this scene's upcoming events" sub-fetch (quiet-empty). `link_out` scenes render intro + link-out cards + a note on the coverage rule instead of entity sections. **Specs:** full vs link-out branches; sub-fetch failure doesn't break the page.
- [ ] 3.4 `VenueDirectoryPage` (`/guide/venues`) — entity cards + filter bar (scene, neighborhood, "include closed" toggle); `VenueProfilePage` (`/guide/venues/:slug`) — the GD1 §7 venue template: header + status badge, crowd/vibe, practical block, chips, links, **upcoming events sub-fetch**, photos (`PhotoUrlBuilder('venues', venueId, …)`), freshness footer; adult-flagged profiles wrap in `AdultGate`. Plus the `by-id/:venueId` resolver-redirect. **Cross-stream venue-link-up (per EV2 OQ3, 8/14 — mirrored in EV2 §3.8):** whichever of EV2/GD3 lands **second** wires Stream A's `EventDetailPage` venue name to `/guide/venues/by-id/{venue_id}` (a one-line edit on their side) and calls it out in that landing PR's checklist so it can't slip. **"Subscribe to this venue's events" (optional, per EV2 OQ6 (d), 10/28):** EV2's calendar feed `/api/events/feed.ics` takes category, scene, neighborhood and series filters, but not `venue_id` yet. If the venue page wants a subscribe link, ask Stream A for the `venue_id` filter first. It's a one-line server change, since the upcoming list already filters by venue. **Specs:** closed venue renders with banner + note; by-id redirects; adult profile gated.
- [ ] 3.5 `OrgDirectoryPage` (`/guide/orgs`) — kind filter (the Freeze directory's data surface: leagues/choruses/recovery/faith/professional/social), scene + neighborhood filters; `OrgProfilePage` (`/guide/orgs/:slug`) — org template (what/when/where, how to join, links, freshness). **Specs.**
- [ ] 3.6 `PlacePage` (`/guide/places/:slug`) — place fields + related article links; adult places gated. `ServicesPage` (`/guide/services`) — kind-grouped list of link-out service cards (services have no detail pages at MVP). **Specs:** adult gating; kind grouping.
- [ ] 3.7 `NeighborhoodsPage` (`/guide/neighborhoods`) — index grouped by GD1 §3's regions (Eastside first-class); `NeighborhoodPage` (`/guide/neighborhoods/:slug`) — rundown article (slug convention) + venues/orgs/places here. **Specs.**
- [ ] 3.8 `GraveyardPage` (`/guide/graveyard`) — the closure graveyard: closed venues + orgs with closure notes and dates, framed as history (voice per GD1 §6). **Spec.**
- [ ] 3.9 `GuideArticlePage` (`/guide/pages/:slug`) — title, markdown body (with the §2.6 in-app-links directive on the rendered body), related-listing links (by shared scene tags), freshness footer; `adult` pages gated; 404 state for unknown/unpublished slugs. The scene-intro and neighborhood-rundown bodies (§3.3, §3.7) render through the same markdown + directive pairing. `SectionIndexPage` (`/guide/sections/:section`) — published pages of that section (title + verified stamp list); `adult` section mounts the gate. **Specs:** markdown renders; an in-body guide link navigates in-app; draft slug → notFound; adult article gated.

### 4. Gates

- [ ] 4.1 `ng build -c local` + `ng build -c development` clean; guide area lazy-chunked (initial bundle unchanged; `marked` only in the guide chunk). "Unchanged" is measured against EV2's recorded 870 kB production initial, so the 23 kB §1 added must be gone (OQ7 (d)).
- [ ] 4.2 vitest green; record the new count.
- [ ] 4.3 **Manual browser loop ≈ milestone M2's UI half (Mason):** offline (`ng serve`, mock): browse home → scene (full + link-out) → venue profile (open, closed, adult-gated) → orgs by kind → neighborhood → graveyard → article; interstitial accepts/declines correctly; then dev mode against the real server: create a venue + profile + page via admin CRUD → browse them live; verify the "Verified {Month Year}" stamp reflects a `verify-listing` run from the test_helper.

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **18+ gate = sessionStorage interstitial** (no persistence, no cookie, no server round-trip), per the Q3(b)/MHMDA posture. *Rec: as pinned; revisit only if WA passes an AV law (tracked in the brainstorm).*
	- Mason- (10/29, in chat) Accept all the open question recommendations.
	- Claude- Decided (10/29) — as pinned (the 18+ gate pin-down, §2.4); no body change.
2. **`/guide/venues/by-id/:venueId` resolver** (client-side lookup + redirect) as the cross-stream linking seam for event pages. *Rec: include; it's ~20 lines and unblocks EV2's venue links without any API change.*
	- Mason- (10/29, in chat) Accept all the open question recommendations.
	- Claude- Decided (10/29) — included (§3.4), with `include_closed=true` per GD2 OQ11 (c); no body change.
3. **ngx-markdown dependency** (with `marked`), route-scoped via `provideMarkdown()` — the knottyyoga blog wiring. *Rec: yes; pin versions compatible with Angular 21 and commit the lockfile.*
	- Mason- (10/29, in chat) Accept all the open question recommendations.
	- Claude- Decided (10/29) — yes. The dependency arrives with §3.1, the first item that renders markdown; §1 doesn't need it.
4. **noindex on adult pages?** Site-wide robots/meta policy is RT4's; nothing here blocks on it. *Rec: defer to RT4; leave a code comment at the AdultGate.*
	- Mason- (10/29, in chat) Accept all the open question recommendations.
	- Claude- Decided (10/29) — deferred to RT4; the code comment goes in with §2.4.
5. **Tiny local date formatter** in `freshness-footer` (Month Year only) instead of importing Stream A's `events-date.utils`. *Rec: as pinned — 5 lines of `Intl.DateTimeFormat` beats a cross-stream import; if the streams later want one shared date module, merge it as a deliberate post-M2 cleanup.*
	- Mason- (10/29, in chat) Accept all the open question recommendations.
	- Claude- Decided (10/29) — as pinned (§2.1); no body change.
	- ~~Claude- Superseded (10/29) by OQ8. GD3 reuses EV2's footer, so it never formats a date and needs no formatter.~~
	- Claude- Correction (10/29, v0.5): that was wrong. The footer is EV2's, but §2.5's cards and §3.9's section index show the "Verified" stamp inline. So the local formatter is built as OQ5 pinned it, in `core/utils/guide-date.utils.ts` (OQ9 (d)).
6. **Empty-state copy** ("Nothing here yet — this guide section is being written") appears wherever content hasn't landed, so GD3 can merge before GD4b/Pool content exists. *Rec: as pinned; the mock dataset keeps the demo experience full regardless.*
	- Mason- (10/29, in chat) Accept all the open question recommendations.
	- Claude- Decided (10/29) — as pinned (§3); no body change.
7. **§1 as built — four calls the plan didn't spell out.** *(Added 10/29.)* (a)–(c) are built as recommended, so "Agreed" covers them. **(d) needs your call**, because it changes when the guide code loads. Answer per letter if you want any changed.
	- **(a) "Organizations" spelled out.** The plan named the methods `getOrgs` / `getOrg`. They're built as `getOrganizations` / `getOrganization`, following the house rule against abbreviations, as GD2 did for its files (GD2 OQ12 (a)). The web addresses stay `/api/guide/orgs` and `/guide/orgs/<slug>`. *Rec: keep.*
	- **(b) Stream B keeps its own copies.** Per the cross-stream pin, the guide imports nothing from Stream A's client code. So it has its own small event type (just what a "coming up" list shows), its own copy of GD1's scene and neighborhood lists for the mock, and its own copy of the few small helpers that clean up the server's JSON. That's under 100 lines repeating what EV2 has. *Rec: keep. Merging them into one shared module is a deliberate later cleanup, like OQ5's date module.*
	- **(c) The demo data is a bit bigger than planned, on purpose.**
		- **Venues:** venues 1–4 are the events demo's own venues, with the same ids and names. So when EV2's event pages start linking to venue pages, the link also works offline.
		- **Organizations:** there's an 18+ one, because GD2 added that box (GD2 OQ11 (b)). There's also a moved one, to prove that a moved listing stays out of both the directory and the graveyard. The softball league is marked seasonal, so pages have a seasonal marker to show.
		- **Pages:** a page scheduled for next week sits beside the draft, to prove neither shows early.
		- *Rec: keep.*
	- **(d) The guide's data code loads with the app, adding 23 kB.** §1.4 puts the provider in `app.config.ts`. That means the seam and its offline demo data load with the app's first download, even on pages that aren't the guide and in the production build. The production initial bundle went from 870 kB (EV2's recorded size) to 893 kB. §4.1 asks for the initial bundle to stay unchanged.
		- The fix: when §3.1 creates `guide.routes.ts`, move the provider into those routes. The seam and its data then load only when someone opens a guide page. Nothing outside `/guide` uses the seam, since EV2 links to venue pages by address.
		- *Rec: move it at §3.1. That's one line out of `app.config.ts` and one line into the guide routes. Alternative: leave it in `app.config.ts` and accept the 23 kB.*
	- I'll go with your recommendations.
	- Claude- Decided (10/29) — (a)–(c) stay as built. (d): the provider moves into `guide.routes.ts` at §3.1 (added to 3.1 and 4.1). Until then it stays in `app.config.ts`, so the initial bundle carries the 23 kB in the meantime.
8. **The freshness footer already exists.** *(Added 10/29, a heads-up for §2.)* EV2 built `shared/components/freshness-footer` for its series pages, and its doc offers it to GD3. It already does most of §2.1:
	- It shows "Verified {Month Year}", or "Not yet verified".
	- It adds a "Closed", "Moved" or "Seasonal" marker. The "Closed" wording can be changed; series say "Ended".
	- It has no slot for a closure note.
	- It uses Stream A's own date formatter.

	So OQ5 (a local date formatter) and §2.1 (a new footer at that same path) were planned without knowing it exists.
	- *Rec:*
		- GD3 uses EV2's footer as it is, and shows the closure note on the page itself, next to the "Closed" banner at the top. That's where GD1's venue page puts the status anyway.
		- §2.1 then becomes "reuse; no new footer". OQ5's formatter isn't needed, because GD3 never formats a date for the footer.
		- A shared component in `shared/` is meant to be shared, so importing it isn't the payload coupling the cross-stream pin guards against.
	- *Alternatives:*
		- Add an optional closure-note input to EV2's footer. That's a small edit to Stream A's file, so ping Levi first.
		- Build GD3's own footer as planned, under a different name.
	- Mason- I'll go with your recommendation.
	- Claude- Decided (10/29) — reuse EV2's footer as it is. §2.1 is now "reuse; nothing to build" and is checked off. The Freshness pin-down, Scope and "Read before starting" lines say so. The closure note goes on the page beside the "Closed" banner. ~~OQ5's formatter is dropped.~~ *(Corrected in v0.5: the cards still need it; see OQ5.)*
9. **§2 as built — five calls the plan didn't spell out.** *(Added 10/29.)* All are built as recommended, so "Agreed" covers them. Answer per letter if you want any changed.
	- **(a) The 18+ gate holds the page as a template.** The plan said the gate wraps the page's content and shows it only after a yes. Done the plain way, Angular still *builds* the hidden content in the background. Its photos could start downloading and its data fetches could run before the reader answered. So a page puts its body inside an `<ng-template>` within the gate, and nothing in it exists until a yes. That costs one extra wrapper line per gated page. *Rec: keep.*
	- **(b) The 18+ question doesn't cover the header.** The plan said "full-viewport". It's built to fill the page area under the header, with the question centered, so the site's menu stays usable. Nothing behind it is visible either way, because of (a). *Rec: keep. Alternative: cover the whole screen.*
	- **(c) A small service for scene and neighborhood names.** The server sends slugs like `capitol-hill`, and the chips and cards need "Capitol Hill". `GuideVocabulariesService` fetches both lists once per visit, the way EV2's `EventVocabulariesService` does for events, and pages hand them to the cards. It can't be app-wide once the seam moves into the guide routes (OQ7 (d)), so §3.1 provides it there too. *Rec: keep.*
	- **(d) A correction to v0.4: OQ5's date formatter is needed after all.** v0.4 said GD3 would never format a date, because the footer is EV2's. But cards (and §3.9's section index) show "Verified {Month Year}" inline, and those aren't footers. The formatter is built as OQ5 originally pinned it, a dozen lines of code, dated in Seattle time so a card and its page's footer always name the same month. *Rec: keep. That's OQ5's own decision, restored.*
	- **(e) Card and link details.**
		- A service has no page, so its card's name links out to its website, in a new tab.
		- Cards show an "18+" badge on adult listings, so a reader knows before clicking that the page will ask.
		- Link-out cards use plain icons (Material Icons has no Discord or Instagram logos), and the platform's name is always written next to the icon, as GD1 asks.
		- *Rec: keep.*

# Change Log

- **10/29/2026 (v0.5)** — **§2 (shared components) built.** 2.2–2.6 are marked complete, with as-built notes. 2.1 was already done (reusing EV2's footer).
	- The work is the link-out card, scene chips, the 18+ gate, the entity card and the in-app-links directive. Three helpers come with them: the guide's date formatter, its display-name lists, and a scene and neighborhood names service.
	- 25 new tests in 7 new spec files. vitest is 251/251, up from 226. It also passes in UTC and in Tokyo time.
	- The local, development and production builds all succeed. The production initial bundle is unchanged at 893 kB.
	- §3.1 now also provides `GuideVocabulariesService` in the guide routes.
	- **Correction to v0.4:** OQ5's formatter is needed for the cards' inline stamp. The "Read before starting" line, OQ5 and OQ8's decided line are fixed.
	- **OQ9 added:** five as-built calls, all recommended to keep.

- **10/29/2026 (v0.4)** — **OQ7 and OQ8 decided** (Mason went with the recommendations).
	- OQ7 (a)–(c) stay as built. For (d), §3.1 now moves `provideGuideAccess` from `app.config.ts` into the guide routes, and §4.1 says the initial bundle must be back to EV2's 870 kB.
	- OQ8: GD3 reuses EV2's freshness footer. §2.1 is checked off with nothing to build. The Freshness pin-down now puts the closure note on the page beside the "Closed" banner. Scope and "Read before starting" are updated, and OQ5 is marked superseded.

- **10/29/2026 (v0.3)** — **OQ1–OQ6 decided** (Mason accepted every recommendation; no body changes). **§1 (access seam) built:**
	- 1.1–1.4 are marked complete, with as-built notes.
	- The work is the seam's four files, the guide types, the offline demo dataset, and 53 new tests in 3 new spec files.
	- vitest is 226/226, up from 173. It also passes in UTC and in Tokyo time.
	- The local, development and production builds all succeed.
	- **OQ7 added:** four as-built calls. (d), where the guide's data code loads, needs Mason's call: it adds 23 kB to the initial bundle unless it moves into the guide routes at §3.1.
	- **OQ8 added:** a heads-up for §2. EV2 already built a shared freshness footer that GD3 can reuse.

- **10/28/2026 (v0.2.1)** — Mirrored [[Bucket EV2 — Events public UI]] OQ6 (d) into §3.4: the events calendar feed has no `venue_id` filter yet, so a "subscribe to this venue's events" link means asking Stream A for that one-line server change first.
- **10/26/2026 (v0.2)** — Folded in [[Bucket GD2 — Directory data layer, server]] OQ5 (decided 10/26): article bodies are markdown with sanitizer-filtered HTML, and they link with plain relative links. The Markdown pin is extended, a new §2.6 `in-app-links` directive turns same-origin clicks into router navigations, and §3.9 applies it (also to the scene-intro and neighborhood-rundown bodies) with an in-app-navigation spec.
- **8/12/2026 (v0.1)** — Initial draft.
