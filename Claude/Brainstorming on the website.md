---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/31/2026
Version: 0.1
tags:
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

There is a website: https://seattlegayscene.com/ that used to be kind of the place to go to see what upcoming events were happening in the world of gay events in Seattle. It's clearly a static, manually maintained website as it always frequently had events listed that had already happened but it was moderately well maintained. It's been largely unmaintained now for a couple of years minus a little effort during pride month.

I am building out a spa, massage space, and fitness / acrobatics studio. I have a large gay clientele. I need to be able to market and gays are a key demographic. Very few gays are in Facebook these days. Google has diminished as an ad platform. Instagram has some relevance but is declining and hard to differentiate yourself on it.

I'd like to have a stie that people are drawn to that I can build community but also use for low key doing my own marketing for free.

I'd like to cover what seattlegayscene did, mostly bar events, but do a lot more. I'd love to list all LGBT relevant events and opportunities. The networking things on meetup. The monthly Gay People in Seattle gatherings. Theatre events, pride things, gay figure drawing, the sports leagues, and any other ideas.

I'd like to cast a wider net though. I'd love to have a guide to being gay in Seattle. List gay gyms, doctors, dentists, contractors. Where to get STD tested. Have information about the crowds of each of the gay bars. List the nude beaches. List bathhouses. List cruising spots. Generally list the assorted gay offerings the city has to offer. Please research and think deeply to help me think of ideas here. I also don't want to moralize here. I want to span the spectrum from family friendly / wholesome to rampantly slutty.

I'd also love a new to Seattle guide that lists the various neighborhoods and gives rundowns of the various options. Proximity to light rail. Talks about the traffic.

Seattle is also not a social / friendly city. There is a widely known phenomenon called the "Seattle Freeze" that unfortunately is very real. I'd like to have a guide on overcoming this. Suggestions like joining a volunteer event, sports league, Seattle Men's chorus, etc. I'd love help think of ideas here.

I think an eventual growth path would be community forums. Letting various people offer services like gay massage therapists, ways for people to make friends, people looking for activity partners (chess, tennis, DnD, board games, etc.). Maybe even a dating / personal section.

This is a lot to be honest. Can you help me organize my thoughts here and give suggestions for things you can think of and help organize this set of ideas into more of a cohesive plan? I'm not really ready to move to implementation or design yet. I'd like to keep brainstorming with your help so please think very deeply and help with this so I can put something together to have friends review before moving on to an actual implementation plan.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# The Pitch (v0.1 — for friend review)

**A maintained, structured home for queer Seattle: every event, every recurring group, every venue, every resource — from drag brunch to bathhouse — with a freshness guarantee nobody else can make.** Events are the heartbeat, the city guide is the moat, the "beat the Seattle Freeze" recurring-groups layer is the wedge, and community features grow on top once trust exists. It runs on the CommunityFinder platform we already built (auth, admin CRUD, photo pipeline, AI-scanner architecture), which is purpose-built to solve the exact thing that has killed or calcified every predecessor: **the maintenance labor of keeping a structured city guide true.**

**One premise correction from research (7/31/2026):** seattlegayscene.com is *not* dead — it still publishes ~2–5 posts/week (20 posts in June 2026) as a one-man press-release/nightlife blog run by Michael Strangeways. What died is its *structure*: the calendar link 404s, the events page returns a 2007 article, the staff page hasn't been touched since 2015. That pattern repeats across the whole landscape: eight half-alive properties, none of which maintains the structured layer. The opportunity is real, but the pitch is not "replace a dead site" — it's **"be the first maintainable one."**

# What the Research Found (July 2026)

Three deep research passes: (1) the event/organization landscape, (2) the guide/directory content itself, (3) peer models, legal, and marketing reality. Full seed inventories are in the Appendix; this section is the synthesis.

## The incumbent landscape

