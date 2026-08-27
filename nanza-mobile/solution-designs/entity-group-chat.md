---
tags: [nanza-mobile, nanza-api, solution-design, plan, posts, chat, groups, entity]
---

# Entity & Group Chat — Plan

> **Status: PLAN — BOTH SIDES BUILT (uncommitted) 2026-08-10.** Remaining: Skylar runs the
> migration + publishes types 0.1.88; mobile's local `PostableType` mirror then drops. Adds chat to
> the new full-screen entity detail and to group detail — the group side needed a new `GROUP`
> postable target plus scoping in nanza-api (done, see the implementation record below). Also
> reshapes the entity screen's feed into a "Buy & Sell" section. Canonical posts model:
> [[../nanza-api/solution-designs/posts|Posts]]; surfaces: [[posts-chat|Posts Chat]],
> [[groups|Groups]], [[entity-screen|Entity Screen]].

## Implementation record — nanza-api (2026-08-10, uncommitted)

- **Schema**: `GROUP` added to `PostableType` (no comment, per the schema rules). **Migration is
  Skylar's to run** (repo rule): the needed SQL is `ALTER TYPE "PostableType" ADD VALUE 'GROUP';`
  — additive, safe. Prisma client + `packages/types` regenerated (union carries `GROUP`; version
  bumped to 0.1.88, publish pending).
- **Correction to this plan**: content items are allowed on **every** post surface since the
  chat release (the old brand/tag-home restriction is gone), so group AND entity chat carry
  images/links/embeds exactly like every other chat — the "text-only v1" open decisions below
  are moot as written; no gating work was done or needed.
- **Write gates**: `validatePostableExists` got its `GROUP` case (rejects `DELETED` groups,
  returns `group.brandId`); new `validateGroupPostPermission` runs on **create and edit**
  (membership is revocable — same discipline as the affiliate gate, whose comment now covers
  both); backed by `isActiveGroupParticipant` in `validation/group.ts` (moderatorId short-
  circuit + ACTIVE `GroupMember`, non-throwing so read paths shape their own responses).
