---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/1/2026
Version: 0.4
tags:
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

There is a website: https://seattlegayscene.com/ that used to be kind of the place to go to see what upcoming events were happening in the world of gay events in Seattle. It's clearly a static, manually maintained website as it always frequently had events listed that had already happened but it was moderately well maintained. It's been largely unmaintained now for a couple of years minus a little effort during pride month.

I am building out a spa, massage space, and fitness / acrobatics studio. I have a large gay clientele. I need to be able to market and gays are a key demographic. Very few gays are in Facebook these days. Google has diminished as an ad platform. Instagram has some relevance but is declining and hard to differentiate yourself on it.

I'd like to have a site that people are drawn to that I can build community but also use for low key doing my own marketing for free.

I'd like to cover what seattlegayscene did, mostly bar events, but do a lot more. I'd love to list all LGBT relevant events and opportunities. The networking things on meetup. The monthly Gay People in Seattle gatherings. Theatre events, pride things, gay figure drawing, the sports leagues, and any other ideas.

I'd like to cast a wider net though. I'd love to have a guide to being gay in Seattle. List gay gyms, doctors, dentists, contractors. Where to get STD tested. Have information about the crowds of each of the gay bars. List the nude beaches. List bathhouses. List cruising spots. Generally list the assorted gay offerings the city has to offer. Please research and think deeply to help me think of ideas here. I also don't want to moralize here. I want to span the spectrum from family friendly / wholesome to rampantly slutty.

I'd also love a new to Seattle guide that lists the various neighborhoods and gives rundowns of the various options. Proximity to light rail. Talks about the traffic.

Seattle is also not a social / friendly city. There is a widely known phenomenon called the "Seattle Freeze" that unfortunately is very real. I'd like to have a guide on overcoming this. Suggestions like joining a volunteer event, sports league, Seattle Men's chorus, etc. I'd love help think of ideas here.

I think an eventual growth path would be community forums. Letting various people offer services like gay massage therapists, ways for people to make friends, people looking for activity partners (chess, tennis, DnD, board games, etc.). Maybe even a dating / personal section.

This is a lot to be honest. Can you help me organize my thoughts here and give suggestions for things you can think of and help organize this set of ideas into more of a cohesive plan? I'm not really ready to move to implementation or design yet. I'd like to keep brainstorming with your help so please think very deeply and help with this so I can put something together to have friends review before moving on to an actual implementation plan.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# The Pitch (v0.2 — for friend review)

**Antifreeze: a maintained, structured home for gay Seattle — every event, every recurring group, every venue, every resource, from drag brunch to bathhouse, with a freshness guarantee nobody else can make.** Events are the heartbeat, the city guide is the moat, the "beat the Seattle Freeze" recurring-groups layer is the wedge (and the brand: *Antifreeze is the cure for the Seattle Freeze*), and gated community features grow on top once trust exists. Scope, decided 8/1: **gay men, King County (Eastside very much included)** — done deeply and honestly, with the big LGBT anchor events listed, sister scenes credited and linked out to their own curators, and the brand architecture ready for sister audiences and cities later (see Brand & Naming). It runs on the CommunityFinder platform we already built (auth, admin CRUD, photo pipeline, AI-scanner architecture), which is purpose-built to solve the exact thing that has killed or calcified every predecessor: **the maintenance labor of keeping a structured city guide true.**

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

# Brand & Naming (Q1 — decided direction, 8/1/2026)

**Decided: the consumer brand is Antifreeze; CommunityFinder stays the platform/repo name.** Your expansion instinct is the whole architecture: a two-axis scheme — **`{audience}.{city}`** — gay first, other underserved communities later, other cities later. That maps 1:1 onto D8 communities-as-tenants: each audience+city pair is one tenant (own database, own CloudFront distribution, one shared server + SPA bundle). "CommunityFinder" as the engine behind an Antifreeze-branded network now reads exactly right.