| Property | What it is | Why it doesn't fill the gap |
|---|---|---|
| **Seattle Gay Scene** (seattlegayscene.com) | One-man nightlife/drag/theatre blog, alive since ~2007 | Blog stream only; calendar 404s; no directory; no structure |
| **Seattle Gay News** (sgn.org) | 3rd-oldest LGBTQ paper in US (1974), real reporting | Weak site, content unfindable; calendar submissions accepted *by mail/fax*; not a guide |
| **CHS Blog** | The actual source of truth for venue openings/closings/ownership | News feed, not a guide; blocks scrapers |
| **EverOut** (The Stranger) | Structured events DB with a `queer` category, recurrence-aware | General-audience, event-first; no crowd profiles, health, services, relocation; blocks scrapers |
| **The Ticket** (Seattle Times) | City guides incl. "First-Timer's Guide to Queer Nightlife" | Written once, never maintained — 4 confirmed factual errors (wrong neighborhoods, 4-year-old "recently") |
| **Sapphie Taffy** | Hand-curated **175+ recurring LGBTQ/QTPOC events** page — the best single resource in the city | One unpaid person, updated 3/2026, already decaying; no directory/guide; single point of failure |
| **Queer Social Club** | Portland queer events curator (14K IG — bigger than Gay City's), now expanding to Seattle | Portland-first; submission+newsletter only, no guide/directory |
| **The Gay Agenda** (thegayagenda.fyi) | Global queer city guide with iOS/Android apps, daily updates, weekly newsletter | **Closest product comp** — but Seattle is not a featured city (yet) |
| **GSBA / TravelOUT** | LGBTQ+ chamber, 1,300+ members, Visit Seattle's designated authority | Pay-to-list directory — answers "who paid dues," not "who should I hire"; no nightlife/adult/health |
| **Pride Pages Seattle** | "Yellow pages but gayer" business directory | Footer says © 2024; semi-stale |
| **Visit Seattle LGBTQ page** | Tourism org resource list | Confirmed stale (2022 header image, dead org names) |
| **GayCities / TravelGay / misterb&b / GayMapper etc.** | SEO travel shells auto-titled "2026" | Recycle each other; several still list venues closed since 2021–2023 |
| **Sniffies** | Seattle-founded map cruising app, ~3M MAU, **$100M Match Group investment 4/2026**, expanding into venue event maps | Owns the adult/hookup end natively — compete adjacent, never head-on |

## The five gaps nobody fills

1. **Nobody tracks closures.** The Comeback (2023), R Place (2021), Re-bar (2020), Purr (2018), Seattle Counseling Service (2022), Lifelong's HIV-housing wind-down (2024–25) — all still listed as alive somewhere prominent.
2. **Nobody stamps freshness.** Zero "last verified" dates anywhere in the market.
3. **Nobody covers the adult layer factually.** The Denny Blaine litigation (July 2026 ruling: park stays nude, with new code-of-conduct/ranger/signage requirements) is the biggest queer-Seattle story of 2025–26 and appears in *zero* guides.
4. **Nobody integrates.** Nightlife guides don't do health; health orgs don't do nightlife; GSBA does neither; no one does relocation. One place for all of it doesn't exist.
5. **Nobody serves relocators or the Freeze.** "34 LGBTQ+ leagues" + choruses + climb nights + run clubs exist and are findable only if you already know they exist. And the recurring-*groups* layer (vs one-off events) is the least-contested, highest-differentiation wedge.

## Why the timing is right

- **A relocation wave is happening now.** May 2026: the Seattle LGBTQ Commission petitioned the mayor to declare a civil state of emergency over trans/nonbinary people relocating from red states "by the thousands." Every existing guide is a tourist guide; nobody serves people *moving here*. Our "New to Seattle" pillar meets the moment.
- **Meta is now documented as hostile distribution.** Jan 2025 hate-speech policy rollback; LGBTQ publishers report suppression and are leaving (Press Pass Q, 5/2026). Facebook/Instagram decline for reaching gay audiences isn't a vibe, it's on the record — **owned channels (site + email + ICS feeds) are the defensible play**, which is exactly what we're building.
- **AI search rewards exactly what we'd be.** 65% of Google searches end zero-click; Perplexity cites niche community sites (46.7% of its top citations); Event JSON-LD schema still earns Google's event carousel. Fresh, structured, schema-marked, factually-consistent data is the winning shape — and the losing shape for every incumbent.
- **The platform is already built.** 1,520 tests green, admin CRUD editor live, events schema designed (SUTP Phase 10), scanner architecture decided (D5). The marginal cost of the structured layer is low for us and brutal for every solo curator.

## Honest headwinds

- **The space is contested, not empty**: The Gay Agenda has apps and a head start; QSC is expanding here; Sniffies has $100M and is moving from hookups into venue events. Differentiation = depth (guide + health + relocation + freshness), not just an events list.
- **Single-maintainer labor is the graveyard cause** — the whole design must answer "who keeps this true in month 18?" (the freshness architecture below is that answer, but it needs honest hours budgeting).
- **Some supply is walled**: Facebook groups (housing, some events) and EverOut can't be scraped; Eventbrite killed its search API; Meetup's API is paywalled. The intake plan (below) routes around this legally and cheaply.

# Product Concept

## Pillar 1 — Events (the heartbeat)

The aggregated queer-Seattle calendar: bar/drag nights, parties, meetups, sports seasons, arts, volunteer days, protests, annual anchors. What the research adds to the SUTP Phase 10 design:

- **Recurring events are the spine of this scene** — Seattle queer life runs on "2nd Saturday" patterns (see Appendix C). The events model needs first-class recurring-series support *editorially* even though the DB stores one row per occurrence (a `series` grouping/label so "T4T @ Kremwerk, 2nd Saturdays" is a page, not just instances).
- **A big share of queer events happen at non-queer venues** (breweries, roller rinks, bookstores, climbing gyms) — venue list must not assume "gay bar."
- **Views that match real usage**: *Tonight*, *This Weekend*, *By Scene* (bears/leather/sapphic/trans/QTPOC/sober/40+/geek), *By Neighborhood*, plus month calendar.
- **Intake stack (decided by research, in priority order):** (1) **a mandatory structured submission form as the only human intake** — the NO STRAIGHT PLANS labor hack: "events submitted via email/DM will NOT be considered" converts curation from research into moderation; (2) **ICS/feed ingestion** where venues publish them (Kremwerk has a clean 4-month feed); (3) **Evvnt publisher partnership** (2,200-calendar syndication network The Seattle Times already uses) = free inbound supply; (4) **the AI scanner** (SUTP Phase 13) over venue sites/Eventbrite org pages; (5) **never scrape Facebook/Instagram or logged-in anything** (legal + technical wall; see Phase E notes).
- Categories (D4's list) extend with: **scene tags** (bear, leather, sapphic, trans, QTPOC, sober, elders, geek, furry) orthogonal to activity categories.

## Pillar 2 — The Guide (the moat)

Evergreen, structured directory + editorial. Sections, each with its volatility tier driving refresh cadence (Appendix has full seed data):

1. **Nightlife venue profiles** — the "which bar is for which crowd" layer (crowd/vibe is LOW-volatility, durable content; ownership/status is HIGH — 2025–26 alone: Cuff sold, Queer/Bar sold + remodeling, Neighbours building for sale). Include the **closure graveyard** (R Place, Re-bar, Comeback, Purr…) — history + trust signal + SEO nobody else has.
2. **Scenes** — landing pages per subculture (bears, leather/kink, sapphic, trans, QTPOC, sober, elders, geeks) linking venues + recurring events + orgs. This is how "wholesome to rampantly slutty" coexists without moralizing: every scene gets the same respectful, factual treatment.
3. **Sexual health** — STI testing (Harborview Sexual Health Clinic, the LGBTQ+ Center's free 6-day testing), one-stop PrEP (Kelley-Ross One-Step, 2 locations), DoxyPEP, mpox, HIV care (Madison Clinic), and the **two corrections every competitor gets wrong**: Lifelong's housing programs moved to Bailey-Boushay (2024–25), and Seattle Counseling Service closed in 2022 (successors: Optimism, Integrative Counseling, the Center's 12-session program).
4. **Everyday services** — LGBTQ-affirming primary care, mental health, gender-affirming entry points (Ingersoll's provider directory is the incumbent to link, not fight), dentists, and the GSBA-gap layer: *editorial* "who's actually good" for realtors, barbers, contractors, massage, tattoo, photographers (GSBA answers "who paid dues," we answer "who to hire").
5. **Fitness & rec** — queer climb nights (5+ gyms on a monthly rotation), queer yoga/fitness, SANCA/Emerald City Trapeze (one org since 2023), the gay-gym-scene question (anecdote-tier, present it as dated "what people say" — an un-Googleable content gap).
6. **The adult layer (18+ section)** — bathhouses (Steamworks; Club Z — 50 years old in 2026, building listed for sale 2/2026, watch), sex-positive orgs (Pan Eros, Seattle Erotic Art Festival), naturist groups (SLUGS, Lake Bronson, Tiger Mountain), **Denny Blaine & Howell Park with the actual current rules** — the July 2026 court ruling (park stays nude; code of conduct, ranger staffing, screening vegetation, signage) plus etiquette, each stamped "as of {date}." Cruising beyond the beaches: see Open Question 3 — the one genuinely contested content call.

## Pillar 3 — New to Seattle (the moment)

A relocation guide, not a tourist guide — currently served by *no one* while demand spikes:

- **Neighborhood rundowns** with queer character, 2026 1BR rent ranges (always ranged + dated; cross-source spread is ±$300), and light-rail access. Capitol Hill is still the infrastructure gayborhood (14+ queer venues walkable); White Center is the clearest "second gayborhood"; Tacoma is a real second scene, not spillover. Appendix F has the table.
- **Transit as of mid-2026**: 1 Line Lynnwood↔Federal Way; **2 Line fully connected across the lake 3/28/2026** (Judkins Park station is a big deal for queer renters); West Seattle/Ballard lines still future.
- **Traffic honesty**: Revive I-5 cuts capacity through 2027 (Lynnwood→Mercer ~50–65 min); 520 is tolled; West Seattle is bridge-dependent. "If you're moving here in 2026–27, don't plan a car commute across the Ship Canal."
- **First-90-days checklist**: get on PrEP/testing rails, pick your scenes, join one recurring thing (→ Pillar 4), the paperwork stuff (license, voter reg), where queer housing leads actually surface (the 13k-member Seattle Queer Housing FB group — link out; we don't need to own it yet).
- **A "moving here because of your state" lane** — resources specific to the 2026 influx (legal name-change/document navigation via Ingersoll/Lavender Rights, the 2026 King County Trans Resource & Referral Guide) — highest-need audience, zero competition, deeply on-mission.

## Pillar 4 — Beating the Freeze (the wedge)

The Seattle Freeze is real (UW Psychiatry publishes tips; Axios/Seattle Times cover it as loneliness infrastructure). The known cure is *recurring shared-activity groups, not one-off events* ("it's tough to make real friends through Meetup events alone" is the consistent finding — and summer is the joining season). Nobody aggregates the queer version of this. We make it a first-class product surface:

- **The recurring-groups directory**: 34+ LGBTQ+ sports leagues (softball since 1979, ORCA swim since 1984, Frontrunners since 1985, Quake rugby, 3 competing kickball operators…), choruses (SMC/SWC, Rainbow City Performing Arts), climb nights, book clubs (Charlie's Queer Books runs 4), gaymers, figure drawing, faith communities, **recovery meetings (gay AA exists and appears on no queer calendar anywhere — a genuinely unserved wholesome layer)**, professional groups (Out in Tech's 2nd-Wed happy hour), volunteer orgs (VolunQueers, Lambert House, GenPride).
- **"Pick your onramp"** — a guided path by personality/interest (sporty / arty / nerdy / sober / outdoorsy / 40+ / BIPOC / trans / new-in-town) → 2–3 matched recurring groups + this month's low-barrier entry events (e.g., Frontrunners' Monday 3-mile Cal Anderson mini-run). Structured "join something" advice beats the generic listicle every outlet writes.
- This pillar is also the honest bridge to Knotty Yoga: a wellness/fitness/acro studio *belongs* in a directory of recurring queer movement spaces — listed by the same rules as everyone else.

## Pillar 5 — Community (the growth path, gated)

Strict order, each gated on traction + capacity: (1) event submissions with review queue (the form IS the community feature); (2) "report stale / is this still open?" flags — community-powered freshness; (3) newsletter replies → reader tips lane; (4) org/venue self-service claiming of their listings; (5) forums *if ever* — build searchable/indexed (Discourse-style) not Discord (research: Discord = "whiteboard erased daily," zero Google/AI visibility; there's also no dominant Seattle gay Discord to compete with — but see Open Question 7); (6) activity-partner matching (chess/tennis/D&D "looking-for-group") — a natural extension of Pillar 4 with real differentiation; (7) **personals/dating: probably never** — FOSTA/SESTA killed Craigslist personals; two live §230-sunset bills in Congress (12/2025–2026); Sniffies owns the space with $100M behind it. If ever revisited: verified identity + bright-line no-commerce rules + real moderation staffing + counsel first.

## The freshness architecture (the differentiator, cross-cutting)

This is the product's core claim and the answer to "why won't this rot like everything else":

1. **Every listing carries `last_verified_at` + `status`** (open / closed / unverified / seasonal) — surfaced publicly ("Verified July 2026"). Closed things stay visible as history, marked closed. No other property does either.
2. **Volatility tiers drive re-verification cadence** (from research): bar event lineups = weekly; ownership/open-closed = monthly; Denny Blaine rules = monthly during litigation; bathhouses = quarterly; health programs/income thresholds = semi-annual; orgs/leagues/clinic locations = annual; rents = quarterly, always ranged.
3. **The AI scanner doubles as verifier, not just discoverer** — per-source cadence from the tier; checks: URL still 200s, event dates sane, "permanently closed" signals, hours changed. Flags diffs into the admin review queue rather than auto-publishing. (Feeds the SUTP Phase 13 design.)
4. **Monitored editorial sources** (CHS, SGN, SGS) for ownership/closure news — human-in-the-loop, low volume.
5. **Community flags** ("this closed," "moved," "new night") as tier-zero signal once there are users.
6. **Single-intake submission rule** keeps human labor = moderation, not research.

## Voice, spectrum, and the adult section

- **Editorial voice**: warm, direct, factual, sex-positive, zero moralizing in either direction — the bathhouse page and the chorus page get the same respectful treatment. House style: state facts + etiquette + safety plainly; date every volatile claim; never euphemize, never leer.
- **Spectrum handling is structural, not tonal**: scene tags + an 18+-gated section for the adult layer (bathhouses, beaches/cruising, kink). WA has **no age-verification law as of mid-2026** (HB 2112 didn't pass; FSC v. Paxton greenlit state AV laws, so watch) — a simple interstitial suffices today and keeps the site far below any "1/3 sexually explicit" threshold. Everything stays guide-level factual, nothing explicit.
- **Privacy by design is a legal requirement here, not a nicety**: Washington's **My Health My Data Act** (private right of action; first class action 2/2025) squarely covers sexual-orientation- and sexual-health-adjacent data + geolocation. Design consequence: no ad-tech pixels, no third-party trackers, privacy-friendly analytics only, no geofencing, minimal accounts data. This is cheap for us and a real trust differentiator ("we can't leak what we don't collect").

## The Knotty Yoga relationship (trust model)

Research-backed pattern for a business-owner-run community site: **full transparency beats stealth** (FTC endorsement rules + audience-trust studies + QSC's "100% independent, funded by supporters" tonal template):

- Persistent site-wide footer: "Founded and funded by [Mason / Knotty Yoga]."
- A public **listing policy / editorial independence page**: how listings get in (free, criteria-based), how sponsors are labeled ("Presented by"), and the bright line: **we never rank our own business in any 'best of' we publish** — Knotty Yoga appears in directories under the same rules as everyone else.
- The marketing payoff is structural, not promotional: the studio is *venue + org + recurring-groups host* inside the most useful queer site in the city, plus "Presented by Knotty Yoga" on the newsletter/flagship guides. Low-key by design, durable because disclosed.

## What we deliberately do NOT build

Geneva/Guilded-style platforms (both dead), WhatsApp Communities (50-group cap), a Discord-first community (unindexed), Facebook/Instagram scraping (walled + ToS + Meta hostility), Eventbrite/Meetup API dependence (killed/paywalled), personals at v1 (see Pillar 5), our own ticketing (link out), tourist content GSBA already owns (World Cup guides).

# How It Rides the Platform

The technical plan stays in [[Setting up the project]]; this doc feeds it product requirements. The mapping:

| Product piece | Platform piece | Status |
|---|---|---|
| Events pillar | SUTP Phases 10–11 (venues/event_sources/events/categories schema + public UI) | Designed, not started |
| Intake: submission form | Phase 10 endpoints + a public submit page (origin=user_submitted, pending) | Add to Phase 10/11 scope |
| Intake: ICS/Evvnt/scanner | SUTP Phases 12–13 (scheduler + AI scanner); scanner gains the **verifier** role + per-source cadence | Design input for Phase 13 |
| Guide/directory | New app tables (places/organizations/listings + scene tags + `last_verified_at`/`status` on everything) via the same CRUD/admin/photos machinery | New — this doc's Phase C |
| Guide editorial pages | Static/CMS-lite pages in the SPA (+ site_meta) | New — Phase C/D |
| Scene/category tags | D4's event_categories + a parallel tag vocabulary shared by events and listings | Extend Phase 10.1 |
| Newsletter | Scheduler job (Phase 12 catalog) + mail infra already proven; list provider TBD | New — Phase E |
| Freshness stamps + closure graveyard | Columns + status enums + public rendering | New — Phase C schema |
| Multi-city future | D8 communities-as-tenants — Seattle is community #1; Tacoma/Portland are `--create_tenant` away; the `city.antifreeze.com` domain lean maps 1:1 onto per-community CloudFront | Already architected |
| SEO/AI surface | Event JSON-LD, sitemap, per-category ICS feeds, Bing indexing (ChatGPT cites Bing), robots.txt allowing OAI-SearchBot/PerplexityBot | Add to Phase 11/15 launch checklist |

Implementation conventions inherit SUTP: lower layers first, tests for everything testable, Claude runs the Linux docker gates, Mason does git and Windows spot-checks.

# Phased Roadmap

Lettered phases (A–H) to avoid colliding with SUTP's numbered technical phases. Within each phase, work items run lower-layer-first: data/taxonomy → server → client → content → distribution.

## Phase A — Research & synthesis (this document)

### A.1 Research the landscape
- [x] Events/orgs landscape deep-dive (3 research passes, 7/31/2026)
- [x] Guide/directory content inventory with volatility ratings (7/31/2026)
- [x] Peer models + legal + marketing/SEO reality (7/31/2026)
- [x] Premise check on seattlegayscene.com (alive-but-hollow; reframed the pitch) (7/31/2026)

### A.2 Synthesize
- [x] Organize into pillars + freshness architecture + roadmap (this doc, 7/31/2026)
- [x] Seed inventories compiled (Appendix) — doubles as future `event_sources`/`venues` seed data (7/31/2026)
- [x] Open Questions drafted with recommendations (7/31/2026)

## Phase B — Validate & decide (friend review round)

### B.1 Review
- [ ] Mason reads + marks up this doc; answer Open Questions inline (esp. Q1 name, Q2 audience, Q3 adult stance, Q6 labor)
- [ ] Friend review (Levi, Caleb + 2–3 target-audience friends incl. at least one recent transplant and one scene-connected person)
- [ ] A 5-question survey for ~10 gay/queer Seattle friends: where do you find out about things now? what can you never find? would you use X? (cheap demand validation)

### B.2 Decisions out of review
- [ ] Lock name/brand + domain (feeds SUTP Q9; unblocks nothing until Phase 15, so no rush — but brand affects everything editorial)
- [ ] Lock audience framing + editorial voice one-pager
- [ ] Lock adult-content stance (Q3) — determines whether Phase D includes the cruising layer
- [ ] Lock the v1 scope cut: recommendation = Events MVP + Nightlife/Scenes/Health guide sections + Freeze directory; defer services-directory editorial and full relocation guide to fast-follow
- [ ] Set the labor budget honestly (hrs/week per person) + the month-18 sustainability answer (Q6)

## Phase C — Content & data foundations (before more code)

### C.1 Taxonomy (lowest layer of the content system)
- [ ] Finalize category + scene-tag vocabularies (events and listings share them) — start from D4 seed + research categories
- [ ] Define listing types + fields: venue, organization, recurring_series, place (park/beach), service_listing, guide_page — each with `status`, `last_verified_at`, scene tags, neighborhood
- [ ] Define the volatility tier enum + per-tier re-verification cadence (from research; see freshness architecture)

### C.2 Seed registries (from the Appendix, cleaned)
- [ ] `event_sources` registry v1: ~40 sources typed by kind (ICS feed / venue site / Eventbrite org / editorial monitor / submission) + scan_hints + cadence tier
- [ ] Venue registry v1: ~35 queer venues + ~25 queer-programming venues, with crowd profiles + status (incl. the closure graveyard list)
- [ ] Organization registry v1: sports leagues, choruses, orgs, recovery, faith, professional (~80 entries)
- [ ] Health/services registry v1: the corrected list (Lifelong→Bailey-Boushay, SCS closed, Kelley-Ross One-Step, etc.)
- [ ] Annual calendar skeleton (the ~30 anchor events with months)

### C.3 Editorial foundations
- [ ] Voice & style one-pager (incl. the date-every-volatile-claim rule + disclaimer patterns for the adult section)
- [ ] Listing policy / editorial independence page draft (the trust model)
- [ ] Page templates: venue profile, org profile, scene landing, guide article (what fields render where)

## Phase D — Guide v1 (schema → server → client → content)

### D.1 Schema + server (follows SUTP conventions: db_schema pair → table_helpers → business logic → endpoints, tests at each layer)
- [ ] Tables: `places`, `organizations`, `listings`(or unified `directory_entries`), `scene_tags` + assignments, shared with events; `last_verified_at`/`status` columns; admin metadata + allowed_tables + `manage_guide` permission
- [ ] Public read endpoints (by scene / neighborhood / type) + admin CRUD rides the framework editor
- [ ] Freshness fields surfaced in all public payloads

### D.2 Client
- [ ] Scene landing pages, venue/org profile pages, neighborhood pages; "Verified {date}" + closed-marker rendering; 18+ interstitial for the adult section
- [ ] Guide article rendering (editorial pages)

### D.3 Content (publish in durability order — durable first)
- [ ] Nightlife venue profiles + crowd guide + closure graveyard
- [ ] Scene landings (bears/leather/sapphic/trans/QTPOC/sober/elders/geek)
- [ ] Sexual health section (the corrected 2026 facts)
- [ ] Freeze directory: leagues/choruses/clubs/recovery/faith/professional + "pick your onramp"
- [ ] Adult section per Q3 decision (bathhouses, Denny Blaine current-rules page with dated stamps)
- [ ] New to Seattle: neighborhoods + transit + first-90-days (can trail as fast-follow)

## Phase E — Events MVP live (rides SUTP Phases 10–13)

### E.1 Scope additions this doc feeds into SUTP Phase 10/11
- [ ] `series` grouping for recurring events + "Tonight / This Weekend / By Scene" views
- [ ] Public submission form (single-intake rule stated loudly) + admin review queue
- [ ] Event JSON-LD, sitemap, per-category ICS export feeds ("subscribe to bear events" is a killer distribution hack)
- [ ] Seed `event_sources` from C.2; first two weeks of events hand-entered via admin CRUD to bootstrap

### E.2 Scanner as verifier (feeds SUTP Phase 13 design)
- [ ] Per-source cadence from volatility tier; verify-mode checks (URL alive, dates sane, closure signals) → diff review queue
- [ ] Guide listings get re-verification passes on their tier cadence, not just events

## Phase F — Distribution & newsletter

- [ ] Newsletter signup live from the first public deploy; weekly "This Week in Queer Seattle" digest starts when events data is 2 weeks solid (the retention spine — beehiiv-class provider, owned list)
- [ ] Bing Webmaster + Google Search Console + rich-results validation (ChatGPT cites Bing; Perplexity rewards fresh structured niche sites)
- [ ] Bluesky + Instagram presence (IG = discovery only, never infrastructure); venue/org cross-promo ("we listed you, here's your badge/link")
- [ ] Coopetition outreach: Sapphie Taffy (credit/ingest/collaborate — their single-point-of-failure is our pitch), QSC Seattle, SGN/SGS (they need a working calendar; we need editorial reach), Evvnt publisher signup
- [ ] Reddit presence policy (helpful answers with links where rules allow — AI-citation play; verify current sub rules first)

## Phase G — Community v1

- [ ] Stale/closed/new-info flag button on every listing → admin queue
- [ ] Org/venue self-service claim + edit-suggestion flow (still admin-approved)
- [ ] Reader-tips lane from newsletter replies
- [ ] Event photos (platform photo pipeline; Metro Weekly's scene-photo archive is the proven engagement moat — with consent/no-outing policy)

## Phase H — Community v2 (traction-gated, each item its own go/no-go)

- [ ] Activity-partner / looking-for-group matching (chess, tennis, D&D, climbing partners) — Pillar 4's natural extension
- [ ] Forums decision per Q7 (if yes: indexed/searchable, tight scope — e.g., housing + newcomers + LFG boards first, not general chat)
- [ ] Services marketplace (massage therapists, trainers offering services) — needs listing-policy + liability review
- [ ] Personals: default **no** (see Pillar 5); revisit only with counsel + §230 clarity

# Open Questions

*(Numbered for inline answers, house style. Recommendations included so "agreed" suffices.)*

1. **Public brand vs. platform name.** CommunityFinder is the repo/platform. Is it also the consumer brand, or do we want a Seattle-punchy brand on top (the winners are punchy: "The Gay Agenda," "Queer Social Club")? Your `gay.seattle.antifreeze.com` lean is interesting precisely because **Antifreeze is a real brand hook** — it names the Freeze-beating mission, and `city.antifreeze.com` maps exactly onto the multi-community tenancy model (Seattle first, Tacoma/Portland later). Other directions: descriptive-SEO ("Gay Seattle Guide"), emerald-themed, or reviving respect for the ancestor ("…scene"-adjacent names risk confusion with the living SGS). *Rec: treat naming as a friend-review exercise with Antifreeze as the front-runner; don't block anything on it.*
2. **Audience framing.** "Gay Seattle" (gay-male-forward, your marketing demographic) vs "Queer Seattle" (full spectrum)? Research reality: the curation energy and unserved demand skew queer/trans/QTPOC (Sapphie Taffy's audience, the relocation wave), and lesbian/sapphic Seattle has exactly one bar but a huge events layer. *Rec: inclusive "queer Seattle" scope with honest gay-male editorial depth where that's what we know — scene tags make breadth structural rather than performative. Your call; it shapes name, voice, and how the site reads.*
3. **The adult layer's outer edge.** Bathhouses + Denny Blaine/Howell + naturist groups + kink orgs feel clearly in (factual, publicly documented, 18+-gated). The contested call is **cruising listings beyond the beaches** (parks/trails/etc.): global travel guides list them with disclaimers; *local community* guides universally don't (venue relationships, sponsors, civic legitimacy — and Sniffies owns real-time cruising natively with Seattle-density data). Options: (a) full listings with etiquette/safety/legal framing; (b) **the middle path: cover the *documented* places (Denny Blaine, Howell, Volunteer Park as history) + etiquette/safety/law editorial + "the apps own real-time" honesty**; (c) omit beyond beaches. *Rec: (b) at launch — it delivers the non-moralizing spectrum promise without becoming a liability/partnership problem; revisit (a) once the site has standing.*
4. **Knotty Yoga visibility.** Invisible / footer-credit / fully open "founded by." *Rec: fully open (footer + about page + listing policy) — research says stealth marketing is the trust-killer and disclosure is the durable version of the marketing benefit.*
5. **Geographic scope at v1.** Seattle city / King County / metro incl. Tacoma+Eastside? *Rec: Seattle + King County listings with a Tacoma "worth the trip" section; Tacoma becomes its own community via tenancy when someone local can own it.*
6. **The labor model (the existential one).** Who does the weekly moderation/verification hours in month 18? Options: (a) Mason+Levi+Caleb rotating editor-of-the-week; (b) scanner+review-queue only, accept thinner editorial; (c) budget a paid part-time curator once there's any revenue; (d) recruit 2–3 volunteer scene-editors (bear scene, sapphic scene, arts) with named credit. *Rec: (a)+(d) designed into the admin tooling from day one (review queues are multi-user already); decide the trigger for (c).*
7. **Forums: build, bridge, or skip.** Research is unambiguous that Discord kills compounding value and there's no dominant Seattle gay Discord to bridge to — but forums have real cold-start + moderation costs, and the §230 sunset bills add risk to all UGC. *Rec: skip at launch; revisit at a concrete traction gate (e.g., 1,000 newsletter subs) starting with narrow boards (housing / newcomers / looking-for-group), never general chat.*
8. **Newsletter platform + cadence.** Weekly Monday digest (The Gay Agenda's cadence) vs Thursday (weekend-planning)? Provider (beehiiv/Buttondown/self-host)? *Rec: Thursday weekly; pick provider at Phase F; signup form from first deploy regardless.*
9. **Legal counsel timing.** MHMDA (health-adjacent data + private right of action), ToS/privacy pages, the adult-section posture, and eventually UGC. *Rec: minimal-tracking design now (defuses most MHMDA exposure by construction); one counsel review before first public deploy; second review gate before any UGC feature.*
10. **Success metrics for v1.** What makes this "working" by month 6? *Rec: (a) coverage: ≥90% of a hand-audited week's queer events listed; (b) freshness: 100% of listings verified within their tier cadence; (c) audience: newsletter subs (target 500 by month 6) + weekly returning visitors; (d) the marketing payoff: Knotty Yoga referral traffic/signups tracked honestly. Agree/adjust targets.*

# Appendix — Seed Inventories (research snapshot, 7/31/2026)

*Everything below is point-in-time research data with status flags — (UNVERIFIED) items need a check before publishing. This appendix is the raw material for Phase C.2's registries.*

## A. Event-source registry candidates (~by intake type)

**Structured/feed-friendly:** Kremwerk complex calendar (clean, 4-month horizon) · Eventbrite org pages (Queer Mountaineers, PNW Black Pride, Queer Figure Drawing, Vertical World) · Meetup group pages (~20 queer groups; API paywalled, pages scannable) · leagueapps (Queer City Sports) · RunSignup · Do206 · Evvnt network (join as publisher) · venue Google calendars where they exist.
**Scanner targets (venue/org sites):** Julia's/Le Faux · Unicorn · Neighbours · Wildrose · Cuff · Queer/Bar · Massive · Diesel · CC's · Lumber Yard · Southgate Rink · Rough & Tumble · Charlie's Queer Books · seattlechoruses.org · rainbowcity.org · league sites (ECSA, Quake, ORCA, Frontrunners, OutLoud, Pride Sports, Rain City Soccer, Cascade Flag, SPHA, Cheer Seattle, Tennis Alliance…) · org calendars (Seattle Pride's year-round aggregated calendar, GSBA, GenPride, Gay City, Ingersoll, Lambert House, PFLAG) · seattleprideguide.com · queersocialclub.com/events-seattle · thegayagenda.fyi/seattle · sapphietaffy.com (with consent/credit).
**Editorial monitors (closure/ownership news, human-reviewed):** CHS · SGN · Seattle Gay Scene · The Stranger/EverOut (read, don't scrape).
**Blocked/never:** Facebook groups (auth-walled; Queer Exchange Seattle, Seattle Queer Housing 13k) · Instagram accounts (@seattlequeerevents, @bloom.seattle, @thesocialqueer 7.7k, @seattledykealliance) — *monitor manually for leads, never scrape; IG/FB event traffic is the biggest supply we route around via the submission form.*

## B. Venues (status as of 7/2026)

**Queer venues, open:** Pony (dive/arty/cruisy; site SSL expired — IG is live channel) · The Cuff Complex (leather/bear; sold 3/2025 to Scott Walent) · Queer/Bar (drag-forward; sold 6/2026 to Cuff owners; July 2026 "summer refresh" — HIGHEST-VOLATILITY ENTRY) · Seattle Eagle (oldest leather bar) · Diesel (bears) · CC's/CC Attle's (mixed, 1st-Fri Leather Social) · Madison Pub (low-key sports/pool) · Wildrose (est. 1984, oldest US lesbian bar; fragile-treasured) · Neighbours (est. 1983 dance club; **building listed for sale ~$7M — watch**) · Massive (2023, 3-floor dance) · Union (cocktail lounge — Capitol Hill, NOT SODO as Seattle Times claims) · Crescent Lounge (karaoke dive since 1940s) · Unicorn/Narwhal (carnival/drag; Mimosas Cabaret Sundays) · Julia's on Broadway/Le Faux (drag dinner theatre) · Kremwerk+Timbre+Cherry (queer-owned electronic, best calendar) · Lumber Yard Bar, White Center (bears/neighborhood; arson 2021 → reopened 11/2022) · Boombox (White Center) · Changes (Wallingford — only gay bar north of the Ship Canal).
**Queer-programming venues:** Southgate Roller Rink (Wed 21+ Pride Skate) · Chop Suey (Flammable Sundays) · Comet (Caliente Thu) · Rough & Tumble, Ballard (women's-sports bar; queer trivia; Out in Tech) · Stoup Cap Hill (Board Gayme Mon) · El Sueñito (book club/run club/karaoke) · Metier (bingo) · Skylark · Rendezvous · Raygun · Asylum Collective (sapphic goth) · Gallery Erato · Phoenix Comics (queer board games) · West Seattle Bowl (3 queer leagues) · Great American Casino Tukwila (Royal Flush drag) · ~15 more in research notes.
**Closure graveyard (all still listed live somewhere):** The Comeback (2023) · R Place (2021, lease lost) · Re-bar (2020) · Purr (2018) · Basic Plumbing/Tribe (2012) · Double Header (2015) · Supernova SoDo queer status (UNVERIFIED — absent from 2026 guides).
**Adult:** Steamworks (1520 Summit, 24/7) · Club Z (1117 Pike, est. 1976 — 50th anniversary 2026; **building listed for sale 2/2026**) · Center for Sex Positive Culture (listed CLOSED — needs primary confirmation) · Pan Eros Foundation (active; Seattle Erotic Art Festival May 1–3, 2026) · Doghouse Leathers (retail/community hub).

## C. The recurring-events skeleton (proof the scene runs on recurrence)

**Weekly:** Mon Board Gayme Night (Stoup) · Tue Dum Top Trivia (Cuff) + Drag Queen Bingo (Unicorn) · Wed Pride Skate (Southgate) + Seattle Gaymers (CC's) + Queeraoke (Lumber Yard) · Thu karaoke (Wildrose) + Caliente (Comet) · Fri Le Faux + MX (Queer/Bar) + Fridays Are a Drag (Neighbours) + Lashes (Unicorn) · Sat Bad Girls Brunch (Julia's) + bear/leather night (Cuff) · Sun Mimosas Cabaret (Unicorn) + Flammable Sundays (Chop Suey) + Bear Social (Lumber Yard) + Pride League bowling.
**Monthly highlights:** 1st Fri The Gathering (PNW Black Pride @ CC's) · 1st Sat Safeword (Kremwerk) · 2nd Thu Centerfold trans cabaret (Queer/Bar) · 2nd Fri Queen4Queen (Pony) · 2nd Sat **T4T** (Kremwerk — longest-running all-trans drag show) + Queerly Beloved (Jet City Improv) · 3rd Fri Lesbian Night (Kamp) · 4th Sun Judy (Queer/Bar) · last Fri Qu-Art sober artists (PUSH/PULL) · + ~40 more in research notes (Sapphie Taffy documents 175+).
**Climb-night rotation:** 1st Thu Vertical World North · 2nd Mon Uplift · 2nd Fri Climb NORA · 3rd Mon Castle · 3rd Thu SBP Poplar · 4th Mon Half Moon · + trans/BIPOC/women's nights.

## D. Organizations (Freeze-directory seed)

**Sports (34+ leagues):** ECSA softball (47th year, Sundays North SeaTac) · Rain(bow) City Softball (women/trans/NB) · Seattle Quake RFC · Mudhens rugby · Frontrunners (est. 1985; Mon Cal Anderson mini-run = perfect low-barrier onramp) · ORCA Swim (est. 1984) · Otters water polo (1990) · Different Spokes + Outspoken cycling · OutLoud Sports (kickball/dodgeball/volleyball/bowling/darts/pickleball) · Pride Sports Seattle · Queer City Sports · Rain City Soccer (2000) · Cascade Flag Football · Seattle Pride Hockey Assoc · Seattle Basketball Club · Seattle Tennis Alliance · Cheer Seattle · 3 West Seattle Bowl leagues · SkiBuddies · Olympic Yacht Club · OutVentures (outdoors, decades old) · Queer Mountaineers (newsletter+Discord+IG) · Team Seattle umbrella. *(No Stonewall Sports chapter — gap/opportunity noted.)*
**Arts:** Seattle Men's/Women's Chorus + Captain Smartypants · Rainbow City Performing Arts (75 musicians; 2026 Benaroya shows) · Three Dollar Bill Cinema (**SQFF + TRANSlations both CANCELLED 2026, "indefinite pause"** — a hole in the annual calendar and a partnership opening) · Intiman (queer-heavy seasons + Cabaret) · ArtsWest · Jet City Improv · queer figure drawing (Elva Bennett; VALA Redmond 3rd Sun; Pride Across the Bridge free sessions) · Charlie's Queer Books (Fremont; 4 book clubs) · Gay History Tour (Ghost Alley Espresso).
**Community/support:** Gay City / Seattle's LGBTQ+ Center (rename in flux; free 6-day STI testing; 12-session counseling; **center closed for parts of July 2026 — watch stability**) · Lambert House (youth) · GenPride/Pride Place (elders) · Entre Hermanos · UTOPIA WA · Lavender Rights Project · Ingersoll (est. 1977; provider directory + Wed/Sat groups) · Pride Foundation · PFLAG Seattle (5 meetings/month) · Camp Ten Trees · Peer Seattle · Seattle Dyke Alliance · SBWN · Queer the Land.
**Recovery (on no queer calendar anywhere):** Gay Men in Recovery + Gay & Lesbian Beginners AA (seattleaa.org / Meeting Guide app) · Qu-Art sober night · Queer Grief Club (virtual).
**Faith:** Queer Dharma (Shambhala, 1st/3rd Mon) · affirming-congregation directories (pridepagesseattle.com/church-spiritual, Peer Seattle).
**Professional:** Out in Tech Seattle (2nd-Wed happy hour @ Rough & Tumble) · Lesbians Who Tech · Seattle Queer in Tech meetup · GSBA networking.
**Meetup social:** Another Gay Social Club (3,899 members — likely the "Gay People in Seattle gatherings" you meant; 2nd Mon + last Fri) · Seattle Queer & Ally Hiking (1,823) · Seattle Gaymers · Queer Geek! · West Seattle LGBT+ · Banter Date (age-segmented men's socials) · Aces & Aros · ~15 more.

## E. Health & services (the corrected 2026 facts)

**Testing/PrEP:** Harborview Sexual Health Clinic (908 Jefferson, walk-ins by 4pm; walk-in mpox vaccine) · the Center's free testing 6 days/week + mailed kits · **Kelley-Ross One-Step PrEP** (in-Center + 7th&Madison; single-visit start; TelePrEP statewide) · PrEP DAP (2026 income limit $6,650/mo) · DoxyPEP per 2025 King County guidelines · Planned Parenthood (U District/CD/Northgate; hormone care at all centers).
**HIV care:** Madison Clinic at Harborview (largest in PNW) · Bailey-Boushay House (**now holds the former Lifelong housing clients — 2024–25 handoff every guide misses**).
**Primary/affirming:** Country Doctor (**rebranding under "Seattle Roots Community Health"**) · Swedish LGBTQIA+ Program (Dr. Kevin Wang) · UW Medicine trans health program (full surgical menu) · Capitol Hill Medical · Cedar River Clinics · named MDs list in research notes.
**Mental health:** **Seattle Counseling Service CLOSED 2022** (the #1 stale-data trap in the market) → Optimism Counseling, Integrative Counseling (Fremont), the Center's 12-session program, NAMI referral layer.
**Gender-affirming entry:** Ingersoll provider directory + 2026 King County Trans Resource & Referral Guide.

## F. Neighborhoods (relocation-guide seed; rents = 2026 ranges, always date-stamp)

| Neighborhood | Queer character | ~1BR 2026 | Link |
|---|---|---|---|
| Capitol Hill | The gayborhood — 14+ venues, Center, clinics; loud, young, spendy | $1,990–2,280 | 1 Line Capitol Hill |
| First Hill | Quiet medical-adjacent Hill access | ~CH −10% | streetcar |
| Central District | Gentrified-historic; Denny Blaine/Howell at its edge | (verify) | **2 Line Judkins Park (new 3/2026)** |
| Beacon Hill | Diverse, cheaper, growing queer presence | ~$1,680 | 1 Line |
| Columbia City | Walkable, diverse, South-End Pride energy | ~$1,950 | 1 Line |
| Georgetown | Industrial-arty, cheapest in-city | ~$1,440 | bus only |
| White Center | **The second gayborhood** — Lumber Yard, Boombox, Southgate, queer-owned cluster, own Pride | ~$1,970 | **no rail** — the drawback |
| West Seattle | Beachy/residential; bridge-dependent | (verify) | no Link yet |
| Ballard/Fremont | Rough & Tumble + Charlie's + SBP queer nights — quiet queer density | $2,070–2,240 | bus |
| Wallingford | Changes; quiet | (verify) | near U District Sta |
| U District | Student, cheap-ish; PP clinic | (verify) | 1 Line |
| Shoreline | Suburban, newly rail-rich | (verify) | 1 Line ×2 |
| Burien | 10th Pride 2026, first parade; growing | ~$1,750–1,960 | bus |
| Renton | Affordable; Cedar River Clinics | ~$1,530–1,900 | bus |
| Tacoma | **Real second scene**: The Mix + Silverstone on St. Helens, Hilltop/6th Ave, own Pride | well below Seattle | T Line, Sounder |

**Transit as of 7/2026:** 1 Line Lynnwood↔Federal Way (41 mi); 2 Line fully connected 3/28/2026 (Redmond↔Lynnwood via downtown; first light rail on a floating bridge); Pinehurst/NE 130th expected 2026. **Traffic:** Revive I-5 through 2027 (Lynnwood→Mercer 50–65 min); 520 tolled; West Seattle Bridge fine but single-point-of-failure. World Cup summer 2026 compounds everything.

## G. Annual calendar skeleton (anchors)

May: Seattle Erotic Art Festival (5/1–3) · SIFF. June: Pride in the Park (6/6) · Burien Pride (6/5–7) · GSBA Luncheon · Trans Pride (6/26, Volunteer Park) · Queer/Pride Festival (6/26–28, ticketed) · PrideFest Capitol Hill + Dyke March + Indigiqueer Fest (6/27) · Parade + PrideFest Seattle Center (6/28, largest free Pride fest in US) · White Center Pride · Wildrose/Cuff/Kremwerk/Neighbours Pride runs. July: Tacoma Pride · Gardens & Gems (Gay City). Aug: Alki Beach Pride (8/1+8/15) · Dining Out For Life (8/13) · **PNW Black Pride Weekend (8/20–23)** · Latinx Pride (9/19 — Sept) · Capitol Hill Block Party (8/7–9, adjacent). Sept: Kremfest (9/25–27) · AIDS Walk NW (date UNVERIFIED). Oct: ~~SQFF~~ (cancelled 2026). Nov–Dec: chorus holiday shows · Jinkx & DeLa · Intiman Naughty List · World AIDS Day (12/1). **Defunct/unverified:** Red Dress Party (ended 2024) · Bat n' Rouge (UNVERIFIED) · Gay Bingo (UNVERIFIED) · NW Leather Celebration (2026 was Sacramento).

## H. Denny Blaine current-state (the content-moat exemplar)

Clothing-optional queer beach, decades-standing. Dec 2023 playground proposal defeated · spring 2025 neighbor lawsuit · July 2025 abatement order (signage, fenced clothing-optional zone) · **~7/15/2026 ruling: park stays open and stays nude**, with required code of conduct, increased ranger staffing, vegetation screening, explicit signage; police calls down 39% since abatement; topless legal parkwide (WA law). Appeal likely; **rules will shift season to season → a dated "current rules + etiquette" page, refreshed monthly during litigation, is exactly the kind of content nobody else will maintain.** Howell Park: smaller, hidden, historically "more for the boys," no facilities. Volunteer Park: treat as history content.

# Change Log

- **7/31/2026 (v0.1)** — Initial brainstorm synthesis from Mason's Overview + three research passes (events landscape, guide/directory content with volatility ratings, peer models/legal/marketing). Premise corrected (SGS alive-but-hollow; contested space). Product concept organized into 5 pillars + freshness architecture + trust model; lettered roadmap A–H dovetailing with [[Setting up the project]]; 10 Open Questions with recommendations; seed inventories appended.