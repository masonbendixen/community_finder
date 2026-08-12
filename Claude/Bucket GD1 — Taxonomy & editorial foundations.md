---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 8/12/2026
Version: 0.1
tags:
---

# Bucket GD1 — Taxonomy & editorial foundations

**Track:** GD — Guide & content · **Size:** S · **Stream:** Phase 0 (joint) · **Owner:** Mason
**Needs:** nothing — first bucket. **Feeds:** everything — EV1/EV2 (Stream A), GD2/GD3 (Stream B), the Content Pool (GD5–GD10), EV5's verifier, RT2's listing policy.

# Context

This is the lowest layer of the whole product: the shared vocabulary and editorial conventions every stream consumes. It implements [[Initial Project Implementation Outline]] Phase 0.2 and Interface Contracts 1 (shared vocabulary) and 3 (freshness convention), distilling [[Brainstorming on the website]] — Pillar 1 (categories + scene tags), Pillar 2 (guide sections + volatility), the freshness architecture, the Voice section, and Appendices A–H (seed data).

**This bucket is pure data + editorial design — no code.** The deliverables are the pinned lists and conventions **in this document**. Where a vocabulary becomes a database table, the table is *created and seeded by the owning stream's bucket* from the lists pinned here:

- **EV1 (Stream A) creates + seeds the shared vocabulary tables** — `categories`, `scene_tags`, `neighborhoods` — as its first work item, exactly per §1–§3 below. (Resolution of the outline's Contract-1 "seeded in Phase 0" wording vs. "GD1 = no code": GD1 locks the values in Phase 0; EV1 lands the DDL + seed as Slice 1 of the first code PR, because the outline's EV1 blurb already scopes "scene tags shared with the guide" to Stream A.)
- **GD2 (Stream B) FKs to those tables** and applies the freshness convention (§5) to its own tables.
- Vocabulary values are **admin-editable data, not code** — additions need no deploy. Renames of a *slug* need a cross-stream ping (Contract 1); display names are free to edit.

**GD1 is "Done" when:** Mason has reviewed and locked §1–§8 (edit inline, then check the boxes in Work Items), and the Stream A/B owners have acknowledged the contracts (outline Phase 0.3).

# Scope

**In:** category vocabulary · scene-tag vocabulary (with coverage posture) · neighborhood vocabulary · listing types + field definitions · the freshness convention (status + `last_verified_at` + volatility tiers + cadences) · slug/ID conventions · voice & style one-pager · page templates (what renders where) · listing-policy draft (feeds RT2).

**Out (explicit non-goals):** any code, DDL, or seed scripts (EV1/GD2 own those) · the registries themselves (~40 event sources, venues, orgs, health — that's GD4's content-ops, using these definitions) · editorial content drafting (Content Pool GD5–GD10) · site-wide SEO conventions (RT4) · brand/design tokens (RT1).

# §1 Category vocabulary (activity type — what kind of thing is happening)

