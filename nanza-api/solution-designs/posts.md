---
tags: [nanza-api, solution-design, posts, content]
---

# Posts — Comments Become the Platform's Content Unit

> **Status:** **Backend implemented 2026-07-31** (nanza-api only — admin and mobile still call the
> old endpoints). Supersedes [[comments|Comments]]: `Comment`, `CommentSubTagValue`, and
> `CommentableType` are gone from the schema. Stage 1 ([[tag-taxonomy-v2|Tag Taxonomy v2]]) landed
> in the same pass. **The migration has not been generated or run** — see *Remaining* below.
>
> **2026-08-01:** the *Chat release* section below extends this design — reference content items
> (LISTING/BID/ENTITY), likes via `SavedItem`, and the composer contract. Design only; mobile
> surface in [[../nanza-mobile/solution-designs/posts-chat|posts-chat]].

## The idea

A **Post** is what a comment becomes when it grows up. Today `Comment` is a body string attached
polymorphically to a listing, bid, entity, or bulk listing. A Post keeps that exact shape — same
polymorphic targeting, same one-level replies — and adds:

1. **Rich content** — ordered **content items** (text blocks, images, links) on top of the body.
2. **Taxonomy tagging** — a post can reference supported tag values and secondary tag values
   independent of where it lives.
3. **Brand posts** — posts whose home is a **brand**, rendered as the homepage feed, and allowed to
   tag across *several* brand tags at once (`class → Warrior` and `print → First Edition` together).

The word changes everywhere: what the UI calls a comment becomes a post. Replying to a post creates
another post.

## Data model

### `Post`

```prisma
enum PostableType {
  LISTING
  BID
  ENTITY
  BULK
  SUPPORTED_TAG_VALUE
  SECONDARY_TAG_VALUE
  BRAND
}

model Post {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // The primary text. Every backfilled comment has one; a plain reply is
  // nothing but one. Optional only because a media-first brand post may lead
  // with content items.
  body   String? @db.Text
  status Status  @default(ACTIVE)

  // Where it lives. Always a real row — BRAND posts point at the Brand.
  postableType PostableType
  postableId   String

  // One level of threading, exactly as Comment does today.
  parent   Post?  @relation("PostThread", fields: [parentId], references: [id], onDelete: Cascade)
  parentId String?
  replies  Post[] @relation("PostThread")

  account   Account @relation(fields: [accountId], references: [id], onDelete: Cascade)
  accountId String

  contentItems       ContentItem[]
  supportedTagValues PostSupportedTagValue[]
  secondaryTagValues PostSecondaryTagValue[]

  @@index([postableType, postableId])
  @@index([parentId])
  @@index([accountId])
}
```

**`BRAND` replaces the earlier `HOMEPAGE` singleton idea** (decided). The homepage feed for a brand
is simply `posts WHERE postableType = 'BRAND' AND postableId = :brandId`. This keeps `postableId`
non-nullable (no singleton special-casing in handlers), and answers brand scoping structurally —
switching brands switches feeds because the home *is* the brand.

### Who can post where (decided)

- **Listings, bids, entities, bulk, tag pages** — any authenticated account, as comments work today.
- **`BRAND` posts require `profile.type === 'AFFILIATE'`** — this is the first feature gated by
  [[profile-types|Profile Types]]. Everyone else reads the brand feed; only affiliates (and admin
  tooling, later) publish to it. `validateBrandPostPermission` no-ops for every other home, so it
  is safe to call unconditionally.

  **Enforced on create AND on edit** (`POST /post` and `PUT /post/:id`), against the profile's
  type *at the time of the request*. Edit was missing the gate until 2026-08-08, and ownership
  alone was not a substitute: `profile.type` is mutable (`PUT /profile/:id/type`), so an affiliate
  who published to a brand feed and was later demoted kept permanent rewrite access to that post —
  a one-time check at create is not a durable authorization boundary when the thing it checks can
  change. Anything gated on profile type must re-check on every mutating request.

  `DELETE /post/:id` deliberately does **not** re-check: removing your own content after demotion
  is acceptable, rewriting it under a brand's banner is not.

  Replies are covered by the same gate without extra code — a reply carries its parent's
  `postableType`, so a reply into a brand feed is a `BRAND` post.

### `ContentItem`

```prisma
enum ContentItemType {
  TEXT
  IMAGE
  LINK
  // VIDEO deliberately deferred — needs upload limits, transcoding, and a
  // playback story that images don't. Add when that pipeline exists.
}

model ContentItem {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())

  type  ContentItemType
  index Int              // ordering within the post

  // Populated per type: TEXT→body, IMAGE→media (S3 key), LINK→url.
  body  String? @db.Text
  media String?
  url   String?

  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId String

  @@index([postId, index])
}
```

