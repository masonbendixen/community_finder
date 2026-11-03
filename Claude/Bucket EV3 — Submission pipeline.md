---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Levi Kuhn (owner) · Mason Bendixen
Last Updated: 11/3/2026
Version: 0.8
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
- The **Submission limits** admin page and its endpoint, so curators can change the rate limits without a deploy (OQ3). The page also has a **pause switch** that closes the form for everyone (OQ10).
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
	- *As built (§3–4):* the 403 is a backstop. honuware's sign-in already refuses an account whose email isn't confirmed, so a signed-in caller is nearly always verified. The test reaches the 403 by clearing `email_verified_at` after signing in. This makes §5.2's "Unverified" state mostly moot (see §5.2).
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
		- *As built:* four more codes: `NOT_AGREED` (the checkbox), `NO_DATES`, `BAD_RELATIONSHIP` and `TOO_MANY_TAGS` (more than 3 of a kind after unknown slugs are dropped).
		- Every 400 also says which form field it is about (`field`, by its JSON name). A date's error also says which date (`date_index`, counting from 0), so §5.2 can put the message beside the right row.
		- Titles and other one-line fields are trimmed, and control characters become spaces (a title ends up in an email subject). Lengths count characters, not bytes.
		- The checks run in the form's order, and the rate limits run last, so a form with a mistake hears about the mistake before it hears about a limit.
- **Place:** pick a venue from the list (type-ahead over the public `venues` read, which is already in `allowed_tables`), or type a place name and address into `location_text`. A curator can attach or create the venue while reviewing.
- **Rate limits (OQ3: admin-configurable, with a page of their own):**
	- at most **10 pending dates** per person across all their submissions, by default;
	- at most **5 submissions** per rolling 24 hours, by default.

	Exceeding either is a 429 with a plain message that uses the current number ("You have 10 dates waiting for review. We'll get to them soon."). The counts come from the tables, so they need no new infrastructure.
	- *As built:* the 429's `detail` is the message, its `error_code` is `RATE_LIMITED`, and its `field` names the limit (`max_pending_dates` or `max_submissions_per_day`). The two messages:
		- "You can have up to 10 dates waiting for review at a time. You have 9 waiting, and this would add 2. We'll get to them soon."
		- "You can send up to 5 submissions in 24 hours. Please try again later."
	- **Where the numbers live:** two app keys in `config_secrets`, with their defaults (10 and 5) in `app_secret_values.cpp`. That is honuware's store for site settings, and the seam `app_secret_keys.h` reserves for "a domain feature [that] needs a tunable." The generic table editor can't reach `config_secrets`, so the only way to change them is a dedicated endpoint that validates what it writes. honuware's Site Theme editor works the same way. There's no new table and no migration: a key missing from an older database reads as its default.
	- **Allowed ranges:** pending dates 1–100, and submissions per 24 hours 1–50. A stored value that is blank, not a number, or out of range reads as the default, so a bad value can never switch a limit off.
	- **The page:** `/admin/events/settings`, titled "Submission limits" and linked from the `/admin/events` header. It has two number fields, each with a one-line explanation and its default shown beside it, plus Save and "Reset to defaults." Anyone with `manage_events` can open it (OQ9).
- **Pause switch (OQ10, decided v0.5):** one switch that closes the submit form for everyone. It's for a spam wave from many new accounts, which the per-person limits can't stop.
	- **Where it lives:** a third app key next to the two limits, `event_submissions_paused`, `"false"` by default. Only the exact value `"true"` pauses submissions. A missing, blank or garbled value reads as open, so a bad value can never close the form by accident.
	- **What it stops:** while paused, `POST /api/event_submissions` refuses every submission with a **503**, `error_code` `SUBMISSIONS_PAUSED`, and the notice *"We're not taking event submissions right now. Please check back soon."*
		- It is checked right after "signed in and verified," before any field. Someone who sends a form gets the notice, not a list of field errors.
		- 503 means "not right now," the same status honuware uses when sign-in is temporarily unavailable.
	- **What keeps working:** "My submissions," withdraw, and curator review. Pausing closes the front door only. The queue still drains.
	- **How the form knows up front:** a public `GET /api/event_submissions/status` → `{accepting_submissions: true|false}`, with no sign-in needed. The submit page checks it on load and shows the notice instead of the form, so nobody fills in twelve dates for nothing.
	- **Who can flip it:** anyone with `manage_events`, on the Submission limits page (OQ9).
	- **So it isn't left on by accident:** while paused, `/admin/events` shows a banner, "Submissions are paused," with a link to turn them back on. Pausing asks for an in-page confirmation; unpausing doesn't.
- **Duplicate hints, never blocks.** For each date, the server looks for approved or pending events that start within ±3 hours and either have the same venue or the same normalized title (lower-cased, punctuation stripped). Matches come back to the submitter ("Is it one of these?") and show in the queue as "possible duplicate of #123." Nothing is refused for being similar.
	- *As built:* the hints sent back to the submitter cover **approved events only**. Another person's pending submission isn't theirs to see, and listing it would reveal what someone else sent in. The queue's "possible duplicate" flag (§5.5) is worked out in the browser from the queue's own rows, which include pending ones, so curators still see both kinds. The server's matching is `FindDuplicateHints`, a pure function; §5.5 mirrors its two rules.
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
	- *As built:* each withdrawn date carries the review note "withdrawn by the submitter." That note is how "My submissions" tells a withdrawn date apart from one a curator turned down. The reason becomes `withdrawn` only if a curator hasn't already given one. Withdrawing stamps `decision_notified_us`, so no later decision on the submission sends an email either.
- **After approval the event is the curators'.** The submitter's view is read-only (OQ5).
- **Mail:** the decision email uses the framework mail path that account verification uses (`util/mail/mail_helper.h`), with the From address resolved by `::Mail::LoadSenderAddress(secrets, txn)`. The body is built with `FormatString` templates and wrapped in `NormalizeCrLf`. The sender claims `decision_notified_us` first (`MarkDecisionNotified`, §2.1) and sends only if it won the claim. A send failure writes an `admin_alerts` row and clears the stamp, leaving `decision_notified_us` empty, so the next decision on that submission, or EV4's digest, surfaces it. It never fails the review.
	- *As built:*
		- The email is sent right away, inside the review's own transaction. That is how honuware sends its account-verification email (`person.cpp`). Both review responses say whether it went (`decision_emailed`).
		- **Two curators at once.** Every path that decides a submission's dates (review one date, review all, withdraw) first locks the submission's row (`GetSubmissionForUpdate`, §2.1).
			- Without the lock, two curators deciding the last two dates at the same moment would each still see the other's date as pending, and neither would send the email.
			- With it, the second curator waits for the first, then sees that curator's decisions.
			- The lock is always taken before any of the dates are touched, so the order is the same everywhere, and two reviews can't deadlock.

# Layered work items

*(Numbered; lower layers first; every item names its tests.)*

### 1. db_schema

*Done 11/2 — Windows: 2233 passed. Linux: 2233 passed (+2 new tests; floor unchanged at 2221 until the bucket exit). The live seed check (§7.3) hasn't been run yet: it's destructive and waits for the go-ahead.*

