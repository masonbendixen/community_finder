---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/19/2026
Version: 0.7.1
tags:
---

# Bucket CM3 — Event photos & video

**Track:** CM — Community · **Size:** M (the outline said S; video and submitter pictures are additions) · **Stream:** A, after EV3's gates (OQ9; the outline's Stream D row points here) · **Owner:** Levi or Mason (OQ9)
**Needs:** EV1 (done: events and venues already take a photo) · EV2 (done: the event page, list rows and sharing tags already show it) · EV3 (built; its gates are still open) for the submit form. **Feeds:** RT4 (sharing images) · RT3 (newsletter imagery) · CM4.

# Context

Mason, 11/3: *"I would also like events to have a main photo and an optional photo gallery and link to a YouTube (or other video) that gets embedded in the event. Visuals really market an event."*

That's the outline's **CM3 — Event photos**, "galleries on events/venues + the consent/no-outing policy," plus embedded video. The brainstorm's community list has the same item: "Event photos (platform photo pipeline; Metro Weekly's scene-photo archive is the proven engagement moat — with consent/no-outing policy)."

**What already exists:**
- **The main photo is half-built.**
	- honuware's photo pipeline stores one photo per row and scales it on demand.
	- EV1 enabled photos for `venues` and `events`. `register_app_services.cpp` lets signed-out visitors read the scaled photos of events, venues, organizations, places and guide pages.
	- EV2 shows the photo:
		- The event page asks `has_photo` first, then shows a 1200×630 image and puts it in `og:image` and the JSON-LD.
		- List rows ask for a 160×160 thumbnail and hide it if it fails.
		- An event with no photo gets the plum-to-coral tile.
- **Curators can only attach it through the generic admin editor.** honuware's CRUD row editor has `hw-photo-upload`. The `/admin/events` editor doesn't.
- **Deleting a row through generic CRUD also deletes its photo** (honuware `delete_item.cpp`).
- **What's missing:**
	- more than one photo per event;
	- any video;
	- a photo that stays hidden until its event is approved (photo visibility is switched on per table, not per row; see GD2 OQ12 (d)).
- **EV3 OQ7 decided no submitter photos** ("a curator adds the flyer on review"). OQ1 reopens it in light of Mason's ask.

**Read before starting:**
- [[Bucket EV1 — Events domain, server]] (house conventions, and §1.6 photo enablement);
- [[Bucket EV2 — Events public UI]] (OQ9 (f) on photos, and §3.7 on the curator editor);
- [[Bucket EV3 — Submission pipeline]] (the form and OQ7);
- [[Bucket GD1 — Taxonomy & editorial foundations]] §6.9 (no outing).

# Scope

**In:**
- **The main photo, everywhere an event shows:**
	- the event's own photo;
	- else its series' photo, else its venue's (OQ2);
	- chosen by the server, so pages stop guessing.
- **A gallery** on an event or a series (OQ7), each photo with alt text, an optional caption and an optional credit. Shown as a grid, with a full-size viewer. The cap is 12 by default, and an admin can change it (OQ5).
- **One embedded video** per event or series: YouTube or Vimeo (OQ3), shown as the provider's player from the start (OQ23, replacing OQ4's click-to-play).
- **Curator tools:**
	- the main photo, gallery and video in the `/admin/events` editor;
	- series through the generic admin editor, which already handles photos and child tables.
- **Submitter additions in the submit form** (OQ1): a video link and one flyer photo. The flyer needs H15, which is in scope.
- **The consent and no-outing policy** for photos (pin-down below), plus a way to ask for a photo's removal.
- The JSON-LD `image` list, test_helper commands, offline-demo placeholder art, and tests at every layer.

**Out (explicit non-goals):**
- Venue galleries. GD's venue page keeps its one photo; GD4 says "venue exterior shots are fine later."
- Video uploads or hosting. We only embed links.
- Instagram, TikTok and Facebook embeds (OQ3).
- Any photo on adult-layer events or pages (GD10 OQ3).
- Readers uploading photos of past events. That's a crowd feature: CM1/CM2 territory, and a separate consent problem.
- Face blurring or automatic moderation.

# Interface contracts this bucket must honor

1. **Namespaces (outline Contract 5):** Stream A owns `/events/*`, `/api/events*`, `/api/event_*` and `manage_events`. Everything here lives inside those. The new table is `event_gallery_photos`, managed under `manage_events`.
2. **Shared-file etiquette (Contract 4):** one-line additions to `create_app_tables.cpp`, `register_app_services.cpp`, the CMakeLists and `web_app.cpp`.
3. **No outing (GD1 §6.9):** being at an event is never reported as identity. This bucket's policy pin-down applies it to pictures.

# Design pin-downs

- **The main photo needs no new storage.** It stays the event's own photo in honuware's pipeline.
	- **The fallback (OQ2):** an event without one shows its series' photo, then its venue's. A series without one shows its venue's. This turns on photos for `series` (photo support and public read).
	- **The server picks.** Event payloads gain `photo: {"table": "events" | "series" | "venues", "id": N}`, or `null`. The client builds the URL with `PhotoUrlBuilder` as it does now.
		- The event page drops its separate `has_photo` call.
		- List rows stop guessing and hiding failures.
	- **One query per list,** not one per row: a join against `table_item_photos` for the three tables.
- **A gallery is rows, not a honuware change.** honuware stores one photo per row, so each gallery picture is a row in a new app table, and the row has its one photo.
	- That reuses upload, scaling, the public read, and the delete clean-up unchanged. A honuware change for several photos per row would touch every consumer.
	- **A row belongs to an event or to a series, never both.**
		- A recurring night's photos belong to the series: each dated instance goes past and drops off the calendar.
		- An event in a series with no gallery of its own shows the series' gallery.
	- **Order:** `sort_order`, then `id`. A new row goes one past the highest `sort_order` in its gallery.
	- **What the event page gets:** `gallery: [{id, alt_text, caption, credit}]`, only on `/api/events/<int>` and only while the event is visible. The series page (`/api/event_series/<string>`) gets the series' own gallery. Lists don't carry it.
	- **Row first, photo second.** A curator adds the row (with its text and the consent box), then uploads its picture through honuware's `/api/upload_photo/event_gallery_photos/<id>/<type>`.
		- A public gallery shows only rows whose photo has been uploaded, so a half-finished add never shows an empty frame.
		- The cap counts every row, uploaded or not.
	- **Adding a row has its own curator route** (a change from v0.1's "no new endpoints"): `POST /api/admin/event_gallery_photos`, under `manage_events`.
		- It stamps the consent record from the session, so nobody can type in someone else's name.
		- It applies the cap.
		- The consent columns have no defaults, so generic CRUD can't add a row without them. Editing, reordering and deleting still go through generic CRUD.
	- **The cap is a setting** (OQ5): `event_gallery_max_photos`, default 12, from 1 to 60.
		- It's read and saved through `GET`/`PUT /api/admin/event_gallery_settings` under `manage_events`, the same shape as EV4's event settings: `{value, default, min, max}`.
		- Lowering it never removes photos. It only stops new ones.
		- *Known limit:* two curators adding at the same moment could both get past a full cap by one. It's curator-only, so this is accepted.
- **Video is a link we parse, never HTML we store.** `video_url` on events and series holds a canonical link:
	- `https://www.youtube.com/watch?v=<11-char id>` or `https://vimeo.com/<digits>`.
	- Pasted variants become canonical: `youtu.be/…`, `/shorts/…`, `/live/…`, `m.youtube.com`, extra `&t=` parameters.
	- Refused: playlists, channel, album, showcase and user pages, and unlisted Vimeo links (their private hash can't be played from the id alone).
	- **`ParseVideoLink` (business_logic) is the one parser.** Payloads carry `video: {"provider": "youtube" | "vimeo", "id": "…"}` or `null`.
	- **An event without its own video shows its series'** (OQ7).
	- **The client builds the player address only from that id:**
		- YouTube: `https://www.youtube-nocookie.com/embed/<id>`;
		- Vimeo: `https://player.vimeo.com/video/<id>?dnt=1`.
		- The address goes through Angular's trusted-URL wrapper once and is cached (knottyyoga's Location-page pattern).
	- **A database CHECK is the backstop.** Curators save events through generic CRUD, which no app code sees, so the column only accepts the two canonical shapes. The editor turns a pasted link into its canonical form before saving.
- **The provider's own player, from the start (OQ23, reversing OQ4's click-to-play on 11/18).** The page shows YouTube's no-cookie player, or Vimeo's do-not-track one, with its own still and one tap to play.
	- The frame has a title ("Video: Leather Night"), allows fullscreen, and doesn't autoplay.
	- It's `loading="lazy"`, so the player loads only when the visitor scrolls near it.
	- *The trade Mason accepted:* every event or series page with a video loads the provider's player script once it's scrolled into view. Each visitor's browser then talks to Google's (or Vimeo's) servers whether or not they watch. The full write-up is under OQ23.
	- *History:* click-to-play (OQ4) first showed the event's own photo on a tile, which read as a frame of the video and was misleading (§17.1). Then it showed a plain gradient, which Mason didn't want. He chose the full player as "the best experience".
- **The page's content security policy must allow the two players.** honuware's `SecurityHeaders` policy rides the API's responses. The page itself is served from S3/CDN in knottyyoga's setup, so the page's policy belongs to PL2.
	- Wherever `index.html` is served, `frame-src` must name `https://www.youtube-nocookie.com` and `https://player.vimeo.com`.
	- `img-src` stays `'self' data:`. The player draws its still inside its own frame, so the page itself shows no outside image.
	- This is a PL2 item, not a honuware change. If PL2 ends up serving the page through the C++ server, it becomes a small honuware change: frame sources set by the app, as `SetServerBanner` is today.
- **The photo policy (consent and no outing).** It's written for curators and shown beside every upload:
	- **Always fine:** flyers and artwork, the venue, performers' own promo images, and crowd shots where nobody can be recognized.
	- **A recognizable attendee** is shown only with their agreement. Performers on stage are public-figure context.
	- **Captions and alt text** never name attendees. Name performers or organizers only with their agreement.
	- **Every gallery photo records who confirmed consent and when** (OQ6). Upload takes a required "Everyone recognizable in this photo agreed to be shown" box.
	- **Removal on request, no questions asked:**
		- every gallery shows "Ask us to remove a photo," using the site's contact address for now (CM1's report flag later);
		- a curator removes it within a day.
	- **Adult-layer events get no photos** (GD10 OQ3). Until events have an 18+ flag (EV6 OQ9), that's a curator rule.
- **Which photos a signed-out visitor sees (H15, OQ1).** honuware's public-photo list now takes an optional per-row rule from the app. CF's rule (`IsPhotoRowPublic`):
	- an `events` photo is public once its event is approved or archived. Pending, rejected and missing events are not;
	- an `event_gallery_photos` photo follows its parent: a series row is always public, an event row follows its event, and a row that's gone isn't;
	- every other listed table (venues, series, organizations, places, guide pages) is public, as before.
	- A photo the rule refuses takes the normal signed-in path, so curators still see it, and its response is never cached publicly.
- **The submitter's flyer (OQ1).** `POST /api/event_submissions/<int>/photo/<jpeg|png>`, with the image as the body.
	- Only the submission's owner may add it. Someone else's submission is a 404, as for withdraw.
	- It goes on every date still waiting for review, and a second flyer replaces the first. A date that's already been decided keeps what it has.
	- JPEG or PNG only: never a vector from a stranger. It must be within `image_max_upload_bytes`, and it must decode.
	- It stays private until a curator approves the date (H15).
- **Sizes.** `hw-photo-upload` already shrinks and re-encodes on the way up, and the server enforces `image_max_upload_bytes`. The sizes:
	- the existing thumbnail (160×160) and hero (1200×630);
	- a gallery thumbnail (480×360);
	- a full view (1600×1200).
	- Scaled copies regenerate on demand. EV4's cleanup job already prunes old ones.
	- *honuware caps every scaled photo at 800×800* (its `image_max_public_scaled_*` defaults). Despite the keys' name, signed-in readers are capped too: `get_scaled_photo.cpp` applies the cap after the login check, whichever way a request came in.
		- **Decided (OQ10):** CF raises it to 1600×1200 (§1.5), so the hero and the full view come out whole for everyone.
- **Sharing tags.** `og:image` stays the main photo. The JSON-LD `image` becomes a list: the main photo, then up to three gallery photos.

# Layered work items

### 0. honuware

- [x] 0.1 **H15** (OQ1), a per-row photo visibility check.
	- `PublicPhotoTables` gains `isRowPublic`, an optional rule the app supplies: `(DatabaseHelper, Transaction, table, id) → bool`. CF's rule is in the photo pin-down above.
	- A photo that isn't public falls back to the normal signed-in read, so curators still see it, and gets `Cache-Control: private, no-cache`.
	- `get_scaled_photo` uses it. Additive: with no rule, every row of a listed table is public as before, so knottyyoga needn't move.
	- **Tests** in honuware (`get_scaled_photo_test.cpp`) and the README. Mason pushed it as `f445a49` on 11/5, and CF and knottyyoga now pin that commit.
	- *Without H15, a pending event's photo would be readable by anyone who guessed its number, as GD2 OQ12 (d) accepted for draft guide pages.*

### 1. db_schema

