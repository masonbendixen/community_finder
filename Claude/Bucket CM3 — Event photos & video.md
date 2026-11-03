---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · Mason Bendixen
Last Updated: 11/3/2026
Version: 0.1
tags:
---

# Bucket CM3 — Event photos & video

**Track:** CM — Community · **Size:** M (the outline said S; video and submitter pictures are additions) · **Stream:** D in the outline, but nearly every file is Stream A's (OQ9) · **Owner:** open (OQ9)
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
- **A gallery** of up to 12 photos (OQ5) on an event or a series, each with alt text, an optional caption and an optional credit. Shown as a grid, with a full-size viewer.
- **One embedded video** per event or series: YouTube or Vimeo (OQ3), click-to-play (OQ4).
- **Curator tools:**
	- the main photo, gallery and video in the `/admin/events` editor;
	- series through the generic admin editor, which already handles photos and child tables.
- **Submitter additions in the submit form,** as OQ1 decides: a video link, and maybe one flyer photo.
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
	- **The fallback, if OQ2 is agreed:** an event without one shows its series' photo, then its venue's. This turns on photos for `series` (photo support and public read).
	- **The server picks.** Event payloads gain `photo: {"table": "events" | "series" | "venues", "id": N}`, or `null`. The client builds the URL with `PhotoUrlBuilder` as it does now.
		- The event page drops its separate `has_photo` call.
		- List rows stop guessing and hiding failures.
	- **One query per list,** not one per row: a join against `table_item_photos` for the three tables.
- **A gallery is rows, not a honuware change.** honuware stores one photo per row, so each gallery picture is a row in a new app table, and the row has its one photo.
	- That reuses upload, scaling, the public read, and the delete clean-up unchanged. A honuware change for several photos per row would touch every consumer.
	- **A row belongs to an event or to a series, never both.**
		- A recurring night's photos belong to the series: each dated instance goes past and drops off the calendar.
		- An event in a series with no gallery of its own shows the series' gallery.
	- **Order:** `sort_order`, then `id`.
	- **What the event page gets:** `gallery: [{id, alt_text, caption, credit}]`, only on `/api/events/<int>` and only while the event is visible. Lists don't carry it.
- **Video is a link we parse, never HTML we store.** `video_url` on events and series holds a canonical link:
	- `https://www.youtube.com/watch?v=<11-char id>` or `https://vimeo.com/<digits>`.
	- Pasted variants become canonical: `youtu.be/…`, `/shorts/…`, `/live/…`, `m.youtube.com`, extra `&t=` parameters.
	- **`ParseVideoLink` (business_logic) is the one parser.** Payloads carry `video: {"provider": "youtube" | "vimeo", "id": "…"}` or `null`.
	- **The client builds the player address only from that id:**
		- YouTube: `https://www.youtube-nocookie.com/embed/<id>`;
		- Vimeo: `https://player.vimeo.com/video/<id>?dnt=1`.
		- The address goes through Angular's trusted-URL wrapper once and is cached (knottyyoga's Location-page pattern).
	- **A database CHECK is the backstop.** Curators save events through generic CRUD, which no app code sees, so the column only accepts the two canonical shapes. The editor turns a pasted link into its canonical form before saving.
- **Click-to-play (OQ4).** The page shows a play tile, not a player: the event's own photo (or the gradient tile), a play button, and "Plays from YouTube."
	- Nothing from YouTube or Vimeo loads until someone presses play. It's private (the outline's "never trackers") and costs no page weight.
	- Pressing play creates the frame with autoplay, a title ("Video: Leather Night"), and fullscreen allowed.
- **The page's content security policy must allow the two players.** honuware's `SecurityHeaders` policy rides the API's responses. The page itself is served from S3/CDN in knottyyoga's setup, so the page's policy belongs to PL2.
	- Wherever `index.html` is served, `frame-src` must name `https://www.youtube-nocookie.com` and `https://player.vimeo.com`.
	- `img-src` stays `'self' data:`, because the play tile uses our own image.
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
- **Sizes.** `hw-photo-upload` already shrinks and re-encodes on the way up, and the server enforces `image_max_upload_bytes`. The sizes:
	- the existing thumbnail (160×160) and hero (1200×630);
	- a gallery thumbnail (480×360);
	- a full view (1600×1200).
	- Scaled copies regenerate on demand. EV4's cleanup job already prunes old ones.
