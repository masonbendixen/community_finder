---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1
tags:
---

# Bucket EV6 — Structured feed import

**Track:** EV — Events engine · **Size:** M · **Stream:** A (owner: Levi)
**Needs:**
- EV1 (done): the ingest door, `event_sources`, and the (source, external key) upsert.
- **EV4** for the schedule (its reserved `import_event_feeds` job) and the scheduler service account. Until EV4 lands, imports run by hand through the test_helper (§7), so EV6 can be built in either order.
- **One small honuware change** for safe fetching (§1). An app-side fallback exists (OQ2).

**Runs before EV5.** EV6 handles every source that publishes machine-readable events; EV5's AI scanner then only has to handle the rest.

**Feeds:**
- **EV5:** it reuses this bucket's `scan_runs` audit table, the fetcher, the robots.txt check, the RSS reader (for its editorial monitors) and the "skip sources that have a feed" rule.
- **GD4a:** the source registry gets a feed column per source, filled from this doc's §Findings.
- **M1/M3:** "EV2 with 2+ weeks of real events" arrives here without hand entry.
- **RT5:** the permission asks in §Findings join the venue outreach kit.

# Context

The outline's EV5 is an AI scanner for event content. Mason's hunch (11/1): **some of the bars publish RSS or other feeds with event information, which a plain importer could read without AI.** On 11/1 three research passes checked about 80 sources by hand with curl:
- the gay bars, clubs and adult venues;
- the mainstream venues, arts spaces and climbing gyms;
- the organizations, leagues, platforms (Meetup, Eventbrite, Evvnt), aggregators and news RSS.

The raw captures are in the session scratchpad. Their findings are below.

**The short answer: the hunch is right, but RSS mostly isn't the feed.**
- **RSS is everywhere and nearly useless for events.** Every Squarespace and WordPress site has an RSS feed, but its dates are *publish* dates, not event dates. RSS earns its keep in two other ways:
	- finding new event pages on Squarespace sites, which is the robots.txt-friendly path (§Findings);
	- **EV5's closure monitors**, the news feeds.
- **The real event data comes in three shapes:**
	- **Squarespace's events collection:** four bars plus several orgs;
	- **calendar (ICS) feeds:** Google Calendar, the WordPress "The Events Calendar" plugin, Teamup;
	- **schema.org `Event` markup on event pages:** ticketing pages and Squarespace event pages.
- **A small importer that reads those three shapes covers about fifteen sources today**, including the richest one found (Pan Eros: 2,401 calendar entries) and the best sports feed (Rain City Soccer's five Google calendars).
- **The catch is permission.**
	- Squarespace's default robots.txt blocks its own JSON and ICS endpoints for every crawler.
	- Meetup, Eventbrite and Do206 forbid automated extraction in their terms.
	- `calendar.google.com`'s robots.txt is `Disallow: /`, even for the public calendar links Google offers for subscribing from any calendar app.

	EV6 honors robots.txt for every page it reads, and fetches a subscription calendar the way a calendar app does (pin-down 11, OQ1). It also lists who to ask.

**Why a deterministic importer before the AI scanner:** a feed parse is exact, cheap, testable with fixed fixtures, and needs no model, no prompt and no per-run cost. Every source it covers is one EV5 never has to scan, and the scanner's guesses are reserved for the sources that need judgment.

# Findings

*Research date 11/1/2026. "Today" counts are what each feed held that day.*

### Gay bars, clubs and adult venues (22 checked)

| Source                                                                                               | What it publishes                                                                                                                                                            | EV6 format    | Usable today?                                                                                                                                                                                                 |
| ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Kremwerk + Timbre Room + Cherry** (one site, `kremwerk.com/upcoming`)                              | Squarespace events collection: 25 upcoming, about 9 weeks ahead. Each event names its room. Per-event ICS works; collection ICS doesn't                                      | `squarespace` | **Yes**, through RSS plus event pages; the JSON needs permission                                                                                                                                              |
| **The Cuff Complex** (`cuffcomplex.com/events`)                                                      | Squarespace collection: 10 upcoming, to 12/19                                                                                                                                | `squarespace` | **Yes**, same path                                                                                                                                                                                            |
| **Queer/Bar** (`thequeerbar.com/events`)                                                             | Squarespace collection: 15 upcoming. Its Ticket Tailor pages carry event markup                                                                                              | `squarespace` | **Yes**, same path. Same owners as the Cuff                                                                                                                                                                   |
| **Unicorn** (`unicornseattle.com/events`)                                                            | Squarespace collection: exactly 30 upcoming, which is Squarespace's page size, so there may be more                                                                          | `squarespace` | **Yes**, same path                                                                                                                                                                                            |
| **Pan Eros Foundation / SEAF / Gallery Erato**                                                       | Public Google Calendar: 5.4 MB, 2,401 entries going back years, 471 recurrence rules, 568 per-date overrides, mixed time formats. Includes non-public items ("Retail Hours") | `ics`         | **Technically yes.** The richest feed found, and the reason the recurrence expander must be real. Needs a filter, and **most of it is adult and kink education**, so it waits on an 18+ flag for events (OQ9) |
| **Club Z** (`thezclub.com`; clubzseattle.com is parked)                                              | Public Google Calendar embedded on its events page: valid, 0 events ("coming soon")                                                                                          | `ics`         | **Subscribe now.** It costs nothing until they post                                                                                                                                                           |
| **Le Faux at Julia's** (SimpleTix)                                                                   | Event markup on each SimpleTix show page, one entry per date. All dates share one URL                                                                                        | `json_ld`     | **Yes**, one source per show page                                                                                                                                                                             |
| **Massive** (`massive.club/calendar`)                                                                | Event markup with **dates but no times** (`"Nov 02, 2026"`). Tixr blocks automated reads                                                                                     | `json_ld`     | **No**: date-only entries are skipped (pin-down 7). Hand entry or EV5                                                                                                                                         |
| **CC's Seattle** (renamed from CC Attle's; `ccsseattle.com`)                                         | Wix CMS records embedded in the calendar page as JSON: 29 events, about 4 weeks                                                                                              | none (OQ6)    | Not by EV6. Undocumented and brittle. **The rendered page shows times 8 hours off; only the embedded data is right**                                                                                          |
| Pony, Seattle Eagle, Wildrose, Union, Crescent Lounge, Boombox, Lumber Yard, Steamworks, Madison Pub | No machine-readable events. Instagram, posters, or prose ("1st Fridays BEEFCAKE")                                                                                            | none          | No. Prose recurring nights become hand-entered series (GD4 §4.8)                                                                                                                                              |

