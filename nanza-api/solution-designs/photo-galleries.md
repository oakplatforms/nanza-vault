---
tags: [nanza-api, nanza-mobile, plan, posts, content-items, images, galleries, affiliates]
---

# Photo Galleries & Image Gates — five library photos, images for public authors only

**Status:** **API built 2026-10-18 and deployed to dev; mobile built 2026-10-19, uncommitted,
pending device QA (`tsc` + eslint clean). No migration, no types bump.** Plan from a voice session the same day. Spans nanza-api (validation, upload pairing, caps,
author gate) and nanza-mobile (multi-select library picker, composer preview, the reader's photo
rail). Estimated 2–3 days: ~1 day API, ~1.5 days mobile, QA after.

> **API implementation record (2026-10-18):** `validation/post.ts` — `GROUP_CHILD_TYPES` gains
> `IMAGE`; a group must be homogeneous (photos or references); `MAX_GALLERY_IMAGE_ITEMS = 5`,
> `MAX_IMAGE_GROUPS = 1`, `MAX_POST_IMAGE_FILES = 6`; the either/or rule is stated in shapes (one
> image group max; with a group ≤ 1 loose IMAGE, without one ≤ `MAX_IMAGE_ITEMS` = 2 loose);
> `assertAuthorMayPostImages` runs whenever any IMAGE leaf exists (live `profile.type` against
> `PUBLIC_AUTHOR_PROFILE_TYPES`, error *Photos in posts are available to affiliate and verified
> profiles*). `routers/post.ts` — `flattenImageLeaves` / `imageLeavesOf` walk the tree in flattened
> index order for the create count + upload loop and for PUT's existing-key check, new-file pairing
> and retained-key diff; both multipart routes take `MAX_POST_IMAGE_FILES` files. Sweeps already
> covered children (delete queries by `postId`, PUT reads `contentItem.findMany`, the draft
> replacement reads the relation) — no change there. `tsc` + eslint clean.
>
> **Mobile implementation record (2026-10-19):** `constants/profileTypes.ts` (new) holds the
> `PUBLIC_AUTHOR_PROFILE_TYPES` mirror + `isPublicAuthorType`; `useCanSharePost` reads it and a new
> `useIsPublicAuthor` hook reads the viewer's own `profile.type` from the session. `imagePicker.ts`
> `pickImage` takes `{ selectionLimit }` (library only, default 1). `composer/draft.ts` — new
> `DraftImageChild` / `DraftGroupChild`, `isPhotoGallery`; `draftTypeCounts.libraryImages` counts
> gallery children as leaves; `deserializePost` restores IMAGE children by media key and buckets a
> loose photo beside a gallery as the camera snap; `serializeDraft` walks gallery children in place
> (files in flattened order). `PostComposer` — `MAX_LIBRARY_IMAGES` → `MAX_GALLERY_PHOTOS = 5`;
> `handleAttachImage` multi-picks, resizes each, two-or-more → one GROUP of IMAGE children, one
> stays loose; the library circle disables once the library slot holds anything (either/or); a
> photo gallery's cap message; both image circles render only for `useIsPublicAuthor()`; the draft
> rail + `draftPreviewItem` map photo children through `draftPreviewChild`. `PostContentItems` —
> `PreviewableContentItem` children are previewable; the IMAGE leaf in a rail takes the new
> `posts.ts contentImageRail` (3:2, radius 12, hairline, slot-width — a bit smaller than a lone
> photo by the slot's peek). Lightbox needed nothing: it already collects rail children. Fold into
[[post-carousel-and-tagging|Posts — Multi-Content Carousel]] / [[posts|Posts]] (api) and
[[../../nanza-mobile/solution-designs/posts-chat|Posts Chat]] (mobile) when it lands, then delete
this file.

## The ask

A poster opens their camera roll, selects **up to five photos** at once, and the post carries them
as **one sliding gallery** — the same slot-width, next-one-peeks rail the card and listing galleries
already use. Camera capture stays single-shot. **Only AFFILIATE and VERIFIED profiles may create a
photo gallery** (Skylar, mid-session 2026-10-18: "affiliates and verified should be able to create
galleries, no one else").

Second ask the same session: **image capture and the image picker themselves become public-author
features** — a BASIC profile gets no photo circle and no camera circle in the composer, and the
API rejects IMAGE items from them. Posting itself stays open: Skylar, after a brief detour,
"don't put a gate over the brand post — no one can see them, but unverified users can create them".
So the home FAB and the BRAND create path are untouched; only *images in posts* are gated.

All open questions were answered the same evening — see *Decisions* at the end. In short: the
image gate is AFFILIATE + VERIFIED ("it's both"); a post carries **one gallery of up to five OR one
single photo, never five loose photos**; BASIC composers get no picker and no camera; gallery
tiles take the post image's aspect but run **a bit smaller than a single photo**, so a gallery reads
as a gallery the way the multi-card rail reads against a single card; the camera never feeds a
gallery.

## Where we stand (verified 2026-10-18)

- **Galleries are `GROUP` content items** whose children are leaves on the same post
  (`ContentItem.parentId`, cascade). The composer's multi-select pickers (Shop, My Library)
  already assemble one; the posted card renders it through `PostContentItems`' GROUP branch — a
  horizontal rail, each child at `getRailSlotWidth` (container minus a fixed 40pt peek).
- **But a group may hold references only.** `GROUP_CHILD_TYPES = REFERENCE_CONTENT_ITEM_TYPES`
  (LISTING / BID / ENTITY) — "media stays top-level" was the 2026-08-12 call. IMAGE is rejected
  as a child.
- **Images are capped at two per post** (`MAX_IMAGE_ITEMS = 2`), which the composer splits into one
  library pick + one camera snap (`MAX_LIBRARY_IMAGES` / `MAX_CAMERA_IMAGES`, client-side only).
- **Upload pairing is positional and top-level.** `POST /post` is multipart; files pair with IMAGE
  items *in index order*, and both the count check and the upload loop filter the post's
  **top-level** `contentItems` — a gallery's children are created in a second step and would be
  skipped. `uploadConfig.array('image', MAX_IMAGE_ITEMS)` also hard-caps multer at two files.
- **The picker is single-select.** `pickImage('library')` calls `launchImageLibrary` with no
  `selectionLimit`; `handleAttachImage` reads `assets[0]` only.
- **The reader's IMAGE leaf already supports preview URIs and the lightbox**: `viewerMediaPaths`
  collects every stored photo in the tree, *rail children included, in order*, into one swipeable
  `ImageViewer` set — so a five-photo gallery swipes as one set for free.
- **Public author types are one constant**: `PUBLIC_AUTHOR_PROFILE_TYPES` (`AFFILIATE`,
  `VERIFIED`) in `constants/profileTypes.ts`, mirrored in mobile's `useCanSharePost`.

## Design — nanza-api

### A photo gallery is a GROUP of IMAGE children

- `GROUP_CHILD_TYPES` grows to include `IMAGE`, with one new rule: **a group is homogeneous** —
  all IMAGE children (a photo gallery) or all reference children (the existing galleries). A mix
  is rejected (`A gallery holds either photos or marketplace items, not both`). Groups still do
  not nest.
- Children keep their own `index` within the group; the group keeps its position in the post.
  No schema change — `ContentItemType.GROUP`, `parentId`, `media` all exist.

### Caps

- New `MAX_GALLERY_IMAGE_ITEMS = 5`: the ceiling on IMAGE children in one group.
- **One photo gallery per post** (mirrors `MAX_COLLECTION_ITEMS = 1`: the gallery is the headline
  gesture; the camera snap is the other image slot).
- **Library budget is either/or (decided):** a post carries ONE library gallery of up to five
  photos, OR one single library photo — never several loose photos. In validator terms: at most
  one library-image *attachment* (a GROUP of IMAGE children, or one top-level IMAGE), with the
  group capped at five. A second loose IMAGE alongside a gallery is rejected.
- The camera snap is its own single slot beside that (see *Camera* below).
- **BASIC authors get zero** — no IMAGE item at all (`MAX_IMAGE_ITEMS` goes from the flat 2 to a
  per-author value). Counted in leaves, like every other budget. Text, links, embeds, cards,
  listings and collections stay open to all.
- Replies are unchanged: `REPLY_MAX_ATTACHMENT_ITEMS = 1` counted in leaves means a gallery can
  never fit in a reply.

### Camera

- **The camera never feeds a gallery (decided).** Capture stays the single-shot top-level IMAGE it
  is today; a gallery's children are library picks only. The composer keeps the two sources on
  separate circles with separate caps, so there is nothing to merge.

### The author gate

- `validateContentItems` already receives `accountId`. It resolves the author's `profile.type`
  (one select, cached for the call) and requires `isPublicAuthorType(type)` for **any IMAGE leaf** —
  loose or inside a group; otherwise `Photos in posts are available to affiliate and verified
  profiles`. One branch covers the picker, the camera and the gallery; it goes through the shared
  constant so a future type is a one-line change.
- Applies to **replies too** (a reply's one attachment can be a photo today; for a BASIC author it
  no longer can) and to `PUT` edits — an existing BASIC author's old photo posts keep rendering
  (nothing is scrubbed), but an edit that carries the photo forward is refused until the photo is
  removed. Demotion follows the same live-state posture as visibility: the gate reads the current
  type.
- Read side needs nothing: visibility rules are the post's, and a gallery is just items.

### Upload pairing — flatten the tree

- Both the pre-check (`imageItemCount`) and the upload loop walk the **flattened** item tree —
  top-level items in index order, a group's children in their index order at the group's
  position — so `image` files pair with IMAGE leaves wherever they sit. The composer appends its
  files in the same flattened order (today's contract, extended).
- The upload still happens **after** the rows exist (each S3 key is its item's id, resized to the
  post-media size), and a failed upload still rolls back the post and sweeps already-uploaded keys.
  The sweep, and `collectImageKeys` used by delete / PUT-replace, must flatten too — otherwise a
  gallery's photos orphan in S3.
- `uploadConfig.array('image', …)` raises its file cap to the new maximum (6 for a public author's
  post). Multer's cap is a transport ceiling, not the budget — the validator enforces the budget.

### Edit (`PUT /post/:id`)

- Galleries already "edit like everything else": PUT replaces the item set, carried-over IMAGE
  items name their existing `media` key (this post's only), media-less IMAGE items pair with new
  files in flattened order. Children follow the same rule, so adding a sixth photo is rejected by
  the cap and removing one is simply a smaller gallery.

### Types

- No new DTO fields: `ContentItem.children` already exists on the published `Post` schema and
  IMAGE items already carry `media`. A types bump is only needed if we surface the caps.

## Design — nanza-mobile

### The composer's image circles

- **BASIC viewers see neither the photo circle nor the camera circle.** The attach row renders
  its remaining sources (cards, shop, link, text, embed) at the same pitch; no disabled state, no
  upsell copy (the public-author distinction is not advertised anywhere else in the app). The
  session-level check is the viewer's own `profile.type` against the `PUBLIC_AUTHOR_PROFILE_TYPES`
  mirror `useCanSharePost` keeps — promote the mirror to a shared `isPublicAuthorType` helper so
  the two reads cannot drift.
- Reply composers follow the same rule (their one attachment is picked from the same row).

### The picker

- `pickImage('library')` takes an optional `selectionLimit`; the composer passes **5**. Camera
  stays single. (Only public authors reach either, per the above.)
- Each asset is resized as today (1200 max side, q85). **One selection** stays a plain IMAGE
  attachment (today's behaviour). **Two or more** become ONE `GROUP` attachment whose children are
  IMAGE leaves — the same "one selection is a plain attachment, two or more become a gallery" rule
  the Shop and Library pickers follow.
- Budget copy: `MAX_LIBRARY_IMAGES` becomes 5, counted in leaves (the gallery's children spend
  from it); the photo circle disables at 5 or once a gallery exists. The "One photo per post"
  string retires with the single-photo budget.

### Composer preview

- The existing draft rail renders a GROUP's children through `PostContentItems` with `railChild`
  and the per-child **X**; the IMAGE leaf already renders `localUri`. Removing a child shrinks the
  gallery; removing the last child removes the attachment — current behaviour, no change.
- Submit appends `image` files in **flattened item order** (top-level order, children in place),
  matching the API contract above. Edit mode re-sends carried-over photos by `media` key.

### The reader's photo rail

- The posted GROUP branch already lays children into slot-width tiles with the 40pt peek. Photo
  children take the post image's existing aspect (`contentImage`), not the tile's 386:251.86
  reference-hero aspect, so five photos cut the same height across the rail.
- **A bit smaller than a single photo (decided):** the gallery tile is deliberately narrower than
  the lone IMAGE leaf, so a gallery reads as a gallery at a glance — the same cue the multi-card
  rail gives against a single card. The rail's slot (container minus the 40pt peek) already does
  this; photo tiles do NOT get the rail-size re-derivation the entity thumb got on 2026-10-18.
  One class (`contentImageRail`) with the aspect and the slot's radius; nothing else.
- **Tap any photo → lightbox** across all the post's photos, already wired (`viewerMediaPaths`).
- Cards render the primary item only, so a gallery that is primary shows the rail on the feed
  card, the detail, the profile tab and a Stories flat page alike — no new surface work.

### Touched code (mapped)

- api: `src/validation/post.ts` (child types, homogeneity, caps, author gate),
  `src/routers/post.ts` (flattened pairing in POST + PUT, multer cap, sweep + `collectImageKeys`
  flatten), `src/constants/profileTypes.ts` (reuse).
- mobile: `src/utils/imagePicker.ts` (`selectionLimit`), `src/components/posts/composer/PostComposer.tsx`
  (hide the two image circles for BASIC viewers, `handleAttachImage` → gallery, caps, submit order),
  `src/hooks/useCanSharePost.ts` (→ shared `isPublicAuthorType`), `src/components/posts/PostContentItems.tsx`
  (photo tile aspect in the rail), `src/styles/components/posts.ts` (one rail-photo class).

## Rollout

1. API first (gate + caps + flattened pairing) — backwards compatible: old builds send one photo at
   a time and never a GROUP of images.
2. Mobile behind the API deploy. Until the mobile build ships, a BASIC user on the old build who
   attaches a photo gets the API's error on send — acceptable for the dev window; sequence the
   deploys close together for prod.
3. QA: 5 photos as primary on card / detail / Stories; 1 photo still plain; mixed group rejected;
   basic author rejected server-side even if the client is patched; delete sweeps all five keys;
   edit removes one photo and keeps the other four.

## Decisions (Skylar, 2026-10-18, typed after the voice session)

0. **Image gate = AFFILIATE + VERIFIED** ("it's both") — the public-author set, same as the
   gallery gate.
1. **One gallery of up to five, OR a single image — not five single images.** Either/or on the
   library slot; the camera snap is separate.
2. **BASIC users see neither the image picker nor photo capture.** Hidden, not disabled; no copy.
3. **Rail photo tiles: the post image's aspect, a bit smaller than a single photo**, so a gallery
   reads as a gallery — "similar to single product vs multi product carousel in posts".
4. **No camera into the gallery.** Capture stays its own single top-level photo.

No open questions remain; the plan is ready to build.

## Follow-up — shorter, mixed-width gallery tiles (Skylar, voice, 2026-10-23; mobile built same day)

Gallery tiles were all the same shape. Now (`galleryRail.ts
getPhotoRailTile(slotWidth, index, seed)`): every tile in a rail shares **one height — 7/10 of
the slot width**, a touch taller than the lone photo's 3:2 (first cut was 16:9; Skylar after
seeing it: "taller, but some images not as wide") — and its **width** comes from a four-step
cycle of shapes (4:3, 3:4 portrait, square, 4:5 portrait), capped at the 2026-10-19 "a little
thinner than the slot" width, so a wide tile sits beside a narrow one while the row stays level
("same height, just the width varies"). The cycle's start is seeded by the gallery's id (the draft's key in the composer), so a
post looks the same on every render and screen but two posts don't share a pattern. Each slot
takes its own tile's width, so photos still sit on the rail's 8 pt gap. The lone photo
(`contentImage`, 3:2) is unchanged. `PostContentItems` gains `railIndex` / `railSeed` for the
composer's per-child preview; the posted GROUP branch passes them itself. Pending device QA.
