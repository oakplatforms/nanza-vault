---
tags: [nanza-mobile, nanza-api, solution-design, plan, collections]
---

# My Library — the locked primary collection (Plan)

> **2026-10-25:** My Library's detail screen shows the edit pencil again — the form holds one field, the **Show value** toggle (`List.isValueVisible`, see [[collections]]); name, banner and delete stay locked. "Never private" is moot: private collections were retired the same day.
>
> **Status: built 2026-10-24 (night), both halves; two-way library sync built 2026-10-24 (evening, uncommitted), pending QA.**
> nanza-api: `src/validation/list.ts` guards (rename / un-flag / banner / thumbnail / logo /
> delete all 409), the name pinned to "My Library" at every create site, and
> `mirrorAddsToPrimary` (adds to any non-primary collection also land in My Library) — the
> guards and name pin are committed, the mirror is uncommitted. nanza-mobile (uncommitted):
> mosaic = primary-only in `CollectionThumb`, `ListThumbnailImage` and `CollectionAddRow`; a plain
> `thumbCardPlain` fill for bannerless non-primary tiles; no pencil on My Library's detail
> screen; Title and Banner rows dropped from the edit form for the primary. Skylar ran the
> DB rename the same night, so the two `MY_LIBRARY_LABEL` overrides are gone — the stored
> `displayName` is the label. Fold into
> [[collections|Collections]] when QA passes.

## The ask (Skylar, voice, 2026-10-24)

- **The primary collection is "My Library" and it's fixed.** Everyone gets it on sign-up. Skylar
  renames every existing primary to *My Library* in the database by hand; after that nobody can
  rename it, give it a banner, or (already true) make it private or delete it.
- **The mosaic grid means "primary".** Today any collection without a banner shows the card
  mosaic. From now on only My Library shows it — everywhere a collection tile appears, including
  the add-to-collection screen — so the grid *is* the primary's identity.
- **Non-primary collections stay editable:** name and banner (gallery only, no camera — someone
  asked, parked). With no banner they show **nothing** for now (a plain tile), not the mosaic. A
  default image may come later.

## Today

- `List.isPrimary` is the flag; the API already refuses a private primary (`PUT /list/:id`) and
  mobile already hides Delete and the visibility toggle for it (`CreateCollection`,
  `CreateCollectionScreen`). Nothing stops renaming or a banner on the primary, server or client.
- `ProfileSections` and `ProfileSeeAllScreen` each carry a `MY_LIBRARY_LABEL = 'My Library'`
  override that relabels the primary on the profile only; the detail screen, picker and
  add-to-collection rows still show the stored name.
- Tiles: `CollectionThumb` (profile Library grid, See-all) and `CollectionTargetCard`
  (pickers, read-only profile) show the banner if set, else `CollectionMosaic`. The picker and
  add-to-collection rows use the small `ListThumbnailImage`. `CollectionDetailScreen` renders a
  banner hero when there is one and a plain header otherwise.
- The OG share card for any collection is a mosaic (`og/`), independent of these tiles.

## Design

### API (nanza-api) — the primary is immutable

