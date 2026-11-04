---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Levi Kuhn (owner) · Mason Bendixen
Last Updated: 11/3/2026
Version: 0.2
tags:
---

# Bucket EV4 — Scheduled jobs

**Track:** EV — Events engine · **Size:** S · **Stream:** A · **Owner:** Levi
**Needs:**
- EV1 (done);
- honuware's scheduler engine, `honuware_scheduler` (done: `components/scheduler/` exists at the pinned SHA, so ledger item H9 has landed);
- **one small honuware change, done first** (§1): the service-account setup must stop requiring knottyyoga's permission. That is a cross-repo bump-point.

**Feeds:**
- **EV6:** feed imports run as a scheduled job.
- **EV5:** the scan and verify jobs.
- **EV3:** the expiry sweep closes out unreviewed submissions.
- **PL3:** the admin-alerts digest, its "ops eye."
- **RT3:** a later newsletter job.

# Context

Implements [[Initial Project Implementation Outline]] Stream A bucket EV4. It **supersedes [[Setting up the project]] Phase 12** under the supersession rule: `communityfinder_helper` on `honuware_scheduler`, archiving past events, the hygiene jobs mirrored from knottyyoga (SUTP Q12, decided), the admin-alerts digest, and the scheduler service account.

**How the engine works** (read `components/scheduler/scheduler/{scheduled_job,api_client,job_scheduler,scheduler}.h` at the pinned SHA):
- A job is an authenticated HTTP call on an interval. It is a `POST` with an **empty body** to `/api/admin/<name>`.
- The engine logs in once per target as a service account, echoes the CSRF cookie, and logs in again on a 401.
- It knows nothing about what a job does, so **every job endpoint must be idempotent and self-gating**: safe to call twice, deciding for itself whether there's work.
- knottyyoga's `src/scheduler/knottyyoga_job_catalog.{h,cpp}` + `main.cpp` are the model. CF's catalog is that file with CF's jobs.

**What blocks the obvious path.** `database_helper/create_database.cpp` has a comment (lines 205–213) explaining why CF never seeded its service account: honuware's `Auth::EnsureSchedulerServiceAccount` hard-requires the `manage_subscriptions` permission and throws without it. That permission is knottyyoga's billing permission, which CF doesn't have. §1 fixes that in honuware.

**Read before starting:**
- knottyyoga's catalog and its test (`knottyyoga_job_catalog_test.cpp`);
- knottyyoga's `src/endpoints/admin_cleanup_*.cpp` (four thin endpoints over platform helpers) and `admin_notify_admin_alerts.cpp` (the digest, with its `admin_alerts_last_notified_us` stamp-forward);
- honuware `business_logic/auth/service_account.h`, `token_cleanup_helper.h`, `sql_util/table_helpers/{login_attempts,idempotency_keys,admin_alerts}.h`, and `business_logic/images/image_helper.h` (`DeleteScaledPhotosOlderThan`);
- CF's `business_logic/app_service_account_config.h` (`scheduler@communityfinder.local`).

# Scope

**In:**
- the honuware service-account change and re-pin;
- the `run_scheduled_jobs` permission and the seeded service account;
- the job endpoints: archive, expire, four hygiene jobs, and the admin-alerts digest;
- the visibility change that keeps past events readable (§3);
- the job catalog and `communityfinder_helper`;
- test_helper commands and tests.

**Out (explicit non-goals):**
- the feed-import job's endpoint (**EV6**; the catalog reserves its slot, disabled);
- the scan and verify jobs (**EV5**, same);
- the newsletter (**RT3**);
- multi-community fan-out (**PL4**; the catalog is written so it's a loop away, as knottyyoga's `BuildMultiTenant…` shows);
- deployment of the helper process (**PL2**).

# Design pin-downs

