---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1
tags:
---

# Bucket GD7 — New to Seattle

**Track:** GD — Guide & content · **Size:** M · **Stream:** the Content Pool (claimed: ________)
**Needs:** GD3 (done: the neighborhood pages and the `new-to-seattle` section). It links into GD5 (scenes), GD6 (health) and GD8 (the Freeze onramps), but doesn't wait for them; a link to an unpublished page just waits to go live.
**Feeds:**
- the GD8 "new in town" and "Eastside-based" onramps;
- GD9's "new to a larger gay community" lane;
- RT3's newsletter, which features a neighborhood each month;
- RT5's relocation-wave outreach.

# Context

Implements [[Initial Project Implementation Outline]] Content Pool bucket GD7: "Neighborhood rundowns (incl. the Eastside as first-class), transit/traffic honesty, first-90-days checklist, moving-because-of-your-state lane with trans-rail link-outs."

From [[Brainstorming on the website]] Pillar 3 ("a relocation guide, not a tourist guide") and Appendix F:
- **Why now:** in May 2026 the Seattle LGBTQ Commission petitioned the mayor over people relocating from red states "by the thousands." Every existing guide is a tourist guide; nobody serves people *moving here*.
- **Neighborhoods:** "with queer character, 2026 1BR rent ranges (always ranged + dated; cross-source spread is ±$300), and light-rail access."
	- Capitol Hill is the infrastructure gayborhood.
	- White Center is "the clearest second gayborhood."
	- **The Eastside is where a large share of gay King County actually lives.** It has no venues but plenty of meetups, every incumbent ignores it, and since 3/28/2026 the 2 Line puts Bellevue and Redmond one direct train from Capitol Hill.
- **Transit and traffic honesty:**
	- the 1 Line runs Lynnwood↔Federal Way;
	- the 2 Line fully connected across the lake on 3/28/2026 (Judkins Park station matters to queer renters);
	- Revive I-5 cuts capacity through 2027;
	- 520 is tolled;
	- West Seattle is bridge-dependent.
- **First 90 days:**
	- get on PrEP and testing rails;
	- pick your scenes;
	- join one recurring thing;
	- the paperwork;
	- where queer housing leads surface (the ~13k-member Seattle Queer Housing Facebook group, as a link-out).
- **The "moving here because of your state" lane:** build the gay men's version deeply, and link out with credit to the trans-specific rails (Ingersoll, Lavender Rights Project, the 2026 King County Trans Resource & Referral Guide).

**Rules:** [[Bucket GD1 — Taxonomy & editorial foundations]] §3 (the 25 neighborhoods in five regions, the Eastside first-class and never "other"), §5 (rents are `quarterly`, always ranged and dated; neighborhood profiles are `annual`) and §6 (date every claim; no undated "newest").

# Where this content lives