Categories answer "what would I be doing?" and are orthogonal to scene tags ("whose crowd is it?"). Seed list is **locked by SUTP Q4a** (Mason's six + the agreed starters). One event may carry several categories.

**Table shape (created by EV1):** `categories` — id, `slug` (stable, kebab-case), `display_name`, `description`, `sort_order`, `enabled`. Admin-editable via the generic CRUD editor.

| slug | display_name | covers (examples from the research) |
|---|---|---|
| `bar` | Bar & club nights | recurring bar nights, dance parties, Safeword, MX |
| `drag` | Drag | Le Faux, Mimosas Cabaret, T4T, Fridays Are a Drag |
| `nightlife` | Nightlife | catch-all late-night: Kremwerk bookings, Massive events |
| `music` | Music & concerts | choruses, Rainbow City shows, DJ sets |
| `theatre` | Theatre & film | Intiman, ArtsWest, Jet City Improv, screenings |
| `arts` | Arts & culture | queer figure drawing, gallery nights, book clubs, history tours |
| `pride` | Pride & festivals | the June run, Alki Beach Pride, PNW Black Pride, Kremfest |
| `sports` | Sports & recreation | league play, climb nights, Pride Skate, runs, bowling |
| `outdoors` | Outdoors | hikes, OutVentures, Queer Mountaineers trips |
| `volunteer` | Volunteering | VolunQueers, Lambert House, cleanup days |
| `protest` | Protest & civic | marches, rallies, public-comment pushes |
| `community` | Community & social | meetups, Gay People in Seattle, game nights, recovery-adjacent socials, professional mixers |

**Candidate extensions** (add via admin the first time a real event needs one — no code, no re-litigating): `food-drink`, `film` (split from theatre), `games` (split from community), `fitness` (split from sports), `health` (testing drives, vaccine clinics), `faith`, `education`, `market`.

# §2 Scene-tag vocabulary (subculture — whose crowd it is)

Scene tags implement the brainstorm's spectrum rule structurally: every scene gets the same respectful, factual treatment. Tags apply to **events, venues (profiles), organizations, places, and guide pages** alike — this is the shared axis that makes "By Scene" views and scene landing pages work across both streams.

Each tag carries a **coverage posture** (Q2, decided 8/1): `full` = we curate this scene ourselves; `link_out` = we list the shared anchors and link out with credit to that community's own curators rather than pretending depth. Posture is a column so GD3 can render link-out cards automatically on `link_out` scene landings.

**Table shape (created by EV1):** `scene_tags` — id, `slug`, `display_name`, `description`, `coverage` (`full` | `link_out`), `sort_order`, `enabled`.

| slug | display_name | coverage | notes |
|---|---|---|---|
| `bear` | Bears | full | Diesel, Cuff, Lumber Yard, Bear Social |
| `leather` | Leather & kink | full | Eagle, Cuff, Doghouse Leathers; deep-dive content sits in the 18+ section per Q3(b) |
| `jock` | Jocks & sports | full | the 34+ leagues, climb nights, Frontrunners |
| `sober` | Sober | full | recovery meetings, Qu-Art, sober socials — the unserved wholesome layer |
| `elders` | Elders | full | GenPride/Pride Place, Rainbow Elder Play & Connect |
| `geek` | Geeks & gaymers | full | Seattle Gaymers, Board Gayme Night, Queer Geek!, Phoenix Comics |
| `furry` | Furries | full | per Pillar 1's tag list |
| `bipoc` | BIPOC | full | PNW Black Pride, Latinx Pride, Entre Hermanos, UTOPIA WA, Eastside QTPOC Collective; label accurately, credit organizers |
| `sapphic` | Sapphic | link_out | Wildrose + shared anchors listed; link out to Sapphie Taffy / Dyke Alliance with credit |
| `trans` | Trans | link_out | shared anchors (Trans Pride, T4T) listed with accurate labels; link out to Gender Justice League, Ingersoll, the trans rails |
| `qtpoc` | QTPOC | link_out | where organizers self-describe QTPOC (vs. gay BIPOC); link out with credit |

**Rules:** tag what's accurate, never aspirational (a mixed night is not `bear` because bears attend) · `sapphic`/`trans`/`qtpoc` tags are for accurate labeling of shared anchors we list anyway — they never gate a curation promise we aren't keeping · "new in town," "18+," and "Eastside" are **not** scene tags (the first is an onramp audience in GD8, the second is the `adult` flag in §4, the third is a neighborhood).

# §3 Neighborhood vocabulary (where — King County, Eastside first-class)

*(Added at draft time — the outline's Contract 1 names only categories + scene tags, but EV2's "By Neighborhood" events view and GD3's neighborhood pages both need one shared list, and D4's `venues` has only address/city. Flagged as OQ1.)*

**Table shape (created by EV1):** `neighborhoods` — id, `slug`, `display_name`, `region` (`seattle-core` | `seattle-north` | `seattle-south` | `eastside` | `south-king`), `sort_order`, `enabled`. Venues, organizations, places, service listings, and guide pages reference it by nullable FK (`neighborhood_id`); null = "location varies / online / countywide."

Seed (from [[Brainstorming on the website]] Appendix F, plus the venue-inventory locations that F's table lacks):

- **seattle-core:** `capitol-hill`, `first-hill`, `central-district`, `downtown` (incl. Denny Triangle — Kremwerk, Massive), `sodo`, `queen-anne`, `u-district`, `wallingford`, `ballard`, `fremont`
- **seattle-south:** `beacon-hill`, `columbia-city`, `georgetown`, `west-seattle`, `white-center`
- **seattle-north:** `shoreline`
- **eastside:** `bellevue`, `kirkland`, `redmond`, `bothell`, `issaquah`, `sammamish`
- **south-king:** `burien`, `renton`, `tukwila`

Rendering rule: region groups the neighborhood pickers and the GD3 neighborhood index; the Eastside is a first-class region, never "other."

# §4 Listing types & fields (what a listable thing is)

The guide's entity model, pinned so GD2's DDL and GD3's templates agree. **Bold** = the shared freshness columns from §5, required on every listable table (Contract 3).

| Listing type | Table (owner) | Fields beyond name/description |
|---|---|---|
| **Venue** | `venues` (EV1, Stream A — Contract 2) | address/city/state/zip, url, lat/lng, `neighborhood_id`, **status, last_verified_at** |
| **Venue guide profile** | `venue_profiles` (GD2, satellite FK → `venues.id` — Contract 2) | crowd/vibe editorial, price band, scene tags, `adult` flag, closure note (graveyard entries), "where this scene talks" links, **its own last_verified_at** (guide facts verify on a different cadence than the venue row) |
| **Organization** | `organizations` (GD2) | `org_kind` (`sports-league` · `chorus-performance` · `club-hobby` · `recovery` · `faith` · `professional` · `community-support` · `social-meetup`), url, meeting cadence text ("Mondays, Cal Anderson"), `neighborhood_id` (nullable), scene tags, links, **status, last_verified_at** |
| **Place** | `places` (GD2) | `place_kind` (`park-beach` · `landmark` · `historical`), address-ish, `neighborhood_id`, `adult` flag, scene tags, **status, last_verified_at** — the structured row behind Denny Blaine/Howell pages; the editorial itself is a guide page |
| **Service listing** | `service_listings` (GD2) | `service_kind` (`sti-testing` · `prep` · `hiv-care` · `primary-care` · `mental-health` · `gender-affirming` · `dental` · `everyday-service`), url/phone/address, `neighborhood_id`, `is_health` flag (drives the MHMDA rendering rule: no user attribution ever attaches to health listings — matters when CM2 arrives), **status, last_verified_at** |
| **Guide page** | `guide_pages` (GD2) | slug, `section` (`scenes` · `health` · `services` · `new-to-seattle` · `freeze` · `new-to` · `adult`), body (editorial), `adult` flag, publish state, scene tags, **status, last_verified_at** — the CMS-lite layer the Content Pool publishes through |
| **Event series** | `series` (EV1, Stream A) | cadence text ("2nd Saturdays"), venue FK, url, **status, last_verified_at** — "T4T, 2nd Saturdays" as a durable page while instances come and go |
| **Event** | `events` (EV1) | *workflow* status (`pending/approved/rejected/archived`) — events are pipeline items, not verified listings; the freshness convention does **not** apply to event rows (their truth is their source + approval) |

**"Where this scene talks" links** (Pillar 5 item 5, zero-UGC community layer): a link = `label`, `url`, `platform` (`discord` · `bluesky` · `instagram` · `facebook-group` · `meetup` · `website` · `other`), attached to a venue profile, organization, place, or scene tag. GD2 picks the storage shape (one table with exactly-one-parent vs. per-parent tables — GD2's call); GD1 pins the field set and the rendering rule: always outbound, always labeled with the platform, never embedded feeds.

**Slug conventions (all vocabularies + all slugged entities):** kebab-case, ASCII, stable forever once published (slugs are URLs and cross-stream references — renaming a slug needs a cross-stream ping per Contract 1; display names change freely). Entity slugs are unique per table.

# §5 The freshness convention (Contract 3 — authoritative definition)

The product's core claim. Every listable table (all of §4 except `events`) carries:

- **`status`** — enum: `open` · `closed` · `moved` · `seasonal` · `unverified`. Semantics: `closed` things **stay visible forever** (the closure graveyard is a feature: history + trust + SEO), rendered with the closed marker and, where known, a closure note ("Closed 2021 — lease lost") and successor pointer. `unverified` = listed from research but not yet human-checked (GD4's rule: every entry gets a verification pass on entry, so `unverified` should be rare and temporary). `seasonal` = alive but dormant off-season (Alki Beach Pride orgs, some leagues).
- **`last_verified_at`** — timestamptz. Set **only by a human verification act** (an admin/editor confirming the facts, or confirming a scanner diff in EV5's queue). Routine edits do not bump it; the verifier UI and admin edit form set it explicitly. Public rendering: **"Verified {Month} {Year}"** on every listing and guide page (GD3 component). Never hide a stale date — the honest date *is* the trust signal.
- **`volatility_tier`** — enum: `weekly` · `monthly` · `quarterly` · `semiannual` · `annual`. Drives EV5's re-verification cadence and the editorial calendar. Not rendered publicly.

**Tier assignments (from the research):**

| Tier | What lives there |
|---|---|
| `weekly` | recurring-event lineups (series and their instances — mostly enforced by the events pipeline itself) |
| `monthly` | venue open/closed + ownership status · **Denny Blaine/Howell rules pages while litigation is live** (the brainstorm's content-moat exemplar) · anything mid-news-cycle |
| `quarterly` | bathhouses/adult venues · rent figures (always ranged, always dated) |
| `semiannual` | health programs, income thresholds (PrEP DAP), clinic service menus |
| `annual` | organizations/leagues · clinic locations · neighborhood profiles · historical pages |

**Defaults by listing type:** venue `monthly` · venue profile `quarterly` (crowd/vibe is durable) · organization `annual` · place `monthly` while litigation is live, else `quarterly` · service listing `semiannual` (health) / `annual` (everyday) · guide page: per page, defaulting by its section (`adult` → `monthly`, `health` → `semiannual`, else `annual`) · series `monthly`.

**Date-stamping rule (voice-level, beyond the columns):** every volatile *claim inside prose* carries its own date — "as of July 2026," "2026 income limit $6,650/mo," "1BR $1,990–2,280 (8/2026, ranged)." If it can change, it gets a date; if it gets a date, EV5 can find and re-check it.

# §6 Voice & style one-pager

The house voice, distilled from the brainstorm's Voice section and Q2/Q3 decisions. Applies to every guide page, listing description, event blurb, and newsletter issue.

1. **Who's talking:** gay men writing for gay men in King County. First person plural is fine ("we like…"); institutional passive voice is not.
2. **Tone:** warm, direct, factual, sex-positive. The bathhouse page and the chorus page get the same respectful treatment — that's the spectrum promise, and it's structural (scene tags + the 18+ gate), never tonal.
3. **Zero moralizing in either direction.** No wink-wink apology for wholesome things, no edgelording about adult things. Non-monogamy, kink, sobriety, faith: all matter-of-fact.
4. **Never euphemize, never leer.** "Clothing-optional queer beach," not "notorious hotspot." State facts + etiquette + safety plainly; nothing explicit — guide-level factual keeps us far below any obscenity threshold and keeps the 18+ gate an interstitial, not an age-verification problem.
5. **Date every volatile claim** (§5). Undated superlatives ("newest," "recently") are banned — they're how The Ticket ended up four years stale.
6. **Link out with credit, don't fake depth** (Q2): sapphic/trans/QTPOC scenes get link-out cards to their own curators, named and credited. Big shared anchors are listed like everything else, labeled accurately.
7. **Minors:** this is an adult site with an 18+ layer. Content for queer youth is **curated pointers only** (Lambert House, Trevor Project, PFLAG) — route to youth specialists, never own that content (GD9 ‡ rule).
8. **Health content is directory, not advice:** what a service is, who it's for, how to start, dated — never therapy-adjacent counsel. Corrections of record (SCS closed 2022; Lifelong→Bailey-Boushay) are stated plainly with dates.
9. **Names:** venues/orgs by their real current names with former names noted ("Gay City — rename in flux, 2026"). People only with consent or public-figure context (no outing, ever — presence at an event is never reported as identity).
10. **Mechanics:** 12-hour times ("9 PM"), "2nd Saturdays" style for recurrence, neighborhood-first location lines ("Capitol Hill — 1114 Howell St"), sentence-case headings, serial comma. Titles name the thing, not the vibe ("Denny Blaine: current rules," not "Everything you need to know about…").
11. **Knotty Yoga:** listed and described under exactly these rules like any other venue/org; the relationship lives in the site-wide footer and About page (RT2), never in editorial copy.

# §7 Page templates (what renders where — GD3 builds these)

Field-by-field render contracts so GD2 payloads and GD3 components agree. Every template ends with the freshness footer: **Verified {Month Year}** (+ closed marker when `status != open`).

- **Venue profile** (`/guide/venues/{slug}`): name + neighborhood + status badge → crowd/vibe editorial (from `venue_profiles`) → practical block (address, url, price band) → scene-tag chips → "where this scene talks" link-out cards → upcoming events at this venue (client-side fetch from Stream A's public events endpoint — the Contract 2 join happens in the browser, not in Stream B's server code) → photos (platform pipeline) → freshness footer. Closed venues render the same page with the closed banner + closure note — the graveyard entry.
- **Organization profile** (`/guide/orgs/{slug}`): name + kind + scene chips → what/when/where (meeting cadence text, neighborhood) → how to join (url, low-barrier entry note) → links → freshness footer.
- **Scene landing** (`/guide/scenes/{slug}`): scene intro editorial (a `guide_pages` row, section `scenes`) → venues in this scene → organizations in this scene → recurring series in this scene (client-side fetch from Stream A) → "where this scene talks" cards. `link_out`-posture scenes render intro + link-out cards + shared anchors only.
- **Neighborhood page** (`/guide/neighborhoods/{slug}`): rundown editorial (a `guide_pages` row) → rent range + transit line (dated) → venues/orgs here → freshness footer.
- **Guide article** (`/guide/pages/{slug}` or section routes): title → body → related listings (by shared scene tags) → freshness footer. `adult`-flagged pages sit behind the 18+ interstitial.
- **Series page** (Stream A, `/events/series/{slug}`): title + cadence text + venue link → description → upcoming instances → add-to-calendar. Uses the same freshness footer component.

# §8 Listing policy — draft (feeds RT2, counsel-reviewed there)

1. **Listings are free and editorial.** Inclusion criteria: real, currently operating (or historically significant — the graveyard), relevant to gay life in King County, publicly verifiable. Nobody pays to be listed; nobody pays to rank.
2. **Money buys labeled placement, never inclusion or ranking** ("Presented by" sponsorships, contextual ads). We never rank our own business in any "best of" we publish.
3. **Founded and funded by Mason Bendixen / Knotty Yoga** — persistent site-wide footer + About page. Knotty Yoga is listed under the same rules as everyone else.
4. **Adult businesses** (bathhouses, sex-positive orgs) are listed factually within the documented-places rule (Q3b), inside the 18+ layer.
5. **Corrections:** every page carries the report path (a mailto at launch; CM1's flag button later). Verified corrections update `last_verified_at`; disputed facts get dated "as of" language.
6. **Removal:** on request by the listed party (except matters of public record, e.g. a closure), on legal requirement, or on failing the inclusion criteria at re-verification.

# Work items

*(No code — checkboxes are review/lock steps. Edit the sections inline, then check.)*

### 1. Vocabularies
- [ ] 1.1 Category list (§1) reviewed + locked (seed of 12 per Q4a; candidates list agreed as admin-add-later)
- [ ] 1.2 Scene-tag list (§2) reviewed + locked — incl. the `coverage` posture column and the three `link_out` tags (OQ2/OQ3)
- [ ] 1.3 Neighborhood list (§3) reviewed + locked — incl. accepting the draft-time addition of the `neighborhoods` vocabulary itself (OQ1)

### 2. Conventions
- [ ] 2.1 Listing types + field matrix (§4) locked — GD2 DDL and GD3 templates build from this verbatim
- [ ] 2.2 Freshness convention (§5) locked — status enum, `last_verified_at` semantics, tiers + defaults (Contract 3 becomes "as defined in GD1 §5")
- [ ] 2.3 Slug conventions locked (kebab-case, stable-forever, rename = cross-stream ping)

### 3. Editorial foundations
- [ ] 3.1 Voice & style one-pager (§6) reviewed + locked
- [ ] 3.2 Page templates (§7) reviewed + locked (GD3 consumes; Stream A consumes the series-page + freshness-footer contracts)
- [ ] 3.3 Listing-policy draft (§8) reviewed — goes to RT2 for the counsel pass as-is

### 4. Hand-off
- [ ] 4.1 Stream A owner (Levi) acknowledged: EV1 creates `categories`/`scene_tags`/`neighborhoods` from §1–§3 verbatim; series-page + freshness-footer contracts noted
- [ ] 4.2 Stream B owner (Mason) acknowledged: GD2 tables per §4, freshness columns per §5
- [ ] 4.3 Outline updated: Phase 0.2 checked off; Contract 1 annotated "values pinned in GD1; tables land in EV1 Slice 1"

# Gates

No docker/ng gates — this bucket ships a locked document, not code. **Done =** all Work Items checked; the two stream owners have read §1–§5 and §7 (the parts their code consumes); the outline's Phase 0.2 boxes are checked. Downstream enforcement: EV1's seed tests assert the §1–§3 lists verbatim, which is what makes this document binding.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **The `neighborhoods` vocabulary is a draft-time addition** — Contract 1 named only categories + scene tags, but EV2's By-Neighborhood view and GD3's neighborhood pages want one shared list, and free-text neighborhood columns would fork immediately. *Rec: accept; it rides EV1's Slice 1 with the other two tables.*
2. **Scene-tag naming calls:** `bipoc` and `qtpoc` as separate tags (gay-BIPOC scene coverage vs. QTPOC-self-described orgs we link out to), `leather` covering kink, `furry` seeded from day one. *Rec: as drafted; all admin-renamable later except slugs.*
3. **`coverage` posture as a column** (`full`/`link_out`) so GD3 renders link-out scene landings automatically, rather than keeping posture as editorial memory. *Rec: yes — it's the Q2 decision made structural.*
4. **Adult flag name:** `adult` boolean on venue_profiles/places/guide_pages driving the 18+ interstitial (§4). Alternative: an `audience` enum (`all`/`adult`) if we ever foresee a third value. *Rec: boolean `adult`; simplest thing that implements Q3(b).*
5. **Series freshness default `monthly`** (not `weekly`): weekly-tier truth for lineups is enforced by the events pipeline itself; the series *entity* (still running? still 2nd Saturdays?) drifts slower. *Rec: as drafted.*
6. **Categories on guide entities:** categories stay events-only for now (orgs have `org_kind`, services have `service_kind`); the `categories` table is generic so the guide can adopt it later without a rename. *Rec: as drafted — avoids a premature everything-taxonomy.*