- `PUT /list/:id`: when the row is primary, reject `name`, `displayName`, and any `isPrimary`
  change (409-style `Error` through `generatePrismaError`, message "My Library can't be
  renamed"). Everything else on the primary (entityList writes, `index`) stays allowed. The
  existing private-primary guard stays.
- `PUT /list/upload-image/:id` and the matching delete-image: reject `banner`, `thumbnail` and
  `logo` on a primary list ("My Library can't have a banner").
- `DELETE /list/:id`: confirm the primary guard exists; add it if not (verify in build — the
  route today checks ownership only).
- Primary creation: no client sends `isPrimary: true` today. `POST /user` (admin-only, the
  sign-up trigger) now renames any primary in the payload to *My Library* and **adds one when the
  payload has none**; `POST /list` pins the name when `isPrimary`; the data-sync on-the-fly
  collection (`services/collection.ts`, an account with no list yet) is created as the primary
  *My Library* instead of a nameless "Collection".
- No schema change. `isPrimary` stays the flag; no `locked` column.
- One-off data: Skylar's manual `UPDATE "List" SET "name"='My Library', "displayName"='My
  Library', "banner"=NULL, "thumbnail"=NULL WHERE "isPrimary" = true AND "type"='COLLECTION'`
  (DBeaver). After it runs, the mobile `MY_LIBRARY_LABEL` overrides come out.

### API — My Library is the master inventory (added 2026-10-24, two-way since the same evening)

Skylar: a save into My Library saves only there; a save into any other collection (a "binder")
saves into that collection **and** into My Library — and, added by voice the same evening,
removals go both ways too. Built as `syncPrimaryCollection` in `services/collection.ts`
(replacing the add-only `mirrorAddsToPrimary`), called inside the transaction of every
collection write path, after the caller's own rows are written, over that write's deltas
(after minus before per list/entity): `POST /entity-list`, `PUT /entity-list/:id` (the
stepper), `DELETE /entity-list/:id`, `POST /entity-list/batch`, `POST /entity-list/batch-by-entity`
(the add-to-collection screen), `POST /scan-items/batch/commit-collection`, and the order
settlement's `transferCollectionEntities`. Rules:

- **Binder → library, same amount, both ways.** Filing 3 adds 3 to the library; stepping 3 → 1
  takes 2; removing the row takes 3. The library floors at 0 (the row goes at 0).
- **Same cards, several binders, once.** When one write files a card into several binders (a
  scan into two, or the add screen), the library receives that card's largest added quantity,
  not the sum; the same for removals, so a binder-to-binder move never double-counts.
- **Adds straight into the library mirror nowhere.**
- **Library → binders on removal.** Taking N out of the library (step down, row removed, a sale
  settling) takes it back out of the binders: the library's *loose* copies — its total above
  what the binders hold — go first, the rest is spread evenly over the binders that hold the
  card (largest first for any odd one, each capped at what it holds). Examples: library 12,
  binders 6 + 6, take 6 → 3 + 3; library 7, one binder with 3, take 3 → binder keeps 3 (loose
  copies went); take 5 → binder 2. **A card at 0 in the library leaves every binder.**
- **Explicit beats mirrored.** A write that touches the library row itself never also receives
  the mirror of the binders in that write, and a library drop in that write skips the binders
  written beside it — the add-to-collection screen sends the library and a binder together and
  the user typed both.
- The primary is `ensurePrimaryCollection`: the flagged row, else (legacy) the oldest
  collection, else My Library created on the spot. `findPrimaryCollection` is the non-creating
  lookup; the order transfer withdraws from it (not from "any collection" as before) and pushes
  the sale down into the seller's binders.
- Mirrored rows don't bump the affiliate "collection cards" counters; only the direct rows do.
- `PUT /list/:id`'s nested `entityList` writes aren't mirrored (no client uses them).
- **Deleting a whole binder (`DELETE /list/:id`) leaves the library alone** — the folder goes,
  the cards stay owned as loose copies. Open to change if Skylar wants the opposite.

### Mobile (nanza-mobile)

- **Tiles — the mosaic is primary-only.** `CollectionThumb`, `CollectionTargetCard` and
  `ListThumbnailImage` branch on `isPrimary` first:
  - primary → `CollectionMosaic` (bleed / fit as today), banner ignored;
  - non-primary with banner → banner cover;
  - non-primary without banner → a plain tile: surface fill + the usual `ThumbShade`, name and
    count as today, no image. (The "later: default image" slot.)
  The add-to-collection rows and the collection picker therefore show the grid for My Library.
- **Detail screen:** no change to layout. On the **primary the edit pencil is hidden** — once
  Title, Banner and Thumbnail join Visibility and Delete as locked, nothing is left to edit
  (Skylar, 2026-10-24). Non-primary collections keep the pencil.
- **Edit form (`CreateCollection`) for the primary:** the Title row and the Banner / Thumbnail
  rows are hidden (not disabled); the form shows the name as a static heading ("My Library")
  and only the fields that remain editable — which, for the primary, is nothing, hence the
  hidden pencil above. The gating stays in the form too, as a backstop for any other entry.
- **Create form:** unchanged; new collections are never primary (`isPrimary: false` already).
- **Banner picker:** gallery only, as today (`launchImageLibrary`); the camera icon on the
  "Add banner" button is decorative — no capture path is added.
- **Labels:** the two `MY_LIBRARY_LABEL` overrides are gone (DB rename done 2026-10-24); the
  stored `displayName` is the label everywhere.
- Query freshness: no new keys; edits still invalidate `['collections', …]` as today.

### Out of scope

- Camera capture for banners (parked). A default banner image for non-primary collections.
- Changing the OG share mosaic.

## Open questions

1. ~~Pencil on the primary's detail screen~~ **Answered (Skylar, 2026-10-24): hide it.** With
   Title, Banner, Thumbnail, Visibility and Delete all gone for the primary, the form has
   nothing left, so `CollectionDetailScreen` drops the edit circle when `isPrimary`.
2. ~~Lock `thumbnail` too?~~ **Built as yes** — banner, thumbnail and logo are all refused on
   the primary (the grid is the identity). Say so if that's wrong.
3. ~~Error copy~~ **Built as** "My Library can't be renamed" / "My Library can't have a banner"
   / "My Library can't be deleted" / "My Library stays your primary collection", all 409.

## Order of work

API guards (small, one PR) → Skylar's DBeaver rename → mobile tiles + form gating → drop the
label overrides. About a day across both repos.