- **Sharing tags.** `og:image` stays the main photo. The JSON-LD `image` becomes a list: the main photo, then up to three gallery photos.

# Layered work items

### 0. honuware

- [ ] 0.1 **Only if OQ1 lets submitters add a photo:** **H15**, a per-row photo visibility check.
	- `PublicPhotoTables` gains an optional rule the app supplies. CF's rule: an `events` photo is public only once its event is approved.
	- A photo that isn't public falls back to the normal signed-in read, so curators still see it.
	- `get_scaled_photo` and `has_photo` use it. Additive, so knottyyoga needn't move.
	- **Tests** in honuware. The cross-repo bump per CLAUDE.md, ideally bundled with H13/H14.
	- *Without H15, a pending event's photo is readable by anyone who guesses its number, as GD2 OQ12 (d) accepted for draft guide pages. That's tolerable for curator uploads, not for strangers' uploads.*

### 1. db_schema

- [ ] 1.1 `src/db_schema/event_gallery_photos.{h,cpp}`:
	- `id`;
	- `event_id` and `series_id`, both nullable FKs with cascade, and a CHECK that exactly one is set;
	- `sort_order` int, default 0;
	- `alt_text` (required, 1–300);
	- `caption` (≤300, default `''`) and `credit` (≤120, default `''`);
	- `consent_confirmed_by_person_id` (FK people, set null on delete, per EV1 OQ8 (e)) and `consent_confirmed_us`;
	- `created_us` and `updated_us`;
	- indexes on (`event_id`, `sort_order`) and (`series_id`, `sort_order`).
- [ ] 1.2 `video_url` on `events` and on `series`: not null, default `''`, with a CHECK for `''` or the two canonical shapes.
- [ ] 1.3 `create_app_tables.cpp` (the three-place rule):
	- `MakeAppTables`;
	- `AppTableCreationOrder` (after `events` and `series`);
	- `PopulateAppTables`, which covers:
		- photo support for `event_gallery_photos`, and `series` if OQ2;
		- the `manage_events` table mapping;
		- admin metadata, so the gallery shows as a child table of events and series in the generic editor.
	- **Tests:** `create_app_tables_test.cpp` (photo support rows, the mapping, the nesting), and the CHECKs (both parents, neither parent, blank alt text, a non-canonical video link).
- [ ] 1.4 `register_app_services.cpp`: public read for `event_gallery_photos`, and `series` if OQ2. **Test:** `register_app_services_test.cpp`, where a signed-out visitor gets a gallery photo.

### 2. table_helpers

- [ ] 2.1 `event_gallery_photos.{h,cpp}`:
	- `AddGalleryPhoto`;
	- `GetGalleryForEvent` and `GetGalleryForSeries` (ordered);
	- `CountForEvent` and `CountForSeries`;
	- `UpdateGalleryPhoto` (bumps `updated_us`);
	- `DeleteGalleryPhoto`.
	- **Tests.**
- [ ] 2.2 `events` and `series`: `video_url` in add, update and the reads. **Tests.**
- [ ] 2.3 **The photo picker:** `GetPhotoOwners(eventIds)` → for each event, which of event, series or venue has a photo, in one query against `table_item_photos`. **Tests:** each of the four outcomes, and a mixed list.

### 3. business_logic

- [ ] 3.1 `video_link.{h,cpp}`: `ParseVideoLink(text)` → `{provider, id, canonicalUrl}` or an error code (`BAD_VIDEO_URL`, `VIDEO_HOST_NOT_SUPPORTED`).
	- **Tests:**
		- each YouTube variant, with and without extra parameters;
		- a playlist-only link (refused);
		- a Vimeo number, and a Vimeo channel or album (refused);
		- `http:`;
		- another host;
		- an id of the wrong length.
