---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1.2
tags:
---

# Bucket GD8 — Freeze directory & "pick your onramp"

**Track:** GD — Guide & content · **Size:** M · **Streams:** two halves.
- **The content** is in the Content Pool (claimed: ________).
- **The onramp chooser, "GD8-flow,"** rides Stream B (owner: Mason; light code, client only).

**Needs:** GD2 + GD3 (done) · GD4b §6's organization rows and GD4a §4.8's series skeleton. The onramps are lists of real groups and regular nights, so they publish once those rows exist.
**Feeds:**
- the brand's namesake promise, "past the Freeze, into community";
- the Success Metrics **Mission** row ("I joined something because of the site");
- GD7's first-90-days page;
- GD9's "start where you are" hub;
- CM4 later: a circle is an onramp's natural ending.

# Context

Implements [[Initial Project Implementation Outline]] bucket GD8: "The recurring-groups directory (leagues/choruses/climb nights/recovery/faith/professional) + the guided onramp flow (sporty/nerdy/sober/40+/BIPOC/new-in-town/Eastside → matched groups + low-barrier entry events). The wedge, and the brand's namesake." The outline's Stream B list carries "GD8-flow done — the 'pick your onramp' interactive bit (light code assist for the Pool's Freeze content)."

From [[Brainstorming on the website]] Pillar 4:
- **The Freeze is real.** UW Psychiatry publishes tips, and Axios and the Seattle Times cover it as loneliness infrastructure.
- **The known cure is recurring shared-activity groups, not one-off events.** "It's tough to make real friends through Meetup events alone." Summer is the joining season.
- **Nobody aggregates the queer version.** There are 34+ sports leagues, choruses, climb nights, book clubs, gaymers, figure drawing, faith communities, professional groups and volunteer orgs. And **recovery meetings: gay AA exists and appears on no queer calendar anywhere.**
- **"Pick your onramp":** "a guided path by personality/interest (sporty / arty / nerdy / sober / outdoorsy / 40+ / BIPOC / new-in-town / Eastside-based) → 2–3 matched recurring groups + this month's low-barrier entry events." Example: Frontrunners' Monday 3-mile mini-run at Cal Anderson. "Structured 'join something' advice beats the generic listicle every outlet writes."
- **The Knotty Yoga bridge:** a wellness, fitness and acro studio belongs in a directory of recurring queer movement spaces, **listed by the same rules as everyone else.**

**What already exists:** GD3's `/guide/orgs` is the directory's data surface, with a kind filter (leagues, choruses, recovery, faith, professional, social), scene and neighborhood filters, and org profiles. Series pages (`/events/series/<slug>`) are the "regular nights." This bucket adds the **editorial on top** and the **chooser** that routes a person to it.

# Where this content lives

| Content | Stored in | Renders at |
|---|---|---|
| The Freeze hub and articles | `guide_pages`, `section = 'freeze'` | `/guide/pages/<slug>`, listed at `/guide/sections/freeze` |
| One page per onramp | `guide_pages`, `section = 'freeze'`, slug `onramp-<name>` | `/guide/pages/onramp-<name>`, and as the chooser's results |
| The chooser | client code only (§3) | `/guide/onramp` |

# The Pool workflow

The same six steps as every Pool bucket (spelled out in [[Bucket GD5 — Nightlife & scenes content]]): claim → refresh the facts → Claude drafts into the vault (`Content\GD8\<slug>.md`) → check → publish in the admin editor and set `last_verified_at` → check it live.

**The check here:**
- that every group named is meeting **now**, at the stated time and place;
- and the "your first time" notes come from someone who has actually shown up. That's the part a listicle can't fake.

# Scope

**In:** the Freeze hub and two practical articles (§1) · nine onramp pages (§2) · the chooser at `/guide/onramp` (§3) · tests for the chooser.

**Out:**
- the org and series rows themselves (**GD4**);
- circles and looking-for-group (**CM4**, gated);
- any tracking of who picks what (OQ3);
- server changes (none needed, OQ1).

# Work items

### 1. The Freeze hub and articles (section `freeze`)

- [ ] 1.1 **"Beating the Seattle Freeze"** (`beat-the-freeze`), the hub *(claimed: ______)*:
	- it's real (cite UW Psychiatry and one or two news pieces, dated);
	- why one-off events don't fix it and recurring groups do;
	- **"pick one thing and show up three times"**;
	- summer is the joining season, though most leagues also run fall and winter seasons;
	- a big **"Pick your onramp →"** button to the chooser;
	- links to the nine onramp pages and the org directory by kind.
