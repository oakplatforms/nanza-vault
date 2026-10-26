---
tags: [nanza-api, nanza-admin, nanza-mobile, plan, sets, tags, posts, search]
---

# Sets & Tags Parity — shared images, sets on posts, set search

**Status:** plan → **API built 2026-10-17 (uncommitted, pending QA).** All three parts landed in
nanza-api in one pass: migrations `20261018120000_set_tag_value_parity` + `20261018130000_post_sets`
to run; types `0.1.114` to publish. **Mobile built 2026-10-17 (uncommitted, pending `npm run types:update` → tsc, then QA):** `compactLabel` / `tagCircleImage` helpers (`src/utils/tagLabels.ts`) across hashtags, chips, filter pills, composer picks, rails; `PostSet` picks in the composer (Set section slotted into the A–Z facet list at S, one n/10 counter, `setIds` on the wire); `SetScreen` (tag page twin + "See all cards"); set hashtags and entity set chips open the page; a tag/set page hides its own hashtag; `set` share slug + letter E, share landing card reused. Admin half not started. Supersedes and **replaces**
the retired *Sets as Tags* plan (deleted 2026-10-17): `Set` stays its own model. Instead of folding
it into the tag taxonomy, the two models are brought to the **same shape and the same surfaces** —
same three images, same post association, same detail page with a chat, same admin editor.
Spans nanza-api + nanza-admin + nanza-mobile.

### What the API build did (2026-10-17)

- Schema: `Set.logo → thumbnail`, `shortName` / `avatar` on both models, `Set.isPrimary`
  (default **true** so existing sets stay in pickers — renamed to `Set.isVisible` 2026-10-23, see
  *Visibility flag* below), `Set.index`, `Set.referenceCode`,
  `PostableType.SET`, `PostSet` join. Set share codes mint at create (letter **E**) and the
  post_sets migration backfills every existing set in a plpgsql loop.
- `PUT /set/:id` is multipart now (banner / thumbnail / avatar files + `remove*` flags, partial
  update); the field-switched `upload-image` / `delete-image` routes accept
  `banner|thumbnail|avatar` with `logo` as an alias. `GET /sets` takes `isVisible=` (was
  `isPrimary=`, still accepted) and orders index-then-A–Z. Deleting a set sweeps its page's posts.

### Visibility flag — `isVisible` beside `isPrimary` (2026-10-23)

`isPrimary` was the wrong word for "shown in pickers", so the flag is now **`isVisible`**, added
*additively* rather than renamed — a rename would have broken released mobile builds that send
`isPrimary=true` to `GET /sets`, and Prisma's diff for a rename is drop-and-add (data loss).
Migration `20261023120000_add_is_visible` adds `Set.isVisible` (default true, indexed with
`brandId`) and backfills it from `isPrimary`. The API filters and validates on `isVisible`;
`isPrimary` is a **deprecated alias** on the query string and the create/update body
(`src/utils/visibleFlag.ts` — `isVisible` wins when both arrive), and every write lands on **both**
columns so old builds keep reading the right value. Mobile pickers and set search now send
`isVisible=true`. Same treatment on `QueryList` ([[query-lists]]). **Not** on `BrandTag` — its
`isPrimary` means "on the card's primary tag line vs the attributes group", not visibility — and
**not** on `SupportedTagValue`, where `isPrimary` keeps curating the Home carousel and `isVisible`
is a separate admin hide/show flag (default true, filter only; nothing hides on it yet).
`Listing.isPrimary` (write-only, always false) was dropped in the same migration.
**Later:** once no released build sends `isPrimary`, drop the alias and the two columns.
- `PUT /supported-tag-value/:id` takes `avatar` / `removeAvatar` / `shortName`; the list search
  spans `shortName`; parent/child selects carry `shortName` + `avatar`.