- [ ] 3.2 `EventHelper`:
	- the resolved `photo` on list and detail reads;
	- `gallery` (the event's own, else its series') on the detail read, only when the event is visible;
	- `video` (the event's own, else its series').
	- **Tests:** the fallbacks; a pending event's gallery hidden from the public read and shown in the curator preview.
- [ ] 3.3 **The gallery cap** (OQ5) and deletes:
	- a 13th row is refused;
	- deleting a row deletes its photo through `ImageHelper::DeletePhotoForItem`. Generic CRUD already does this.
	- **Tests.**
- [ ] 3.4 **If OQ1:** `SubmissionHelper` takes `video_url`, checks it with `ParseVideoLink` (field `video_url`, the EV3 problem shape), and stores it on each date's event. **Tests.**

### 4. endpoints

- [ ] 4.1 Payloads gain `photo` and `video` (`/api/events/upcoming`, `/api/events/<int>`, `/api/event_series*`), and `gallery` (`/api/events/<int>` and `/api/event_series/<string>`). **Tests** (`handle_full`, `ThreadPool::Shutdown()` first):
	- an event with each kind of photo;
	- a gallery in order;
	- a video;
	- a pending event's gallery not shown publicly.
- [ ] 4.2 **No new curator endpoints.** Gallery rows go through generic CRUD under `manage_events`, as events do, and their photos through honuware's upload and delete endpoints. **Tests:**
	- a curator adds a row and uploads its photo;
	- someone without `manage_events` is refused.
- [ ] 4.3 **If OQ1:** `POST /api/event_submissions` takes `video_url`. **Tests:** accepted, and refused with the field named.
- [ ] 4.4 **If OQ1 includes a flyer:** after H15, the submitter uploads one photo to their pending event. One more route, `POST /api/event_submissions/<int>/photo`, checks that the caller owns the submission and that it's pending, then hands off to `ImageHelper`. **Tests.**

### 5. client (`ui/src/app/`)

- [ ] 5.1 **Types and access:** `photo`, `gallery` and `video` on the event and series types, and in the offline mock. The demo gets placeholder art (generated shapes and color fields, not people), one gallery and one video id. **Specs.**
- [ ] 5.2 **One place for photo addresses:** `photoFor(event, width, height)`. Event rows and the event page switch to it, and the event page drops its `has_photo` call. **Specs:** each owner, and none.
- [ ] 5.3 `shared/components/photo-gallery`:
	- a thumbnail grid;
	- a full-size viewer in a dialog: arrow keys, Escape, swipe on a phone, alt text, caption and credit, focus returned on close;
	- "Ask us to remove a photo."
	- **Specs.**
- [ ] 5.4 `shared/components/video-embed`: the play tile, then the frame on press. It's built only from `{provider, id}`. **Specs:**
	- no frame before the press;
	- the right address for each provider;
	- the frame's title;
	- the address isn't rebuilt on every change check.
- [ ] 5.5 **The event page and the series page:** the hero (resolved), then the video, then the gallery. The JSON-LD `image` list. **Specs.**
- [ ] 5.6 **The curator editor** (`pages/admin/manage-events.*`):
	- the main photo through `hw-photo-upload`;
	- a video field that shows what it found ("YouTube video ✓") or why not, and saves the canonical link;
	- **a gallery section:**
		- thumbnails with their alt, caption and credit;
		- move up and down;
		- remove, with a confirm in the page;
		- "Add photo": alt text, caption, credit, the consent box, and `hw-photo-upload` with `deferUpload`. The row is created first, then its photo uploaded.
	- **Specs.**
	- *Series use the generic admin editor (photo upload plus the gallery as a child table) until someone asks for more.*
- [ ] 5.7 **If OQ1:** the submit form gets an optional "Video link" with the same feedback, and maybe a "Flyer" upload (H15). "My submissions" shows them. **Specs.**

### 6. test_helper

