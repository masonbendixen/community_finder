---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen · Levi Kuhn
Last Updated: 11/1/2026
Version: 0.1.1
tags:
---

# Bucket GD4 — Seed registries load

**Track:** GD — Guide & content · **Size:** M · **Streams:** split.
- **GD4a**, event sources and the recurring-series skeleton, rides Stream A (owner: Levi).
- **GD4b**, venues, profiles, organizations, places and services, rides Stream B (owner: Mason).

**Needs:** EV1 (done) for GD4a · GD2 + GD3 (done) for GD4b · **§1, the dropdown slice, first** · EV6's feed research for GD4a's feed columns, if EV6 lands first.

**Feeds:**
- milestone **M1**: "GD4a seeded";
- milestone **M2**: "GD4b seeded";
- the Content Pool (GD5–GD10), whose pages link to these rows by slug;
- EV5 and EV6, which work from the `event_sources` registry.

# Context

Implements [[Initial Project Implementation Outline]] bucket GD4: "The Appendix inventories entered as real data via admin CRUD: ~40 event sources (Wave 1 slice), then venues + orgs + health registries (Wave 2 slice). Content-ops, not code; every entry gets a fresh verification pass on entry." It also carries the outline's **do-first item** (GD2 OQ9 (f), 10/29): switch the admin editor's fixed-value fields to dropdowns before entry starts.

**The raw material** is [[Brainstorming on the website]]'s Appendix A–H (a research snapshot from **7/31/2026**). That data is now three months old, and the appendix itself flags fast-moving entries:
- Queer/Bar sold 6/2026 and is remodeling;
- Neighbours' building is for sale;
- Club Z's building is for sale;
- Gay City's rename is "in flux";
- the Center was closed for parts of July.

So **nothing is entered from the appendix as-is**. Every row is checked against a primary source on the day it's entered (§2), and that check is what sets its "Verified" stamp.

**Definitions this bucket uses, not re-litigates:**
- [[Bucket GD1 — Taxonomy & editorial foundations]] §4 (listing types and fields), §5 (status, `last_verified_at`, tiers), §6 (voice, for the one-line descriptions) and §8 (listing policy: inclusion criteria);
- [[Bucket GD2 — Directory data layer, server]]'s tables;
- [[Bucket EV1 — Events domain, server]]'s `venues`, `event_sources` and `series`.

# Scope

**In:**
- §1, the dropdown slice (code, seed-only);
- §2, the entry runbook and verification standard;
- §3, the optional bulk loader (OQ1);
- the registries as claimable checklists: §4 event sources and series (GD4a), §5 venues and profiles, §6 organizations, §7 places, §8 services, §9 "where this scene talks" links (GD4b);
- §10, gates.

