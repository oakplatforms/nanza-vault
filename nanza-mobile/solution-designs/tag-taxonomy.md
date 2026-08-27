---
tags: [nanza-mobile, solution-design, tags, taxonomy]
---

# Tag Taxonomy (mobile surface) — Rail + Two Detail Pages

> **Status:** Refactored to v2 + Posts on 2026-07-31. Canonical data model in
> [[../nanza-api/solution-designs/tag-taxonomy-v2|nanza-api tag-taxonomy-v2]]; conversation model
> in [[../nanza-api/solution-designs/posts|Posts]].

## What the client refactor changed (2026-07-31)

- `SubTagValueScreen` → **`SecondaryTagValueScreen`**, and it now targets **itself**
  (`postableType="SECONDARY_TAG_VALUE"`) instead of passing its parent's id with a filter. A
  secondary tag value can belong to several classes, so its conversation can't be a slice of any
  one class's thread. The union (posted-here + tagged-here) is resolved server-side, so the client
  just names the target.
- `components/comments/` → **`components/posts/`**: `CommentThread`→`PostThread`,
  `CommentItem`→`PostItem`, `CommentInput`→`PostInput`, `useComments`→`usePosts`, and
  `styles/components/comments.ts`→`posts.ts`. The `subTagValueId` filter prop is gone from
  `PostThread` — there's no filter param any more.
- Services: `Comment.ts`→`Post.ts` (`/posts`, `/post/:id`), `SubTagValue.ts`→
  `SecondaryTagValue.ts` (`/secondary-tag-value(s)`).
- Types: `CommentDto`→`PostDto`, `CommentableType`→`PostableType`,
  `SubTagValueDto`→`SecondaryTagValueDto`, plus `ContentItemDto`/`ContentItemType` for stage 3.
- **`commentCount` is unchanged on the wire** — the API emits it alongside `postCount`, so feed
  cards and the optimistic count-bump caches in `usePosts` still work untouched. Switching those
  reads to `postCount` is a later cleanup, not part of this refactor.

## Slim facets read + paged sections (2026-08-18)

The search filter screen (`SearchScreen/Filter` → `OpenTagSection`) and the post composer's
tag screen (`TagPickerLayer` → `TagPickerSection`) both listed a brand's tags with their
values via `GET /brand-tags?…&usePagination=false&include=supportedTagValues…`. That hands
back whole value rows — description (~half the bytes), timestamps, admin ids — and on prod
ran to **~450KB for one brand (21 tags, 565 values)**, for screens that read display names
and ids. Both now ride a purpose-built slim read and page values in on demand:

- **`GET /brand-tag-facets?brandId=`** (`useFetchBrandTags`, `brandTagService.facets`) —
  a Prisma `select`, not an include: brand tag `id`/`index`, `tag {id,name,displayName}`,
  and per value `id`/`name`/`displayName`/`isPrimary`/`index` only; the first
  `valuesLimit` values per tag (**6** — `BRAND_TAG_VALUES_LIMIT`, the sections' collapse
  count) ordered index-then-displayName, plus `_count.supportedTagValues` for the rest.
  Parents only (`isChild=false`), in rows and count. Flags: `valuesIsPrimary=true`
  (composer), `includeChildren=true` (nests `children.child` id/name/displayName),
  `valueIds=` (pins). The shape is a strict subset of `BrandTagDto`, so no new DTO. Not
  paginated at the brand-tag level — a brand's tag set is small. ~10–15KB on prod.
