---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1
tags:
---

# Bucket GD9 — "New to…" life-transition guides

**Track:** GD — Guide & content · **Size:** M · **Stream:** the Content Pool. Each lane is a separate claim, so the bucket can be farmed one page at a time.
**Needs:**
- GD3 (done: the `new-to` and `adult` sections, the 18+ gate, and the article page's related-listings block);
- GD4b §6's organization rows and GD4a §4.8's series skeleton, so the groups named here have profiles to link. A lane can publish with plain outbound links first and gain profile links when the rows land;
- GD10, for the two † lanes. They live in its section and pass its gates, including the counsel pass (GD10 OQ2).

**Feeds:**
- GD8's onramp pages, which link the matching lane under "Related", and the "start where you are" hub GD8 names (§1 here);
- GD7's first-90-days page, through lane 2.2;
- GD5's `sober` and `elders` scene pages;
- CM4 later: "newly single" and "out later in life" are natural circles (OQ5).

# Context

Implements [[Initial Project Implementation Outline]] bucket GD9. Mason reframed it on 8/5 (outline OQ6) from "coming-out resources" into a family of guides that meet gay men at every fresh start, the way "New to Seattle" meets people who are relocating. The outline's lanes and rules:

- **Ten lanes:**
	- newly out older;
	- new to a larger gay community;
	- newly single after a long-term relationship;
	- new to non-monogamy;
	- new to being a sober gay man;
	- newly non-religious;
	- new to sexual exploration†;
	- new to kink†;
	- new to gay parenting;
	- newly out young‡.
- **†** lives in, and is linked from, the 18+ section.
- **‡** is curated pointers only (Lambert House, the Trevor Project, PFLAG). An adult site with an 18+ layer routes minors to youth specialists rather than owning that content (GD1 §6.7).
- **All gay-men-voiced, link-out-heavy, no therapy-adjacent advice.**
- **Shares a "start where you are" surface** with Pillar 4's onramps (GD8).
- **A research pass at draft time** for the parenting, post-religious and newly-single lanes. It ran on 11/1/2026; see **# Findings**. The brainstorm's appendices already cover the sober, kink and community lanes.

**Why this layer earns its place:** the brainstorm's first market gap is "nobody tracks closures," and the research found the same pattern in these lanes:
- Seattle Counseling Service closed in 2022;
- SAGE's national elder hotline ended in 2023;
- Northwest Adoption Exchange renamed itself "Alongside" on 10/11/2026;
- Families Like Ours looks defunct but is still linked from UW's LGBTQ resources page;
- the old Muslim Alliance domain now hosts gambling spam.

A lane that says "you'll still see these listed; here's what's true now" beats every resource list a reader will find.

# Where this content lives

| Content | Stored in | Renders at |
|---|---|---|
| The hub, "Start where you are" | `guide_pages`, `section = 'new-to'`, slug `start-where-you-are` | `/guide/pages/start-where-you-are`, and first in `/guide/sections/new-to` |
| The eight mainstream lanes | `guide_pages`, `section = 'new-to'`, slug `new-to-<lane>` | `/guide/pages/new-to-<lane>` |
| The young lane ‡ | `guide_pages`, `section = 'new-to'`, slug `new-to-out-young`, **no scene tags** | `/guide/pages/new-to-out-young` |
| The two † lanes | `guide_pages`, `section = 'adult'`, **`adult = true`**, slug `new-to-<lane>` | behind the 18+ gate; listed at `/guide/sections/adult` |

A page's scene tags drive GD3's related-listings block (the venues, organizations and places that share a tag). So each lane's tags in §2's table are chosen on purpose, and the young lane has none.

# The Pool workflow

The same six steps as every Pool bucket (spelled out in [[Bucket GD5 — Nightlife & scenes content]]): claim → refresh the facts → Claude drafts into the vault (`Content\GD9\<slug>.md`) → check → publish in the admin editor and set `last_verified_at` → check it live.

**The check here:**
- every group named is meeting **now**, at the stated time and place, and every price is current;
- every line in a facts box is re-read against its source on the drafting day and carries "checked <date>";
- a reader who has lived the transition reads the draft (OQ1);
- nothing on the page is advice a counselor, lawyer or doctor should give. Those needs get a directory, a phone line or an organization.

# Scope

**In:** the hub (§1) · ten lane pages (§2) · the organization rows these lanes need, handed to GD4b (§3) · gates (§4).

**Out:**
- entering the rows themselves (**GD4b**);
- support circles (**CM4**, gated);
- anything for minors beyond the pointer page (GD1 §6.7);
- counseling, legal or medical advice of any kind;
- code: none is needed. The hub is an ordinary guide page (OQ7).

# What a lane page says

300–700 words, in this order:

1. **Who this is for**, in one line.
2. **What actually changes**: three to five plain points from gay men who've been through it. Practical, not psychological.
3. **Where to start**: two to four groups, each with what they do, when, where and the cost, linking the org profile. Ordered by evidence that they're active.
4. **The facts box**, where the lane involves law or process. Each line cites its source (an RCW section or an agency page) and says "checked <date>". Under the box, one line: "This is what the law says, not advice about your situation," then where free legal help is (OQ4).
5. **"You'll still see these listed"**: the closures and renames a reader will actually run into, each dated (OQ6).
6. **When you need more than a guide**: directories, phone lines and organizations only.
7. **Related**: the matching onramp, scene and other lanes.

**Voice:**
- gay-men-voiced (GD1 Q2);
- no moralizing in either direction, especially in the non-monogamy and sex lanes;
- labels as the groups themselves use them;
- prices and meeting times only when checked that week.

# Work items

### 1. The hub: "Start where you are" (`start-where-you-are`)

- [ ] 1.1 **The hub page** *(claimed: ______)*, tier `annual`:
	- one line on the idea: everyone here started somewhere;
	- a card for each mainstream lane (who it's for, "Read →");
	- the young-lane card **last**, worded for a young reader and pointing to the pointer page;
	- one plain line: "Two more lanes, sexual exploration and kink, are in the 18+ section →", linking `/guide/sections/adult`. There are no adult cards on this page (OQ3);
	- "Just want to meet people? Pick an onramp →" (GD8's chooser) and "New to Seattle? →" (GD7).
- [ ] 1.2 **Links in:** once the hub is published, the guide home's "New to…" tile points at it. That's a one-line edit in GD3's files (Stream B). GD8's onramp pages link the matching lanes from their "Related" line.

### 2. The ten lanes

| § | Lane | Slug | Section | Scene tags | Tier | Research |
|---|---|---|---|---|---|---|
| 2.1 | Newly out, later in life | `new-to-out-later` | `new-to` | `elders` | `annual` | partial (lanes 1 and 3 of the 11/1 pass) |
| 2.2 | New to a big gay community | `new-to-gay-community` | `new-to` | none (a routing page) | `annual` | appendix; routes to GD5/GD7/GD8 |
| 2.3 | Newly single after a long relationship | `new-to-single` | `new-to` | `elders` | `annual`; facts box re-checked each January | **done 11/1** |
| 2.4 | New to non-monogamy | `new-to-non-monogamy` | `new-to` | none | `annual` | **not yet**, at claim time |
| 2.5 | New to being a sober gay man | `new-to-sober` | `new-to` | `sober` | `annual` | appendix (7/31); confirm at claim |
| 2.6 | Newly non-religious | `new-to-non-religious` | `new-to` | none | `annual` | **done 11/1** |
| 2.7 | New to gay parenting | `new-to-gay-parenting` | `new-to` | none | **`quarterly`** until DCYF publishes its revised foster rule, then `annual` | **done 11/1** |
| 2.8 | New to sexual exploration † | `new-to-sexual-exploration` | `adult`, `adult = true` | per GD10 | `annual` | GD6 + GD10 |
| 2.9 | New to kink † | `new-to-kink` | `adult`, `adult = true` | `leather` | `annual` | appendix + GD10 §4 |
| 2.10 | Newly out, young ‡ | `new-to-out-young` | `new-to` | **none, ever** | `semiannual` | pointers only |

- [ ] 2.1 **Newly out, later in life** (`new-to-out-later`) *(claimed: ______)*. For men coming out at 35, 50 or 70, often after a straight marriage.
	- **What changes:** telling a wife and kids; the scene skews young, so where the older men actually are; groups over apps for a first footing; and, if a marriage is ending, lane 2.3's legal facts.
	- **Where to start:**
		- **Gay Fathers Association of Seattle (GFAS)**: peer support since 1975 for gay, bi and questioning men who are or were married or partnered. Thursdays 7:30–9 pm at the Seattle LGBTQ+ Center, plus Zoom. *Confirm it's still meeting; its site's newest update is 7/4/2025.*
		- **GenPride Senior Men's Support Group**: 2nd and 4th Tuesdays, 2–3:30 pm, for men 50 and up. Free.
		- **Mature Friends**: a social club for 40+, \$40 a year, with a full November 2026 calendar.
		- **Seattle Men's Chorus**: audition-based and famously multigenerational.
	- **Family and work:**
		- PFLAG Seattle for the people you're telling (five meetings a month per the 7/31 appendix; confirm);
		- Washington's Law Against Discrimination covers sexual orientation at work (RCW 49.60; check the cite at drafting).
	- **You'll still see:** SAGE's national LGBTQ+ elder hotline, which ended in 2023 (SAGE now points to 988).
	- **Not listed until confirmed:** Seattle Prime Timers (its site's certificate has expired and no 2025–26 activity turned up).
	- **Related:** `onramp-40-plus`, the `elders` scene, lane 2.3.
- [ ] 2.2 **New to a big gay community** (`new-to-gay-community`) *(claimed: ______)*. For men from somewhere small, or around this many gay men for the first time. Short and mostly routing.
	- **What changes:** there's suddenly plenty, and the Freeze anyway; bars aren't the only door; the scenes (bears, leather, jocks, geeks) are invitations, not boxes; the apps; and getting on the sexual-health rails (PrEP, testing) early.
	- **Where to start:** GD8's chooser and onramps, GD7's first 90 days, GD5's scene intros, GD6's health page, and the Center's Resource & Referral line.
	- **Related:** `beat-the-freeze`, `onramp-new-in-town`.
- [ ] 2.3 **Newly single after a long relationship** (`new-to-single`) *(claimed: ______)*. **Three doors: divorce, breakup and widowed.** Built from Findings lane 3.
	- **Divorce:** WashingtonLawHelp's free interactive forms → the King County family law facilitators (walk-in 9 am–noon weekdays, Seattle and Kent) → a free 30-minute QLaw consult. **KCBA's family-law clinics were closed to new applicants as of their 10/27/2026 update**; re-check on the drafting day.
	- **Facts box** (each line re-read at drafting):
		- no-fault, with a 90-day minimum after filing and service, for marriages and registered domestic partnerships alike (RCW 26.09.030);
		- same-sex domestic partnerships not ended or converted by 6/30/2014 became marriages, unless a partner was 62 or older. The original registration date counts as the date of the marriage, which matters for dividing property after a long relationship (RCW 26.60.100);
		- couples who never married: the "committed intimate relationship" doctrine, linking WashingtonLawHelp's page.
	- **Grief, LGBTQ+-specific first:**
		- **GenPride Grief & Loss**: in person on the 2nd and 4th Tuesdays, 5–6:30 pm; online on the 1st and 3rd Thursdays, 3:30–5 pm, since 9/17/2026;
		- **LGBT Widows and Widowers**: an online forum, monthly on Sundays at 11 am Pacific;
		- then **The Healing Center**: free Zoom groups for losing a spouse or partner, including one for 60+. It has no LGBTQ+ statement, so say so neutrally;
		- and **EvergreenHealth**'s bereavement groups in Kirkland (unconfirmed: the site blocks automated reads).
	- **Getting back out:** Mature Friends, GenPride, Pride Across the Bridge's Rainbow Elder Breakfast on the Eastside, Frontrunners, and the choruses.
	- **Say the gap plainly:** there's no LGBTQ+ divorce or breakup group in King County that we could find (OQ5). The nearest are GFAS (men leaving a marriage) and the grief groups.
	- **You'll still see:** Seattle Counseling Service (closed 5/6/2022) and the SAGE elder hotline (ended 2023).
	- **Related:** lane 2.1, `onramp-40-plus`.
- [ ] 2.4 **New to non-monogamy** (`new-to-non-monogamy`) *(claimed: ______)*. Mainstream guide content, handled matter-of-factly: it's common among gay men, nobody here will judge it, and nobody here will push it either.
	- **Not yet researched:** the 11/1 pass didn't cover this lane, so the claimer runs a short research pass first.
	- **What it covers:**
		- the words in plain terms (open, monogamish, poly);
		- that agreements and talking come first, through link-outs to well-regarded resources picked at drafting rather than our own how-to;
		- the health rails (GD6: PrEP, DoxyPEP, a testing cadence);
		- any local ENM or poly community groups, saying who they serve.
	- **Placement:** anything about sex venues links only to the gated 18+ pages.
- [ ] 2.5 **New to being a sober gay man** (`new-to-sober`) *(claimed: ______)*. For the newly sober and the sober-curious in a scene built around bars.
	- **What changes:** the default social plan is a drink; dating sober; Pride and party season; and, plainly, where to find help if crystal is part of the story.
	- **Where to start:**
		- **Gay Men in Recovery** and **Gay & Lesbian Beginners** AA, named, with seattleaa.org's finder and the Meeting Guide app for current times (GD8 OQ4);
		- **Qu-Art**'s sober artists' night (last Friday at PUSH/PULL per the 7/31 appendix; confirm);
		- the sober-friendly socials and venues with real NA drinks from GD5's `sober` scene.
	- **To research at claim:** Crystal Meth Anonymous meetings in Seattle, and non-12-step options (SMART Recovery, Recovery Dharma) with any LGBTQ+ meetings.
	- **When you need more:** Washington's recovery help line (check the name and number at drafting) and 988.
	- **How it differs from `onramp-sober`:** the onramp is groups to join, and this lane is what changes and where help is. Each links the other.
- [ ] 2.6 **Newly non-religious** (`new-to-non-religious`) *(claimed: ______)*. **Two exits, side by side:** leaving religion, or leaving a church but not the faith. Built from Findings lane 2.
	- **Leaving religion:**
		- **Recovering from Religion**: a free peer line and chat, plus a free LGBTQ+-only online group on the third Sunday (next on 11/15/2026; its page lists two different times, so confirm);
		- **Sunday Assembly Seattle**: 2nd Sundays at 11 am in Seattle Public Library branches;
		- **Seattle Atheists**: weekly Green Lake walks, with Cedar River Secular Society for south King County;
		- **Dharma Buddies**: a queer men's Buddhist sangha, on Zoom Sundays at 11 and sitting in person on the 1st and 3rd Saturdays.
	- **Keeping the faith, leaving the church:**
		- **Emerald City MCC** (Sundays 11:30 am, U District);
		- **Congregation Tikvah Chadashah** (Capitol Hill, since 1980);
		- **Affirmation** for LGBTQ+ Mormons and **Q Christian Fellowship** (national);
		- the finders: Reconciling Ministries (Methodist), ReconcilingWorks (Lutheran), UCC Open and Affirming, More Light (Presbyterian) and Church Clarity.
	- **Facts box:** the United Methodist Church lifted its bans on LGBTQ+ clergy and same-sex weddings on 5/1/2024. About a quarter of US congregations had already left, mostly conservative ones, so a "Methodist" sign may not mean UMC. Use the finder.
	- **Religious trauma, directories only:**
		- Secular Therapy Project;
		- Reclamation Collective's directory. Its queer group is \$200–400 a season on a sliding scale, while RfR's groups are free;
		- Journey Free.
	- **You'll still see:** Humanists of Washington (domain gone; last event 10/2021) and the old Muslim Alliance domain, which now hosts spam. Name it with **no link**; the group lives at masgd.org.
	- **Not listed until confirmed:** Dignity/Seattle (newest content 2024) and Evangelicals Concerned. Also re-check the appendix's Queer Dharma (Shambhala, 1st and 3rd Mondays), which the 11/1 pass didn't look at.
	- **Family:** PFLAG.
- [ ] 2.7 **New to gay parenting** (`new-to-gay-parenting`) *(claimed: ______)*. Built from Findings lane 1.
	- **Four routes, each linking its source:**
		1. **Foster or foster-adopt** through DCYF or a private agency. DCYF charges no licensing fee. Applicants must be 21 or older, and the home study takes about 120 days. Zoom info sessions run on the first Tuesday of each month, and DCYF's FAQ names LGBTQ+ applicants as welcome. Agencies: Amara (Seattle and Tacoma) and the YMCA of Greater Seattle.
		2. **Adoption from foster care:** Alongside, formerly Northwest Adoption Exchange.
		3. **Private or infant adoption.** The research didn't find a Washington agency with a current LGBTQ+ statement, so research it at claim.
		4. **Surrogacy:** Men Having Babies for costs and its assistance program (GPAP).
	- **Facts box: surrogacy in Washington**, "law in effect since 1/1/2019, RCW checked <date>":
		- paid surrogacy is legal (26.26A.715);
		- everyone must be 21 or older, each side has its own lawyer, the intended parents pay for the surrogate's, and the agreement is signed before any medical procedure (26.26A.705/.710);
		- both dads can be named on the birth record through a court order sought before or after the birth (26.26A.740/.750);
		- a donor is not a parent (26.26A.605);
		- a court may recognize more than two parents in limited cases (26.26A.460(3)).
		- **Parental leave:** Washington's Paid Family and Medical Leave covers bonding for any parent, including non-birth, adoptive and foster parents, within 12 months. Confirm the "up to 12 weeks" figure; it was seen only in a 2021 guide.
	- **Dated news note, the 2026 foster settlement:**
		- since DCYF's 5/22/2026 statement, a foster license can't be denied over religious beliefs alone. Per KUOW and OPB (10/15–17/2026), foster parents no longer have to provide an LGBTQ+-affirming home;
		- DCYF still controls placements, and nothing changed for gay applicants;
		- **re-check the revised rule (WAC) before publishing**, which is why this lane stays on the `quarterly` tier.
	- **Dads' community, ranked by evidence:**
		- PEPS's LGBTQIA+ Parents group (2026 events confirmed);
		- Seattle Gay Dads (playground meetups; only a 2026 footer, so confirm);
		- GFAS (skews toward men who came out after a marriage);
		- Family Equality's virtual groups as the national backup.
	- **Legal help:** QLaw's free clinic. Its site blocks automated reads, so confirm the details by hand.
	- **You'll still see:**
		- Northwest Adoption Exchange (renamed on 10/11/2026);
		- Families Like Ours, Rainbow Families of Puget Sound and Proud & Loud Families NW. All look defunct but are still listed elsewhere, so per OQ6 only the one a reader will actually hit gets named, and the rest are left off.
- [ ] 2.8 **† New to sexual exploration** (`new-to-sexual-exploration`, section `adult`, `adult = true`) *(claimed: ______)*. For the newly out and the newly single figuring out sex in a big city.
	- **Health rails first:** GD6's PrEP, DoxyPEP and testing (the Center's free testing six days a week; Harborview).
	- **Consent and boundaries:** link-outs.
	- **The apps and their etiquette:** said plainly.
	- **Where things happen:** link GD10's pages (the bathhouses, Denny Blaine's current rules, and the etiquette, safety and law page).
	- No photos (GD10 OQ3). Written here; GD10 §4.4 checks it, and its counsel pass covers it.
- [ ] 2.9 **† New to kink** (`new-to-kink`, section `adult`, `adult = true`) *(claimed: ______)*.
	- **Education before parties:** munches and classes first, with Pan Eros Foundation's education.
	- **Then where the scene is:** the Cuff, and GD10's `leather-kink-seattle` page. Safeword at Kremwerk (1st Saturday per the 7/31 appendix) is a party, not a first step, so say so.
	- **Negotiation, safewords and safety:** link-outs.
	- **To research at claim:** current Seattle leather clubs and munch listings.
	- Nothing unconfirmed is stated as fact. GD10 §4.4's gate and its counsel pass apply.
- [ ] 2.10 **‡ Newly out, young** (`new-to-out-young`) *(claimed: ______)*. **Pointers only**, per OQ2.
	- **One short paragraph:** this site is for adults, and these people are the experts for you.
	- **Then four cards:**
		- **Lambert House** (Capitol Hill's center for LGBTQ+ youth; check ages and hours at drafting);
		- **The Trevor Project** (24/7 phone, text and chat);
		- **PFLAG Seattle** (for parents and families);
		- **988**. Note that 988's LGBTQ+ youth option ended in July 2025 while Trevor's own line continues; confirm at drafting.
	- **Nothing else:**
		- no scene tags, so the related-listings block stays empty;
		- no events;
		- no links into the 18+ section or to any `adult = true` page;
		- no dating, app or bar content.

### 3. Rows for GD4b (from the 11/1 research)

Groups these lanes name that GD4b §6 doesn't carry yet. Agencies, national lines and online-only groups stay as links in page prose rather than rows. Each is confirmed active on entry, as usual.

- [ ] 3.1 `community-support`:
	- Gay Fathers Association of Seattle (GFAS; confirm 2026 meetings);
	- Seattle Gay Dads (confirm);
	- PEPS, as its LGBTQIA+ Parents group.
- [ ] 3.2 `faith`:
	- Emerald City Metropolitan Community Church;
	- Congregation Tikvah Chadashah;
	- Dharma Buddies.
- [ ] 3.3 `social-meetup`:
	- Sunday Assembly Seattle;
	- Seattle Atheists / Evergreen Secular Coalition (with Cedar River Secular Society noted);
	- Mature Friends (`elders`).
- [ ] 3.4 **Series under existing orgs** (GD4a §4.8's skeleton): GenPride Grief & Loss (in person and online) and GenPride Senior Men's group.
- [ ] 3.5 **Pride Across the Bridge** is in GD4 §6.2 only as figure drawing. Its Rainbow Elder Breakfast, book club and office hours suggest PAB should be the organization, with figure drawing as one of its series. That's GD4's call.

### 4. Gates

- [ ] 4.1 Every group named on every page has been confirmed as meeting now, with the stated cadence and cost (recorded on the claim line).
- [ ] 4.2 Every facts-box line carries its source and checked date, and was re-read on the drafting day.
- [ ] 4.3 Every lane has had its lived-experience reader (OQ1).
- [ ] 4.4 **The young page:** no scene tags, no events, no link to the `adult` section or any `adult = true` page, and its four pointers checked live. **The hub:** no adult cards.
- [ ] 4.5 The † lanes are `section = 'adult'` with `adult = true`, and passed GD10's §6 gates and counsel pass.
- [ ] 4.6 No therapy-adjacent advice: every "when you need more" item is a directory, a phone line or an organization (reviewer confirms).
- [ ] 4.7 Every "you'll still see" callout carries a date and a source.
- [ ] 4.8 Every in-body link resolves. The hub's lane links resolve to published pages before the guide home links the hub.

# Findings (research pass, 11/1/2026)

**Method:** one research agent, run on 11/1/2026 over the three lanes the outline flagged (parenting, post-religious, newly single).
- **Verified** means the page was loaded that day and showed what's described.
- **Unverified** means the information came only from search snippets or third-party listings, or the site blocked automated reads.
- **Evidence** is the newest dated item actually seen. A bare copyright footer is weak evidence and is marked as such.

## Lane 1: gay parenting

| Name | What | Scope · cost | Evidence (2026) | Status |
|---|---|---|---|---|
| WA DCYF Foster Parenting (dcyf.wa.gov/services/foster-parenting) | State licensing: inquiry form or 888-KIDS-414; Zoom info sessions on the first Tuesday monthly, 12–1 and 6–7 pm | State · no DCYF fee (private agencies may charge); monthly payment once a child is placed | Caregiver News 3/5/2026 | Verified |
| DCYF foster FAQ (dcyf.wa.gov/node/4053) | Names "people who are LGBTQ+" as welcome; single, partnered or married; renters OK; 21+; home study about 120 days | State · free | Parent site has 2026 items | Verified |
| Amara (amarafamily.org) | Child-placing agency: foster licensing, kinship, adoption, free caregiver groups; Seattle and Tacoma offices, Pierce-focused; 2023 HRC "Innovator" seal | Local/regional · no charge | News 10/12, 9/2 and 8/26/2026 | Verified |
| YMCA of Greater Seattle foster care | Long-term, short-term, therapeutic and respite care; monthly caregiver group; lists sexual orientation among who can foster; mostly western King | Local · not stated | 2026 footer only (weak) | Verified page |
| Alongside, formerly NW Adoption Exchange (wa.alongsidekids.org) | Adoption from foster care: info sessions, agency choice, profiles (youth profiles need sign-in from 10/27); HRC "All Children All Families" logo | State · free | Renamed 10/11/2026; nwae.org redirects here | Verified |
| PEPS (peps.org) | Peer parent groups, including an **LGBTQIA+ Parents** group, in person or online | Local/regional · flexible pricing | Events through 12/5/2026 | Verified |
| Seattle Gay Dads (seattlegaydads.weebly.com) | Gay, bi, pan and trans fathers and intending fathers; monthly playground meetup (2nd weekend), quarterly dads' nights; Facebook (~300), WhatsApp, email, calendar feed | Local · free | 2026 footer only (weak) | Verified page; activity unconfirmed |
| Gay Fathers Association of Seattle (gfas.org) | Peer support since 1975 for gay, bi and questioning men who are or were married or partnered; Thursdays 7:30–9 pm at the Center (400 E Pine) plus Zoom; monthly potluck | Local · free; optional dues from \$25/yr | Newest page update 7/4/2025 | Verified page; 2026 unconfirmed |
| Family Equality (familyequality.org) | National: guide to spotting affirming agencies, virtual peer groups, provider database, foster law by state | National · free | Guide dated 5/21/2026 | Verified |
| Men Having Babies (menhavingbabies.org) | Surrogacy nonprofit for gay men: conferences, webinars, provider ratings, GPAP (grants and discounts from 70+ providers) | National · fees not stated | Brussels conference 11/13–15/2026; no PNW events | Verified |
| QLaw Foundation LGBTQ2S+ Legal Clinic (with KCBA) | Free 30-minute phone consults, including family law and divorce; (206) 483-2725 x102 | Local · free | WA 211 listing current | QLaw site unverified (blocks reads); 211 listing verified |
| WA Paid Family & Medical Leave (paidleave.wa.gov) | Bonding leave for "moms, dads, non-birth parents and guardians" after birth, adoption or foster placement, within 12 months | State benefit | 2027 rate banner | Verified; "up to 12 weeks" unverified |

## Lane 2: newly non-religious

| Name | What | Scope · cost | Evidence (2026) | Status |
|---|---|---|---|---|
| Recovering from Religion (recoveringfromreligion.org) | Peer line 844-368-2848, web chat, 50+ peer groups, library | National · free | 2025 footer | Verified |
| RfR LGBTQ+ Virtual Support Group (meeting #1002) | Third Sunday monthly, online; LGBTQ+ only, 16+; no need to be an atheist; latecomers may be shut out | National · free | Next 11/15/2026; page lists both 8–10 pm and 2–4 pm Chicago time, so confirm | Verified |
| Secular Therapy Project (seculartherapy.org) | Directory of secular, evidence-based clinicians by state | National | 2026 footer (weak) | Verified |
| Journey Free (journeyfree.org) | Private online group, coaching, Religious Trauma Syndrome resources | National · prices not listed | No 2026 content | Verified page; activity unconfirmed |
| Reclamation Collective, Queers Reclaiming Together | 10-session virtual group for LGBTQ+ people healing from religious trauma (LICSW-led); also a practitioner directory | National · \$200–400 sliding scale, scholarships | Winter 2026 season 1/15–4/2 | Verified |
| Sunday Assembly Seattle (meetup.com/sa_seattle) | Secular "congregation": assembly 2nd Sundays 11 am at SPL branches, conversation circles, walks; 1,838 members | Local · free | 11/8/2026 at Douglass-Truth; next 12/13 | Verified |
| Seattle Atheists / Evergreen Secular Coalition (seattleatheists.org) | Weekly Green Lake walks, drinks, art walks; includes Cedar River Secular Society (south King) and Seattle Skeptics; 2,585 members | Local · dues, waivable | Events 11/2, 11/5, 11/8 and 11/15/2026 | Verified |
| Dharma Buddies (meetup.com/Dharma-Buddies) | Queer men's and nonbinary Buddhist sangha; Zoom Sundays 11 am; in person 1st and 3rd Saturdays; 475 members | Local · not stated | Events 11/8, 11/15 and 11/22/2026 | Verified |
| Reconciling Ministries Network (rmnetwork.org) | Finder for affirming United Methodist congregations | Finder · free | 2026 toolkit | Verified |
| ReconcilingWorks (reconcilingworks.org) | Finder for affirming Lutheran congregations; Seattle, Bellevue and Kenmore congregations appear in search | Finder · free | News 10/26–28/2026 | Verified site; congregations unverified |
| UCC Open and Affirming (openandaffirming.org) | Finder for affirming UCC congregations | Finder · free | Posts 7/2–8/10/2026 | Verified |
| More Light Presbyterians (mlp.org) | Map of affirming Presbyterian congregations | Finder · free | 2026 footer (weak) | Verified page |
| Church Clarity (churchclarity.org) | Crowd-sourced scores for how clearly churches state LGBTQ+ policy (5,554 published) | National · free | 2/17/2026 (became a 501(c)(3)) | Verified |
| Q Christian Fellowship (qchristian.org) | LGBTQ+ Christian community: quarterly virtual groups, annual conference | National | Conference 1/21–24/2027 | Verified |
| Emerald City MCC (mccseattle.org) | LGBTQ-founded Protestant church; Sundays 11:30 am at The Wesley Club, 1414 NE 42nd St Ste 202; livestream | Local · free | 2026 footer (weak) | Verified |
| Congregation Tikvah Chadashah (tikvahchadashah.org) | LGBTQ-welcoming chavurah-style synagogue on Capitol Hill since 1980; Shabbat services and potlucks | Local | Shabbat Fri 11/6, Hanukkah Fri 12/11 (weekdays match 2026) | Verified |
| Affirmation (affirmation.org) | LGBTQ+ Mormons: peer community, summits, conference | National | "Affirmation Together 2026," 10/16–12/5 | Verified |

**Dated fact:** the UMC General Conference voted 692–51 on 5/1/2024 to repeal its bans on LGBTQ clergy and on officiating same-sex weddings. About a quarter of US congregations, mostly conservative ones, had already left (NPR via OPB, 5/1/2024).

## Lane 3: newly single

| Name | What | Scope · cost | Evidence (2026) | Status |
|---|---|---|---|---|
| GenPride Grief & Loss (genprideseattle.org/programs) | Peer-led, for LGBTQ+ elders: in person 2nd and 4th Tuesdays 5–6:30 pm; online 1st and 3rd Thursdays 3:30–5 pm | Local · free | Online group started 9/17/2026 | Verified |
| GenPride Senior Men's Support Group | 2nd and 4th Tuesdays 2–3:30 pm, men 50+; GenPride also runs Hot Lunch, game club and an elders meet & greet | Local · free (Hot Lunch \$5 suggested for 60+) | Same page | Verified |
| The Healing Center (healingcenterseattle.org) | Free Zoom groups for losing a spouse or partner (Foundations, Memory Keepers 60+, Survivors of Suicide); join any time; (206) 523-1206 | Local · free | Newsletters 2/1–4/1/2026 | Verified; no LGBTQ+ statement |
| EvergreenHealth Grief & Bereavement (Kirkland) | Hospice bereavement groups, including spousal loss, for deaths in the last 13 months; (425) 899-1077 | Eastside · free per WA 211 | Search shows Nov–Dec 2026 groups | Unverified (blocks reads) |
| LGBT Widows and Widowers Support Group (Meetup) | Online peer forum, monthly on Sundays, 11 am Pacific; based in Illinois; 269 members | National · not stated | "Coping with Loneliness" 11/8/2026 | Verified |
| WashingtonLawHelp.org | Divorce guides, free interactive filing and response forms, separation, ending an unmarried relationship, domestic partnerships | State · free | 2026 footer (weak) | Verified |
| WA Courts family law forms | Official divorce, separation and parenting-plan forms | State · forms free, filing fees apply | — | Verified |
| King County family law facilitators | Procedural help without a lawyer (no legal advice); walk-in 9 am–noon weekdays, Seattle W-280 and Kent 1D | Local · WA 211 says \$30/visit, waivable; county page silent | 10/12/2026 closure notice | Verified; fee unverified |
| KCBA Neighborhood Legal Clinics | Free 30–60 minute volunteer-attorney consults; 206-267-7070 | Local · free | **Family-law clinics closed to new applicants as of 10/27/2026** | Verified |
| Mature Friends (mature-friends.org) | LGBTQ+ social club for 40+ (250+ members): dinners, walks, potlucks, bridge, bowling, book club, travel | Local · \$40/yr | Full November 2026 calendar; banquet 12/6/2026 | Verified |
| Pride Across the Bridge (prideacrossthebridge.org) | Eastside nonprofit: Rainbow Elder Breakfast, office hours at Together Center (Redmond), book club, figure drawing, D&D | Eastside · not stated | Events 11/4–11/18/2026 | Verified |
| Seattle Men's Chorus | Audition-based; weekly rehearsals; a large social network | Local · dues not stated | Spring 2027 shows listed | Verified site; audition dates unverified |
| Seattle Frontrunners | LGBTQ+ running and walking club; membership on Heylo | Local · \$30/yr (\$10 limited income) | No dated item | Verified page; activity unconfirmed |
| Seattle LGBTQ+ Center (formerly Gay City) | 400 E Pine: Resource & Referral line, mutual-aid closet, clinic; hosts GFAS | Local · free | City profile 6/1/2026 | City page verified; gaycity.org blocks reads |

## Legal facts (statutes read 11/1/2026 at app.leg.wa.gov)

- **Uniform Parentage Act**, ch. 26.26A RCW, in effect since **1/1/2019** (.903).
	- Paid surrogacy is legal (.715).
	- The surrogate must be 21 or older, have given birth before, have had no more than two prior surrogacy births, and have a medical evaluation, a mental-health consultation and her own lawyer. Each intended parent needs the same, minus the prior birth (.705).
	- At least one party is a Washington resident or a procedure happens here. The agreement is signed and notarized or witnessed, names each side's lawyer, has the intended parents paying for the surrogate's, and is signed before any medical procedure (.710).
	- Each intended parent is a parent at birth by operation of law (.740). A judgment can be sought before, at or after birth, and the registrar lists each intended parent on the birth record (.750).
	- Practitioners say genetic surrogacy needs a post-birth order; not confirmed in the statute.
	- A donor is not a parent (.605). A court may find more than two parents where not doing so would harm the child (.460(3)).
- **Divorce:** no-fault ("irretrievably broken"), with a 90-day minimum after filing and service; applies to registered domestic partnerships too (RCW 26.09.030).
- **Domestic partnerships:** same-sex partnerships not dissolved or converted by **6/30/2014** became marriages, except where either partner was 62 or older. The original registration date is the legal date of the marriage (RCW 26.60.100; Referendum 74, 11/6/2012).
- **Unmarried couples:** the committed-intimate-relationship doctrine (WashingtonLawHelp's page; no date).
- **Foster rules, 2026:**
	- DCYF's statement of 5/22/2026: no license denial solely over religious beliefs about marriage, gender or sexuality. DCYF keeps "full and sole authority" over placements and can decline a placement when a home can't meet a child's needs.
	- KUOW/OPB (10/15–17/2026): the settlement removed the requirement to provide an LGBTQ+-affirming home. An informational intake question starts in 2027.
	- Neither changes eligibility for LGBTQ+ applicants. **The revised WAC text wasn't checked.**

## Closed, renamed or stale (as of 11/1/2026)

| Item | What was seen |
|---|---|
| Seattle Counseling Service | **Closed 5/6/2022** (stopped intake 3/1/2022). Still in some directories (Archives West; Seattle Gay Scene 3/3/2022) |
| SAGE National LGBTQ+ Elder Hotline | "Discontinued in 2023"; SAGE now points to 988 and SAGEYou |
| Northwest Adoption Exchange | **Renamed "Alongside" on 10/11/2026**; nwae.org redirects |
| Families Like Ours | Site times out; GuideStar says it may have ceased. Still linked from UW ISCRM's LGBTQ resources page |
| Rainbow Families of Puget Sound | Domain doesn't resolve; the Center's page lists only a Facebook group and a Yahoo board |
| Proud & Loud Families NW | Meetup group not found |
| Pride Across the Bridge "Pride Parenting" | Last listed 6/11/2023; not on PAB's current program list (PAB itself is active) |
| PEER Seattle "Proud Parents" | Last dated listing 6/13/2025; PEER's families page is from 2019 |
| Humanists of Washington | Domain gone; its south-sound Meetup's last event was 10/2/2021 |
| muslimalliance.org (old MASGD domain) | **Now hosts gambling spam.** MASGD is at masgd.org, with no dated content seen |
| Dignity/Seattle | Newest items are from 2024; no Mass schedule posted |
| Seattle Prime Timers | Site certificate expired; no 2025–26 activity found |
| "Tears for Queers" (Gay City, 2024) | Page gone; no newer session |
| OUT In The Wild; a 50+ queer hiking Meetup | One deleted; the other has no organizer or events |
| Seattle Choruses | Site now splits into seattlemenschorus.org and seattlewomenschorus.org (not a closure) |
| Evangelicals Concerned | Blocked by bot protection; status unknown |

## Gaps

- No active LGBTQ+-specific divorce or breakup support group in King County turned up (OQ5).
- The revised foster-licensing rule (WAC) after the 2026 settlement wasn't checked. Re-check before lane 2.7 ships.
- No Washington private or infant adoption agency with a current LGBTQ+ statement was confirmed. WA 211 lists a Seattle office for Open Adoption & Family Services, with no statement seen.
- EvergreenHealth, QLaw and gaycity.org block automated reads. Spot-check them in a browser.
- Non-monogamy (2.4), and the sober (2.5) and kink (2.9) lanes' 2026 details, weren't in this pass.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Who reads each lane before it goes live?** Anyone can check the facts, but "what actually changes" is only right if someone who has lived it says so. *Rec: each lane gets one reader with lived experience, named on the claim line. For the private lanes (sober, newly single, non-religious, out later in life), it can be recorded as "a reader who asked not to be named." The young lane is pointers only, so its check is that the four links work and the page links nowhere else. Offering Lambert House a look is a courtesy, not a gate.*
2. **The young lane: how far does "pointers only" go?** *Rec: the page names Lambert House, the Trevor Project, PFLAG (for parents) and 988, and nothing else. No scene tags, so GD3's related-listings block stays empty and no bar ever shows up there. No events, no links into the 18+ section, no dating or app talk. It lives in `new-to` and sits last on the hub. Tier `semiannual`, because national lines change (988's LGBTQ+ youth option ended in 2025).*
3. **The 18+ lanes on the hub.** The hub is mainstream and carries the young-lane card. Putting adult cards beside it, even gated ones, mixes the two. *Rec: no adult cards on the hub. Use one plain line linking the 18+ section, where the two lanes are listed with the rest of GD10's pages. That's the outline's "lives in and links from the 18+ section."*
4. **Stating the law.** The parenting and newly single lanes quote Washington statutes (surrogacy, divorce's 90 days, the 2014 partnership rule). Is that legal advice? *Rec: no. Each line says what the statute says, cites it, and carries its checked date, and the box ends with "not advice about your situation" plus where free help is (QLaw's clinic, WashingtonLawHelp, the court facilitators). These mainstream lanes don't need a counsel pass. If RT2's counsel pass happens anyway, hand it the two boxes.*
5. **The missing group.** The research found no LGBTQ+ divorce or breakup support group in King County. *Rec: say so plainly on the newly single page and point to what's closest: GFAS for men leaving a marriage, and the grief groups for widowers. Record it as a CM4 circle candidate rather than starting a group ourselves now.*
6. **Which dead listings do we name?** *Rec: name the ones a reader will actually run into in 2026, each with its date: Seattle Counseling Service, the SAGE elder hotline, Northwest Adoption Exchange's rename, Families Like Ours (still linked from UW), and the hijacked old MASGD domain (named, never linked). Small groups that just went quiet (the deleted Meetups, Rainbow Families of Puget Sound) are simply left off. Groups whose 2026 activity couldn't be confirmed (GFAS, Seattle Gay Dads, Seattle Prime Timers, Dignity/Seattle) get a call or visit before they're listed. If nobody can confirm one, it stays off.*
7. **A hub page, or a new step in GD8's chooser?** GD8's chooser already routes people who want to join something. The life-transition lanes are a different question ("what just changed for you?"). *Rec: the hub is an ordinary guide page with no new code, and it cross-links the chooser both ways. Like the chooser, nothing is counted (GD8 OQ3).*

# Change Log

- **11/1/2026 (v0.1)** — Initial draft. Sources: the outline's GD9 entry (the 8/5 ten-lane reframe and its placement rules), the brainstorm's Pillar 4 and Appendix D, and a research pass run the same day over the parenting, post-religious and newly-single lanes, kept here as **# Findings**. The draft has:
	- a "Start where you are" hub and ten lane pages, eight mainstream, two in the 18+ section, and a pointer-only page for young readers;
	- a shared page shape (who it's for → what changes → where to start → a dated facts box → "you'll still see these listed" → when you need more → related);
	- rows handed to GD4b;
	- seven open questions.