- [ ] 1.2 **"When to sign up: the joining calendar"** (`joining-calendar`) *(claimed: ______)*. Month by month: when each league's registration opens (softball, kickball, soccer, flag football, hockey, swim), chorus auditions, climb-night rotations, and the big volunteer moments (Pride in June, Dining Out For Life in August, the AIDS Walk). **Tier `annual`, re-checked each January**, since registration dates are the most-asked, least-findable fact in this whole layer. Every date is verified with the org.
- [ ] 1.3 **"Volunteering"** (`volunteer-seattle`) *(claimed: ______)*: VolunQueers, Lambert House, GenPride, Seattle Pride's volunteer corps, Dining Out For Life, the Center. Volunteering is the most reliable Freeze-breaker for people who don't want to play a sport.

### 2. The nine onramp pages (`onramp-<name>`, section `freeze`)

**What an onramp page says**, 250–500 words:
1. **Who this is for**, in one line;
2. **Two or three matched recurring groups**, each with what they do, when, where and how to start, linking the org profile;
3. **This month's low-barrier entry points**: the regular nights or first-timer sessions, linking series pages. The chooser also shows live upcoming events (§3);
4. **"Your first time":** what to expect, what to bring, and how not to be the person standing by the wall. From someone who's been;
5. **Related:** the scene intro, and GD9's lanes where relevant.

**Tier `annual`**, with the groups' rows carrying their own freshness.

- [ ] 2.1 `onramp-sporty` *(claimed: ______)*: Frontrunners' Monday mini-run (the brainstorm's perfect example), OutLoud's kickball and dodgeball (no skill needed), ORCA swim and climb nights. Scene `jock`.
- [ ] 2.2 `onramp-arty` *(claimed: ______)*: the choruses (Seattle Men's Chorus, Rainbow City Performing Arts; audition seasons), queer figure drawing, Jet City Improv, Charlie's book clubs.
- [ ] 2.3 `onramp-nerdy` *(claimed: ______)*: Seattle Gaymers, Board Gayme Night at Stoup, Queer Geek!, Phoenix Comics, trivia nights. Scene `geek`.
- [ ] 2.4 `onramp-sober` *(claimed: ______)*: gay AA (Gay Men in Recovery; Gay & Lesbian Beginners), linking seattleaa.org's meeting finder for current times rather than listing times that change; Qu-Art's sober artists' night; sober-friendly socials. Scene `sober`. Links GD9's sober lane.
- [ ] 2.5 `onramp-outdoorsy` *(claimed: ______)*: OutVentures, Queer Mountaineers (the queer night at every climbing gym, the strongest entry point here), SkiBuddies, and Seattle Queer & Ally Hiking. The feed research on 11/1 found that last group's Meetup has had no events since 1/31/2026, so confirm it's active. **Different Spokes is out**: there's no Seattle club now.
- [ ] 2.6 `onramp-40-plus` *(claimed: ______)*: GenPride, Rainbow Elder Play & Connect, the choruses (a famously multigenerational onramp), Banter Date's age-segmented socials, and Mature Friends (a 40+ club, \$40 a year; GD9's research found a full November 2026 calendar). Scene `elders`. Links GD9's out-later-in-life lane (`new-to-out-later`). Leave out Seattle Prime Timers unless someone confirms it's active: its site's certificate has expired.
- [ ] 2.7 `onramp-bipoc` *(claimed: ______)*: PNW Black Pride's The Gathering, Entre Hermanos, UTOPIA WA, Eastside QTPOC Collective, Beardslee Latin Night. Organizers credited by name; labels as the organizers use them. Scene `bipoc`.
- [ ] 2.8 `onramp-new-in-town` *(claimed: ______)*: Another Gay Social Club's monthly socials (its Meetup group returned "not found" on 11/1, so find where it organizes now), the two lowest-barrier sports, one volunteer shift, and GD7's first-90-days page.
- [ ] 2.9 `onramp-eastside` *(claimed: ______)*: the monthly Eastside LGBTQ+ Happy Hour, the East King County All Ages Meetup, VALA figure drawing in Redmond, Rainbow on the Eastside, Eastside QTPOC Collective, and the 2 Line to Capitol Hill for the rest. Links GD7's Eastside page. *Mason's home turf: he's the natural checker.*

### 3. GD8-flow: the chooser (Stream B code, `ui/src/app/pages/guide/onramp/`)