- **The permission:** a new app permission **`run_scheduled_jobs`** (not pricing-eligible). It is granted to the scheduler role and to Administrator, so an admin can run a job by hand with curl or the test_helper. Every job endpoint calls `RequirePermission(run_scheduled_jobs)` before anything else. The scheduler role gets **only** this permission. The endpoints do their own work, so the account can't edit tables through the CRUD editor.
- **The service account:** `scheduler@communityfinder.local` (already in `app_service_account_config.h`). Its password comes from `SCHEDULER_SERVICE_ACCOUNT_PASSWORD`. `PopulateTables` provisions it through the parameterized `EnsureSchedulerServiceAccount` (§1) with `{"run_scheduled_jobs"}`. When the variable is unset, the seed **skips the account with a printed warning** rather than failing. A developer who never runs the helper shouldn't have `--recreate_database` break (OQ2). The framework already keeps service accounts out of the user lists by email domain.
- **Jobs, endpoints and default intervals** (seconds; `0` disables, as in knottyyoga):

	| Job | Endpoint | Default | What it does |
	|---|---|---|---|
	| `archive_past_events` | `/api/admin/archive_past_events` | 3600 | Approved events that **ended** more than 6 hours ago → `archived`. "Ended" is `ends_at_us`, or `starts_at_us` + 2 hours when there's no end, matching EV2's ICS default. |
	| `expire_unreviewed_events` | `/api/admin/expire_unreviewed_events` | 3600 | Pending events whose **start** has passed → `rejected`, with `review_notes` "Date passed before review," no reviewer, and `reviewed_us` = now. This closes EV3 submissions that nobody reviewed in time. |
	| `cleanup_expired_tokens` | `/api/admin/cleanup_expired_tokens` | 86400 | `Auth::TokenCleanupHelper::CleanupExpired` |
	| `cleanup_login_attempts` | `/api/admin/cleanup_login_attempts` | 86400 | `LoginAttempts::PurgeOlderThan`, with the retention from secrets (knottyyoga: 30 days) |
	| `cleanup_scaled_photos` | `/api/admin/cleanup_scaled_photos` | 86400 | `ImageHelper::DeleteScaledPhotosOlderThan` (scaled copies regenerate on demand) |
	| `cleanup_idempotency_keys` | `/api/admin/cleanup_idempotency_keys` | 86400 | `IdempotencyKeys::DeleteExpiredIdempotencyKeys`, mirrored per SUTP Q12 even though CF barely uses keys yet |
	| `notify_admin_alerts` | `/api/admin/notify_admin_alerts` | 3600 | Mails new `admin_alerts` rows to `admin_alerts_recipient` and stamps `admin_alerts_last_notified_us` forward. An empty recipient makes it a no-op. |
	| `import_event_feeds` | *(EV6)* | **0** | reserved. [[Bucket EV6 — Structured feed import]] §6 adds `/api/admin/import_event_feeds` and sets it to hourly (each tick imports what's due within a 60-second budget) |
	| `cleanup_scan_runs` | *(EV6)* | *(added by EV6)* | daily once EV6's `scan_runs` table exists: deletes runs older than 180 days |
	| `run_event_scan` | *(EV5)* | **0** | reserved, as SUTP Phase 12 specified |

- **Two separate sweeps rather than one "archive" job.** They mean different things. An approved past event *happened*; it's history and should stay readable. A pending past event *never got decided*; it was never public and must not become public. Keeping them apart is what lets §3's visibility rule be simple: **archived events only ever come from approved ones.**
- **Expired submissions get their email.** When `expire_unreviewed_events` closes the last pending date of an EV3 submission, it calls EV3's `OnDateReviewed` and sends the decision email with a "we didn't get to this in time, sorry" line. If EV3 hasn't landed, the hook is a no-op stub that EV3 fills in. Whichever bucket lands second wires it.
	- **The reason it records** *(added 11/2, EV3 v0.2)*: EV3's `event_submissions.reject_reason` allows `expired` for this sweep. When the sweep rejects a submitted date and the submission has no reason yet, it sets `expired`, so "My submissions" can say "the date passed before we reviewed it." A reason a curator already set stays as it is.
- **Archived means "past and settled," not "hidden" (OQ3).** Today, public reads mean `status = 'approved'`, so archiving an event would empty the past days of the month calendar and turn every old event's link into a 404. That breaks shared links and search results. The rule becomes **public = `approved` or `archived`**:
	- **The upcoming list and the ICS feeds don't change.** Their windows start at "now" (the feed at now − 1 day), so they still show what's coming.
	- **The calendar's past days keep their events.**
	- **An old event's page still loads**, with a "This event has passed" note. That note comes from the time, not the status, so it's right even before the job runs.

	What `archived` still buys:
	- a re-scan can't reopen it (`archived_kept`, EV1 OQ10 (d));
	- the review queue and the editor's default lists can leave it out;
	- EV5's verifier skips it.
- **Every job endpoint returns counts** (`{"archived": 12}`, `{"device_tokens_deleted": 3, …}`), so the helper's log, the test_helper and curl show what happened. Each runs in one transaction.
- **Batch limits:** archive and expire each handle at most 500 rows per call, oldest first. An hourly interval drains any backlog, and a first run on a large database can't hold a long transaction.

# Layered work items

*(Numbered; lower layers first; every item names its tests.)*

### 1. honuware: parameterize the service account (cross-repo bump-point)

- [x] 1.1 In `components/platform/business_logic/auth/service_account.{h,cpp}`, `EnsureSchedulerServiceAccount` gains a `const std::vector<std::string_view>& permissionNames` parameter. It links the scheduler role to each named permission and throws if one is missing, as it does today for the hard-coded one. **Keep the old signature as a forwarder** that passes `{"manage_subscriptions"}`, with a comment marking it for removal, so knottyyoga compiles unchanged until its batched re-pin (OQ1). **Tests** (honuware): one permission; two; a missing one throws; the forwarder behaves exactly as before.
	- *As built (11/3):*
		- **The old function did a bit more than this doc said.** It also granted `manage_class_schedule` when that permission existed. The forwarder keeps doing exactly that, and keeps knottyyoga's original role description.
		- **The new form** describes the role generically and checks every name **before writing anything**, so a missing permission throws with no half-made account. It also reuses an existing `scheduler` role without duplicating its links.
		- **Tests:** five new ones in `service_account_test.cpp` (one permission, two, a missing one throws and writes nothing, idempotent, reuses an existing role), plus `ForwarderKeepsTheLegacyRoleDescription`. The old tests still pass, so the forwarder behaves as before.
- [ ] 1.2 The co-dev loop: CF builds against the local honuware tree (`HONUWARE_SRC_DIR`, the `x64-Debug-local` preset on Windows). Mason pushes honuware, then CF re-pins (`GIT_TAG`) and Claude verifies a strict pinned Linux gate (fresh clone, no override). knottyyoga's re-pin stays batched and isn't required here.
	- *Status (11/3):* the local half is done; CF builds and tests against the local tree on both platforms (§8.1). **Waiting on Mason's honuware push.** Then Claude re-pins `GIT_TAG` and runs the strict pinned gate.
- [x] 1.3 Ledger: add the change as **H13** in [[Setting up the project]]'s cross-repo ledger, plus a follow-up to remove the forwarder once knottyyoga moves to the new signature. *Added 11/3, with OQ6's extraction note as a second follow-up.*
- [x] 1.4 **Bundle with EV6's H14 if both are near.** [[Bucket EV6 — Structured feed import]] §1 proposes a second small, additive honuware change: fetch options on `HttpRequest` (timeout, size cap, private-address check). Making both in one honuware push saves a bump cycle. Neither needs knottyyoga to move. *Not bundled (11/3): EV6 hasn't started, so H13 goes alone rather than waiting.*

### 2. Permission, account and seed (`business_logic/create_app_tables.cpp`, `database_helper/create_database.cpp`)

- [x] 2.1 `PopulateAppPermissions` gains `run_scheduled_jobs`, granted to Administrator by name. The constant `kPermissionRunScheduledJobs` lives in a new `db_schema/scheduled_jobs.h`. It gates endpoints, not a table, so it has no `admin_table_permissions` rows. **Test:** `create_app_tables_test.cpp`, where the permission exists and Administrator holds it.
- [x] 2.2 `create_database.cpp`: replace the "DEFERRED to Phase 12" comment with the provisioning call. If `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` is set, call `EnsureSchedulerServiceAccount(..., App::kSchedulerServiceAccountEmail, password, {kPermissionRunScheduledJobs})`. If not, print "Scheduler service account skipped: set SCHEDULER_SERVICE_ACCOUNT_PASSWORD to create it." The call goes after `PopulateAppTables`, since the permission must exist. **Test:** the provisioning moves into a reachable function (`ProvisionSchedulerAccount` in `create_app_tables.cpp`, the EV1 §1.6 lesson that the exe's anonymous namespace is untestable) with tests for set (account plus role plus the one permission), unset (skipped, nothing written) and re-run (idempotent).
	- *As built (11/3):* `ProvisionSchedulerAccount(tx, db, password)` returns false for an empty password, and the seed prints the skip line. honuware's example JSON says "seeding throws if unset." That is knottyyoga's behavior, not CF's (OQ2).
	- **Mason's dev database** gains the permission and the account only on its next `--recreate_database`. No migration was written, since nothing is deployed yet.

### 3. Past events stay readable (EV1 and EV2 code, Stream A)

- [x] 3.1 `EventHelper`: public reads accept `approved` or `archived`. That covers `GetUpcomingPublic`'s range query, `GetEventDetail` for anonymous callers, and `GetSeriesDetail`'s instance list. `TableHelpers::Events::GetEventsInRange` takes a status list instead of one status. **Tests:**
	- an archived event appears in a past-dated window and in its detail;
	- it never appears in the default upcoming window once it's in the past;
	- pending and rejected stay hidden.
	- *As built (11/3):*
		- The rule lives in one place: `Events::IsPublicEventStatus` / `PublicEventStatuses()`. The detail endpoint and the single-event `.ics` both use it.
		- **One rename.** `GetEventsInRange` was dropped rather than widened. EV3 had already added `GetEventsStartingBetween(…, statuses)`, the same query with a status list, so the calendar now uses that.
- [x] 3.2 ICS (`get_events_ics_feed.cpp`): the now − 1 day window now also carries yesterday's archived events, so a subscriber's calendar doesn't lose the event they went to last night. **Test.** *As built: no change to the endpoint itself. It reads through `GetUpcomingPublic`, so 3.1 covers it; the new test is in `get_events_ics_feed_test.cpp`.*
- [x] 3.3 Client: `EventDetailPage` shows "This event has passed" when its end (or start + 2 hours) is in the past, and hides add-to-calendar. **Spec.** The calendar needs no change; it already shows whatever the month window returns. *As built: the event types and the mock community-access layer also accept `archived`, so the mock-data build behaves like the server.*

### 4. business_logic (`src/business_logic/jobs/`, new; house `PRIVATE`/`PUBLIC` CMake split)

- [x] 4.1 `event_lifecycle_jobs.{h,cpp}`:
	- `ArchivePastEvents(Transaction&, nowUs, limit) → {archived}`;
	- `ExpireUnreviewedEvents(Transaction&, nowUs, limit) → {expired, submissionNotices}`, where the notices go to EV3's mailer.

	The SQL lives in two new `TableHelpers::Events` methods, `GetApprovedEndedBefore(cutoffUs, limit)` and `GetPendingStartedBefore(nowUs, limit)`. Updates go through `UpdateEvent`, so `updated_us` bumps. **Tests:**
	- the 6-hour grace (5h59m stays, 6h01m goes);
	- the no-end-time rule;
	- the batch limit;
	- archived, rejected and future events untouched;
	- an expired event's note and empty reviewer;
	- re-running is a no-op;
	- an ingest of an archived key reports `archived_kept`, as EV1 OQ10 (d) promised, now with a real archive in the loop.
	- *As built (11/3):*
		- **Each event is re-read under its row lock before it changes** (`GetEventForUpdate`), so a curator's decision made after the sweep's scan wins. For a submitted date, the submission is locked first, the same order EV3 uses.
		- `GetApprovedEndedBefore` also requires the start to be before the cutoff, so a bad end time (before the start) can't archive a future event.
		- **The email.** The expiry sweep returns the claimed decision notices, and the caller sends each with EV3's `SubmissionHelper::SendDecisionNotice`. A failed send records an admin alert, which the digest mails, and never fails the job.
		- A curator's existing reject reason is kept; only an empty one becomes `expired`.
		- **Tests:** 10 in `event_lifecycle_jobs_test.cpp`, covering the bullets above plus "one of two dates expiring sends no email yet."
- [x] 4.2 `admin_alerts_digest.{h,cpp}`: the knottyyoga digest logic, moved out of the endpoint so it can be tested: render the lines, find the max `created_us`, read and stamp the secret. The subject defaults to "{site name} admin alerts" when `admin_alerts_digest_subject` is empty. **Tests:** no new alerts means no mail; new alerts produce one mail and a stamp; a second run sends nothing; an empty recipient is a no-op. *As built: also tested are that a configured subject wins and that a failed send leaves the stamp, so the next run retries the same alerts.*
- [x] 4.3 *(added while building)* `scheduled_jobs.{h,cpp}`: **one entry point for all seven jobs.** `Jobs::RunScheduledJob(tx, db, secrets, mail, name)` runs a job by name and returns its counts as JSON, or nothing for an unknown name. The endpoints (§5) and the test_helper's `run-job` (§7) both call it, so curl, the helper process and the test_helper can't drift apart.
	- `LookupRetentionUs` reads a retention secret. When the secret is missing, malformed or not positive, it uses 30 days, knottyyoga's default. It reads `auth_login_attempts_retention_in_micros` and `scaled_photo_max_age_us`.
	- **Tests:** every job runs on an empty database and reports exactly its count keys; an unknown name gives nothing; the retention fallbacks.

### 5. endpoints (`src/endpoints/`, EV1 §5 conventions)

- [x] 5.1 Seven thin TUs, one per job (`admin_archive_past_events`, `admin_expire_unreviewed_events`, the four `admin_cleanup_*`, `admin_notify_admin_alerts`). Each checks `run_scheduled_jobs`, runs one transaction, and returns its counts. **Tests** per endpoint:
	- 401 signed out; 403 without the permission (grant it to another user first, the EV1 OQ11 pattern);
	- 200 with counts for the service-account-shaped caller;
	- calling twice is safe.
	- *As built (11/3):*
		- **Shared code.** The seven TUs share `scheduled_job_endpoint.{h,cpp}`: `PostScheduledJob` runs the permission check and `Jobs::RunScheduledJob` in one transaction. Each TU is its route plus one line.
		- **The counts are logged server-side.** honuware's engine logs only `job_success job=… status=200 duration_ms=…`, not the response body, so a 200 also writes `[scheduled_job] event=job_ran path=/api/admin/<job> result={…}` to the **server's** log. That is where §8.4's "a 200 with counts" shows up.
		- **Tests.** The shared 401/403 checks loop over all seven jobs in `scheduled_job_endpoint_test.cpp`, which also covers a 200 for every job on an empty database and an Administrator-style holder running one by hand. Each endpoint's own test file proves its effect and that a repeat finds nothing.
		- **The caller in tests** is the real scheduler account: `JobsEndpointTest::SignInAsScheduler` provisions it through `ProvisionSchedulerAccount`.
- [x] 5.2 Seven volatile anchors in `web_app.cpp` (own lines, alphabetized) and the CMakeLists sources. *A missing anchor 404s in Release only.* The floor raise in §8 is the alarm.

### 6. The helper process (`src/scheduler/`, new)

- [x] 6.1 `communityfinder_job_catalog.{h,cpp}`:
	- `JobTarget` and `JobIntervals` (one field per table row);
	- `BuildCommunityFinderJobs(target, intervals)`, which skips zero intervals;
	- `ValidateCommunityFinderConfig`;
	- `ResolveServiceAccountPassword`.

	knottyyoga's file, with CF's jobs. **Test** (`communityfinder_job_catalog_test.cpp`): every enabled job present with path and interval; zero intervals dropped; the two reserved jobs absent at their default of 0; validation messages for an empty URL, email or password and negative intervals.
- [x] 6.2 `main.cpp`: ABSL flags `server_url` (default `http://localhost:18081`), `service_account_email` (default `scheduler@communityfinder.local`), `service_account_password` (falls back to the env var), and one `*_interval` flag per job. It also adds the `X-Origin-Secret` header when `HONUWARE_ORIGIN_SECRET` is set (knottyyoga's `MaybeAddOriginSecretHeader`, needed once PL2 puts CloudFront in front). Fixed mode only; control mode is PL4.
	- *As built (11/3):*
		- **The flags:** `import_event_feeds_interval` and `run_event_scan_interval` exist, default 0, with "reserved, leave at 0" help. Nothing stops a non-zero value, but those endpoints don't exist yet, so the log would show a 404.
		- **The header** reads `HONUWARE_ORIGIN_SECRET`, falling back to `KNOTTYYOGA_ORIGIN_SECRET`, the same lookup as the server's origin guard.
		- **The startup log** never prints the password. A failed first login, refused or unreachable, gives the reason and asks whether the server is running and whether the database was seeded with the same `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` (see §8.4).
- [x] 6.3 CMake: the `communityfinder_helper` target links `communityfinder_core` and `honuware_scheduler`. Update the app-superset `cmake/honuware_layering.cmake` so the helper target may link the scheduler side branch. Add a VS launch entry and a `launch_defaults.local.json` example (the password goes in the git-ignored local file only).
	- *As built (11/3):*
		- **No allow-list change was needed.** The layering check governs only the DAG components, and the helper is a leaf executable like the test_helper, so only its comment changed. The catalog and its test also build into `communityfinder_test_cases`, which was already allowed to link `honuware_scheduler`.
		- **Launch config.** `sync_launch_targets.ps1` added the `x64-Debug-local` launch entry, and `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` is in Mason's git-ignored `launch_defaults.local.json` (`all.env`), so the seed and the helper both see it.
		- The VS `x64-Debug` config gets its entry the next time the script runs for that config.

### 7. test_helper (`src/test_helper/commands/jobs_commands.{h,cpp}`)

- [x] 7.1 Three commands, all calling the business logic in-process, so no running server is needed:
	- `run-job [rj] --name=<job>` prints the counts;
	- `create-alert [ca] --description="…"` writes an `admin_alerts` row to exercise the digest;
	- `age-event [ae] --id=N --hours_ago=N` moves an event's start and end into the past, so archive and expire can be tried on demand.

	`create-event --past` already exists from EV1. **Tests** (`jobs_commands_test.cpp`).
	- *As built (11/3):*
		- `run-job` goes through `Jobs::RunScheduledJob` (§4.3), the same path as the endpoints. A missing or unknown name prints the seven job names.
		- `age-event` keeps the event's length: the end moves with the start.
		- **One-shot mode:** `--command=run-job --name=…`, `--command=create-alert --description=…` and `--command=age-event --id=… --hours_ago=…` all work. The flags were added to `main.cpp` during §8.4.
		- **Mail:** `run-job --name=notify_admin_alerts` sends a real email only when the test_helper was started with `--send_real_email`. Otherwise it goes to the capturing test mailer.
		- **Tests:** registration and aliases; `run-job`'s output and a repeat; `create-alert` feeding the digest; `age-event` then both sweeps; `age-event`'s flag checks.

### 8. Gates

- [x] 8.1 Docker suite green per slice, Windows too ("Windows: N passed. Linux: M passed."). §1 runs the honuware suite as well, and knottyyoga's only at its own batched re-pin. Raise the floor at bucket exit.
	- *Result (11/3), against the local honuware tree:* **Windows: 2390 passed. Linux: 2390 passed.** The floor in `build_and_test.sh` was raised from 2314 to **2380**.
	- The four executables (server, helper, test_helper, database_helper) also build on Linux. The gate builds only the test binary.
	- honuware's own Linux suite: **1913 passed** (floor 1700). Its Windows coverage of H13 comes from CF's Windows run, which links honuware's component tests.
	- **Still to come (§1.2):** the strict pinned gate after Mason's push.
- [x] 8.2 vitest green and `ng build` for all three configurations (for §3.3). *Result (11/3):* vitest **452 passed** (65 files). `ng build` passes for production, development and local. Prettier is clean after reformatting `event-detail.spec.ts`. The production bundle-size warning (912 kB against a 500 kB budget) was there before EV4.
- [x] 8.3 Live seed check (announced, destructive) with `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` set:
	- the scheduler person exists, `email_verified` is set, and it has the scheduler role holding only `run_scheduled_jobs`;
	- Administrator holds `run_scheduled_jobs` too.

	Then once more with the variable unset: the seed prints the skip line and exits 0.
	- *Result (11/3).* Run on a throwaway `communityfinder`, with Mason's dev database renamed aside and then restored unchanged.
		- **Unset:** the seed printed the skip line and exited 0. No scheduler person or role was created; `run_scheduled_jobs` was still seeded.
		- **Set:** the person exists with `email_verified_at` set. Its only role is `scheduler`, which holds only `run_scheduled_jobs`. The Administrator role (named `admin` in the database) holds it too.
- [ ] 8.4 **The phase gate (SUTP Phase 12's):**
	1. Run `communityfinder_helper --archive_past_events_interval=60 --expire_unreviewed_events_interval=60` against the local server.
	2. Its log shows the login, then a 200 with counts each minute.
	3. In the test_helper, `create-event --past --status=approved` and a pending `create-event` that you then `age-event` into the past.
	4. Within a minute, the first is `archived`, and its page still loads with "This event has passed"; the second is `rejected` with "Date passed before review."
	5. `create-alert` and then `run-job --name=notify_admin_alerts` with a recipient configured: one email arrives.
	- *Result (11/3): everything passes except the real email.* Same throwaway database, with a server on port 18091 so 18081 stayed free.
		1. **The helper** logged in once, then called both jobs every minute. Each answered 200, and the counts appear in the server log: `{"archived":1}`, then `{"decision_emails_sent":0,"expired":1}`, then zeros on later ticks.
		2. **The events.** Within a minute, the approved past event was `archived`; anonymous `GET /api/events/<id>` still returns 200. The aged pending event was `rejected` with "Date passed before review" and no reviewer; anonymous reads 404. The "This event has passed" note is client-side and covered by the §3.3 spec. Nobody looked at it in a browser.
		3. **The digest** works end to end with the capturing mailer: `{"alerts_count":1,"mail_sent":true}`, then a repeat gives `{"alerts_count":0,"mail_sent":false}` with the same stamp.
		4. **The real send was refused** by Gmail ("Mail sender rejection"). This happened with the throwaway's own seeded settings, and again after copying in **Mason's dev mail settings**, so his dev database has the same problem. Per the comment in `create_database.cpp`, Gmail returns this error when the app password doesn't belong to the sender address. **Mason: check that `HONUWARE_MAIL_APP_PASSWORD` (in `launch_defaults.local.json`) belongs to the configured sender, then re-run step 5.**
		5. **The refused send is handled correctly.** It left the stamp alone, so the next run retried the same alert, as §4.2 intends.
	- **Fixed during the gate (11/3):**
		- **The test_helper's one-shot mode** lacked flags for the new commands. `--id`, `--hours_ago` and `--description` are now declared in `main.cpp` (§7.1's tests call the commands through the registry, so they never noticed).
		- **The helper aborted silently when the server wasn't reachable.** libcurl's error reaches `main` as an exception from `Initialize()`. knottyyoga's `main` catches it, but CF's didn't. Uncaught, the helper aborted; in a Windows Debug build it sat at a hidden abort dialog that looked like a hang. It now catches the error like knottyyoga does, and exits in about 2 seconds with "… could not log in as … : CURL error: Could not connect to server). Is the server running, and was the database seeded … with the same SCHEDULER_SERVICE_ACCOUNT_PASSWORD?" A wrong password gives "… the login was refused …".

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **How to change honuware without forcing a knottyyoga re-pin.** The service-account setup needs to accept the permission names instead of assuming knottyyoga's `manage_subscriptions`. If we just change the function, knottyyoga stops compiling the next time it re-pins, and you batch those re-pins deliberately. *Rec: add the new version and keep the old one as a thin forwarder, marked for removal. knottyyoga then moves over whenever its batch happens. The cost is one extra function in honuware for a while.*
	- **Built as recommended, please confirm.** (11/3)
2. **What if the scheduler password isn't set when the database is rebuilt?** knottyyoga's seed fails outright. For CF, that would mean every developer needs the variable even if they never run the helper. *Rec: skip the account with a printed warning. The helper will then fail to log in, with an obvious message, which is easy to diagnose.*
	- **Built as recommended, please confirm.** (11/3)
3. **Should past events stay visible?** Today "archived" would mean hidden. Archiving would then empty the past days of the month calendar and turn every old event link into "not found," which hurts anyone who shared a link and anything a search engine indexed. *Rec: archived events stay readable (their page, and the calendar's past days). The upcoming list and calendar feeds still only show what's coming, because their date windows start at now. "Archived" keeps its real job: a re-scan can't reopen the event, and it drops out of the review screens.*
	- **Built as recommended, please confirm.** (11/3)
4. **Unreviewed events whose date passes are marked "not listed," not archived.** They were never public, so they mustn't become visible on the way out. Rejecting them with "Date passed before review" keeps "archived" meaning "happened and was listed." *Rec: as stated. EV3's submitter gets the "sorry, we didn't get to it in time" email.*
	- **Built as recommended, please confirm.** (11/3)
5. **Default intervals.** Archive and expire hourly; hygiene daily; the alerts digest hourly (it only mails when there's something new). *Rec: as in the table. All are flags, so a deployment can change them without code.*
	- **Built as recommended, please confirm.** (11/3)
6. **Where the jobs' code lives.** knottyyoga's hygiene and digest endpoints are app code over platform helpers. The tidier home is honuware itself (SUTP's H8 note floated the digest as an extraction candidate), but that's another cross-repo change and another knottyyoga clean-up. *Rec: CF writes its own thin endpoints now, over the platform helpers, with the digest logic tested in `business_logic/jobs/`. Add a ledger note that the hygiene endpoints are extraction candidates once both apps carry copies.*
	- **Built as recommended, please confirm.** (11/3) The note is follow-up (2) on ledger item H13.
7. *(New, 11/3)* **Should the month calendar show past days at all?** With OQ3, the calendar's past days now fill with archived events. That's useful for "what happened last week," but it also means the first thing someone sees on the 28th is a month of finished events above the few still coming. *Rec: leave it as it is for now. The upcoming list is the "what's next" view, and the calendar is the month at a glance. Revisit if testers find the past days distracting. One option then is to dim past days rather than hide them.*

# Change Log

- **11/3/2026 (v0.2)**
	- **Built:** all eight sections. Each section's as-built notes record where the code differs from the plan.
	- **Added:** §4.3, one shared entry point for the jobs, used by the endpoints and the test_helper. The endpoints log their counts server-side, because honuware's engine logs only the status code.
	- **Ledger:** item H13 added to [[Setting up the project]].
	- **Open questions:** OQ1–OQ6 built as recommended and awaiting confirmation; OQ7 is new (past days on the month calendar).
	- **Gates:** §8.1–§8.3 pass. §8.4 passes except the real email: Gmail refuses the dev mail credentials, a configuration issue rather than EV4 code.
	- **Two fixes found by the gate:** the test_helper's missing one-shot flags, and the helper's silent abort when the server is down.
	- **Still waiting:** Mason's honuware push, then the re-pin and strict pinned gate (§1.2); and the real-email step once the dev mail credentials are fixed.
- **11/2/2026 (v0.1.1)** — The expiry sweep now sets EV3's new `expired` reject reason on a submission that has none yet (the "Expired submissions get their email" pin-down). From EV3 v0.2 §1.1.
- **11/1/2026 (v0.1)** — Initial draft. It supersedes SUTP Phase 12, grounded in the honuware scheduler at the pinned SHA, knottyyoga's catalog and hygiene endpoints, and CF's deferred service-account comment. Found while drafting: honuware's `EnsureSchedulerServiceAccount` still requires knottyyoga's `manage_subscriptions`, so a small honuware change comes first (§1, proposed ledger item H13). Also proposed: past events stay publicly readable once archived (§3, OQ3), and unreviewed past events are rejected rather than archived (OQ4). Six open questions.
