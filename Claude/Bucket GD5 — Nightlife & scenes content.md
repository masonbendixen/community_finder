---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1
tags:
---

# Bucket GD5 — Nightlife & scenes content

**Track:** GD — Guide & content · **Size:** M · **Stream:** the Content Pool (claimed: ________)
**Needs:** GD3 (done: the pages these publish into) · **GD4b's venue, profile and org rows** (§5–§6 and §9 there), since every page here links to them by slug. Drafting can start today; *publishing* waits on the rows.
**Feeds:**
- milestone **M2**: "≥1 Pool section published with freshness stamps";
- milestone **M3**: "GD3 live with GD5-minimum content";
- RT4's target query family ("gay bars seattle");
- RT3's newsletter, which links scene pages.

# Context

Implements [[Initial Project Implementation Outline]] Content Pool bucket GD5: "Venue crowd profiles, scene landings (bears/leather/jocks/sober/elders/geeks/BIPOC), the closure graveyard, 'where this scene talks' links. The durable low-volatility editorial core."

From [[Brainstorming on the website]]:
- **Pillar 2.1, nightlife venue profiles:** "the 'which bar is for which crowd' layer." Crowd and vibe are LOW-volatility, durable content; ownership and status are HIGH. Include the closure graveyard: history, a trust signal, and SEO nobody else has.
- **Pillar 2.2, scenes:** landing pages per subculture we can cover honestly. "Every scene gets the same respectful, factual treatment." Sapphic, trans and QTPOC scenes get link-out cards to their own curators, with credit.
- **Pillar 5.5:** "where this scene talks" link-out cards: community value with zero UGC risk.
- **The incumbent gap:** The Ticket's "First-Timer's Guide to Queer Nightlife" was written once and never maintained, and has four confirmed factual errors. Being the maintained version of that page is the point.

**The rules this content follows:** [[Bucket GD1 — Taxonomy & editorial foundations]] §2 (scene tags and coverage posture), §5 (date every volatile claim), §6 (voice: gay men for gay men, warm, direct, factual, zero moralizing, never euphemize, never leer) and §7 (the venue-profile and scene-landing templates). The outline's OQ9 sets the authorship model: **Claude drafts from the research; humans verify the feet-on-the-ground facts before publish; every page stamps its verified date.**

# Where this content lives