- [ ] 3.1 `onramp-catalog.ts`: the nine onramps as data. Each has a slug, title, one-line pitch, its page slug (`onramp-<name>`), and an **event filter** for the live strip, as a category slug (`sports`, `arts`) and/or a scene slug (`jock`, `geek`, `sober`, `elders`, `bipoc`). `new-in-town` and `eastside` carry no filter, so they show the soonest events (Eastside's limited to its region). **Spec:** every entry's page slug matches the `onramp-` convention; every filter slug is in the seeded vocabularies (checked against the mock's lists).
- [ ] 3.2 `OnrampPage` at **`/guide/onramp`**, a literal route placed before any `:slug` routes in `guide.routes.ts` (the route-ordering hazard). **Three short steps, all optional after the first:**
	1. **"What sounds fun?"**: pick up to two of sporty · arty · nerdy · outdoorsy · sober-friendly;
	2. **"Where are you based?"**: Seattle · Eastside · South King County;
	3. **"Any of these fit?"**: 40+ · BIPOC · new in town.

	**Results:** one to three onramp cards (the pitch, "Read the onramp →"), each with a live **"Coming up"** strip of the next four matching events. That's `GuideAccess`'s existing upcoming-events read with the onramp's filter, then narrowed client-side to the chosen region through the neighborhood vocabulary's `region`. It stays quiet-empty on failure, per the GD3 isolation pattern.
	- **The answers live only in the URL** (`/guide/onramp?fun=sporty,nerdy&base=eastside&also=new`), so results are shareable and the back button works. Nothing is stored or sent anywhere (OQ3).
	- **Matching rule:** each chosen interest gives its onramp; "Eastside" adds `onramp-eastside`; each "also" adds its onramp. Results are capped at three, interests first.
- [ ] 3.3 **Specs** (`onramp-page.spec.ts`, offline data):
	- each step's choices map to the right cards;
	- the cap of three;
	- the URL round-trips;
	- the region narrowing (an Eastside pick hides a Capitol Hill event);
	- a failed events read leaves the cards and hides the strip;
	- no answers shows the chooser, not empty results.
- [ ] 3.4 **Links in:** the guide home's "Beat the Freeze" tile gets a second link, "Pick your onramp"; the hub page (1.1) carries the button. One-line edits in GD3's own files (same stream).
- [ ] 3.5 **Gates:** vitest green with the new count; `ng build` for all three configurations; the guide chunk's growth recorded (it should be a few kB).

### 4. Gates (content)

- [ ] 4.1 Every group named on every page has been confirmed as meeting now, with the stated cadence (recorded on the claim line).
- [ ] 4.2 Every "your first time" section has a named contributor who has attended.
- [ ] 4.3 Knotty Yoga, if listed anywhere here, appears only under the same rules as any studio: no ranking, no special framing (GD1 §6.11).
- [ ] 4.4 Every in-body link resolves; the chooser's nine page slugs all resolve to published pages before the chooser is linked from the guide home.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **A chooser that's just client code, or a matching engine?** The chooser could score every org against your answers using tags and kinds. That needs server work and gives you whatever the data says, good or bad. The alternative is that each onramp is a hand-written page naming the two or three groups a person would actually recommend, and the chooser just routes you to the right pages, with live events added. *Rec: hand-picked pages plus a client-only chooser. It's better advice, it's a day of code instead of a week, and the directory still exists for browsing everything.*
2. **Which onramps at launch?** The brainstorm names nine. *Rec: all nine. Each is one page, and the chooser needs all its destinations to exist before it's linked.*
3. **Counting how many people use the chooser.** It would be nice to know, but the site's posture is no tracking (MHMDA-minimal, "we can't leak what we don't collect"). *Rec: count nothing. The Mission metric comes from the newsletter's quarterly one-click poll (RT3), as the Success Metrics menu already says.*
4. **Recovery meeting times.** Gay AA meetings are the unserved wholesome layer, but their times and places change, and the AA intergroup keeps the authoritative list. *Rec: name the meetings and link seattleaa.org's finder and the Meeting Guide app for current times, rather than copying times we'd have to keep fresh.*

# Change Log

- **11/1/2026 (v0.1.2)** — From [[Bucket GD9 — New to… life-transition guides]]'s research:
	- 2.6 adds Mature Friends and names GD9's lane slug;
	- Seattle Prime Timers stays off unless someone confirms it's active.
- **11/1/2026 (v0.1.1)** — Notes from [[Bucket EV6 — Structured feed import]]'s research:
	- Different Spokes is out (no Seattle club);
	- Seattle Queer & Ally Hiking's Meetup has been quiet since 1/31/2026, and Another Gay Social Club's Meetup group is gone, so confirm where both organize now;
	- Queer Mountaineers runs every climbing gym's queer night.
- **11/1/2026 (v0.1)** — Initial draft from the outline's GD8 blurb and its Stream B "GD8-flow" line, and the brainstorm's Pillar 4 and Appendix D. Three Freeze articles, nine onramp pages, and a client-only chooser at `/guide/onramp` with live events and URL-only answers. Four open questions.
