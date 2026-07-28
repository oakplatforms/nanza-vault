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
   - `30_SECS` — high-churn discovery: homepage feed (`['Feed']`) + querylist shelves
     (`['queryLists','homepage']`).
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
- **Discovery surfaces** — `['Feed']`, `['queryLists']`, `['entities']`, other lists. These are
  invalidated **lazily** (default). We do **NOT** `setQueriesData`-patch the feed to inject the
  new value (that fights the model and caused stale/duplicated data) and we do **NOT** actively
  refetch them on edit — they re-pull the real value when the user next opens that page.

See `src/components/listings/EditListing/index.tsx` for the reference implementation of this
split (both the edit and delete branches).

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
