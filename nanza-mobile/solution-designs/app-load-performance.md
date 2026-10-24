---
tags: [nanza-mobile, solution-design, performance, home, cart, posts, session]
---

# App-Load Performance — the five-request diet

> **Status: shipped in code 2026-08-14 (pending review/deploy).** One pass over every request the
> app fires on cold start, driven by a network audit: each request now fetches only what the first
> screen actually renders, and everything heavier moved behind the interaction that needs it.
> API halves live in nanza-api (`cartPreview.ts`, `user.ts view=session`, the posts list trim —
> see [[../nanza-api/solution-designs/posts|nanza-api Posts]] § Payload budget).

## The rule this release established

**A load-time request may only carry what load-time UI renders.** Anything else — full carts,
list aggregates, post item trees, seller shipping matrices — belongs to the screen that opens it,
fetched on that screen's mount with the load-time copy as placeholder where a instant paint
matters. This is the payload-budget rule from the Posts design, generalized to the whole cold
start.

## The two-request gate (2026-09-28; three from 2026-09-11)

The home page's first paint now waits on **exactly two requests**, the same for guests and
signed-in viewers: Trending (primary tag values) and Groups. `useHomeReady` fetches both
together and latches `isReady`; everything else lands after the commit and fills in where it
belongs.

**Affiliates left the gate on 2026-09-28** along with the row itself: the affiliate circle
carousel that opened the page is gone (component deleted, `GET /profiles?type=AFFILIATE` no
longer fires from Home), so the gate went from three requests to two. The page now opens with
Trending — untitled, no divider — and the order below it is **What's new 1–3 → Groups → What's
new 4–6 → Query List 1 → What's new 7–9 → Query List 2 → Query List 3** (Groups moved down
under the first band; Query List 3 follows Query List 2 directly since there are only nine
posts). Cutting the affiliate request also drops one round trip from the cold start.

- **`/posts` (the What's-new bands) is off the gate.** It was a fourth gate and the most
  expensive read on the page, and it was stretching the cold start. `WhatsNewSection` now owns
  the request — it mounts once the page is up and calls `usePosts` itself — so the three bands
  appear as the posts land. (Was the fourth of three gates; now the third of two.) The one-cache-entry rule still holds (every band and PostThread
  share the `['Posts','BRAND',id,viewer]` key); only *when* it fires changed. *(Superseded
  2026-10-18: the read is three posts, then infinite scroll — see below.)*
- **Query List shelves** stay behind `deferredActive` (after interactions), as before.
  *(Retired from Home 2026-10-18 — see below.)*
- **Groups shelf shows for guests.** `GroupBannerCarousel` had hidden itself for signed-out
  viewers since 2026-08-30; it now renders for everyone so the page reads the same in both
  states, and guest tiles open the group detail like anyone else's (with no Join pill on the
  tile or the detail) — see [[groups|Groups]].

## Affiliates back in the gate (2026-10-08)

The affiliate circles returned to the top of Home on 2026-10-08 (see
[[../nanza-api/solution-designs/affiliate-rail-latest-post-and-seen|Affiliate Rail]]), and with
them their one request — now `GET /profiles/affiliates/latest-posts` — rejoined `useHomeReady`.
The gate is **three requests** again: affiliates, Trending, Groups. Posts and the Query List
shelves stay off it.

## The feed goes infinite; Query Lists leave Home (2026-10-18)

Skylar's call, from a request audit of the home page: **the page should never load a post it is
not about to show.** Two changes, both in `HomeScreen`:

