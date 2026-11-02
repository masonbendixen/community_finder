---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Levi Kuhn (owner) · Mason Bendixen
Last Updated: 11/1/2026
Version: 0.1
tags:
---

# Bucket EV3 — Submission pipeline

**Track:** EV — Events engine · **Size:** S · **Stream:** A · **Owner:** Levi
**Needs:** EV1 (done: `origin = 'user_submitted'` is reserved, and the review endpoint exists) · EV2 §3.7 (done: the `/admin/events` review queue) · accounts (done: registration has been public since SUTP Q7). **Feeds:** the Success Metrics "community signal" row (outside submissions per week) · CM1 (the same moderate-before-publish pattern, applied to listings) · RT2 (the terms of service must cover submissions before the public launch) · M3.

# Context

Implements [[Initial Project Implementation Outline]] Stream A bucket EV3: "Public submit form (the single-intake rule stated loudly), pending queue, admin review UI polish, submitter feedback. Light UGC — moderated before publish."

Where it comes from in [[Brainstorming on the website]]:

- **Pillar 1, intake stack item 1:** "a mandatory structured submission form as the only human intake." This is the NO STRAIGHT PLANS labor hack. Their rule, "events submitted via email/DM will NOT be considered," turns curation from research into moderation.
- **The freshness architecture, item 6:** the single-intake rule keeps human labor at moderation, not research.
- **Pillar 5, item 1:** "event submissions with review queue (the form IS the community feature)."
- **The Instagram/Facebook wall** (Appendix A, "Blocked/never"): IG and FB event traffic is "the biggest supply we route around via the submission form." Organizers who only post on Instagram reach us through this form.

**What already exists.** EV1 reserved `events.origin = 'user_submitted'`, and `created_by_person_id` records who entered an event. EV2 §3.7 built the curator side: a pending queue at `/admin/events` with Approve, Reject and Edit. EV2's OQ9 (j) left one gap for this bucket: "Reject asks for its note in the browser's own pop-up box. EV3 owns any queue polish."

So this bucket adds the **submitter's side** (the form, "my submissions," and the email when a decision is made), plus the queue polish that submissions need.

**Read before starting:** [[Bucket EV1 — Events domain, server]] (house conventions, the review endpoint, OQ8 (e) on person columns) and [[Bucket EV2 — Events public UI]] §3.7 (the queue and form this bucket extends, and the Pacific-time form rules in `manage-events-form.ts`).

# Scope

**In:**
- The `event_submissions` table and `events.submission_id`.
- Table helpers and a `SubmissionHelper`, covering validation, rate limits, duplicate hints, withdrawal, and the decision email.
- Submitter endpoints under `/api/event_submissions*`, plus an extension of the review endpoint.
- The `/events/submit` page and the `/account/submissions` page.
- Queue polish in `/admin/events`.
- test_helper commands and tests at every layer.

