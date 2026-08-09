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

**`components/tags/HomeTagCarousel`** is the home shelf: a one-liner that passes the tag name
`'class'` and the selected brand. It sits in the HomeScreen list header between
`GroupCircleCarousel` and the Top Picks grid, and is hidden while switching brands so it never
flashes the previous brand's tags. It renders nothing when the selected brand has no tag by that
name — deliberately *not* falling back to some other tag, since a shelf silently showing a
different taxonomy than intended is worse than no shelf.

> **This is a placeholder for a dynamic homepage config.** When that lands, the tag name and the
> shelf's position come from config rather than the `HOME_TAG_NAME` constant.

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
  no include needed for the rail's images.
- `SubTagValueScreen` needs the parent id to mount the thread; it comes back on the plain
  `GET /sub-tag-value/:id` (no `select`, so all scalars return) and the thread is skipped if absent.