- Posts: `setIds` on `POST /post` + `PUT /post/:id`, `validatePostAssociations` (one cap of 10
  across both arrays; on PUT the omitted array is read from the row so the cap is checked against
  what the post ends up with), `taggedSetPostsFilter`, `type=SET` read union, `SET` added to the
  author-feed homes, `postInclude.sets` (slim: id / name / displayName / shortName).
- Search: set `displayName` / `shortName` contains + `code` equals arms on `GET /search` —
  **retracted 2026-10-25** (see Part 3); sets are reached through the filter's `setId=`.
- Share: `set` slug in `shareTaxonomy`, resolver case, OG card (tag value template), meta,
  ogEdge regex, reserved username.

## The idea

`Set` (Outbreak, Welcome to Rathe, …) and `SupportedTagValue` (Warrior, Boltyn, First Edition,
…) are the two curated "things a card belongs to". They already look alike (name / displayName /
banner / description / brand) and the UI treats them alike in the search filter, but they have
drifted:

| | `Set` | `SupportedTagValue` |
|---|---|---|
| short label | — | — |
| banner | `banner` | `banner` |
| small image | `logo` | `thumbnail` |
| circle image | — | — (reuses `thumbnail`) |
| attachable to a post | **no** | yes (`PostSupportedTagValue`) |
| a post home with a chat | **no** | yes (`PostableType.SUPPORTED_TAG_VALUE`) |
| detail screen on mobile | **no** (tap → search filter) | `SupportedTagValueScreen` |
| global search by name | **no** | n/a (tags filter, not search) |
| admin editor | modal with banner + logo | detail page with banner + thumbnail |
| share link | — | `referenceCode` (`tag` slug) |

Four decisions (Skylar, 2026-10-17):