Separate rows rather than a JSON blob, for the same reason `QueryCriterion` is a table: composers
add and reorder items individually, and per-type validation stays in SQL-shaped code.

**Where content items are allowed (decided):** only on posts whose home is `BRAND`,
`SUPPORTED_TAG_VALUE`, or `SECONDARY_TAG_VALUE`. Listing/bid/entity/bulk threads stay text-only —
media in marketplace threads is a moderation surface that doesn't exist yet. Enforced in the create
handler; revisit when moderation tooling lands.

**`body` vs `TEXT` items (decided):** `Post.body` is the primary text — the thing a backfilled
comment is, the thing a reply is. Content items are *additional* rich blocks below it. There is one
way to write a plain comment, not two.

### Tagging

```prisma
model PostSupportedTagValue {
  post                Post              @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId              String
  supportedTagValue   SupportedTagValue @relation(fields: [supportedTagValueId], references: [id], onDelete: Cascade)
  supportedTagValueId String

  @@id([postId, supportedTagValueId])
  @@index([supportedTagValueId])
}

model PostSecondaryTagValue {
  post                Post              @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId              String
  secondaryTagValue   SecondaryTagValue @relation(fields: [secondaryTagValueId], references: [id], onDelete: Cascade)
  secondaryTagValueId String

  @@id([postId, secondaryTagValueId])
  @@index([secondaryTagValueId])
}
```

**Cascade deletes the join rows, never the post.** Deleting a tag must not delete user content.

**Tag validation (revised 2026-08-01, Skylar): the home type decides the tagging freedom.** The
post's home says where it comes from, and that's the rule:

- **`BRAND` posts — multiple associations.** Up to 5 supported values and 25 secondary values,
  anywhere within the brand, spanning brand tags freely (no same-tree restriction). Tagging
  `class → Warrior` and `print → First Edition` together is the intended headline case.
- **Every other home — at most ONE supported tag value, and it's optional.** Secondary values must
  be **linked** (via `SupportedTagValueSecondaryTagValue`) to that one value. On a
  supported-tag-value page the page's own value *is* the association, so its linked secondary
  values are available with no pick; a secondary-tag-value page is itself the home and carries no
  further associations. No association → no secondary values. Listing/bid posts will usually carry
  none at all.

This supersedes the earlier "permissive within the brand" rule for non-brand homes — that
permissiveness now belongs to brand posts only, which is where the multi-tag fan-out actually
lives. It also killed the `isRichPostableType(type) ? brandId : null` scoping pattern: no more
null-scope special-casing, since the rule branches on the home type directly.

## Home vs tags, and what each page shows

Every post has a **home** (where it was written) and, separately, **tags**. These do different
jobs, which is why only one of them is a join table:

- **Home** is `postableType` + `postableId` — exactly one, so a polymorphic pointer covers it. A
  listing is only ever a home; nothing tags a listing, and a post can't live in two places. That's
  why there is no `PostListing` join.
- **Tags** are references to things a post is *about*, and there can be several — a brand post may
  tag `class → Warrior` **and** `print → First Edition`, spanning two brand tags. A single
  `secondaryTagValueId` column couldn't express that, so tags need real join tables.

A tag page shows the union of both:

```
posts WHERE (postableType = 'SECONDARY_TAG_VALUE' AND postableId = :id)          -- posted here
   OR (id IN (SELECT postId FROM PostSecondaryTagValue WHERE secondaryTagValueId = :id))  -- tagged here
```

A post written on the Warrior page that tags Boltyn appears in both places, authored once. The same
union applies to supported tag values. The brand feed stays home-only (brand posts aren't a tag
target).

## Scaling the union

The union is the only query here with a non-obvious cost, so it's worth being explicit about where
it holds up and where it doesn't.

**What's fine at any size.** Marketplace threads (listing/bid/entity/bulk) never touch the union —
they're a straight `postableType`/`postableId` lookup, and that's the overwhelming majority of
traffic. The tag side of the union has its own index on the join.

**Where it degrades.** Postgres can't satisfy an `OR` across two access paths with one index, so a
tag page runs both branches and merges them (a BitmapOr), then **sorts the merged set** for
`ORDER BY createdAt DESC` before slicing a page. The sort is the real cost: it materializes every
matching post to find the newest ten. Fine for a tag with 50 posts; wasteful for one with 50,000.
Page-based `OFFSET` compounds it — page 20 still produces and discards everything before it.

**Indexes carry the common case** (applied 2026-07-31):

```prisma
@@index([postableType, postableId, createdAt])  // Post — home branch, pre-ordered
@@index([supportedTagValueId, postId])          // join — tag branch, covering
@@index([secondaryTagValueId, postId])
```

Trailing `createdAt` lets the home branch come back already sorted, so top-level thread reads need
no sort at all. `postId` on the joins makes the tag lookup covering — it resolves without a second
hop to the join row.