- [ ] 6.1 Two commands for local data:
	- `set_event_video --event_id N --url …`, which goes through `ParseVideoLink`;
	- `add_gallery_photo --event_id N | --series_id N --file … --alt …`, which uploads through `ImageHelper` and records the consent as the test admin.
	- **Tests.**

### 7. Gates

- [ ] 7.1 Docker suite green per slice, Windows too ("Windows: N passed. Linux: M passed."); raise the floor at bucket exit.
- [ ] 7.2 `ng build` for local, development and production; vitest green, with the count recorded; the bundle change noted. The viewer and player load with the event pages, not up front.
- [ ] 7.3 **Live seed check after §1** (destructive, announced):
	- `event_gallery_photos` with its CHECKs and indexes;
	- `video_url` on both tables, with its CHECK;
	- the photo-support and public-read rows;
	- the `manage_events` mapping;
	- nothing new in `allowed_tables`.
- [ ] 7.4 **Manual loop (Mason or Levi), on a phone as well as desktop:**
	1. Add a main photo, three gallery photos and a YouTube link to an event.
	2. Reorder the photos, then remove one.
	3. Check the event page, the list thumbnail, and a series instance with no photo of its own.
	4. Press play.
	5. Check the shared-link preview.
	6. Follow the removal-request path.

# Open Questions

*(Numbered for inline answers; recommendations included so "agreed" suffices.)*

1. **Who can add pictures and video?** EV3 decided curators add the flyer on review, because a stranger's photo raises the consent and no-outing questions this bucket answers, and invites people to use us as free image hosting.
	- *Rec: submitters may give a video link* (it points at something already public, and a curator sees it before anyone else does).
	- *Rec: the flyer waits.* Curators add the main photo and gallery in this bucket. A submitter's flyer upload is a follow-up that needs H15 first, so a stranger's image is never public before a curator approves it.
	- *Alternative:* do H15 now and let submitters attach one flyer.
	- Mason- Let's do H15 now.
2. **When an event has no photo of its own, should it borrow one?** First from its series, then from its venue. A weekly night then looks right without someone uploading to every date.
	- *Rec: yes.* It turns on photos for series.
	- Mason- I like this recommendation.
3. **Which video sites?**
	- *Rec: YouTube and Vimeo,* played from YouTube's no-cookie address and Vimeo's do-not-track player.
	- Instagram, TikTok and Facebook embeds only work by loading their own scripts on our page, which the "never trackers" rule rules out. Their links can still go in the event's Link field.
	- Mason- YouTube and Vimeo are fine.
4. **Click-to-play, or load the player right away?**
	- *Rec: click-to-play.* Nothing from YouTube loads, and nobody is tracked, until someone chooses to watch. It also keeps pages fast.
	- Mason- This sounds good.
5. **How many gallery photos per event or series?**
	- *Rec: 12,* enough to market a night without becoming an archive. Easy to raise later.
	- Mason- This sounds good but let's make this configurable via the admin portal.
6. **Record who vouched for consent on each photo?**
	- *Rec: yes.* A required "everyone recognizable agreed" box on upload, stored with who ticked it and when. When a removal request comes in, we can see who added the photo and on what understanding.
	- Mason- This sounds good to me.
7. **Galleries on series pages too?**
	- *Rec: yes, and the same for video.* A recurring night's pictures belong to the series, because each dated instance drops off the calendar once it's past. An event shows its own gallery, or else its series'.
	- Mason- This sounds good.
8. **Where does the removal-request link go before CM1 exists?**
	- *Rec: the site's contact address,* with the photo's page in the subject, and a curator acts within a day. CM1's report flag replaces it.
	- Mason- Your recommendation is fine.
9. **Which stream owns it, and when?** The outline puts CM3 in Stream D, gated on traction. But almost every file is Stream A's (the events tables, the event pages, the curator editor, the submit form), and Mason wants it for marketing now.
	- *Rec: run it in Stream A after EV3's gates, with Levi or Mason as owner.* Update the outline's Stream D row to point here.
	- Mason- Sounds good.

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
