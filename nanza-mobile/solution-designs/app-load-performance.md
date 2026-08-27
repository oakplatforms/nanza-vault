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

## What loads now (and what moved)

### 1. Home feed: `/feed` is retired on the frontend
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
- A brand with more than three query lists shows only the first three on Home; "See all" pages the
  full list per shelf. The admin-side ordering (`index`) decides which three.
- `refreshCurrentUser` still refetches the whole session shape from ~20 call sites. Next candidate
  if the session call is still hot: a targeted `refreshLists()` for the four collection-create
  call sites.