- **`/posts` asks for THREE, not nine.** `usePosts` / `postsQueryOptions` take a `pageSize`
  (default still 9) and the size is now **part of the query key** —
  `['Posts', type, id, viewer, pageSize]` — so the homepage pages at
  `HOME_POSTS_PAGE_SIZE = 3` (exported from `WhatsNewSection`) in its own cache entry while the
  What's New screen, tag/set pages and detail threads keep theirs at 9. Invalidations match on
  the `['Posts', type, id]` prefix and reach every size. Every reader of the home entry passes the
  same size (the bands via `PostThread`'s new `pageSize` prop, HomeScreen's own paging hook,
  `HomeViewerSwap`'s sign-in prefetch) — the one-cache-entry rule now holds *per surface*.
- **The page is an infinite scroll.** The FlatList's rows are the feed's loaded pages after the
  first, each rendered as an untitled 3-post band (`WhatsNewSection start=3,6,9…`, keyed by the
  index of its first post), and `onEndReached` (threshold one viewport) calls `fetchNextPage` for
  the next three. Band 1 (titled, with "See all") stays in the list header over Groups. The
  shape is now **circles → Trending → What's new 1–3 → Groups → 4–6 → 7–9 → … → spinner while
  the next page is in flight → legal footer**. Stacked bands keep the 8pt post-to-post seam
  (the band's 2pt bottom margin plus the thread's 6pt bottom padding), so the rows read as one
  feed. Virtualised at band granularity.
- **The Query List shelves and `/query-lists/homepage` are gone from Home.** `QueryListShelf`,
  `QueryListShelves`, `useHomepageQueryLists`, `queryListService.getHomepage` and the
  `HomepageQueryList` type are deleted; `FeedGridScreen` still pages query-list results
  through `getResults`. The API endpoint stays (no mobile callers). The home page's cold start
  is therefore the three gate requests plus one 3-post read — nothing else fires until the
  reader scrolls.
- **`isSwitchingBrand` is now derived from the feed.** It used to ride the query-lists read
  (brand-tagged response); it is now "the brand whose feed has loaded" (a ref set when the
  home posts entry reports `isFetched && !isLoading`) differing from `selectedBrandId`. Unset
  on the cold open, so the first `/posts` landing after the paint never blanks the shelves; a
  brand visited earlier reports fetched from cache and shows at once.
- **`PostDetailScreen` no longer mounts `usePosts`** to find the tapped card's thread copy —
  with the homepage on its own key that read fetched a 9-post page of its own. It uses
  `useCachedPost`, a read-only `getQueriesData` lookup across every size's entry for the
  thread.

## The sign-in swap (2026-10-10)

The cold-start gate opened the page whole, but **signing in (or out) undid it**: the live viewer
flips the moment the session lands, and every viewer-keyed read — the affiliate rail
(`viewerHasSeen`), Groups (memberships), the `/posts` behind the three What's new bands — gets a
brand-new empty cache entry at once. Each section collapsed to nothing and popped back as its own
request returned, so the page re-assembled itself shelf by shelf under the reader.

**Fix: a committed viewer that lags the live one.** `contexts/CommittedViewerContext` exposes
`useViewerAccountId()` / `useViewerKey()` — the live session by default (so nothing changes
anywhere else), overridable by a provider. `screens/home/data/HomeViewerSwap` wraps HomeScreen
and provides it:

1. On the cold start it commits the session's viewer as soon as auth is determined (no
   prefetch — `useHomeReady` is already holding the page).
2. When the live viewer later changes, the sections keep rendering the committed (old) viewer's
   cached data while the provider **prefetches the new viewer's three reads** into the cache —
   `affiliateRailQueryOptions`, `homeGroupsQueryOptions`, `postsQueryOptions`, the exact option
   objects the sections spread, so the entries are the ones they will read.
3. Once all three have settled (settled, not succeeded — same posture as the gate), the
   committed viewer advances and every section switches key in the same render, onto warm
   cache. One swap, no collapse, no stagger. A brand switch or a second session change mid-flight
   cancels and restarts the wait.

`usePosts` and `useHomeReady` read the viewer from the context too, so under Home the gate and
the bands follow the committed viewer; `PostDetailScreen`, `WhatsNewScreen` and the rest sit
outside the provider and keep the live viewer. The affiliate ring also keys on the committed
viewer, so a guest's rows never show a member's pink ring during the swap. Trending and the Query
List shelves are not viewer-scoped and never move.

## What loads now (and what moved)

### 1. Home feed: `/feed` is retired on the frontend
> *Historical (2026-08-14 → 2026-10-18): the Query List shelves described here left Home on
> 2026-10-18 — see "The feed goes infinite" above. `FeedGridScreen` still pages query-list
> results.*
- Top Picks and its `/feed` request (limit 20) are **gone from HomeScreen**. The homepage is now
  groups → tags → What's-new band 1 → **Query List 1** (in Top Picks' old slot) → band 2 →
  Query List 2 → band 3 → Query List 3.
- `/query-lists/homepage` fetches with **`limit=3`** — exactly the three shelves the page renders,
  each with its first page of results embedded (one request, no per-shelf calls). Since
  2026-08-18 each row is **slim**: `id`, `name`, `displayName`, `index`, `brandId` + `results` /
  `resultsTotal` — the list's config (criteria, combinator, types, images, audit columns) is
  what the api resolves on but the shelf never reads, so it stays server-side.
- **Empty lists never occupy an interleave slot.** The interleave addresses shelves by index,
  so a list that draws nothing must not count — an empty first list collapsed bands 1 and 2
  into six straight posts (found 2026-08-14). The guard is **client-side only** (Skylar's
  call): `useHomepageQueryLists` filters to lists with at least one card of a renderable type
  (LISTING/BID/BULK_LISTING — ENTITY has no homepage card), so `index` always means "the Nth
  shelf that actually draws". The API returns the first `limit` primary lists as-is — homepage
  lists are curated to have items, so an empty one is an admin-side surprise, and the page
  just shows fewer shelves that session.