**The escape hatch, when the union itself gets slow.** Stop making it a union: write a
`PostSecondaryTagValue` row for a post's *own* tag value at creation, so a post homed on Boltyn
also carries a join row pointing at Boltyn. The tag page then becomes a single indexed join with no
`OR`:

```sql
SELECT ... FROM "Post" p
JOIN "PostSecondaryTagValue" j ON j."postId" = p.id
WHERE j."secondaryTagValueId" = ? ORDER BY p."createdAt" DESC
```

That's a deliberate denormalization — the home is represented twice — and it adds a write-path
invariant to keep correct, which is why it isn't done up front. Take it when tag-page reads show up
in slow queries, not before.

**If deep pagination ever matters**, tag feeds want keyset pagination (`WHERE createdAt < ?`)
rather than `OFFSET`. `paginatePrisma` is page-based, so that's a real change rather than a tweak —
only worth it if users actually scroll far.

## Replies

Unchanged from `Comment`, restated because they're real constraints:

- A reply is a `Post` with `parentId` set; **a reply cannot have replies** (handler-enforced, as
  `validateParentComment` does today).
- A reply inherits its parent's home, so a thread never splits across pages.
- Replies inherit the root's tags and **carry no content items** — a reply is plain text (decided).

## Rollout: three stages, in order (decided)

1. **[[tag-taxonomy-v2|Taxonomy v2]]** — drop/recreate the empty `SubTagValue` tables as
   `SecondaryTagValue` + join; rework the admin dialog and mobile rail. Small, and it means the
   comment-side join only migrates once.
2. **Comment→Post rename** — mechanical and id-preserving (below). Big blast radius, low logic
   risk. No new features in this stage.
3. **`ContentItem` + brand posts** — the genuinely new surface: composer UIs, affiliate gating,
   image caps. VIDEO stays deferred beyond this.

## Migration from `Comment` (stage 2)

> [!important] **Requirement: old nanza-mobile builds must not break — but the comment *feature*
> need not keep working in them.**
> `GET /app-version` drives a *prompt*, not a forced upgrade, so shipped builds run indefinitely
> and will keep calling `/comments`. However **the `Comment` table is effectively empty** (the
> feature shipped recently and carries no meaningful user data), so there is nothing to preserve.
> The bar is therefore: *old builds must render and navigate without crashing*, not *old builds
> must show conversations*.

That makes stage 2 far cheaper than a true compatibility layer.

### The migration

1. **Create** `Post`, `ContentItem`, and the tag joins.
2. **No backfill.** Confirm with `SELECT count(*) FROM "Comment"` first; assuming it's empty (or
   only test rows), `Comment` is dropped rather than migrated. If a handful of real comments exist,
   copy them across preserving ids — it's a one-query backfill, not a project.
3. **Point the new clients at `/posts`.**
4. **Leave thin legacy shims** at the five old paths so old builds get well-formed empty responses
   instead of 404s or 500s (below).
5. **Drop the shims** when old-build traffic dies off.

### Legacy shims: shaped, not functional

Because there's no data to serve, the old endpoints become deliberate no-ops that satisfy the
client's parser:

| Old endpoint | Shim behaviour |
|---|---|
| `GET /comments` | `200` with an **empty paginated envelope** — `{ data: [], total: 0, page: 0 }`, matching `paginatePrisma`'s shape |
| `GET /comment/:id` | `404` (already a path the client handles) |
| **`POST /comment`** | **`4xx` with `errorMessage: 'Comments are being migrated to posts.'`** |
| `PUT` / `DELETE /comment/:id` | Same `4xx` message |

The shape matters more than the status. An empty *envelope* is safe; a bare `{}` or a `500` is
what actually crashes a screen, because the client does `data?.pages.flatMap(p => p.data)` and
will throw on a missing `data` array. Same for `commentCount` on listing/bid/bulk responses (28
usages in mobile): keep emitting the field as `0` rather than removing it, so nothing reads
`undefined` where a number is expected.

**Writes return an honest error, not a fake success** (decided). A fabricated `200` would make the
comment appear locally and vanish on refresh, silently eating what the user typed.
`CommentThread` already renders `errorMessage` from a failed write, so the message surfaces
without any client change.

The message is deliberately **"Comments are being migrated to posts."** — *not* "please update the
app." At the time this ships there may be no newer build to update to, so telling users to update
would be advice they can't act on. Describing the actual state is both true and self-explanatory,
and it stays true whether or not a new release exists yet.

**Blast radius:** ~9 API routers reference `commentableType`/`commentCount` (listing, bid,
bulkListing, group, savedItem, universal, supportedTagValue, comment, referenceResolver) and **26
mobile files** touch comments. With no data migration, the work is the rename plus the new post
surfaces — the shims are a single small router.

### The same applies to Taxonomy v2's rename