| Content | Stored in | Renders at |
|---|---|---|
| A venue's crowd profile | `venue_profiles.crowd_vibe`: **plain text**. Blank lines become paragraphs (`white-space: pre-line`), but markdown and links don't render, so a profile names things and the scene intro links them | `/guide/venues/<slug>` |
| A scene's intro | `guide_pages`, `section = 'scenes'`, **slug = the scene tag's slug** (GD2's page-slug convention) | `/guide/scenes/<slug>` |
| Nightlife articles | `guide_pages`, `section = 'scenes'`, free slugs | `/guide/pages/<slug>` and `/guide/sections/scenes` |
| Closure notes | `venues.status = 'closed'` + `venue_profiles.closure_note` | `/guide/graveyard` and the venue's own page |
| "Where this scene talks" | `guide_links` (GD4 §9) | profile and scene pages |

Links between pages are plain relative markdown links (`/guide/venues/the-cuff-complex`), keyed by slug (GD2 OQ5).

# The Pool workflow (the same for every Pool bucket)

1. **Claim** a page by writing your name on its line below.
2. **Refresh the facts.** The appendix is a 7/31/2026 snapshot. Re-check every fact the page will state against a primary source: the venue's own site or social, a recent news item (CHS, SGN), or a visit.
3. **Draft.** Claude drafts from the research into the vault, at `Content\GD5\<slug>.md` (OQ1), in the GD1 §6 voice, with every volatile claim dated.
4. **Feet-on-the-ground check.** Someone who has actually been there reads the crowd and vibe claims and corrects them. This is the step a search engine can't do, and it's why the page is worth reading.
5. **Publish** in the admin editor:
	- for a page: a `guide_pages` row with the slug, section, title, body, scene tags and `adult`; set `draft` off and `publish_at` to now; `status = open`;
	- for a profile: paste into `crowd_vibe`;
	- set `last_verified_at` to today. **That is the verification act** (GD1 §5).
6. **Check it live:** the page renders, every in-body link clicks through to a real page, and the "Verified {Month Year}" stamp shows.

# Scope

**In:** the crowd profiles for GD4 §5's venues · intros for every scene tag (8 full, 3 link-out) · four nightlife articles · the graveyard's history notes · "where this scene talks" coverage per scene.

**Out:**
- the 18+ layer (bathhouses, cruising, kink deep-dives: **GD10**; the leather intro here stays at the bar-and-social level and links into GD10);
- health (**GD6**);
- neighborhood rundowns, including White Center as the second gayborhood (**GD7**, which links here);
- the recurring-groups directory and onramps (**GD8**);
- photos (CM3);
- any ranking or "best of" list (OQ4).

# Work items

### 1. Venue crowd profiles (`venue_profiles.crowd_vibe`)

**What a profile says**, in this order and in short paragraphs:
1. **Who goes:** the crowd, honestly and accurately, never aspirationally;
2. **The vibe**, and the nights that matter;
3. **When to go:** busy nights, quiet nights, and the age skew if it's real and well known;
4. **Practical notes:** cover, dress codes (leather nights), cash-only, accessibility if known, food;
5. **History in a line**, dated ("est. 1983"; "sold to the Cuff's owners, June 2026").

**Target length:** 80–200 words. **Tier:** `quarterly`, the profile default; ownership and status are watched monthly on the venue row.

- **1.1 Capitol Hill core** *(claimed: ______)*:
	- [ ] The Cuff Complex
	- [ ] Seattle Eagle
	- [ ] Diesel
	- [ ] Pony
	- [ ] CC Attle's
	- [ ] Madison Pub
	- [ ] Crescent Lounge
	- [ ] Union
- **1.2 Dance, drag and shows** *(claimed: ______)*:
	- [ ] Neighbours
	- [ ] Massive
	- [ ] Kremwerk / Timbre Room / Cherry
	- [ ] Queer/Bar (re-check after the remodel)
	- [ ] Unicorn / Narwhal
	- [ ] Julia's on Broadway / Le Faux
	- [ ] Wildrose (a `sapphic` anchor we list, labeled accurately)
- **1.3 Beyond the Hill** *(claimed: ______)*:
	- [ ] Lumber Yard Bar
	- [ ] Boombox
	- [ ] Changes
- **1.4 Queer nights at mainstream venues** *(claimed: ______)*. Describe the night, not the whole venue:
	- [ ] Southgate (Pride Skate)
	- [ ] Chop Suey (Flammable Sundays)
	- [ ] Comet (Caliente)
	- [ ] Rough & Tumble
	- [ ] Stoup (Board Gayme Night)
	- [ ] El Sueñito
	- [ ] Charlie's Queer Books
	- [ ] Great American Casino (Royal Flush)
	- [ ] the rest of GD4 §5.2 as entered
- **1.5 Graveyard notes** *(claimed: ______)*. Each closed venue's `closure_note` gets one or two dated sentences: what it was, when and why it closed, and where its crowd went if that's known. Framed as history, not loss-porn (GD1 §6):
	- [ ] The Comeback
	- [ ] R Place
	- [ ] Re-bar
	- [ ] Purr
	- [ ] Basic Plumbing / Tribe
	- [ ] Double Header

### 2. Scene intros (`guide_pages`, section `scenes`, slug = scene tag)

**What an intro says**, 250–600 words:
- what the scene is here, in King County, now;
- where it gathers, linking venue profiles and series;
- the regular nights, linking series pages;
- the orgs, linking org profiles;
- how to walk in for the first time;
- where it talks online (the link-out cards render from `guide_links`; the prose can name them too).

The scene page already lists venues, orgs, places and upcoming events beneath the intro (GD3 §3.3), so **the intro is the human part**, not a duplicate list.

- **2.1 Full-coverage scenes** *(claimed: ______)*:
	- [ ] `bear`: Diesel, the Cuff, Lumber Yard and Bear Social; White Center's bear-neighborhood character
	- [ ] `leather`: the Eagle, the Cuff, CC's Leather Social, Doghouse Leathers, Safeword. **Bar-and-social level only;** one line links to GD10's 18+ leather and kink page
	- [ ] `jock`: the 34+ leagues and how seasons work, climb nights, Frontrunners' Monday mini-run. Links GD8's sporty onramp
	- [ ] `sober`: recovery meetings (gay AA appears on no queer calendar anywhere), Qu-Art, sober-friendly socials and venues with real NA drinks. Links GD8's sober onramp and GD9's sober lane
	- [ ] `elders`: GenPride and Pride Place, Rainbow Elder Play & Connect, the 40+ onramp
	- [ ] `geek`: Seattle Gaymers, Board Gayme Night, Queer Geek!, Phoenix Comics
	- [ ] `furry`: *research pass needed*. The appendix has no furry venues or orgs. If nothing local and verifiable turns up, the intro says so honestly and points to regional cons and groups, or the tag is disabled until there's content (OQ3)
	- [ ] `bipoc`: PNW Black Pride (The Gathering; August weekend), Latinx Pride, Entre Hermanos, UTOPIA WA, Eastside QTPOC Collective, Beardslee Latin Night. Organizers credited by name
- **2.2 Link-out scenes** (`sapphic`, `trans`, `qtpoc`) *(claimed: ______)*. Short, about 150 words: what we list (the shared anchors: Wildrose, Trans Pride, T4T and so on), why we point elsewhere for depth (the coverage rule, said plainly and warmly), and credited link-out cards (Sapphie Taffy, Seattle Dyke Alliance, Gender Justice League, Ingersoll, Lavender Rights Project, UTOPIA WA):
	- [ ] `sapphic`
	- [ ] `trans`
	- [ ] `qtpoc`

### 3. Nightlife articles (`guide_pages`, section `scenes`)

- [ ] 3.1 **"Seattle's gay bars: which bar is for which crowd"** (`gay-bars-seattle`) *(claimed: ______)*. The money page: RT4's top target query, and the page The Ticket got wrong.
	- It covers every open queer venue in one or two lines each, grouped by crowd (leather and bears · dance · drag and shows · low-key · beyond the Hill), each linking its profile.
	- It ends with "Gone but not forgotten," which links the graveyard.
	- **Tier `monthly`**, because it names many venues whose status moves.
- [ ] 3.2 **"Drag in Seattle"** (`drag-in-seattle`) *(claimed: ______)*: the regular shows (Le Faux, Mimosas Cabaret, Fridays Are a Drag, T4T, Centerfold, Judy, Royal Flush), brunches, etiquette (tip the queens: bring small bills), and where the big names play. Performers are named as billed acts (OQ2).
- [ ] 3.3 **"Your first night out on Capitol Hill"** (`first-night-out-capitol-hill`) *(claimed: ______)*: a walkable route by mood, practical notes (Link's Capitol Hill station, last trains, the rideshare crush at 2 AM), and the safety basics (buddy up, watch your drink, Seattle Police's non-emergency line versus 911). Links GD7's neighborhood page.
- [ ] 3.4 **"Gone but not forgotten: Seattle's closed gay bars"** (`closed-gay-bars-seattle`) *(claimed: ______)*: the history article over the graveyard. R Place, Re-bar, the Comeback, Purr, Basic Plumbing, Double Header, with dates and sources. **Tier `annual`.** The graveyard page itself has no intro slot, so this article is linked from 3.1 and from GD3's graveyard page (OQ5).

### 4. "Where this scene talks"

- [ ] 4.1 For each full scene, confirm GD4 §9 found its online gathering places: a Discord, a Bluesky account, an Instagram, a Meetup or a Facebook group. Fill gaps *(claimed: ______)*. **Never invent one.** A scene with none says so in its intro ("This scene mostly meets in person; here's where").

### 5. Gates

- [ ] 5.1 Every page and profile has been through all six workflow steps, including a named feet-on-the-ground checker (the claim line records who).
- [ ] 5.2 Every in-body link resolves to a live page (click through; or run `check-guide-links` if it exists, OQ6).
- [ ] 5.3 The voice check, a second read against GD1 §6:
	- no undated superlatives ("newest," "recently");
	- no aspirational tags;
	- no leering and no euphemism;
	- no patrons named;
	- sister scenes credited.
- [ ] 5.4 **M2 minimum:** at least one scene intro plus its venues' profiles published with stamps. **M3 minimum:** all of §2.1, 3.1, and §1.1–1.2.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Where do drafts live before they're published?** The database is where a page finally lives, but drafting there means editing long text in a form field, and reviewing it means opening each page's preview. Drafting in this vault (`CommunityFinder\Content\GD5\<slug>.md`) lets you read and comment the same way you answer these questions, and Claude can revise in place. Once approved, the text is pasted into the admin editor and the vault file is marked "published {date}." *Rec: vault drafts. If the other Pool contributors can't reach the vault, use a shared doc folder instead, same shape.*
2. **Naming people.** GD1 §6.9 allows people "only with consent or public-figure context." Drag performers and DJs are billed publicly under their stage names. *Rec: name performers as billed acts, in their stage names. Never name patrons, bartenders or anyone's real name behind a persona.*
3. **The `furry` scene with nothing local to list.** The tag was seeded from the brainstorm's list, but the research found no King County furry venues or orgs. *Rec: do the research pass. If it's thin, publish an honest short intro pointing to regional groups, or turn the tag off (it's admin data) until there's something real.*
4. **No rankings.** "Best gay bars" listicles are what every SEO shell publishes. Ours says which bar is for which crowd. That's more useful, ages better, and never puts us in the business of ranking venues that might sponsor us later (GD1 §8.2). *Rec: "good for" framing, never numbered rankings.*
5. **An intro for the graveyard page.** GD3's graveyard page lists closed venues with their notes but has no editorial slot. *Rec: the history lives in article 3.4, and the graveyard page gets a one-line link to it ("The stories behind these closures →"). That's a one-line GD3 edit, made when 3.4 publishes.*
6. **A link checker.** Every Pool bucket links pages by slug, and a renamed or missing slug is a silent broken link. *Rec: a small test_helper command, `check-guide-links`, that reads every published page's markdown, finds the `/guide/...` and `/events/...` links, and reports any that don't resolve. About a day of code in Stream B. Whichever Pool bucket publishes first asks for it.*

# Change Log

- **11/1/2026 (v0.1)** — Initial draft from the outline's GD5 blurb, the brainstorm's Pillars 2.1, 2.2 and 5.5 and Appendices B and C, GD1's posture and voice, and GD2/GD3's storage and routes. Defines the Pool workflow the other content buckets repeat. Six open questions.
