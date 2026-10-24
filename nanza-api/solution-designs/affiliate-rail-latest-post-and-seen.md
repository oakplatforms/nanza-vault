---
tags: [nanza-api, nanza-mobile, plan, profiles, posts, home, affiliates]
---

# Affiliate Rail — Latest Post, Seen State, Recency Order

**Status:** **built 2026-10-09 in api + mobile, uncommitted, pending QA** (migration
`20261009120000_post_seen` to run; types `0.1.107`; mobile `useCanSharePost` and this rail
compile against the current package). Same session also opened BRAND posts on profiles and
direct reads to everyone — recorded in [[open-posting-and-friend-visibility]] (third pass).
Plan from a voice session the same day. Spans nanza-api (one model, one write, one read endpoint, read decoration) and
nanza-mobile (rail swap, tap-to-post, seen stamp, ring state). Builds on
[[verified-profiles-and-affiliate-rail]] (rail restored 2026-10-08). When it lands, fold into
[[posts]] (a "Seen = `PostSeen`" section beside Likes), the mobile
[[../../nanza-mobile/solution-designs/whats-new|What's New]] and
[[../../nanza-mobile/solution-designs/app-load-performance|app-load-performance]] designs, and
retire this file.

## The ask (Skylar, 2026-10-09)

The affiliate circles at the top of Home become a "what's new from affiliates" rail, Stories-style:

1. **Tap opens the affiliate's latest post**, not their profile.
2. **The rail is ordered by recency.** One circle per affiliate, the affiliate with the newest
   post first. A new post moves that affiliate to the front.
3. **Pink ring = unseen.** Signed out: no ring at all (Skylar, 2026-10-10 — was always pink;
   a guest has no seen state, so the ring said nothing). Signed in: pink until the
   viewer has opened that affiliate's latest post; then the ring drops. The circle keeps its
   spot (Skylar, 2026-10-10 — it used to move to the end of the rail; seen is ring-only now).
   Other viewers are unaffected (seen is per viewer).
4. **"Seen" is a backend fact on posts, per viewer** — stamped whenever a post detail opens,
   from the rail or from the feed (so reading the same post in What's New also clears the
   circle). Only the rail consumes it today. **Scoped to affiliate-authored posts (Skylar,
   2026-10-10):** a row per viewer per post is a thousand rows for a post a thousand people
   open, and nothing reads them for other authors yet — so the stamp is a no-op for everyone
   else until a surface needs it.
5. **The rail response stays light**: per affiliate, is there a post, have I seen it, and the
   post id. The post body is fetched on tap. ~10 affiliates, so no scale concern.

## What exists today

- **Rail (mobile):** `AffiliateCircleCarousel` → `GET /profiles?type=AFFILIATE&page=0&limit=20`
  (`['profiles','affiliates','home']`, mirrored in the `useHomeReady` first-paint gate), ordered by
  username, every circle `ring="brand"` (pure dress, no state), tap → `UserProfile`.
- **Post detail (mobile):** route `PostDetail: { post: HydratedPostDto, postableType, postableId }`
  — a full post object, no id-only form. `usePostDetail` **skips** `GET /post/:id` when the
  feed card already carried every content item (`alreadyComplete`), so a detail read is not a
  reliable "the user opened this" signal.
- **Likes (api):** `PostLike` (`@@unique([accountId, postId])`), `POST /post/:id/like`
  behind `isRootPostViewable`, and `hydratePosts → withPostLikes` batching `likeCount` /
  `viewerHasLiked` onto every read. This is the pattern to mirror.
- **Viewer:** `resolveRequesterAccountId(principalId)` → account id or `null` for guests.
  Guests may only GET (the authorizer denies other methods).
- **Seen precedent:** `Connection.recipientSeenAt` (inbox friend-request row).
- **No "seen" concept anywhere, no latest-post lookup, no index for it** (`Post` has only
  `@@index([accountId])`; "latest" means `createdAt`, there is no `publishedAt`).

## Design

### API

**1. `PostSeen` model** — a per-viewer, per-post row, idempotent, mirroring `PostLike`:

```prisma
model PostSeen {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  seenAt    DateTime @updatedAt
  account   Account  @relation(fields: [accountId], references: [id], onDelete: Cascade)
  accountId String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId    String
  @@unique([accountId, postId])
  @@index([postId])
  @@index([accountId])
}
```

Back-relations `Account.postSeens PostSeen[]`, `Post.seens PostSeen[]`. Plus one index on
`Post` for the latest-post lookup (and the profile Posts tab gets it for free):
`@@index([accountId, postableType, createdAt])`. Migration `20261009120000_post_seen`
(create table + both indexes), Skylar runs it. `seenAt` re-stamps on every open — "last
opened", which is the more useful fact later; "first seen" is `createdAt`.

**2. `POST /post/:id/seen` `{ accountId }`** — the stamp. `validateAccount` (authenticated),
`isRootPostViewable` gate (404 like likes), then the **author-type gate** (2026-10-10): the
post is read with `account.profile.type`, and unless `isPostSeenTracked(type)` — i.e. the type
is in `POST_SEEN_AUTHOR_TYPES = ['AFFILIATE']` in `utils/postSeen.ts` — the route answers
`200 { seen: false, seenAt: null }` without writing. Otherwise
`prisma.postSeen.upsert({ where: { accountId_postId }, create, update: {} })` (`@updatedAt`
bumps `seenAt`) and `{ seen: true, seenAt }`. Fire-and-forget on the client. Explicit rather
than a side effect of `GET /post/:id`, because that read doesn't always happen (see "What
exists"). Guests never call it. Widening (VERIFIED, everyone) is that one constant plus its
mobile mirror.

**3. `viewerHasSeen` on post reads — deferred.** `withPostSeen` exists in `utils/postSeen.ts`
(one `findMany` per page, `false` for guests) but is NOT wired into `hydratePosts` yet
(Skylar, 2026-10-09: nothing renders it, so don't pay the query). One line to enable when a
surface needs it.

**4. `GET /profiles/affiliates/latest-posts`** — the one call the rail makes (renamed from
`/rail` on 2026-10-10: "rail" named the UI, not the payload). Path is plural on purpose:
`/profile/latest-posts` would be swallowed by `/profile/:id`. Public (guests allowed); viewer resolved
when present. Three queries:

1. Profiles: `type: 'AFFILIATE', isPublic: true, username: { not: null }` (VERIFIED never
   appears here — settled 2026-10-08), minus authors the viewer has blocked / been blocked by
   (`findBlockedConnectionAccountIds`, the same subtraction `GET /posts` makes).
2. Latest root brand post per affiliate: `post.findMany({ where: { accountId: { in }, postableType: 'BRAND',
   parentId: null, status: 'ACTIVE' }, orderBy: { createdAt: 'desc' }, distinct: ['accountId'],
   select: { id, accountId, createdAt } })`.
3. Viewer's seen rows for those post ids (skipped for guests).

Response, ordered on the server so every client shows the same rail:

```json
{ "data": [
  { "id": "<profileId>", "accountId": "…", "username": "…", "avatar": "…",
    "latestPostId": "<postId>|null", "latestPostAt": "<iso>|null", "viewerHasSeen": false }
] }
```

**Order rule (Skylar, 2026-10-10):** (a) has a post, newest first → (b) no post yet, username
asc. Seen is NOT a sort key — it only drives the ring, so a circle stays where recency put it
after the viewer opens its post. (Until 2026-10-10 seen affiliates were bucketed to the end;
dropped because the rail visibly reshuffled under the reader on every open.) A guest's rail is
the same order, rendered with quiet rings. A new post is a new id, so it is unseen by
everyone until opened: "new post moves them to the front" falls out with no extra state.

**5. Types:** `PostSeenDto` arrives automatically (Prisma → JSON schema → types); the rail
row type does not (route shapes aren't in the package), so mobile declares `AffiliateRailItem`
locally. Bump `packages/types` → `0.1.107`.

**Keep:** `GET /profiles` as is (admin / other callers). `POST /post/:id/like` untouched.

### Mobile

1. **Rail data.** `profileService.latestPosts()` → `GET /profiles/affiliates/latest-posts`.
   Query key `['profiles','affiliates','latest-posts', viewerKey]` (viewer-scoped, like posts — the old key
   wasn't, and this response now differs per viewer). Same key + fn mirrored in `useHomeReady`,
   so the rail stays in the first-paint gate. `AFFILIATES_DISPLAY_COUNT` goes; the endpoint
   returns every affiliate.
2. **Ring.** `CircleThumb` gains `ring?: 'brand' | 'quiet'`; `circleThumbRingQuiet` is the
   default `border` grey at the brand ring's 2.5 width (so the image doesn't breathe when the
   state flips). Rule in the rail: pink only when there is a post the viewer hasn't seen —
   `item.latestPostId && (!isSignedIn || !item.viewerHasSeen) ? 'brand' : 'quiet'`. An
   affiliate with **no post yet** wears the quiet ring for everyone, guests included (Skylar,
   2026-10-09: "wait until they have a post to show the ring"); the pink appears with their
   first post.
3. **Tap.** `latestPostId` → `postService.get(latestPostId)` (one `GET /post/:id`, the normal
   hydrated shape) → `navigation.push('PostDetail', { post, postableType: 'BRAND', postableId:
   post.postableId })`. The circle shows `pending` (the existing 0.45 dim) while the fetch is in
   flight; a failed fetch falls back to the profile. No `latestPostId` → `UserProfile` as today.
   `push`, never `navigate` (house rule).
4. **Stamp.** `useMarkPostSeen(livePost)` in `PostDetailScreen`: on mount, signed-in only,
   once per open, **and only when `post.account.profile.type` is in the hook's mirror of
   `POST_SEEN_AUTHOR_TYPES` (`['AFFILIATE']`, 2026-10-10)** — so no request leaves the device
   for anyone else's post; `profile.type` rides on every post read via `postInclude`. Then
   `postService.markSeen(id, { accountId })` fire-and-forget. On success:
   `setQueriesData` flips `viewerHasSeen` on the matching rail row (immediate ring change
   under the sheet). No invalidate since 2026-10-10: seen no longer changes the order, so the
   cached rail is already right once the flag flips. Every post detail does this, feed or rail,
   which is what makes "read it in What's New" clear the circle too.
   **Posting refreshes the rail (Skylar, 2026-10-10):** both composer publish paths (create and
   draft-publish-in-place) invalidate the rail key with `refetchType: 'active'` when the author
   is an affiliate and the post is a root BRAND post — Home sits under the composer modal, so
   the author watches their own circle jump to the front, pink, as the sheet closes. Before this
   it stayed stale until an app reload. Group posts and replies don't touch the rail.
5. ~~`viewerHasSeen` on `HydratedPostDto`~~ — not sent yet (API §3 deferred), so nothing to add.

### Not in scope

A badge or indicator for seen anywhere else; a "seen by" count; marking replies seen (the
stamp is on the root post the detail shows); prefetching rail posts (fetch on tap, by request).

## Open questions for Skylar

1. ~~Affiliates with no post yet — pink or grey?~~ **Answered (Skylar, 2026-10-09): no pink
   until they have a post.** Quiet ring, end of the rail, for guests and members alike.
2. ~~Re-open = re-stamp.~~ **Answered (Skylar, 2026-10-09): yes, update the timestamp per open.**
3. ~~`viewerHasSeen` on every post read — now or defer?~~ **Answered (Skylar, 2026-10-09):
   deferred as the cheaper option.** Helper built, not wired (API §3).

## Scope change 2026-10-10 — affiliate-authored posts only

Skylar, before deploying: "we only want to limit this for affiliates for now". Built the same
day in both repos: `POST_SEEN_AUTHOR_TYPES` + `isPostSeenTracked` in `utils/postSeen.ts`, the
route no-ops (`{ seen: false }`) for other authors, and `useMarkPostSeen` takes the post and
skips the request unless the author is an affiliate. Rows for BASIC/VERIFIED authors are never
written. The `PostSeen` model, migration and rail endpoint are unchanged.

## Order of work

1. ~~nanza-api~~ Done 2026-10-09: `PostSeen` + `Post` index, migration, `utils/postSeen.ts`
   (`markPostSeen`, `getViewerSeenPostIds`, `withPostSeen` — the last NOT wired, see §3),
   `POST /post/:id/seen`, `GET /profiles/affiliates/latest-posts` (was `/rail` until
   2026-10-10, same day the seen bucket left the sort), types `0.1.107`.
2. ~~nanza-mobile~~ Done 2026-10-09: `profileService.latestPosts` + `AffiliateRailItem` +
   viewer-scoped key, `postService.markSeen`, `CircleThumb` `ring="quiet"` +
   `circleThumbRingQuiet`, `AffiliateCircleCarousel` rewritten (ring rule, tap → fetch →
   push, pending dim, profile fallback), `useHomeReady` mirror, `useMarkPostSeen` in
   `PostDetailScreen`.
3. Skylar: run the migration (dev), post as an affiliate, check guest vs member rails, open from
   rail and from feed, confirm the ring clears in place (QA 2026-10-10: seen works signed in;
   the move-to-end was the thing that got dropped).
4. Fold into the solution designs (posts.md Seen section; whats-new.md line 49 is already
   stale; groups.md ring meaning; verified-profiles plan status), delete this plan.
