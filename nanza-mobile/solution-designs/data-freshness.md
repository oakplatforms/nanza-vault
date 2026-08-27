---
tags:
  - nanza-mobile
  - solution-design
  - data-freshness
---

# Data Freshness (React Query)

How nanza-mobile keeps data fresh without refetch storms. The model is **lazy-invalidate +
refetch-on-page-visit**, with a deliberate split between *price values* (fetched actively) and
*discovery surfaces* (marked stale, refetched when you open them).

## The problem this solves

Nearly every query used `staleTime: 0` and mutations invalidated broad top-level keys
(`['Feed']`, `['search']`, `['lists']`, …). With staleTime 0, `invalidateQueries` refetches
every *mounted* matching query immediately — so changing one value (a collection quantity, a
listing price) refetched the home feed, search, saved items, and every mounted list at once.
Because the search overlay sits at the top of the app tree, all those discovery queries were
mounted behind it, amplifying the storm.

## The model

1. **Invalidation is lazy by default.** `App.tsx` installs an `AppQueryClient` subclass whose
   `invalidateQueries` defaults `refetchType: 'none'` — it marks queries stale **without**
   refetching a mounted-but-backgrounded screen.
2. **Screens refetch when they regain focus.** `AppNavigator`'s `NavigationContainer
   onStateChange` (debounced) calls `refetchQueries({ type: 'active', stale: true })` — on every
   navigation settle it refetches only the now-visible screen's queries that are mounted **and**
   stale. Fresh screens do nothing → no storm.
3. **`staleTime` tiers** (`src/constants.ts` `STALE_TIME`) decide "how stale is stale":
   - `INSTANT` (0) — price-critical live surfaces: the Entity modal
     (`ProductListings`, `ProductBids`, `entityDetail`).
   - `30_SECS` — high-churn discovery: the homepage querylist shelves
     (`['queryLists','homepage']`; the `['Feed']` query is retired — see
     [[app-load-performance|App-load performance]]).
   - `LIVE` (5 min) — the default cache window for everything else. **Not 0** — 0 makes every
     focus refetch everything.
4. **Same-screen updates opt in explicitly.** Where the user stays on a screen and must see the
   change live (message thread, comments, group moderation, the realtime socket
   `services/realtime/socket.ts`), the invalidation passes `refetchType: 'active'`. Same-screen
   modals (no navigation change, so onStateChange doesn't fire) call the screen's own
   `refetch()` directly after save (e.g. `CollectionDetailScreen.handleSheetUpdate`).

## The values-vs-surfaces split (the key rule for mutations)

A listing/bid edit affects two kinds of thing, handled differently:

- **Price VALUES** — lowest ask / highest bid. The Entity modal's "Buy N for $X" reads
  `['ProductListings', entityId]`, falling back to `entityDetail.lowestAsk`. These are
  invalidated with **`refetchType: 'active'`** so the value is current immediately.
- **Discovery surfaces** — `['queryLists']`, `['entities']`, other lists (`['Feed']` retired
  2026-08-14; its invalidations became `['queryLists']`). These are
  invalidated **lazily** (default). We do **NOT** `setQueriesData`-patch the feed to inject the
  new value (that fights the model and caused stale/duplicated data) and we do **NOT** actively
  refetch them on edit — they re-pull the real value when the user next opens that page.

See `src/components/listings/EditListing/index.tsx` for the reference implementation of this
split (both the edit and delete branches).

## Homepage load order (paint whole, then fill in)

Freshness decides *when* a query refetches; this decides *when the page is allowed to show*.
The homepage used to let every shelf resolve on its own clock, so it assembled itself row by
row — groups, then Trending, then What's new, then the feed — shifting under the reader. The
model now has two tiers:

1. **Above the fold, resolved together.** `screens/home/data/useHomeReady.ts` runs the groups,
   Trending and What's-new queries in one `useQueries` and reports a single `isReady`.
   `HomeScreen` holds one `LoadingIndicator` until that plus brands plus auth are all in hand,
   then commits the whole first screen at once. `isReady` waits for queries to be **settled,
   not successful** — a shelf that errors renders nothing anyway, and blocking on it would
   strand the reader on a spinner.

   This is a **readiness gate, not an awaited batch**: no `Promise.all`, nothing chained. The
   three queries fetch independently on their own clocks; the gate reads their subscriptions
   and only decides when to render, leaving each one's retry, cache and error handling
   untouched.

   **The gate is one-way, and it has to be (2026-08-09).** It is *not* plain
   `results.every(r => !r.isPending)` — that shipped a cold-load flash. `selectedBrandId`
   starts `null` and is only assigned by an effect after `['brands']` resolves, so `enabled`
   flips false→true *after* first paint. The three queries are independent store
   subscriptions and don't transition atomically, so for one commit they can all read
   non-pending while still holding no data. Because `HomeScreen` renders its loading state as
   a **sibling branch** (an early `return`), that single frame unmounted the spinner, painted
   an empty page, and remounted the spinner — read as "loads, flashes, reloads". Two things
   fix it together: the settled check also requires `!r.isFetching`, which closes the window
   `isPending` leaves open on a just-enabled query; and readiness is **latched in a ref**, so
   it can never go true→false and tear the tree down. Readiness is one-way by nature — the
   first screen can't un-have its data — and the latch encodes that. Brand switches are
   unaffected: they're handled in-page by `isSwitchingBrand`, which never unmounts.

   **The splash waits for readiness, it doesn't guess (2026-08-09).** The above fixed iOS but
   not Android, because the real whole-screen flash was upstream: `SplashAnimation` dismissed
   on a fixed 2.5s timer with no knowledge of what was underneath it. iOS usually had home
   ready by then so the hand-off was invisible; Android reaches TTI later, so the splash
   regularly peeled away onto a still-loading screen — splash out, spinner, then the page fade
   in. Two visible transitions where there should be one. `AppReadyContext` (wrapping both the
   navigator and the splash in `App.tsx`) lets the first screen signal `markAppReady()`, and
   the splash lifts only once **both** a 2.5s floor has passed (the brand animation gets its
   moment) and the app has real content.

   Because that overlay covers the entire app, **failing to lift is the only unacceptable
   outcome**, so three independent paths bring it down: the ready signal, a hard
   `SPLASH_MAX_MS` (6s) ceiling that never consults readiness, and the animation callback
   (which fires on cancel as well as finish). A `startedRef` makes the fade idempotent so
   those racing is harmless, and `useAppReady` returns an inert fallback rather than throwing
   when no provider is present — a cosmetic hand-off must never crash the app. Worst realistic
   case is the splash lingering to 6s, never sticking.

   **Deep-link cold starts release the splash from the navigator.** Only Home participates in
   the handshake, but a share link cold-starts onto a state of just `[Share]` (and `wallet`,
   `user/:id` likewise skip Main), so Home never mounts and would leave the splash riding to
   its ceiling. `AppNavigator`'s `onReady` checks the root route and calls `markAppReady()`
   when it isn't `Main` — the destination screen has its own loading UI and speaks for itself.

   **Native insurance.** Android had no `windowBackground` and inherited
   `Theme.AppCompat.DayNight`'s **white** default under a dark-only app, so any frame where no
   React view had painted flashed white — during bundle load, and in the single-frame gap
   Android can leave when a native-driven `Animated.View` tears down its hardware layer. It's
   now pinned to `@color/app_window_background` (`#010101`, mirroring `neutral.950`), and
   `bottomTabNavigatorContainer` got a background so Fabric can't flatten it away and expose
   the window. These make a dropped frame invisible rather than merely rare.

   The commit itself **cross-fades** rather than cutting. The content fades in over 220ms
   while the spinner fades out over 160ms, both on the native driver, and the spinner is only
   unmounted once its animation finishes (`spinnerDone`). The overlap is the point:
   unmounting the spinner the instant the data landed left a frame where it was already gone
   and the content was still at opacity 0, which read as a blink. During the fade the spinner
   sits in the main tree as an `absoluteFill` overlay with `pointerEvents="none"`, so the
   live page beneath it is already interactive. `TopHeader` renders in *both* branches and
   stays fixed across the whole transition — fading it too would read as the entire screen
   re-rendering, the exact thing this is meant to stop.

   **Exactly three things block the page: groups, Trending, What's new.** Nothing else may be
   added to that list. Inbox count, wallet balance and the rest of the header are deliberately
   *not* in the gate — `TopHeader` renders outside the `FlatList` and owns its own queries, so
   it fills in on its own without holding the page. Brands and auth-determined are the only
   other conditions, and only because the shelves need a `selectedBrandId` to query and a
   signed-in user shouldn't flash the guest state.
