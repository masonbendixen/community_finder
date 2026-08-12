---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen (owner)
Last Updated: 8/12/2026
Version: 0.1
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
- EV2's `events-date` utilities are Stream A's files — this bucket needs only "Month Year" formatting; keep a tiny local formatter rather than importing across streams (see OQ5).

# Scope

**In:** `GuideAccess` seam + full offline mock dataset · guide client area (`/guide/*`): guide home, scene landings (full + link-out postures), venue directory + venue profiles (incl. graveyard rendering), org directory (the Freeze-directory data view) + org profiles, places, services list, neighborhoods index + pages, guide-article pages with markdown rendering, section indexes · shared freshness-footer + link-out-card + scene-chips components · the 18+ interstitial · "upcoming events at this venue/scene" sub-fetches (via Stream A's public API, from Stream B-owned code) · header "Guide" nav link · vitest specs throughout.

**Out (explicit non-goals):** all editorial content (**Pool GD5–GD10** — this bucket ships with mock/demo content and empty-state handling) · registry data entry (**GD4b**) · the "pick your onramp" interactive flow (**GD8-flow** — it mounts at `/guide/onramp` later) · report-stale flags, claims (**CM1**) · site-wide SEO/meta/OG (**RT4** — pages set `document.title` only) · event pages (**EV2**) · admin editing UI (the Phase 7 CRUD editor already covers guide tables via GD2's metadata).

# Design pin-downs

- **Freshness rendering (GD1 §5/§7):** every entity/article page ends with the freshness footer: "Verified {Month} {Year}" from `last_verified_at_us` (absent → "Not yet verified" in the same quiet style — honesty over polish); `status != 'open'` adds the marker banner (venues/orgs: "Closed" + `closure_note`; `seasonal`: "Seasonal"; series-style "Ended" wording stays Stream A's concern). Closed pages render fully — the graveyard is history, not an error state.
- **18+ gate (Q3b):** a data-driven interstitial, not age verification (WA has no AV law as of mid-2026; the brainstorm's posture). Mechanics: entities/pages carry `adult` from the server; an `AdultGate` wrapper component checks `sessionStorage['cf_adult_ok']` — unset → full-viewport interstitial ("18+ section · I'm 18 or older / Take me back"), accept → set flag + render, decline → `router.navigate('/guide')`. **sessionStorage, not cookies/localStorage** (MHMDA-flavored minimalism: nothing persistent, nothing sent to the server). The `/guide/sections/adult` index route also mounts the gate.
- **Link-out cards (GD1 §4/§7):** platform icon + label, outbound `target="_blank" rel="noopener noreferrer"`; used for "where this scene talks" on profiles AND as the whole body of `link_out`-posture scene landings ("we point you to this scene's own curators, with credit").
- **Cross-stream reads happen in Stream B files:** venue/scene pages show "coming up" by calling Stream A's public `/api/events/upcoming?venue_id=…` / `?scene=…` from `GuideAccess` (own HTTP method, own mock) — no import of Stream A client code, no payload coupling. Sub-fetch failure or an empty result renders as a quiet empty state, never an error (the knottyyoga isolation pattern) — so this bucket works even before EV1's endpoints exist.
- **Venue-page identity:** URLs use `venue_profiles.slug` (`/guide/venues/:slug`). A resolver route `/guide/venues/by-id/:venueId` looks the slug up from the directory listing client-side and redirects — this is what lets Stream A's event pages link to venue pages later with zero payload change (EV2 OQ3's follow-up: after this lands, ping Levi; linking is a one-line edit on their side).
- **Markdown:** `guide_pages.body_markdown` renders via ngx-markdown with `provideMarkdown()` **route-scoped on the guide routes** (knottyyoga's blog keeps `marked` out of the initial bundle this way). Content is trusted-editor-authored; default sanitization stays on.

# Layered work items

*(Access seam first, then shared components, then pages; specs named per item.)*

### 1. Access seam (`ui/src/app/core/guide-access/` — mirror `community-access/`'s 4 files)

- [ ] 1.1 `guide-access.ts` — `GUIDE_ACCESS` token + interface: `getScenes()`, `getScene(slug)`, `getVenues(filters?)`, `getVenue(slug)`, `getOrgs(filters?)`, `getOrg(slug)`, `getPlaces()`, `getPlace(slug)`, `getServices(kind?)`, `getNeighborhoods()`, `getNeighborhood(slug)`, `getGraveyard()`, `getGuidePage(slug)`, `getGuidePagesBySection(section)`, `getVenueEvents(venueId)` + `getSceneEvents(sceneSlug)` (the Stream A public-API reads); types file with `normalizeX` statics (`?? []`, number coercion, freshness fields on every entity type).
- [ ] 1.2 `guide-access.http.ts` — HTTP impl over `/api/guide/*` (+ `/api/events/upcoming` for the two event reads), `HONUWARE_API_BASE`, `withCredentials: true`, query-string building. **Spec:** URL/query construction per method (HttpClient spy).
- [ ] 1.3 `guide-access.mock.ts` — the offline demo dataset (relative dates where dated): GD1's scene tags with postures; 5 venues with profiles across neighborhoods (one **closed** with a closure note, one **adult**); 6 orgs across kinds (league, chorus, recovery, professional…); Denny Blaine as an adult place; 6 services (mixed `is_health`); 3 published pages (a scene intro, a neighborhood rundown, an adult-section page) + 1 draft that must never surface; links on several entities; demo venue events. **Spec:** draft never returned; graveyard returns exactly the closed entities.
- [ ] 1.4 `guide-access.providers.ts` — `provideGuideAccess(useMock)`; wire into `app.config.ts` beside `provideCommunityAccess` (one-line shared-file edit).

### 2. Shared components (`ui/src/app/shared/components/`)

- [ ] 2.1 `freshness-footer/` — inputs: `lastVerifiedAtUs`, `status`, `closureNote?`; renders the stamp + marker per the pin-down (a local "Month Year" formatter — see OQ5). **Spec:** verified / never-verified / closed / seasonal renderings.
- [ ] 2.2 `link-out-card/` — input: the GD1 link shape; platform icon mapping; outbound rel attributes. **Spec:** href + rel.
- [ ] 2.3 `scene-chips/` — chips from scene-tag slugs → `routerLink /guide/scenes/:slug`. **Spec.**
- [ ] 2.4 `adult-gate/` — the interstitial wrapper per the pin-down (content-projected body renders only past the gate). **Spec:** blocked without the flag; accept sets sessionStorage + renders; decline navigates away.
- [ ] 2.5 `entity-card/` (venue/org/place/service card: name, chips, neighborhood, one-line description, freshness stamp inline, closed badge) — the list-item workhorse. **Spec.**

### 3. Guide pages (`ui/src/app/pages/guide/`, lazy `guide.routes.ts`; `app.routes.ts` one-liner `{ path: 'guide', loadChildren: … }`; header gains "Guide" — one-line shared-file edits)

- [ ] 3.1 Route scaffolding — child routes in hazard-safe order (literals before `:slug`): `''` home · `venues` · `venues/by-id/:venueId` · `venues/:slug` · `orgs` · `orgs/:slug` · `places/:slug` · `services` · `neighborhoods` · `neighborhoods/:slug` · `graveyard` · `scenes/:slug` · `sections/:section` · `pages/:slug`; `provideMarkdown()` in the route providers. Every page sets `document.title` and carries loading / notFound / error flags (knottyyoga detail-page pattern).
- [ ] 3.2 `GuideHomePage` (`/guide`) — the hub per GD1 §7: scene grid (full-coverage scenes as tiles; `link_out` scenes as a credited "sister scenes" card row), section links (Health · Services · New to Seattle · Beat the Freeze · 18+), neighborhoods + graveyard links. **Spec:** postures split correctly from mock.
- [ ] 3.3 `SceneLandingPage` (`/guide/scenes/:slug`) — GD1 §7's scene template: intro article (page-slug convention), venues, orgs, places, links, "this scene's upcoming events" sub-fetch (quiet-empty). `link_out` scenes render intro + link-out cards + a note on the coverage rule instead of entity sections. **Specs:** full vs link-out branches; sub-fetch failure doesn't break the page.
- [ ] 3.4 `VenueDirectoryPage` (`/guide/venues`) — entity cards + filter bar (scene, neighborhood, "include closed" toggle); `VenueProfilePage` (`/guide/venues/:slug`) — the GD1 §7 venue template: header + status badge, crowd/vibe, practical block, chips, links, **upcoming events sub-fetch**, photos (`PhotoUrlBuilder('venues', venueId, …)`), freshness footer; adult-flagged profiles wrap in `AdultGate`. Plus the `by-id/:venueId` resolver-redirect. **Specs:** closed venue renders with banner + note; by-id redirects; adult profile gated.
- [ ] 3.5 `OrgDirectoryPage` (`/guide/orgs`) — kind filter (the Freeze directory's data surface: leagues/choruses/recovery/faith/professional/social), scene + neighborhood filters; `OrgProfilePage` (`/guide/orgs/:slug`) — org template (what/when/where, how to join, links, freshness). **Specs.**
- [ ] 3.6 `PlacePage` (`/guide/places/:slug`) — place fields + related article links; adult places gated. `ServicesPage` (`/guide/services`) — kind-grouped list of link-out service cards (services have no detail pages at MVP). **Specs:** adult gating; kind grouping.
- [ ] 3.7 `NeighborhoodsPage` (`/guide/neighborhoods`) — index grouped by GD1 §3's regions (Eastside first-class); `NeighborhoodPage` (`/guide/neighborhoods/:slug`) — rundown article (slug convention) + venues/orgs/places here. **Specs.**
- [ ] 3.8 `GraveyardPage` (`/guide/graveyard`) — the closure graveyard: closed venues + orgs with closure notes and dates, framed as history (voice per GD1 §6). **Spec.**
- [ ] 3.9 `GuideArticlePage` (`/guide/pages/:slug`) — title, markdown body, related-listing links (by shared scene tags), freshness footer; `adult` pages gated; 404 state for unknown/unpublished slugs. `SectionIndexPage` (`/guide/sections/:section`) — published pages of that section (title + verified stamp list); `adult` section mounts the gate. **Specs:** markdown renders; draft slug → notFound; adult article gated.

### 4. Gates

- [ ] 4.1 `ng build -c local` + `ng build -c development` clean; guide area lazy-chunked (initial bundle unchanged; `marked` only in the guide chunk).
- [ ] 4.2 vitest green; record the new count.
- [ ] 4.3 **Manual browser loop ≈ milestone M2's UI half (Mason):** offline (`ng serve`, mock): browse home → scene (full + link-out) → venue profile (open, closed, adult-gated) → orgs by kind → neighborhood → graveyard → article; interstitial accepts/declines correctly; then dev mode against the real server: create a venue + profile + page via admin CRUD → browse them live; verify the "Verified {Month Year}" stamp reflects a `verify-listing` run from the test_helper.

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **18+ gate = sessionStorage interstitial** (no persistence, no cookie, no server round-trip), per the Q3(b)/MHMDA posture. *Rec: as pinned; revisit only if WA passes an AV law (tracked in the brainstorm).*
2. **`/guide/venues/by-id/:venueId` resolver** (client-side lookup + redirect) as the cross-stream linking seam for event pages. *Rec: include; it's ~20 lines and unblocks EV2's venue links without any API change.*
3. **ngx-markdown dependency** (with `marked`), route-scoped via `provideMarkdown()` — the knottyyoga blog wiring. *Rec: yes; pin versions compatible with Angular 21 and commit the lockfile.*
4. **noindex on adult pages?** Site-wide robots/meta policy is RT4's; nothing here blocks on it. *Rec: defer to RT4; leave a code comment at the AdultGate.*
5. **Tiny local date formatter** in `freshness-footer` (Month Year only) instead of importing Stream A's `events-date.utils`. *Rec: as pinned — 5 lines of `Intl.DateTimeFormat` beats a cross-stream import; if the streams later want one shared date module, merge it as a deliberate post-M2 cleanup.*
6. **Empty-state copy** ("Nothing here yet — this guide section is being written") appears wherever content hasn't landed, so GD3 can merge before GD4b/Pool content exists. *Rec: as pinned; the mock dataset keeps the demo experience full regardless.*