Mobile's shipped build calls `/sub-tag-value(s)`. Those tables are empty too, so the same treatment
works: keep the paths returning empty envelopes (or, since it's nearly free, have them read
`SecondaryTagValue` — see [[tag-taxonomy-v2|v2]]). Either way old builds show an empty rail rather
than erroring.

### Sunsetting the shims

Removal stays a later, evidence-driven decision: log a counter per legacy route tagged with the
caller's app version, raise `LATEST_APP_VERSION` to nudge the tail, then delete.

## What shipped (2026-07-31) and what's left

**Built in nanza-api:**

- `src/routers/post.ts` — the `/posts` surface. `GET /posts` runs the union for tag targets and a
  plain home lookup elsewhere. Creating a reply copies the parent's tags. `PUT` replaces the
  item/tag sets in a transaction; deleting a post with replies tombstones it (status `DELETED`,
  content items removed) so the thread survives.
- `src/validation/post.ts` — `validatePostableExists` returns the target's `brandId`, which is what
  scopes tags; affiliate gate on `BRAND`; content items gated to brand/tag homes; `LINK` urls
  checked for an http(s) scheme.
- `src/utils/postCount.ts` — counting plus **`withPostCount(record, count)`**, which emits
  `postCount` *and* a `@deprecated` `commentCount` alias. All eleven emission sites across six
  files go through it, so retiring the alias is one line.
- `src/routers/comment.ts` — reduced to legacy shims (below).
- `src/routers/secondaryTagValue.ts` — replaces `subTagValue.ts` (see
  [[tag-taxonomy-v2|Tag Taxonomy v2]]).

**Deltas from the design, decided during implementation:**

- **`ContentItemType.TEXT` → `RICH_TEXT`**, and **`VIDEO` is in the enum after all** — as a
  reserved value rejected by an `ENABLED_CONTENT_ITEM_TYPES` allowlist in validation. Adding an
  enum value later is a migration; adding it now makes enabling video a one-line validation change.
  The `VIDEO` case in the per-type switch is unreachable but present, so the switch stays
  exhaustive and there's one obvious place to fill in.
- **Both count fields are emitted**, rather than keeping only the legacy name. New clients read
  `postCount` and never learn the old one, which is what makes the alias genuinely retireable.
- Write shims return **`410 Gone`** (not a generic 4xx) — the resource is deliberately gone, which
  is exactly what 410 means, and it distinguishes migration from a transient failure in logs.

**Remaining:**

1. **Generate and run the migration.** Not done — it needs a live database to diff against, and
   Docker wasn't running. Before applying, confirm `Comment`, `CommentSubTagValue`, and
   `SubTagValue` are empty in dev *and* prod; the migration drops all three.
2. **Publish `@oakplatforms/types` 0.1.75** (bumped, built, `PostDto`/`ContentItemDto`/
   `SecondaryTagValueDto` present, `CommentDto` gone).
3. **Admin and mobile passes** — both still call the old endpoints.

**Visible consequence worth planning for:** `commentCount` now counts posts, and posts start empty.
Prod mobile will show `0` on feed cards after the migration. Not a crash — but it reads as counts
"disappearing" rather than as a migration, so it's worth timing alongside the client releases.

## Decisions log (2026-07-30)

- `BRAND` home target instead of a `HOMEPAGE` singleton; `postableId` stays non-nullable.
- Brand posts: **AFFILIATE-only** creation (first consumer of `ProfileType`).
- Content items: tag + brand posts only; listings/bids stay text-only for now.
- `VIDEO` deferred; `TEXT`/`IMAGE`/`LINK` ship first.
- Tagging: permissive within the brand, no same-tree restriction; multi-tag allowed.
- `Post.body` is the primary text; content items are additional blocks; replies are plain text.
- Three-stage rollout, taxonomy v2 first.
- Rate limits: reuse the 50/day account cap for text posts; image-bearing posts need their own
  (lower) cap — set when stage 3 is scoped.
- **Old mobile builds must not break, but need not keep the feature** (Skylar): `Comment` is
  empty, so no backfill and no true compatibility layer — just thin shims at the five old paths
  returning well-formed empty envelopes so old builds render. Writes return
  `4xx "Comments are being migrated to posts."` — an honest error rather than a fake success, and
  deliberately not "update the app" since a newer build may not exist yet. Shim removal is a
  later, telemetry-driven decision.

## Chat release (2026-08-01) — API implemented 2026-08-01

The next release turns every post surface into a **chat**: listings, bids, lots, supported tag
values, and secondary tag values each keep their own thread (exactly where comments used to live),
entered through a fixed "Join the chat" input. **No brand/homepage posts yet** — the composer's
expanded view is deliberately the future brand composer, but creation stays per-target this
release; the affiliate brand feed comes later. Mobile surface:
[[../nanza-mobile/solution-designs/posts-chat|posts-chat]]. Mocks for the post cards are
incoming and get folded in here when they land.