**Out (explicit non-goals):**
- editorial pages (the Content Pool GD5–GD10; profile `crowd_vibe` here is one to three factual sentences, and GD5 writes the real crowd profiles);
- event entry beyond what a source brings in (events arrive by EV3, EV5 and EV6, or the EV2 editor);
- photos (CM3's consent policy first; venue exterior shots are fine later);
- anything that needs a schema change;
- individual people as listings (GD1 §6.9).

# Design pin-downs

- **Venues are entered by Stream B, together with their profiles.** The `venues` table is Stream A's *schema* (Contract 2 is about files, not data), but every guide venue needs both rows. GD2 OQ9 (d) nests the profile under its venue in the admin editor, so one sitting does both. GD4a's sources then pick the venue from the list. *Order: neighborhoods (seeded) → venues + profiles → event sources → series.*
- **Every venue gets a `venues` row; only guide-worthy venues get a profile.**
	- **Profile:** every queer venue, adult venue and graveyard entry, plus each mainstream venue that hosts a standing queer night. That's the "venues in this scene" list on scene pages.
	- **Venue row only:** a brewery that hosts one queer trivia a year.
- **Slugs are forever** (GD1 §4): kebab-case from the current name (`the-cuff-complex`, `queer-bar`, `seattle-eagle`). Settle the slug before saving; the Pool's pages will link to it.
- **Descriptions are one to three factual sentences** in the GD1 §6 voice, with every volatile claim dated ("Sold to the Cuff's owners, June 2026").
- **Status:**
	- an appendix item marked UNVERIFIED that you can't confirm is entered as `unverified` with a note, or not entered at all if it might not exist (OQ4);
	- something that turns out closed is entered as `closed` with a `closure_note` (the graveyard is a feature);
	- "indefinite pause" (Three Dollar Bill Cinema's 2026 festivals) is `seasonal` with the pause in the description (OQ5).
- **Scene tags:** accurate, not aspirational (GD1 §2): a mixed bar is not `bear` because bears go there. `sapphic`/`trans`/`qtpoc` label shared anchors accurately and never promise curation.
- **The 18+ flag:** set on adult venues' profiles (Steamworks, Club Z), sex-positive orgs (Pan Eros, per GD2 OQ11 (b)) and the Denny Blaine and Howell places.
- **Health flag:** `is_health` starts checked (GD2 OQ9 (b)). Uncheck it only for everyday services.

# Layered work items

### 1. Do first: dropdowns for the fixed lists (code, seed-only; owner: whoever starts entry first)

- [ ] 1.1 In `create_app_tables.cpp`, add a `PopulateAdminEnums` that registers one honuware admin enum per fixed list, using the platform's `AdminEnums` / `AdminEnumValues` / `AdminColumnEnums` helpers (knottyyoga's `PopulateAdminEnums` in its `create_database.cpp` is the model). It links each enum to its column's `admin_column_data_info` row:
	- **EV1:** `scene_tags.coverage` · `neighborhoods.region` · `venues.status` / `volatility_tier` · `event_sources.kind` / `volatility_tier` · `series.status` / `volatility_tier` · `events.origin` / `status`
	- **GD2:** `venue_profiles.volatility_tier` · `organizations.org_kind` / `status` / `volatility_tier` · `places.place_kind` / `status` / `volatility_tier` · `service_listings.service_kind` / `status` / `volatility_tier` · `guide_pages.section` / `status` / `volatility_tier` · `guide_links.platform`
	- plus EV3's and EV6's lists if those have landed.

	**The values come from the same constants that build the CHECKs** (`db_schema/freshness.h` and each table's value list), so a dropdown can never offer a value the database refuses.
- [ ] 1.2 **Test** (`create_app_tables_test.cpp`): for every CHECK-constrained text column in the app schema, an admin enum exists, is linked to that column, and lists exactly the CHECK's values in the same order. Written generically over `MakeDatabaseInfo`, like GD2 OQ9 (e)'s checkbox test, so a future enum column fails the test until it gets its dropdown.
- [ ] 1.3 Live seed check (announced, destructive), then a browser check: `/admin/tables` → organizations → new row shows Kind, Status and Cadence as dropdowns.

### 2. The entry runbook (read once; applies to every row in §4–§9)

- [ ] 2.1 **Verification standard. A row is verified when someone has, that day:**
	1. **Confirmed it exists and is operating** from a primary source: its own website or social account showing activity in the last 60 days, its Google Maps listing with current hours, or a phone call. For a closure, a news report (CHS, SGN, The Stranger) or the business's own announcement.
	2. **Checked the facts entered** (address, URL, meeting cadence, kind) against that source.
	3. **Set `last_verified_at_us` to today in the admin editor.** The verification act is explicit (GD1 §5): routine edits never set it, and the loader (§3) sets it only when the file row carries a `verified_on` date.

	Write the source in this doc's checklist line ("✓ 11/4, site + IG") so the next verifier knows where to look.
- [ ] 2.2 **Claim a batch** by putting your name on its heading, enter it in one sitting, then tick the lines. Batches are sized for an hour or two each.
- [ ] 2.3 **When the appendix is wrong,** fix the row and add a dated note under the batch ("Diesel: closed Mondays now, 11/4"). The brainstorm stays as the historical snapshot; nobody edits it.

### 3. Optional: a bulk loader (OQ1, so build it only if "Agreed")

- [ ] 3.1 `business_logic/registry_loader.{h,cpp}`: `LoadRegistry(Transaction&, DatabaseHelper, const Json::Value&) → RegistryLoadReport`. It takes JSON with one array per table (`venues`, `venue_profiles` keyed by the venue's slug-to-be, `organizations`, `places`, `service_listings`, `event_sources`, `series`, `guide_links`, and scene tags by slug).
	- **Insert-only by slug:** a row whose slug already exists is skipped and reported, never overwritten. After the first load the database is the source of truth, and edits happen in the admin editor.
	- `populate_sample_data`'s shape and report wording (EV2 §6) are the model.
- [ ] 3.2 `database_helper --load_registry=<path>`. It is **not** destructive-gated: it only adds rows, and real data is the point. It refuses to run if `--populate_sample_data` rows are present, so fake and real listings never mix (OQ6).
- [ ] 3.3 **Tests** (`registry_loader_test.cpp`): every table loads; re-running changes nothing; an unknown slug reference fails the whole load before writing; `verified_on` sets `last_verified_at_us` and its absence leaves it empty; a CHECK-violating value fails with the row named.
- [ ] 3.4 The registry files live in `data/registry/*.json` in the repo, so entries are reviewed in PRs. *This is a public repo, but these are public facts about public businesses, which the site publishes anyway.*

### 4. GD4a — Event sources and recurring series (Stream A; owner: Levi)

**Event-source fields:**
- `name`;
- `url` (the page or feed);
- `kind`: `ics_feed` · `venue_site` · `ticketing_org` · `aggregator` · `editorial_monitor` · `submission` · `social`;
- `venue_id` when the source is one venue's own calendar (its events default to that venue, EV1 OQ10 (b));
- `scan_hints`: plain-English notes for EV5 ("events page lists only this month; drag nights are on IG first");
- `volatility_tier`: `weekly` for calendars, `monthly` for slow org pages;
- `enabled`;
- `notes`.

**Check EV6's research table** ([[Bucket EV6 — Structured feed import]] §Findings) before entering a source. Where a machine-readable feed exists, its URL goes in, and once EV6's columns land, `feed_format` is set (`ics`, `squarespace` or `json_ld`). `kind` still says what the source is: Kremwerk stays `venue_site` with `feed_format = squarespace`.

> **Corrections from the 11/1 feed research** (EV6 §Findings has the evidence). Apply them while entering:
> - **Never link `neighboursnightclub.com`.** It now redirects to a gambling site. Neighbours is Instagram/Facebook-only until a new site appears, and whether it's open needs checking.
> - **CC Attle's is now "CC's Seattle"** at `ccsseattle.com`. Club Z's site is **`thezclub.com`**, not clubzseattle.com (parked).
> - **Dead sites:** `dieselseattle.com` (host error) and `changesinwallingford.com` (no DNS). Pony's certificate expired again on 11/1.
> - **Gone:** The Gay Agenda (shut down), Different Spokes (no Seattle club found) and Team Seattle (dormant since about 2018). Enter them as closed if at all.
> - Of the test Meetup groups, Another Gay Social Club and Seattle Gaymers return "Group not found", and Seattle Queer & Ally Hiking has had no events since 1/31/2026.
> - Kremwerk's feed runs about 9 weeks ahead, not the appendix's 4 months.
> - EverOut's robots.txt no longer blocks crawlers. Its terms still need checking, so it stays human-read until RT5 asks.

**Sources never scraped** are still entered (`social`, `enabled = false`), so the registry is the complete map of where events come from and EV5 knows never to visit them.

- **4.1 Venue calendars** *(claimed: ______)*. `venue_site` unless EV6 found a feed:
	- [ ] Kremwerk + Timbre Room + Cherry complex (the appendix's "clean 4-month feed"; about 9 weeks ahead as of 11/1)
	- [ ] Julia's on Broadway / Le Faux
	- [ ] Unicorn / Narwhal
	- [ ] Neighbours *(its old domain is hijacked: `social` rows only)*
	- [ ] Wildrose
	- [ ] The Cuff Complex
	- [ ] Queer/Bar *(remodeling since 7/2026; check it's reopened)*
	- [ ] Massive
	- [ ] Diesel
	- [ ] CC's Seattle (formerly CC Attle's; `ccsseattle.com`)
	- [ ] Lumber Yard Bar
	- [ ] Southgate Roller Rink
	- [ ] Rough & Tumble
	- [ ] Charlie's Queer Books
	- [ ] Pony *(site SSL expired in 7/2026; IG is its live channel: enter the site plus a disabled `social` row)*
- **4.2 Organization calendars** *(claimed: ______)*:
	- [ ] Seattle Pride (year-round community calendar)
	- [ ] GSBA
	- [ ] GenPride
	- [ ] Gay City / Seattle's LGBTQ+ Center
	- [ ] Ingersoll Gender Center
	- [ ] Lambert House
	- [ ] PFLAG Seattle
	- [ ] Seattle Men's / Women's Chorus (`seattlechoruses.org`)
	- [ ] Rainbow City Performing Arts
- **4.3 Leagues** *(claimed: ______)*, `monthly` tier (seasons move slowly):
	- [ ] ECSA
	- [ ] Seattle Quake
	- [ ] ORCA
	- [ ] Seattle Frontrunners
	- [ ] OutLoud Sports
	- [ ] Pride Sports Seattle
	- [ ] Rain City Soccer
	- [ ] Cascade Flag Football
	- [ ] Seattle Pride Hockey Association
	- [ ] Cheer Seattle
	- [ ] Seattle Tennis Alliance
	- [ ] Queer City Sports (LeagueApps)
- **4.4 Ticketing and group platforms** *(claimed: ______)*, `ticketing_org`:
	- [ ] Eventbrite: Queer Mountaineers
	- [ ] Eventbrite: PNW Black Pride
	- [ ] Eventbrite: Queer Figure Drawing
	- [ ] Eventbrite: Vertical World
	- [ ] Meetup: Another Gay Social Club
	- [ ] Meetup: Seattle Queer & Ally Hiking
	- [ ] Meetup: Seattle Gaymers
	- [ ] Meetup: Queer Geek!
	- [ ] Meetup: West Seattle LGBT+
	- [ ] Meetup: Banter Date
	- [ ] Meetup: Aces & Aros
	- [ ] RunSignup (Frontrunners races and Pride runs)
- **4.5 Aggregators** *(claimed: ______)*, `aggregator`, **`enabled = false` until permission or credit terms are settled** (RT5's coopetition outreach):
	- [ ] Sapphie Taffy (consent and credit first; the brainstorm's rule)
	- [ ] Queer Social Club (Seattle)
	- [ ] ~~The Gay Agenda (Seattle)~~ *(shut down by 11/2026; skip)*
	- [ ] seattleprideguide.com
	- [ ] Do206
	- [ ] Evvnt (publisher signup is RT5's)
- **4.6 Editorial monitors** *(claimed: ______)*, `editorial_monitor`, closure and ownership news for EV5, never events:
	- [ ] CHS Capitol Hill Seattle
	- [ ] Seattle Gay News
	- [ ] Seattle Gay Scene
	- [ ] The Stranger / EverOut (read by humans; `enabled = false` for any automated pass. The Stranger's RSS feeds are fine for EV5's monitors, and EverOut's robots.txt no longer blocks crawlers, but its terms need checking)
- **4.7 Never-scrape social leads** *(claimed: ______)*, `social`, `enabled = false`, for humans to check for leads:
	- [ ] @seattlequeerevents
	- [ ] @bloom.seattle
	- [ ] @thesocialqueer
	- [ ] @seattledykealliance
	- [ ] Queer Exchange Seattle (FB group)
- **4.8 The recurring-series skeleton** (Appendix C), so a scene's "regular nights" exist before the first event does (OQ3) *(claimed: ______)*. Each needs a slug, title, cadence text, venue, categories and scene tags, `status = open`, and verification that it still runs:
	- **Weekly:**
		- [ ] Board Gayme Night (Stoup, Mon)
		- [ ] Dum Top Trivia (Cuff, Tue)
		- [ ] Drag Queen Bingo (Unicorn, Tue)
		- [ ] Pride Skate (Southgate, Wed)
		- [ ] Seattle Gaymers night (CC's, Wed)
		- [ ] Queeraoke (Lumber Yard, Wed)
		- [ ] Wildrose karaoke (Thu)
		- [ ] Caliente (Comet, Thu)
		- [ ] Le Faux (Julia's, Fri)
		- [ ] MX (Queer/Bar, Fri)
		- [ ] Fridays Are a Drag (Neighbours)
		- [ ] Lashes (Unicorn, Fri)
		- [ ] Bad Girls Brunch (Julia's, Sat)
		- [ ] Mimosas Cabaret (Unicorn, Sun)
		- [ ] Flammable Sundays (Chop Suey)
		- [ ] Bear Social (Lumber Yard, Sun)
	- **Monthly:**
		- [ ] The Gathering (PNW Black Pride @ CC's, 1st Fri)
		- [ ] Leather Social (CC's, 1st Fri)
		- [ ] Safeword (Kremwerk, 1st Sat)
		- [ ] Centerfold (Queer/Bar, 2nd Thu)
		- [ ] Queen4Queen (Pony, 2nd Fri)
		- [ ] T4T (Kremwerk, 2nd Sat)
		- [ ] Queerly Beloved (Jet City Improv, 2nd Sat)
		- [ ] Judy (Queer/Bar, 4th Sun)
		- [ ] Qu-Art sober artists (PUSH/PULL, last Fri)
		- [ ] Out in Tech happy hour (Rough & Tumble, 2nd Wed)
		- [ ] Another Gay Social Club (2nd Mon + last Fri)
		- [ ] Rainbow Elder Play & Connect (Crossroads, 3rd Sat)
		- [ ] VALA queer figure drawing (Redmond, 3rd Sun)
		- [ ] Queer Dharma (1st/3rd Mon)
	- **The climb-night rotation:**
		- [ ] Vertical World North (1st Thu)
		- [ ] Uplift (2nd Mon)
		- [ ] Climb NORA (2nd Fri)
		- [ ] Castle (3rd Mon)
		- [ ] SBP Poplar (3rd Thu)
		- [ ] Half Moon (4th Mon)

### 5. GD4b — Venues and profiles (Stream B; owner: Mason)

**Profile fields:**
- `slug`;
- `crowd_vibe` (one to three sentences now; GD5 deepens them);
- `price_band` (`$`–`$$$`);
- `adult`;
- `closure_note` for closed venues;
- scene tags;
- `last_verified_at_us`.

**Venue fields:** name, address, city, zip, url, neighborhood, status.

- **5.1 Queer venues** *(claimed: ______)*:
	- [ ] Pony (Capitol Hill) · dive/arty/cruisy
	- [ ] The Cuff Complex (Capitol Hill) · `leather`, `bear` · sold 3/2025
	- [ ] Queer/Bar (Capitol Hill) · drag-forward · sold 6/2026, remodeling: **the highest-volatility entry**, set its tier to `monthly`
	- [ ] Seattle Eagle (Capitol Hill) · `leather` · oldest leather bar
	- [ ] Diesel (Capitol Hill) · `bear`
	- [ ] CC Attle's (Capitol Hill) · mixed · 1st-Fri Leather Social
	- [ ] Madison Pub (Capitol Hill) · low-key sports and pool
	- [ ] Wildrose (Capitol Hill) · `sapphic` · est. 1984, the oldest US lesbian bar
	- [ ] Neighbours (Capitol Hill) · dance club, est. 1983 · **building for sale: watch**
	- [ ] Massive (downtown) · three-floor dance club, 2023
	- [ ] Union (Capitol Hill, *not* SODO as the Seattle Times has it) · cocktail lounge
	- [ ] Crescent Lounge (Capitol Hill) · karaoke dive since the 1940s
	- [ ] Unicorn / Narwhal (Capitol Hill) · carnival and drag
	- [ ] Julia's on Broadway / Le Faux (Capitol Hill) · drag dinner theatre
	- [ ] Kremwerk + Timbre Room + Cherry (downtown) · queer-owned electronic
	- [ ] Lumber Yard Bar (White Center) · `bear` · arson 2021, reopened 11/2022
	- [ ] Boombox (White Center)
	- [ ] Changes (Wallingford) · the only gay bar north of the Ship Canal
- **5.2 Venues with a standing queer night** *(claimed: ______)*. Profiles describe the night, not the whole venue:
	- [ ] Southgate Roller Rink (White Center) · Wed 21+ Pride Skate
	- [ ] Chop Suey (Capitol Hill) · Flammable Sundays
	- [ ] Comet Tavern (Capitol Hill) · Caliente Thursdays
	- [ ] Rough & Tumble (Ballard) · queer trivia; Out in Tech
	- [ ] Stoup Brewing Capitol Hill · Board Gayme Night
	- [ ] El Sueñito · book club, run club, karaoke
	- [ ] Charlie's Queer Books (Fremont) · four book clubs
	- [ ] Jet City Improv · Queerly Beloved
	- [ ] Phoenix Comics & Games · queer board games
	- [ ] West Seattle Bowl · three queer leagues
	- [ ] Great American Casino Tukwila · Royal Flush drag
	- [ ] Gallery Erato
	- [ ] Raygun Lounge
	- [ ] Asylum Collective · `sapphic` goth
	- [ ] Métier Brewing · bingo
	- [ ] The climbing gyms from §4.8, as venue rows only, unless GD5 writes a climbing profile
- **5.3 Adult venues** *(claimed: ______)*. Profile `adult` = true, tier `quarterly`:
	- [ ] Steamworks (1520 Summit Ave, Capitol Hill) · 24/7
	- [ ] Club Z (1117 Pike St) · est. 1976, its 50th year · **building listed for sale 2/2026: watch**
	- [ ] Doghouse Leathers (retail and community hub) · `leather` · decide whether its profile is adult (it's a shop)
	- [ ] Center for Sex Positive Culture · *listed CLOSED in research, needs primary confirmation*: enter as closed only once confirmed
- **5.4 The closure graveyard** *(claimed: ______)*. `status = closed`, `closure_note` with year and reason, a successor where one exists, no `last_verified_at` needed beyond confirming the closure:
	- [ ] The Comeback (2023)
	- [ ] R Place (2021, lease lost)
	- [ ] Re-bar (2020)
	- [ ] Purr (2018)
	- [ ] Basic Plumbing / Tribe (2012)
	- [ ] Double Header (2015)
	- [ ] Supernova SoDo: *UNVERIFIED queer status; enter only if confirmed it was a queer venue*

### 6. GD4b — Organizations (Stream B)

**Fields:**
- `name`, `slug`;
- `org_kind`: `sports-league` · `chorus-performance` · `club-hobby` · `recovery` · `faith` · `professional` · `community-support` · `social-meetup`;
- `description`, `url`;
- `meeting_cadence_text` ("Mondays 6:30 PM, Cal Anderson");
- neighborhood, which is empty for countywide or online groups;
- `adult`;
- scene tags;
- status.

- **6.1 Sports** *(claimed: ______)*, all `jock` unless noted:
	- [ ] ECSA softball (Sundays, north SeaTac; 47th year)
	- [ ] Rain City Softball (women, trans and NB · `sapphic`, `trans`)
	- [ ] Seattle Quake RFC
	- [ ] Seattle Mudhens rugby *(verify who it serves)*
	- [ ] Seattle Frontrunners (est. 1985; Monday Cal Anderson mini-run, the perfect low-barrier onramp)
	- [ ] ORCA Swim (est. 1984)
	- [ ] Seattle Otters water polo (1990)
	- [ ] Different Spokes
	- [ ] Outspoken cycling
	- [ ] OutLoud Sports (kickball, dodgeball, volleyball, bowling, darts, pickleball)
	- [ ] Pride Sports Seattle
	- [ ] Queer City Sports
	- [ ] Rain City Soccer (2000)
	- [ ] Cascade Flag Football
	- [ ] Seattle Pride Hockey Association
	- [ ] Seattle Basketball Club
	- [ ] Seattle Tennis Alliance
	- [ ] Cheer Seattle
	- [ ] the three West Seattle Bowl leagues *(one row or three: decide on entry)*
	- [ ] SkiBuddies
	- [ ] Olympic Yacht Club
	- [ ] OutVentures (outdoors, decades old)
	- [ ] Queer Mountaineers
	- [ ] Team Seattle (umbrella)
- **6.2 Arts and performance** *(claimed: ______)*:
	- [ ] Seattle Men's Chorus
	- [ ] Seattle Women's Chorus
	- [ ] Captain Smartypants
	- [ ] Rainbow City Performing Arts
	- [ ] Three Dollar Bill Cinema (SQFF and TRANSlations on "indefinite pause" for 2026 · OQ5)
	- [ ] Intiman Theatre
	- [ ] ArtsWest
	- [ ] Pride Across the Bridge figure drawing
	- [ ] Gay History Tour
- **6.3 Community and support** *(claimed: ______)*:
	- [ ] Gay City / Seattle's LGBTQ+ Center (use the current name, with the former name noted)
	- [ ] Lambert House (youth: pointer-only per GD1 §6.7)
	- [ ] GenPride / Pride Place (`elders`)
	- [ ] Entre Hermanos (`bipoc`)
	- [ ] UTOPIA WA (`bipoc`, `qtpoc`)
	- [ ] Lavender Rights Project (`trans`)
	- [ ] Ingersoll Gender Center (`trans`, est. 1977)
	- [ ] Pride Foundation
	- [ ] PFLAG Seattle
	- [ ] Camp Ten Trees
	- [ ] Peer Seattle
	- [ ] Seattle Dyke Alliance (`sapphic`, link-out)
	- [ ] Queer the Land
	- [ ] Seattle Counseling Service (**`closed`, 2022** — the market's number-one stale-data trap; its `closure_note` names the successors, which GD6 spells out)
- **6.4 Recovery, faith and professional** *(claimed: ______)*:
	- [ ] Gay Men in Recovery AA (`sober`; point at seattleaa.org and the Meeting Guide app for times)
	- [ ] Gay & Lesbian Beginners AA (`sober`)
	- [ ] Queer Grief Club (virtual)
	- [ ] Queer Dharma (Shambhala, 1st/3rd Mon · `faith`)
	- [ ] Out in Tech Seattle
	- [ ] Seattle Queer in Tech
	- [ ] Lesbians Who Tech (`sapphic`, link-out)
	- [ ] GSBA (`professional`)
- **6.5 Social and Meetup groups** *(claimed: ______)*, `social-meetup`:
	- [ ] Another Gay Social Club (~3,900 members; likely the "Gay People in Seattle gatherings")
	- [ ] Seattle Queer & Ally Hiking
	- [ ] Seattle Gaymers (`geek`)
	- [ ] Queer Geek! (`geek`)
	- [ ] West Seattle LGBT+
	- [ ] Banter Date
	- [ ] Aces & Aros
- **6.6 Eastside, first-class** *(claimed: ______)*:
	- [ ] Monthly Eastside LGBTQ+ Happy Hour
	- [ ] East King County LGBTQ+ All Ages Meetup
	- [ ] Rainbow Elder Play & Connect (`elders`)
	- [ ] Rainbow on the Eastside (annual, August)
	- [ ] Eastside QTPOC Collective (Together Center, Redmond · `qtpoc`)
- **6.7 Pride organizations** *(claimed: ______)*:
	- [ ] Seattle Pride
	- [ ] PNW Black Pride (`bipoc`)
	- [ ] Burien Pride
	- [ ] Alki Beach Pride
	- [ ] White Center Pride
- **6.8 Sex-positive** *(claimed: ______)*. `adult` = true:
	- [ ] Pan Eros Foundation / Seattle Erotic Art Festival

### 7. GD4b — Places (Stream B)

*(claimed: ______)* `place_kind`: `park-beach` · `landmark` · `historical`. The editorial is GD10's or GD7's page; the row is the structured anchor.

- [ ] Denny Blaine Park (Central District edge · `park-beach` · adult · tier **`monthly` while litigation is live**)
- [ ] Howell Park (`park-beach` · adult)
- [ ] Volunteer Park (`historical` as cruising history; adult; Trans Pride is held here)
- [ ] Cal Anderson Park (`landmark`: Pride, Frontrunners, the Hill's commons) *(addition: confirm we want it)*
- [ ] Seattle AIDS Memorial Pathway (`landmark`) *(addition: confirm)*

### 8. GD4b — Service listings (Stream B)

*(claimed: ______)* `service_kind`: `sti-testing` · `prep` · `hiv-care` · `primary-care` · `mental-health` · `gender-affirming` · `dental` · `everyday-service`. Tier `semiannual` for health. **Every one is re-verified on entry; Appendix E is "the corrected 2026 facts" as of July.**

- [ ] Harborview Sexual Health Clinic (908 Jefferson; walk-ins by 4 PM; walk-in mpox vaccine)
- [ ] The Center's free STI testing (six days a week, plus mailed kits)
- [ ] Kelley-Ross One-Step PrEP, in-Center location
- [ ] Kelley-Ross One-Step PrEP, 7th & Madison location
- [ ] Kelley-Ross TelePrEP (statewide) *(a third row, or a line in the two above)*
- [ ] Planned Parenthood: U District
- [ ] Planned Parenthood: Central District
- [ ] Planned Parenthood: Northgate *(hormone care at all three)*
- [ ] Madison Clinic at Harborview (`hiv-care`; the largest in the PNW)
- [ ] Bailey-Boushay House (`hiv-care`; now holds the former Lifelong housing clients)
- [ ] Country Doctor → **Seattle Roots Community Health** (check which name is live)
- [ ] Swedish LGBTQIA+ Program
- [ ] UW Medicine trans health program (`gender-affirming`)
- [ ] Capitol Hill Medical
- [ ] Cedar River Clinics (Renton and others)
- [ ] Optimism Counseling (`mental-health`)
- [ ] Integrative Counseling (Fremont · `mental-health`)
- [ ] The Center's 12-session counseling program (`mental-health`)
- [ ] NAMI referral line (`mental-health`)
- [ ] Ingersoll provider directory (`gender-affirming`; a link to the incumbent, not a fight with it)
- [ ] 2026 King County Trans Resource & Referral Guide (`gender-affirming`)

*(The PrEP DAP program and DoxyPEP guidance are facts with numbers in them. They belong in GD6's health page prose with dates, not as listings.)*

### 9. GD4b — "Where this scene talks" links (Stream B)

*(claimed: ______)* Collected **while entering** §5–§6: for each venue or org, its Discord, Bluesky, Instagram, Facebook group or Meetup, as `guide_links` rows (label, URL, platform, parent). Scene-level links (a scene's Discord, say) attach to the scene tag. Outbound only, never embedded (GD1 §4).

- [ ] Venues' links entered alongside 5.1–5.3
- [ ] Organizations' links entered alongside 6.1–6.8
- [ ] Scene-level links: at least one per `full` scene, if one exists. Record "none found" honestly rather than inventing one.

### 10. Gates

- [ ] 10.1 §1's test green on Windows and Linux, and its live seed check done.
- [ ] 10.2 If §3 is built: its tests green on both platforms, and a dry-run load into a scratch database before the real one.
- [ ] 10.3 **GD4a done** (milestone M1's "GD4a seeded"):
	- every §4 line ticked or explicitly dropped with a reason;
	- `/api/event_series` lists the skeleton;
	- the admin editor's `event_sources` shows every row with a kind and tier.
- [ ] 10.4 **GD4b done** (M2's "GD4b seeded"):
	- every §5–§9 line ticked or dropped with a reason;
	- `/guide/venues`, `/guide/orgs?kind=sports-league`, `/guide/services` and `/guide/graveyard` show the rows;
	- every open row shows "Verified {Month Year}", with **zero** "Not yet verified" on open rows, except where a deliberate `unverified` note explains why;
	- adult rows sit behind the 18+ gate.
- [ ] 10.5 The outline's Stream A row "GD4a done" and Stream B row "GD4b done" ticked.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Type it all in by hand, or load it from files?** The outline says "via admin CRUD," which works, but this is roughly 250 rows, and the editor is one form per row. The alternative (§3) is a small loader: the rows live in JSON files in the repo, the database helper adds any whose slug isn't there yet, and nothing existing is ever overwritten. Its upsides: entries get reviewed in a pull request like code, a fresh database can be refilled in seconds, and several people can prepare batches at once without stepping on each other. Its downsides: a day of code, and two places a row could be edited. The fix for that is "files for the first load, the admin editor after that." *Rec: build the loader. Hand entry stays fine for one-off additions.*
2. **Who enters venue rows?** `venues` is Stream A's table, but every guide venue needs a venue row plus a profile, and the editor shows them together. *Rec: Stream B enters venues together with their profiles. Stream A's sources then pick from the list. Contract 2 is about who edits which code files, not who types data.*
3. **Seed the recurring-series skeleton now?** The outline's GD4a mentions only the ~40 sources. But scene pages and the series index are empty until series exist, and Appendix C already lists about 35 regular nights. *Rec: yes, as §4.8. Each series is verified like any listing, so the stale ones get dropped, not entered.*
4. **What about appendix entries we can't confirm?** Some are marked UNVERIFIED (Supernova's queer status, the Center for Sex Positive Culture's closure), and others may simply have gone quiet. *Rec: if it might not exist, leave it out and note why. If it clearly exists but we can't confirm the details, enter it as `unverified` with a note. GD1 says `unverified` should be rare and temporary.*
5. **"Indefinite pause" isn't one of GD1's statuses.** Three Dollar Bill Cinema cancelled its 2026 festivals but still exists. The choices are `seasonal` (alive but dormant) or `open` with the pause in the description. *Rec: `seasonal`, with the pause dated in the description. It renders the "Seasonal" marker, which is honest without calling it closed.*
6. **Keep sample data and real data apart.** EV2 §6's `--populate_sample_data` fills a database with made-up listings. Mixing those with real ones on a database that later goes live would be embarrassing. *Rec: the loader refuses to run on a database that has the sample source in it. The production database is never given sample data.*
7. **Knotty Yoga's own listing.** GD1 §8.3 says it's listed under the same rules as everyone else. *Rec: enter it when it opens to the public, by those rules: real, operating and verifiable. That keeps the founded-by relationship in the footer and About page, never in the registry.*

# Change Log

- **11/1/2026 (v0.1.1)** — §4 picks up the 11/1 feed research from [[Bucket EV6 — Structured feed import]]: set `feed_format` where a feed exists. Plus a corrections callout covering the hijacked Neighbours domain, CC's and Club Z's real sites, dead sites, gone sources and missing Meetup groups, with inline notes on the affected rows.
- **11/1/2026 (v0.1)** — Initial draft from the outline's GD4 blurb and do-first item, the brainstorm's Appendix A–H, GD1 §4–§8, GD2's tables and EV1's sources and series. The registries are laid out as claimable checklists, with the dropdown slice (§1) first and an optional bulk loader (§3, OQ1). Seven open questions.
