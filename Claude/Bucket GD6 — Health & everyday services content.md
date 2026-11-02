---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1
tags:
---

# Bucket GD6 — Health & everyday services content

**Track:** GD — Guide & content · **Size:** S · **Stream:** the Content Pool (claimed: ________)
**Needs:** GD3 (done) · GD4b §8's service-listing rows, for the services directory; the articles can publish without them.
**Feeds:** GD7's first-90-days checklist ("get on PrEP and testing rails") · CM2, whose health-category recommendations render as unattributed aggregates, keyed off `is_health` · RT2's privacy page, which cites this section's no-tracking posture.

# Context

Implements [[Initial Project Implementation Outline]] Content Pool bucket GD6: "The corrected 2026 facts (Lifelong→Bailey-Boushay, SCS closed, Kelley-Ross One-Step, PrEP DAP numbers) + affirming-services editorial. Semi-annual refresh tier."

From [[Brainstorming on the website]] Pillar 2.3–2.4 and Appendix E:
- **Sexual health:**
	- STI testing: Harborview's Sexual Health Clinic, and the Center's free testing six days a week;
	- one-stop PrEP (Kelley-Ross One-Step, at two locations), DoxyPEP, mpox, and HIV care (Madison Clinic);
	- **the two corrections every competitor gets wrong**: Lifelong's housing programs moved to Bailey-Boushay (2024–25), and Seattle Counseling Service closed in 2022.