- [x] 1.1 `src/db_schema/event_submissions.{h,cpp}`. Columns:
	- `submitted_by_person_id` (BIGINT, required, a **plain id, not an FK**, per EV1 OQ8 (e): deleting an account must not delete its events)
	- `submitter_note` (text, default `''`)
	- `relationship` (CHECK `IN ('organizer','venue','performer','attendee','other')`, default `'other'`)
	- `regular_thing` (bool, default false)
	- `reject_reason` (nullable; CHECK in the six reasons plus `withdrawn` and `expired`, the last for EV4's expiry sweep, so "My submissions" can say "the date passed before we reviewed it")
	- `message_to_submitter` (default `''`)
	- `decision_notified_us` (nullable)
	- `created_us`, `updated_us`
	- Index `(submitted_by_person_id, created_us)`, which serves "my submissions" and the 24-hour count.

	*Note:* the reason lives on the submission, not the event. "Reject all" is the normal case, and a per-date reason would be noise.
- [x] 1.2 `events.submission_id`: a nullable FK to `event_submissions` (cascade: deleting a spam submission deletes its dates, which is wanted), plus index `events_submission_idx`. `make_app_tables.cpp` order: `event_submissions` goes **before** `events`.
	- **Pre-launch, a `--recreate_database` covers it.** Once real data exists, a column on an existing table needs a migration in `BuildAllMigrations`. Note it in the PR.
- [x] 1.3 `create_app_tables.cpp` (the three-place rule):
	- admin metadata for `event_submissions`: top-level, friendly names, display template `Submission #{id}`, and column info with the person id and timestamps read-only;
	- ~~nest it under `events`~~ **top-level only, not nested.** The editor nests a table under the tables it has an FK onto. Here the FK runs from `events` to `event_submissions`, so the only way to nest would be marking `events` itself as nested, which would also put it under venues, series and sources. *(Corrected in v0.2.)*
	- `events.submission_id` gets column info too, read-only: the pipeline sets it, and a curator never should;
	- add `event_submissions` to the events-domain `admin_table_permissions` list, which becomes 12 tables;
	- **not** in `allowed_tables`.

	**Tests:** `make_database_info_test.cpp` (columns, CHECKs, the FK) and `create_app_tables_test.cpp` (the metadata, the twelve-table permission map, staying off the public list, not nested, the read-only pipeline columns). The two indexes are live-only, as EV1's are (the harness builds tables from `MakeDatabaseInfo`), so §7.3's live seed check covers them.

### 2. table_helpers

*Done 11/2 — Windows: 2248 passed. Linux: 2248 passed (+15 new tests: 10 in `event_submissions_test.cpp`, 5 in `events_test.cpp`).*

- [x] 2.1 `event_submissions.{h,cpp}`: `AddSubmission`, `GetSubmission`, `GetSubmissionsForPerson(personId)` (newest first, `id` tiebreak), `CountSubmissionsSince(personId, sinceUs)` (the window includes `sinceUs`), `UpdateSubmission` (bumps `updated_us`), `MarkDecisionNotified(id, nowUs)`. **Test.**
	- *As built:* `MarkDecisionNotified` returns a `bool`. It stamps only while `decision_notified_us` is still empty (`UPDATE … WHERE decision_notified_us IS NULL RETURNING id`) and says whether this call did the stamping. Only the caller that wins the stamp sends the email. So if two curators decide a submission's last two dates at the same moment, the second waits on the row lock, finds it stamped, and sends nothing. A failed send clears the stamp again (see "Mail" above).
	- *Added while building §3:* `GetSubmissionForUpdate(id)`, which reads the submission and locks its row (`SELECT … FOR UPDATE`) until the transaction ends. Every path that decides dates takes it first (see "Mail" above for why). It has a test.
- [x] 2.2 `events.{h,cpp}` additions:
	- `GetEventsBySubmission(submissionId)`;
	- `GetEventsBySubmissions(ids)` (batch, ordered by submission, then start);
	- ~~`CountPendingForCreator(personId)`~~ **`CountPendingSubmittedBy(personId)`**. It counts through `event_submissions.submitted_by_person_id` rather than `events.created_by_person_id`, so an admin's own ingested or hand-entered pending events never count against their submission limit. *(Renamed in v0.3.)*
	- `GetEventsStartingBetween(fromUs, toUs, statuses)`, the duplicate-hint scan, half-open like `GetEventsInRange`, which reuses the `(status, starts_at_us)` index.

	**Tests** for each, including the cascade (deleting a submission deletes its dates and nothing else).

### 3. business_logic (`src/business_logic/events/`)

*§3.1–3.3 done 11/2, together with §4.1–4.5 (§3.4 was added in v0.5) — Windows: 2312 passed. Linux: 2312 passed. That is 64 new tests, 42 of them here:*
- *25 in `submission_helper_test.cpp`;*
- *8 in `submission_email_test.cpp`;*
- *7 in `submission_settings_test.cpp`;*
- *1 in `app_secret_values_test.cpp`;*
- *1 in `event_submissions_test.cpp`, for `GetSubmissionForUpdate`.*

*§3.4 done 11/2, together with §4.6 and §6.1 — Windows: 2324 passed. Linux: 2324 passed. That is 12 new tests: 2 here (§3.4), 4 in §4.6 and 6 in §6.1.*

- [x] 3.1 `submission_helper.{h,cpp}`, in the knottyyoga style: request/result structs and error-code strings, no exceptions.
	- `Submit(Transaction&, const SubmissionRequest&, personId) → SubmitResult`, in two passes like `IngestEvents`. The first validates everything and checks the rate limits, read through §3.3 on every call so a change on the settings page applies to the very next submission. The second writes the envelope, the dates and the tag assignments. The result carries the ids, any ignored slugs, and the duplicate hints per date.
	- `GetMySubmissions(Transaction&, personId) → std::vector<SubmissionView>`, with each date's status, and reason and message where decided.
	- `Withdraw(Transaction&, submissionId, personId)`: `NOT_YOURS`, `NOTHING_PENDING`.
	- `OnDateReviewed(Transaction&, eventId) → std::optional<DecisionNotice>`. When the event belongs to a submission with no pending dates left and no notice sent yet, it returns the notice (recipient, per-date outcomes, message).
	- `FindDuplicateHints(...)`, a free function that is unit-tested without a database: the normalization and the ±3-hour window.

	**Tests** (`submission_helper_test.cpp`):
	- every validation code;
	- both rate limits, at the boundary, at the defaults and at a lowered setting;
	- the honeypot writing nothing;
	- dates landing `pending`/`user_submitted` with the creator set and no source;
	- the 13th date refused;
	- duplicate hints by venue and by title, and none across a 4-hour gap;
	- withdraw by the owner and by someone else;
	- `OnDateReviewed` returning a notice only after the last pending date, and only once;
	- an unverified account refused.

	*As built:*
	- `Submit` also takes the `SecretsHelper`, which it needs to read the limits (§3.3).
	- **The curator's side moved in here too**, so `admin_review_event` stays thin (validate, authorize, delegate):
		- `ReviewDate(eventId, action, reviewer, notes, decision)` is EV1's `ReviewEvent`, plus the reason and message stored on the submission and the notice when this was the last pending date.
		- `ReviewSubmission(submissionId, …)` decides every pending date at once (§4.2's group route). It returns `SUBMISSION_NOT_FOUND`, `NOTHING_PENDING`, or a decision error.
		- `SendDecisionNotice(notice)` sends the email, or on failure writes the alert and releases the claim.
		- Two more codes: `BAD_REASON`, for a reason that isn't one of the six or comes with an approval, and `FIELD_TOO_LONG`, for a message over 2,000 characters.
	- **Reason and message:** a decision writes only what it states. An approval never clears an earlier reason, and a message replaces the earlier one only when one is sent. A reason or message sent with an event that didn't come from a submission is accepted and ignored.
	- **When the account is gone:** if the submitter's account was deleted before the last decision, the claim is taken and no email is sent.
	- `GetMySubmissions` returns no curator note (`review_notes`): it stays with the curators. Its dates come from one batch read.
	- The failed-send case is tested here (`AFailedSendAlertsAndReleasesTheClaim`, with a mail helper that throws), not at the endpoint (see §4.2).
- [x] 3.2 `submission_email.{h,cpp}`: `RenderDecisionEmail(notice, siteName, siteUrl) → {subject, body}`, using `FormatString` placeholders and `NormalizeCrLf`. **Test:** all listed, all not listed, a mix, the message included, no message.
	- *As built:*
		- **Subjects:** "Your event is listed: {title}", "Your event wasn't listed: {title}", or, for a mix, "Your event has been reviewed: {title}".
		- **The body:** one line per date ("November 13, 2026, 7:00 PM: Listed (see it)" or "…: Not listed"), in Seattle time. Then "Why: …" in plain words, but only when something wasn't listed and the reason has words (`other` has none). Then the curator's message, with its line breaks kept. Last, a link to "My submissions."
		- Everything the submitter or curator typed is HTML-escaped, including `{` and `}`, so typed text can't trigger a `FormatString` placeholder.
		- The tests also cover escaping, the `\r\n` line endings, and the words for each reason.
- [x] 3.3 `submission_settings.{h,cpp}` (OQ3), the rate limits as settings:
	- **Keys:** `kEventSubmissionMaxPendingDates` (`"event_submission_max_pending_dates"`) and `kEventSubmissionMaxPerDay` (`"event_submission_max_per_day"`) go in `app_secret_keys.h`, the app's first keys of its own. Their defaults, `"10"` and `"5"`, go in `app_secret_values.cpp`.
	- `struct SubmissionLimits { int64_t maxPendingDates; int64_t maxSubmissionsPerDay; }`, plus each limit's default and range as constants, so the endpoint and the page never hard-code them.
	- `LoadSubmissionLimits(SecretsHelper&, Transaction&) → SubmissionLimits`. A blank, non-numeric or out-of-range stored value reads as the default.
	- `SaveSubmissionLimits(SecretsHelper&, Transaction&, partial) → SaveLimitsResult`. Only the fields present are written. Any out-of-range field refuses the whole save with `LIMIT_OUT_OF_RANGE` and the field's name, and nothing is written (the Site Theme rule: a half-saved form is worse than a refused one).

	**Tests** (`submission_settings_test.cpp`):
	- the defaults with nothing stored;
	- save then load;
	- blank, junk and out-of-range stored values each read as the default;
	- a save with one bad field writes neither field;
	- saving one field leaves the other alone;
	- `AppSecretValuesTest.FrameworkAndAppKeySetsDoNotOverlap` still green with the two new keys.

	*As built:* as planned. Two tests were added: the registered defaults ("10" and "5") match the code's defaults, and the range ends (1 and 100, 1 and 50) save.
	- *Renamed in §3.4 (v0.6):* once the switch joined the limits, the names above became `SubmissionSettings`, `SubmissionSettingsUpdate`, `SaveSettingsResult`, `LoadSubmissionSettings` and `SaveSubmissionSettings` (`result.settings`). Nothing else changed.
- [x] 3.4 **The pause switch** (OQ10, added v0.5; see "Pause switch"):
	- the key `kEventSubmissionsPaused` (`"event_submissions_paused"`) in `app_secret_keys.h`, with its default `"false"` in `app_secret_values.cpp`;
	- the §3.3 load and save gain a third field, `submissionsPaused`. Only `"true"` reads as paused. A save writes `"true"` or `"false"`, and it is part of the same all-or-nothing save as the limits;
	- `Submit` returns `SUBMISSIONS_PAUSED` (with the notice as its message) right after the verified-email check.

	**Tests:**
	- the switch reads as open with nothing stored, and with a blank or junk value;
	- save then load;
	- saving the switch leaves the limits alone, and saving a limit leaves the switch alone;
	- a paused `Submit` writes nothing, and the same request goes through once it's unpaused;
	- the registered default matches the code's default.

	*As built:*
	- **Where the check sits:** right after the verified-email check, before the honeypot and every field. A paused form gets the notice even when a field is wrong; a bot gets it too.
	- **All or nothing:** a save with an out-of-range limit writes neither the limits nor the switch.
	- **Tests:**
		- `OnlyTrueStoredPausesSubmissions` (new): `""`, `"false"`, `"yes"`, `"TRUE"`, `"True"`, `"1"`, `" true"` and `"true "` all read as open;
		- `SaveThenLoad` now pauses and then resumes, and checks the stored `"true"` / `"false"`;
		- `SavingOneFieldLeavesTheOthers` covers the switch alone and a limit alone;
		- `OneBadFieldWritesNothing` checks the switch isn't written either;
		- `AppSecretDefaultsMatchTheCodeDefaults` (renamed from `…TheLimitDefaults`) covers the switch's `"false"`, and so does `AppSecretValuesTest.RegistersTheSubmissionSettingDefaults` (renamed);
		- `SubmissionHelperTest.APausedSubmitWritesNothing` (new): refused while paused, a bad form hears "paused" too, "My submissions" and withdraw still work, and the same request goes through once unpaused.

### 4. endpoints (EV1 §5 conventions: PascalCase methods, volatile anchors, `handle_full` tests with `ThreadPool::Shutdown()`)

*§4.1–4.5 done 11/2, together with §3.1–3.3 (§4.6 was added in v0.5) — Windows: 2312 passed. Linux: 2312 passed. 22 of the new tests are here:*
- *9 in `event_submissions_test.cpp`;*
- *5 in `admin_event_submission_settings_test.cpp`;*
- *4 in `admin_review_submission_test.cpp`;*
- *3 more in `admin_review_event_test.cpp`;*
- *1 more in `crud_test.cpp`.*

*§4.6 done 11/2, together with §3.4 and §6.1 — Windows: 2324 passed. Linux: 2324 passed. 4 of the 12 new tests are here.*

*The tests share two new helpers in `events_endpoint_test_support`: `SwitchTo` (sign back in as a person already created) and `AddSubmission` (a sample submission written through `SubmissionHelper`).*

- [x] 4.1 `event_submissions.cpp`, one TU and one anchor:
	- `POST /api/event_submissions`: signed in and verified. Body `{title, dates:[{starts_at_us, ends_at_us?}], venue_id?, location_text?, description?, url?, cost_text?, category_slugs?[], scene_tag_slugs?[], regular_thing?, relationship?, submitter_note?, agreed:true, website?}` (`website` is the honeypot). Response `{submission_id, event_ids, ignored_slugs, possible_duplicates}`.
	- `GET /api/event_submissions/mine`.
	- `POST /api/event_submissions/<int>/withdraw`.

	**Tests:** 401 signed out · 403 unverified · 400 per validation family, with `error_code` · 429 at each limit · the happy path's rows · `mine` lists only your own, with statuses · withdraw · someone else's withdraw is a 404, so another person's submission ids reveal nothing.

	*As built:*
	- **Ignored slugs are reported separately for each kind:** `ignored_category_slugs` and `ignored_scene_tag_slugs` replace `ignored_slugs`.
	- **Each `possible_duplicates` entry** is `{date_index, event_id, title, starts_at_us}`.
	- **Errors:**
		- A 400 carries `error_code`, `field` and, for a date, `date_index`.
		- A wrong JSON type (a number for the title, say) is a 400 naming the field. It is refused before the helper sees it.
		- `EMAIL_NOT_VERIFIED` is a 403, and `RATE_LIMITED` is a 429 (see "Rate limits").
	- **The honeypot** answers 200 with `submission_id: 0`.
	- **`GET …/mine`** returns `{items:[{id, created_us, relationship, regular_thing, submitter_note, reject_reason, message_to_submitter, dates:[{event_id, title, starts_at_us, ends_at_us|null, state}]}]}`, newest first.
		- `state` is `waiting`, `listed`, `not_listed`, `withdrawn` or `past`.
		- Dates are in start order.
		- No curator note is ever included.
	- **Withdraw** returns `{submission_id, withdrawn_event_ids}`.
		- Someone else's submission and a missing one both get the same 404 body.
		- A second withdraw is a 400 `NOTHING_PENDING`.
- [x] 4.2 Extend `admin_review_event.cpp`: optional `reason` and `message_to_submitter`, stored on the submission. After a successful review it calls `OnDateReviewed`, and on a notice it sends the email (§3.2) and stamps it. New sibling route `POST /api/admin/review_submission/<int>` `{action, reason?, notes?, message_to_submitter?}` approves or rejects every pending date of a submission in one transaction and sends one email.

	**Tests:**
	- a reviewed submitted date stores the reason and message;
	- the email is sent once, after the last date;
	- the group route decides all pending dates and leaves decided ones alone;
	- a mail failure still returns 200 and writes an `admin_alerts` row;
	- the existing review tests stay green.

	The fake mail sender is the framework's test hook. Check `util/mail/` for the exact seam before writing these.

	*As built:*
	- **Each route** checks `manage_events` first, then reads the body, then hands off to `SubmissionHelper` (§3.1). The body reader is shared by both routes (`review_request_body.{h,cpp}`).
	- **Responses:**
		- `review_event` returns `{event_id, status, decision_emailed}`.
		- `review_submission` returns `{submission_id, status, decided_event_ids, decision_emailed}`.
	- **Errors:**
		- A bad decision is a 400 with `error_code` and `field`: `BAD_REASON` names `reason`, and `FIELD_TOO_LONG` names `message_to_submitter`.
		- For the group route, an unknown submission is a 404, and one with nothing pending is a 400 `NOTHING_PENDING`.
	- **The failed send is tested one layer down.** honuware's endpoint test harness wires in a fake mail sender that always succeeds and can't be swapped, so an endpoint test can't make a send fail. The "returns 200 and writes an `admin_alerts` row" case is therefore tested in §3.1 (`AFailedSendAlertsAndReleasesTheClaim`). The endpoint only passes the result through.
	- **The other cases are tested here:**
		- the stored reason and message;
		- exactly one email, sent with the last date, and none on a later re-decision;
		- the group route leaving decided dates alone;
		- a reason on a feed's event, ignored with no email.
- [x] 4.3 The queue needs to read submissions. Rather than a new endpoint, `event_submissions` goes into the generic CRUD access through its `manage_events` table permission (§1.3), the way EV2 §3.7 reads `events`. **Test:** `crud_test.cpp`, where an admin reads it and a signed-out caller is refused.
	- *As built:* `OnlyManageEventsReadsEventSubmissions` runs the real seed. A curator with `manage_events` (not a full admin, since curators are the real audience) gets 200 and the row. A signed-out visitor and a signed-in member without the permission both get 400, the generic contract's "not an allowed table."
- [x] 4.4 Shared-file one-liners: the anchors, the CMakeLists sources.
	- *As built:* three new anchors in `web_app.cpp`, one per TU. The settings TU's GET anchor also carries its PUT. The submitter TU's `PostEventSubmission` anchor also carries `mine` and withdraw.
- [x] 4.5 `admin_event_submission_settings.cpp` (OQ3), one TU and one anchor, gated by `manage_events` (OQ9):
	- `GET /api/admin/event_submission_settings` → `{max_pending_dates: {value, default, min, max}, max_submissions_per_day: {…}}`;
	- `PUT /api/admin/event_submission_settings` with `{max_pending_dates?, max_submissions_per_day?}` → the same shape, as saved.

	**Tests:** 401 signed out · 403 without `manage_events` · GET returns the defaults and ranges · PUT one field leaves the other · 400 `LIMIT_OUT_OF_RANGE` names the field and writes nothing · after lowering the pending limit to 2, the next `POST /api/event_submissions` over it is a 429 whose message says 2.

	*As built:* as planned. In a PUT, a missing field and a `null` field both mean "leave it as it is." Three PUTs are 400s:
	- a field that isn't a whole number (`"ten"`, `2.5`), which names the field;
	- a body with neither field;
	- a body that isn't JSON.
- [x] 4.6 **The pause switch** (OQ10, added v0.5; see "Pause switch"):
	- `POST /api/event_submissions` answers `SUBMISSIONS_PAUSED` with a 503, its notice as `detail`;
	- new public route `GET /api/event_submissions/status` → `{accepting_submissions}`, with no sign-in needed. It is in the same TU, so it rides the existing anchor;
	- the settings GET and PUT gain `submissions_paused: {value, default}`. A PUT accepts `true` or `false`; anything else is a 400 that names the field. A PUT with only the switch is allowed.

	**Tests:**
	- while paused, a submit is a 503 with `SUBMISSIONS_PAUSED`, and `mine` and withdraw still work;
	- the status route answers signed out, both paused and open;
	- the PUT flips the switch and leaves the limits alone, and a non-boolean is a 400;
	- the 401 and 403 cases cover the switch too.

	*As built:* as planned.
	- **The 503's body** carries `error_code` and `detail`, and no `field`.
	- **The status route** is `GetEventSubmissionStatus` in `event_submissions.cpp`. It rides the `PostEventSubmission` anchor.
	- **The PUT:**
		- `submissions_paused` must be a JSON boolean. `"true"`, `1`, `0` and `"yes"` are each a 400, "submissions_paused must be true or false", naming the field.
		- `null` means "leave it," as for the limits.
		- A body with nothing to change is a 400, "No submission settings in the request."
		- An out-of-range limit sent beside the switch writes neither.
	- **Tests:**
		- in `event_submissions_test.cpp`, two new: `APausedSubmitIs503ButMineAndWithdrawStillWork` and `TheStatusRouteAnswersSignedOut`;
		- in `admin_event_submission_settings_test.cpp`, two new: `ThePauseSwitchFlipsOnItsOwn` and `PausingAppliesToTheNextSubmission` (pause by PUT, the next submit is a 503, resume, it goes through);
		- the existing five now check the switch in every response; the 401/403 test also sends a switch-only PUT, and `BadValuesAre400AndWriteNothing` covers the non-booleans.

### 5. client (`ui/src/app/`)

*§5 done 11/3. Client only, so no C++ run. vitest: 63 files, 419 passed (was 53 files, 303). `ng build` local, development and production all succeed; see §7.2.*

- [x] 5.1 Access layer: `core/event-submissions-access/` (interface, HTTP, mock, providers; the `events-admin-access` precedent), with `submit`, `getMine` and `withdraw`. The mock keeps submissions in memory and fakes duplicate hints. **Spec:** URLs and bodies, and the mock's behavior.
	- *Added v0.5:* `getStatus` (the public `GET /api/event_submissions/status`, §4.6). The mock can be paused.
	- *As built:*
		- **Errors.** Every call errors with a `SubmissionProblem` (`status`, `error_code`, `field`, `date_index`, `detail`), read from the server's problem body by `toSubmissionProblem`. Pages never see a raw HTTP error. A reply with no problem body gets a plain "Something went wrong" message.
		- **The offline loop works end to end.** The mock writes into the same in-memory store the `/admin/events` queue reads: an `event_submissions` row, plus pending `events` rows with `origin user_submitted` and their tags. So a form sent offline shows up in the queue, and a decision there shows up on "My submissions."
		- **The mock runs the server's checks with the server's messages,** in the server's order: signed in, the pause switch, the honeypot, the fields, the venue, then both limits. Its duplicate hints come from approved events only, by the same two rules (`core/utils/duplicate-hints.ts`, which the queue uses too).
		- **One settings store for the whole offline app** (`SubmissionSettingsMockStore`). Pausing on the Submission limits page closes the offline form too.
		- **The offline demo now seeds two submissions,** so the queue's grouping and flags show without sending one:
			- Ada's two-date "Queer Craft Night," a regular thing with a note and no link;
			- Grace's "Leather Night!", an hour after the listed Leather Night at the same venue, so it's flagged as a possible duplicate.
		- **`events-admin-access` changed too** (for §5.5 and §5.6):
			- `reviewEvent` takes a decision (`notes`, plus `reason` and `message_to_submitter` for a submitted event);
			- new `reviewSubmission`, `getSubmissionSettings` and `saveSubmissionSettings`;
			- the responses carry `decision_emailed`.
		- **Specs:** 10 for the HTTP side (URLs, bodies, `withCredentials`, the problem mapping, the 503, filling in missing fields), 12 for the mock (the rows it writes, hints, the 503, the honeypot, the field checks, both limits, each date state, withdraw), 6 + 8 for `events-admin-access`, 4 for the duplicate rules.
- [x] 5.2 `pages/events/submit-event.*` (`/events/submit`, ahead of `:id` in `events.routes.ts`).
	- Signed out: a short explainer with sign-in and register buttons that return here.
	- Unverified: "confirm your email" with a resend link.
		- *Note from §3–4 (v0.4):* honuware's sign-in refuses an unconfirmed account, so this state is nearly unreachable. Keep it as a small fallback that also handles the server's 403 `EMAIL_NOT_VERIFIED`, not as a designed screen.
	- **Paused** (*added v0.5*, OQ10): the page checks `getStatus` on load. When submissions are paused, it shows the notice, plus the single-intake line so nobody sends the event another way, instead of the form, whether the visitor is signed in or not. A 503 `SUBMISSIONS_PAUSED` on send (paused while the form was open) shows the same notice and keeps what was typed.
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

	**Specs:** Seattle time to `_us` for several dates (reuse the form-rule tests' DST fixtures); the 12-date cap; each server error code shown next to the right field; signed-out and unverified states; duplicates shown; the honeypot never visible; the paused notice on load, and on a 503 with the typed text kept.

	*As built:*
	- **The form's rules are in `submit-event-form.ts`** (`composeSubmission`), tested on their own. Each date row reuses the event editor's Seattle-time rule: an end time earlier than the start, with no end date, means the next morning.
	- **Additions to the spec:**
		- **An optional "End date, if another day" on each row,** so a festival spanning several days fits. The spec listed only an end time.
		- **The browser asks for a place:** a venue from the list, or "Somewhere else" with text. The server accepts neither, so this is the browser being stricter.
		- **"Check again" on the paused notice** re-reads the status, so nobody has to reload. A status that can't be read counts as open; the send says if it isn't.
	- **Sign-in returns here.** `/events` joined the app's `returnUrlAllowlist` so `/login?returnUrl=/events/submit` is honored.
		- **Register can't return here:** honuware's register page takes no return address. The notice says to confirm the email, then sign in and come back.
	- **Unverified:** a plain message, with no resend link. honuware has no resend-confirmation call, and the state is a fallback anyway (v0.4 note).
	- **A refusal shows in its place.** One with a field and a `date_index` shows under that date row. One with a form field shows under the field. Anything else (a rate limit) shows above Send. `EMAIL_NOT_VERIFIED` switches to the unverified notice, and `SUBMISSIONS_PAUSED` to the paused one, keeping what was typed.
	- **Tags:** once 3 of a kind are picked, the rest of that kind's chips are disabled.
	- **The honeypot** is a "Website" field moved off-screen by the stylesheet, with `aria-hidden`, `tabindex=-1` and `autocomplete=off`. It's sent only when something filled it in.
	- **After sending,** a possible duplicate says which of your dates it matches ("your date 2") when there's more than one. A note appears if some tags weren't available.
	- **Dates and times use honuware's picker** (`hw-simple-date` from `@honuware/ui/controls`; *changed v0.8*, at Mason's request). Each row has a calendar for the start date, a time list for the start time (every 15 minutes, and any time can be typed), then the same for the end time and end date.
		- **The form still holds text** ("2026-10-31", "22:00"), so `composeSubmission` is unchanged. `core/utils/date-picker-values.ts` converts both ways by reading the picker's local date and clock time, so what the picker shows is what's sent, whatever the browser's time zone.
		- **Three things the page handles around the control:**
			- The control announces only readable values, so clearing a box or typing something unreadable wouldn't reach the form. The page also reads the picker's own value as it's typed in.
			- A picker takes its starting value once. Feeding each pick back in would rewrite the box mid-typing. When the form comes back after a pause, the date rows are rebuilt, so the pickers show what was typed.
			- The control's own date-adapter import doesn't reach it inside another page, so the page provides Material's native adapter. It also sets the outline look so the pickers match the other fields.
	- **Specs:** 7 for the form rules (including both 2026 DST nights in one submission), 28 for the page and 5 for the picker conversions. Among the page's: every refusal code in its place; the 503 bringing the form back with the title, description, venue and dates intact; and typing, picking and clearing in the pickers.
- [x] 5.3 `pages/account/my-submissions.*` (`/account/submissions`, signed in): cards per submission with date rows and status chips, the reason and message when not listed, a link when listed, and Withdraw while pending, with a confirm in the page, not a browser pop-up. **Specs:** each status renders; withdraw updates in place.
	- *As built:*
		- **The route** sits behind `AuthGuard`, beside `/account`. The page is in the initial bundle like the other account pages (see §7.2).
		- **Each card** has the title (with "(3 dates)" when there are several), when it was sent, the relationship, "A regular thing," and a row per date with its state chip. A listed date has a "See it" link.
		- **The reason** uses the decision email's words. It shows only when a date was turned down or went past (`not_listed`, `past`), so a withdrawal gets no "Why." The curator's message shows as a quote.
		- **Withdraw** appears while any date is waiting. Its confirm says how many dates it takes out, and that anything listed stays listed. On success, those dates turn to Withdrawn in place. A refusal toasts the server's message.
		- **No submissions** shows a link to the form.
		- **Specs:** 10 (each state, the reason and message, the link, Withdraw only while waiting, confirm and cancel, the in-place update, a refusal, empty, a failed load).
- [x] 5.4 Links: a "Submit an event" button on the events list header and the empty state; the header's account menu gets "My submissions"; the footer gets "Submit an event." **Specs:** the existing header and footer specs updated.
	- *As built:*
		- **The footer also gets a line:** "Know of an event? Submit it here. It's the only way we take listings." That repeats the single-intake message where people look for contact details.
		- **There was no footer spec,** so `footer.spec.ts` is new (3 tests). The header spec gains the menu test. The events list spec checks the button and the empty-state link.
		- **The header and footer are Stream C's files.** The PR should say so.
- [x] 5.5 Queue polish in `pages/admin/manage-events.*`:
	- the origin badge and submitter;
	- the note and relationship;
	- grouping by `submission_id` with Approve all / Reject all;
	- duplicate and no-link flags;
		- *Note from §3–4 (v0.4):* the duplicate flag is worked out in the browser from the queue's rows. The server's hints go only to the submitter and cover approved events only (see "Duplicate hints"). Use the same two rules as `FindDuplicateHints`:
			- same venue, or the same title once it is lower-cased with everything but letters and digits removed;
			- starts within 3 hours, either way.
	- the in-page reject panel (reason, curator note, message to the submitter), replacing `window.prompt`.
		- *Note from §3–4 (v0.4):* use the review responses' `decision_emailed` to confirm the email went, e.g. "Decided; the submitter has been emailed."

	- *Added v0.5 (OQ10):* while submissions are paused, a banner at the top reads "Submissions are paused," with a link to the Submission limits page to turn them back on.

	**Specs:** grouping; approve-all calls the group route once; the reject panel sends all three fields; non-submitted events look as before; the banner shows only while paused.

	*As built:*
	- **Grouping:** the queue stays in start order. A submission's dates sit together in one box, where its first date falls. A curator's or a feed's event stands alone and looks as it did. Approve all / Reject all appear only for a submission with more than one date waiting.
	- **The box's header:** "Submitted by Ada Lovelace," the relationship, "A regular thing," and the note as a quote.
		- **Names** come from the generic editor's people lookup. If that can't be read, it falls back to "member #2."
	- **The duplicate flag** applies to submitted events only. It compares against the rest of the queue and the latest 500 approved events, never the submission's own dates. Each flag links to the event it points at ("Possible duplicate of #103 Leather Night"). "No link" shows when the URL is blank.
	- **The reject panel** opens under the event, or under the submission for Reject all.
		- **The reason** (six choices: Already listed, Not a fit, Missing details, Already over, Spam, Other, or none) and **the message to the submitter** appear only for submitted events. For a curator's own event the panel is just the note, as before.
		- **A blank reason or message is left out of the request.**
	- **The toast** adds "The submitter has been emailed." when `decision_emailed` is true. For example: "Approved all 2 dates of "Queer Craft Night". The submitter has been emailed."
	- **The paused banner** links to the Submission limits page ("Turn them back on").
	- **Other details:**
		- the header also has the "Submission limits" button (§5.6);
		- if the settings can't be read, the banner stays hidden rather than the page failing.
	- **Specs:** the queue's 3 existing tests rewritten for the panel and the seeded submissions, plus 10 new. Among the new: approve-all is one `reviewSubmission` call and no `reviewEvent`; the panel's three fields reach the request and the stored submission; the email note appears only on the last date.
- [x] 5.6 **Submission limits page** (OQ3): `pages/admin/submission-settings.*` at `/admin/events/settings`, registered ahead of any `/admin/events/:id` route, with a "Submission limits" link in the `/admin/events` header.
	- `getSubmissionSettings` and `saveSubmissionSettings` go on `events-admin-access` (interface, HTTP and mock).
	- **The page:**
		- "Dates one person can have waiting for review" and "Submissions one person can send in 24 hours";
		- each a number field with its default and allowed range shown beside it, taken from the GET rather than hard-coded;
		- Save (enabled only when something changed) and "Reset to defaults";
		- a server `LIMIT_OUT_OF_RANGE` shown next to the field it names;
		- *added v0.5 (OQ10):* at the top, a **"Taking submissions / Paused"** switch. It saves on its own, apart from the limits' Save. Pausing asks for an in-page confirmation ("Nobody will be able to submit an event until you turn this back on"); unpausing doesn't. "Reset to defaults" resets the limits only, never the switch.

	**Specs:** loads and shows the values; Save sends only the changed fields; reset fills in the defaults; a server range error lands on the right field; signed-out and non-curator visitors are redirected, the same as `/admin/events`; pausing confirms first and sends only the switch; reset leaves the switch alone.

	*As built:*
	- **Each limit's hint** reads "Default 10 · allowed 1 to 100," from the GET.
	- **Save** sends only the limits that differ from what's saved.
		- **The browser checks the range first,** so a value out of range never leaves the page.
		- **A server `LIMIT_OUT_OF_RANGE`** shows under the field it names, in the page's own words ("Enter a whole number from 1 to 50."), not the server's.
	- **"Reset to defaults"** only fills in the fields. Save applies them, so a slip can be undone.
	- **The switch** saves on its own and sends only `submissions_paused`.
		- **Turning it off** shows the in-page confirm. The switch stays on until "Pause submissions" is pressed.
		- **The save updates only the switch,** so a limit being typed isn't lost.
	- **Access:** the route is under `/admin`, so it has `/admin`'s guards (`AuthGuard` + `AdminGuard`). A signed-out visitor goes to sign in, then back here. A signed-in non-admin goes home. That's the same as `/admin/events`, which is what the spec asks. (The server checks `manage_events`, per OQ9.)
	- **Specs:** 12, including the redirects through the app's real `/admin` route.

### 6. test_helper (`src/test_helper/commands/submission_commands.{h,cpp}`)

*§6.1 done 11/2, together with §3.4 and §4.6 — Windows: 2324 passed. Linux: 2324 passed. 6 of the 12 new tests are here.*

- [x] 6.1 Two commands:
	- `submit-event [se] --as=<email> --title=… --starts_in_hours=N [--dates=3] [--venue_id=N]`, which creates a verified non-admin demo person if the email is new;
	- `list-submissions [ls] [--as=<email>]`.

	**Tests** (`submission_commands_test.cpp`): rows created as that person; the limits enforced through the command too. *The "Demo:" title prefix convention from EV1 §7.4 applies.*

	*As built:* three commands, in a new "Submissions" category.
	- **`submit-event`** goes through `SubmissionHelper::Submit` with the helper's real settings, so the pause switch and both limits apply exactly as on the form.
		- The dates are weekly, the first N hours from now. "Demo: " is added to the title unless it's already there.
		- A new email becomes a verified member named "Demo Submitter," with no roles. Its random password is printed once, so you can sign in as them and see "My submissions."
		- A refusal prints the error code, the field, the date, and the message, and exits 1.
	- **`list-submissions`** shows each submission and each date's state. Without `--as`, it shows the logged-in person's.
	- **New: `submission-settings [ss] [--paused=true|false] [--max_pending_dates=N] [--max_submissions_per_day=N]`.** With no flags, it shows the settings. With flags, it saves them through `SaveSubmissionSettings`, with the same checks as the page. It's the command-line way to pause submissions during a spam wave, and the tests use it to prove the limits reach `submit-event`.
	- **One-shot mode:** `main.cpp` gains `--as`, `--dates`, `--paused`, `--max_pending_dates` and `--max_submissions_per_day`.
	- **Tests** (6 in `submission_commands_test.cpp`):
		- the submitter created verified, with the dates pending, weekly, titled "Demo: …" and made by them;
		- a second submit reusing the person, with no double prefix;
		- the command's own flag checks and the form's checks (too soon, 13 dates, unknown venue) refusing with nothing written;
		- a lowered pending limit and the pause switch, both set with `submission-settings`, holding `submit-event`;
		- `submission-settings` showing the values, and refusing a bad value without saving the good one beside it;
		- `list-submissions` showing "listed" and "waiting," the same with `--as` or logged in.

### 7. Gates

- [ ] 7.1 Docker suite green per slice, Windows too ("Windows: N passed. Linux: M passed."); raise the floor at bucket exit (EV1 §7.2 policy).
- [x] 7.2 `ng build` for local, development and production; vitest green, with the new count recorded.
	- *Done 11/3:*
		- **vitest:** 63 files, 419 passed (was 53 and 303; +116 tests in 10 new spec files and 7 extended ones).
		- **Builds:** local, development and production all build.
		- **Production budget warning:** it's the old one; the initial bundle has been over its 500 kB budget since before EV2 (OQ10 there). RT4's pre-launch pass owns it.
		- **Initial bundle:** 882.65 kB at `c30000b`, built from a copy of that commit, and 911.77 kB now. That's +29.1 kB, or +6.6 kB compressed. Almost all of it is in `main`:
			- the "My submissions" page (eager, like the other account pages);
			- the submissions access layer, with its offline mock;
			- the larger review-access layer.
			- As with EV2, the mock is demo code that production carries. Making "My submissions" load lazily would save a few kB; that's left for RT4's pass.
		- **The submit form loads separately.** The lazy `events-routes` file grew from 43 kB to 152 kB, mostly the Material form controls (autocomplete, chips, radio, select).
	- **A test-runner quirk, and the rule it sets.** The unit-test runner rewrites imports in its own way (Vite's SSR transform). It leaves an imported constant unrewritten when it's a class field's whole initializer, as in `readonly maxDates = MAX_SUBMISSION_DATES`.
		- **The symptom:** when the spec files are bundled together and the constant lands in another chunk, the field reads `undefined`. Here, the 12-date cap's button never disabled in the full run, though it passed alone.
		- **The app build is unaffected.** This was checked in the production bundle.
		- **The fix:** the three such fields (`maxDates`, `relationships`, `rejectReasons`) are now getters.
		- **The rule for client code:** expose an imported constant to a template through a getter, not a field. A constant from the component's own file is fine as a field.
- [ ] 7.3 Live seed check after §1 (destructive, announced): `event_submissions` present with its CHECKs and index, `events.submission_id` FK and `events_submission_idx` present, twelve tables mapped to `manage_events`, nothing new in `allowed_tables`. The rate-limit keys and the pause key need no seed check: a database without them reads the defaults (§3.3, §3.4).
- [ ] 7.4 **Manual loop (Mason or Levi):**
	1. Register a fresh account and confirm its email. The form refuses it before confirmation.
	2. Submit a three-date event.
	3. As an admin, open `/admin/events`: the three dates show grouped, with the badge.
	4. Approve two, then reject one with a message. One email arrives, listing all three outcomes.
	5. "My submissions" shows Listed, Listed, Not listed with the message.
	6. Submit again and withdraw it.
	7. Push past the pending limit and read the 429 message.
	8. On the Submission limits page, lower the pending limit to 2. The next submission that would go over 2 gets a 429 that says 2. Reset to defaults.
	9. Pause submissions (confirm the prompt). `/admin/events` shows the banner, and `/events/submit` shows the notice instead of the form, signed in and signed out. "My submissions" and withdraw still work. Turn submissions back on from the banner's link, and the form returns.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Must people sign in to submit an event?** Signing in means registering and confirming an email first. That's a small hurdle for an organizer, but it gives us a real email to reply to, a way to stop a repeat spammer (disable the account), and a "My submissions" page with no extra machinery. The alternative is a no-account form that emails a confirmation link to whatever address is typed. That is friendlier, but it needs its own confirmation tokens, its own spam defenses, and a way to show status without an account. *Rec: signed in with a confirmed email at launch; revisit if the submission numbers say the hurdle is costing us.*
	- Mason- They need to sign in to submit events.
	- Claude- Decided (11/2) — signed in with a confirmed email, as pinned ("Who can submit"); no body change.
2. **Up to 12 dates, no "repeats forever."** A weekly event is listed by adding its next few dates and ticking "This is a regular thing," and a curator then decides whether it deserves a series page. The alternative is letting submitters set up a rule ("every 2nd Saturday"), but EV1 deliberately has no repeat rules (series are labels, not generators). *Rec: 12 dates, with series made by curators.*
	- Mason- works for me
	- Claude- Decided (11/2) — 1–12 dates, with series made by curators, as pinned; no body change.
3. **Rate limits: 10 dates waiting for review per person, and 5 submissions a day.** Enough for a busy organizer, small enough that one bad account can't flood the queue. *Rec: as stated; both are constants and easy to change.*
	- Mason- Let's make this configurable in the admin console with bespoke UI.
	- Claude- Decided (11/2) — configurable, with a page of its own. 10 and 5 stay as the defaults.
		- **Storage:** two app keys in `config_secrets`, honuware's store for site settings. It's out of the generic editor's reach, so only the new endpoint can change it. This is the Site Theme editor's pattern, and it needs no new table or migration.
		- **Ranges:** 1–100 and 1–50. A bad stored value reads as the default.
		- **The page:** "Submission limits," at `/admin/events/settings`.
		- **Where it's in the doc:** the "Rate limits" pin-down, new items §3.3, §4.5 and §5.6, and step 8 of the manual loop.
		- Two follow-ups are new questions: OQ9 (who can change the limits) and OQ10 (a pause switch).
4. **One email per submission, when it's fully decided.** It lists each date's outcome and the curator's message. Per-date emails would spam a twelve-date organizer. No email at all would leave submitters guessing, which is how SGN's mail-and-fax calendar feels. *Rec: one email per submission; withdrawals send none.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (11/2) — one email per submission, sent when it's fully decided; withdrawals send none. As pinned; no body change.
5. **No edits after submitting.** To fix a typo, a submitter withdraws and resubmits while it's still waiting, or writes to us once it's listed. Letting submitters edit approved events would reopen review every time. Letting them edit pending ones is possible, but it adds a form state and a race with a curator mid-review. *Rec: no edits at launch. CM1's correction flow covers listed events later.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (11/2) — no edits after submitting; withdraw and resubmit instead. As pinned; no body change.
6. **Does submission need the terms of service first?** Submissions are user-generated content, so RT2's terms should say how we use what's submitted. *Rec: build it now. The public launch (M3) already requires RT2, so the terms exist before strangers can reach the form. No feature switch is needed.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (11/2) — build it now. RT2's terms gate the public launch, not this bucket. No body change.
7. **Photos from submitters: not yet.** A flyer image would make listings nicer, but photos raise the consent and no-outing questions CM3 exists to answer (a party photo can out someone), and they invite storage abuse. *Rec: no submitter photos in EV3; a curator can add the flyer on review through the existing photo pipeline.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (11/2) — no submitter photos; a curator adds the flyer on review. As scoped; no body change.
8. **Is a link required?** A link (ticket page, Instagram post, venue page) is the fastest way for a curator to check that an event is real. But some grassroots events have none. *Rec: optional, with the queue flagging "no link" so the curator knows to look harder.*
	- Mason- I'll go with your recommendation.
	- Claude- Decided (11/2) — optional, with the "no link" flag in the queue. As pinned; no body change.
9. **Who can change the submission limits?** Anyone with `manage_events` (the curators reviewing the queue), or full admins only? The curators are the people who feel a flood first and who know when a big weekend needs more room. A full admin can already do everything a curator can. The risk of letting curators change them is small: the ranges are capped (100 dates, 50 submissions a day), and a curator can already approve or reject anything. *Rec: anyone with `manage_events`.*
	- Claude- Built as recommended (11/2) so §4.5 wasn't blocked: both settings routes check `manage_events`. If you'd rather limit it to full admins, it's a one-line change in each route plus its test. Please confirm.
	- Mason- manage events seems like the better choice.
	- Claude- Decided (11/2) — anyone with `manage_events`, as built in §4.5. The pause switch (OQ10) uses the same gate. No code change.
10. **Should the limits page also have a "pause submissions" switch?** If a spam wave hits from many new accounts at once, the per-person limits don't help, and today the only fix is disabling accounts one by one. A switch would close the form for everyone with a "we're not taking submissions right now" notice. It's about an hour's work on the same page, but it's one more thing that can be left on by accident. *Rec: not at launch. Add it the first time a wave happens, since the page and endpoint will already exist. The minimum of 1 means the limits themselves can't be used to close the form.*
	- Claude- Built as recommended (11/2): there's no pause switch. Please confirm.
	- Mason- Let's add this in the admin console. This seems like if we need it, we would really need to have it.
	- Claude- Decided (11/2) — build it now, on the Submission limits page. The full design is the "Pause switch" pin-down. In short:
		- a third app key, where only `"true"` pauses;
		- a paused submit is a 503 `SUBMISSIONS_PAUSED`;
		- a public status read lets the form show the notice up front;
		- "My submissions," withdraw and review keep working;
		- a banner on `/admin/events` while paused, so it isn't left on.
		- **Where it's in the doc:** the Scope line, new items §3.4 and §4.6, additions to §5.1, §5.2, §5.5 and §5.6, §7.3, and step 9 of the manual loop.

# Change Log

- **11/3/2026 (v0.8)** — **The submit form's dates and times use honuware's picker** (`hw-simple-date`), at Mason's request, in place of the browser's own date and time boxes. The details are in §5.2's as-built notes.
	- vitest: 64 files, 427 passed (was 63 and 419). `ng build` local, development and production all succeed; the initial bundle is 909.96 kB. No C++ changed.
	- The curator's event editor (`/admin/events`, EV2) still uses the browser's boxes. Switching it is a follow-up, if Mason wants it.
- **11/3/2026 (v0.7)** — **§5 (client) implemented, and §7.2 done.**
	- **Results:**
		- vitest: 63 files, 419 passed (was 53 and 303).
		- `ng build` local, development and production all succeed.
		- The production budget warning predates EV2.
		- The initial bundle is +29.1 kB (+6.6 kB compressed).
	- No C++ changed, so there was no Windows or Linux run.
	- The as-built notes are under each item. The decisions worth knowing:
	- **The offline demo runs the whole loop.**
		- The submissions mock writes into the store the review queue reads, so an offline submission can be reviewed and then seen on "My submissions."
		- The demo seeds two sample submissions, so the grouping and flags show.
		- One settings store links the limits page, the pause switch and the form.
	- **Sign-in returns to the form;** `/events` joined the return-address allowlist. Register can't return there, and there's no resend link, because honuware has neither.
	- **The form is a little stricter and roomier than specified:**
		- it asks for a place;
		- each date row can end on another day;
		- the paused notice has "Check again."
	- **The queue's duplicate flag** checks the rest of the queue and the latest 500 approved events.
	- **The footer gained a single-intake line.** The header and footer are Stream C's files, so the PR should say so.
	- **A test-runner quirk sets a client rule:** expose an imported constant to a template through a getter, not a class field (§7.2).
	- **Still to do:**
		- 7.1's floor at bucket exit;
		- 7.3's live seed check (destructive, needs Mason's go-ahead);
		- 7.4's manual loop.
- **11/2/2026 (v0.6)** — **The pause switch (§3.4, §4.6) and the test_helper commands (§6.1) implemented.** Windows: 2324 passed. Linux: 2324 passed (12 new tests). The as-built notes are under each item. The decisions worth knowing:
	- **Renamed:** the §3.3 limits types and functions became "settings" (`SubmissionSettings`, `LoadSubmissionSettings`, `SaveSubmissionSettings`, …), since they now carry the switch too.
	- **The switch is checked before the honeypot and every field,** so a paused form always gets the notice.
	- **A third command, `submission-settings [ss]`:** it shows or changes the limits and the switch from the command line. It's the way to pause without the page (§5.6 isn't built yet), and the tests use it to prove the limits reach `submit-event`.
	- **`submit-event` prints a new demo submitter's random password once,** so you can sign in as them and see "My submissions."
- **11/2/2026 (v0.5)** — Mason answered OQ9 and OQ10.
	- **OQ9:** anyone with `manage_events` can change the submission settings, as built. No code change.
	- **OQ10:** the pause switch is now in scope; Mason decided against the recommendation to wait. It adds the "Pause switch" pin-down, the Scope line, new items §3.4 and §4.6, additions to §5.1, §5.2, §5.5 and §5.6, §7.3, and step 9 of the manual loop.
- **11/2/2026 (v0.4)** — **§3 (business_logic) and §4 (endpoints) implemented.** Windows: 2312 passed. Linux: 2312 passed (64 new tests). The as-built notes are under each item and in the pin-downs. The decisions worth knowing:
	- **The curator's side lives in `SubmissionHelper`** (`ReviewDate`, `ReviewSubmission`, `SendDecisionNotice`), so the review endpoints stay thin.
	- **A row lock makes the email reliable when two curators work at once.** Every path that decides dates first locks the submission (`GetSubmissionForUpdate`, added to §2.1). That way, two curators finishing a submission at the same moment can't both miss the email.
	- **Duplicate hints:** the ones sent back to a submitter are approved events only. The queue works out its own flag in the browser (§5.5).
	- **Response details:**
		- ignored slugs are reported per kind;
		- errors carry `field` and `date_index`;
		- review responses carry `decision_emailed`.
	- **New codes:** `NOT_AGREED`, `NO_DATES`, `BAD_RELATIONSHIP`, `TOO_MANY_TAGS`, `SUBMISSION_NOT_FOUND`, `BAD_REASON`.
	- **Withdraw** keeps a curator's reason, and stamps the submission so no email follows.
	- **The failed-send case is tested at the business layer.** honuware's endpoint test harness can't make a send fail.
	- **Signed in but unverified** is nearly unreachable, because honuware's sign-in refuses unconfirmed accounts. §5.2's state shrinks to a fallback.
	- **OQ9 and OQ10** were built as recommended (curators can change the limits; no pause switch), pending Mason's confirmation.
- **11/2/2026 (v0.3)** — **§2 (table_helpers) implemented.** Two refinements made while building it:
	- `CountPendingForCreator` became `CountPendingSubmittedBy`. It counts through the submission, so an admin's other pending events don't count against them.
	- `MarkDecisionNotified` became a claim that returns whether it stamped. The decision email now goes out exactly once even when two curators finish a submission at the same moment. The "Mail" pin-down says how a failed send releases the claim.
- **11/2/2026 (v0.2)** — Mason answered OQ1–OQ8, and all but OQ3 went with the recommendation.
	- **OQ3:** the rate limits become admin-configurable on a "Submission limits" page. This adds the Scope line, the rewritten "Rate limits" pin-down, new items §3.3, §4.5 and §5.6, and step 8 of the manual loop. It also raises two new questions, OQ9 and OQ10.
	- **Two §1 corrections found while implementing it:**
		- `event_submissions` is top-level, not nested under `events`, because the FK runs the other way (1.3);
		- `reject_reason` also allows `expired`, for EV4's expiry sweep (1.1).
	- **§1 (db_schema) implemented.**
- **11/1/2026 (v0.1)** — Initial draft from the outline's EV3 blurb, the brainstorm's intake stack, EV1's reserved `user_submitted` origin and review endpoint, and EV2 §3.7's queue (including its OQ9 (j) hand-off of the reject pop-up). Eight open questions.