**Dead, broken or hijacked domains (fix in GD4 before anything links them):**
- **`neighboursnightclub.com` now redirects to a Vietnamese football-betting site. Never link it.**
- `dieselseattle.com` shows a hosting error page.
- `changesinwallingford.com` no longer resolves.
- `ponyseattle.com`'s certificate expired on 11/1, so normal browsers refuse it. It may be renewed.
- `ccattles.net` is superseded by `ccsseattle.com`.
- `clubzseattle.com` is parked; the real site is `thezclub.com`.
- `seattleeagle.com` works over plain HTTP only.

Doghouse Leathers' RSS is new products, not events.

### Organizations, leagues and platforms

| Source | What it publishes | EV6 format | Usable today? |
|---|---|---|---|
| **PFLAG Seattle** | The Events Calendar (WordPress): ICS and REST, 11 upcoming, one entry per date | `ics` | **Yes.** The best org feed |
| **Rainbow City Performing Arts** (`rainbowcity.org`) | Same plugin: 2 concerts. Its Teamup calendar is mostly rehearsals | `ics` | **Yes** for the concerts feed; skip Teamup |
| **Rain City Soccer** | Five public Google Calendars linked from its own calendar page. The Kickabouts calendar repeats weekly through 12/29 by recurrence rule; others are monthly clinics | `ics` | **Yes.** The best sports feed. The club links the feeds itself, so subscribing is intended |
| **Entre Hermanos, ECSA (softball)** | The Events Calendar, 0 upcoming (off-season). **An empty feed comes back as a 0-byte HTML page, not an empty calendar** | `ics` | **Yes**, seasonal. The importer must read "empty" as no events, not an error |
| **TRACTION** (`tractionpnw.org`, trans community calendar) | The Events Calendar: 280 upcoming, but the list feed returns only 30; the month view returns the rest one month at a time | `ics` with a `{month}` URL | **Ask first.** It's a curated community calendar, effectively an aggregator |
| **PNW Black Pride, Three Dollar Bill Cinema, Queer Social Club** | Squarespace collections. Three Dollar Bill's event markup has a wrong UTC offset but the right instant. Queer Social Club holds 111 events | `squarespace` | **Yes** for PNW Black Pride and Three Dollar Bill. **Ask first** for Queer Social Club (its list is its curation) |
| **Frontrunners** | Weekly runs in prose only. The annual race is on RunSignup's keyless API, but the 2027 date isn't set | none | Hand-entered series. One race a year isn't worth an adapter |
| **Meetup** | Per-group iCal now needs a signed token (403 for everyone else). The API needs Meetup Pro. **The terms forbid automated extraction** | none | **No** (OQ5). Two of our three test groups no longer exist |
| **Eventbrite** | Event markup on pages, but **§13 of the terms bans automated extraction**. The API needs our own OAuth token | none | **No** for pages; the API is a later option (OQ5) |
| **Evvnt** (publisher program) | A keyed JSON API with stable ids and incremental sync, but no city or radius filter, and terms forbid modifying content | none | Later: RT5's publisher signup decides (OQ5) |
| **LeagueApps** (OutLoud, Queer City Sports, ECSA registration) | A per-league API key, issued by each league's admin | none | **Ask each league** for its key. It's read-only and meant for this |
| **Do206** | An upcoming-events sitemap plus event markup, but **the terms ban scraping** | none | **No**, without a partnership |
| **Seattle Quake** | A Google Calendar hidden inside a Wix widget | `ics` once known | **One email** asking for its public calendar address |
| **Pride Sports Seattle** | 403 to everything, with a header reserving text-and-data-mining rights | none | **Ask first** |
| Seattle Pride, the choruses, Gay City, GenPride, Lambert House, Ingersoll, GSBA, ORCA | No feed. Pictures of calendars, prose, a Cloudflare challenge (GSBA) or a never-configured calendar (ORCA) | none | Hand entry, EV5, or ask (GSBA's member calendar can export) |

**Gone (remove from GD4's lists):**
- **The Gay Agenda** has shut down; every URL redirects to a "has been sunset" page.
- **Different Spokes** has no Seattle presence.
- **Team Seattle**'s site was last updated around 2018.

### Aggregators and news (for EV5, not imported here)

- **News RSS for closure monitoring works:**
	- **CHS** (`capitolhillseattle.com/feed/`) holds only about 2 days, so poll every few hours, or use its food-and-drink category feed;
	- **SGN** (`sgn.org/rss/87/News`): non-standard dates, so the RSS reader must be lenient;
	- **Seattle Gay Scene** (`seattlegayscene.com/feed/`): its weekly "where to go" posts are a discovery signal too;
	- **The Stranger**: per-category feeds (`/category/queer/feed/`).
- **EverOut's robots.txt no longer blocks crawlers**, so the brainstorm's note is out of date as far as robots goes. Its terms weren't checked, and it's The Stranger's calendar: ask.
- **Out x Out** (`outxout.com`) has event markup for the recurring nights of the Instagram-only bars. It's an aggregator, so ask before using it.
- **Sapphie Taffy** posts monthly prose roundups. RSS spots a new post; the events inside need EV5, and permission.

### Mainstream venues with queer nights, arts spaces and climbing gyms (25 checked)

| Source | What it publishes | EV6 format | Usable today? |
|---|---|---|---|
| **Gallery Erato** | The same Pan Eros Google Calendar as above (10 future entries located at the gallery) | `ics` | One source with Pan Eros, not two. **Mostly adult and kink education**, so it waits on an 18+ flag for events (OQ9) |
| **Rendezvous / Jewelbox Theater** (`rendezvous.squarespace.com/events`) | Squarespace collection: 19 upcoming. General venue; the drag brunch and a queer holiday show are in it | `squarespace` with an include filter | **Yes** |
| **El Sueñito Brewing** (Wix) | An event-pages sitemap (720 URLs, past included, each with a last-modified date). Every event page has event markup and a per-event ICS. Some events are at its Bellingham taproom | `json_ld_list` over the sitemap, excluding `Bellingham` | **Yes**: Queer Book Club, drag brunch and bingo |
| **Stoup Brewing (Capitol Hill)** | An undocumented JSON endpoint behind its events page: 650 entries about three years ahead, **list entries without a year** | none (OQ6) | No adapter. Board Gayme Night is a stable weekly: a hand-entered series |
| **Phoenix Comics & Games** (Shopify "Calee" app) | Undocumented app JSON: 47 events, recurrence as vendor fields. **Its `start_date_time` is local time with a misleading `Z`**; only `start_date_time_utc` is right | none (OQ6) | No adapter. Queer Geek and XL Bears meetups: hand entry or EV5 |
| **Chop Suey** | Event RSS and a sitemap without show dates. Each event page lists every date as plain HTML, with no event markup | none | No. Flammable Sundays is a stable weekly: a hand-entered series |
| **ArtsWest** | Event markup on show pages with **floating local times** (no offset) | `json_ld` per show page | Possible. No recurring queer night found |
| **Seattle Bouldering Project** (via SweatPals) | Event markup on each SweatPals event page; the host page lists only 6 upcoming links | `json_ld` per page | Partly. See Queer Mountaineers below |
| **The other climbing gyms** (Edgeworks, Half Moon, Castle, Climb NORA, Uplift, Vertical World North) | Booking systems (RedPoint HQ, Approach, RockGymPro). The queer nights appear only as prose or programs. VW North's iCal link answers "Invalid guid" | none | No. **Every gym's queer night is run by Queer Mountaineers**, so the organizer is the source (§8.2) |
| Skylark Cafe, Southgate Roller Rink, Rough & Tumble, Raygun Lounge, Jet City Improv, Intiman, Métier, West Seattle Bowl | HTML only, an empty or stale feed, or ticketing systems with no feed (Salesforce PatronTicket, TicketWeb) | none | No. Southgate's Pride Skate (Wed 21+) is a stable weekly: a hand-entered series |
| **Charlie's Queer Books** | A session-based private API behind its React storefront | none | Ask them |

**Dead or changed:**
- **Great American Casino Tukwila closed on 11/1/2026** (Maverick Gaming), and **its domain now redirects to an unrelated online-casino affiliate site**. Drop the Royal Flush drag show unless it has moved, and never link the old domain.
- **`comettavern.com` is parked for sale.** The Comet appears to be Instagram-only, and **"Caliente Thursdays" wasn't found in any source**.
- Southgate's site works over plain HTTP only. Intiman serves an empty page to non-browsers.

**Climbing-night facts that differ from GD4's rotation:**
- Half Moon's queer climb club is **every Tuesday**, with Queer Mountaineers on the **4th Tuesday**; GD4 says 4th Monday.
- **Edgeworks' Queer Climb Night (2nd Wednesday) is missing** from GD4's list.
- Climb NORA is in Federal Way.
- Vertical World North's site lists no queer night, and Uplift's page shows only a stale date.
- "Castle" is Castle Climbing Club in West Seattle, open since 1/1/2026.

### What the findings mean for the design

1. **Two readers, with three ways to find the pages, cover everything usable:**
	- **`ics`** reads whole calendars: Google Calendar, The Events Calendar, Teamup.
	- **Event markup (JSON-LD) on event pages** is the other reader. The page list comes from:
		- one page itself (`json_ld`: SimpleTix, Ticket Tailor, ArtsWest);
		- an RSS feed or sitemap whose links are event pages (`json_ld_list`: El Sueñito's Wix sitemap);
		- Squarespace's RSS (`squarespace`, the same path with Squarespace's own key rule: four bars, Rendezvous and the Squarespace orgs).

	The undocumented one-venue backends (CC's Wix data, Stoup, Phoenix's Calee app) don't justify adapters (OQ6).
2. **Recurrence is unavoidable:**
	- the Google feeds (Pan Eros, Rain City, Club Z) state recurrence as rules, with per-date overrides and exclusions;
	- everything else lists each date separately.
3. **Times arrive in every form:**
	- UTC;
	- a named zone;
	- floating local time;
	- all-day;
	- offsets with and without a colon;
	- one wrong offset on a right instant.

	The rule: **parse every value to an instant, store UTC, and display in Pacific.**
4. **Stable ids are platform-specific:**
	- Google UIDs are stable, but a recurring series shares one;
	- The Events Calendar's ICS UIDs embed the start time, so they change when an event is rescheduled;
	- Squarespace's 24-character item id is the same in its JSON, its per-event ICS and its RSS guid;
	- ticketing pages share one URL across dates.

	Each needs its own key rule (pin-down 5).
5. **Robots and terms decide more than technology does** (OQ1, OQ5). Cloudflare-managed robots files are spreading (Edgeworks, ma.to). They block AI crawlers by name (ClaudeBot, GPTBot) and signal `ai-train=no`. EV6's honest, non-AI user agent is matched on `*`, but **EV5 inherits the question of how an AI scanner identifies itself**.
6. **The organizer often beats the venue.** Queer Mountaineers runs the queer night at every climbing gym, so one organizer source replaces seven booking systems. **Many of the best-known nights are stable rituals** (Board Gayme Night, Pride Skate, Flammable Sundays, the climb nights). A hand-entered series (GD4 §4.8) is cheaper than any importer for those.

# Design pin-downs

1. **Deterministic only.** EV6 parses formats; it never guesses. Anything that needs judgment (prose, posters, a page without markup) is EV5's. A source is EV6's when its new **`feed_format`** column is set, whatever its `kind`. `kind` says what the source *is*; `feed_format` says how to *read* it. EV5 skips any source with a `feed_format`.
2. **One door in.** Every imported item goes through EV1's `EventHelper::IngestEvents`, so EV6 gets the same upsert key, the same minor/major change rules and the same outcomes as EV5. New events land **pending** unless the source is trusted (pin-down 10).
3. **Never hold a transaction across the network.**
	1. Fetch and parse first, outside any transaction.
	2. Then, per source, open one short transaction for the ingest, the `last_scanned_at_us` touch and the `scan_runs` row.

	One bad source never rolls back another.
4. **The import window is now − 1 day to now + 90 days.** Recurrence is expanded only inside it. Entries outside are ignored, so Pan Eros's years of history cost parse time but write nothing.
5. **External keys, by format:**
	- **ICS, one-off:** the UID. For The Events Calendar (its `PRODID` names the plugin), use only the UID's leading id, the part before the first `-`, because the rest is the start and end time and changes on a reschedule.
	- **ICS, one date of a recurring series:** `UID|<that date's original start>`. The original start comes from the override's `RECURRENCE-ID` when there is one, so a moved date keeps its key.
	- **Squarespace:** the 24-character item id, the same in both modes. A source switching from the RSS path to JSON after permission keeps its keys.
	- **JSON-LD (`json_ld`, `json_ld_list`):** the markup's `@id` when present; otherwise `url|startDate`, because ticketing pages share one URL across dates.
6. **Time rules:**
	- **Every value becomes a UTC instant.** `Z` is UTC. A numeric offset (`-08:00`, `-0800`, `+00:00`) is applied as given.
	- **Floating local time, any `TZID`, and an ICS file's `X-WR-TIMEZONE` all read as Pacific**, through `Events::kEventsTimeZone`. A feed in another zone isn't a King County feed. An unrecognized `TZID` is logged in the run's details, not guessed.
	- **Recurrence expands in local wall-clock time** (honuware's `CalendarDateWallClockToUs` / `LocalDateTokenUs`), so a weekly 7 pm stays 7 pm across a DST change. *11/1/2026 is itself the fall-back day, which the tests use.*
7. **Skipped, never guessed, and counted per reason in the run's details:**
	- **all-day and date-only entries** (no start time to show);
	- **recurrence rules using parts the expander doesn't support**;
	- entries without a title or start;
	- entries outside the window;
	- entries the source's text filter excludes.
8. **Cancellations.** An entry the feed marks cancelled (ICS `STATUS:CANCELLED`, JSON-LD `EventCancelled`, or a cancelled override of one date) is sent with a new **`cancelled`** flag on `IngestItem`:
	- an approved or pending event goes to **pending** with the review note "Cancelled at the source", which takes it off the public site;
	- a cancelled entry that's new is not created;
	- rejected and archived events stay untouched, as always.
9. **Events that vanish from a feed are reported, not acted on.** After a complete fetch, any future event of the source whose key wasn't seen is listed in the run's details as "missing from the feed." Feeds are often partial (Squarespace's 30-item page, The Events Calendar's 30-item list), so a curator decides (OQ4).
10. **Create-time defaults live on the source, and apply to every ingest door** (EV5 too), only when the item doesn't state them:
	- **Venue:** the item's location name is matched to a venue by exact, case-insensitive name. Without a match, the source's own `venue_id` is used, else no venue, with the text kept as `location_text`. This is how Kremwerk's one feed puts each event in the right one of its three rooms.
	- **Categories and scene tags:** a new event with none stated takes the source's `default_category_slugs` / `default_scene_tag_slugs`. Unknown slugs are dropped and reported, as EV1 already does.
	- **Series:** a new event with no stated series joins the series *at its venue* whose normalized title matches. Normalizing lowercases, drops punctuation and emoji, and collapses spaces, so "Karaoke Every Monday!!! " finds "Karaoke every Monday."
	- **Trusted sources:** a new event from a source with `auto_approve` lands **approved**, with the review note "Auto-approved: trusted feed" and no reviewer. A major change keeps it approved. Default off (OQ3).

	Because these apply only at creation, a curator's later edit to an event's categories or series survives every re-import.
11. **Polite fetching:**
	- **robots.txt is honored for every page EV6 reads**: RSS feeds, sitemaps and event pages, in the `squarespace`, `json_ld` and `json_ld_list` formats (OQ1). It's read once per host per run and matched on our product token and `*`, with `Allow`/`Disallow` longest-match and the `*` and `$` wildcards. A `Crawl-delay` is honored.
	- **A subscription calendar (`ics`) is fetched the way a calendar app fetches it, with no robots check.** These are links publishers offer precisely so calendar apps can subscribe: Google's "public address in iCal format", The Events Calendar's export, a club's own "subscribe" link. `calendar.google.com`'s robots.txt is `Disallow: /`, which is aimed at crawlers of its web pages. Honoring it would refuse every Google calendar, including the ones Rain City links on its own page for subscribing. **The admin rule that goes with it:** an `ics` source is a whole-calendar subscription link the publisher offers. Never use one-event export links, such as Squarespace's per-event `?format=ical`, which its robots.txt disallows.
	- **A source with a recorded `feed_permission_note`** (who said yes, and when) skips the robots check. That's the only other override.
	- **Limits:**
		- one request per second per host;
		- at most 50 requests per source per run;
		- a 20-second timeout per request;
		- a 10 MB response cap;
		- compressed transfer accepted.
	- **Conditional GET:** the feed's `ETag` / `Last-Modified` are stored on the source and sent back as `If-None-Match` / `If-Modified-Since`. A 304 is a cheap "not modified" run.
	- **The User-Agent** names us (`CommunityFinderFeeds/1.0`, renamed when the brand is final) with a contact URL from the app's secret defaults.
12. **The URL guard (server-side request forgery).** Feed URLs are admin-entered, but the server fetches them, so a bad URL must not reach the server's own network:
	- `http`/`https` only;
	- no private, loopback, link-local or cloud-metadata addresses (`10/8`, `172.16/12`, `192.168/16`, `127/8`, `169.254/16`, `100.64/10`, `::1`, `fc00::/7`, `fe80::/10` and their IPv4-mapped forms), **checked against the address actually connected to**, which closes DNS rebinding. That's the honuware part (§1);
	- redirects are followed by EV6 itself, at most 3 hops, each re-checked.
13. **Budgeted runs.** One import run (one job tick) handles due sources until **60 seconds** have passed, then returns; the rest are due on the next tick.
	- **A feed source is due** once its `last_scanned_at_us` is more than 20 hours old, so feeds import daily whatever their volatility tier. A feed is one cheap request, and the tier is EV5's cost control.
	- **The list paths (`squarespace`, `json_ld_list`) cost one request per event page.** So they read new links daily, re-read known future events on the source's tier, and skip links that are clearly past. A Squarespace event-date path or a sitemap's old `lastmod` is enough to tell, with no fetch. A sitemap is walked newest-`lastmod` first, so El Sueñito's 720 URLs don't eat the request cap.

# Scope

**In:**
- the honuware fetch options (§1);
- the `event_sources` feed columns and the shared `scan_runs` table (§2);
- their helpers (§3);
- the readers (ICS, event markup, RSS/Atom/sitemap lists, Squarespace), the robots check, the fetcher, the importer and the four create-time ingest rules (§4);
- one admin endpoint (§5);
- the EV4 job wiring and a failure alert (§6);
- test_helper commands (§7);
- turning on the feeds the Findings found (§8).

**Out:**
- **the AI scanner and the verifier (EV5)**, including the closure-news monitors. The RSS reader is built here and reused there.
- **an 18+ flag for events** (OQ9: a separate small Stream A follow-up);
- **Meetup, Eventbrite, Evvnt, LeagueApps and Do206** (OQ5);
- adapters for one venue's undocumented data (CC's, Stoup, Phoenix; OQ6);
- all-day events (pin-down 7);
- acting on vanished events (pin-down 9);
- outbound feeds (EV2 has them);
- a client page for runs. Admin CRUD shows `scan_runs` read-only, and the review queue already shows each event's source.

# Work items

### 1. honuware: safe-fetch options on `HttpRequest` (proposed ledger item **H14**, after EV4's H13)

- [ ] 1.1 Add optional fields to `Http::HttpRequest` (`foundation/util/http/http_client.h`). **The defaults keep today's behavior**, so the scheduler, knottyyoga and every current caller are unaffected, and knottyyoga needs no re-pin.
	- `timeoutSeconds` (0 = none) → `CURLOPT_TIMEOUT`, plus a connect timeout;
	- `maxResponseBytes` (0 = unlimited): the write callback aborts past it;
	- `acceptCompressed` → `CURLOPT_ACCEPT_ENCODING ""`;
	- `webOnly` → `CURLOPT_PROTOCOLS_STR "http,https"` plus a `CURLOPT_PREREQFUNCTION` check that refuses a connection to a private, loopback, link-local or metadata address (curl ≥ 7.80; the pin has 8.21).
- [ ] 1.2 Redirects stay unfollowed (curl's default), so the caller can re-check each hop.
- [ ] 1.3 **Tests** in honuware: each option's curl mapping. A live `127.0.0.1` request with `webOnly` is refused (the existing local-Crow test pattern in `http_client_test.cpp`). An over-size body fails with a clear error. A timeout fires.
- [ ] 1.4 **The cross-repo bump**, per CLAUDE.md: Mason pushes honuware → Claude re-pins and verifies a fresh-clone build → Mason commits. *EV4 §1 (H13) is also a honuware change; doing both in one bump saves a cycle.*

### 2. db_schema

- [ ] 2.1 **`event_sources` gains** (constants in `db_schema/event_sources.h`):

	| Column | Type | Default | Meaning |
	|---|---|---|---|
	| `feed_format` | TEXT NULL, CHECK in (`ics`, `squarespace`, `squarespace_json`, `json_ld`, `json_ld_list`) | NULL | How EV6 reads the source; NULL = not EV6's. `squarespace_json` is the robots-disallowed JSON endpoint, so it's for permission holders only |
	| `feed_permission_note` | TEXT | `''` | Who granted permission to read past robots.txt, and when. Non-empty is the robots override for page formats |
	| `include_text` | TEXT | `''` | `\|`-separated, case-insensitive substrings matched against the title **and** location; empty keeps all (Rendezvous: `drag\|queer\|gay`) |
	| `exclude_text` | TEXT | `''` | The same, applied after include (Pan Eros: `Retail Hours`; El Sueñito: `Bellingham`) |
	| `default_category_slugs` | TEXT | `''` | Comma-separated, for new events that state none |
	| `default_scene_tag_slugs` | TEXT | `''` | The same |
	| `auto_approve` | BOOLEAN | false | Trusted source (pin-down 10, OQ3) |
	| `feed_etag`, `feed_last_modified` | TEXT | `''` | Conditional-GET validators, written by the importer |

	**The URL conventions** (in the column comment and the admin label):
	- `ics`: the `.ics` URL, which may contain `{month}`. The importer fetches one URL per month of the window, as `YYYY-MM`.
	- `squarespace` / `squarespace_json`: the collection URL, without `?format=`.
	- `json_ld`: the page URL.
	- `json_ld_list`: an RSS/Atom feed or sitemap whose links are event pages.
- [ ] 2.2 **New table `scan_runs`** (`db_schema/scan_runs.{h,cpp}`), shared with EV5, in `kEventsDomainTables` after `event_sources`:
	- `id`;
	- `source_id`, FK → `event_sources`, ON DELETE CASCADE per the house rule;
	- `kind`, CHECK in (`feed_import`, `ai_scan`, `verify`). EV5's two kinds are defined now so EV5 needs no CHECK change on a live database;
	- `started_us`, `finished_us`;
	- `outcome`, CHECK in (`ok`, `not_modified`, `fetch_failed`, `robots_disallowed`, `parse_failed`, `partial`);
	- `http_status` (nullable);
	- `items_seen`, `created_count`, `updated_count`, `unchanged_count`, `pending_again_count`, `skipped_count`;
	- `error_text`;
	- `details`: a JSON text of skip reasons with counts, ignored slugs, unrecognized TZIDs and missing keys;
	- an index on (`source_id`, `started_us`).
- [ ] 2.3 **`create_app_tables.cpp`:** the three places (`MakeAppTables`, `AppTableCreationOrder`, `PopulateAppTables`), plus admin column labels and admin enums for `feed_format`, `kind` and `outcome` (GD4 §1's pattern). `scan_runs` gets read-only admin table permissions for Administrator.
- [ ] 2.4 **Tests:** the CHECKs reject an unknown format, kind and outcome; deleting a source cascades its runs; GD4 §1's every-CHECK-has-an-enum test covers the new columns. Then the live `--recreate_database` check (Mason runs it, per the destructive-op rule).

### 3. table_helpers

- [ ] 3.1 `EventSources` gains:
	- `ListDueFeedSources(nowUs, staleBeforeUs, limit)`: enabled, `feed_format` set, URL non-empty, and never scanned or scanned before `staleBeforeUs`, oldest first. **It skips URL-less sources**, such as the sample source from EV2 OQ11 (d);
	- `UpdateFeedValidators(id, etag, lastModified)`.

	**Tests:** due/not-due/disabled/URL-less/never-scanned ordering.
- [ ] 3.2 `ScanRuns` (`table_helpers/scan_runs.{h,cpp}`):
	- `Insert`;
	- `ListForSource(sourceId, limit)`, newest first;
	- `CountTrailingFailures(sourceId)`: consecutive non-`ok`/`not_modified` runs, newest first;
	- `DeleteOlderThan(cutoffUs)`.

	**Tests** for each, including a failure streak broken by a `not_modified` run.

### 4. business_logic (`business_logic/feeds/`)

**The parsers are pure functions** (text in, items and skip reasons out) with no database and no network. They're tested with small inline fixtures shaped like the 11/1 captures but with **generic placeholder names and titles**, per the house rule on test data.

- [ ] 4.1 **`ics_parser`** → `ParsedFeed { items, skips, warnings }`:
	- **Lines:** unfolding (CRLF plus a space or tab; bare LF tolerated); property parameters (`TZID`, `VALUE=DATE`); text unescaping (`\,` `\;` `\n` `\\`).
	- **Components:** `VEVENT` only; nested `VALARM` ignored.
	- **Times:** `DTSTART`, plus `DTEND` or `DURATION`.
	- **Text fields:**
		- `SUMMARY` → title, with HTML entities decoded (Squarespace's ICS keeps `&amp;`);
		- `DESCRIPTION` → plain text: tags stripped, entities decoded, capped at 4,000 characters;
		- `URL`;
		- `LOCATION` → location name;
		- `STATUS:CANCELLED`.
	- **An empty body, or a 0-byte HTML page, is an empty calendar** (The Events Calendar's off-season answer), not an error.
	- **Recurrence:**
		- **`RRULE`** with `FREQ` of DAILY, WEEKLY, MONTHLY or YEARLY, plus `INTERVAL`, `COUNT`, `UNTIL`, `BYDAY` (with ordinals in a monthly rule: `1SA`, `-1FR`), `BYMONTHDAY` and `BYMONTH`. Any other rule part skips that series, with the part named;
		- **`EXDATE`**, including comma lists and `TZID`;
		- **`RECURRENCE-ID` overrides**, which replace their date's occurrence, and a cancelled override removes it;
		- **`RDATE`** single dates.

		Expansion is in local wall-clock time and limited to the window.
	- **Tests:**
		- folding and unescaping;
		- each time form (`Z`, `TZID`, floating, `VALUE=DATE` skipped);
		- a weekly series crossing **11/1/2026's fall-back**, which keeps 7 pm local on both sides;
		- "first Saturday" monthly;
		- "last Friday";
		- `COUNT` vs `UNTIL`;
		- an `EXDATE` removed;
		- a moved override keeping its original-date key;
		- a cancelled override;
		- an unsupported `BYSETPOS` skipped with its reason;
		- the window bounds;
		- The Events Calendar's UID rule;
		- an empty-HTML body;
		- a 5,000-entry synthetic calendar parsed under a time bound.
- [ ] 4.2 **`json_ld_extractor`**:
	- finds every `<script type="application/ld+json">` block and parses it with `Json::Value`;
	- walks objects, arrays and `@graph`;
	- **keeps any `@type` ending in `Event`** (`MusicEvent`, `SocialEvent`, `ComedyEvent`, …);
	- maps `name`, `startDate`, `endDate`, `url`, `description`, `location.name`, `eventStatus` (`EventCancelled`) and `@id`;
	- **dates:** ISO with `Z`, `±HH:MM` or `±HHMM`, or no offset (→ Pacific); date-only and non-ISO values are skipped.

	**Tests:**
	- several Events in one page;
	- `@graph`;
	- an array of occurrences sharing a URL (keys differ by start);
	- `-0800` without a colon;
	- a wrong offset on a right instant, which keeps the instant;
	- `+00:00`;
	- `"Nov 02, 2026"` skipped;
	- a malformed block skipped while the others still parse.
- [ ] 4.3 **`link_list_reader`**: RSS 2.0, Atom and sitemaps (`<urlset>`), minimal. There's no XML library in the dependencies, so it's a small tag scanner that handles entities and CDATA.
	- **RSS and Atom:** `guid` / `id`, `link`, `title`, and a **lenient** `pubDate` that accepts RFC 822, ISO and the JavaScript `Date.toString()` form SGN emits.
	- **Sitemaps:** `loc` and `lastmod`.
	- Built here for the list paths and **reused by EV5's monitors**.

	**Tests:** RSS, Atom, a sitemap, SGN's date, a CDATA title, a missing guid falling back to the link.
- [ ] 4.4 **`page_list_adapter`** (`squarespace`, `json_ld_list`) and **`squarespace_json`**:
	- **The list paths:**
		1. read the list (4.3). For `squarespace`, that's `<collection>?format=rss`;
		2. drop links that are clearly past: a Squarespace event-date path such as `/2026/11/06/`, or a sitemap `lastmod` older than the window. Walk a sitemap newest first;
		3. fetch each remaining page and run 4.2 on it.

		**`squarespace` keys** on the item id from the guid (`site:collection:item`). `json_ld_list` keys per pin-down 5.
	- **`squarespace_json`:** one read of `?format=json`, mapping `upcoming[]`'s epoch-millisecond times, `title`, `fullUrl`, `excerpt` and `location.addressTitle` (the room).
	- **Both Squarespace modes** trim and collapse titles (Unicorn's trailing spaces), and record a warning when the list holds exactly 30 items (a possibly capped list).

	**Tests:**
	- each path from fixtures;
	- the same Squarespace item gives the same key in both modes;
	- the past-link skip for both kinds of list;
	- the request cap on a long sitemap;
	- the 30-item warning.
- [ ] 4.5 **`robots_policy`**: parse, group selection (our token, else `*`), longest-match `Allow`/`Disallow` with `*` and `$`, and `Crawl-delay`. A missing robots.txt (404) allows all; a 5xx disallows for this run.

	**Tests:** Squarespace's default file blocks `?format=json` and allows `?format=rss`; TRACTION's `/wp-json/` disallow; an empty `Disallow:`; the wildcard and `$` cases. **Plus the fetcher's rule (4.6):** an `ics` source is fetched even when its host's robots.txt is `Disallow: /`, as Google's is, and a page format on that same host is refused.
- [ ] 4.6 **`feed_fetcher`** (wraps `Http::HttpClientPtr`, so tests use honuware's `Http::Test::TestHttpClient` from `honuware_testing`):
	- the URL guard;
	- robots, cached per host per run;
	- the per-host one-per-second pacing (with an injectable clock and sleeper for tests);
	- conditional-GET headers;
	- following a 3xx up to 3 hops, each guarded;
	- the per-source request cap;
	- the honuware options from §1.

	**Tests:** every refusal (scheme, a private IP literal, a redirect to a private IP, robots); 304 handling; the hop limit; the request cap; the `If-None-Match` round trip; pacing.
- [ ] 4.7 **`feed_importer`** (`ImportDueFeeds(nowUs, budget)`, `ImportSource(sourceId, nowUs, optional documentText)`):
	- per source: fetch → parse → text filters → `IngestItem`s → one transaction (`IngestEvents`, `TouchLastScanned`, `UpdateFeedValidators`, a `scan_runs` row, and the "missing from the feed" list);
	- **after 3 trailing failures it raises one admin alert** (`TableHelpers::AdminAlerts`), which EV4's digest emails;
	- **`documentText`** parses supplied text instead of fetching. That's for single-document formats only, and it's what the test_helper uses.

	**Tests:** a full run over two fake sources (one ok, one failing) that leaves the good one's events and both runs; the time budget stops early and the next run resumes; the third failure raises exactly one alert; filters; `documentText`.
- [ ] 4.8 **The ingest rules, in EV1's `EventHelper`** (same stream; pin-downs 8 and 10):
	- `IngestItem.cancelled`;
	- the create-time venue name match, default categories and scene tags, and series title match;
	- `auto_approve`;
	- new outcome constants (`cancelled_to_pending`, `skipped_cancelled`) and the review notes.

	The `/api/admin/ingest_events` JSON gains an optional `cancelled`.

	**Tests:**
	- each rule fires only at creation, and a curator's edit survives a re-ingest;
	- the venue match picks the right room and falls back to the source's venue;
	- a cancelled approved event becomes pending with its note, and a cancelled new one is not created;
	- auto-approve creates approved, and a major change stays approved;
	- unknown default slugs are reported.
- [ ] 4.9 **`honuware_layering.cmake`:** register `business_logic/feeds` under business_logic. It may link foundation's http but not endpoints.

### 5. endpoints

- [ ] 5.1 **`POST /api/admin/import_event_feeds`** (`admin_import_event_feeds.cpp`, anchored in `RegisterAllEndpoints()` per the volatile-anchor rule):
	- **who:** `run_scheduled_jobs` (EV4's permission) or `manage_events`;
	- **body:** empty runs the due sources within the budget (the job's call). `{"sourceId": N}` imports that one source now, due or not (the "import now" path);
	- **returns** the run summaries;
	- **idempotent and self-gating**, as scheduler endpoints must be.

	**Tests:**
	- 401 and 403;
	- an empty body with nothing due;
	- a `sourceId` run;
	- an unknown source gives 404;
	- **`ThreadPool::Shutdown()` before the database assertions**, per the testing conventions.

### 6. EV4 wiring

- [ ] 6.1 Set EV4's reserved `import_event_feeds` job to **hourly** (3600 s). Each tick imports what's due within the 60-second budget.
- [ ] 6.2 Add **`cleanup_scan_runs`** to EV4's daily hygiene jobs: delete runs older than 180 days, through `ScanRuns::DeleteOlderThan`.
- [ ] 6.3 **If EV6 lands before EV4:** skip this section; imports run by hand (§7) until EV4 exists.

### 7. test_helper commands

- [ ] 7.1 **`import-feed --source-id N [--file path]`:** imports one source now. With `--file`, it parses the file instead of fetching, so offline development and the 11/1 captures work with no network.
- [ ] 7.2 **`list-scan-runs [--source-id N]`:** the latest runs with outcome, counts and details.
- [ ] 7.3 **`create-feed-source --format ics --url … [--venue-id N]`:** a source row for local testing.
- [ ] 7.4 **Tests** in `events_commands_test.cpp`'s style: `--file` over a small fixture creates pending events; a second run is all `unchanged`.

### 8. Turning the feeds on (content-ops, rides GD4a)

- [ ] 8.1 **Set `feed_format` on the sources the Findings marked "Yes":**
	- **`squarespace`:** Kremwerk complex, Cuff, Queer/Bar, Unicorn, Rendezvous (with `include_text`), PNW Black Pride, Three Dollar Bill;
	- **`ics`:** Club Z, PFLAG, Rainbow City (the concerts feed only), Rain City Soccer (Kickabouts, Programs, Ripple Effect and Skills Clinics as four sources), Entre Hermanos, ECSA. **Pan Eros / Gallery Erato is entered but left disabled** until OQ9 is settled;
	- **`json_ld`:** the Le Faux show page;
	- **`json_ld_list`:** El Sueñito's event sitemap (excluding `Bellingham`; include `queer|drag`).

	Set each source's default categories and scene tags while entering it.
- [ ] 8.2 **The asks**, for RT5's outreach (each yes recorded in `feed_permission_note`):
	- **JSON permission** from the Squarespace venues (it's one request instead of one per event);
	- **Queer Mountaineers:** does it keep one calendar (Google, Eventbrite organizer, SweatPals host) for all its gym nights? One organizer source would replace seven gym booking systems;
	- TRACTION; Queer Social Club; Charlie's Queer Books;
	- **Seattle Quake's calendar address**;
	- **LeagueApps keys** from OutLoud, Queer City Sports and ECSA;
	- Pride Sports Seattle; GSBA's calendar export; EverOut and Out x Out terms.
- [ ] 8.3 **The GD4 corrections** from §Findings:
	- the dead and hijacked domains (Neighbours, Great American Casino);
	- the renamed CC's and Club Z's real domain;
	- the gone sources;
	- the parked Comet site and the unconfirmed Caliente night;
	- the climb-night rotation (Half Moon's Tuesday, Edgeworks' missing 2nd Wednesday).

	GD4 v0.1.1 carries the first batch.

### 9. Gates

- [ ] 9.1 The docker gate green with the new test count, and the floor raised. The Windows build and tests run too, with both results reported.
- [ ] 9.2 **No test touches the network.** Every fetch goes through the test HTTP client.
- [ ] 9.3 **A live smoke run, by hand** (Levi or Mason, on a dev database):
	1. `import-feed` against PFLAG's ICS, Kremwerk on the RSS path, and Rain City's Kickabouts;
	2. the events appear pending in `/admin/events`;
	3. approve a few and see them on `/events`;
	4. **compare the Kickabouts dates with the club's Google Calendar page**, across the DST change;
	5. `list-scan-runs` shows the runs, and a second run shows `unchanged` or `not_modified`.
- [ ] 9.4 The live seed check (`--recreate_database`, run by Mason) with the new columns and table.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Honor robots.txt, even where it's just Squarespace's default?** Squarespace blocks its own JSON and ICS links for every crawler on every site. The venues almost certainly never chose that and probably don't know. Ignoring it would be technically easy, and it's one request per venue instead of one per event. But a site that asks venues for trust shouldn't start by ignoring their sites' stated rules. *Rec: honor it for every page EV6 reads. Use the RSS-plus-event-pages path, which robots.txt allows, and ask each venue for JSON permission in RT5's "you're listed" outreach. A yes goes in `feed_permission_note`, and that source switches to the faster path with the same keys.*
	- **The one exception, subscription calendars.** Google's robots.txt says `Disallow: /` for all of `calendar.google.com`, including the public `.ics` links Google itself offers for "subscribe from any calendar app." Every phone's calendar app fetches those links without asking robots.txt, because robots.txt is for crawlers of web pages. *Rec: fetch an `ics` source the way a calendar app does, with no robots check, and only ever enter a whole-calendar subscription link the publisher offers (pin-down 11). If you'd rather be stricter, the cost is losing Rain City, Club Z and Pan Eros until each one says yes.*
2. **Change honuware, or keep the fetch options in this app?** Safe fetching needs a timeout, a size cap and the private-address check inside curl, and today's honuware HTTP client has none of them. The change is small and additive, and the timeout would also protect the scheduler from a hung server. But it costs a cross-repo bump. *Rec: honuware (§1), bundled with EV4's H13 change into one bump. Fallback: an app-side `FeedHttpClient` implementing the same interface on libcurl, if the bump is inconvenient. It can move to honuware later.*
3. **Should trusted feeds skip review?** A venue's own calendar is first-party: Kremwerk's 25 events would otherwise be 25 clicks a week. The risk is a feed posting something we wouldn't list. *Rec: build `auto_approve` but leave it off everywhere at launch. Turn it on per source after a month of imports a curator never had to reject.*
4. **What happens when an event vanishes from a feed?** It might be cancelled, or the feed might just be partial: Squarespace and The Events Calendar both page at 30. Pulling it automatically would pull real events. *Rec: list vanished events in the run's details for a curator, and act only on explicit cancellations (pin-down 8). Revisit after a few weeks of real runs.*
5. **Meetup, Eventbrite, Evvnt, LeagueApps, Do206.** Meetup and Eventbrite forbid automated extraction from their pages, and their APIs need a paid plan (Meetup Pro) or our own token (Eventbrite). Evvnt needs a publisher account and has no city filter. Do206 needs a partnership. LeagueApps needs each league's key. *Rec: none in EV6.*
	- Organizers on these platforms can submit through EV3.
	- RT5 signs up as an Evvnt publisher, to measure what it would actually add.
	- The LeagueApps keys are asked for in 8.2.
	- An Eventbrite API adapter can be a later small bucket if EV3 submissions show demand.
6. **One-venue undocumented data.** Four venues expose event data that only their own site's code uses:
	- CC's Seattle's hidden Wix data (the visible page is 8 hours off);
	- Stoup's events endpoint (no year in its list);
	- Phoenix's Shopify calendar app (one time field claims UTC but isn't);
	- West Seattle Bowl's widget (stale, and its host's robots.txt says `Disallow: /`).

	Each adapter would be bespoke, for one venue, on a structure that can change without notice. *Rec: no adapters. Their queer nights are mostly stable rituals (Board Gayme Night, CC's weekly nights), so hand-enter them as series (GD4 §4.8). EV5 reads the pages if a schedule churns.*
7. **All-day events.** Some calendar entries have a date but no time: 22 of Pan Eros's, for example, and multi-day festivals often come this way. The events pages have no all-day display, so they'd show as midnight. *Rec: skip them at first (counted in the run), and hand-enter the few that matter. An "all day" display is a small EV2 follow-up if they pile up.*
8. **Cancelled events: disappear or show "Cancelled"?** Pin-down 8 takes a cancelled event off the site by returning it to pending. Someone who planned to go might prefer seeing "Cancelled" on it. *Rec: take it off for now. A public "Cancelled" badge needs a status EV1 doesn't have, and can come with EV2 polish if people ask.*
9. **Adult events have nowhere to go yet.** The guide's venues, organizations, places and pages each have an 18+ flag and sit behind GD3's 18+ gate. **Events don't**: an approved event shows on `/events`, the home-page strip and the ICS feeds for everyone. Pan Eros's calendar, the richest feed found, is mostly kink and sex-positive education ("Men Tying Men", play parties, Consent Academy). Approving them today would put 18+ listings on the main calendar with no gate. *Rec:*
	- **leave Pan Eros disabled for now** (8.1). Any other event a curator judges 18+ stays unapproved until the flag exists;
	- **add an events 18+ flag as a small Stream A follow-up:**
		- `events.adult` and `series.adult`, with an event inheriting its series' flag;
		- `event_sources.default_adult` for create-time, like the other defaults;
		- an "18+" badge on cards, GD3's gate on the event page, and 18+ events left out of the home-page strip and the public ICS feeds.

		It's EV2-sized polish, and GD10 needs it too.
	- Then turn Pan Eros on, with its events defaulting to 18+.

# Change Log

- **11/1/2026 (v0.1)** — Initial draft, written after Mason asked whether the bars publish RSS or other feeds the AI scanner wouldn't need to read.
	- **Research:** three hand research passes over about 80 sources (bars and adult venues; mainstream venues, arts and climbing gyms; orgs, leagues, platforms, aggregators and news).
	- **The answer:** RSS rarely carries event dates. About fifteen sources publish real event data as Squarespace collections, calendar (ICS) feeds or event markup, and permission matters more than technology.
	- **The design:**
		- deterministic readers (ICS with real recurrence expansion; event markup from one page, an RSS feed or sitemap, or Squarespace's RSS) behind one polite fetcher with a URL guard. It honors robots.txt for pages and fetches subscription calendars as a calendar app does;
		- EV1's ingest door, plus four create-time rules (venue name match, default tags, series title match, trusted-source auto-approve) and a cancellation flag;
		- a shared `scan_runs` audit table for EV5;
		- an hourly EV4 job with a 60-second budget;
		- a proposed honuware fetch-options change (H14).
	- **Gaps found:** events have no 18+ flag, so Pan Eros waits (OQ9). Nine open questions.