### Reference content items: LISTING / BID / ENTITY

Three new `ContentItemType` values that don't carry their own media — they **point at a system
record**, and the renderer draws the record's own assets (a listing item renders from
`listing.entity.image`, price, condition — the same data a feed card uses). The motivating flow:
someone in the *Warrior* chat says "I'm selling this" and embeds their actual listing.

**`LISTING` covers both Listings and Lots** (decided): there is no separate BULK type — a lot is
just a kind of listing to the composer and the renderer. Expansion resolves a LISTING
`referenceId` in two batched steps: `findMany` against `Listing` first, leftovers against
`BulkListing`; the `reference` object carries a `kind: 'listing' | 'bulk'` so the client picks the
right card.

```prisma
enum ContentItemType {
  RICH_TEXT
  IMAGE
  LINK
  VIDEO     // still reserved/rejected
  LISTING   // new — referenceId → Listing OR BulkListing (lots)
  BID       // new — referenceId → Bid
  ENTITY    // new — referenceId → Entity (cards)
}

model ContentItem {
  // …existing fields…
  referenceId String?   // set only for reference-type items
}
```

Like the post's own polymorphic home, `referenceId` has **no FK** (the types share it), so:

- **Write-time validation** (`validation/post.ts`): the referenced record must exist; a
  LISTING/BID reference must be ACTIVE (PUBLISHED for lots) at post time.
- **Read-time expansion:** `GET /posts` expands references server-side — each reference item comes
  back with a `reference` object shaped like the feed-card DTO for its type, so mobile renders with
  zero extra round trips. Expansion batches by type (one `findMany` per type per page, not N+1).
- **Tags stay slim on the wire** (decided): post responses do NOT expand full tag DTOs (no
  banners/descriptions/thumbnails). Each post carries only
  `tags: [{ id, displayName, kind: 'supported' | 'secondary' }]` — exactly enough to render a
  hashtag row and navigate on tap. The tag pages themselves load their own full data.

#### Entity references carry fair market, not the order book (2026-08-08)

`PostEntityReference` used to expand `lowestAsk` + `highestBid`. Nothing on the card rendered
them: the Figma spec (33373-76173) draws a **fair market** price, which is `Product.price` — a
different number from the lowest ask. Printing an ask under the FAIRMARKET mark would assert
something untrue, so mobile had been shipping a hardcoded `999.99` placeholder rather than use
the field it was given.

Both order-book fields are now **removed** from the reference and replaced by a single derived
`price`: `entitySelect` pulls `product: { select: { price: true } }` (Product is 1:1 with Entity)
and the resolver flattens it to `price: number | null`, coercing the Decimal to a real number the
same way listing/bid references do. `null` means the entity has no product row — **unpriced, not
free** — and the client drops the price pill entirely rather than rendering `$0.00`.

The general rule this reinforces: the payload budget is about *what the card renders*. A field
that's cheap to select still doesn't belong on the reference if no card draws it — and a field
that's nearly-but-not-quite the right number is worse than an absent one, because a client will
either misreport it or paper over it with a placeholder.

#### Lot references: the cover image leads