- [x] 1.1 `src/db_schema/event_gallery_photos.{h,cpp}`:
	- `id`;
	- `event_id` and `series_id`, both nullable FKs with cascade, and a CHECK that exactly one is set;
	- `sort_order` int, default 0;
	- `alt_text` (required, 1–300);
	- `caption` (≤300, default `''`) and `credit` (≤120, default `''`);
	- `consent_confirmed_by_person_id` and `consent_confirmed_us`, both required with no default. *As built:* the person is a plain `people.id`, not an FK. honuware's FKs always cascade, so a link would delete the photos someone added when their account is deleted (EV1 OQ8 (e)).
	- `created_us` and `updated_us`;
	- indexes on (`event_id`, `sort_order`) and (`series_id`, `sort_order`).
- [x] 1.2 `video_url` on `events` and on `series`: not null, default `''`, with a CHECK for `''` or the two canonical shapes (`db_schema/video_links.h`).
- [x] 1.3 `create_app_tables.cpp` (the three-place rule):
	- `MakeAppTables`;
	- `AppTableCreationOrder` (after `events` and `series`);
	- `PopulateAppTables`, which covers:
		- photo support for `event_gallery_photos` and `series`;
		- the `manage_events` table mapping;
		- admin metadata, so the gallery shows as a child table of events and series in the generic editor. The consent columns are read-only there.
	- The gallery cap's default comes from the app's secret defaults (`app_secret_values.cpp`).
	- **Tests:** `create_app_tables_test.cpp` (photo support rows, the mapping, the nesting), and the CHECKs (both parents, neither parent, blank alt text, a non-canonical video link).
- [x] 1.4 `register_app_services.cpp`: public read for `series` and `event_gallery_photos`, with `IsPhotoRowPublic` as the row rule (H15). **Tests:** `register_app_services_test.cpp`: a signed-out visitor gets a series photo and a gallery photo; a pending event's photo needs a login until the event is approved.
- [x] 1.5 **The public photo size (OQ10):** `image_max_public_scaled_width` = 1600 and `image_max_public_scaled_height` = 1200 in CF's seed.
	- Both keys belong to honuware, which already seeds them at 800. They can't go in `FillInAppSecretDefaults`: that would fail `AppSecretValuesTest.FrameworkAndAppKeySetsDoNotOverlap` and hit the unique `name`.
	- *As built (11/18):*
		- `App::FillInFrameworkSecretOverrides` (`app_secret_values.cpp`) lists CF's values for keys honuware defaults, apart from the app's own defaults.
		- `ApplyFrameworkSecretOverrides` (`create_app_tables.cpp`) UPDATEs honuware's rows. It throws if a row is missing, so a key honuware renames fails the seed instead of leaving the site on 800.
		- `PopulateConfigSecrets` (`create_database.cpp`) calls it, beside the mail app password's UPDATE.
		- The test main gives the test secrets double the same values, so tests see what a live server does.
	- honuware is unchanged, and knottyyoga stays at 800×800.
	- An existing database keeps 800 until it's reseeded or an admin edits the two rows.
	- **Tests:**
		- the list: two honuware keys, 1600 and 1200, neither an app default nor the same as honuware's value;
		- the seed step: honuware's rows are overwritten (still one row each), and a missing row is refused;
		- end to end: a signed-out visitor who asks for a 2048×1536 photo gets exactly 1600×1200.
		- The live seed check (§7.3) confirms the rows.

### 2. table_helpers

- [x] 2.1 `event_gallery_photos.{h,cpp}`:
	- `AddGalleryPhoto` and `GetGalleryPhoto`;
	- `GetGalleryForEvent` and `GetGalleryForSeries` (ordered), each row with `has_photo`;
	- `UpdateGalleryPhoto` (bumps `updated_us`);
	- `DeleteGalleryPhoto`.
	- *As built:* no `CountFor*`. A gallery is at most the cap's rows, so the business layer counts the rows it already reads.
	- **Tests.**
- [x] 2.2 `events` and `series`: `video_url` in add, update and the reads. **Tests.**
- [x] 2.3 **The photo picker:** `Events::GetPhotoOwners(eventIds)` → for each event, which of event, series or venue has a photo, in one query against `table_item_photos`; `Series::GetPhotoOwners(seriesIds)` does the same for series and their venues. **Tests:** each of the four outcomes, and a mixed list.

### 3. business_logic

- [x] 3.1 `video_link.{h,cpp}`: `ParseVideoLink(text)` → `{provider, id, canonicalUrl}` or an error code (`BAD_VIDEO_URL`, `VIDEO_HOST_NOT_SUPPORTED`). `StoredVideoLink(video_url)` reads a stored link back.
	- **Tests:**
		- each YouTube variant, with and without extra parameters;
		- a playlist-only link (refused);
		- a Vimeo number, a channel video (accepted), and a channel, album, showcase, user page or unlisted link (refused);
		- `http:`, no scheme, capitals and stray spaces;
		- another host;
		- an id of the wrong length.
- [x] 3.2 `EventHelper`:
	- the resolved `photo` on list and detail reads, for events and for series;
	- `gallery` (the event's own, else its series') on the event detail read, only when the event is visible; the series' own on the series detail read;
	- `video` (the event's own, else its series').
	- **Tests:** the fallbacks; a pending event's gallery hidden from the public read and shown in the curator preview.
- [x] 3.3 **The gallery** (`event_gallery_helper.{h,cpp}`) **and its cap** (`gallery_settings.{h,cpp}`, OQ5):
	- `AddGalleryPhoto` checks one parent that exists, trimmed alt text (required), caption and credit lengths counted in characters, and the consent box. Then the cap, then it stamps the consent record. It writes nothing on a refusal.
	- a row past the cap is refused with `GALLERY_FULL` and the cap;
	- `LoadGallerySettings` / `SaveGallerySettings` for `event_gallery_max_photos` (1–60, default 12). A stored value out of range reads as the default;
	- `DeleteGalleryPhoto` deletes the row and its photo through `ImageHelper::DeletePhotoForItem`. Generic CRUD already does this.
	- **Tests.**
- [x] 3.4 `SubmissionHelper` takes `video_url`, checks it with `ParseVideoLink` (field `video_url`, the EV3 problem shape), and stores the canonical link on each date's event. **Tests.**
- [x] 3.5 **The flyer** (OQ1): `SubmissionHelper::AttachFlyer` locks the submission and checks:
	- the owner (`NOT_YOURS`);
	- something is still pending (`NOTHING_PENDING`);
	- JPEG or PNG (`BAD_IMAGE_TYPE`);
	- the size (`IMAGE_TOO_LARGE`), and that the image decodes (`BAD_IMAGE`).
	- Then it uploads to each pending date. **Tests**, including a second flyer replacing the first and every refusal writing nothing.
- [x] 3.6 **The H15 rule:** `photo_visibility.{h,cpp}`, `IsPhotoRowPublic`. **Tests:** each event status, gallery rows on each kind of parent, a missing row, other tables.

### 4. endpoints

- [x] 4.1 Payloads gain `photo` and `video` (`/api/events/upcoming`, `/api/events/<int>`, `/api/event_series*`), and `gallery` (`/api/events/<int>` and `/api/event_series/<string>`). **Tests** (`handle_full`, `ThreadPool::Shutdown()` first):
	- an event with each kind of photo;
	- a gallery in order;
	- a video;
	- a pending event's gallery not shown publicly.
- [x] 4.2 **Two new curator endpoints** (a change from v0.1, see the gallery pin-down), both under `manage_events`:
	- `POST /api/admin/event_gallery_photos` adds a row: `{event_id | series_id, alt_text, caption?, credit?, consent_confirmed}` → `{id}`. A refusal is a 400 with `error_code` and `field`, plus `max_photos` on `GALLERY_FULL`.
	- `GET`/`PUT /api/admin/event_gallery_settings` reads and saves the cap.
	- Editing, reordering and deleting rows go through generic CRUD. Their photos go through honuware's upload and delete endpoints.
	- **Tests:** a curator adds a row and their consent record is stamped; each refusal names its code and field; a full gallery says what the cap is; someone signed out or without `manage_events` is refused; the settings read, save and refuse bad values.
- [x] 4.3 `POST /api/event_submissions` takes `video_url`. **Tests:** accepted and stored in its canonical form, and refused with the field named.
- [x] 4.4 **The flyer route:** `POST /api/event_submissions/<int>/photo/<jpeg|png>`, the image as the body, anchored through the submission endpoints' anchor. **Tests:** signed out, someone else's submission (404), a vector refused, the flyer on each waiting date, nothing pending after a withdraw.

### 5. client (`ui/src/app/`)

- [x] 5.1 **Types and access:** `photo`, `gallery` and `video` on the event and series types, and in the offline mock. The demo gets placeholder art (generated shapes and color fields, not people), one gallery and one video id. **Specs.**
	- *As built (11/18):* `PhotoRef`, `VideoRef` and `GalleryPhoto` in `core/types/events.types.ts`, with `toPhotoRef`, `toVideoRef`, `normalizeGallery` and `normalizeEventDetail`. `getEvent` now returns an `EventDetail` (the event plus its gallery).
	- The mock follows the server's fallback: Leather Night (103) has its own photo, a gallery of four and the demo video; Bear Happy Hour's series (11) and The Lantern Room (venue 2) have photos.
	- `core/mock/demo-photo-art.ts` draws the placeholders as SVG shapes, and `DemoPhotoUrlBuilder` stands in for honuware's `PhotoUrlBuilder` in mock mode only.
	- **The demo's video is `lVhJ_A8XUgc`** (OQ18, Phase 2 §15), which plays.
- [x] 5.2 **One place for photo addresses:** `photoFor(event, width, height)`. Event rows and the event page switch to it, and the event page drops its `has_photo` call. **Specs:** each owner, and none.
	- *As built:* `EventPhotos` (`core/services/event-photos.service.ts`): `photoFor(owner, size)` and `galleryPhotoUrl(photo, size)`, with the sizes as constants (list thumb 160, hero 1200×630, gallery thumb 480×360, full view 1600×1200). The event page no longer asks `has_photo`.
- [x] 5.3 `shared/components/photo-gallery`:
	- a thumbnail grid;
	- a full-size viewer in a dialog: arrow keys, Escape, swipe on a phone, alt text, caption and credit, focus returned on close;
	- "Ask us to remove a photo."
	- **Specs.**
	- *As built:* 11 specs. The viewer is `role="dialog"` with `aria-modal` and CDK's focus trap. The removal links are mailto links to `SiteConfig.contactEmail` (see OQ8), one for the whole gallery and one for the photo being viewed.
- [x] 5.4 `shared/components/video-embed`: the play tile, then the frame on press. It's built only from `{provider, id}`. **Specs:**
	- no frame before the press;
	- the right address for each provider;
	- the frame's title;
	- the address isn't rebuilt on every change check.
	- *As built:* 6 specs. The address is a `computed`, so it's sanitized once per video. A new video puts the tile back. `core/utils/video-links.ts` mirrors the server's `ParseVideoLink`, for the editor's and the submit form's feedback.
	- *Changed 11/18 (Phase 2 §17):* first the `posterUrl` input went (§17.1). Then the tile itself went (§17.2, OQ23). It's now the provider's player from the start, lazy and not autoplaying, with 5 specs.
- [x] 5.5 **The event page and the series page:** the hero (resolved), then the video, then the gallery. The JSON-LD `image` list. **Specs.**
	- *As built:* `og:image` stays the main photo. The JSON-LD `image` list is the main photo, then up to 3 gallery photos (`JSON_LD_GALLERY_IMAGES`). A gallery with no main photo gives JSON-LD images but no `og:image`. The series page gets its hero and `og:image` too.
- [x] 5.6 **The curator editor** (`pages/admin/manage-events.*`):
	- the main photo through `hw-photo-upload`;
	- a video field that shows what it found ("YouTube video ✓") or why not, and saves the canonical link;
	- **a gallery section:**
		- thumbnails with their alt, caption and credit;
		- move up and down;
		- remove, with a confirm in the page;
		- "Add photo": alt text, caption, credit, the consent box, and `hw-photo-upload` with `deferUpload`. The row is created first (`POST /api/admin/event_gallery_photos`), then its photo uploaded. A `GALLERY_FULL` refusal says what the cap is.
	- **the gallery cap** on the admin Event settings page, beside EV4's settings (`/api/admin/event_gallery_settings`);
	- **Specs.**
	- *Series use the generic admin editor (photo upload plus the gallery as a child table) until someone asks for more.*
	- *As built (11/18), in `pages/admin/event-editor.*`:*
		- **The video field** says "YouTube video ✓" or why not as it's typed. Save refuses an unreadable link, and stores the canonical link or `''` (`videoColumnValue` in `manage-events-form.ts`).
		- **The main photo and the gallery** appear below the form only once the event is saved. A new event says to open it again after adding it. Both save on their own, apart from the form's Save. Each upload has a "Which photos are fine" note with the policy (`photo-policy-note.ts`).
		- **The gallery** is `event-gallery-editor.*`, showing "3 of 12 photos".
			- A move swaps the two rows' `sort_order`. If two rows share a `sort_order`, it renumbers the whole gallery instead.
			- Remove deletes the photo, then the row. A row whose photo is already gone is still removed.
			- "Add photo" checks the alt text, the consent box and the chosen file before sending anything. Then it makes the row and uploads the photo against it. If the upload fails, the row is deleted again, so no row is ever left without a picture.
			- What was typed stays after a refusal, so a `GALLERY_FULL` can be fixed by removing a photo and pressing Add again.
		- **The cap** is a card under a new "Photo galleries" heading on the Event settings page (`gallery-settings-card.*`), checked against 1–60 before sending.
		- **Not built:** changing a gallery photo's alt text, caption or credit after it's added. For now, remove and re-add the photo, or edit the row in the generic admin. Editing is being added in Phase 2 §10 (OQ12).
		- 15 new gallery specs, 6 for the cap card, and 9 more in the editor and form specs.