0. **A `shortName` on both models.** The display name is the full label ("Boltyn, Breaker of
   Dawn"); every compact surface — hashtags, chips, filter pills, composer picks, the circle
   caption — shows **`shortName` and falls back to `displayName`** when it's empty. Only the
   detail page hero shows `displayName`. Curators populate short names over time; nothing is
   derived automatically.
1. **Three standard images on both models**: `banner`, `thumbnail`, `avatar`. They serve
   different frames: banner is the wide hero, thumbnail is the 4:3 card tile (home rail
   `TagThumb`), avatar is the circle (`CircleThumb` — the children rail under a tag value, the
   search filter's circles, the composer's picks). **Circles read `avatar` and fall back to
   `thumbnail`**; tiles and heroes keep reading `thumbnail` / `banner`. Sets get all three so they
   can take any of those slots (homepage set rails later).
2. **Sets on posts, exactly like tags.** A post can carry sets and tag values together, **ten
   combined** (five sets + five tags is valid, eleven of anything is not). A set is also a post
   **home** — a set detail page with its own chat, mirroring the tag value page — so tapping
   `#Outbreak` on a post lands on the set page, not the search filter.
3. **Global search matches sets.** Typing "outbreak" into search returns the **cards** in that
   set (not the set row itself), alongside the usual name / number matches. **Retracted
   2026-10-25** — see Part 3.

## Part 1 — image parity

### Schema

```prisma
model Set {
  shortName   String?   // new
  banner      String?
  thumbnail   String?   // renamed from `logo`
  avatar      String?   // new
  …
}

model SupportedTagValue {
  shortName String?     // new
  banner    String?
  thumbnail String?
  avatar    String?     // new
  …
}
```

One migration: `ALTER TABLE "Set" RENAME COLUMN "logo" TO "thumbnail"` (keeps the S3 keys, no
re-upload) + `ADD COLUMN "avatar"`, `ADD COLUMN "shortName"` on both tables. Nothing is
backfilled into `avatar` or `shortName` — the fallback rules below cover the gap until curation
fills them.

`shortName` follows the standard-field convention (`name` = system key, `displayName` = full
human label, `shortName` = compact human label); it is **not** `Set.code` (WTR, OUT — a catalog
code that stays as it is) and is never unique.

`List` already carries `banner` + `logo` + `thumbnail`; it is **out of scope** here (its `logo` is a
different thing — a collection's brand mark). `Group` has `banner` + `thumbnail` and already
reads thumbnail as its avatar; also out of scope.

### API

- **`SupportedTagValue`** — `PUT /supported-tag-value/:id` already takes multipart `banner` /
  `thumbnail` with `removeBanner` / `removeThumbnail`; add `avatar` + `removeAvatar` on the same
  pattern (upload → delete old key → write; delete route sweeps all three keys). Resize: avatar
  uses the thumbnail's `inside` options at a smaller width (400, webp) — **no server-side
  cropping**; the admin uploads a pre-cropped square (Skylar, 2026-10-17).
- **`Set`** — converge on the tag value's pattern: `PUT /set/:id` becomes multipart and accepts
  the three files + three remove flags. The field-switched `PUT /set/upload-image/:id?field=` and
  `DELETE /set/delete-image/:id?field=` keep working for one release with `field` widened to
  `banner|thumbnail|avatar` (`logo` accepted as an alias for `thumbnail` so the current admin
  build doesn't break), then go. The set `DELETE` sweeps all three keys.
- **`shortName`** is a plain text field on both `PUT`s (max 24 chars, trimmed, empty → null)
  and rides on every select that already carries `displayName`.
- **Reads**: every select that names `thumbnail` on a tag value (`parent` / `child` in the
  children include, the trending/primary rails, post `supportedTagValues` includes) adds
  `avatar`. Set selects (`cardSelects`, `referenceResolver`, `postReferences`, `bulkListing`,
  `group`, `offer`, search `include=set`) rename `logo → thumbnail` and add `avatar`.
- **Types**: `SetDto.logo` → `thumbnail`, `avatar` on both DTOs. Types bump.

### Mobile

Two helpers, two rules, shared by tag values and sets:

- `compactLabel(value) = value.shortName || value.displayName || value.name` — used by every
  compact surface: `PostHashtags` (`#Boltyn` not `#BoltynBreakerOfDawn`), the search filter's
  chips and selected pills, the composer's picks and chip grid, `TagThumb` / `CircleThumb`
  captions, the entity detail's tag line (`getEntityTagSegments`), `FeedBulkCard`'s set chip.
  The **detail hero** (`SupportedTagValueScreen` / `SetScreen` title) keeps `displayName`.
- `tagCircleImage(value) = value.avatar ?? value.thumbnail` — used by every circle call site —

- `TagCarousel` `'circle'` variant (children rail under a tag value) and the `CircleThumb` it
  renders;
- `AccordionTagSection` in the search filter (`tagValue.thumbnail` → helper);
- the composer's picks / chips wherever an image shows;
- `fetchBrandSets` maps `set.logo` → `set.thumbnail` and carries `avatar`.

Tiles (`TagThumb`, `'card'` variant) and heroes keep `thumbnail` / `banner` directly — the whole
point of three fields is that each frame has its own asset.

### Admin

- **Tags › tag value detail** — a `Short name` input under `Display name` (with the fallback
  explained in its helper text); `TagValueImageField` already renders banner + thumbnail; add a
  third row for avatar (square preview). Label the three with their frames ("Banner — wide
  hero", "Thumbnail — 4:3 tile", "Avatar — circle") so curators upload the right crop.
- **Sets** — the create/edit modal is replaced by a **detail page** that reuses
  `TagValueImageField` for the three images, plus displayName / shortName / description / code /
  brand. The
  list page stays. (This is the "admin dashboard for sets similar to tags" Skylar asked for; the
  rest of that editor — `isPrimary`, `index`, `isTrending` — is Part 2's question 3.)

## Part 2 — sets on posts and the set page

### Schema

```prisma
enum PostableType {
  …
  SUPPORTED_TAG_VALUE
  SET            // new home: posts written on a set's page
  BRAND
  GROUP
}

model PostSet {
  createdAt DateTime @default(now())

  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId String
  set    Set    @relation(fields: [setId], references: [id], onDelete: Cascade)
  setId  String

  @@id([postId, setId])
  @@index([setId, postId])
}

model Post { …  sets PostSet[] }
model Set  { …  posts PostSet[] }
```

`PostSet` mirrors `PostSupportedTagValue` row for row. Cascade deletes the join, never the post
(same rule as tags: deleting a set must not delete user content).

A separate join table rather than reusing `PostSupportedTagValue` because sets are **not** tag
values any more (that was the retired plan) — keeping the two foreign keys honest is worth one
small table, and the "combined ten" cap is a validation rule, not a storage shape.

### Writes

`POST /post` and `PUT /post/:id` gain `setIds: string[]` beside `supportedTagValueIds`. One
validator change in `src/validation/post.ts`:

- `validatePostTags` becomes `validatePostAssociations(type, postableId, brandId,
  rawSupportedTagValueIds, rawSetIds)` → `{ supportedTagValueIds, setIds }`.
- **Combined cap**: `supportedTagValueIds.length + setIds.length <= MAX_POST_ASSOCIATIONS` (10,
  the renamed `MAX_POST_TAG_VALUES`). Each array is de-duplicated first; the error names the
  combined rule ("A post cannot have more than 10 tags and sets combined").
- **Brand scoping**: every set must have `set.brandId === brandId` when the home has a brand
  (same as tag values; `BULK` homes stay unscoped). A set with `brandId = null` can't be attached
  to a brand-scoped post.
- `resolvePostBrand` (the `switch (postableType)`) gains a `SET` arm: look up the set, return its
  `brandId`. The set page's composer then scopes exactly like the tag page's.
- `PUT` with `setIds` present replaces the set rows (deleteMany + create), mirroring the tag path's
  `wantsTags`.

### Reads

- `postInclude` adds `sets: { include: { set: { select: { id, name, displayName, shortName,
  avatar, thumbnail, brandId } } } }` beside `supportedTagValues` (whose select gains
  `shortName` too). `PostDto` gains `sets:
  PostSetDto[]`.
- **Set page union** — `GET /posts?type=SET&id=<setId>` returns the home arm (`postableType =
  SET AND postableId = :setId`) ∪ the tagged arm. The tagged arm is a second builder,
  `taggedSetPostsFilter(setId, viewer, friends, blocked)`, identical to
  `taggedTagValuePostsFilter` with `sets: { some: { setId } }` as the predicate. The three
  no-leak rules ride along unchanged: GROUP-homed posts excluded, BRAND-homed posts
  viewer-filtered, blocked authors excluded. **This is the one place a set-driven read can leak
  members-only or friends-only content — route every set read through it.**
- The brand feed's "public homes" list (`postableType in ['BRAND', 'SUPPORTED_TAG_VALUE']`,
  `src/routers/post.ts` ~L241/L256) adds `SET` so set-page posts show on the brand feed the way
  tag-page posts do.
- `GET /set/:id` is already guest-safe; it gains `include=brand` support (the mobile screen
  needs `brandId` for the composer, as the tag screen uses `include=brandTag`).

### Share links (recommended, see question 2)

`Set.referenceCode String? @unique`, minted lazily on first share like posts, slug `set` in the
og-sharing taxonomy (`ogHandler` / `metaHandler` / `ogEdgeHandler` + mobile's slug derivation).
The OG card reuses the tag value template with the set's banner/thumbnail.

### Mobile

- **Composer** — `TagPickerLayer` grows a **Set** section that sits in the SAME alphabet as the
  tag facets (at S — the filter screen's `facetSections` rule, Skylar 2026-10-17; brand's sets via
  `fetchBrandSets`, searchable through the same bottom pill). Picks are one
  list of `{ kind: 'tag' | 'set', id, displayName, image }`; the cap counter counts both; on
  submit they split into `supportedTagValueIds` + `setIds`. A set page's composer locks its own
  set the way the tag page locks its tag (`lockedSetIds`).
- **Hashtags** — `PostHashtags` renders sets and tags in one run (`#Outbreak #Warrior`), sets
  first. `onPressTag` routes by kind: set → `SetScreen`, tag → `SupportedTagValueScreen`.
- **Hide the page's own hashtag** (Skylar, 2026-10-17). On a tag value page or a set page every
  post in the thread carries that page's association, so the run reads `#Holla #Holla #Holla`
  down the screen. The thread passes `hiddenAssociationIds` (the same ids it already passes as
  `lockedTags` / `lockedSetIds` to the composer) and `PostHashtags` skips them. Purely a render
  rule — the row stays on the post, and the same post shows the tag everywhere else (home,
  group, profile, another tag's page). A post whose only association is the page's own renders
  no hashtag run at all.
- **`SetScreen`** — new, a copy of `SupportedTagValueScreen` with the set's banner hero,
  displayName, description, and `DetailChatSection` / `JoinChatFooter` with `postableType="SET"`.
  No children rail (sets have none). Registered beside `SupportedTagValueScreen` in
  `AppNavigator` with the same vertical entrance; `types/navigation.ts` adds `SetScreen: { setId }`.
- **Entry points flip to the page**: the set chip on `DetailTitleBlock` (listing / bid detail) and
  `FeedBulkCard` navigate to `SetScreen` instead of the search filter; the set page itself offers
  "See all cards" → the search screen filtered by `setId` (so the filter path survives, one tap
  deeper). The search filter's set accordion keeps filtering (it's a filter, not a link).
  **Except the entity screen (Skylar, 2026-10-20):** its labelled "Set" row (`renderSetRow`, above
  the primary-tag rows) went back to the search filter — on the card's own page the set reads as
  one more attribute, so it filters exactly like the tag rows beneath it. (The entity screen's
  `DetailTitleBlock` runs in `lockupMeta` mode and draws no set chip of its own.) The set page
  stays reachable from post #hashtags, share links and set search.
- **Post cards everywhere** (home, groups, profile tabs, item chats) read `post.sets` — same
  components, no new surface.

### Admin

- Posts list / reported-post queue show set hashtags next to tag hashtags (read-only).
- The Sets detail page (Part 1) is the curation surface; question 3 decides which flags it gets.

## Part 3 — global search by set

> **Retracted 2026-10-25 (Skylar, voice).** The set arms are out of `GET /search` again: a card
> search is cards only (`displayName` / `name` / `product.number`). Matching set names drowned
> the card you typed — "Hala" returned every card in the *Hala armory deck*, so the Hala hero
> card itself was lost in the set's cards. To browse a set, filter by it (`setId=`, the filter
> screen's Set chips) and search within. The filter screen's own set search and the composer's
> set picker (below) are unchanged. The rest of this section is the original design, kept for
> the record.

`GET /search` entity `where.OR` (the `searchTerm` branch in `src/routers/search.ts`) gains two
arms:

```ts
{ set: { displayName: { contains: searchTerm, mode: 'insensitive' } } },
{ set: { code: { equals: searchTerm, mode: 'insensitive' } } },
```

"outbreak" (or "OUT") returns every card in the set, ordered by the existing price-desc sort,
with the set chip already in each row (`include=set`). No new endpoint, no new UI: the result
list is the card list. The profile arm is untouched. Index: `Set.displayName` has none and the
table is small (hundreds of rows), so the join is cheap; add `@@index([displayName])` only if
the plan shows a seq scan on prod.

The set accordion in the filter screen and `setId=` stay as they are — filtering by set and
searching for a set are different gestures and both should work.

**Filter screen too (Skylar, 2026-10-17):** the filter's bottom search pill also finds sets — a
second debounced read, `GET /sets?brandId=&search=&isPrimary=true` (`useSearchBrandSets`, the
tag search's twin), drawn as a "Set" chip section above the tag hits with the facet Set section's
chips and tap handler. The composer's tag picker filters the brand's loaded sets client-side for
the same effect. Three places type a set name, three places get the set back: main search (its
cards), filter (a set filter chip), composer (a set association).

## Phasing & sizing

| Phase | Scope | Repos | Size |
|---|---|---|---|
| A — images + short names | migration (rename + `avatar` + `shortName`), STV + Set multipart avatar + `shortName`, read selects, types bump, admin inputs/avatar rows + Sets detail page, mobile `compactLabel` + `tagCircleImage` helpers across the compact surfaces | api, admin, mobile | ~1–1½ days |
| B — sets on posts | `PostSet` + `SET` home, validator + reads + union filter, types; mobile composer Sets section, hashtags, `SetScreen`, entry points; admin hashtags | api, mobile, admin | ~2–3 days |
| C — search | two `OR` arms | api | ~½ day |

A ships alone (no reader breaks; `logo` alias keeps the current admin working). C is independent
and can ship with A. B needs A's `avatar`/`thumbnail` on sets for the circles.

Migrations: one for A (`set_thumbnail_avatar`), one for B (`post_sets`) — run by Skylar as usual.
Types bump once per phase.

## Note — children ordering: index first, then A–Z (checked 2026-10-17)

Skylar's note: "add A–Z support instead of indexing for children tags — children are shared, so one
might be index 2 under one class and 7 under another." Checked against the code: the children
carousel on the tag detail page (`TagCarousel parentId=…` → `GET /supported-tag-values?parentId=`)
already orders by `index ASC NULLS LAST, displayName ASC`. So **index wins when set, and anything
without an index falls through to alphabetical** — clearing `index` on a parent's children is all
it takes to make that rail A–Z, no code change. Because `index` is one column on the shared child
row, a child can't hold a different position per parent; the only per-parent-correct ordering is
A–Z, which is what you get with `index` cleared. The composer's child chip sections sort
client-side (`sortTagValues`: numeric when both sides are numbers, else A–Z) and ignore `index`
entirely.

## Decisions log (2026-10-17, voice)

1. **Avatar crop** — none on the server. Avatars arrive pre-cropped square from the admin; the
   API only downsizes (`inside`, 400 wide, webp) like the other two images.
2. **Share links for sets** — yes, in phase B: `Set.referenceCode` + the `set` slug through the
   og-sharing slug-sync checklist.
3. **Curation flags on `Set`** — `isPrimary` + `index` in phase A (`isPrimary=false` replaces
   mobile's `isHiddenSetName` stopgap for PROMO-style sets; `index` orders pickers). Renamed to
   `isVisible` 2026-10-23 (alias kept).
   `isTrending` deferred until a set actually trends.
4. **Vouchers** — a set-homed post counts toward the [[affiliate-vouchers]] 5th-post hook like
   any other post (the hook counts by author, not home). Intended.
5. **Cap copy** — one combined `n/10` counter in the composer.
6. **Own-hashtag suppression** — a tag/set page hides its own hashtag on every post in its
   thread; shown everywhere else (see Part 2 → Mobile).

## Open questions

1. **`shortName` length.** 24 characters proposed (fits a chip and a hashtag); the ask was
   "about four characters" for the longest hero names, so curators will mostly go shorter.

## Related

- [[posts]] — the association model, `taggedTagValuePostsFilter`, the home-vs-tags split this
  extends to sets.
- [[tag-taxonomy-v3-is-child]] — the single-table taxonomy; sets deliberately stay outside it.
- [[trending-tag-search]] / [[post-carousel-and-tagging]] — the composer picker this adds a
  Sets section to; the second deferred "set association" twice, this is where it lands.
- [[og-sharing]] — the slug taxonomy a `set` share type joins (question 2).
- [[image-cdn]] — on-demand resizing; avatar/thumbnail/banner are stored once and sized per
  tile by the client.
- [[query-lists]] — `QueryCriterionField.SET` is unchanged by this design.