- `QueryListShelves` exports `useHomepageQueryLists(active)`; HomeScreen shares the same query to
  derive `isSwitchingBrand` (the brand-tagging pattern the retired feed query used).
- `FeedGridScreen` behind "See all" pages **query-list results only** — the `{ kind: 'feed' }`
  route source and `services/api/Feed.ts` are deleted. The API `/feed` endpoint stays (no FE
  callers). `['Feed']` invalidations across the app became `['queryLists']` (or were dropped where
  one already existed).

### 2. Posts: limit 9, two items per card, detail fetches the tree
- `usePosts` PAGE_SIZE is **9** — the three 3-post What's-new bands, no over-fetch. One value per
  `['Posts', type, id]` key: every consumer shares the cache, so the size can't vary per caller.
  *(2026-10-18: the size is now a `pageSize` argument and part of the key; Home pages at 3, the
  default stays 9 — see "The feed goes infinite" above.)*
- The API list read returns only the **first text run + primary attachment** per post (see the
  nanza-api Posts design for the trim mechanics and why it's not `take: 2`).
- **PostDetailScreen now fetches `GET /post/:id`** (`usePostDetail`, key `['PostDetail', id]`)
  with the tapped card as `placeholderData` — text/primary paint instantly, carousel slots and
  galleries fill in. Mutations that change a post (edit, set-primary, reply, body update)
  invalidate `['PostDetail', id]` alongside the thread keys.
- **The by-id fetch is skipped when nothing was trimmed.** List rows carry `contentItemCount`;
  when the card's `contentItems.length` already covers it (most posts have ≤2 items), opening a
  post costs only the replies request. Rows cached from before the field existed just fall back
  to fetching.
- **The replies fetch is skipped when `_count.replies` is 0.** So opening a typical no-reply,
  no-gallery post costs **zero** extra requests — the card already is the post. Posting a reply
  refetches the thread caches, the count goes above zero, and the replies query switches on.

### 3. Cart: the orders request stays whole, just deferred off the cold start
- **REVERTED same-day (Skylar's call):** a `/cart-preview` + on-open-`/orders` split was built,
  but the orders flow is released and the split touched its data path — deleting a cart item
  went stale (the context's direct `refetch` became an invalidation the `refetchType:'none'`
  client default never ran) and the empty-cart auto-close threw `GO_BACK`. Rather than QA the
  whole checkout surface for a load-time win, the split was backed out: one `['CartOrders']`
  query in `CartContext`, exactly as released, and the API `cartPreview.ts` router was removed.
- The load-time win kept: `fetchCartOrders` is now **gated on `isAppReady`** (the splash-lift
  latch from AppReadyContext), so the heaviest launch request fires right after first paint
  instead of competing with it — same deferral idea as the homepage shelves, one line of risk.
- Durable lesson for a future retry: `refreshCart` consumers rely on a **direct refetch**;
  any future split must not swap it for invalidation (the app-wide default is
  `refetchType:'none'` — see [[data-freshness|Data Freshness]]), and the empty-cart auto-close
  in ReviewCartScreen fires on whatever query it watches going empty.

### 4. Session user: `view=session` skips the derived aggregates
- `GET /user/:authId` with `account.profile` included used to compute **`profile.deals`** (two
  order-count aggregates) and **`enrichListsBatch`** (walks every EntityList row with product
  prices, per list) on every app start — the actual cost of the "user request".
- `view=session` (passed by SessionContext and the signup flow) skips both; the reference-code
  mint stays. The include set is unchanged — seller/customer rows are cheap single rows that
  Wallet/Selling/checkout read from `currentUser`, so dropping them would break real screens.
- The two consumers that rendered those aggregates from the session now fetch their own:
  own-profile **Deals** reads `GET /profile/:id` (which computes deals regardless of includes);
  the composer's **CardPickerLayer** collection rows read `getAllListsQueryConfig` like the
  sibling collection screens. Everything else only ever read `id/type/isPrimary` from session
  lists (verified by audit 2026-08-14).

## Known trade-offs
- Opening the cart now shows a spinner on first open (the full orders are on the wire) instead of
  rendering instantly from an always-loaded cache. Deliberate: every session paid the old cost,
  few sessions open the cart.
- ~~A brand with more than three query lists shows only the first three on Home~~ — moot since
  2026-10-18: query lists no longer render on Home.
- The home feed pages three posts per scroll (2026-10-18), so a reader who scrolls far makes
  many small `/posts` requests rather than a few large ones. Deliberate: the cold start pays for
  exactly three, and the later pages ride the scroll.
- `refreshCurrentUser` still refetches the whole session shape from ~20 call sites. Next candidate
  if the session call is still hot: a targeted `refreshLists()` for the four collection-create
  call sites.
