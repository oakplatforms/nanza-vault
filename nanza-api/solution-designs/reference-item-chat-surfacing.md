---
tags: [nanza-api, solution-design, plan, posts, chat]
---

# Reference-Item Chat Surfacing — embedded items join the chat

> **Status: IMPLEMENTED (uncommitted), 2026-08-14 evening — pending Skylar's QA.** Built same
> session as planned: `referencedPostsFilter` beside the tag filter, wired into the GET /posts
> root read for LISTING/BID/ENTITY/BULK, friend-prefetch extended. The `@@index([referenceId])`
> was already in the schema (Skylar's earlier schema pass). Mobile got the OPTIONAL touch too:
> the three composer publish paths mark embedded items' chats stale (LISTING invalidates both
> the LISTING and BULK keys — the client can't tell a lot embed apart; the wrong one no-ops).
> Work through the QA checklist below, then fold into [[posts|Posts]] and delete this plan.

## Goal

A post that **embeds** a listing, bid, entity, or lot as a reference content item should also
surface in that item's chat thread — the same way a **tagged** post surfaces on its tag value's
page today. The embed *is* an association; it just isn't consulted by the feed reads yet.

## Why it's cheap: the tag union is the template

`GET /posts` for a `SUPPORTED_TAG_VALUE` home already runs a server-side union — posts written
there **plus** posts written elsewhere that tag it — through one where-builder,
`taggedTagValuePostsFilter` (`src/validation/post.ts:333`), which carries the two viewer rules
every tag-driven read must: GROUP-homed posts excluded (members-only stays members-only), BRAND
posts friend-scoped via `brandVisibilityArms`. This plan is that pattern with one predicate
swapped: match on the reference instead of the tag join.

## The work

1. **New where-builder** beside the tag one (same file — it's the rule's single home):

   ```ts
   export const referencedPostsFilter = (
     referenceId: string,
     viewerAccountId: string | null,
     friendAccountIds: string[]
   ): Prisma.PostWhereInput => ({
     contentItems: { some: { referenceId } },
     postableType: { not: 'GROUP' },
     OR: [
       { postableType: { not: 'BRAND' } },
       ...brandVisibilityArms(viewerAccountId, friendAccountIds),
     ],
   })
   ```

   - `referenceId` alone is the predicate — ids are cuids, so no type disambiguation is needed
     for correctness (add `type` to the `some` only if the index wants it).
   - **Gallery children come free**: a GROUP item's children are ContentItem rows on the same
     `postId`, so `contentItems: { some: ... }` matches a card embedded inside a gallery too.
   - **Lots**: a lot embed is stored as a `LISTING`-type item whose `referenceId` is the
     BulkListing id (expansion tries listing → bulk), so the BULK chat's arm is the same filter
     with the lot's id — no special case.

2. **Wire into GET /posts home mode** (`src/routers/post.ts`, the root-feed `where` builder
   around line 305): for `postableType` in `LISTING | BID | ENTITY | BULK` and no `parentId`:

   ```ts
   { parentId: null, OR: [homeWhere, referencedPostsFilter(id, viewerAccountId, friendAccountIds)] }
   ```

   and extend the `friendAccountIds` prefetch condition to those types (today it runs only for
   BRAND roots and SUPPORTED_TAG_VALUE). Thread reads (`parentId` set) stay untouched — replies
   scope by parent.

3. **Index** — the new read filters ContentItem by `referenceId`, which has no index
   (`@@index([postId, index])` only). Add to `schema.prisma`:

   ```prisma
   @@index([referenceId])
   ```

   Skylar runs the migration himself, per the repo rules.

## What does NOT change

- **Mobile: nothing.** `DetailChatSection` already queries `GET /posts?postableType=LISTING|BID|
  ENTITY|BULK&postableId=…` on those detail screens — the union just starts returning more rows.
- The list-payload trim, hydration, likes, counts, pagination: all downstream of the `where`,
  untouched. `status: 'ACTIVE'` on the route already keeps drafts out.
- Tag surfacing, GROUP chat gating, reply pagination.

## QA checklist (the visibility edges)

- [ ] A GROUP-homed post embedding a listing does **not** appear in that listing's public chat.
- [ ] A BRAND-homed post embedding a listing appears in the listing's chat only for the author's
      friends / affiliate-author cases (mirror the brand-feed rules exactly).
- [ ] A gallery child reference surfaces the post; removing the item via edit (PUT replaces the
      tree) drops it on the next read.
- [ ] Tombstoned references (deleted listing) — post still renders in the chat with the tombstone
      card; decide whether that's desired or the arm should require the target live (lean: keep
      tombstones, matches tag behaviour of not re-validating).
- [ ] A post written IN the listing's chat that also embeds that listing isn't duplicated (single
      query, one row — should be structurally impossible, verify anyway).
- [ ] Mobile cache: creating an embedding post should mark the item's chat stale — same
      invalidation idiom the tag pages got on 2026-08-14 (`['Posts', <type>, <referenceId>]` for
      each embedded reference in the three composer publish paths). This is the one OPTIONAL
      mobile touch; without it the chat catches up on its usual stale-refetch.

## Fold-in note

When this lands, fold the union into [[posts|Posts]] § "the tag-page union" (rename the section
to cover both association kinds) and delete this plan.