2. **Below the fold, deferred.** The Query List shelves (`['queryLists','homepage']` — one
   request for all three, Top Picks and `['Feed']` retired) are gated behind `deferredActive`, flipped by
   `InteractionManager.runAfterInteractions` **once `isHomeReady` is true**. They fetch during
   the window between paint and the reader's first scroll, so they've landed by the time
   anyone reaches them.

The groups and Trending queries in `useHomeReady` deliberately **mirror** the ones inside
`GroupCircleCarousel` and `TagCarousel` — same keys, same params, same `staleTime`. React Query
dedupes on the key, so the children read that cache entry rather than firing a second request,
and each component stays self-contained for use on other screens. **If you change a mirrored
query's key or params, change it in both places** or the homepage will double-fetch.

**What's new is deliberately not mirrored.** Posts are an `useInfiniteQuery`, which `useQueries`
can't run, and the first attempt hand-rolled a plain query beside it under
`['Posts','BRAND',id,'first']` — one segment off `usePosts`'s `['Posts','BRAND',id]`, therefore a
separate cache entry, therefore two `/posts` requests per homepage load. The fix is that
`useHomeReady` just calls `usePosts` itself and reads its `isLoading`. A near-copy of a query is
a double fetch; call the hook.

### The post bands (3 / 3 / 3)