| Content | Stored in | Renders at |
|---|---|---|
| A neighborhood's rundown | `guide_pages`, `section = 'new-to-seattle'`, **slug = the neighborhood's slug** (GD2's page-slug convention) | `/guide/neighborhoods/<slug>`, above the venues, orgs and places there (GD3 §3.7) |
| Relocation articles | `guide_pages`, `section = 'new-to-seattle'`, free slugs | `/guide/pages/<slug>`, listed at `/guide/sections/new-to-seattle` |

# The Pool workflow

The same six steps as every Pool bucket (spelled out in [[Bucket GD5 — Nightlife & scenes content]]): claim → refresh the facts → Claude drafts into the vault (`Content\GD7\<slug>.md`) → check → publish in the admin editor and set `last_verified_at` → check it live.

**The check here has two halves:**
- **Facts** (rents, stations, travel times) are checked against current sources on the day.
- **Character** ("quiet," "loud and young," "where the bears live") is checked by someone who lives there or did. Mason's Eastside knowledge is the brainstorm's own example.

# Scope

**In:** the relocation hub (§1) · neighborhood rundowns, prioritized (§2) · four relocation articles (§3) · a quarterly rent refresh (§4).

**Out:**
- the Freeze content and onramps (**GD8**; the hub links them);
- health detail (**GD6**);
- nightlife detail (**GD5**);
- Tacoma and Everett, which are out of v1 scope (brainstorm Q5) and get at most a one-line "worth the trip" mention;
- trans-specific relocation guidance (link-out per Q2);
- housing listings of our own (we link to where leads surface).

# Work items

### 1. The hub

- [ ] 1.1 **"New to Seattle (and gay)"** (`new-to-seattle`) *(claimed: ______)*. One page that answers "I just moved here, now what?":
	- where people live, linking the neighborhood index;
	- getting around (§3.2);
	- your first 90 days (§3.1);
	- making friends despite the Freeze (GD8);
	- if you moved because of your state (§3.3).

	Warm, short and practical.

### 2. Neighborhood rundowns (one page per neighborhood slug)

**What a rundown says**, 200–400 words:
1. **Queer character:** who lives there and what's within reach, honestly and dated;
2. **Housing:** the 1BR range, *always a range, always dated*, with the sources named ("rent.com and Zumper listings, 11/2026");
3. **Getting around:** the nearest Link station or the honest "no rail," and a typical time to Capitol Hill;
4. **The trade-offs**, said plainly ("the drawback is no rail");
5. links to the scene and venue pages nearby.

The page already lists the venues, orgs and places in the neighborhood beneath the rundown (GD3 §3.7), so the rundown is the human part. **Tiers:** the page is `annual`; its rent line is re-checked quarterly (§4).

- **2.1 Launch set: Seattle** *(claimed: ______)*:
	- [ ] `capitol-hill`: the gayborhood: 14+ venues, the Center, clinics; loud, young, spendy
	- [ ] `first-hill`: quiet, medical-adjacent, Hill access by streetcar
	- [ ] `central-district`: gentrified and historic; Denny Blaine and Howell at its edge (link GD10's place pages from the 18+ section, not inline); **Judkins Park station (2 Line, 3/2026)**
	- [ ] `beacon-hill`: diverse, cheaper, a growing queer presence, 1 Line
	- [ ] `columbia-city`: walkable, diverse, South End Pride energy, 1 Line
	- [ ] `white-center`: **the second gayborhood**: Lumber Yard, Boombox, Southgate, a queer-owned cluster and its own Pride; no rail
	- [ ] `ballard` and `fremont`: quiet queer density (Rough & Tumble, Charlie's, climb nights)
	- [ ] `u-district`: student, cheap-ish, Planned Parenthood clinic, 1 Line
- **2.2 Launch set: the Eastside, first-class** *(claimed: ______)*. The angle every incumbent misses: no venues, a real recurring layer, and the 2 Line:
	- [ ] `bellevue`: the hub; downtown high-rises; the Crossroads events; 2 Line ×4 stations
	- [ ] `redmond`: Microsoft-land and Marymoor; Eastside QTPOC Collective (Together Center); VALA's queer figure drawing; 2 Line ×3 stations
	- [ ] `kirkland`: waterfront and restaurants; quiet, meetup-based; no rail (I-405 Stride BRT planned)
	- [ ] `bothell`: UW Bothell and breweries (Beardslee Latin Night); the cheapest on the Eastside; no rail
	- [ ] `issaquah`: a trailhead town, outdoorsy, car-first
	- [ ] `sammamish`: plateau families, thin rental stock, car-only
- **2.3 Second wave** *(claimed: ______)*:
	- [ ] `georgetown`
	- [ ] `west-seattle`
	- [ ] `wallingford`
	- [ ] `shoreline`
	- [ ] `burien` (10th Pride in 2026, its first parade)
	- [ ] `renton`
	- [ ] `downtown`
	- [ ] `queen-anne`
	- [ ] `sodo`
	- [ ] `tukwila`

	A neighborhood with no rundown still has its page, listing what's there. GD3 shows an empty intro quietly.

### 3. Relocation articles

- [ ] 3.1 **"Your first 90 days"** (`first-90-days`) *(claimed: ______)*. A checklist page, written as a plain bulleted list. Nothing is saved per reader. Don't count on markdown checkboxes: Angular's sanitizer may strip the `<input>` they render to, so check the preview.
	- **Health rails:** get tested, start or transfer PrEP, find a doctor (GD6's pages);
	- **People:** pick two scenes (GD5), join **one** recurring thing in your first month (GD8). Summer is the joining season, so say so;
	- **Paperwork:** the WA driver's license deadline after moving (verify the current rule and date it), voter registration, vehicle registration, the ORCA transit card;
	- **Housing leads:** the Seattle Queer Housing Facebook group (link-out), and what to know about application fees and screening (WA's tenant law basics, linked to the state's page and dated).
- [ ] 3.2 **"Getting around: Link, buses, and the traffic truth"** (`getting-around-seattle`) *(claimed: ______)*. **Tier `monthly` near any station opening, otherwise `annual`.**
	- **Link:** the 1 Line's span; the 2 Line's full connection on 3/28/2026; the newer north stations (check Pinehurst/NE 130th's status and date it).
	- **Last trains for club nights:** look up the current late-night service and say plainly when the last train leaves Capitol Hill for Bellevue and Redmond. This is the most useful line on the page for the Eastside.
	- **Traffic honesty:** Revive I-5 through 2027 (Lynnwood→Mercer ~50–65 minutes at peak); 520 tolls; West Seattle's single bridge. "If you're moving here in 2026–27, don't plan a car commute across the Ship Canal."
	- **Bikes, scooters and the hills,** in one honest paragraph.
- [ ] 3.3 **"Moving here because of your state"** (`moving-because-of-your-state`) *(claimed: ______)*. The gay men's version, deeply:
	- what Washington's laws protect, each claim with a dated, linked source (the state's anti-discrimination law; the 2023 shield law for gender-affirming and reproductive care; marriage and parental recognition);
	- the practical first steps (§3.1);
	- where people land (§2);
	- and **the trans-rail link-out**: Ingersoll, Lavender Rights Project, and the 2026 King County Trans Resource & Referral Guide, named and credited.

	*Facts, not legal advice:* every legal claim links its source.
- [ ] 3.4 **"The Eastside, honestly"** (`eastside-honestly`) *(claimed: ______)*. The page no incumbent has written: who lives there, what the recurring layer is (the monthly Eastside LGBTQ+ Happy Hour, the East King County LGBTQ+ All Ages Meetup, Rainbow Elder Play & Connect, Rainbow on the Eastside in August, Eastside QTPOC Collective), how the 2 Line changed nights out, and what's still missing. Links GD8's Eastside onramp.

### 4. The quarterly rent refresh

- [ ] 4.1 **Every quarter**, re-pull 1BR ranges for the launch-set neighborhoods from the same two or three listing aggregators. Update each rundown's rent line with the new range and date, and re-stamp the page. Keep the ranges honest: the brainstorm found a ±$300 spread between sources, so show the range, never a single number. The owner is named on the claim line.

### 5. Gates

- [ ] 5.1 Every rent figure is a range with a month and year and a named source. Every station and travel-time claim is dated.
- [ ] 5.2 Every rundown in §2.1–2.2 has had its character checked by someone who lives there or did (recorded on the claim line).
- [ ] 5.3 The Eastside pages read as first-class: none frames the Eastside as "outside Seattle" or "other" (GD1 §3's rule).
- [ ] 5.4 Every in-body link resolves. Legal claims in 3.3 each link a government or court source.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **How many neighborhoods at launch?** There are 25 neighborhood slugs. Writing all of them well before launch is a lot, and a thin rundown is worse than none. *Rec: launch with the 14 in §2.1–2.2: Seattle's queer-relevant core plus all six Eastside cities, since the Eastside is the differentiator. The second wave follows. The empty pages still list what's there.*
2. **Where rent numbers come from.** Listing aggregators disagree by about ±$300, and rents move. *Rec: two or three named aggregators, shown as a range, dated, refreshed quarterly. Never a single "average rent" number.*
3. **Tacoma.** It's a real second scene but out of v1 scope (brainstorm Q5). *Rec: one line in the hub ("Tacoma has its own scene, worth the trip on the Sounder"), and no pages until Tacoma is its own community.*
4. **The legal facts in "moving because of your state."** These are the claims most likely to change with court cases and new laws, and the most costly to get wrong. *Rec: every one links a government or court source and carries a date. The page is in the `monthly` tier while any relevant law is in flux, and RT2's counsel pass reads this page too.*

# Change Log

- **11/1/2026 (v0.1)** — Initial draft from the outline's GD7 blurb, the brainstorm's Pillar 3 and Appendix F (including the 8/2026 Eastside rows), and GD1 §3/§5. A 14-neighborhood launch set (the Eastside first-class), four relocation articles and a quarterly rent refresh. Four open questions.