- **Tags**: `validatePostTags` rejects any tag values on `GROUP` posts — the structural
  no-leak guarantee (the tag-page union can't surface them).
- **Read scoping**: `GET /posts?postableType=GROUP` → 403 for guests/non-members;
  `GET /post/:id` on a group post → **404** for outsiders (indistinguishable from nonexistent);
  `POST /post/:id/reference-code` refuses group posts (share codes resolve publicly, OG crawler
  included — the group's own `G` code stays the invite surface).
- **Saved items**: `type=post` reads exclude `GROUP`-homed likes outside the viewer's
  `getActiveGroupIds` (leave/ban drops them).
- **Delete**: group delete (a soft delete) now runs `sweepPostsForTarget('GROUP', id)` — no
  undelete exists and posts have no FK, so the thread would otherwise linger addressable.
- Verified: `tsc --noEmit` and eslint clean across the five touched files.

## Implementation record — nanza-mobile (2026-08-10, uncommitted)

- **Entity Buy & Sell**: the raw grid became the planned two-state section — collapsed it's the
  group page's `global/CardCarousel` ("Buy & Sell", default-width `ListingThumb`/`BidThumb`
  tiles, "See all" pill); expanded it's the original measured 2-column grid under a
  `ShelfHeading` (no action — See all is one-way). A price-row Sell Now / Buy Now tap while
  collapsed parks the item id in a ref, expands, and the grid item's own `onLayout` completes
  the scroll (the grid must lay out before the item has a y).
- **Entity chat**: `DetailChatSection postableType="ENTITY"` mounted below the section, behind
  the `feedReady` gate — lifts posts-chat's "no entity chat UX" deferral with zero API change.
- **Group chat**: the visual-only placeholder in `GroupDetailScreen` is now a real
  `DetailChatSection postableType="GROUP"` with the placeholder's exact container styles,
  rendered only for `isActiveMember` (the API 403s outsiders; hiding beats a pill that can only
  fail).
- **Types**: `src/types/index.ts` widens `PostableType` with a local `| 'GROUP'` arm until
  0.1.88 lands — drop it on install.
- Verified: `tsc --noEmit` and eslint clean on the three touched files.

## What we know going in (mapped 2026-08-10)

- **`ENTITY` is already a fully-wired post home in the API** — enum member, live
  `validatePostableExists` case (returns the entity's `brandId`), delete sweep in the entity
  router, and it's in the published `PostableType` union. Entity chat is client-only work; the
  posts-chat design's "entity surfaces get no chat UX yet" deferral lifts here.
- **`GROUP` is not a postable type** anywhere — schema, validation, or types package. The group
  detail screen ships a visual-only chat placeholder whose comment names exactly this
  precondition.
- **There is no post-level visibility model.** `GET /posts`, `GET /post/:id`, `getPostCount`,
  and the saved-items read all filter nothing; the only scoped read is the share-code resolver.
  The tag-page union (`posted here OR tagged here`) is the one query that surfaces posts away
  from their home — the main leak path to design against.
- Mobile's posts layer is a pure pass-through on `postableType` — `usePosts`/`PostThread`/
  `JoinChatFooter`/`DetailChatSection` never switch on it, so a new member is a types bump plus
  call sites.
- Group membership is derived client-side (`isActiveMember = moderator || member.status ===
  'ACTIVE'`); server-side the primitives exist in `src/validation/group.ts`
  (`validateGroupMembership`, `getActiveGroupIds`, the `build*VisibilityFilter` pattern).

## Part A — nanza-api: the `GROUP` postable target

### Schema

Add `GROUP` to `PostableType` (+ migration), regenerate + publish `@oakplatforms/types`.
**No `groupId` column** — the home pointer (`postableType='GROUP'`, `postableId=groupId`) IS the
group scoping, same as every other home. Structural, not advisory: group posts self-segregate on
every home-scoped read.

### Write path

- `validatePostableExists` gets `case 'GROUP'`: load the group, reject `DELETED`, return
  `{ brandId: group.brandId }`. (TS exhaustiveness makes this a compile error until added —
  the enum can't half-land.)
- **New `validateGroupPostPermission(type, id, accountId)`** — no-op unless `GROUP`; requires
  the author to be the group's moderator or an `ACTIVE` member (the server-side twin of the
  client's `isActiveMember` — note the moderator may hold no `GroupMember` row, so check
  `moderatorId` first). Called from **create AND edit**, the same mutable-permission discipline
  the brand gate learned the hard way: membership is revocable (bans), so a one-time check at
  create is not a durable boundary. Delete stays ownership-only, mirroring BRAND.
  Replies need no extra code — a reply carries its parent's home, so the same gate fires.
- **Tags: rejected on `GROUP` posts** (`validatePostTags` throws). This is the structural
  answer to "group content must not be seen outside the group": the tag-page union's second arm
  (`tagged here`) is the only query that lifts a post away from its home, and a post that can't
  carry tag associations can't ride it. No exclusion filters to maintain, nothing to forget in
  a future read path. If groups ever need taxonomy tagging, the price is an
  `AND NOT postableType='GROUP'` (or membership filter) on the union's tag arm — deferred until
  wanted.
- Rate limit: the existing 50/day account cap applies unchanged.

### Read path scoping

- **`GET /posts` with `postableType=GROUP`**: requires a signed-in viewer who passes the same
  moderator-or-ACTIVE-member check; 403 otherwise. Guests never read group threads.
- **`GET /post/:id`**: when the post's home is `GROUP`, apply the same check (respond 404, not
  403, so non-members can't probe which ids exist).
- **Share codes: refused for group posts** — `POST /post/:id/reference-code` rejects
  `GROUP`-homed posts, so the public resolver and OG crawler paths can't leak content.
  (Sharing the *group* keeps its own `G` code — that's the invite surface.)
- **Saved items**: liking stays enabled in-group, but the saved-items read filters
  `GROUP`-homed liked posts to the viewer's `getActiveGroupIds` — leaving or being banned
  removes them from the list.
- **Counts**: `getPostCount` is home-scoped, so group counts only surface where the group
  itself is being rendered — no change needed.
- **Delete sweep**: `sweepPostsForTarget(prisma, 'GROUP', id)` added to the group delete
  handler (missing today; posts have no FK to their home).

### Entity (API): no changes

Already a valid home. Entity threads follow the listing/bid rules as they stand: text-only
(content items stay gated to brand/tag homes), single optional tag association, plain
home-scoped reads.

## Part B — nanza-mobile: entity screen

### "Buy & Sell" section (replaces the raw feed grid)

- **Collapsed (default)**: the group page's shelf — `global/CardCarousel` titled **"Buy &
  Sell"**, same `ListingThumb`/`BidThumb` tiles, horizontal — with a **"See all"** action pill
  (the home shelves' label; the group page uses that slot for "Add new", but here create lives
  in the bid/sell pills above, so the pill only reveals).
- **Tap "See all" → the section expands in place** into the existing 2-column grid (the current
  hand-rolled `thumbWidth` grid, unchanged), heading stays, pill disappears. No navigation, no
  new screen.
- The price row's Sell Now / Buy Now **scroll-to-item taps first expand the grid**, then scroll
  — a horizontal rail can't host a vertical scroll target.
- Empty feed: section renders nothing (current behavior).

### Chat section

Below Buy & Sell, the standard mount, exactly like BidScreen's:

```tsx
<DetailChatSection postableType="ENTITY" postableId={entity.id} brandId={entity.brandId} />
```

deferred behind the existing `feedReady` gate. Guests get the auth window via the pill's own
gate. Text-only composer comes free (the API refuses content items on entity posts; the attach
circles render but the API's rule is the boundary — same as listing/bid threads today).

## Part C — nanza-mobile: group chat

Replace the placeholder block in `GroupDetailScreen` with the real mount, preserving the
current spacing:

```tsx
<DetailChatSection
  postableType="GROUP"
  postableId={groupId}
  brandId={group.brandId}
  containerStyle={[detailStyles.chatSection, cardStyles.groupDetailChatSection]}
/>
```

- **Rendered only for `isActiveMember`** — non-members of a public group see the rail but no
  chat at all (the server 403s them anyway; hiding beats a pill that can only fail). Private
  groups already gate the whole body.
- Needs the types bump (`GROUP` in the `PostableType` union) — mobile mirrors locally until
  the package publishes, the established pattern.

## Rollout

1. nanza-api: enum + migration, validators, read scoping, sweep; regenerate + publish types.
2. nanza-mobile: entity Buy & Sell + entity chat (unblocked immediately — can ship first).
3. nanza-mobile: group chat mount once types land.
4. Fold outcomes into [[../nanza-api/solution-designs/posts|posts]], [[groups]],
   [[entity-screen]], [[posts-chat]] and retire this plan.

## Open decisions (Skylar)

1. **Content items in group chat** — plan says text-only v1 (the marketplace-thread rule);
   groups are moderated spaces, so allowing images/links/embeds like the tag chats is
   defensible. Which?
2. **Non-members and a public group's chat** — hidden entirely (planned) vs read-only preview.
3. **Tags rejected on group posts** (planned) — confirm; the alternative (allow + exclusion
   filter on the union) trades simplicity for future taxonomy reach.
4. **Entity chat text-only** (planned, matches listing/bid) — confirm.
