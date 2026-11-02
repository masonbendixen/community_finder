---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1.1
tags:
---

# Bucket GD10 — Adult layer

**Track:** GD — Guide & content · **Size:** S · **Stream:** the Content Pool (claimed: ________)
**Needs:**
- GD3 (done; the 18+ gate is built);
- **the refresh commitment**: one named person who re-checks the beach pages every month while the Denny Blaine litigation runs (OQ1);
- RT2's counsel pass before the section is linked publicly (OQ2).

**Feeds:**
- GD9's two adult lanes, sexual exploration and kink (marked † there), which live in this section;
- GD5's leather and bear scene intros, which link here for the 18+ half;
- **adult events, which have no 18+ flag yet.** The guide's tables have one, but `events` doesn't. [[Bucket EV6 — Structured feed import]] OQ9 proposes a small Stream A follow-up (`events.adult`, inherited from the series; GD3's gate on event pages; left out of the home strip and ICS feeds). Until it lands, 18+ events (Pan Eros's calendar, most of it) stay off the public calendar.

# Context

Implements [[Initial Project Implementation Outline]] bucket GD10: "Q3(b) as decided: bathhouse pages, Denny Blaine/Howell current-rules pages with dated stamps (monthly refresh during litigation), etiquette/safety/law editorial, naturist orgs, 18+ gate."

**The stance is decided** ([[Brainstorming on the website]] Q3, option (b), 8/1):
- documented places (Denny Blaine, Howell, and Volunteer Park as history);
- etiquette, safety and current-rules editorial with dated stamps;
- **no location listings beyond the beaches.** Local guides that need venue relationships and civic legitimacy all stop exactly here, and Sniffies owns the real-time layer anyway.

**Why it's worth doing:** "Nobody covers the adult layer factually." The Denny Blaine litigation is the biggest queer-Seattle story of 2025–26 and appears in zero guides. From Appendix H:
- the Dec 2023 playground proposal was defeated;
- a neighbor lawsuit followed in spring 2025, then a July 2025 abatement order (signage, a fenced clothing-optional zone);
- **the ~7/15/2026 ruling: the park stays open and stays nude**, with a required code of conduct, more ranger staffing, vegetation screening and explicit signage;
- police calls are down 39% since the abatement, and topless is legal parkwide under WA law;
- **an appeal is likely**, so the rules may shift season to season. A dated current-rules page, refreshed monthly, "is exactly the kind of content nobody else will maintain."