What's new isn't one block — the homepage weaves **nine posts from that single request** through
the page in three 3-post bands:

```
groups · Trending · What's new 1–3     ← the only titled band
Top picks
What's new 4–6  →  Query List 1
What's new 7–9  →  Query List 2
legal footer
```

The bands sit **before** their query list, not after. That's the whole point of them — they
break up the carousels, so a band has to land between Top picks and Query List 1, and between
the two lists. Attaching them after each shelf (the first attempt) left Top picks running
straight into Query List 1 and stranded the last band at the bottom of the page.

Mechanically: `WhatsNewSection` takes `start` (0/3/6) and `showHeading`; `PostThread` grew a
`previewStart` to offset its preview window (it could previously only slice from 0); and
`QueryListShelves` grew an `index` + `renderBefore` so the homepage can emit one shelf at a time
with its band ahead of it, instead of the whole stack at once. Passing neither still renders the
full stack, which is what every other caller wants.

Only band 1 carries the "What's new" heading and its "See all" pill — they introduce the feed
once. Repeating them would read as three separate sections rather than one feed threaded through
the page. A band renders only alongside its query list (`renderBefore` fires from inside the
shelf's own existence check) and renders nothing when its window starts past the last post — so
a brand with fewer posts or one query list ends early rather than leaving an empty titled shell
or orphaned posts.

One consequence worth keeping: the mid-page spinner is now scoped to brand switching only.
On a cold open the first shelf is legitimately empty for a beat after commit, and a spinner there
would reintroduce exactly the stagger this removes — the shelf just appears when it lands.

## Gotchas

- **Persistent screens.** Bottom tabs are `lazy: true` but NOT `unmountOnBlur`, and stack
  screens stay mounted — so "refresh on next mount" never fires on return. The global
  onStateChange handler is what refreshes them; there is no per-screen `useRefreshOnFocus` (an
  earlier version had one; it was removed as redundant).
- **Invalidate the key the UI actually reads.** The classic bug: a listing edit invalidated
  `['UserProductListing']`/`['AccountListings']`/etc. but not `['ProductListings', entityId]`,
  so the Buy button never updated. Trace the consumer.
- **Snapshot fallbacks.** The Entity modal falls back to `entityDetail.lowestAsk` (a fetched,
  staleTime-0 query), NOT the raw route-param `entity.lowestAsk` snapshot — the snapshot is
  never refreshed.
- **Orders key mismatch:** the orders screens read `['BuyerOrders']`/`['SellerOrders']` but some
  mutations invalidate `['orders']` — they don't match; a latent bug to fix.

## Related

- [[architecture|Architecture]] — the provider tree / navigation wiring these hook into.
- Project `CLAUDE.md` → "Query freshness" (the enforced rules, rules 26–30).
