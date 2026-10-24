---
tags: [nanza-api, solution-design, plan, posts, tagging]
---

# Posts — Multi-Content Carousel & Open Tagging — Plan

> **Status: PLAN, 2026-08-11 (Skylar, voice session).** Full-stack with
> [[../nanza-mobile/solution-designs/post-carousel-and-tagging|the mobile plan]]. Extends
> [[posts|Posts]] (canonical model) and revises two of its decided rules: the one-attachment v1
> cap and the one-parent-tag association rule. Also revises
> [[../nanza-mobile/solution-designs/entity-group-chat|Entity & Group Chat]]'s "tags rejected on
> GROUP posts" — group posts become taggable, with the tag-page union taking the exclusion filter
> that plan already priced.

## Implementation record — nanza-api (2026-08-11, uncommitted)

Built while Skylar reviewed the plan. Deltas and specifics beyond the design below:

- **Schema**: `ContentItem.isPrimary Boolean @default(false)` (no comment, per repo rules).
  **Migration is Skylar's to run**: `ALTER TABLE "ContentItem" ADD COLUMN "isPrimary" BOOLEAN
  NOT NULL DEFAULT false;` — additive, safe. Prisma client + `packages/types` regenerated,
  version **0.1.89** (publish pending; 0.1.88 from group chat was still unpublished, so this
  bump carries both).
- **Caps** (`validation/post.ts`): `MAX_ATTACHMENT_ITEMS = 5` / `REPLY_MAX_ATTACHMENT_ITEMS = 1`
  (attachment = any non-RICH_TEXT item — image, link, embed, and references all compete for the
  same carousel slots); `MAX_IMAGE_ITEMS` now equals the attachment cap;
  `MAX_POST_TAG_VALUES` 30 → **10 flat**. `validateContentItems` takes `{ maxAttachments }`.
- **Tags**: `validatePostTags` unified — every home (GROUP included) takes up to 10 values,
  brand-scoped wherever the home has a brand (`BULK` keeps its unscoped behavior — lots span
  brands, preserved from before). Children must link to *any* of the post's parents (plus the
  page's own value on a supported-tag-value home); **BRAND keeps its no-linkage permissiveness**
  (2026-08-01 decision, deliberately not tightened). The "only the page's own value" rejection
  on tag-page homes is gone — the ambient value can now be joined by others.
- **Group no-leak moved from structural to filtered**: new `taggedTagValuePostsFilter(id)`
  builder in `validation/post.ts` — the tag-union's tagged-here arm with
  `postableType: { not: 'GROUP' }`. `GET /posts` uses it; it is the single home for the rule
  and every future tag-driven read must go through it. (Audited: the union is today's only
  such read; `postCount` and the tag routers are home-scoped.)
- **Create** (`POST /post`): multipart is now `uploadConfig.array('image', 5)` — same `image`
  field name, so single-file old-build uploads still parse; files pair with IMAGE items in
  index order and the count must match exactly. The reply branch merged into the main path:
  replies validate with `maxAttachments: 1`, reject `supportedTagValueIds` loudly, and **no
  longer copy the parent's tag rows**. Primary stamping: lowest-index attachment gets
  `isPrimary` (root posts only). Upload rollback sweeps *all* already-uploaded keys.
- **Edit** (`PUT /post/:id`): replies may now send `contentItems` (1 attachment); tags still
  rejected. Root edits re-stamp the primary: the new set's item matching the old primary's
  identity (type + media/url/referenceId) keeps the flag, else the lowest-index attachment
  leads again — so an ordinary edit never silently undoes a promotion.
- **Swap**: `PUT /post/:id/primary-content-item` `{ accountId, contentItemId }` — owner-only,
  root-only, re-runs the brand/group mutable-permission gates, swaps the two items' `index`
  values and moves the flag in one transaction. Legacy posts (no flag) treat the first
  attachment as current primary; promoting it just backfills the flag. Deliberately not a full
  PUT so a swap can't fail on a since-died embedded reference.
- Verified: `tsc --noEmit` and per-file eslint clean; repo-wide `npm run lint` failures are
  pre-existing in untouched files.

**Not done here:** publish 0.1.89, run the migration (both Skylar), and the whole mobile half.

## Second revision — per-type budgets (2026-08-12 PM, voice; implemented same session)

Supersedes the flat five-leaf attachment pool: **images are the expensive leaves** (storage,
upload) — everything else is a data association over existing records, so each category
budgets separately, counted in leaves:

- `MAX_IMAGE_ITEMS = 2` (one library pick + one camera snap — the split is client-side; the
  server just sees two IMAGE items), multer's file cap follows it.
- `MAX_TEXT_ITEMS = 5` (top-level RICH_TEXT rows), `MAX_LINK_ITEMS = 5` (LINK+EMBED — one
  paste field), `MAX_LISTING_ITEMS = 5` (LISTING+BID — one shop picker),
  `MAX_ENTITY_ITEMS = 5`, `MAX_COLLECTION_ITEMS = 1` (briefly opened to five mid-session,
  re-decided back to ONE the same afternoon). Cards are **either/or: a collection and
  individual cards cannot share a post** — validated server-side alongside the caps.
- `MAX_CONTENT_ITEMS` 20 → 35 (must clear the 27-leaf worst case + gallery wrappers).
- Replies keep the flat ONE-attachment total (`maxAttachments` is now reply-only — roots pass
  none and get the per-type budgets).

## Implementation record — galleries & collections (nanza-api, 2026-08-12, uncommitted)

- **Schema**: `COLLECTION` added to `ContentItemType`. **Migration is Skylar's to run**:
  `ALTER TYPE "ContentItemType" ADD VALUE 'COLLECTION';` — additive, safe. Types bumped to
  **0.1.90** (publish pending): `PostCollectionReference` (+`accountId` — the collection
  screens are keyed by owner + list) and the widened `PostReference` union in `shared.ts`.
- **Validation**: `validateContentItems` returns `ValidatedContentItem[]` — GROUP is a
  structural NODE (`childItems`, reference leaves only, no nesting), everything else a leaf.
  Caps count LEAVES (`draft` mirrors this client-side); flat `MAX_CONTENT_ITEMS` counts every
  row. `MAX_COLLECTION_ITEMS = 1`. COLLECTION leaves batch-validate against `List` **with
  ownership** (`accountId` threaded through) — attaching is sharing, so it must be yours.
- **Create**: top-level rows ride the post create; gallery children `createMany` right after
  against the created group ids (a child's required `post` relation can't fill through a
  doubly-nested create), with delete-the-post rollback. **PUT rejects galleries** ("Post edits
  cannot modify galleries yet") — create-only, and silently accepting one would drop its
  children.
- **Expansion** (`postReferences.ts`): `COLLECTION` joins the reference types — lean card
  (name, cover `thumbnail??logo??banner`, `entityCount`, up to 4 `previewImages`, owner
  `accountId`); **private collections expand to `null`** (tombstone), so flipping a collection
  private un-shares it from every old post. Children were already walked.
- Verified: `tsc --noEmit` + eslint clean.

## Revision — galleries & collections (2026-08-12, voice)

Supersedes the post-level carousel direction below where they conflict:

- **No post-level pager.** Posts render text + the primary item; multi-item "galleries" are a
  CONTENT ITEM, not a presentation of the post's flat attachment list.
- **`GROUP` content items go live** (were schema-reserved/rejected): the composer's
  multi-select pickers create ONE group item whose children are reference leaves
  (LISTING/BID from the shop picker, ENTITY from the library picker). Renders as the small
  tile rail. Create-only this release — no post-edit batch; "batch" is simply the one create
  request carrying the group.
- **New `ContentItemType.COLLECTION`** — `referenceId` → `List.id` (the collections model).
  Attaching a whole collection to a post. Create-gate: the collection must belong to the
  posting account. Read: expansion returns a lean collection card (name, image, count);
  private collections expand to `null` (tombstone) so a later privacy flip un-shares it.
  Migration (Skylar): `ALTER TYPE "ContentItemType" ADD VALUE 'COLLECTION';`
- **Caps recounted in LEAVES**: the 5-attachment cap counts group CHILDREN plus loose
  attachments — a group wrapper is free, its contents are not. Pickers cap their selection at
  the post's remaining slots (4 when a primary already sits on the post).
- New composer text circle (multiple RICH_TEXT blocks) — no api change; RICH_TEXT items
  already interleave.

## The ask

1. **Root posts carry up to 5 content-item attachments** (was 1). The **first attached is the
   primary** — it's what renders with the post on feed/thread cards; the rest show as a carousel
   (slots 2–5) on the post detail. The author can promote a carousel item to primary; the two
   items **swap positions**.
2. **Replies flip**: they gain **at most one attachment** (was zero — "the way the root works
   now") and **lose tag associations** (today they copy the root's tag rows).
3. **Open tagging**: any root post can carry up to **10 supported tag values (parents and
   children combined — one flat cap)** — picked from a new full-screen tag picker on mobile, no
   longer limited to one, on every home including listing/bid/entity **and GROUP**. The picker
   offers **parents only for now** (a UI deferral, not a server rule); Set association is
   explicitly out of scope (Set isn't a tag today; think it through separately).
4. **Group posts tag silently**: a post created in a group can carry the same 10 tags — they
   exist for reference/discovery *from* the post — but the post must **never surface on a tag's
   page**, because group content stays members-only. The 5-and-10 caps apply identically in
   groups.

## Data model

`ContentItem` already has `index Int` (ordering within the post — `@@index([postId, index])`).
Add:

```prisma
model ContentItem {
  // …existing fields…
  isPrimary Boolean @default(false)
}
```

- **Invariant: at most one `isPrimary` item per post, and only on root posts.** Enforced in the
  write paths (create sets it on the first attachment; the swap endpoint moves it
  transactionally). `index` keeps carousel order; `isPrimary` marks the feed representative.
- **Decision to confirm:** `isPrimary` is technically derivable ("primary = the attachment with
  the lowest index") since the swap exchanges *positions* anyway — after a swap, primary is
  always in the original slot. If we hold that invariant strictly, the column is redundant and
  could be skipped. Kept as an explicit column per Skylar's ask, and it keeps reads
  self-describing (`WHERE isPrimary` beats "min index among non-text items") — but if we'd
  rather not carry two sources of truth, index-only is the lean alternative.
- **Migration is Skylar's to run** (repo rule): additive column, safe. No schema comments (repo
  rule). Regenerate + publish `@oakplatforms/types` after.

## Validation changes (`src/validation/post.ts`)

**Caps** (today: `MAX_IMAGE_ITEMS = 1`, `MAX_CONTENT_ITEMS = 20` structural):

- New `MAX_ATTACHMENT_ITEMS = 5` — attachments being non-`RICH_TEXT` items (IMAGE, LINK, EMBED,
  LISTING, BID, ENTITY). `MAX_IMAGE_ITEMS` rises to 5 (images just count as attachments; the
  one-image rule was the v1 cap, not a storage constraint — S3 keys are already per-content-item
  so multi-image needs no key re-cut, as the Posts design anticipated).
- `MAX_CONTENT_ITEMS = 20` stays as the structural ceiling (text blocks interleave).
- **Replies: at most ONE attachment, no tags.** The create path's loud rejection
  ("Replies cannot carry content items") relaxes to a count check; the multipart image path must
  work for replies too. The reply-create's parent-tag copy block is **removed** — replies carry
  no tag rows. *Consequence, deliberate:* replies stop riding the tag-page union on their own;
  threads already render under their root, which keeps its tags.
  - **Confirmed (Skylar, 2026-08-11): optional, up to one** — a reply may carry at most one
    content item, never required, never tagged.

**Tags** (`validatePostTags` — today one parent max on non-brand homes, children linked to it,
GROUP rejected, flat `MAX_POST_TAG_VALUES = 30`):

- **Every home: up to 10 supported tag values FLAT — parents and children combined**
  (`MAX_POST_TAG_VALUES` drops 30 → 10; no separate parent cap — corrected by Skylar
  2026-08-11: "there shouldn't be a hard rule on 10 parents; it can be 10 supported tags,
  parent and children — the UI just doesn't allow child tagging yet"). Brand-scoped as today.
  The one-parent rule and the "only brand posts fan out" special case collapse; BRAND keeps its
  permissive behavior, now under the same 10 cap.
- **Children:** the new mobile picker offers parents only (UI deferral, not a server rule).
  Server-side children stay accepted under the flat cap; with multiple parents possible, the
  linkage rule generalizes to *linked to any of the post's parent values* (incl. the page's own
  value on a supported-tag-value home).
- **`SUPPORTED_TAG_VALUE` home:** the page's own value stays auto-associated (ambient); the
  picker can now *add* up to 9 more alongside it. The "anything else arriving is a client bug"
  rejection goes away.
- **GROUP home: the rejection flips to acceptance** — same 10-parent rule as everywhere.

## Group posts must not leak through tags

The structural no-leak guarantee ("a post that can't be tagged can't ride the union") is traded
for a filter, exactly as the group-chat plan priced:

- **Tag-page union** (`GET /posts` for tag homes, both arms merged): the *tagged-here* arm gains
  `AND postableType <> 'GROUP'`. The *posted-here* arm can't match group posts by construction.
- **Audit every read that joins `PostSupportedTagValue` outward** — today the union is the only
  such path, but this filter is now an invariant every future tag-driven read must carry. Add a
  helper (e.g. `publicTagArmFilter`) rather than inline conditions, so the rule has one home.
- Already safe, no change: `GET /post/:id` 404s outsiders on group posts; share codes refuse
  group posts; saved-items reads scope group likes to active memberships; `getPostCount` is
  home-scoped.

## The swap

Promoting a carousel item swaps its **position** with the primary: if the primary sits in slot 1
and the user promotes the item in slot 4, the old primary lands in slot 4 and the promoted item
takes slot 1 (and `isPrimary`). Feed cards re-render off the new primary; the detail carousel
reflects the exchanged slots.

- **Endpoint:** small and dedicated — `PUT /post/:id/primary-content-item` (repo rule: PUT,
  never PATCH) with
  `{ contentItemId }` — rather than a full `PUT` round-trip of the item tree (PUT replaces sets
  and re-validates references; a swap shouldn't fail because an embedded listing has since
  died — the same posture as the tombstone rule).
- Owner-only (standard ownership check), root posts only, item must belong to the post and be an
  attachment (not RICH_TEXT). Transaction: clear old primary + swap the two `index` values + set
  new primary.

## Read path

- Post list/detail payloads already return `contentItems` ordered by `index`; add `isPrimary` to
  the selects and to `PostContentItemDto`. No payload-budget change — same items, one boolean.
- Feed/thread cards render the primary item only (client-side pick — no new endpoint shape).

## Rollout

1. Schema: `isPrimary` (+ Skylar runs migration), types regenerate/publish.
2. Validation: caps (5 attachments, reply's one), tag rule (10 parents everywhere, GROUP
   acceptance), union filter + helper.
3. Swap endpoint.
4. Mobile ([[../nanza-mobile/solution-designs/post-carousel-and-tagging|plan]]).
5. Fold durable outcomes into [[posts|Posts]] (caps table, tag model, swap) and the group-tag
   filter into the groups section; retire this plan.

## Open decisions (Skylar)

1. `isPrimary` column vs. index-only ("primary = lowest-index attachment") — column planned.
2. ~~Replies: attachment optional vs. required~~ — **resolved: optional, up to one** (2026-08-11).
3. ~~Separate parent cap~~ — **resolved: one flat cap of 10, parents + children** (2026-08-11);
   children linked to any of the post's parents.
4. Set association — explicitly deferred; Set is not a tag today and needs its own thinking.
5. Swap prompt audience — owner-only assumed.