- **Everyday services:** LGBTQ-affirming primary care, mental health, gender-affirming entry points (Ingersoll's provider directory is the incumbent to link, not fight), dentists, and "the GSBA-gap layer: *editorial* 'who's actually good'." GSBA answers "who paid dues"; we answer "who to hire." CM2's crowd recommendations are the scalable answer here later.

**Rules that matter more here than anywhere:**
- [[Bucket GD1 — Taxonomy & editorial foundations]] §6.8: **health content is directory, not advice.** It says what a service is, who it's for, how to start, and when that was checked. It never offers therapy-adjacent counsel, and it states corrections of record plainly with dates.
- The **MHMDA posture**: no third-party embeds (no map widgets, no booking iframes, no analytics) on these pages, and no user ever attached to a health listing.
- **Gender-affirming care is a link-out lane** (Q2): we point to Ingersoll, Lavender Rights Project and the 2026 King County Trans Resource & Referral Guide, with credit.

# Where this content lives

| Content | Stored in | Renders at |
|---|---|---|
| Health articles | `guide_pages`, `section = 'health'`, tier `semiannual` (GD2 OQ9 (a)) | `/guide/pages/<slug>`, listed at `/guide/sections/health` |
| Everyday-services articles | `guide_pages`, `section = 'services'` | `/guide/pages/<slug>` |
| The directory itself | `service_listings` (GD4 §8) | `/guide/services`, kind-grouped link-out cards (GD3 §3.6; no detail pages at MVP) |

# The Pool workflow

The same six steps as every Pool bucket (spelled out in [[Bucket GD5 — Nightlife & scenes content]]): claim → refresh the facts → Claude drafts into the vault (`Content\GD6\<slug>.md`) → check → publish in the admin editor and set `last_verified_at` → check it live.

**The "check" step here is a fact check, not a vibe check.** Every claim is confirmed against the provider's own current page, or by phone, on the day. The appendix's facts are from July 2026, and health programs change their hours, sites and eligibility often. A clinician friend reading the drafts is welcome but isn't a gate (OQ1).

# Scope

**In:** the health hub and eight health articles (§1) · the services hub and three services articles (§2) · a refresh calendar (§3).

**Out:**
- the listing rows themselves (**GD4 §8**);
- crowd recommendations (**CM2**);
- trans-specific care guides (link-out per Q2);
- anything that reads as medical advice;
- insurance navigation beyond "call and ask" pointers;
- youth health (pointer-only, GD9's young lane).

# Work items

### 1. Health articles (section `health`)

**Every health article ends with the same dated line:** "Facts checked {Month Year}. Hours and eligibility change, so call ahead." The freshness footer carries the stamp as well.

- [ ] 1.1 **"Sexual health in Seattle: where to start"** (`sexual-health-seattle`), the hub *(claimed: ______)*. One screen: get tested, get on PrEP, DoxyPEP, mpox, HIV care, and the corrections. Each points to its article and the directory.
- [ ] 1.2 **"Getting tested"** (`sti-testing-seattle`) *(claimed: ______)*:
	- Harborview Sexual Health Clinic (908 Jefferson; walk-in cutoff; the walk-in mpox vaccine);
	- the Center's free testing (days and hours), and its mailed kits;
	- Planned Parenthood's three Seattle centers;
	- what's free and what isn't;
	- how often people test (stated as the King County or CDC guidance, linked and dated, never our own recommendation).
- [ ] 1.3 **"Starting PrEP"** (`start-prep-seattle`) *(claimed: ______)*:
	- Kelley-Ross One-Step (single-visit start; both locations; TelePrEP statewide);
	- **the WA PrEP Drug Assistance Program** (the 2026 income limit was $6,650/month in July; re-check the current figure and date it);
	- what insurance usually covers, as a pointer to call;
	- injectable options, if local providers offer them (verify).
- [ ] 1.4 **"DoxyPEP"** (`doxypep-seattle`) *(claimed: ______)*: what it is in one sentence, King County's 2025 guidelines (linked and dated), and who offers it locally. Directory, not advice.
- [ ] 1.5 **"Mpox vaccine"** (`mpox-vaccine-seattle`) *(claimed: ______)*: where to get it (walk-in sites), and the current King County notice, linked and dated. **Tier `monthly` during any outbreak notice.**
- [ ] 1.6 **"HIV care"** (`hiv-care-seattle`) *(claimed: ______)*:
	- Madison Clinic at Harborview (the largest in the PNW);
	- **Bailey-Boushay House, which now holds the former Lifelong housing clients** (the 2024–25 handoff), stated as a correction of record with dates;
	- support organizations.
- [ ] 1.7 **"Finding an affirming therapist"** (`affirming-therapy-seattle`) *(claimed: ______)*:
	- **"Seattle Counseling Service closed in 2022"** in the first paragraph (the market's number-one stale-data trap);
	- the successors: Optimism Counseling, Integrative Counseling (Fremont), and the Center's 12-session program;
	- NAMI's referral line;
	- the therapist directories people actually use (Psychology Today's LGBTQ filter, and others, verified).
- [ ] 1.8 **"Finding an LGBTQ-affirming doctor"** (`affirming-primary-care-seattle`) *(claimed: ______)*:
	- the Swedish LGBTQIA+ Program;
	- Capitol Hill Medical;
	- Country Doctor, **now Seattle Roots Community Health** (check which name is live and date it);
	- UW Medicine;
	- the link-out paragraph for gender-affirming care: Ingersoll's directory, Lavender Rights Project, and the 2026 King County Trans Resource & Referral Guide, credited.
- [ ] 1.9 **"Corrections we keep making"** (`health-corrections`) *(claimed: ______)*: a short, dated list of the stale facts still circulating elsewhere (SCS closed 2022; Lifelong's housing to Bailey-Boushay; Country Doctor's rename) and what's true now. It doubles as the trust signal and the page other sites can cite.

### 2. Everyday services (section `services`)

- [ ] 2.1 **"Finding gay-friendly services in King County"** (`gay-friendly-services`), the hub *(claimed: ______)*. Says plainly how to find barbers, realtors, contractors, massage therapists, tattoo artists, photographers and dentists today:
	- GSBA's directory, linked honestly: "a chamber of commerce, so these are members, not reviews";
	- the services we list, and how we choose them;
	- that crowd recommendations are coming (CM2).

	No paid placement, ever (GD1 §8).
- [ ] 2.2 **"Dentists"** (`affirming-dentists`) *(claimed: ______)*. *A research pass is needed:* Appendix E has no dentists. List only providers we can verify as affirming (an explicit statement on their site, or GSBA membership plus a human's experience). An honest short list beats a padded one.
- [ ] 2.3 **"Massage, bodywork and fitness"** (`massage-bodywork`) *(claimed: ______)*. Listed by the same rules as everyone else. **Knotty Yoga appears here only under those rules** (GD1 §6.11 and §8.3): no ranking, no "our pick," and the founded-by relationship stays in the footer and About page.

### 3. The refresh calendar

- [ ] 3.1 **Twice a year** (January and July, matching the `semiannual` tier), the owner re-checks every §1 fact and re-stamps each page that's still true. **PrEP DAP's income limits** move with the federal poverty guidelines each year, so January's pass checks them first.
- [ ] 3.2 **Between passes:** a provider closing, moving or changing its program triggers an immediate fix. The CHS and SGN monitors (GD4 §4.6) and, later, EV5's verifier surface these.

### 4. Gates

- [ ] 4.1 Every fact on every page has been checked against the provider's own current page or by phone, on a recorded date. The claim line says who checked and when.
- [ ] 4.2 The directory-not-advice read: no sentence tells a reader what *they* should do medically. Guidance is always attributed and linked ("King County recommends…").
- [ ] 4.3 No third-party embeds on any health page (check the rendered HTML for iframes and external scripts). Links out are plain links.
- [ ] 4.4 Every in-body link resolves; every directory row in `/guide/services` shows a current stamp.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Should a medical professional read these pages before they go up?** The pages are directories, not advice, so the risk is a wrong fact (hours, eligibility), not a wrong diagnosis. A fact check against each provider's own page catches that. *Rec: welcome a clinician friend's read if one's available, but don't make it a gate. The fact check and the dated "call ahead" line are the gate.*
2. **How specific about cost and eligibility?** Exact dollar figures (PrEP DAP's income limit) are the most useful thing on the page and the most likely to go stale. *Rec: state them, always with "as of {Month Year}" and a link to the source, and check them first in each refresh.*
3. **A services section that starts thin.** Without CM2's recommendations, "who's actually good" can only come from what we personally know. *Rec: launch the services hub honestly thin (the GSBA pointer, our few verified listings, "recommendations are coming"), rather than padding it with names we can't vouch for.*
4. **Gender-affirming care as link-out only.** Per Q2, we point to the trans community's own rails rather than curating them. *Rec: as stated. The paragraph in 1.8 names and credits them; no separate gender-affirming guide.*

# Change Log

- **11/1/2026 (v0.1)** — Initial draft from the outline's GD6 blurb, the brainstorm's Pillars 2.3–2.4 and Appendix E, GD1 §6.8's directory-not-advice rule, and the MHMDA posture. Eight health articles, a hub, three services articles and a twice-yearly refresh calendar. Four open questions.