**The legal frame** (brainstorm, Trust & Spectrum):
- WA has **no age-verification law as of mid-2026** (HB 2112 didn't pass; FSC v. Paxton greenlit state AV laws, so watch).
- A simple interstitial suffices today, and the site stays far below any "one-third sexually explicit" threshold.
- **Everything stays guide-level factual, nothing explicit.** That's also the bar for every sentence in this bucket.

**What's built:** GD3's `AdultGate` is a sessionStorage interstitial that holds a gated page's body as a template, so nothing renders or fetches before a yes. It's used on every listing marked `adult` and on every page in section `adult`. Cards show an "18+" badge. GD3 OQ4 left `noindex` for adult pages to RT4.

# Where this content lives

| Content | Stored in | Renders at |
|---|---|---|
| The 18+ hub and articles | `guide_pages`, `section = 'adult'`, `adult = true` (GD2 requires it in this section) | `/guide/pages/<slug>` behind the gate, listed at `/guide/sections/adult` |
| Bathhouses | `venues` + `venue_profiles`, `adult = true` (GD4 §5) | `/guide/venues/<slug>`, gated |
| The beaches | `places`, `adult = true` (GD4 §7), each linking its current-rules page | `/guide/places/<slug>`, gated |
| Naturist and sex-positive orgs | `organizations`, `adult = true` where they are | `/guide/orgs/<slug>`, gated |

# The Pool workflow

The same six steps as every Pool bucket ([[Bucket GD5 — Nightlife & scenes content]]): claim → refresh the facts → Claude drafts into the vault (`Content\GD10\<slug>.md`) → check → publish and stamp → check it live.

**Two differences here:**
- **The fact check is about rules and law.** Every rule is sourced (Seattle Parks notices, the court's order or reliable reporting on it, the posted signage) and dated. A rule we can't source doesn't go on the page.
- **The tone check:** factual, plain, unembarrassed, never titillating. Write it the way a good city guide writes about a nude beach in Berlin.

# Scope

**In:** the hub and two cross-cutting articles (§1) · the beach current-rules pages and history (§2) · bathhouse profiles (§3) · the orgs and leather/kink pages (§4) · the monthly refresh routine (§5).

**Out:**
- **any location that isn't a documented, public, long-established place** (Q3(b));
- real-time or user-reported anything (cruising maps, "who's there now");
- photos of any adult place or person (OQ3);
- explicit description of sex acts;
- the AV-law response (OQ4: watch only);
- `noindex` and robots policy (**RT4**).

# Work items

### 1. The hub and cross-cutting articles

- [ ] 1.1 **"18+ in Seattle"** (`adult-seattle`), the hub *(claimed: ______)*:
	- one paragraph on what this section is and isn't: documented places, current rules, etiquette and safety; **not** a cruising map;
	- links to everything below;
	- a line pointing to the apps for real-time, without endorsing any;
	- the "last checked" stamp, prominently.
- [ ] 1.2 **"Etiquette, safety and the law"** (`adult-etiquette-safety-law`) *(claimed: ______)*. **The page that does the most good:**
	- **Consent and etiquette:** no means no, read the room, no photos ever, respect the families at a shared park;
	- **Safety:** go with a friend or tell one; know your exits; PrEP, DoxyPEP and testing (linking GD6, never giving advice);
	- **The law, stated plainly:**
		- WA's public-indecency and lewd-conduct statutes, cited by RCW number and linked, with what they mean in practice;
		- the difference between legal nudity (Denny Blaine's rules) and illegal conduct;
		- that topless is legal statewide;
		- **written by the counsel pass, or reviewed by it** (OQ2).
	- Dated, with a "this is not legal advice" line linking the source statutes.

### 2. The beaches and the history

- [ ] 2.1 **"Denny Blaine: current rules"** (`denny-blaine-current-rules`) *(claimed: ______, plus the refresh owner)*. **The content-moat exemplar.**
	- **At the top:** "Rules as of {date}: open, clothing-optional in {zone}, code of conduct in effect." That's the line people come for, so it goes first.
	- **The rules**, item by item, each sourced: the code of conduct's text or summary, ranger presence, signage, the clothing-optional zone's boundaries as posted.
	- **The case so far:** a short dated timeline (2023 → 2025 lawsuit → July 2025 abatement → ~7/15/2026 ruling → appeal status).
	- **Getting there:** transit (the 2 Line at Judkins Park plus a bus or walk, verified), no parking promises.
	- **Etiquette specific to this beach.**
	- **Tier `monthly`** while the litigation runs (§5). The page says plainly: "We re-check these rules every month while the case continues."
- [ ] 2.2 **The Denny Blaine place row** (`places`, `adult = true`; GD4 §7 creates it): a short neutral description and a link to 2.1. The rules live only on 2.1, so there's one place to update.
- [ ] 2.3 **"Howell Park"** (`howell-park`) *(claimed: ______)*: smaller, more hidden, no facilities, historically "more for the boys." Its current status (does any rule or ruling touch it?), verified. Tier `monthly` alongside Denny Blaine, since a ruling at one may touch the other. Plus its place row.
- [ ] 2.4 **"Volunteer Park: a history"** (`volunteer-park-history`) *(claimed: ______)*. **History, explicitly not a current location** (Q3(b)). Its place in Seattle's gay history and how the city policed it, with sources (the Seattle Municipal Archives, the Northwest LGBT history projects, verified). The park's queer present is daytime and public (Trans Pride is held there), and the page says so.

### 3. Bathhouses (venue profiles, `adult = true`)

- [ ] 3.1 **Steamworks Seattle** *(claimed: ______)*:
	- what it is, hours (24/7 per the appendix; verify), membership and ID rules, and its posted etiquette page;
	- the profile's crowd text is factual and short;
	- website `swbaths.com` (verified live 11/1).
	- **Tier `quarterly`.**
- [ ] 3.2 **Club Z** (Zodiac Social Club, est. 1976) *(claimed: ______)*:
	- the same fields, plus its 50th anniversary in 2026;
	- **a watch item:** the building was listed for sale in 2/2026. The profile says what's known and dated; a sale or closure moves it to the graveyard with its note;
	- website **`thezclub.com`** (clubzseattle.com is a parked domain, per the 11/1 feed research);
	- its public Google Calendar (empty as of 11/1) is EV6's to import, with its events marked 18+ once events can be (EV6 OQ9).
	- **Tier `quarterly`**, monthly while the sale watch is open.
- [ ] 3.3 **"Bathhouses in Seattle"** (`bathhouses-seattle`) *(claimed: ______)*: a short orientation to what to expect, the house rules both post, safety, and links to the two profiles. History (the Seattle bathhouses that closed) can link the graveyard.

### 4. Orgs, leather and kink

- [ ] 4.1 **Naturist orgs** *(claimed: ______)*. Each needs verifying before it's listed: SLUGS, the Lake Bronson club, and any Tiger Mountain naturist groups. Listed as `organizations`, adult only where they are (a family naturist club may not be). If one can't be verified as active, it isn't listed.
- [ ] 4.2 **Sex-positive orgs** *(claimed: ______)*:
	- **Pan Eros Foundation** (active), its Gallery Erato, and the **Seattle Erotic Art Festival** (5/1–3/2026; the next year's dates verified);
	- **the Center for Sex Positive Culture is listed CLOSED in the appendix, which needs primary confirmation**: graveyard if confirmed;
	- Pan Eros's public calendar is EV6's richest feed, so its events come in that way, **once events have an 18+ flag** (EV6 OQ9).
- [ ] 4.3 **"Leather and kink in Seattle"** (`leather-kink-seattle`), the deep dive *(claimed: ______)*:
	- the scene's institutions: the Cuff, the Seattle Eagle, Doghouse Leathers as the retail and community hub;
	- the clubs and titles, verified current;
	- the events calendar's leather-tag view;
	- etiquette for a first time, from someone who's been;
	- **NW Leather Celebration's 2026 edition was in Sacramento**, so it's history-or-travel, not local.

	GD5's leather scene intro links here for anything 18+.
- [ ] 4.4 **GD9's adult lanes**: the sexual-exploration and kink "New to…" pages are written in GD9 but live here (`section = 'adult'`). This bucket's gate checks them too.

### 5. The monthly refresh

- [ ] 5.1 **Owner:** one named person (OQ1), on the claim line of 2.1 and 2.3.
- [ ] 5.2 **The routine, first week of each month:**
	1. check Seattle Parks' Denny Blaine notices;
	2. search for news on the case (appeal filed? ruled?);
	3. if anything changed, update 2.1 and 2.3 the same day;
	4. **re-stamp either way.** The stamp is the promise.
- [ ] 5.3 **When the litigation ends** (an appeal decided or none filed in time), the pages drop to tier `quarterly` and the page's "every month" line comes out.
- [ ] 5.4 **The Club Z sale watch** rides the same monthly check until it resolves.
- [ ] 5.5 **The AV-law watch:** each January, the owner checks whether WA (or a court) has changed the age-verification picture, and records it in this doc's change log. A change re-opens GD3 OQ1.

### 6. Gates

- [ ] 6.1 **The tone read:** every page reads as factual guide content. Nothing explicit; nothing a reasonable reader would call pornographic.
- [ ] 6.2 **The location rule:** no page names or describes a place outside the documented set (the two beaches, Volunteer Park as history, the bathhouses as businesses).
- [ ] 6.3 Every rule and legal statement is sourced and dated on the page.
- [ ] 6.4 **Every page in section `adult` has `adult = true`** (GD2 enforces it), and opens behind the gate in a fresh tab (check by hand).
- [ ] 6.5 No photos on any page or listing in this layer (OQ3).
- [ ] 6.6 RT2's counsel pass has signed off on 1.2 and the section's framing before the guide home's 18+ link goes live.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Who owns the monthly refresh?** The outline makes it a precondition: "needs GD3 + the refresh commitment." A stale Denny Blaine page is worse than none, because people will rely on it. *Rec: name one person before 2.1 publishes. If nobody can commit to monthly, publish the page with a "status as of" line only, and no rules detail.*
2. **Wait for the lawyer?** The law paragraph (1.2) is the only place this bucket states law, and the 18+ section is the part of the site most likely to draw a complaint. *Rec: the pages can be drafted and published unlinked (reachable only by URL) now, but the guide home's 18+ link waits for RT2's counsel pass, which is already scoped to the MHMDA posture and policies.*
3. **Photos.** A beach or bathhouse photo invites the question "who's in it?" *Rec: no photos anywhere in this layer, not even of empty buildings. The text and the stamp carry it.*
4. **What if WA passes an age-verification law?** FSC v. Paxton means a state law would likely stand. Our share of explicit content is tiny, and the bills have so far targeted sites where a third or more is sexual material. *Rec: watch only (5.5). If a law passes, re-read its threshold. We'd likely stay out of scope, and if not, the section can go behind sign-in, which GD3's gate makes a small change.*
5. **The Center for Sex Positive Culture.** The appendix lists it as closed, unconfirmed. *Rec: confirm from a primary source (its own site or a statement) before the graveyard entry. If it can't be confirmed either way, leave it out.*

# Change Log

- **11/1/2026 (v0.1.1)** — Corrected the Feeds line: events have **no** 18+ flag yet (only the guide's tables do). Adult events, including Pan Eros's and Club Z's calendars, wait on EV6 OQ9's proposed follow-up.
- **11/1/2026 (v0.1)** — Initial draft from the outline's GD10 blurb, the brainstorm's Q3(b) decision, Appendix H (the Denny Blaine timeline) and the adult inventory in the appendix. Also drew on GD3's built 18+ gate and the 11/1 feed research (Club Z's real domain and public calendar; Steamworks' site). Hub, etiquette/safety/law, two beaches, Volunteer Park as history, two bathhouses, orgs, a leather/kink deep dive, and a monthly refresh routine. Five open questions.