A `kind: 'bulk'` reference carries both `image` (the lot's own cover, `BulkListing.image`) and
`childImages` (up to 6 child pictures for the tile's carousel). **`image` is the first page when
it exists** — it's the seller's deliberate choice of how the lot presents — with the children
following it, deduped against it.

The API has always resolved this correctly (`image: bulk.image ?? childImage`). The client had
the precedence inverted (fixed 2026-08-08): it checked `childImages` first and only fell back to
`image`, so a lot with any children — i.e. essentially every lot — never showed its cover in a
post. `BulkThumb`, the lot card used everywhere else, has always done `bulk.image || first
child`; the post content item was the one surface that disagreed.

**Shipping note:** the contract lives in `packages/types` (`PostEntityReference`), which mobile
consumes from the GitHub registry. The client change is only live once that package is published
past 0.1.87 and reinstalled — until then mobile typechecks against the old shape.

### Payload budget (decided): post payloads must not balloon

The slim-tags rule is one instance of a general rule for `GET /posts`:

- **Reference expansion is a `select`, not an `include` tree.** The `reference` object carries only
  what the card renders — entity image/name, price, condition name, `kind` — not the full
  listing DTO with its nested account/brand/set/relations. Shape it like the leanest feed-card
  select in the codebase, not like a detail response.
- **Replies are never embedded in list payloads** — reply counts as scalars, threads fetched
  per-post via `parentId` as today.
- **Counts are scalars** (`likeCount`, `replyCount`, `viewerHasLiked`) computed with batched
  groupBy maps, never arrays of who-liked.
- A page of 10 posts should stay in the low tens of KB even when every post embeds a reference.
  If a new field wants onto the post payload, the question is "does a card render it?" — if not,
  it belongs on a detail fetch.
- **Deleted/inactive references** aren't scrubbed from old posts; they render as a tombstone
  ("listing no longer available"). The expansion simply returns `reference: null` and the client
  handles it — same posture as the polymorphic home.

### Per-type v1 constraints (decided 2026-08-01)

- **RICH_TEXT is a plain string in v1** — no formatting, no markup. The type name stays so real
  rich text can arrive later without an enum change. Text blocks may appear **above and below other
  items, multiple times** — text is an ordered content item like any other, not a special field.
  `Post.body` remains the fast path for a **plain, attachment-free post or reply**; the moment the
  composer tree holds any other item, its text segments are stored as RICH_TEXT items in tree
  order.
- **IMAGE: one per post**, resized for mobile at ~**500–600px** via the standard `uploadImage`
  resize options (this is chat media, not artwork).
- **LINK** stores just the `url`; the client owns the preview treatment (see the mobile doc — a
  Nanza-styled link row opening an in-app browser sheet, never leaving the app).

This supersedes the 2026-07-30 decision limiting content items to tag/brand posts: **the composer
(and therefore content items) is available on every post surface** this release. The moderation
concern that motivated the restriction is now handled by the caps below.

### Likes = SavedItem, count derived

Hearting a post is **the same action as saving a listing** — one `SavedItem` row, so liked posts
appear in the account's saved-items surface for free, and unliking is deleting the row. There is no
separate `PostLike` model.

```prisma
model SavedItem {
  // …existing listing/bid/bulkListing targets…
  post   Post?   @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId String?

  @@unique([accountId, postId])
  @@index([postId])
}
```

The post's public **like count is derived** — `count(SavedItem where postId)` — exposed the same way
`postCount` is (a `likeCount` on post responses, batched with a groupBy map for lists). Post
responses also carry `viewerHasLiked` when a viewer account is known, so the heart renders filled
without a second call. Routes: `POST /saved-item` already exists — it gains the `postId` target and
the same duplicate-safe upsert semantics as the listing path.

### Image upload & S3 lifecycle

Post images follow the platform's id-keyed S3 convention (one folder per model, row id as the
filename — `uploadImage`'s 4th `imageId` argument, as `entity/` and `supported-tag-value/` do):

```
post/<contentItemId>.webp
```

Keyed by the **content item's** id, not the post's — one image per post is a v1 *cap*, not a
schema shape, so the keys don't need re-cutting when multi-image arrives. Resize at upload to the
~500–600px mobile spec (standard `uploadImage` resize options; chat media, not artwork).

**Upload flow:** `POST /post` is multipart (`uploadConfig`), same as group create. The handler
creates the post + content item rows first (so the item id exists), then uploads with
`imageId = contentItem.id` and writes the key to `media`. If the upload throws, the create is
rolled back — no half-posts with dead image slots.

**Deletion invariants — S3 must never leak:**

- **Deleting a post** (both paths): the hard delete *and* the tombstone-with-replies path already
  remove content item rows — both must first collect the IMAGE items' `media` keys and
  `deleteImage()` each (best-effort `.catch`, like the group banner deletes).
- **Removing a content item on edit:** `PUT /post` replaces the item set in a transaction; the
  handler diffs old→new and sweeps the `media` keys of removed IMAGE items.
- **Target sweeps:** posts have no FK to their polymorphic home, so deleting a listing/tag doesn't
  cascade to them — the target's delete handler sweeps its posts (as the comment design did) and
  must sweep their image keys in the same pass.
- **Known gap, accepted for v1:** `Post.accountId` cascades at the DB level, so account deletion
  removes posts *without* running any handler — those image keys orphan. Deliberately out of scope
  here: **account deletion is getting its own solution design** (Skylar, 2026-08-01), which owns
  the S3 sweep for post media along with every other cascade-bypassed asset (profile
  avatars/banners, listing images, …).

### Composer contract

What the modal needs from the API, all of which already exists after stage 1:

- **Tag picker:** hierarchical — brand tags expanding to supported tag values. Both levels already
  have endpoints (`GET /brand-tags?brandId=`, `GET /supported-tag-values?brandTagId=`). Selected
  values land in `supportedTagValueIds[]`. **Non-brand posts pick at most ONE** (revised
  2026-08-01 — the picker is single-select on listing/bid); the 5-value cap applies only to the
  future brand composer. On tag pages the icon is disabled and the page's own value is the
  association.
- **Tag chicklets:** once a post has its supported tag value (picked, or ambient on a tag page),
  the composer offers that value's linked secondary tag values
  (`GET /secondary-tag-values?supportedTagValueId=&isPrimary=true`) as tappable chicklets →
  `secondaryTagValueIds[]`, **capped at 25 per post total**, and **server-validated as linked to
  the associated supported value** on non-brand posts. A secondary-tag-value page shows no
  chicklets — it *is* one.
- **Create:** `POST /post` with `body`, `contentItems[]` (now possibly reference items), and
  `secondaryTagValueIds[]` — the shape shipped on 2026-07-31, plus `referenceId` per item.
- **Share: deferred.** A share button on posts (and on the tag pages themselves) needs
  `referenceCode` on `SupportedTagValue`/`SecondaryTagValue` plus OG/universal-router work — a
  separate lift, explicitly out of this release. Now designed:
  [[../nanza-mobile/solution-designs/post-tag-sharing|Post & Tag Sharing]].

### Caps (supersedes the stage-3 placeholder)

- Text/reply posts: existing 50/day.
- Posts carrying IMAGE items: lower cap (set at implementation; follow `usageCaps` conventions).
- Reference items are cheap (no upload) — they count as text for capping.

## What the chat release built in nanza-api (2026-08-01)

The API half is done; mobile is next. **The migration is written but not run** — it is additive
only (no drops), so deploying the code before applying it is not destructive, but the new columns
must exist before any of it works.

**New files**

- `src/services/postReferences.ts` — batched reference expansion. One `findMany` per type per page,
  never N+1. LISTING resolves in two steps (Listings, then leftovers as BulkListings) and stamps
  `kind: 'listing' | 'bulk'`. Unresolved targets come back `reference: null` for the client's
  tombstone. Every select in here is the leanest card shape — this file is where the payload budget
  is actually enforced.
- `src/utils/postLikes.ts` — `likeCount`/`viewerHasLiked` as batched scalars (one groupBy + one
  viewer lookup per page), mirroring `postCount`'s shape.
- `src/utils/sweepPosts.ts` — `sweepPostsForTarget()`: deletes a target's thread **and** its posts'
  S3 image keys. Called from every target delete handler.

**Schema (additive)**

`ContentItemType` += `LISTING`/`BID`/`ENTITY`; `ContentItem.referenceId` (no FK, indexed);
`SavedItem.postId` + `@@unique([accountId, postId])` + index, with `Post.savedItems` back-relation.

> [!warning] Adding three enum values in one migration fails on **PostgreSQL ≤ 11**. Split into
> three migrations if the target is that old; anything ≥12 is fine.

**Deltas from the design, decided during implementation**

- **The `isRichPostableType` gate is gone, not just widened.** The design said content items open up
  to every surface; that made the helper dead code. Removing it exposed a latent bug: tag ids were
  scoped `isRichPostableType(type) ? brandId : null`, so a listing/bid post sending
  `supportedTagValueIds` hit *"tags are not supported on this post target"* — which would have
  broken the chat release's "tags icon enabled on listing/bid" requirement on day one. The fix
  became the **revised association model** (see *Tag validation* above, `validatePostTags` in
  code): the rule branches on the home type directly, so there's no scope-juggling left to get
  wrong. On `PUT`, an omitted `supportedTagValueIds` fills in from the post's current rows before
  validating, so the linkage rule holds on partial edits without replacing sets the caller didn't
  send.
- **`validateContentItems` is now async** — reference items are checked against the database
  (exists + still live), batched by type. Listings/bids must be ACTIVE and lots PUBLISHED *at post
  time*; later deletion is fine and renders as a tombstone.
- **An IMAGE item validates with no `media`.** The key is the content item's own id, which doesn't
  exist until the insert — so the row is created first and `media` written back after upload. The
  router requires the file and the item to arrive together (either alone is an error) and deletes
  the post if the upload throws.
- **`viewerHasLiked` comes from the JWT, never a query param** — via the existing
  `resolveRequesterAccountId`, which returns null for guests rather than throwing. No client
  changes, nothing spoofable.
- **Replies reject attachments loudly.** The composer hides the attach icons in reply mode, so a
  content item arriving on a reply is a client bug; silently dropping it would hide that.
- **`/saved-items` learned posts too.** Liking writes a SavedItem, so without a `post` include and a
  `type=post` filter, liked posts would have surfaced in the saved-items list as rows with every
  target null. `/saved-items/ids` gained `postIds` for the heart state.
- **Image caps live in validation, tag caps in `validateTagIds`** (5 supported / 25 secondary flat,
  counted after de-duplication so re-sending an id doesn't burn budget). The per-post image cap is a
  constant, not a `usageCaps` lifetime counter — those track lifetime creations per account, which
  is a different question from "how many images may one post carry".

**S3 lifecycle — every path that drops an image row now sweeps its key:** post delete (hard *and*
tombstone), `PUT` edit-diff (keys not carried into the new item set), and the six target delete
handlers (listing, bid, bulk, entity, supported tag value, secondary tag value). The two tag routers
previously swept posts with a raw `deleteMany` that leaked images; both now call
`sweepPostsForTarget`. The account-deletion cascade gap is still open and still belongs to the
separate account-deletion design.

**Not done here:** the mobile surface, and `@oakplatforms/types` still needs regenerating +
publishing for the new fields (`reference`, `likeCount`, `viewerHasLiked`, `referenceId`, `postId`).

**Review pass (2026-08-01) — four defects found and fixed in the same session:**

1. **Lot card image fallback bug** (`postReferences.ts`): `children.find(l => l.image ??
   l.entity?.image)?.image` matched a child on its *entity's* image but then read the child's own
   (null) `image` — a lot whose children only had entity images rendered imageless. Now maps each
   child to `image ?? entity.image` first.
2. **S3 leak in the create rollback**: if the upload succeeded but the `media` write-back failed,
   the catch deleted the post rows and stranded the just-uploaded object. The rollback now sweeps
   the uploaded key too.
3. **`PUT` image spoofing / dead slots**: `PUT` has no upload path, yet validation accepts IMAGE
   items with empty `media` (that allowance exists for the multipart create). An edit could mint a
   dead image slot — or worse, claim *another post's* key, which this post's deletion would then
   sweep out from under it. Edits now require every IMAGE item's `media` to be one of the post's
   own existing keys.
4. **Old-build crash via liked posts**: `/saved-items` with no `type` param — exactly what shipped
   mobile builds call — would have returned like rows whose `listing`/`bid`/`bulkListing` are all
   null, the precise shape the backward-compat rule bans. The unfiltered list now excludes post
   rows; `type=post` opts in (new clients only).

**Accepted behavior, noted:** `PUT` re-runs reference liveness, so editing a post whose embedded
listing has since died requires dropping that item — an edit is a new write; the tombstone posture
covers reads. And the slimmed `postInclude` account select (`id` + profile `id`/`username`/`avatar`)
should be checked against what mobile's `PostItem` actually reads during the mobile pass.

## Groups (2026-08-02) — schema reserved, DEFERRED

Renamed from the short-lived "gallery" design the same day (Skylar: a *group* holds any content
items, not just images). **v1 ships single-item posts only** — the group plumbing stays in the
schema so enabling it later is validation + composer work, not a migration:

- `ContentItemType.GROUP` + a self-relation on `ContentItem` (`parentId`/`children`, cascade,
  children keep `postId`). Reads already return top-level items with `children` included, and
  reference expansion already walks children.
- **Validation rejects GROUP** (same reserved posture as VIDEO). Create/PUT are back to the
  single-image multipart flow.
- v1 attachment family, one per post: IMAGE (library or camera), LINK, LISTING (incl. lots),
  BID, ENTITY. The composer hides ALL attach circles once one is attached; removing it brings
  them back. Mobile renders a GROUP carousel branch already (future-safe, nothing emits it).

Types: 0.1.83 (GROUP replaces GALLERY). Deleting a post now hard-deletes its replies too
(cascade via parentId; thread-wide S3 sweep first) — the tombstone path is gone.

## Content-item input hardening (2026-08-02)

Threat model per type, and where each is enforced:

- **IMAGE** — raster only on the post path (JPEG/PNG/WebP; **SVG rejected** — it skips sharp
  and can carry scripts). Sharp's decode→re-encode to webp IS the sanitizer: polyglots, EXIF
  payloads, and appended data don't survive it. Multer caps files at 5MB.
- **LINK** — scheme allowlist (`http(s)` only) at create blocks `javascript:`/`data:`/`file:`.
  Rendered as RN Text + WebView sheet — no HTML-injection surface in RN.
- **EMBED** — hostname ALLOWLIST (exact match on youtube hosts as of 2026-08-02, both ends),
  never a substring test. The player URL is CONSTRUCTED on a fixed origin from an extracted id,
  never attacker-controlled; players run `originWhitelist https://*`. The unparsable-code
  fallback only reaches the in-app browser for real http(s) bodies. **The allowlist is the
  security boundary, not a feature gate** — the player is a WebView, so an arbitrary host in
  the frame executes its own scripts inside the app. Widening it to "any valid URL" is not a
  safe simplification; each new platform needs its own constructed player URL. Rejected embeds
  are stored as LINK items instead, which render as a preview card, never an inline frame.
- **`/link-preview`** — SSRF-guarded: http(s) only; hostname AND its DNS resolution checked
  against private/loopback/link-local ranges (v4 + v6); redirects followed MANUALLY (≤3 hops)
  with every hop re-validated (a 302 into 169.254.169.254 is the classic metadata grab);
  5s timeout, 200KB body cap; `og:image` returned only when http(s).
- **Body/tag text** — rendered through RN Text everywhere; no innerHTML anywhere in the app.