**Out (explicit non-goals):**
- Anonymous submissions (OQ1).
- Photos from submitters (OQ7, and CM3's consent policy).
- Organizers editing an event after they submit it (OQ5).
- Venue or org self-service claiming (**CM1**).
- Feed and scanner intake (**EV6**/**EV5**, which are separate doors).
- Cross-source duplicate merging (EV5's follow-up). This bucket only *hints* at duplicates; it never merges.
- The terms-of-service text itself (**RT2**).

# Interface contracts this bucket must honor

1. **Namespaces (outline Contract 5):** Stream A owns `/api/events*`, `/api/event_*` and `manage_events`. Submitter routes are `/api/event_submissions*`. They follow EV1 OQ4's precedent of a sibling path rather than nesting under `/api/events/<int>`. Client routes are `/events/submit` and `/account/submissions`.
2. **Shared-file etiquette (Contract 4):** one-line, alphabetized additions to `web_app.cpp`, the CMakeLists, `create_app_tables.cpp` (own lines only) and `app.routes.ts`. The header and footer gain one "Submit an event" link each. Those are Stream C's files, so announce it in the PR.
3. **No outing (GD1 §6.9):** who submitted an event is never shown publicly. Curators see it; the public event page never does.

# Design pin-downs

- **Who can submit:** any signed-in account with a **verified email** (honuware's `email_verified`). No role or permission is needed, so a fresh registration can submit once it has confirmed its address. An unverified account gets a 403 with "Confirm your email address first," and the form shows that up front.
- **The single-intake rule, stated loudly:** the form's header says *"This form is the only way we take event listings. We don't take events by email, DM, or comment, so please don't send them that way."* The footer, the empty events list ("Know of something? Submit it") and RT2's About page repeat it.
- **What a submission is:** one set of details (title, place, description, link, cost, tags) on **1–12 dates**. Each date becomes its own `events` row:
	- `origin = 'user_submitted'`, `status = 'pending'`
	- `created_by_person_id` = the submitter
	- `source_id` and `external_key` NULL. EV1's idempotency constraint ignores NULL sources, so nothing collides.
	- `submission_id` points at the shared envelope.

	"It happens every 2nd Saturday" means listing the next few dates and ticking **"This is a regular thing,"** which tells a curator to consider making a series. Series stay curator-made (EV1 pin 3: series are labels, not generators).
- **Times** are typed in Seattle time and stored as `_us` UTC instants. This reuses EV2's `manage-events-form.ts` rules: an end earlier than the start means the next morning, and DST edges are tested there.
- **Validation (server, `SubmissionHelper`):**
	- **Title:** 3–120 characters.
	- **Dates:** each start is at least 1 hour from now and at most 365 days out. An end must come after its start and at most 4 days after it, so multi-day festivals fit.
	- **Count:** 1–12 dates, no two identical.
	- **Links:** `http`/`https` only, at most 500 characters.
	- **Lengths:** description ≤ 4,000, place ≤ 300, cost ≤ 100, note ≤ 1,000.
	- **Tags:** at most 3 categories and 3 scene tags. Unknown or disabled slugs are dropped and reported, as EV1 OQ10 (c) does for ingest.
	- **Venue:** an unknown `venue_id` is a 400.
	- **Error codes:** `TITLE_LENGTH`, `START_TOO_SOON`, `START_TOO_FAR`, `END_BEFORE_START`, `TOO_LONG`, `TOO_MANY_DATES`, `DUPLICATE_DATE`, `BAD_URL`, `FIELD_TOO_LONG`, `VENUE_NOT_FOUND`, `RATE_LIMITED`, `EMAIL_NOT_VERIFIED`.
- **Place:** pick a venue from the list (type-ahead over the public `venues` read, which is already in `allowed_tables`), or type a place name and address into `location_text`. A curator can attach or create the venue while reviewing.
- **Rate limits (OQ3):**
	- at most **10 pending dates** per person across all their submissions;
	- at most **5 submissions** per rolling 24 hours.

	Exceeding either is a 429 with a plain message ("You have 10 dates waiting for review. We'll get to them soon."). The counts come from the tables, so they need no new infrastructure.
- **Duplicate hints, never blocks.** For each date, the server looks for approved or pending events that start within ±3 hours and either have the same venue or the same normalized title (lower-cased, punctuation stripped). Matches come back to the submitter ("Is it one of these?") and show in the queue as "possible duplicate of #123." Nothing is refused for being similar.
- **Abuse basics:**
	- a hidden honeypot field: if it's filled in, the server answers 200 and writes nothing;
	- the agreement checkbox ("This is accurate, and I'm OK with it being listed publicly") must be ticked;
	- the `spam` reject reason;
	- the framework's account disable for repeat offenders.
- **Review (the queue polish):** EV2's queue gains, for submitted events:
	- an origin badge ("Submitted by {display name}");
	- the submitter's note and relationship ("I'm the organizer");
	- the dates of one submission grouped together, with **Approve all / Reject all**;
	- duplicate hints, and a "no link" flag;
	- an in-page reject panel replacing the browser pop-up: a **reason** (`duplicate` · `not-a-fit` · `missing-details` · `past` · `spam` · `other`), a curator-only note (`review_notes`, as today), and an optional **message to the submitter**.
- **Submitter feedback:**
	- **"My submissions"** (`/account/submissions`): each submission, its dates and their states ("Waiting for review" · "Listed" with a link · "Not listed" with the reason and message · "Withdrawn"), newest first.
	- **One email per submission**, sent when its last pending date is decided. It lists each date's outcome and the curator's message. It is stamped `decision_notified_us` so it is sent once. A per-date email would send twelve emails for a twelve-date submission.
	- Withdrawing sends no email.
	- If EV4's archive job closes a submission because its dates passed unreviewed, EV4 sends the same email with a "we didn't get to this in time, sorry" line (EV4 §2.3).
- **Withdraw:** a submitter can withdraw a submission while any of its dates are pending. Those dates become `rejected`, with reason `withdrawn` and the submitter recorded as reviewer. Approved dates stay listed, and the submitter asks a curator through the site's contact address to remove one.
- **After approval the event is the curators'.** The submitter's view is read-only (OQ5).
- **Mail:** the decision email uses the framework mail path that account verification uses (`util/mail/mail_helper.h`), with the From address resolved by `::Mail::LoadSenderAddress(secrets, txn)`. The body is built with `FormatString` templates and wrapped in `NormalizeCrLf`. A send failure writes an `admin_alerts` row and leaves `decision_notified_us` empty, so the next decision on that submission, or EV4's digest, surfaces it. It never fails the review.

# Layered work items

*(Numbered; lower layers first; every item names its tests.)*

### 1. db_schema

- [ ] 1.1 `src/db_schema/event_submissions.{h,cpp}`. Columns:
	- `submitted_by_person_id` (BIGINT, required, a **plain id, not an FK**, per EV1 OQ8 (e): deleting an account must not delete its events)
	- `submitter_note` (text, default `''`)
	- `relationship` (CHECK `IN ('organizer','venue','performer','attendee','other')`, default `'other'`)
	- `regular_thing` (bool, default false)
	- `reject_reason` (nullable; CHECK in the six reasons plus `withdrawn`)
	- `message_to_submitter` (default `''`)
	- `decision_notified_us` (nullable)
	- `created_us`, `updated_us`
	- Index `(submitted_by_person_id, created_us)`, which serves "my submissions" and the 24-hour count.

	*Note:* the reason lives on the submission, not the event. "Reject all" is the normal case, and a per-date reason would be noise.
- [ ] 1.2 `events.submission_id`: a nullable FK to `event_submissions` (cascade: deleting a spam submission deletes its dates, which is wanted), plus index `events_submission_idx`. `make_app_tables.cpp` order: `event_submissions` goes **before** `events`.
	- **Pre-launch, a `--recreate_database` covers it.** Once real data exists, a column on an existing table needs a migration in `BuildAllMigrations`. Note it in the PR.
- [ ] 1.3 `create_app_tables.cpp` (the three-place rule):
	- admin metadata for `event_submissions`: top-level, friendly names, display template `Submission #{id}`, and column info with the person id and timestamps read-only;
	- nest it under `events`;
	- add `event_submissions` to the events-domain `admin_table_permissions` list, which becomes 12 tables;
	- **not** in `allowed_tables`.

	**Tests:** `make_database_info_test.cpp` (columns, CHECKs, the FK, the index) and `create_app_tables_test.cpp` (the metadata, the twelve-table permission map, and staying off the public list).

### 2. table_helpers

- [ ] 2.1 `event_submissions.{h,cpp}`: `AddSubmission`, `GetSubmission`, `GetSubmissionsForPerson(personId)` (newest first, `id` tiebreak), `CountSubmissionsSince(personId, sinceUs)`, `UpdateSubmission` (bumps `updated_us`), `MarkDecisionNotified(id, nowUs)`. **Test.**
- [ ] 2.2 `events.{h,cpp}` additions:
	- `GetEventsBySubmission(submissionId)`;
	- `GetEventsBySubmissions(ids)` (batch);
	- `CountPendingForCreator(personId)`;
	- `GetEventsStartingBetween(fromUs, toUs, statuses)`, the duplicate-hint scan, which reuses the `(status, starts_at_us)` index.

	**Tests** for each, including the cascade (deleting a submission deletes its dates).

### 3. business_logic (`src/business_logic/events/`)

- [ ] 3.1 `submission_helper.{h,cpp}`, in the knottyyoga style: request/result structs and error-code strings, no exceptions.
	- `Submit(Transaction&, const SubmissionRequest&, personId) → SubmitResult`, in two passes like `IngestEvents`. The first validates everything and checks the rate limits. The second writes the envelope, the dates and the tag assignments. The result carries the ids, any ignored slugs, and the duplicate hints per date.
	- `GetMySubmissions(Transaction&, personId) → std::vector<SubmissionView>`, with each date's status, and reason and message where decided.
	- `Withdraw(Transaction&, submissionId, personId)`: `NOT_YOURS`, `NOTHING_PENDING`.
	- `OnDateReviewed(Transaction&, eventId) → std::optional<DecisionNotice>`. When the event belongs to a submission with no pending dates left and no notice sent yet, it returns the notice (recipient, per-date outcomes, message).
	- `FindDuplicateHints(...)`, a free function that is unit-tested without a database: the normalization and the ±3-hour window.

	**Tests** (`submission_helper_test.cpp`):
	- every validation code;
	- both rate limits, at the boundary;
	- the honeypot writing nothing;
	- dates landing `pending`/`user_submitted` with the creator set and no source;
	- the 13th date refused;
	- duplicate hints by venue and by title, and none across a 4-hour gap;
	- withdraw by the owner and by someone else;
	- `OnDateReviewed` returning a notice only after the last pending date, and only once;
	- an unverified account refused.
- [ ] 3.2 `submission_email.{h,cpp}`: `RenderDecisionEmail(notice, siteName, siteUrl) → {subject, body}`, using `FormatString` placeholders and `NormalizeCrLf`. **Test:** all listed, all not listed, a mix, the message included, no message.

### 4. endpoints (EV1 §5 conventions: PascalCase methods, volatile anchors, `handle_full` tests with `ThreadPool::Shutdown()`)

- [ ] 4.1 `event_submissions.cpp`, one TU and one anchor:
	- `POST /api/event_submissions`: signed in and verified. Body `{title, dates:[{starts_at_us, ends_at_us?}], venue_id?, location_text?, description?, url?, cost_text?, category_slugs?[], scene_tag_slugs?[], regular_thing?, relationship?, submitter_note?, agreed:true, website?}` (`website` is the honeypot). Response `{submission_id, event_ids, ignored_slugs, possible_duplicates}`.
	- `GET /api/event_submissions/mine`.
	- `POST /api/event_submissions/<int>/withdraw`.

	**Tests:** 401 signed out · 403 unverified · 400 per validation family, with `error_code` · 429 at each limit · the happy path's rows · `mine` lists only your own, with statuses · withdraw · someone else's withdraw is a 404, so another person's submission ids reveal nothing.
- [ ] 4.2 Extend `admin_review_event.cpp`: optional `reason` and `message_to_submitter`, stored on the submission. After a successful review it calls `OnDateReviewed`, and on a notice it sends the email (§3.2) and stamps it. New sibling route `POST /api/admin/review_submission/<int>` `{action, reason?, notes?, message_to_submitter?}` approves or rejects every pending date of a submission in one transaction and sends one email.

	**Tests:**
	- a reviewed submitted date stores the reason and message;
	- the email is sent once, after the last date;
	- the group route decides all pending dates and leaves decided ones alone;
	- a mail failure still returns 200 and writes an `admin_alerts` row;
	- the existing review tests stay green.

	The fake mail sender is the framework's test hook. Check `util/mail/` for the exact seam before writing these.
- [ ] 4.3 The queue needs to read submissions. Rather than a new endpoint, `event_submissions` goes into the generic CRUD access through its `manage_events` table permission (§1.3), the way EV2 §3.7 reads `events`. **Test:** `crud_test.cpp`, where an admin reads it and a signed-out caller is refused.
- [ ] 4.4 Shared-file one-liners: the anchor, the CMakeLists sources.

### 5. client (`ui/src/app/`)

- [ ] 5.1 Access layer: `core/event-submissions-access/` (interface, HTTP, mock, providers; the `events-admin-access` precedent), with `submit`, `getMine` and `withdraw`. The mock keeps submissions in memory and fakes duplicate hints. **Spec:** URLs and bodies, and the mock's behavior.
- [ ] 5.2 `pages/events/submit-event.*` (`/events/submit`, ahead of `:id` in `events.routes.ts`).
	- Signed out: a short explainer with sign-in and register buttons that return here.
	- Unverified: "confirm your email" with a resend link.
	- **The form:**
		- the single-intake banner;
		- title;
		- a date list (add or remove rows, 1–12, each a date, start time and optional end time in Seattle time);
		- venue type-ahead, or "Somewhere else" with a place field;
		- description, link and cost;
		- up to 3 category and 3 scene chips;
		- "This is a regular thing";
		- "How do you know about it?" (relationship);
		- a note to curators;
		- the agreement checkbox;
		- the hidden honeypot.
	- **After sending:** "Thanks, it's waiting for review. We'll email you when it's decided," any possible duplicates with links ("if it's already listed, you can withdraw yours"), and links to "My submissions" and "Submit another."

	**Specs:** Seattle time to `_us` for several dates (reuse the form-rule tests' DST fixtures); the 12-date cap; each server error code shown next to the right field; signed-out and unverified states; duplicates shown; the honeypot never visible.
- [ ] 5.3 `pages/account/my-submissions.*` (`/account/submissions`, signed in): cards per submission with date rows and status chips, the reason and message when not listed, a link when listed, and Withdraw while pending, with a confirm in the page, not a browser pop-up. **Specs:** each status renders; withdraw updates in place.
- [ ] 5.4 Links: a "Submit an event" button on the events list header and the empty state; the header's account menu gets "My submissions"; the footer gets "Submit an event." **Specs:** the existing header and footer specs updated.
- [ ] 5.5 Queue polish in `pages/admin/manage-events.*`:
	- the origin badge and submitter;
	- the note and relationship;
	- grouping by `submission_id` with Approve all / Reject all;
	- duplicate and no-link flags;
	- the in-page reject panel (reason, curator note, message to the submitter), replacing `window.prompt`.

	**Specs:** grouping; approve-all calls the group route once; the reject panel sends all three fields; non-submitted events look as before.

### 6. test_helper (`src/test_helper/commands/submission_commands.{h,cpp}`)

- [ ] 6.1 Two commands:
	- `submit-event [se] --as=<email> --title=… --starts_in_hours=N [--dates=3] [--venue_id=N]`, which creates a verified non-admin demo person if the email is new;
	- `list-submissions [ls] [--as=<email>]`.

	**Tests** (`submission_commands_test.cpp`): rows created as that person; the limits enforced through the command too. *The "Demo:" title prefix convention from EV1 §7.4 applies.*

### 7. Gates

- [ ] 7.1 Docker suite green per slice, Windows too ("Windows: N passed. Linux: M passed."); raise the floor at bucket exit (EV1 §7.2 policy).
- [ ] 7.2 `ng build` for local, development and production; vitest green, with the new count recorded.
- [ ] 7.3 Live seed check after §1 (destructive, announced): `event_submissions` present with its CHECKs and index, `events.submission_id` FK present, twelve tables mapped to `manage_events`, nothing new in `allowed_tables`.
- [ ] 7.4 **Manual loop (Mason or Levi):**
	1. Register a fresh account and confirm its email. The form refuses it before confirmation.
	2. Submit a three-date event.
	3. As an admin, open `/admin/events`: the three dates show grouped, with the badge.
	4. Approve two, then reject one with a message. One email arrives, listing all three outcomes.
	5. "My submissions" shows Listed, Listed, Not listed with the message.
	6. Submit again and withdraw it.
	7. Push past the pending limit and read the 429 message.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Must people sign in to submit an event?** Signing in means registering and confirming an email first. That's a small hurdle for an organizer, but it gives us a real email to reply to, a way to stop a repeat spammer (disable the account), and a "My submissions" page with no extra machinery. The alternative is a no-account form that emails a confirmation link to whatever address is typed. That is friendlier, but it needs its own confirmation tokens, its own spam defenses, and a way to show status without an account. *Rec: signed in with a confirmed email at launch; revisit if the submission numbers say the hurdle is costing us.*
	- Mason- They need to sign in to submit events.
2. **Up to 12 dates, no "repeats forever."** A weekly event is listed by adding its next few dates and ticking "This is a regular thing," and a curator then decides whether it deserves a series page. The alternative is letting submitters set up a rule ("every 2nd Saturday"), but EV1 deliberately has no repeat rules (series are labels, not generators). *Rec: 12 dates, with series made by curators.*
	- Mason- works for me
3. **Rate limits: 10 dates waiting for review per person, and 5 submissions a day.** Enough for a busy organizer, small enough that one bad account can't flood the queue. *Rec: as stated; both are constants and easy to change.*
	- Mason- Let's make this configurable in the admin console with bespoke UI.
4. **One email per submission, when it's fully decided.** It lists each date's outcome and the curator's message. Per-date emails would spam a twelve-date organizer. No email at all would leave submitters guessing, which is how SGN's mail-and-fax calendar feels. *Rec: one email per submission; withdrawals send none.*
	- Mason- I'll go with your recommendation.
5. **No edits after submitting.** To fix a typo, a submitter withdraws and resubmits while it's still waiting, or writes to us once it's listed. Letting submitters edit approved events would reopen review every time. Letting them edit pending ones is possible, but it adds a form state and a race with a curator mid-review. *Rec: no edits at launch. CM1's correction flow covers listed events later.*
	- Mason- I'll go with your recommendation.
6. **Does submission need the terms of service first?** Submissions are user-generated content, so RT2's terms should say how we use what's submitted. *Rec: build it now. The public launch (M3) already requires RT2, so the terms exist before strangers can reach the form. No feature switch is needed.*
	- Mason- I'll go with your recommendation.
7. **Photos from submitters: not yet.** A flyer image would make listings nicer, but photos raise the consent and no-outing questions CM3 exists to answer (a party photo can out someone), and they invite storage abuse. *Rec: no submitter photos in EV3; a curator can add the flyer on review through the existing photo pipeline.*
	- Mason- I'll go with your recommendation.
8. **Is a link required?** A link (ticket page, Instagram post, venue page) is the fastest way for a curator to check that an event is real. But some grassroots events have none. *Rec: optional, with the queue flagging "no link" so the curator knows to look harder.*
	- Mason- I'll go with your recommendation.

# Change Log

- **11/1/2026 (v0.1)** — Initial draft from the outline's EV3 blurb, the brainstorm's intake stack, EV1's reserved `user_submitted` origin and review endpoint, and EV2 §3.7's queue (including its OQ9 (j) hand-off of the reject pop-up). Eight open questions.