- **"Show more" pages the rest in** — `useFetchSupportedTagValues` now starts at **page 0**
  of `GET /supported-tag-values?brandTagId=&limit=20` (same ordering as the facets read) and
  dedupes against the six it already holds, so the overlap never double-renders; it reports
  "more" from the count before it has fetched anything, which is what lets a section offer
  the pill up front. Enabled on expand, which brings the **first page only**; every page
  after pulls in **as the open section scrolls toward its end** — `useScrollTrigger`
  (`src/hooks/useScrollTrigger.tsx`): the host (the composer layer's ScrollView; the filter
  page via PageLayout's new `onScroll` passthrough) owns a hub, each expanded section with
  pages left subscribes and `measureInWindow`s itself on scroll, loading when its bottom is
  within half a screen of the fold (and once on settle with no margin, so a page that lands
  with room on screen fills it). Before this, expand fired every page back to back — a
  277-value tag meant a dozen requests at once (Skylar, 2026-08-18). The composer passes
  `isPrimary=true&include=children.child` (the list route gained `include` for this) so
  paged-in parents bring their children.
- **Pins, for the composer** — re-opening the tag screen floats/expands the sections
  holding the post's committed picks, which only works if those values are in the payload.
  The layer sends its committed ids as `valueIds`; the api merges them into their brand
  tag's list past the cut (a pinned **child** pins its parents instead — children show
  under a parent's section, never in the tag grid). Pins still have to pass the value
  filters, so they can't smuggle in a value the screen wouldn't list. The query key carries
  the pin set, so distinct committed sets are distinct cache entries.
- **`TagPickerSection` owns its parent→children sections now.** Before, `TagPickerLayer`
  flattened tag sections and parent sections into one list; with paging, a parent loaded by
  "Show more" has to bring its children section with it, so the tag section draws, right
  after itself, one nested `TagPickerSection` per loaded parent with children (child
  sections don't page — a parent's children arrive complete). **The parent chip is the
  door** (Skylar, 2026-08-18): a parent's children section shows only while the parent is
  selected — or one of its children already is, so a committed child is never
  selected-but-hidden — and folds away when it's deselected; an unpicked parent's children
  never clutter the screen. The float-to-top logic treats a committed child pick as a pick
  on its parent's tag section.

Caveat: numeric tags (cost, power, life) sort alphabetically on the server, so the first six
of cost are 0, 1, 10, 2, 3, 4 (the client re-sorts them numerically for display). An
admin-set `index` on the values fixes the cut where it matters.

## Overview

The taxonomy becomes browsable in the app: a circle-thumb rail on the homepage leads into a
**supported tag value** page (e.g. the class *Warrior*), which leads into a **sub tag value** page
(e.g. the hero *Boltyn*). Both pages reuse the group/listing/bid detail treatment — banner with a
bottom gradient, title over it, content below.

## `TagCarousel`

`components/tags/TagCarousel` is one rail with two modes, and each mode can be addressed by id or
by name:

| props | rail contents | tap target |
|---|---|---|
| `subTagId` (a **SupportedTagValue** id) | that value's sub tag values | `SubTagValueScreen` |
| `supportedTagId` (a **BrandTag** id) | that brand tag's supported tag values | `SupportedTagValueScreen` |
| `supportedTagName` + `brandId` | same, resolved by tag name | `SupportedTagValueScreen` |

Precedence is `subTagId` → `supportedTagId` → `supportedTagName`. **Name addressing is the point
for callers that don't hold an id**: BrandTag ids differ per environment and per brand, but the tag
*name* is a stable slug (admin slugifies `displayName` → `name`, so "Class" is stored `class`).

Crucially the name is resolved **server-side, in the same request** —
`/supported-tag-values?tagName=class&brandId=…` filters through the `brandTag → tag` relation
(case-insensitive). The rail is therefore always exactly **one** call; it never pulls the brand's
whole tag set to find one id.

The rail renders `CircleThumb` over each row's `thumbnail` and returns `null` while loading or
empty, mirroring `GroupCircleCarousel` (also the source of the `CardCarousel` + `inset="sm"` +
`itemGap="gutter"` shelf conventions). The title defaults to the resolved tag's display name.

**`components/tags/HomeTagCarousel`** is the home shelf ("Trending"): a one-liner that passes
the selected brand alone — no tag name — so the rail spans every brand tag, mixing a class
beside a print run. Which values appear is an admin decision twice over (2026-08-17): the
server keeps only values that are **`isPrimary` AND carry an `index`** for this brand-wide
mode — isPrimary curates within one tag, but across a whole brand it can run to thousands,
so the index is the shelf's actual shortlist. Scoped rails (by brandTagId, tagName, or
parentId) stay isPrimary-only. It sits in the HomeScreen list header between
`GroupCircleCarousel` and the Top Picks grid, and is hidden while switching brands so it never
flashes the previous brand's tags. It renders nothing when the selected brand has no primary,
indexed values.

> **This is a placeholder for a dynamic homepage config.** When that lands, the shelf's
> position comes from config.

## Detail pages

**`SupportedTagValueScreen`** — banner + name, then in order: a `TagCarousel` of its sub tag values,
the description, and the conversation.

**`SubTagValueScreen`** — the bottom layer, so no rail: banner + name, description, conversation.

Both are registered on the root stack (`AppNavigator`) with the horizontal-slide interpolator used
by the other detail pages, and typed in `RootStackParamList`.

## The conversation is one thread, filtered

This is the subtle part. A comment's target is **always** the supported tag value
(`commentableType: 'SUPPORTED_TAG_VALUE'`) — a sub tag value is never a comment target. Sub tag
values are *tags on comments*, so the sub tag value page shows the **parent's conversation filtered
to its own tag**:

- `useComments(type, id, enabled, subTagValueId?)` appends `subTagValueId` to the request and puts
  it in the query key, so filtered and unfiltered views cache separately. The existing mutation
  invalidations use a 3-element key prefix, which still matches the 4-element filtered keys —
  React Query matches by prefix, so no invalidation changes were needed.
- `CommentThread` takes an optional `subTagValueId`. `SubTagValueScreen` passes its own id along
  with the **parent's** `commentableId`.
- Posting from a filtered view sends `subTagValueIds: [subTagValueId]` so the new comment stays
  visible where it was written. Only top-level posts send it — replies inherit the root comment's
  tags server-side.

## Gotchas

- **The rail's `supportedTagId` is a `BrandTag` id, not a `Tag` id** — supported tag values hang off
  `BrandTag` (per-brand config), so that is what `/supported-tag-values?brandTagId=` wants. This is
  the main reason name addressing exists: `supportedTagName` is matched against `Tag.name` but
  resolves to the `BrandTag` row joining that tag to the selected brand.
- **Name resolution belongs on the server, not the client.** The first cut matched the tag name
  against `GET /brand-tags?brandId=&include=tag` in the component, which meant the homepage pulled
  every brand tag *and its supported values* just to find one id. The `tagName` + `brandId` filter
  on `/supported-tag-values` replaced that with a single lean request.
- `/supported-tag-values` has no `select`, so `thumbnail` comes back with the rest of the scalars —
  no include needed for the rail's images. It does take `include=` now (2026-08-18, e.g.
  `children.child`) for the tag screen's paged-in parents.
- `SubTagValueScreen` needs the parent id to mount the thread; it comes back on the plain
  `GET /sub-tag-value/:id` (no `select`, so all scalars return) and the thread is skipped if absent.