**Domain reality (checked via registry RDAP, 8/1/2026; Mason's constraint: .com only). A registry 404 = "appears unregistered" — that's not a reservation, so buy before socializing the name widely:**

| Domain | Status |
|---|---|
| **`antifreezeseattle.com`** | ✅ **Appears unregistered — Mason's exact proposal works** |
| `theantifreeze.com` | ✅ Appears unregistered — the best future cross-city umbrella; worth ~$12/yr as insurance |
| `antifreezehq.com` · `joinantifreeze.com` | ✅ Appear unregistered (backups) |
| `defrostseattle.com` · `warmfrontseattle.com` | ✅ Appear unregistered (alternate brands "Defrost" / "Warm Front" for the friend review, same Freeze-cure idea) |
| `thawseattle.com` | Registered (2022) |
| `getantifreeze.com` | Registered 7/12/2026 — three weeks ago, unrelated party; no action |
| `antifreeze.com` | **Registered since 1996 via MarkMonitor** (Fortune-500 brand-protection registrar) — corporate-held, effectively unbuyable |
| `seattleantifreeze.com` | Registered (2007, Namecheap) |
| `antifreeze.org` | Registered (1999) |
| `antifreeze.gay` | Appears unregistered, but **ruled out 8/1 — Mason wants .com** |

**Recommendation (updated 8/1 for the .com constraint): buy `antifreezeseattle.com` now.** The site lives at **`gay.antifreezeseattle.com`** (the apex redirects there until a second audience exists). The two-axis expansion survives intact: **audiences are subdomains** (`trans.antifreezeseattle.com`, with partners, someday), **cities are sibling apexes** (`antifreezeportland.com`, …), and one wildcard cert (`*.antifreezeseattle.com`) + the per-community CloudFront model covers it — nothing in D8 changes. Optionally also grab `theantifreeze.com` as the future umbrella. Trademark note: "Antifreeze" for a community website is a suggestive use of a dictionary word in a service class unrelated to the automotive-coolant brands — low risk, but fold a trademark sanity check into the Phase-B counsel review (and skip "Icebreaker"-style alternates outright; that one's a famous apparel brand). Feeds SUTP Q9 (prod `website_address` secrets + SES/SPF/DKIM) once purchased.

**Candidate vetting — Mason's 24 + Claude's 8, all RDAP-checked 8/1/2026** *(incorporating and replacing Mason's inline list, per house rule)*:

**✅ Available, with verdicts:**

| Domain | Verdict |
|---|---|
| **`antifreezeseattle.com`** | **Still the pick.** The only name that covers the *whole* product: "antifreeze" is a product metaphor (what keeps you running in the cold), so it spans events + nightlife + guide + fitness without shrinking to a friendship app — and it scales to other cities and audiences |
| **`warmupseattle.com`** | **Best new find (Claude).** "Warm Up Seattle" = the Freeze cure *and* "the warm-up" — what you do before going out, and before a workout. Nightlife + fitness double meaning, Knotty-Yoga-adjacent, citywide. Strong #2 for the friend review |
| `beyondthefreeze.com` | Best of your freeze-phrase family — hopeful arc, if a bit abstract. #3 |
| `defrostseattle.com` · `warmfrontseattle.com` | City-anchored alternate brands ("Defrost" / "Warm Front" — the weather pun); solid #4/#5 |
| `stopthefreeze.com` · `endthefreeze.com` | Available, but campaign-slogan energy — reads activist rally, not a night out |
| `thawthefreeze.com` · `warmthefreeze.com` | Available, but tongue-twisty/semantically doubled ("thaw the freeze") |
| `defrostfriends.com` | Available, but toy-brand energy (microwaving your friends) |
| `yournextfriend.com` · `friendshipfound.com` | Available, but the friendship-app trap (below) — plus a faint dating-site read |
| `theantifreeze.com` · `antifreezehq.com` · `joinantifreeze.com` | Umbrella/backup insurance for the Antifreeze family |

**❌ Taken** (year registered): `communityfinder.com` (2004 — fine: it's the *platform's* internal name; as a consumer brand it was generic anyway) · `unfreeze.com` (2001) · `breakthefreeze.com` (2018 — linguistically your best freeze-verb, gone) · `icebreakerhq.com` + `icebreakerclub.com` (both 2025 — and the Icebreaker apparel trademark/SEO swamp made them a dodge anyway) · `frozennomore.com` (2026; Disney-joke risk regardless) · `warmconnections.com` (2017 — **expires 9/14/2026**, drop-watchable, but reads senior-dating) · `meetyourpeople.com` (2007) · `findyourcircle.com` (2007 — a shame, given the Circles feature) · `localcircles.com` (2011, an actual company) · `findyourcrew.com` (2006) · `buildyourcircle.com` (2025) · `hugitout.com` (2003) · `skipsmalltalk.com` (2002) · `justsayhello.com` (2002) · `jointhefun.com` (2003) · `findyourpeople.com` (2006) · `meltthefreeze.com` (3/2024) · `freezebreakers.com` (2026) · `seattlethaw.com` (2/2025) · `outandaboutseattle.com` (2013) · `gayseattle.com` (1997) · `thawseattle.com` (2022) · `getantifreeze.com` (7/2026) · `seattleantifreeze.com` (2007). **Special case: `outinseattle.com` is in redemption period** (its 2025 registration is lapsing) — if "Out in Seattle" appeals as a classic-idiom fallback, a ~$10–25 backorder (DropCatch/GoDaddy) might catch it when it drops in the next month or two; it loses the Freeze story, so opportunistic-only.

**Two vetting notes that matter more than any single name:**

1. **The friendship-app trap.** The `meetyourpeople` / `findyourcircle` / `justsayhello` family names the *Freeze pillar*, not the product. This site is events + city guide + adult layer + community — a friendship-brand over the bathhouse guide and the drag calendar undersells the moat and reads strangely. The name needs to hold the whole spectrum; product-metaphor names (Antifreeze, Warm Up) do, feature names don't.
2. **⚠️ There is a land rush on this exact naming space.** Eight freeze/social-adjacent names were registered 2024–2026: `meltthefreeze` (3/2024), `seattlethaw` (2/2025), `buildyourcircle`, `icebreakerhq`, `icebreakerclub` (2025), `frozennomore`, `freezebreakers`, `getantifreeze` (2026). That's consistent with the Timeleft/Mixingle/We3 "Seattle Freeze solution" wave the research found — someone else is circling. **Pick and register within days, not weeks.**

**Ranked shortlist for the friend review:** 1. `antifreezeseattle.com` (the pick) · 2. `warmupseattle.com` · 3. `beyondthefreeze.com` · 4. `defrostseattle.com` · 5. `warmfrontseattle.com` — plus `theantifreeze.com` as umbrella insurance either way.

**Site identity for friend review** — header: **ANTIFREEZE**; tagline candidates: "The cure for the Seattle Freeze" · "Gay Seattle, thawed" · "Where gay Seattle finds each other" · "Every event. Every crew. All of gay Seattle."

# Product Concept

## Pillar 1 — Events (the heartbeat)

The aggregated queer-Seattle calendar: bar/drag nights, parties, meetups, sports seasons, arts, volunteer days, protests, annual anchors. What the research adds to the SUTP Phase 10 design:

- **Recurring events are the spine of this scene** — Seattle queer life runs on "2nd Saturday" patterns (see Appendix C). The events model needs first-class recurring-series support *editorially* even though the DB stores one row per occurrence (a `series` grouping/label so "T4T @ Kremwerk, 2nd Saturdays" is a page, not just instances).
- **A big share of queer events happen at non-queer venues** (breweries, roller rinks, bookstores, climbing gyms) — venue list must not assume "gay bar."
- **Views that match real usage**: *Tonight*, *This Weekend*, *By Scene* (bears/leather/sapphic/trans/QTPOC/sober/40+/geek), *By Neighborhood*, plus month calendar.
- **Intake stack (decided by research, in priority order):** (1) **a mandatory structured submission form as the only human intake** — the NO STRAIGHT PLANS labor hack: "events submitted via email/DM will NOT be considered" converts curation from research into moderation; (2) **ICS/feed ingestion** where venues publish them (Kremwerk has a clean 4-month feed); (3) **Evvnt publisher partnership** (2,200-calendar syndication network The Seattle Times already uses) = free inbound supply; (4) **the AI scanner** (SUTP Phase 13) over venue sites/Eventbrite org pages; (5) **never scrape Facebook/Instagram or logged-in anything** (legal + technical wall; see Phase E notes).
- Categories (D4's list) extend with: **scene tags** (bear, leather, jock, sober, elders, geek, furry — plus sapphic/trans/QTPOC as accurate labels on the shared anchors we list, per the Q2 coverage rule) orthogonal to activity categories.

## Pillar 2 — The Guide (the moat)

Evergreen, structured directory + editorial. Sections, each with its volatility tier driving refresh cadence (Appendix has full seed data):

1. **Nightlife venue profiles** — the "which bar is for which crowd" layer (crowd/vibe is LOW-volatility, durable content; ownership/status is HIGH — 2025–26 alone: Cuff sold, Queer/Bar sold + remodeling, Neighbours building for sale). Include the **closure graveyard** (R Place, Re-bar, Comeback, Purr…) — history + trust signal + SEO nobody else has.
2. **Scenes** — landing pages per subculture we can cover honestly (bears, leather/kink, jocks/sports, sober, elders, geeks, the BIPOC gay scene) linking venues + recurring events + orgs. This is how "wholesome to rampantly slutty" coexists without moralizing: every scene gets the same respectful, factual treatment. Per Q2, sapphic/trans/QTPOC-wide scenes get **link-out cards to their own curators** (Sapphie Taffy, Dyke Alliance, Gender Justice League) with credit rather than pretended depth — while the big shared anchors (Pride weekend, festivals, mixed venues like Kremwerk) are listed like everything else.
3. **Sexual health** — STI testing (Harborview Sexual Health Clinic, the LGBTQ+ Center's free 6-day testing), one-stop PrEP (Kelley-Ross One-Step, 2 locations), DoxyPEP, mpox, HIV care (Madison Clinic), and the **two corrections every competitor gets wrong**: Lifelong's housing programs moved to Bailey-Boushay (2024–25), and Seattle Counseling Service closed in 2022 (successors: Optimism, Integrative Counseling, the Center's 12-session program).
4. **Everyday services** — LGBTQ-affirming primary care, mental health, gender-affirming entry points (Ingersoll's provider directory is the incumbent to link, not fight), dentists, and the GSBA-gap layer: *editorial* "who's actually good" for realtors, barbers, contractors, massage, tattoo, photographers (GSBA answers "who paid dues," we answer "who to hire").
	- Mason- I think that there is an opportunity here to make this crowd sourced where we have people provide suggestions and actually do a vote. Possibly do reviews.
	- Claude- Great instinct — recorded (8/5) as bucket **CM2 — Crowd recommendations** in [[Initial Project Implementation Outline]]: suggest-a-business + one-click "recommend" with public counts and optional short moderated tips, slotted right after launch. Two risk-shapings baked in: **free-text reviews are deferred** to a counsel-gated go/no-go (negative reviews of named small businesses are the defamation/moderation surface; votes are structurally positive), and **health-category recommendations render as unattributed aggregates only** (MHMDA — a user's name never publicly attaches to "recommends this HIV clinic"). It also solves this section's labor problem: community endorsement scales where editorial vetting can't. 
5. **Fitness & rec** — queer climb nights (5+ gyms on a monthly rotation), queer yoga/fitness, SANCA/Emerald City Trapeze (one org since 2023), the gay-gym-scene question (anecdote-tier, present it as dated "what people say" — an un-Googleable content gap).
6. **The adult layer (18+ section)** — bathhouses (Steamworks; Club Z — 50 years old in 2026, building listed for sale 2/2026, watch), sex-positive orgs (Pan Eros, Seattle Erotic Art Festival), naturist groups (SLUGS, Lake Bronson, Tiger Mountain), **Denny Blaine & Howell Park with the actual current rules** — the July 2026 court ruling (park stays nude; code of conduct, ranger staffing, screening vegetation, signage) plus etiquette, each stamped "as of {date}." Cruising beyond the beaches: **decided (Q3, option b)** — cover the *documented* places (Denny Blaine, Howell, Volunteer Park as history) plus etiquette/safety/current-law editorial, with "the apps own real-time" honesty; no location listings beyond that until the site has standing, and maybe never.

## Pillar 3 — New to Seattle (the moment)

A relocation guide, not a tourist guide — currently served by *no one* while demand spikes:

- **Neighborhood rundowns** with queer character, 2026 1BR rent ranges (always ranged + dated; cross-source spread is ±$300), and light-rail access. Capitol Hill is still the infrastructure gayborhood (14+ queer venues walkable); White Center is the clearest "second gayborhood"; **the Eastside (Bellevue, Kirkland, Redmond, Bothell, Issaquah, Sammamish) is where a large share of gay King County actually lives** — venue-free but meetup-rich, ignored by every incumbent guide, and since 3/2026 the 2 Line puts Bellevue/Redmond one direct train from Capitol Hill; Tacoma is a real second scene but out of v1 scope (Q5). Appendix F has the table.
- **Transit as of mid-2026**: 1 Line Lynnwood↔Federal Way; **2 Line fully connected across the lake 3/28/2026** (Judkins Park station is a big deal for queer renters); West Seattle/Ballard lines still future.
- **Traffic honesty**: Revive I-5 cuts capacity through 2027 (Lynnwood→Mercer ~50–65 min); 520 is tolled; West Seattle is bridge-dependent. "If you're moving here in 2026–27, don't plan a car commute across the Ship Canal."
- **First-90-days checklist**: get on PrEP/testing rails, pick your scenes, join one recurring thing (→ Pillar 4), the paperwork stuff (license, voter reg), where queer housing leads actually surface (the 13k-member Seattle Queer Housing FB group — link out; we don't need to own it yet).
- **A "moving here because of your state" lane** — the 2026 influx spans the whole LGBT spectrum; we build the gay-men's version deeply (this guide) and **link out with credit** to the trans-specific rails (Ingersoll, Lavender Rights, the 2026 King County Trans Resource & Referral Guide) rather than curating them ourselves (Q2) — still high-need, still uncontested.

## Pillar 4 — Beating the Freeze (the wedge)

The Seattle Freeze is real (UW Psychiatry publishes tips; Axios/Seattle Times cover it as loneliness infrastructure). The known cure is *recurring shared-activity groups, not one-off events* ("it's tough to make real friends through Meetup events alone" is the consistent finding — and summer is the joining season). Nobody aggregates the queer version of this. We make it a first-class product surface:

- **The recurring-groups directory**: 34+ LGBTQ+ sports leagues (softball since 1979, ORCA swim since 1984, Frontrunners since 1985, Quake rugby, 3 competing kickball operators…), choruses (SMC/SWC, Rainbow City Performing Arts), climb nights, book clubs (Charlie's Queer Books runs 4), gaymers, figure drawing, faith communities, **recovery meetings (gay AA exists and appears on no queer calendar anywhere — a genuinely unserved wholesome layer)**, professional groups (Out in Tech's 2nd-Wed happy hour), volunteer orgs (VolunQueers, Lambert House, GenPride).
- **"Pick your onramp"** — a guided path by personality/interest (sporty / arty / nerdy / sober / outdoorsy / 40+ / BIPOC / new-in-town / **Eastside-based**) → 2–3 matched recurring groups + this month's low-barrier entry events (e.g., Frontrunners' Monday 3-mile Cal Anderson mini-run). Structured "join something" advice beats the generic listicle every outlet writes.
- This pillar is also the honest bridge to Knotty Yoga: a wellness/fitness/acro studio *belongs* in a directory of recurring queer movement spaces — listed by the same rules as everyone else.

## Pillar 5 — Community (the growth path, gated)

Strict order, each gated on traction + capacity: (1) event submissions with review queue (the form IS the community feature); (2) "report stale / is this still open?" flags — community-powered freshness; (3) newsletter replies → reader tips lane; (4) org/venue self-service claiming of their listings; (5) **"where this scene talks" link-out cards** (Discord servers, Bluesky, FB groups, Meetup) on scene/org pages — community value with zero UGC risk, shippable with the Phase D content; (6) **Circles** — the Q7 answer: bounded rooms instead of forums (next subsection); (7) activity-partner / looking-for-group — as posts *inside* the relevant circles, not a public classifieds board; (8) **personals/dating: probably never** — FOSTA/SESTA killed Craigslist personals; two live §230-sunset bills in Congress (12/2025–2026); Sniffies owns the space with $100M behind it. If ever revisited: verified identity + bright-line no-commerce rules + real moderation staffing + counsel first.

### Circles, not forums (the Q7 model — for Mason's read)

What you asked for isn't a forum with better rules — it's a different shape. Facebook/Bluesky-style acrimony runs on three mechanics: a **global feed** (an audience to perform for), **strangers** (no relationship cost to cruelty), and **engagement scoring** (conflict is the growth loop). Remove all three and the failure mode has nothing to feed on. What remains is the **bounded room** — front porch, not town square:

- **A Circle** = a named room attached to a real-world thing: a league's players, a recurring event's regulars, a "New in Town — Fall 2026" cohort, the board-gayme crowd. **Never created empty** — each launches with its existing group and a recruited **steward** (1–2 members with moderation powers). Cold-start solved by construction.
- **Visibility per circle**: member-only (default) · public-read/member-post (announcement-style circles — a league posting season info) · unlisted/invite. **No global timeline anywhere** — a member's home is just "your circles."
- **Posting**: chronological posts + comments within the room. No reshares, no quote-posts, no follower counts, no algorithmic ranking, no trending. At most one low-heat reaction (or none at v1).
- **Membership & accountability**: open-join / steward-approval / invite-only per circle; real accounts (existing auth); code-of-conduct acceptance at join; per-circle boot + site-wide ban tooling; report button → admin queue. Stewards moderate their room; site admins backstop. Circle creation is admin-approved at first — no open room-creation free-for-all.
- **Platform fit**: honuware's roles/permissions machinery already models "members of X can see Y"; concretely ~4 new tables (`circles`, `circle_members`, `circle_posts`, `circle_comments` + reports) with server-side visibility checks, all admin-CRUD-manageable day one. And Pillar 4's "pick your onramp" gets its natural ending: *join the circle*.
- **Still gated**: this is user-generated content — it stays in Phase H behind the traction gate and the Q9 counsel gate, launching with 3–5 seeded circles tied to real groups.
- **Open sub-questions for you (Q7 continued)**: the name (Circles / Crews / Rooms?); default visibility (rec: member-only); any public-read circles at launch (rec: announcement circles only); reactions (rec: none at v1).

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
- **Audience rule (Q2, decided 8/1)**: the voice is **gay men writing for gay men** — that's who we are and who the marketing serves, and pretending otherwise would dilute both. Coverage rule: list the LGBT-wide anchors everyone attends (Pride weekend, festivals, mixed venues); **link out with credit, don't fake depth** on sapphic/trans/QTPOC-specific scenes; sister sites (`trans.seattle.…` etc.) happen only ever in partnership with people from those communities who can do them justice — recorded as a standing principle, and the brand architecture already supports it.
- **Spectrum handling is structural, not tonal**: scene tags + an 18+-gated section for the adult layer (bathhouses, beaches/cruising, kink). WA has **no age-verification law as of mid-2026** (HB 2112 didn't pass; FSC v. Paxton greenlit state AV laws, so watch) — a simple interstitial suffices today and keeps the site far below any "1/3 sexually explicit" threshold. Everything stays guide-level factual, nothing explicit.
- **Privacy by design is a legal requirement here, not a nicety**: Washington's **My Health My Data Act** (private right of action; first class action 2/2025) squarely covers sexual-orientation- and sexual-health-adjacent data + geolocation. Design consequence: no ad-tech pixels, no third-party trackers, privacy-friendly analytics only, no geofencing, minimal accounts data. This is cheap for us and a real trust differentiator ("we can't leak what we don't collect"). Note this bans *surveillance adtech*, not advertising — direct-sold sponsorships and contextual ads are fully compatible (see **Ads & money**).

## The Knotty Yoga relationship (trust model)

Research-backed pattern for a business-owner-run community site: **full transparency beats stealth** (FTC endorsement rules + audience-trust studies + QSC's "100% independent, funded by supporters" tonal template):

- Persistent site-wide footer: "Founded and funded by [Mason / Knotty Yoga]."
- A public **listing policy / editorial independence page**: how listings get in (free, criteria-based), how sponsors are labeled ("Presented by"), and the bright line: **we never rank our own business in any 'best of' we publish** — Knotty Yoga appears in directories under the same rules as everyone else.
- The marketing payoff is structural, not promotional: the studio is *venue + org + recurring-groups host* inside the most useful queer site in the city, plus "Presented by Knotty Yoga" on the newsletter/flagship guides. Low-key by design, durable because disclosed.

## Ads & money (Q9 follow-up — yes, ads are allowed)

MHMDA does **not** ban advertising — it effectively bans *surveillance* advertising: collecting/sharing health-adjacent data, third-party tracking pixels, geofencing near clinics. What works (and is the better business at local scale anyway):

1. **Direct-sold sponsorships** — "Presented by" placements on the site, newsletter, or a section, sold straight to venues, orgs, and gay-friendly businesses. No user data touched. This is how every surviving peer monetizes (QSC's memberships + sponsorships, CHS's pay-what-you-can, city newsletters), and it's what SUTP's "eventually advertising" becomes.
2. **Contextual ads** — matched to the *page* (a bar page carries a bar's ad), never to the person. MHMDA-compatible.
3. **House ads** — Knotty Yoga placements under the same labeling rules the trust model already sets.
4. **Reader support** — tip jar / memberships (the QSC model) as a second leg once there's an audience.
5. **Out:** programmatic networks (AdSense-style) and Meta/retargeting pixels — that's the MHMDA blast radius *and* the trust-killer, and at niche-local traffic it pays peanuts anyway; direct local sponsors pay real money.

**The bright line, codified:** money buys clearly-labeled *placement*, never *inclusion or ranking*. Listings stay free and editorial forever — our whole critique of GSBA ("a directory of who paid dues") depends on us never becoming one.

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
| Circles (Q7 model) | honuware auth/roles + ~4 new tables (circles/members/posts/comments) with server-side visibility checks; admin CRUD from day one | New — Phase H, gated |
| Multi-city/audience future | D8 communities-as-tenants — each `{audience}.{city}` pair is a tenant; Tacoma or a partner-run trans sister site are `--create_tenant` away; the `gay.antifreezeseattle.com` scheme (audience subdomains, per-city apexes) maps 1:1 onto per-community CloudFront | Already architected |
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
- [x] Mason reads + marks up this doc; Open Questions answered inline (8/1/2026 — Q1/2/3/4/5/6/8 decided; Q7 model, Q9 definitions, Q10 menu delivered for his read)
- [ ] Friend review (Levi, Caleb + 2–3 target-audience friends incl. at least one recent transplant and one scene-connected person)
- [ ] A 5-question survey for ~10 gay/queer Seattle friends: where do you find out about things now? what can you never find? would you use X? (cheap demand validation)

### B.2 Decisions out of review
- [~] Lock name/brand + domain — **brand decided: Antifreeze**, .com constraint added, and **`antifreezeseattle.com` appears available** (RDAP 8/1); remaining: **[Mason] buy it** (+ optionally `theantifreeze.com` as future umbrella) and confirm display name/tagline with friends (feeds SUTP Q9)
- [x] Lock audience framing — **gay men, link-out posture for sister scenes** (Q2, 8/1); voice one-pager itself still lands in C.3
- [x] Lock adult-content stance — **Q3 option (b)** (8/1): documented places + etiquette/safety/law editorial; no location listings beyond the beaches
- [x] Lock geographic scope — **King County incl. the Eastside** (Q5 + Mason follow-up, 8/1); Tacoma/Everett = future communities via tenancy
- [ ] Lock the v1 scope cut: recommendation = Events MVP + Nightlife/Scenes/Health guide sections + Freeze directory; defer services-directory editorial and full relocation guide to fast-follow (friend-review input wanted)
- [~] Labor model **decided: (a) rotating editor-of-the-week + (c) paid part-time curator at a trigger** (Q6, 8/1); still to set: the honest hrs/week budget + confirming the (c) trigger proposed in Q6's note

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
- [ ] "Where this scene talks" link-out cards on scene/org pages (Discord/Bluesky/FB groups/Meetup) — the zero-UGC community layer (Q7)
- [ ] Adult section per Q3(b): bathhouses + Denny Blaine/Howell current-rules pages with dated stamps + etiquette/safety/law editorial (no location listings beyond the beaches)
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

- [ ] Newsletter signup live from the first public deploy; weekly **Thursday** "This Week in Gay Seattle" digest starts when events data is 2 weeks solid (Q8 decided; the retention spine — beehiiv-class provider, owned list)
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

- [ ] **Circles** per the Q7 model (bounded member-visible rooms, stewards, no global feed — see "Circles, not forums"): launch with 3–5 seeded circles attached to real groups, admin-approved creation, counsel gate first (UGC)
- [ ] Activity-partner / looking-for-group — as posts inside the relevant circles rather than a public classifieds board (Pillar 4's natural extension)
- [ ] Services marketplace (massage therapists, trainers offering services) — needs listing-policy + liability review
- [ ] Personals: default **no** (see Pillar 5); revisit only with counsel + §230 clarity

# Success Metrics — Options Menu (Q10)

Pick ~5 — one per row that matters to you; my default is the **Solid** column with the Mission row as north star. Everything here is measurable **without user tracking** (aggregate server-side counts, newsletter-provider stats, quarterly one-click polls, promo codes) — consistent with the MHMDA privacy posture. The Business row stays internal, never published.

| Dimension | Metric (how measured) | Lean @ mo. 6 | Solid @ mo. 6 | Stretch @ mo. 6 |
|---|---|---|---|---|
| Coverage | % of a hand-audited sample week's gay-King-County events that are on the site (1 volunteer-hour audit per quarter) | 70% | 90% | 98% |
| Freshness | % of listings inside their volatility tier's verification window (admin dashboard count) | 90% | 100% | 100% + public per-page stamps |
| Site audience | Weekly unique visitors (aggregate, cookieless) | 300 | 1,000 | 3,000 |
| Return habit | Share of visitors returning within 14 days (aggregate) | 15% | 25% | 40% |
| Newsletter | Subscribers / open rate (provider stats) | 200 / 35% | 500 / 45% | 1,500 / 50% |
| Calendar feeds | Unique ICS-feed subscribers per week | 25 | 100 | 400 |
| Search & AI surface | Top-3 Google for target queries ("gay bars seattle" family) + cited by Perplexity/ChatGPT on a monthly test-prompt set | 1 query | 2 queries + 1 AI citation | 5 queries + repeat citations |
| Community signal | Outside event submissions/week arriving through the form | 2 | 5 | 15 |
| **Mission (Freeze)** | "I joined something because of the site" reports (quarterly 1-click newsletter poll) | 5 | 25 | 100 |
| Business (KY, internal) | Knotty Yoga referral visits + "found via Antifreeze" intro-code redemptions per month | measurable at all | 20 visits + 5 codes | 75 visits + 20 codes |
| Sustainability | Curation hrs/week team-wide, sustained 4 straight weeks (self-report) | ≤6 | ≤4 | ≤3 (scanner carrying the rest) |

Reading the menu: **coverage + freshness are the identity claims** — if either drops below Lean, that's the alarm, regardless of audience numbers. The Mission row is the one worth quoting ("we can count people who joined something because of us"). The Sustainability row is Q6's month-18 answer made visible. Mark a column per row (or write in your own targets) and I'll fold the picks into Phase B.2.

# Open Questions

*(Numbered for inline answers, house style. Recommendations included so "agreed" suffices.)*

1. **Public brand vs. platform name.** CommunityFinder is the repo/platform. Is it also the consumer brand, or do we want a Seattle-punchy brand on top (the winners are punchy: "The Gay Agenda," "Queer Social Club")? Your `gay.seattle.antifreeze.com` lean is interesting precisely because **Antifreeze is a real brand hook** — it names the Freeze-beating mission, and `city.antifreeze.com` maps exactly onto the multi-community tenancy model (Seattle first, Tacoma/Portland later). Other directions: descriptive-SEO ("Gay Seattle Guide"), emerald-themed, or reviving respect for the ancestor ("…scene"-adjacent names risk confusion with the living SGS). *Rec: treat naming as a friend-review exercise with Antifreeze as the front-runner; don't block anything on it.*
	- Mason- I like gay.seattle.antifreeze.com. It let's use move to different targets (gay first then other underserved communities and could also move to different cities)
	- Claude- Decided — **Antifreeze** is the brand; CommunityFinder stays the platform name. Your two-axis expansion (`{audience}.{city}`) is now the recorded architecture and maps 1:1 onto D8 communities-as-tenants. One practical snag (RDAP-checked 8/1): **antifreeze.com is corporate-held since 1996** via MarkMonitor — effectively unbuyable — and seattleantifreeze.com/antifreeze.org are taken too. **antifreeze.gay appears unregistered**: recommend verifying + registering it now (check the price — dictionary .gay names are often registry-premium) and running this site at `seattle.antifreeze.gay`, adding a neutral umbrella apex only when audience #2 materializes. Full analysis + tagline candidates in the new **Brand & Naming** section.
	- Mason- I don't know that this is actually decided then. I think having a non-dotcom domain would be a bad idea. I really like the antifreeze idea but maybe help brainstorm some other ideas with an available domain? Is antifreezeseattle.com available? Then we could do gay.antifreezeseattle.com.
	- Claude- Good news — **`antifreezeseattle.com` appears unregistered** (Verisign RDAP, 8/1), so your exact scheme works: **`gay.antifreezeseattle.com`**, audiences as subdomains, future cities as sibling apexes (`antifreezeportland.com`, …). Also apparently available if you want alternates for the friend review: `theantifreeze.com` (best cross-city umbrella — worth ~$12/yr as insurance), `antifreezehq.com`, `joinantifreeze.com`, and alternate-brand options `defrostseattle.com` / `warmfrontseattle.com`. Taken: `thawseattle.com` (2022) and `getantifreeze.com` (registered 7/12/2026 by an unrelated party). The Brand & Naming section is rewritten around the .com constraint with the full availability table. Two cautions folded in there: RDAP "available" is not a reservation — **buy it before the name gets socialized** — and we'll run a trademark sanity check inside the Phase-B counsel review (dictionary word, unrelated service class → low risk). B.2 now reads decided-pending-purchase.
	- Claude- Second vetting round done (8/1): your 24 candidates + 8 of mine, all RDAP-checked with verdicts in the **Candidate vetting** table above (replacing the raw list, house rule). Short version: 8 of yours are available but the strongest (`breakthefreeze.com`, `unfreeze.com`, `findyourcircle.com`) are taken; `communityfinder.com` is taken (2004) and was generic as a consumer brand anyway; the friendship-family names undersell the full product (events + guide + adult layer, not just friend-finding). One real find: **`warmupseattle.com` is available** — Freeze cure + pre-going-out + fitness triple meaning — my #2 behind `antifreezeseattle.com`, which stays the pick. Also: `outinseattle.com` is lapsing (redemption period — backorderable if you want a fallback), and **eight freeze-adjacent names were registered by others in 2024–2026**, so whichever you choose, register it within days.
2. **Audience framing.** "Gay Seattle" (gay-male-forward, your marketing demographic) vs "Queer Seattle" (full spectrum)? Research reality: the curation energy and unserved demand skew queer/trans/QTPOC (Sapphie Taffy's audience, the relocation wave), and lesbian/sapphic Seattle has exactly one bar but a huge events layer. *Rec: inclusive "queer Seattle" scope with honest gay-male editorial depth where that's what we know — scene tags make breadth structural rather than performative. Your call; it shapes name, voice, and how the site reads.*
	- Mason- I like sticking to gay for now. I am not trans or a lesbian. Nor are the friends helping with this. This is already a big ask and I feel like trying to mix queer and trans in would unfortunately dilute things. I'm open to doing a sister site later (trans.seattle.antifreeze.com) but I would want to partner with people from that community(ies) who actually know that space and can do it justice.
	- Claude- Decided — **gay men, done honestly.** Folded in as the Voice section's audience rule with ripples through the pitch, scene tags, scene pages, relocation lane, and newsletter name: we list the LGBT-wide anchors everyone attends, **link out with credit** (Sapphie Taffy, Dyke Alliance, Gender Justice League) instead of pretending depth on sister scenes, and sister sites happen only in partnership with people from those communities — recorded as a standing principle the brand architecture already supports.
3. **The adult layer's outer edge.** Bathhouses + Denny Blaine/Howell + naturist groups + kink orgs feel clearly in (factual, publicly documented, 18+-gated). The contested call is **cruising listings beyond the beaches** (parks/trails/etc.): global travel guides list them with disclaimers; *local community* guides universally don't (venue relationships, sponsors, civic legitimacy — and Sniffies owns real-time cruising natively with Seattle-density data). Options: (a) full listings with etiquette/safety/legal framing; (b) **the middle path: cover the *documented* places (Denny Blaine, Howell, Volunteer Park as history) + etiquette/safety/law editorial + "the apps own real-time" honesty**; (c) omit beyond beaches. *Rec: (b) at launch — it delivers the non-moralizing spectrum promise without becoming a liability/partnership problem; revisit (a) once the site has standing.*
	- Mason- I don't really know this space that well and I'd hate to list something that isn't safe / legal and cause trouble so I think b might be the best option for now.
	- Claude- Decided — **(b)**, folded into Pillar 2 and Phase D.3: the documented places (Denny Blaine, Howell, Volunteer Park as history) + etiquette/safety/current-rules editorial with dated stamps; no location listings beyond that. Your instinct matches the research: local guides that need venue relationships and civic legitimacy all stop exactly here, and Sniffies owns real-time anyway.
4. **Knotty Yoga visibility.** Invisible / footer-credit / fully open "founded by." *Rec: fully open (footer + about page + listing policy) — research says stealth marketing is the trust-killer and disclosure is the durable version of the marketing benefit.*
	- Mason- That sounds fine to me.
	- Claude- Decided — fully open (footer + about page + listing policy + never self-rank). Lands with Phase C.3's listing-policy draft.
5. **Geographic scope at v1.** Seattle city / King County / metro incl. Tacoma+Eastside? *Rec: Seattle + King County listings with a Tacoma "worth the trip" section; Tacoma becomes its own community via tenancy when someone local can own it.*
	- Mason- Definitely all of King County. Not sure about Everett and Tacoma. Let's go with King County for now.
	- Claude- Decided — **King County**, with your Eastside follow-up folded in: Kirkland, Redmond, Bellevue, Bothell, Issaquah, and Sammamish added to Appendix F (8/2026 rents, the 2-Line reality, an Eastside-reality note) plus an "Eastside-based" onramp in Pillar 4 — the Eastside is treated as first-class, not spillover. Tacoma/Everett stay out of v1 as future communities (each is one `--create_tenant` away when someone local can own it).
6. **The labor model (the existential one).** Who does the weekly moderation/verification hours in month 18? Options: (a) Mason+Levi+Caleb rotating editor-of-the-week; (b) scanner+review-queue only, accept thinner editorial; (c) budget a paid part-time curator once there's any revenue; (d) recruit 2–3 volunteer scene-editors (bear scene, sapphic scene, arts) with named credit. *Rec: (a)+(d) designed into the admin tooling from day one (review queues are multi-user already); decide the trigger for (c).*
	- Mason- I'll go with your recommendation (a+c)
	- Claude- Decided — **(a) rotating editor-of-the-week now + (c) paid part-time curator at a trigger** ((d) volunteer scene-editors stays an opportunistic add-on, not a dependency). Proposed trigger to confirm: **(c) activates when recurring sponsor/membership revenue reaches ~$500/mo OR the rotation exceeds 4 hrs/person/week for four straight weeks — whichever comes first.** Still open in B.2: the honest hrs/week budget the three of you will commit (the metrics menu's Sustainability row makes it visible month to month).
7. **Forums: build, bridge, or skip.** Research is unambiguous that Discord kills compounding value and there's no dominant Seattle gay Discord to bridge to — but forums have real cold-start + moderation costs, and the §230 sunset bills add risk to all UGC. *Rec: skip at launch; revisit at a concrete traction gate (e.g., 1,000 newsletter subs) starting with narrow boards (housing / newcomers / looking-for-group), never general chat.*
	- Mason- I want to get away from people arguing and disparaging each other publicly. I have no interest in being a bluesky or Facebook. I'm wondering if there is a way though to let people publish things and have a model where people who are part of certain groups / communities can see and possibly comment on those posts. We could also link to bluesky or discord servers.
	- Claude- Not a forum, then — a **Circles** model, drafted for your read in the new "Circles, not forums" subsection under Pillar 5: bounded member-visible rooms attached to real-world groups, with stewards, chronological posts + comments, and **none of the mechanics that make Facebook/Bluesky combative** (no global feed, no strangers, no reshares or scoring — nothing to perform for, nobody to dunk on). Membership per circle (open/approval/invite), code of conduct at join, report queue + admin backstop, admin-approved circle creation, and a hard rule that no circle launches empty. Looking-for-group posts live inside circles. The "link to bluesky or discord servers" half of your idea ships much earlier and risk-free as "where this scene talks" cards on scene/org pages (now a Phase D.3 item). Circles stay Phase H behind the traction + counsel gates. React to the sub-questions at the end of that subsection (name, default visibility, reactions). 
8. **Newsletter platform + cadence.** Weekly Monday digest (The Gay Agenda's cadence) vs Thursday (weekend-planning)? Provider (beehiiv/Buttondown/self-host)? *Rec: Thursday weekly; pick provider at Phase F; signup form from first deploy regardless.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided — Thursday weekly ("This Week in Gay Seattle"), provider picked at Phase F, signup form live from the first deploy.
9. **Legal counsel timing.** MHMDA (health-adjacent data + private right of action), ToS/privacy pages, the adult-section posture, and eventually UGC. *Rec: minimal-tracking design now (defuses most MHMDA exposure by construction); one counsel review before first public deploy; second review gate before any UGC feature.*
	- Mason- What is MHMDA and UGC?
	- Claude- Definitions first. **MHMDA** = Washington's *My Health My Data Act* (passed 2023, effective 2024) — a state privacy law covering "consumer health data" defined very broadly: sexual health, gender-affirming care, *inferences* about either, even precise location near health services. Its teeth: a **private right of action** — individuals can sue directly (first class action filed 2/2025), not just the Attorney General. A gay site that lists PrEP/testing resources *and* runs ad trackers would sit squarely in its blast radius; our answer is architectural — no third-party trackers, no ad pixels, no geolocation, aggregate-only analytics — so there's essentially nothing to sue over ("we can't leak what we don't collect"). **UGC** = user-generated content: anything visitors publish through us (submitted events, comments, future circle posts). It matters because hosting other people's words is where platform-liability law (§230 and the two live sunset bills) applies. With those defined, the actual Q9 decision is still open — my rec stands: minimal-tracking design now, one counsel review before the first public deploy, a second counsel gate before Circles. Say "agreed" and I'll mark it decided.
	- Mason- How much would legal review for this cost? So I can't put ads on the site?
	- Claude- Two answers. **Cost (ballparks, not quotes):** the pre-launch review in question — privacy policy + terms of service + an MHMDA-posture check for a *no-tracking, no-UGC* editorial site — is the cheap end of internet law: a solo internet/privacy attorney typically does it as a **flat package around $1.5k–3.5k**; boutique-firm hourly ($300–600/hr × ~3–8 hrs) lands roughly $1k–5k. Our no-tracker architecture is exactly what keeps the bill small — there's little surface to review. The second review (before Circles/UGC, Phase H) is a separate, similar-sized engagement much later. Cheaper routes to try first: **QLaw Association of Washington** (the LGBTQ+ bar association's lawyer-referral service — on-brand, likely sympathetic), **Communities Rise** (low-cost WA small-business/nonprofit legal help), or GSBA member attorneys (doubles as a relationship play). Ask 2–3 for flat-fee quotes on exactly "MHMDA posture + privacy policy + ToS for a no-tracking community website." Policy generators (Termageddon-class, ~$100–300/yr) are an acceptable stopgap, not a substitute. **Ads: yes — you can absolutely run ads.** MHMDA kills *surveillance* advertising (tracking pixels, programmatic profiling, geofencing near clinics), not advertising itself. Direct-sold sponsorships, contextual ads, house ads, and reader memberships are all fine — and better business at local scale anyway (programmatic pays peanuts; local sponsors pay real money). New **Ads & money** section added under the trust model, including the bright line: money buys labeled *placement*, never listing *inclusion or ranking* — otherwise we become the pay-to-list directory we're critiquing. With both clarified, the Q9 recommendation itself (counsel review before first public deploy; second gate before Circles) still awaits your "agreed."
10. **Success metrics for v1.** What makes this "working" by month 6? *Rec: (a) coverage: ≥90% of a hand-audited week's queer events listed; (b) freshness: 100% of listings verified within their tier cadence; (c) audience: newsletter subs (target 500 by month 6) + weekly returning visitors; (d) the marketing payoff: Knotty Yoga referral traffic/signups tracked honestly. Agree/adjust targets.*
- Mason- Please suggest a variety of options for me.
- Claude- A full menu is now in the **Success Metrics — Options Menu** section above: 11 dimensions (coverage, freshness, site audience, return habit, newsletter, calendar feeds, search/AI surface, community signal, the Freeze mission metric, the internal Knotty Yoga metric, and labor sustainability), each with Lean / Solid / Stretch month-6 targets, all measurable without user tracking. Mark a column per row that matters to you — my default: Solid everywhere, Mission as north star, and coverage+freshness as the identity claims where slipping below Lean is the real alarm.

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
| Bellevue | Eastside hub — downtown high-rises, international, tech-family; no gay venues but a real community layer (Crossroads events); cross-lake nightlife now one direct train | ~$2,890 avg (8/2026) | **2 Line ×4 stations** |
| Kirkland | Waterfront + Google campus, restaurant-y; quiet meetup-based queer presence | (verify — likely $2.2–2.5k) | no rail; I-405 Stride BRT planned |
| Redmond | Microsoft-land + Marymoor; Eastside QTPOC Collective (Together Center), VALA queer figure drawing 3rd Sun | ~$2,630 avg (8/2026) | **2 Line ×3 stations (Downtown Redmond, new 3/2026)** |
| Bothell | UW-Bothell college town + breweries (Beardslee Latin Night 2nd Sat); cheapest Eastside | ~$2,095 avg (8/2026) | no rail; SR-522 Stride BRT planned |
| Issaquah | Trailhead town, outdoorsy, car-first | ~$2,455 avg (8/2026) | no rail (Issaquah Link is a 2040s project) |
| Sammamish | Plateau families, thin rental stock | (thin data) | car-only |
| Burien | 10th Pride 2026, first parade; growing | ~$1,750–1,960 | bus |
| Renton | Affordable; Cedar River Clinics | ~$1,530–1,900 | bus |
| Tacoma *(out of v1 scope — future community, Q5)* | **Real second scene**: The Mix + Silverstone on St. Helens, Hilltop/6th Ave, own Pride | well below Seattle | T Line, Sounder |

**Eastside reality (added 8/1 per Mason — his home turf and Knotty Yoga's):** no gay venues, but a real recurring layer — Monthly Eastside LGBTQ+ Happy Hour · East King County LGBTQ+ All Ages Meetup · Rainbow Elder Play & Connect (Crossroads, 3rd Sat) · Rainbow on the Eastside (annual, Aug) · Eastside QTPOC Collective (Together Center, Redmond) · Beardslee Bothell Latin Night — and since 3/28/2026 the 2 Line makes Capitol Hill a direct train from Bellevue/Redmond (check last-train times for club nights). A large share of gay King County lives here and every incumbent guide is Seattle-proper-centric: an underserved audience the site should treat as first-class, not spillover. Eastside rents from rent.com/zumper-family aggregators, 8/2026.

**Transit as of 7/2026:** 1 Line Lynnwood↔Federal Way (41 mi); 2 Line fully connected 3/28/2026 (Redmond↔Lynnwood via downtown; first light rail on a floating bridge); Pinehurst/NE 130th expected 2026. **Traffic:** Revive I-5 through 2027 (Lynnwood→Mercer 50–65 min); 520 tolled; West Seattle Bridge fine but single-point-of-failure. World Cup summer 2026 compounds everything.

## G. Annual calendar skeleton (anchors)

May: Seattle Erotic Art Festival (5/1–3) · SIFF. June: Pride in the Park (6/6) · Burien Pride (6/5–7) · GSBA Luncheon · Trans Pride (6/26, Volunteer Park) · Queer/Pride Festival (6/26–28, ticketed) · PrideFest Capitol Hill + Dyke March + Indigiqueer Fest (6/27) · Parade + PrideFest Seattle Center (6/28, largest free Pride fest in US) · White Center Pride · Wildrose/Cuff/Kremwerk/Neighbours Pride runs. July: Tacoma Pride · Gardens & Gems (Gay City). Aug: Alki Beach Pride (8/1+8/15) · Dining Out For Life (8/13) · **PNW Black Pride Weekend (8/20–23)** · Latinx Pride (9/19 — Sept) · Capitol Hill Block Party (8/7–9, adjacent). Sept: Kremfest (9/25–27) · AIDS Walk NW (date UNVERIFIED). Oct: ~~SQFF~~ (cancelled 2026). Nov–Dec: chorus holiday shows · Jinkx & DeLa · Intiman Naughty List · World AIDS Day (12/1). **Defunct/unverified:** Red Dress Party (ended 2024) · Bat n' Rouge (UNVERIFIED) · Gay Bingo (UNVERIFIED) · NW Leather Celebration (2026 was Sacramento).

## H. Denny Blaine current-state (the content-moat exemplar)

Clothing-optional queer beach, decades-standing. Dec 2023 playground proposal defeated · spring 2025 neighbor lawsuit · July 2025 abatement order (signage, fenced clothing-optional zone) · **~7/15/2026 ruling: park stays open and stays nude**, with required code of conduct, increased ranger staffing, vegetation screening, explicit signage; police calls down 39% since abatement; topless legal parkwide (WA law). Appeal likely; **rules will shift season to season → a dated "current rules + etiquette" page, refreshed monthly during litigation, is exactly the kind of content nobody else will maintain.** Howell Park: smaller, hidden, historically "more for the boys," no facilities. Volunteer Park: treat as history content.

# Change Log

- **8/5/2026 (v0.4.1)** — Mason's crowdsourced-recommendations idea (services layer, Pillar 2.4) answered in-thread and tracked as bucket **CM2** in [[Initial Project Implementation Outline]] (votes + suggestions early; free-text reviews counsel-gated; health categories unattributed).
- **8/1/2026 (v0.4)** — Domain vetting round: Mason's 24 candidates + 8 Claude additions all RDAP-checked; raw list replaced by the **Candidate vetting** table in Brand & Naming (available-with-verdicts / taken-with-years / the friendship-app-trap note / the ⚠️ freeze-name land-rush warning — 8 adjacent names registered 2024–2026, so register within days). Findings: `communityfinder.com`, `unfreeze.com`, `breakthefreeze.com`, `findyourcircle.com` taken; **`warmupseattle.com` available and the new #2**; `outinseattle.com` lapsing (backorderable). Ranked shortlist for friend review: antifreezeseattle > warmupseattle > beyondthefreeze > defrostseattle > warmfrontseattle (+ theantifreeze.com umbrella insurance). Recommendation unchanged: **`antifreezeseattle.com`**.
- **8/1/2026 (v0.3)** — Second Q&A round: Q1 re-opened on the .com constraint and re-resolved — **`antifreezeseattle.com` appears available** (8-domain RDAP sweep in Brand & Naming; scheme = `gay.antifreezeseattle.com`, audiences as subdomains, cities as sibling apexes; trademark sanity check added to counsel scope; B.2 = decided-pending-purchase). Q9 follow-ups answered: legal-review cost ballparks ($1.5k–3.5k flat / QLaw + Communities Rise as cheap routes) and the ads clarification — MHMDA bans *surveillance* adtech, not advertising — captured in a new **Ads & money** section (direct sponsorships / contextual / house ads / memberships in; programmatic + pixels out; placement-never-inclusion bright line). Platform-mapping row and privacy bullet updated to match. Still pending Mason: buy the domain, Q7 Circles sub-questions, Q9 "agreed," Q10 metric picks, B.2 scope-cut + hrs/week.
- **8/1/2026 (v0.2)** — Mason's Open Question answers folded in (Q1–6 + Q8 decided; Q9 defined + pending sign-off; Q7 and Q10 developed for discussion): brand = **Antifreeze** with domain reality checked via RDAP (antifreeze.com corporate-locked since 1996; **antifreeze.gay appears available** — new Brand & Naming section with the `{audience}.{city}` two-axis scheme); audience = **gay men** with the link-out coverage rule (pitch, scene tags, scene pages, relocation lane, onramps, voice, newsletter renamed accordingly); adult layer locked to **Q3(b)**; geography = **King County incl. six Eastside cities** added to Appendix F (per Mason's follow-up) with 8/2026 rents + 2-Line reality + an Eastside-reality note and onramp; labor = **(a)+(c)** with a proposed trigger in Q6's note; **"Circles, not forums"** model drafted for Q7 (bounded rooms, stewards, no global feed; link-out cards pulled forward to Phase D); **Success Metrics options menu** added for Q10; roadmap B.1/B.2/D.3/F/H and the platform-mapping table updated to match.
- **7/31/2026 (v0.1)** — Initial brainstorm synthesis from Mason's Overview + three research passes (events landscape, guide/directory content with volatility ratings, peer models/legal/marketing). Premise corrected (SGS alive-but-hollow; contested space). Product concept organized into 5 pillars + freshness architecture + trust model; lettered roadmap A–H dovetailing with [[Setting up the project]]; 10 Open Questions with recommendations; seed inventories appended.