- [x] 5.7 The submit form gets an optional "Video link" with the same feedback, and an optional "Flyer" (JPEG or PNG), uploaded after the submit succeeds to `/api/event_submissions/<id>/photo/<type>`. "My submissions" shows them. **Specs.**
	- *As built (11/18):* the video link is checked before sending (field `video_url`) and sent canonical.
	- **The flyer** is picked with `hw-photo-upload` in `deferUpload` mode, so it looks like every other upload. The page then sends the file's own bytes.
		- The picker accepts any image, so the page refuses anything but a JPEG or PNG before sending (by type, or by extension when the file has none).
		- No re-encode: `hw-photo-upload`'s resize is private. A flyer over the server's 10 MB limit gets the server's "The flyer is too large." Phase 2 §13 makes the limit a setting and shrinks flyers first (OQ13).
	- The flyer goes up only after the submission is in. If it fails, the submission stands, and the thanks say "Your event was sent, but its flyer couldn't be added." with the server's reason. There's no way to retry yet; Phase 2 §11 adds one on "My submissions" (OQ14).
	- A honeypot send (`submission_id` 0) uploads nothing.
	- **A gap:** if submissions are paused between opening the form and sending it, the form comes back with what was typed, but the chosen flyer has to be picked again. Accepted as it is (OQ15).
	- **"My submissions"** says "With a photo · With a Vimeo video" under each date. For that, the server's `/mine` now sends `has_photo` and `video` per date (`submission_helper.*`, `event_submissions.*`, `events_json.*`, with a test in each layer). **These server changes aren't committed yet.**

### 6. test_helper

- [x] 6.1 Two commands for local data, in a new "Media" group (`commands/media_commands.*`), named in the house's kebab-case:
	- `set-event-video` (`sv`) `--event_id N | --series_id N --url …`, which goes through `ParseVideoLink`. `--clear` removes the video;
	- `add-gallery-photo` (`ag`) `--event_id N | --series_id N --file … --alt … [--caption …] [--credit …]`. It adds the row through `EventGalleryHelper` (so the cap applies), uploads the file through `ImageHelper`, and records the logged-in person as the consent record. A login is required. If the upload fails, the row is taken back out.
	- **Tests.**

### 7. Gates

- [x] 7.1 Docker suite green per slice, Windows too ("Windows: N passed. Linux: M passed."); raise the floor at bucket exit.
	- **Bucket exit (11/18), strict pinned to honuware `838045f`: Windows 2525 passed. Linux 2525 passed.** That includes the two new `/mine` tests. The 4 CRUD failures from the earlier §1.5 run are gone, since `838045f` has the CRUD write check. The floor is raised from 2490 to 2515 (`server/docker/build_and_test.sh`).
	- *Server side (11/5):* Windows 2500 passed, Linux 2500 passed, pinned to honuware `f445a49`. knottyyoga is 5454 on each. The floor is now 2490. The client gates (§7.2) are still to come.
	- *§1.5 (11/18), pinned to `f445a49`:* Linux 2506 passed. Windows: 2503 of 2507 passed. §1.5's 6 new tests pass on both.
		- The 4 Windows failures are CF's generic-CRUD tests in `crud_test.cpp` and `roles_permissions_test.cpp`. They were edited at 11:04, between the two builds, to expect honuware's uncommitted CRUD write check (knottyyoga Beta feedback Phase 1.2), which `f445a49` doesn't have. That's also the 2507th test. They should pass once that change is pushed and re-pinned.
- [x] 7.2 `ng build` for local, development and production; vitest green, with the count recorded; the bundle change noted. The viewer and player load with the event pages, not up front.
	- *11/18:* all three builds pass. **Vitest: 563 passed in 71 files** (was 487 in 67).
	- **Production bundle, before → after:**
		- initial: 918.46 → 931.83 kB raw (+13.4); 212.9 → 216.5 kB transferred (+3.6);
		- `events-routes`: 161.14 → 177.76 kB (+16.6): the viewer, the player and the flyer;
		- `admin-routes`: 218.95 → 236.69 kB (+17.7): the gallery editor and the cap card.
	- The 500 kB initial budget warning was already there before this bucket.
	- **Lazy as required:** the player and the viewer ("Plays from YouTube", "Photo removal") are only in the `events-routes` chunk, and the gallery editor only in `admin-routes`.
	- **Why initial grew:** mostly the offline mock. `app.config.ts` picks mock or HTTP at runtime from `environment.useMock`, so the mock's data and handlers ship in every build, production included. That was true before CM3. This bucket adds the demo gallery, the placeholder art, the gallery mock and the video-link parser the mocks use.
	- *A follow-up:* swap the mocks in at build time for the `local` config, so production leaves them out. That was agreed in OQ16 and is planned as Phase 2 §14. Measured: 61.7 kB of mock code in production, 44.0 kB of it up front.
- [ ] 7.3 **Live seed check after §1** (destructive, announced). *Until this runs, Mason's dev database lacks CM3's schema and every event page fails (Phase 2 §8).*
	- `event_gallery_photos` with its CHECKs and indexes;
	- `video_url` on both tables, with its CHECK;
	- the photo-support rows for `series` and `event_gallery_photos`;
	- the `manage_events` mapping;
	- `event_gallery_max_photos` = 12 in `config_secrets`;
	- `image_max_public_scaled_width` = 1600 and `image_max_public_scaled_height` = 1200 in `config_secrets` (§1.5);
	- nothing new in `allowed_tables`;
	- *(Phase 2 §12.1)* the `curator` role, holding `manage_events` and nothing else;
	- *(Phase 2 §18)* after `--populate_sample_data`: photos on a few events and one series, a three-photo gallery on one event, and the video on one event and one series.
- [ ] 7.4 **Manual loop (Mason or Levi), on a phone as well as desktop:**
	1. Add a main photo, three gallery photos and a YouTube link to an event.
	2. Reorder the photos, then remove one.
	3. Check the event page, the list thumbnail, and a series instance with no photo of its own.
	4. Press play.
	5. Check the shared-link preview.
	6. Follow the removal-request path.

# Phase 2 — Follow-ups (OQ11–21 and Mason's new work)

Planned 11/18 from Mason's answers to OQ11–18 and his two new-work items. Each section names where it came from.
- **The rule from OQ12 applies throughout:** anything that can be created gets full CRUD.
- **Order:** §8 first, because it blocks the manual loop (§7.4). Then the client-only items (§9–11, §14–15), then §12, §13 and §18. §17 is done. §8.2 and §13 are cross-repo bump points.
- **Layering:** as in §0–7. Server items go endpoints → business_logic → table_helpers. Each new endpoint is anchored in `RegisterAllEndpoints()`, and each new method gets a test.

### 8. Event pages fail on the dev database (Mason New Item 1)

Mason: *"When clicking on an event, I get: This event couldn't be loaded. Please try again in a moment. Is this something that is supposed to be implemented in a later phase or just broken. If there is no work item for this, lets add one to this document."*

**Neither a later phase nor broken code: the dev database is older than CM3.** Checked 11/18:
- `GET /api/events/upcoming` returns 200, with CM3's `photo` field, so the running server is the new build.
- Every `GET /api/events/<id>` tried (ids 25, 10 and 17) returns a 500.
- A read-only query of `communityfinder` shows no `event_gallery_photos` table, and no `video_url` column on `events` or `series`.
- The event page reads the event's gallery inside `GetEventDetail`, and any database error there becomes a 500 (`get_event_detail.cpp:42`). The list doesn't read galleries, so it kept working.

The items:
- [x] 8.1 **Recreate the dev database** with `--recreate_database`, which is §7.3. ✅ 11/18, Mason ran it.
	- The recreate seeds no events. The sample events come from a second flag, `--populate_sample_data`, which can run alone or with `--recreate_database` (it runs after it). Both need `HONUWARE_ALLOW_DESTRUCTIVE=1`.
	- [ ] Still to check: with the sample data loaded, an event page opens. Then §7.3's checklist can be ticked against the new schema.
	- The sample data has no photos, gallery or video, so it doesn't show CM3 (**OQ22**).
- [x] 8.2 **The startup schema check (OQ20), in honuware,** so the next out-of-date database explains itself. Mason: *"I've been burned by this a lot."* ✅ 11/18. It's in honuware `0b249be`, which CF pins from 11/19.
	- At startup, the server compares the tables and columns in its composed `DatabaseInfo` with what `information_schema` says the database has.
	- Anything missing gets one clear warning, listing every missing table and column, for example: "The database is older than this build: missing table event_gallery_photos; missing column events.video_url. Run --recreate_database (dev) or --migrate."
	- The server keeps running. Extra tables or columns the build doesn't know about are ignored.
	- **Tests:**
		- a current database gives no warning;
		- a missing table is named;
		- a missing column is named;
		- several missing items share one message;
		- an extra column is ignored.
	- A cross-repo bump point: Mason pushes honuware, I re-pin CF and verify a fresh pinned build, Mason commits, and knottyyoga follows.
	- **As built:**
		- **honuware** `data/sql_util/database_access/schema_check.{h,cpp}`, in `DbMeta`:
			- `FindMissingSchema(transaction, databaseInfo)` makes one read of `information_schema.columns` (public schema). It returns the missing tables, in the build's order, and the missing columns as `table.column`. A missing table's columns aren't listed again.
			- `DescribeMissingSchema(missing)` gives the one-line warning, or "" when nothing is missing. Its advice names both apps' database-helper flags: "Run the database helper with --migrate, or --recreate_database on a dev database."
			- `WarnIfDatabaseIsOlderThanBuild(transaction, databaseInfo)` logs it and returns whether it did. If the check itself fails, that's logged too, and it never throws.
		- **CF** `main.cpp` calls it first, right after the composed `DatabaseInfo` is built, so the warning is logged even when a later startup step trips over the same gap.
		- **Tests** (`SchemaCheckTest`, 5): the five above. "An extra column is ignored" also covers an extra table.
		- **knottyyoga** gets it by adding the same call to its `main.cpp` after the bump. That's its own change.

### 9. Edit Event: the next ten events, and search results that tell events apart (Mason New Item 2)

Mason: *"In Admin Dashboard / Events / Edit Event, I would like: A list of the ten next upcoming events I can click on to edit. If I type a search term, I'd like the result to show information like the date, time, and venue in addition to the event title. Currently, I wanted to edit Board Game Night and it is a series and showed a bunch of items with the same name and I had to try each one to edit the right one."*

**Why it happens:** the search box uses honuware's generic lookup (`getFkOptions('events', …)` in `manage-events.ts:230`). That returns each event's display name, and the events table's display template is just `{title}` (`create_app_tables.cpp:799`). So every Board Game Night looks the same.
- Changing the template isn't the fix. Times are stored as microseconds, which a template can't format, and the template also labels events in every picker in the generic admin.

The items:
- [x] 9.1 **table_helpers:** an `Events` method that finds events whose title contains the search text. It ignores case, and escapes `%` and `_` in the search text. It returns the id, title, start, end, venue name or location text, series title and status, sorted as in 9.2, with a limit. **Tests:** a match in the middle of a title, any capitalization, a `%` in the search taken literally, the limit, and the order.
- [x] 9.2 **business_logic:** `EventHelper::FindForCurators(transaction, searchText, limit)`.
	- With search text: upcoming matches first (soonest first, counting one still running as upcoming), then past ones (most recent first).
	- With none: the next `limit` upcoming events.
	- Every status is included and labelled, since curators edit pending and rejected events too.
	- **Tests.**
- [x] 9.3 **endpoint:** `GET /api/admin/events/search?q=<text>&limit=<n>` under `manage_events`, returning `{items: [{id, title, starts_at_us, ends_at_us, venue_name, location_text, series_title, status}]}`. The limit defaults to 10, with a maximum of 50. **Tests** (`handle_full`): signed out, without the permission, a search, the blank list, and the limit clamped.
- [x] 9.4 **client:** the Edit Event tab.
	- Under the search box, a "Coming up" list of the next ten events: "Thu, Nov 20 · 7:00 PM · Cafe Mox · Pending", each opening in the editor.
	- Search results show the same date, time and venue under the title, so "Board Game Night" appears once per date.
	- The list refreshes after a save.
	- The search moves off `getFkOptions` onto 9.3. The access layer and the offline mock get the new call.
	- **Specs.**
- **As built (§9), 11/18:**
	- **table_helpers:** `Events::FindByTitle` and `Events::GetNotEndedBy`, both in every status, with the venue, location text and series title joined in. **Tests:** `FindByTitleMatchesAnywhereInTheTitleInAnyCase`, `FindByTitleTakesLikeCharactersLiterally`, `FindByTitleListsWhatHasNotEndedSoonestFirstThenThePastNewestFirst`, `GetNotEndedByListsWhatIsComingSoonestFirstInEveryStatus`.
	- **business_logic:** `EventHelper::FindForCurators` returns `CuratorEventMatch` rows; `ClampCuratorFinderLimit` gives the default of 10 and the cap of 50. **Tests:** the clamp, the search telling events apart, and the blank list.
	- **endpoint:** `endpoints/admin_event_search.{h,cpp}`, anchored in `RegisterAllEndpoints()`. `ends_at_us` is a real null when an event has no end. **Tests** (`AdminEventSearchTest`, 3): 401 and 403; a search showing each date with its venue, series and status; the blank list with the limit clamped.
	- **client:** `EventsAdminAccess.findEvents(search, limit?)`, over HTTP and in the mock. Edit Event's "Coming up" and its search results share one line format.

### 10. Edit a gallery photo's words (OQ12)

- [x] 10.1 **client:** an "Edit" button on each row of the event editor's gallery. ✅ 11/18: `EventGalleryEditor.startEdit` and `saveEdit`, with the specs listed below. It opens the alt text, caption and credit in place, with Save and Cancel and the same length limits, and alt text still required.
	- It saves through generic CRUD (`updateItem` on `event_gallery_photos`), sending only the fields that changed, trimmed. The database's CHECK refuses blank alt text even if the client let it through.
	- **Specs:** editing each field, Cancel leaving the row as it was, blank alt text refused before sending, and a failed save saying so and keeping what was typed.
	- *Series galleries are still edited in the generic admin, which already has full CRUD.*

### 11. Add, replace or remove the flyer from "My submissions" (OQ14, and the full-CRUD rule)

- [x] 11.1 **business_logic:** `SubmissionHelper::RemoveFlyer(submissionId, personId)`. It removes the photo from every date of the submission still waiting for review, with the same checks as `AttachFlyer`: a 404 for someone else's submission, and `NOTHING_PENDING` when no date is waiting. **Tests.**
- [x] 11.2 **endpoint:** `DELETE /api/event_submissions/<int>/photo`, anchored through the submission endpoints' anchor. **Tests:** signed out, someone else's submission, removing from each waiting date, and nothing pending.
- [x] 11.3 **client:** on "My submissions", while any date of a submission is waiting:
	- "Add a flyer" (or "Replace the flyer" once one is attached, which `/mine`'s `has_photo` tells us) opens the same `hw-photo-upload` picker with the same JPEG/PNG check;
	- "Remove the flyer" asks in the page first.
	- The thanks page's failed-flyer message gains "You can add it from My submissions."
	- **Specs.**
- **As built (§11), 11/18:**
	- **business_logic:** `RemoveFlyer` returns the same `FlyerResult` as `AttachFlyer`, and touches only the dates still waiting. **Tests:** `RemovingTheFlyerTakesItOffThePendingDatesOnly` (a second removal is harmless) and `RemovingTheFlyerChecksWhoseItIsAndThatSomethingIsWaiting` (someone else's, an unknown id, nothing pending).
	- **endpoint:** the `DELETE` route sits in `event_submissions.cpp` beside the `POST`, so it shares that TU's anchor. A refusal is a 400 with `error_code` `NOTHING_PENDING`, as for attaching.
	- **client:** `EventSubmissionsAccess.removeFlyer`, over HTTP and in the mock, plus the three buttons on "My submissions". From §13.3, adding or replacing a flyer here goes through honuware's prepare step, as the submit form does.

### 12. Photo-removal requests reach the right people (OQ11, OQ19)

As OQ19 decided (all four recommendations):
- [x] 12.1 **The Curator role:** seeded in `create_app_tables.cpp` beside `PopulateAppPermissions`, holding `manage_events`, with both looked up by name like the Administrator grant.
	- It's seed data, not a new table, so the three-place rule doesn't apply. `create_app_tables_test.cpp` covers it, and §7.3's live check gains a line for it.
	- **Tests.**
- [x] 12.2 **business_logic:** `PhotoRemovalRequestHelper`. It takes the page, title, photo description, optional message and optional reply address, and emails the site address plus everyone holding `manage_events`, through `::Mail::LoadSenderAddress` and `FormatString`, with `NormalizeCrLf` on the HTML. **Tests:** who it goes to, a curator without the permission left out, and the reply address shown only when given.
- [x] 12.3 **endpoint:** `POST /api/photo_removal_requests`, public, with the honeypot and a rate limit per visitor. **Tests:** sent, the honeypot quietly accepted and dropped, the rate limit, and length limits.
- [x] 12.4 **client:** "Ask us to remove a photo" and "Ask us to remove this photo" open a short form in the gallery and the viewer instead of a mailto. It confirms when sent. `SiteConfig.contactEmail` stays as the fallback shown if sending fails. **Specs.**
- [x] 12.5 **client:** open the admin Events and Event settings pages to `manage_events` holders.
	- A CF guard on those routes replaces `AdminGuard` there. The dashboard shows a curator only the tiles they can use. Users and Roles stay admin-only.
	- Check whether `@honuware/ui/crud`'s table picker already hides tables a curator can't reach. If it doesn't, that's a web-components change for this phase.
	- **Specs.**
- **As built (§12), 11/18:**
	- **12.1:** the role is named `curator` (`DbSchema::kRoleNameCurator`) and holds only `manage_events`. Both are looked up by name, so a re-seed is harmless. **Test:** `SeedsTheCuratorRoleHoldingOnlyManageEvents`. §7.3's live check gains its line.
	- **12.2:** `business_logic/events/photo_removal_request_helper.{h,cpp}`. **Nothing is stored; the email is the request.**
		- **The checks** trim everything and name the wrong field: `REQUIRED`, `TOO_LONG`, `BAD_PAGE`, `BAD_EMAIL`. The limits are page path 300, page title 200, photo 500, message 2000, reply address 254.
		- **The page link** is built from the site's own address plus a site-relative path, so it can never point off-site. Everything the visitor typed is HTML-escaped.
		- **The rate limit is a decision beyond the plan:** 3 an hour per visitor (by client IP), *and* 20 an hour for the whole site, so a flood from many addresses still can't fill the curators' inboxes.
		- **Recipients:** each gets their own email, so no one sees anyone else's address. If none can be sent, the result is `NOT_SENT`, and the form offers the contact address.
		- **Tests** (5): the checks; escaping and the on-site link; the two rate limits; the site plus everyone holding `manage_events`; one email each.
	- **12.3:** `endpoints/photo_removal_requests.{h,cpp}`, anchored in `RegisterAllEndpoints()`. **Tests** (5): emailed to the site and its curators; past the CSRF guard here and nowhere else; the honeypot; 3 an hour; a 400 naming the field.
		- **A finding along the way, the CSRF guard:** honuware's `CsrfGuard` refuses any `POST` without the `csrft` token cookie. A visitor who never signed in has no such cookie, so a public form could never be sent. honuware's `CsrfGuard` gained `AllowWithoutToken(path)` / `IsAllowedWithoutToken(url)`: an app names the exact public routes that serve people who aren't signed in. CF names only this one. The rest of the site keeps the check (3 new `CsrfGuardTest`s in honuware).
	- **12.4:** `shared/components/photo-removal-form` (`app-photo-removal-form`), opened from the gallery and the viewer.
		- It checks the server's rules first, with the same messages. From the gallery, where no photo is chosen, the message has to say which one.
		- The `website` honeypot is hidden from people.
		- If sending fails, the form says so and shows the site's address, with the request written out to copy.
		- `CommunityAccess.requestPhotoRemoval`, over HTTP and in the mock.
	- **12.5:**
		- `core/guards/manage-events.guard.ts`: `canManageEvents` (an admin, or anyone holding `manage_events`) and the `/admin` guard. Anyone else is sent where `AdminGuard` sends them.
		- `AdminGuard` stays on Users, Roles and (from §13.3) Site settings. The dashboard shows a curator Events and Manage Data, and the latter reads "Browse and edit the events tables".
		- **The table picker needed no change:** honuware's `get_db_schema` already cuts `root_tables` to what the caller's permissions reach, so a curator's picker lists only the events tables.
		- The header's Admin link shows for curators too.
		- **Specs:** the guard (a curator in and kept out, an admin everywhere, a member and a signed-out visitor), the dashboard per role, and the header.
		- **OQ24 (11/20):** guide curators (`manage_guide`) now get in too.
			- `/admin`'s door is `AdminAreaGuard` (an admin, or `manage_events`, or `manage_guide`). `ManageEventsGuard` moved onto the two Events routes.
			- A guide curator sees Manage Data alone, listing the guide tables.
			- **Specs:**
				- `admin-area.guard.spec.ts` covers every kind of person through the real routes: events curator, guide curator, both, admin, member, signed out.
				- The dashboard and the header cover a guide curator, and someone curating both.
			- **Server test:** `EachCuratorsTableListHoldsOnlyTheirOwnTables`. Each kind of curator's `root_tables` holds their own tables and the shared `venues`. Neither sees the other's private tables, or `people` and `roles`.

### 13. A configurable upload limit, shared with knottyyoga (OQ13, OQ21)

A cross-repo bump point: Mason pushes honuware and publishes `@honuware/ui`, then I re-pin CF and verify a fresh pinned build, then Mason commits. knottyyoga follows the same way.
- [x] 13.1 **honuware server:** `GET`/`PUT /api/admin/image_upload_settings` under `admin_portal`, for `image_max_upload_bytes`, with the default and allowed range (10 MB; 1–50 MB per OQ21), refusing values out of range. ✅ 11/18. It's in honuware `0b249be`, which CF pins from 11/19.
	- A read that any signed-in session can call returns the limit, so the upload control knows it.
	- The upload routes' existing check stays the last word.
	- **Tests.**
- [x] 13.2 **honuware web components:** ✅ 11/18. Published as `@honuware/ui` 0.4.0 on 11/19, which CF uses.
	- `hw-photo-upload` reads the limit. It says before uploading when a file is still too big after shrinking, instead of waiting for the server's refusal.
	- Its "prepare the image" step becomes a public export, so an app can shrink a file it sends itself.
	- A settings card for the limit, for an app to put on its settings page.
	- **Specs.**
- [x] 13.3 **CF:** a new admin-only "Site settings" page (OQ21) on the admin dashboard, holding the card. The submit form's flyer and §11's "Add a flyer" go through honuware's prepare step, so a flyer is shrunk to at most 4096 px and sent as a JPEG, as a curator's upload is. **Specs.** ✅ 11/18. Re-checked against the published 0.4.0 on 11/19.
- [ ] 13.4 **knottyyoga:** the card on its settings page after the bump. That's knottyyoga's own change and gate.
- **As built (§13), 11/18:**
	- **13.1, honuware server:**
		- `platform/business_logic/images/image_upload_limit.{h,cpp}` (`Images`): `LoadImageUploadLimit`, `ValidateImageUploadLimit` and `SaveImageUploadLimit`. The constants are a 10 MB default, 1 MB minimum and 50 MB maximum. The load returns what's stored, which is what the upload routes enforce, even if it was set outside the range by hand. A value that's unset, or isn't a positive whole number, reads as the default.
		- `platform/endpoints/image_upload_settings.{h,cpp}`, anchored in `RegisterFrameworkEndpoints()`:
			- `GET /api/image_upload_limit`, for anyone signed in, returns `{max_upload_bytes}`;
			- `GET`/`PUT /api/admin/image_upload_settings`, under `admin_portal`, return `{max_upload_bytes, default_bytes, minimum_bytes, maximum_bytes}`.
			- A value out of range is a 400 naming `max_upload_bytes`; a body that isn't JSON is a 400, not a 500.
		- `kPermissionAdminPortal` moved into honuware's `platform/db_schema/permissions.h`. CF's two file-local copies don't clash with it.
		- **Tests:** `ImageUploadLimitTest` (5) and `ImageUploadSettingsTest` (7). The last of these shows the existing upload route refusing a 1 MB + 1 byte image once 1 MB is saved: the server stays the last word.
	- **13.2, `@honuware/ui`:**
		- **access:** an `ImageUploadSettingsAccess` seam of its own (`HONUWARE_IMAGE_UPLOAD_SETTINGS_ACCESS`, HTTP by default), with `getUploadLimit`, `getSettings` and `saveSettings`.
			- *Why not add to `PhotoAccess`:* knottyyoga's `ServerAccess` extends `PhotoAccess`, so new methods there would break its build on upgrade. A separate seam is additive.
		- **photos:** `prepareImageForUpload(file, maxDimension)` is exported, with `MAX_UPLOAD_IMAGE_DIMENSION` (4096), `checkImageUploadSize`, `ImageTooLargeError` and `formatMegabytes`.
			- **`hw-photo-upload`:**
				- It looks the limit up only when a file is chosen. If the lookup fails or no seam is provided, it doesn't block, and the server still decides.
				- A file still too big after shrinking shows, in the control: "This image is 12 MB even after shrinking, and the limit is 10 MB. Try a smaller image." The size rounds *up*, so a file just over never reads "10 MB … limit 10 MB".
				- The new public `preparePendingPhoto()` lets a page get the prepared bytes and send them itself.
			- **`hw-image-upload-settings`:** the card. It takes whole megabytes from 1 to 50, has "Use the default", and has messages for saved, a save failure, and a load failure with Try again.
		- **testing:** `MockImageUploadSettingsAccess`. `provideHonuwareAccessMock` supplies it, so CF's offline demo has the card working.
		- **Specs:** web components 550 SUCCESS, lint clean, `npm run build` clean.
		- **Both READMEs** describe the seam, the prepare step and the card.
	- **13.3, CF:**
		- **The Site settings page:** `pages/admin/site-settings` at `/admin/settings`, under `AdminGuard`, holding `<hw-image-upload-settings>`. It has an admin-only dashboard tile.
		- **The flyer:** the submit form and "My submissions" call `preparePendingPhoto()` *before* anything is sent.
			- A flyer that's too big, or can't be read, is refused at its field, and nothing is submitted.
			- Otherwise the prepared JPEG is attached, so the flyer type sent is always `jpeg`.
			- *This is a decision beyond the plan:* checking first means a too-big flyer can't leave a submission without its flyer.
		- **Specs:** the page (it shows the limit, saves through the card, and links back), the dashboard's admin tiles, the guard keeping a curator out of `/admin/settings`, and too-big and unreadable flyers on both pages.
	- **The bump point (11/19):**
		- honuware was pushed as `0b249be` ("CM3 Phase 2 — Schema check, upload limit, CSRF exemption"), and the web components as `@honuware/ui` 0.4.0 (tag `v0.4.0` at `1c4eb96`).
		- CF's `GIT_TAG` is now `0b249be`, and `package.json` asks for `0.4.0`. `npm install` replaced the local overlay with the published package, and the lock file changed only that entry.
		- *The first publish run stopped at the CI's version check:* the release commit hadn't bumped `projects/honuware-ui/package.json`, so nothing went out. A second commit bumped it, and the tag was moved there.

### 14. Leave the offline demo out of production builds (OQ16)

In its own commit.
- [x] 14.1 `app.config.ts` gets its access providers from one small file (for example `core/app-access-providers.ts`, which uses HTTP only). The `local` configuration's `fileReplacements` swaps in a mock version. `environment.useMock` then only selects the build configuration. The specs aren't affected, since they import the mocks directly. ✅ 11/18
	- **As built:**
		- `core/app-access-providers.ts`: `provideAppAccess()` returns honuware's HTTP access plus CF's four HTTP seams.
		- `core/app-access-providers.mock.ts`: the same export, with the mock access, the demo photos, `DemoPhotoUrlBuilder` and CF's four mocks.
		- **The guide's access had to be swapped too,** because the guide routes chose mock or HTTP the same way. `core/guide-access/guide-access-provider.ts` and its `.mock.ts` give `provideGuideAppAccess()`. `guide.routes.ts` now exports `guideRoutes(guideAccess)`, so the guide's test harness passes the mock in directly.
		- `angular.json`'s `local` configuration replaces both files.
- [x] 14.2 **Measure before and after**, the same way as OQ16 (from the build's stats file). Expected: about 44 kB less up front (932 → about 888 kB), and 17.6 kB less in the guide chunk. Then confirm `ng build` for all three configurations, and that `ng serve` in the local configuration still runs the demo. ✅ 11/18
	- **Production, before → after** (both measured with 0.3.1 of the library):
		- **initial:** 944.72 → 896.60 kB raw (−48.1). *The "before" is higher than the 932 kB in §7.2 because §9–12 and §15 came first.*
		- **main:** 176.6 → 129.05 kB;
		- **the guide chunk:** 148.3 → 130.5 kB (−17.8);
		- **mock code in production:** 64.6 kB → 0.
	- All three configurations build. The `local` build has the mocks: the demo's events, gallery and video, and the guide.
	- *The offline demo under `ng serve` (local) is for Mason to try by eye.* It's in §16.4.

### 15. The demo's video (OQ18)

- [x] 15.1 `MOCK_VIDEO.id` becomes `lVhJ_A8XUgc`, and the CRUD seed's `video_url` becomes `https://www.youtube.com/watch?v=lVhJ_A8XUgc`. Specs that name `demoVideo01` are updated. The §5.1 note about the made-up id goes. ✅ 11/18

### 16. Gates for Phase 2

- [x] 16.1 Docker plus Windows green after each server slice ("Windows: N passed. Linux: M passed."), with the floor raised at the end. ✅ 11/19
	- **Strict pinned to honuware `0b249be` (fresh clone, no override): Windows 2572 passed. Linux 2572 passed.** The floor is raised from 2515 to 2562 (`server/docker/build_and_test.sh`).
	- **11/18, co-dev (CF built against the local honuware tree): Windows 2572 passed. Linux 2572 passed** (was 2525). honuware's own Linux gate for the new honuware tests: 58 passed.
	- The floor stayed 2515 until the re-pin, because 20 of those tests are honuware's new ones (CsrfGuard 3, SchemaCheck 5, ImageUploadLimit 5, ImageUploadSettings 7), which a build pinned to `838045f` didn't have.
- [x] 16.2 Vitest green with the count, and `ng build` for local, development and production. §14's bundle change is recorded. ✅ 11/18
	- **Vitest: 620 passed in 74 files** (was 562 in 71), against the local `@honuware/ui` build.
	- `ng build` passes for production, development and local.
	- **Production now:** initial 904.28 kB raw (209.06 kB transferred), main 129.12 kB, guide chunk 130.53 kB, mock code 0. §14's numbers are in §14.2.
		- **Since §14, initial grew 7.7 kB:** the new library's `photos` entry went from 7.6 to 14.2 kB up front, because it now holds the prepare step, the limit check and the settings card.
		- It's up front because the account pages (`/account`, `/account/submissions`) load eagerly and use `photos`. Lazy-loading them would take all 14.2 kB out of the initial bundle. That's a small follow-up for any later bucket; it isn't planned here.
		- The 500 kB budget warning was already there.
- [x] 16.3 §13's bump point verified with a fresh pinned build. ✅ 11/19
	- **C++:** Windows 2572 passed, Linux 2572 passed, both fresh at `0b249be` (§16.1). The Linux log says "pinned SHA". The Windows tree's override is empty, and its `_deps/honuware-src` is at `0b249be`.
	- **Client, on the published `@honuware/ui` 0.4.0:** Vitest 620 passed in 74 files. `ng build` passes for production, development and local. Production's initial bundle is 904.28 kB, the same as against the local build.
- [ ] 16.4 The manual loop (§7.4) gains:
	- the Edit Event list and search;
	- editing a gallery photo's words;
	- adding, replacing and removing a flyer from "My submissions";
	- a removal request arriving in each inbox;
	- a curator who isn't an admin, using the Events pages and kept out of Users and Roles;
	- a guide curator (OQ24), seeing only Manage Data with the guide tables, and kept out of the Events pages;
	- the video showing as YouTube's player, with its still, on an event and a series. Check one Vimeo video too (§17.2);
	- a fresh `--populate_sample_data` showing photos, a gallery and the video, after §18;
	- a server started on an old database printing §8.2's warning.

### 17. The video tile showed the event's photo, not the video (Mason, 11/18)

Mason: *"I put a video in the event. When I click on the event, it shows the main event photo stretched as the representation of the video instead of a frame from youtube for the video. When I click on it, it plays the right video but this is misleading."*

**Why it looked that way:** OQ4 says nothing loads from YouTube until someone presses play, so the tile borrowed the page's own photo.
- The server keeps a photo's proportions when it scales (`bounding_rect.cpp`), so a tall flyer stays tall.
- The tile then cropped it to 16:9 (`object-fit: cover`), which zooms far into its middle, and darkened it to 60%.
- And it was never a picture of the video.

- [x] 17.1 **client:** the tile no longer shows the page's photo. It's the gradient, the play button and "Plays from YouTube". ✅ 11/18
	- `VideoEmbed` lost its `posterUrl` input, and the event and series pages stopped passing the hero.
	- Specs: the tile has no image; the event page's hero still appears exactly once.
	- Vitest: 563 passed in 71 files. Prettier clean.
- [x] 17.2 **The video's real still: YouTube's or Vimeo's own player, on the page from the start** (OQ23: "the full player"). ✅ 11/18
	- **`VideoEmbed`:**
		- The tile, the play button, "Plays from …" and the press handling are gone.
		- It's now one frame, titled `Video: <title>`, with fullscreen allowed.
		- It has `loading="lazy"`, so a visitor who never scrolls near it never loads the player.
		- The address is still built only from `{provider, id}` and trusted once per video.
	- **`videoEmbedUrl`:** no autoplay any more, since nobody has pressed anything. It's now `youtube-nocookie.com/embed/<id>` and `player.vimeo.com/video/<id>?dnt=1`.
	- **Specs:**
		- the player is there at once, titled, lazy and not autoplaying, with no stand-in image or button;
		- Vimeo's do-not-track address;
		- trusted once;
		- nothing from a bad id;
		- a new video moves the one player;
		- the event and series pages show the player between the hero and the gallery.
	- **Gates:**
		- Vitest: 562 passed in 71 files. That's one fewer, because the press test went with the tile.
		- `ng build` (production) is clean. The events chunk went from 177.76 to 175.37 kB.
	- **No server work, and nothing for PL2:** `frame-src` already names both players, and `img-src` is unchanged.
	- *The server and browser thumbnail versions planned here were dropped. Their costs and benefits stay under OQ23.*

### 18. Sample data that shows off CM3 (OQ22)

- [x] 18.1 **business_logic:** `PopulateSampleData` adds, all made up:
	- **Main photos** on a few sample events and one series.
		- Generated SVG art: shapes and colour, no people, so there's no consent question, in the style of the offline demo's `demo-photo-art.ts`.
		- Stored with `Images::ImageHelper::UploadAndAssociatePhoto(transaction, table, id, bytes, "svg")`, the same path a real upload takes.
	- **A three-photo gallery** on one event.
		- Rows in `event_gallery_photos` with alt text, a caption and the credit "Sample art".
		- Consent recorded against the reviewer `PopulateSampleData` already receives.
		- If there's no administrator (id 0), the gallery is skipped and reported. `consent_confirmed_by_person_id` is a plain required id, not a link, so the database would accept 0. But that would record consent from nobody.
	- **The OQ18 video**, `https://www.youtube.com/watch?v=lVhJ_A8XUgc`, on one event and one series.
	- **Re-running** leaves existing photos, gallery rows and videos alone and never adds a second gallery, as it does for the listings. `SampleDataReport` gains photo, gallery and video counts.
	- **Tests** in `populate_sample_data_test.cpp`:
		- the first run adds each kind;
		- a second run adds nothing;
		- the video passes the column's CHECK;
		- the gallery is public once its event is approved;
		- with no administrator, the gallery is skipped and reported.
- [x] 18.2 **README:** step 7's sample output gains the new counts line.
- **As built (§18), 11/18:**
	- `SampleDataReport` gains `photosAdded`, `galleryPhotosAdded`, `videosAdded` and `mediaAlreadyThere`. A re-run counts each photo, the gallery and each video once in `mediaAlreadyThere` instead of adding anything. It also gains `gallerySkippedNoReviewer`.
	- `kSampleVideoUrl` is the OQ18 video, the same one the offline demo shows.
	- **Tests** (3 new): `AddsPhotosAGalleryAndTheVideoThatThePagesShow` (each kind, the video passing its CHECK, and the gallery public once its event is approved), `ARerunLeavesThePhotosGalleryAndVideosAlone`, and `WithNoAdministratorTheGalleryIsSkippedAndSaysSo`.
	- The database helper prints the new counts. README step 7 shows the line.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Who can add pictures and video?** EV3 decided curators add the flyer on review, because a stranger's photo raises the consent and no-outing questions this bucket answers, and invites people to use us as free image hosting.
	- *Rec: submitters may give a video link* (it points at something already public, and a curator sees it before anyone else does).
	- *Rec: the flyer waits.* Curators add the main photo and gallery in this bucket. A submitter's flyer upload is a follow-up that needs H15 first, so a stranger's image is never public before a curator approves it.
	- *Alternative:* do H15 now and let submitters attach one flyer.
	- Mason- Let's do H15 now.
	- **Decided (11/4): H15 now, plus a video link and one flyer from submitters.** Built as §0.1, §3.4–3.6 and §4.3–4.4. The flyer goes on every date still waiting and stays private until a curator approves the date.
2. **When an event has no photo of its own, should it borrow one?** First from its series, then from its venue. A weekly night then looks right without someone uploading to every date.
	- *Rec: yes.* It turns on photos for series.
	- Mason- I like this recommendation.
	- **Decided (11/4)**, as built: event → series → venue, and series → venue.
3. **Which video sites?**
	- *Rec: YouTube and Vimeo,* played from YouTube's no-cookie address and Vimeo's do-not-track player.
	- Instagram, TikTok and Facebook embeds only work by loading their own scripts on our page, which the "never trackers" rule rules out. Their links can still go in the event's Link field.
	- Mason- YouTube and Vimeo are fine.
	- **Decided (11/4)**, as built.
4. **Click-to-play, or load the player right away?**
	- *Rec: click-to-play.* Nothing from YouTube loads, and nobody is tracked, until someone chooses to watch. It also keeps pages fast.
	- Mason- This sounds good.
	- **Decided (11/4): click-to-play.** This is client work (§5.4).
	- **Reversed (11/18) by OQ23:** the page now loads the provider's player from the start, because its still and one-tap play are the better experience.
5. **How many gallery photos per event or series?**
	- *Rec: 12,* enough to market a night without becoming an archive. Easy to raise later.
	- Mason- This sounds good but let's make this configurable via the admin portal.
	- **Decided (11/4): 12 by default, and an admin can set it from 1 to 60.** The server side is built (§3.3, §4.2). The control on the Event settings page is §5.6. Lowering the cap never removes photos.
6. **Record who vouched for consent on each photo?**
	- *Rec: yes.* A required "everyone recognizable agreed" box on upload, stored with who ticked it and when. When a removal request comes in, we can see who added the photo and on what understanding.
	- Mason- This sounds good to me.
	- **Decided (11/4)**, as built. The server stamps who and when from the curator's session. Nobody types it in.
7. **Galleries on series pages too?**
	- *Rec: yes, and the same for video.* A recurring night's pictures belong to the series, because each dated instance drops off the calendar once it's past. An event shows its own gallery, or else its series'.
	- Mason- This sounds good.
	- **Decided (11/4)**, as built: series have their own gallery and video, and an event without its own shows its series'.
8. **Where does the removal-request link go before CM1 exists?**
	- *Rec: the site's contact address,* with the photo's page in the subject, and a curator acts within a day. CM1's report flag replaces it.
	- Mason- Your recommendation is fine.
	- **Decided (11/4): the site's contact address.** This is client work (§5.3).
	- *Built (11/18):* `SiteConfig.contactEmail`, set to `community.finder.seattle@gmail.com`, the site's sending address. OQ11 widens this: requests go to that address plus everyone who holds `manage_events`, sent by the server (Phase 2 §12, details in OQ19).
9. **Which stream owns it, and when?** The outline puts CM3 in Stream D, gated on traction. But almost every file is Stream A's (the events tables, the event pages, the curator editor, the submit form), and Mason wants it for marketing now.
	- *Rec: run it in Stream A after EV3's gates, with Levi or Mason as owner.* Update the outline's Stream D row to point here.
	- Mason- Sounds good.
	- **Decided (11/4): Stream A, with Levi or Mason as owner.** The outline's Stream D row now points here.
10. **Should signed-out visitors get bigger photos?** honuware caps every photo a signed-out visitor gets at 800×800 (`image_max_public_scaled_width` and `_height`). So the 1200×630 hero already comes out at 800×420, and the 1600×1200 full view would come out at 800×600. *(Corrected 11/18: this said signed-in people get the full size. They don't: the cap applies to everyone.)*
	- *Rec: raise CF's cap to 1600×1200* in this app's database seed. The full view then looks sharp on a phone or laptop. The cost is more stored scaled copies, which EV4's cleanup job already prunes.
	- *Alternative:* keep 800×800 and use the smaller sizes everywhere.
	- Mason- I'll go with your recommendation.
	- **Decided (11/18): CF raises the cap to 1600×1200** in its seed. honuware's defaults stay 800×800, so knottyyoga is unchanged. Built the same day (§1.5).

*Added 11/18 after the client build (§5). OQ11–15 and OQ18 are about CM3. OQ16 and OQ17 came up while measuring the bundle and reach beyond it. All eight were answered on 11/18, and the work they call for is Phase 2.*

11. **Which address should photo-removal requests go to?**
	- **What it does:** every gallery on an event or series page has an "Ask us to remove a photo" link, and the full-size viewer has "Ask us to remove this photo" for the picture on screen.
		- Either link opens the visitor's own email app, addressed to us, with the subject "Photo removal: Leather Night" and the page's address in the body. The viewer's link also says which photo, for example "Photo 2 of 4 ("Midnight on the floor.")".
		- The photo policy promises a photo comes down within a day of a request, so a person has to read this inbox every day.
	- **What I did:** OQ8 said "the site's contact address". The only address the app knows is the one it sends mail from, `community.finder.seattle@gmail.com`, so I used that (`SiteConfig.contactEmail` in `ui/src/app/core/services/site-config.service.ts`).
	- **Why I'm asking:** I don't know if anyone reads that inbox. It's the sending address, and a sending address often isn't watched.
	- *Rec: keep it, if someone checks that inbox at least daily.*
	- *Alternative:* name another address, for example a shared inbox for curators. It's one line in the client, so a change later needs a deploy. CM1's report flag will replace this link eventually.
	- **What I need from you:** "keep it", or the address to use.
	- Mason- I would like to use this site email address AND send email to all of the email addresses for the admin users. If there is another, lesser permission that allows even editing but not full admin access, that would be useful. Maybe we can add that role and a corresponding permission. Give the permission to this new role and admin.
	- **Decided (11/18): requests go to the site address and to everyone who holds the permission. That's admins, plus a new role that can edit without full admin access.** Planned as Phase 2 §12.
		- **The lesser permission already exists:** `manage_events` lets someone curate events, galleries, series, venues and the vocabularies, but not manage users or roles. Only the Administrator role holds it today, and the `/admin` pages let in admins only (`AdminGuard`), so a non-admin can't use it yet.
		- **Sending to admins needs the server to send the mail.** A mailto link to every admin would print their addresses into the public page, where address-harvesting bots read them. So the link becomes a short form on the page, and the server emails everyone.
		- The details are **OQ19**.

12. **Should curators be able to fix a gallery photo's words after adding it?**
	- **Now:** the event editor's gallery lists each photo's alt text, caption and credit, but can't change them. To fix a typo, a curator has to:
		- remove the photo and add it again, which means uploading it again and re-ticking the consent box; or
		- find the row in the generic admin (the "Gallery Photos" table) and edit it there.
	- **Why it matters:** the policy says captions name performers or organizers only with their agreement. If a DJ asks to have their name taken off a caption, a curator should be able to do it in a few seconds without touching the photo.
	- *Rec: add an "Edit" button to each row.* It would open the three fields in place, with the same length limits and alt text still required, and save through generic CRUD.
		- The server already allows this edit, and the database refuses blank alt text even on an edit.
		- It's a client-only change to `event-gallery-editor.*`, with specs.
	- *Alternative:* leave it as it is until a curator asks. §5.6 only asked for the words to be shown.
	- **What I need from you:** "add it" or "leave it".
	- Mason- Delete / recreate is annoying. Let's allow editing of these. I general, anything that can be created should have full CRUD support. Please add a memory for that.
	- **Decided (11/18): add editing** (Phase 2 §10), **and the rule from now on: anything that can be created gets full CRUD** (create, read, update, delete). It's saved as a memory.
		- Checked against what CM3 creates: the video and the main photo already have it. The gallery photo's words get it in §10. The flyer gets replace and remove in §11.

13. **Should a big flyer be shrunk in the browser before it's sent?**
	- **Now:** the submit form sends the flyer exactly as picked. The server refuses anything over 10 MB (`image_max_upload_bytes`).
		- Most flyers fit: a phone photo of a paper flyer is usually 2–5 MB.
		- Some don't: photos from 48-megapixel phone cameras, and PNG exports of designed flyers, can go over 10 MB.
		- When one doesn't fit, the person sees "Your event was sent, but its flyer couldn't be added. The flyer is too large." The submission is in, but there's no way to retry from that page (see OQ14).
	- **Why curators don't hit this:** `hw-photo-upload` shrinks every image to at most 4096 px and re-saves it as a JPEG before upload. That step is private inside `@honuware/ui`, so CF's flyer code can't call it.
	- **The options:**
		1. **A web-components change:** make that "prepare the image" step public, for example as an exported `prepareImageForUpload(file)` function. You'd make and publish the change in honuware-web-components, then CF shrinks flyers exactly as curators' uploads are shrunk. Flyers would always go up as JPEGs.
		2. **Copy about 40 lines of resizing code into CF.** That duplicates honuware's code, which the house rules discourage.
		3. **Leave it.** Most flyers fit.
	- *Rec: option 1,* because it keeps one copy of the code and knottyyoga could use it too. Until it's published, option 3.
	- **What I need from you:** which option, and for option 1, whether you'll make the web-components change or want me to draft it.
	- Mason- I like having a limit but this should be configurable in the site settings (and defaul to 10MB). We can change this in web components as well and have knotty yoga get the same feature.
	- **Decided (11/18): keep a limit, default 10 MB, that an admin can change in site settings. Build it in honuware** (server and web components), so knottyyoga gets it too. As part of that, the web components' "prepare the image" step becomes public (option 1), so the flyer is shrunk like every other upload. Planned as Phase 2 §13.
		- **Today:** the limit already is a setting: `image_max_upload_bytes` in `config_secrets`, 10485760 bytes (10 MB) by default, checked by honuware's upload routes. But no screen in honuware or CF lets anyone change it, and the upload control doesn't know it.
		- **CF has no "site settings" page yet,** only Event settings. Where the setting goes, and its allowed range, is **OQ21**.

14. **Should someone be able to add or replace a flyer after sending, from "My submissions"?**
	- **Now:** a flyer can only be attached in the moment the form is sent. If that upload fails (too large, or a dropped connection), nothing else on the site lets them add one. The form also says we don't take events by email, so they can't send it any other way.
	- **The server already supports it:** the flyer route takes a flyer for any of your own submissions that still has a date waiting for review, and puts it on every waiting date. Sending another one replaces it.
	- *Rec: yes.* Add an "Add a flyer" button to each submission on "My submissions" while any of its dates is waiting, which becomes "Replace the flyer" once one is attached. The failed-flyer message on the thanks page would point there.
		- It's client-only, and small.
		- It also covers the gap in OQ15.
	- *Alternative:* leave it; a curator can add a photo when they review.
	- **What I need from you:** "yes" or "leave it".
	- Mason- Yes
	- **Decided (11/18): yes.** Planned as Phase 2 §11, together with "Remove the flyer", per the full-CRUD rule in OQ12.

15. **Keep the chosen flyer when submissions pause mid-form?**
	- **Now:** if curators pause submissions while someone has the form open, pressing Send shows the "We're not taking event submissions right now" notice. After "Check again", the form comes back with everything typed, but the flyer they picked is gone and has to be picked again. The picker is rebuilt along with the form.
	- *Rec: keep it.* The page holds on to the chosen file and gives it back to the new picker. This is a small fix, with a spec.
	- *Alternative:* leave it. A pause is rare, and the person can pick the file again.
	- **What I need from you:** "agreed" or "leave it".
	- Mason- I say leave it. This is extreme edge case to me that isn't worth complicating the system for.
	- **Decided (11/18): leave it.** Once §11 is built, someone who loses their flyer this way can add it from "My submissions".

16. **Should production builds leave out the offline demo?**
	- **Background:** `app.config.ts` decides at runtime, from `environment.useMock`, whether to use the offline mock or the real server. So every production visitor downloads all the mock code and demo data, though the site never runs it.
	- **Measured 11/18:**
		- 61.7 kB of production JavaScript is mock code: 44.0 kB of the 168.4 kB `main` file that every page loads first, and 17.6 kB in the guide pages' chunk;
		- the biggest pieces are the guide's demo data (13.1 kB), the events demo data (9.3 kB) and the submissions mock (8.8 kB);
		- CM3 added 2.4 kB of new mock files, plus its additions to existing ones.
	- **The fix:** a build-time swap. `app.config.ts` gets its providers from one small file, and the `local` configuration replaces that file with the mock version, using Angular's `fileReplacements`. Production then contains only the real-server code. The tests aren't affected, because they import the mocks directly.
		- The up-front bundle would drop from about 932 kB to about 888 kB.
		- It wouldn't clear the 500 kB budget warning, which was there long before CM3.
	- *Rec: yes, as its own small change after CM3,* so this bucket's diff stays about photos.
	- *Alternative:* leave it. 44 kB is noticeable on a slow phone, but not serious.
	- **What I need from you:** "yes" (and whether now or later) or "leave it".
	- Mason- Yes, let's go with your recommendation.
	- **Decided (11/18): yes, as its own small change after CM3.** Planned as Phase 2 §14, in its own commit, so the photo work's diff stays about photos.

17. **Are the three seeded accounts still using the password `changeme`? (Outside CM3, but it's a security question.)**
	- **What I found while measuring OQ16:** `create_database.cpp` (lines 108–113) creates your, Levi's and Caleb's accounts with the password `changeme`.
		- The seed doesn't make anyone change it when they first sign in.
		- The repository's README also mentions the word.
		- The offline demo's login uses the same password, so it's also in the production JavaScript (OQ16). That copy only works against the in-memory demo, though.
	- **The risk:** if any live database was seeded this way and those passwords were never changed, anyone who guesses or reads `changeme` can sign in as an administrator.
	- *Rec:*
		1. Check now that all three accounts on every live database have their own passwords.
		2. Change the seed so these accounts must change their password on first sign-in. honuware already has a must-change-password flag, so this is a small server change with a test.
		3. Change the demo login's password, whether or not OQ16 goes ahead.
	- *Alternative:* the seed reads the starting passwords from environment variables, as the scheduler service account's already does (`SCHEDULER_SERVICE_ACCOUNT_PASSWORD`).
	- **What I need from you:**
		- whether the live accounts are still on `changeme`;
		- which seed fix you want;
		- whether this belongs in CM3 or in a separate security item.
	- Mason- The site isn't live. Please leave this as is. We will change those passwords during deploy.
	- **Decided (11/18): no change.** The site isn't live, and Mason will change the three passwords during deployment. The demo login's `changeme` stays as well.

18. **What should the offline demo's video be?**
	- **Now:** in the local build, Leather Night has a video. Its id, `demoVideo01`, is made up. The play tile works, but pressing it opens YouTube's player showing "Video unavailable". The browser still has internet in the local build, so a real id would play.
	- *Rec: give me a YouTube or Vimeo link you're happy to show in demos,* ideally one from a Seattle venue or your own. It goes only into the demo data and never touches production.
	- *Alternative:* keep the placeholder. The demo then shows the tile and the player opening, but not a playing video.
	- **What I need from you:** a link, or "keep the placeholder".
	- Mason- Use this: https://www.youtube.com/watch?v=lVhJ_A8XUgc
	- **Decided (11/18): the demo uses YouTube video `lVhJ_A8XUgc`.** Planned as Phase 2 §15.

*Added 11/18 from the answers above and Mason's new work. They shape Phase 2 §8, §12, §13, §17 and §18. OQ19–23 were all answered on 11/18. OQ23 settled on the full player after a follow-up discussion.*

19. **How should photo-removal requests and the new curator role work? (Follows OQ11.)** Four parts, each with a recommendation, so "agreed" covers all four.
	1. **How a request is sent.**
		- *Rec: a short form on the page.* "Ask us to remove this photo" opens it in place, with a box for a message. The server emails the request to everyone in part 2, and the visitor sees "Thanks, we'll take it down within a day."
		- The form gets the submit form's honeypot plus a rate limit, so a bot can't use it to flood the inbox.
		- *Alternative:* keep the mailto link to the site address only, and have that Gmail inbox auto-forward to the admins. That needs no code, but it fixes the recipient list by hand outside the app, so a new curator wouldn't get the requests automatically.
	2. **Who gets the email.**
		- *Rec: the site address, plus everyone who holds `manage_events`.* That's the permission it takes to remove a photo, so whoever gets the request can act on it. Admins have it through their role, and the new role in part 3 would have it too.
		- *Alternative:* a separate `receive_photo_removal_requests` permission, so who can edit and who gets the email can be set apart. That's more to manage, and no case needs it yet.
	3. **The new role.**
		- *Rec: a "Curator" role holding only `manage_events`.*
			- A curator can use the admin Events page (review queue, New Event, Edit Event and galleries) and the Event settings page, whose server routes check `manage_events`. In the generic data editor, they reach only the events tables.
			- A curator can't manage users or roles, which need `admin_portal`, and can't touch the guide, which needs `manage_guide`.
			- Today `AdminGuard` lets only admins into `/admin`, so the Events pages need their own permission check, and the dashboard should show a curator only the tiles they can use.
			- Mason assigns the role on the Roles & Permissions page.
		- *Alternative:* the role also gets `manage_guide`, so one curator role covers events and the guide.
	4. **What the visitor must give.**
		- *Rec: no sign-in.* The person in the photo may not have an account. The message is optional, and so is an email address to reply to. The request carries the page, the photo's number and its caption automatically.
		- *Alternative:* require sign-in, which keeps out abuse but turns away exactly the people most likely to ask.
	- **What I need from you:** "agreed", or your choice on any of the four parts.
	- Mason- I'll go with your recommendations.
	- **Decided (11/18): all four recommendations.**
		1. A short form on the page, sent by the server, with a honeypot and a rate limit.
		2. It goes to the site address plus everyone holding `manage_events`.
		3. A new Curator role holding only `manage_events`.
		4. No sign-in; the message and the reply address are optional.
	- Planned as Phase 2 §12.

20. **Should the server say plainly when the database is older than the code?**
	- **What happened (11/18):** clicking any event showed "This event couldn't be loaded. Please try again in a moment."
		- The cause wasn't the code: the dev database `communityfinder` was created before CM3, so it has no `event_gallery_photos` table and no `video_url` columns. I checked with a read-only query.
		- Every event page reads its gallery, so every one returned a 500 from the server. The events list kept working, which made it look like a page bug.
		- The fix is §7.3's `--recreate_database`.
	- **The general problem:** `--migrate` exists, but its lists are empty until the first deploy (`all_migrations.cpp`). So after any schema change, an old database fails in confusing ways until someone recreates it.
	- *Rec: yes, in honuware.* At startup, the server compares the tables and columns it expects (from its `DatabaseInfo`) with what the database has. If anything is missing, it logs one clear message, for example: "The database is older than this build: missing table event_gallery_photos, column events.video_url. Run --recreate_database." It keeps running, so the parts that still work stay usable. It's a small honuware change with tests, and knottyyoga gets it too.
	- *Alternative:* the server refuses to start with an old database, which is louder but blocks all work until the recreate.
	- *Alternative:* leave it, and remember to recreate after schema changes.
	- **What I need from you:** "yes", "refuse to start", or "leave it".
	- Mason- I like the recommendation to warn people that the database is out of sync. I've been burned by this a lot.
	- **Decided (11/18): yes. Warn at startup, and keep running.** It's built in honuware, so knottyyoga gets it too. Planned as Phase 2 §8.2, a cross-repo bump point.

21. **Where should the upload limit be set, and what range should it allow? (Follows OQ13.)**
	- **Where:** CF's admin has an Event settings page, but nothing for the site as a whole.
		- *Rec: a new "Site settings" page* on the admin dashboard, admin-only, holding honuware's upload-limit card. It's also where future site-wide settings would go, like the `manage_site_meta` settings `create_app_tables.cpp` mentions. knottyyoga would put the same card on its own settings page.
		- *Alternative:* add it to the Event settings page. That's quicker, but the limit covers every upload, including guide and user photos, not only events.
	- **Range:**
		- *Rec: 1 to 50 MB, default 10 MB.* Anything bigger than 50 MB is too slow on a phone connection, and the server holds each upload in memory while it handles it.
		- *Alternative:* a different upper bound.
	- **Who can change it:**
		- *Rec: admins only,* since it affects the whole site and server load.
	- **What I need from you:** "agreed", or the page, range or permission you want.
	- Mason- Agreed.
	- **Decided (11/18): a new admin-only "Site settings" page; 1 to 50 MB, default 10 MB.** Planned as Phase 2 §13.

22. **Should the sample data show off CM3's photos and video?**
	- **Today:** `database_helper --populate_sample_data` adds a small guide and four weeks of events from the day it runs. It has no photos, galleries or video (`populate_sample_data.cpp` never mentions them). So a freshly loaded dev site looks as it did before CM3, and the manual loop (§7.4) starts with uploading pictures by hand.
	- *Rec: yes, a little.*
		- A main photo on a few events and one series.
		- A three-photo gallery on one event.
		- The OQ18 video (`lVhJ_A8XUgc`) on one event and one series.
		- The pictures would be generated art like the offline demo's (`demo-photo-art.ts`): shapes and colour, no people, so there's no consent question. They'd be stored through honuware's photo storage like a real upload.
		- Re-running leaves them alone, as it does the listings.
		- With tests.
	- *Alternative:* the video only. That's one column, with no image storage involved.
	- *Alternative:* leave it, and add pictures by hand when trying CM3.
	- **What I need from you:** "yes", "video only", or "leave it".
	- Mason- I like the recommendation.
	- **Decided (11/18): yes, a little.** A main photo on a few events and one series, a three-photo gallery on one event, and the OQ18 video on one event and one series. The art is generated, with no people in it. Planned as Phase 2 §18.

23. **How should the video tile show a real frame from the video? (Follows Phase 2 §17.)**
	- **Background:**
		- OQ4 decided click-to-play so that nothing from YouTube or Vimeo loads, and nobody is tracked, until someone presses play.
		- To keep that promise, the tile borrowed the event's own photo, which Mason found misleading (§17).
		- It's now a plain gradient: honest, but it doesn't show what the video is. A frame needs the video site's thumbnail, and the question is who fetches it.
	- *Rec: our server fetches it once and serves it like our own photos.*
		- The first time an event's or a series' video tile is shown, the server gets the thumbnail and keeps it:
			- YouTube's is at a fixed address: `https://i.ytimg.com/vi/<id>/hqdefault.jpg`, 480×360.
			- Vimeo's comes from its oEmbed lookup: `https://vimeo.com/api/oembed.json?url=https://vimeo.com/<id>` gives a `thumbnail_url`.
		- It serves the thumbnail from our address, for example `/api/events/<id>/video_thumbnail`.
			- The route looks up the event's stored video, so it can't be used to make our server fetch arbitrary videos.
			- A changed link fetches again.
			- A failed fetch falls back to the gradient.
		- Visitors' browsers never contact YouTube before play, OQ4 holds, and the page's `img-src 'self'` stays. It works for Vimeo too.
		- honuware already has an outbound `HttpClient` (`foundation/util/http/http_client.h`), with a test double for the tests.
		- It's CF-side unless knottyyoga wants video tiles too.
	- *Alternative: the browser loads YouTube's thumbnail directly* from `i.ytimg.com`.
		- It's client-only and small, and no cookies or scripts are involved.
		- But every visitor's browser contacts Google's image server when the page opens, which is the thing OQ4 ruled out. Their IP address reaches Google; `referrerpolicy="no-referrer"` can at least hide which page they were on.
		- The page's `img-src` must add `https://i.ytimg.com` (PL2).
		- Vimeo has no fixed thumbnail address, so Vimeo tiles would stay the gradient.
	- *Alternative:* keep the plain gradient tile as it is now.
	- **What I need from you:** "server", "browser", or "keep it plain".
	- Mason- I'm a bit puzzled by this. When I've embedded YouTube videos on a page before, it definitely showed a still from the video. I won't want the plain gradient. Why is it bad to have the client's contact Google?
	- **Decided (11/18): not the plain gradient.** The tile shows the video's real still. Who fetches it is still open (below).
	- **Answer to Mason's questions (11/18):**
		- **What you saw before was YouTube's full player.**
			- A normal YouTube embed loads YouTube's player as soon as the page opens: a frame from youtube.com, several hundred KB of its script, and the requests that script makes. The still you saw was the player's own picture.
			- That's the "load the player right away" option OQ4 turned down for click-to-play. Click-to-play shows a stand-in until someone presses play, so the only question is what the stand-in shows.
		- **My recommendation shows that same still too.** The server option and the browser option show the identical picture from YouTube. The only difference is which computer asks Google for it:
			- in the server option, our server asks, once per video;
			- in the browser option, every visitor's browser asks, on every page view.
			- *My write-up above made the server option sound like it showed something else. It doesn't.*
		- **Why it matters that visitors' browsers contact Google, and how much:**
			- **What Google receives** from a thumbnail request: the visitor's IP address, their browser, and the video's id. Unless we set `no-referrer`, it also gets the address of the page they're on. No cookie or script is involved.
			- **Why that's sensitive here:** for most sites it's nothing. For this one, it amounts to "this IP address was looking at Leather Night." The outline's no-outing rule (GD1 §6.9) says being at an event is never reported as identity, and a third party's log of who looked at which event is a mild version of exactly that. An IP address isn't a name, but Google can often connect it to an account, for example when the same household is signed into Gmail.
			- **The honest size of the risk is small.**
				- Google already sees a lot of this from visitors on Chrome or Android. A site using Google Fonts or Google Maps sends the same kind of request.
				- Anyone who presses play sends all of this and more anyway.
				- So click-to-play protects the people who look but don't watch, who are often the curious ones who aren't out.
			- **Legally,** it's not a concern for a US site today. For context: in January 2022 a court in Munich ruled that a site sending visitors' IP addresses to Google through Google Fonts, without consent, broke the GDPR. That's why many sites now serve fonts and thumbnails from their own servers.
		- **So the choice is between:**
			- **Server:**
				- Google sees only our server, once per video, and Vimeo works too.
				- It's a slice of server work with tests: a small new table that keeps each video's thumbnail, a fetch through honuware's `HttpClient`, one route each for events and series, and the client using it.
			- **Browser:**
				- About an hour of client work, plus `https://i.ytimg.com` added to the page's `img-src` (PL2).
				- Each visitor's IP address reaches Google when the page opens.
				- Vimeo would need a lookup on vimeo.com from every browser too, or keep the gradient.
			- **Browser now, server before launch:** you see stills right away, since the site isn't live. The browser version (about an hour's work) is thrown away when the server version replaces it.
			- **Load YouTube's player right away.** This is what Mason has done before, and it reverses OQ4's click-to-play. *(Mason asked 11/18: "Why can't we load YouTube's whole player?" We can.)*
				- **The least work:**
					- The tile goes, and the page shows the player from the start, still from `youtube-nocookie.com`. Vimeo works the same way.
					- It's minutes of client work plus spec updates. The page's `frame-src` already allows both players.
				- **It looks and works exactly like YouTube:** the still, the title, and one tap to play.
					- On some phones the click-to-play tile needs a second tap, because the browser blocks autoplay once the player appears. The full player doesn't have that problem.
				- **The cost is page weight and privacy:**
					- Before anyone presses play, every event or series page with a video downloads YouTube's player. That's commonly measured at roughly half a megabyte to over a megabyte of script, which is slower on phones. The lists aren't affected.
					- Each visitor's browser also talks to several Google servers, so Google sees the visit in more detail than a thumbnail request shows. YouTube says its no-cookie mode stores nothing about the visitor until play, but its scripts still run.
				- **It gives the same look as the browser-thumbnail option, at many times the weight and with more sent to Google.** Its one real advantage over that option is the single tap on phones.
		- *Rec: still the server,* because this site's visitors are the people the no-outing rule protects, and the cost is one slice of work, not a redesign. The browser option is a reasonable call if you judge the risk small: it's what any site using Google Fonts does.
	- **What I need from you now:** "server", "browser", "browser now, server before launch", or "player right away".
	- Mason- Let's just do the full player. It's the best experience.
	- **Decided (11/18): the full player, from the start.** This reverses OQ4. Built as Phase 2 §17.2: lazy-loaded and not autoplaying, with no server work.
24. **Should someone who curates only the guide get into `/admin`?** *(Raised 11/18 while building §12.5.)*
	- **Background:**
		- `/admin` now lets in admins and anyone holding `manage_events` (the new Curator role).
		- `manage_guide` gives the server's permission to edit the guide's ten tables plus venues through Manage Data (GD2 OQ3). But someone holding only `manage_guide` is still sent away from `/admin`, so they can't reach the tables they're allowed to edit. That was also true under `AdminGuard`, so nothing has got worse.
		- Today no seeded role holds `manage_guide` except through the Administrator, so nobody is affected yet.
	- **Rec: leave it until a Guide curator role exists,** and do it in that bucket (GD2 or later). Then the guard admits `manage_guide` as well, and such a person sees only Manage Data, listing only the guide tables. The server already cuts the list, so it's a small client change with specs.
	- **Alternative:** widen the guard now. It's about an hour with specs, and it's ready for whenever someone is given `manage_guide` by hand.
	- **What I need from you:** "later", or "now".
	- Mason- Let's do it now.
	- **Decided (11/20): now.** Built (see §12.5's as-built notes):
		- **The door:** `core/guards/admin-area.guard.ts` adds `canManageGuide`, `canUseAdminArea` and `AdminAreaGuard`. `/admin` now lets in admins, events curators and guide curators.
		- **The Events pages:** `/admin/events` and `/admin/events/settings` now carry `ManageEventsGuard` of their own, so a guide curator can't reach them.
		- **Manage Data** stays open to everyone the door lets in.
		- **The dashboard:** a guide curator sees only Manage Data, described as "Browse and edit the guide tables". Someone holding both permissions gets "the events and guide tables". The header's Admin links follow the door.
		- **Server:** no code change. A new test pins down what each curator's table list holds: `CommunityFinderCrudTest.EachCuratorsTableListHoldsOnlyTheirOwnTables`.
		- **Still true:** no seeded role holds `manage_guide`. To try it, make a role holding `manage_guide` in Manage Data (`roles` + `role_permissions`), then assign it on the Roles page.
25. **What version should the `@honuware/ui` release be?** *(Raised 11/18 for the §13 bump point.)*
	- **Background:**
		- The library is at 0.3.1. §13.2 adds the upload-limit seam, the prepare step, `preparePendingPhoto()`, the settings card and the mock.
		- All of it is additive. An app that never provides the new seam keeps working, because the control doesn't block without it. knottyyoga's `ServerAccess` is untouched, because the seam isn't on `PhotoAccess`.
	- **Rec: 0.4.0.** Below 1.0, a new minor version is the usual way to say "new features", and CF's `"@honuware/ui": "0.3.1"` pin won't pick it up by surprise.
	- **Alternative:** 0.3.2, if you keep minor bumps for breaking changes.
	- **What I need from you:** the number when you publish. I'll put it in CF's `package.json` at the re-pin.
	- **Decided (11/19): 0.4.0.** Mason published it, and CF uses it.

# Change Log

- **11/3/2026 (v0.1)** — Drafted from Mason's ask: a main photo, an optional gallery, and an embedded video, because "visuals really market an event."
	- **What exists:** honuware's photo pipeline already gives events and venues one photo each, and EV2 already shows it.
	- **What this adds:**
		- a gallery, as rows of a new table, each with its one photo, so honuware is unchanged;
		- a server-chosen main photo, with a series-then-venue fallback;
		- YouTube and Vimeo links, parsed to an id and played only on a click;
		- curator tools in `/admin/events`;
		- the consent and no-outing policy.
	- **One honuware change (H15)** is needed only if submitters may upload a photo.
	- **The page's content security policy belongs to PL2,** which must allow the two players.
	- OQ1–OQ9 are open.
- **11/4/2026 (v0.2)** — Mason answered OQ1–9, and the server side is built (§0, §1–4, §6). The client (§5) is next.
	- **Decisions:** H15 now, with a video link and one flyer from submitters (OQ1); event → series → venue for the main photo (OQ2); YouTube and Vimeo (OQ3); click-to-play (OQ4); a cap of 12 that an admin can change (OQ5); who confirmed consent, and when (OQ6); galleries and video on series too (OQ7); the site's contact address for removal requests (OQ8); Stream A with Levi or Mason as owner (OQ9). The outline's Stream D row now points here.
	- **H15** was built in honuware: `PublicPhotoTables` gains an optional per-row rule. It's additive, so knottyyoga needn't change.
	- **Changes from v0.1:**
		- two curator routes after all: `POST /api/admin/event_gallery_photos`, so the server stamps the consent record from the session, and `GET`/`PUT /api/admin/event_gallery_settings` for the cap;
		- the cap is a setting (1–60, default 12), not a constant;
		- no `CountFor*` table methods, because the business layer counts the rows it already reads;
		- `consent_confirmed_by_person_id` is a plain id, not an FK, so deleting an account doesn't delete the photos that person added;
		- the flyer goes on every date still waiting for review;
		- the test_helper commands use kebab-case names (`set-event-video`, `add-gallery-photo`).
	- **OQ10 added:** whether signed-out visitors get photos bigger than honuware's 800×800 cap.
	- **Gates:** Linux 2500 passed against the local honuware tree (was 2417). The Windows run was stopped for low memory.
- **11/5/2026 (v0.2.1)** — H15 is pushed (honuware `f445a49`), and CF and knottyyoga are re-pinned to it.
	- **Strict pinned gates:** CF Windows 2500 passed, Linux 2500 passed. knottyyoga Windows 5454 passed, Linux 5454 passed (7 more than before: H15's honuware tests). The CF floor is now 2490.
	- **A test fix:** the first pinned run failed `GetEventSeriesTest.ASeriesCarriesItsPhotoVideoAndGallery`. The test kept a pointer into a JSON value that was freed at the end of its line, so whether it passed depended on leftover memory. It now parses the response into a local first. No other test in the app has that pattern.
- **11/18/2026 (v0.2.2)** — Mason checked in the server side (§0–4, §6) as `7422b8c`, answered OQ10, and §1.5 is built: CF serves photos up to 1600×1200.
	- **§1.5:** CF's seed overwrites honuware's two 800 defaults, as it already does for the mail app password. The keys belong to honuware, so they can't join the app's own defaults. No honuware change, and knottyyoga is unchanged.
	- **A correction:** honuware's 800×800 cap applies to signed-in readers too, not only signed-out visitors, as OQ10 and the sizes pin-down had said.
	- §7.3's live seed check now covers the two rows. It hasn't been run yet.
	- **Gates (pinned):** Linux 2506 passed. Windows 2503 of 2507 passed; the 4 failures come from separate CRUD work in progress, not §1.5 (see §7.1). The floor stays 2490 until the bucket exits.
- **11/18/2026 (v0.3)** — The server side went in as `9617614`, which re-pinned honuware to `838045f` and `@honuware/ui` to 0.3.1. The client (§5) is now built, and so are the gates that need no hands (§7.1, §7.2).
	- **What's new:**
		- the photo gallery and the click-to-play video on the event and series pages, with every photo address built in one place (`EventPhotos`);
		- in the curator's editor, the main photo, the video field, a gallery editor, and the gallery cap on the Event settings page;
		- on the submit form, a video link and a flyer; "My submissions" shows both.
	- **A small server addition for §5.7:** `/api/event_submissions/mine` sends `has_photo` and `video` for each date. It has a test in business_logic and one in endpoints.
	- **Gates:** Windows 2525 passed. Linux 2525 passed (pinned `838045f`). The floor is now 2515. Vitest: 563 passed. `ng build` passes for local, development and production. The bundle change is noted in §7.2.
	- **OQ11–18 added** for Mason:
		- the removal-request address (OQ11);
		- editing a gallery photo's words (OQ12);
		- shrinking big flyers (OQ13), retrying a flyer from "My submissions" (OQ14), and keeping the flyer over a pause (OQ15);
		- leaving the offline demo out of production (OQ16);
		- the seeded accounts' `changeme` password (OQ17, outside CM3, and the most urgent);
		- a real video for the demo (OQ18).
	- **Still to do:** §7.3, the live seed check (destructive, needs Mason's go-ahead), and §7.4, the manual loop (Mason or Levi).
- **11/19/2026 (v0.7.1)** — §13's bump point is done (§16.3).
	- Mason pushed honuware as `0b249be` and published `@honuware/ui` 0.4.0 (OQ25 decided).
	- CF re-pinned to both:
		- `GIT_TAG` is `0b249be`;
		- `package.json` asks for `0.4.0`, and `npm install` replaced the local overlay.
	- **Strict pinned gates:** Windows 2572 passed. Linux 2572 passed. The floor is now 2562.
	- **Client on the published library:** Vitest 620 passed. All three builds pass.
	- *The first publish stopped at the CI's tag-and-version check* (the version file still said 0.3.1). A second commit bumped it and the tag moved there. Nothing was published until that run.
	- **Still to do:**
		- Mason commits CF;
		- knottyyoga's own follow-up (§8.2's call and §13.4's card);
		- the checks that need hands (§8.1, §7.3, §7.4, §16.4);
		- OQ24.
- **11/18/2026 (v0.7)** — Phase 2 is built (§8.2, §9–15, §18), across all three repos. It waits on the §13 bump point, and on the checks that need hands (§8.1's event page, §7.3, §16.4).
	- **CF server:**
		- the curator search route behind Edit Event (§9);
		- removing a flyer (§11);
		- the Curator role, and photo-removal requests emailed to the site and its curators (§12);
		- the startup schema check called from `main.cpp` (§8.2);
		- sample photos, a gallery and the video (§18).
	- **CF client:**
		- "Coming up" and search results that tell events apart (§9);
		- editing a gallery photo's words (§10);
		- add, replace or remove the flyer on "My submissions" (§11);
		- the removal-request form (§12.4);
		- curators in `/admin` (§12.5);
		- the offline demo out of production (§14);
		- the real demo video (§15);
		- the Site settings page, and flyers shrunk and checked before sending (§13.3).
	- **honuware (uncommitted):**
		- `DbMeta::WarnIfDatabaseIsOlderThanBuild` (§8.2);
		- the upload limit's business logic and routes (§13.1);
		- `CsrfGuard::AllowWithoutToken` (§12.3);
		- `kPermissionAdminPortal` in `permissions.h`.
	- **`@honuware/ui` (uncommitted):** the upload-limit seam and mock, the public prepare step, `preparePendingPhoto()`, and the settings card (§13.2).
	- **Decisions beyond the plan:**
		- a site-wide hourly cap on removal requests beside the per-visitor one;
		- a narrow CSRF exemption for that one public route (the guard refused every visitor who'd never signed in);
		- the upload limit as its own seam, because knottyyoga extends `PhotoAccess`;
		- the control fails open if the limit can't be read;
		- a flyer is prepared before anything is sent, so a too-big flyer can't leave a submission without it.
	- **Gates:**
		- CF co-dev: Windows 2572 passed, Linux 2572 passed; honuware's gate 58 passed;
		- Vitest 620 passed in 74 files;
		- web components 550 SUCCESS, lint and build clean;
		- `ng build` passes for all three configurations.
		- §14 took 48.1 kB off production's initial bundle, and all 64.6 kB of mock code. The floor stays 2515 until the re-pin.
	- **OQ24–25 added:** whether a guide-only curator gets into `/admin`, and the library's version number.
	- **Next (the bump point):**
		1. Mason commits and pushes honuware.
		2. Mason commits and publishes the web components (OQ25).
		3. I re-pin CF's `GIT_TAG`, bump `@honuware/ui` and run `npm install`, verify a fresh pinned build on both platforms, and raise the floor.
		4. Mason commits CF.
		5. knottyyoga follows with §8.2's call and §13.4's card.
- **11/18/2026 (v0.6)** — OQ23 decided: the full player. Mason: *"Let's just do the full player. It's the best experience."* This reverses OQ4's click-to-play.
	- Built as §17.2: `VideoEmbed` is now the provider's player from the start, lazy and not autoplaying, and `videoEmbedUrl` dropped autoplay.
	- The design pin-down, §5.4's note and OQ4 are updated, and the thumbnail plans are dropped.
	- Vitest: 562 passed. Production build clean.
- **11/18/2026 (v0.5.1)** — OQ23 gains a fourth option after Mason asked why we can't load YouTube's whole player: we can (it reverses OQ4). Its costs and benefits are written up beside the other three.
- **11/18/2026 (v0.5)** — Mason answered OQ19–23. No code changed.
	- **Decided:**
		- OQ19: all four recommendations, so §12 is no longer waiting;
		- OQ20: warn at startup and keep running, in honuware (§8.2, now fully planned);
		- OQ21: a new admin-only Site settings page, 1–50 MB (§13);
		- OQ22: sample photos, a gallery and the video (new §18).
	- **OQ23 needs more discussion.** Mason doesn't want the plain gradient, and asked why it's bad for visitors' browsers to contact Google.
		- The answer is written under OQ23: what he saw before was YouTube's full player; my recommendation shows the same still; and what a visitor's browser sends Google, why that matters on this site, and how small the risk honestly is.
		- §17.2 has both versions planned. It waits on "server", "browser", or "browser now, server before launch".
- **11/18/2026 (v0.4.1)** — The video tile showed the event's photo, which read as a frame of the video (Mason).
	- The tile is now the plain gradient (Phase 2 §17.1). Vitest: 563 passed.
	- How to show a real frame is **OQ23**.
- **11/18/2026 (v0.4)** — Mason answered OQ11–18 and added two new-work items. Phase 2 (§8–16) plans the follow-ups. No code changed.
	- **Decisions:**
		- removal requests go to the site address and everyone with the editing permission, plus a new non-admin role (OQ11);
		- editing gallery photos' words, and **full CRUD for anything that can be created** from now on (OQ12, saved as a memory);
		- a configurable upload limit (10 MB default) built in honuware for both apps, with the image-shrink step made public (OQ13);
		- add, replace and remove a flyer from "My submissions" (OQ14);
		- leave the pause case as it is (OQ15);
		- the demo out of production builds, in its own commit (OQ16);
		- the seeded passwords stay until deployment, since the site isn't live (OQ17);
		- a real demo video, `lVhJ_A8XUgc` (OQ18).
	- **Mason's new item 1, "This event couldn't be loaded":** not a bug and not a later phase. The dev database predates CM3: no gallery table and no `video_url` columns, so every event page returns a 500. Running §7.3's recreate fixes it (§8).
	- **Mason's new item 2:** a "Coming up" list and search results with the date, time and venue on Edit Event, through a new curator search route (§9).
	- **OQ19–21 added:**
		- how removal requests and the curator role work (OQ19);
		- whether the server should say plainly when the database is older than the code (OQ20);
		- where the upload limit is set, and its range (OQ21);
		- whether the sample data should include photos and video (OQ22). That was added after Mason recreated the database (§8.1) and found the recreate seeds no events, because they come from `--populate_sample_data`.
