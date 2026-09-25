# React Query Invalidation Review

**Status:** REVIEW / PLAN — no code changes. Decide rollout later.
**Date:** 2026-07-26
**Trigger:** Adding one item to a collection from the search overlay fired a burst of
refetches (`lists`, `feed`, `search`, `groups`, `homepage`). Investigation showed this
is **systemic**, not specific to that path.

---

## TL;DR

Two things multiply each other into refetch storms:

1. **`staleTime: 0` almost everywhere.** `STALE_TIME['LIVE']` (= 0) is used in **64** hooks
   — ~94% of all keyed queries ([src/constants.ts](../src/constants.ts)). With staleTime 0,
   `invalidateQueries(key)` doesn't just mark stale — it **immediately refetches every
   MOUNTED query** under that key. (Global defaults: `staleTime` 5min, `refetchOnMount: true`,
   `refetchOnWindowFocus: false`, `refetchOnReconnect: true` — [App.tsx:32-46](../App.tsx).)
2. **Broad top-level keys invalidated on nearly every mutation.** The shared chokepoint roots
   — `['Feed']`, `['entities']`, `['search']`, `['filteredProductEntities']`, `['lists']`,
   `['listEntities']`, `['collectTab']`, `['savedItems']`, `['connections']`, `['groups']` —
   are each subscribed by many mounted screens, so invalidating one refetches all of them.

Net: a single collection-quantity tap can refetch the home feed, global search, saved items,
and every mounted collection list — even though the row that changed **already updated
optimistically** and needed no refetch.

**Nothing nukes the whole cache** — no bare `invalidateQueries()`, no `resetQueries()`/
`clear()`. The problem is **breadth of keys**, not blanket clears. Good.

---

## Proposed convention (the house rule to adopt)

For any mutation's post-success cache work, split targets into three buckets:

1. **On-screen + NOT already optimistic** → `refetchQueries({ queryKey, type: 'active' })`.
   Refetch only what the user is looking at and that isn't already updated locally.
2. **Discovery / background surfaces** (feed, search, other lists, saved, homepage, groups) →
   `invalidateQueries({ queryKey, refetchType: 'none' })`. Mark stale so they refresh on their
   **next mount**, but don't refetch them while they sit unmounted-but-cached or mounted behind
   an overlay. `refetchOnMount: true` (already the global default) makes this correct.
3. **Already optimistic** (localQty rows, saved-items sets, setQueriesData patches) → **do
   nothing**. The UI is already right; a refetch just re-fetches what we just set.

Rule of thumb: **`refetchType: 'none'` is the default; `type: 'active'` is the exception,
used only for the exact surface that changed and can't self-update.**

Note: `refetchType: 'none'` is not yet used anywhere in the codebase — the good scoped patterns
that DO exist use `type: 'active'` on refetch and `predicate` on invalidate (see "Prior art").

---

## Ranked offenders (from the audit)

### 1. Collection add / qty / remove — WORST
- Helper: [src/utils/invalidateCollectionQueries.ts:7-24](../src/utils/invalidateCollectionQueries.ts)
  — 11 broad invalidations (`lists`, `list`, `Feed`, `entities`, `setEntities`,
  `filteredProductEntities`, `search`, `savedItems`, `userCollection`, `collectTab`,
  `collectionDetail`) + 2 scoped active refetches (`entityDetail`, `listEntities` — already good).
- Called from **5+ entry points**: `AddToCollectionScreen/index.tsx:86`;
  `usePersistCollectionQuantity.ts:21,25,37` (3 paths); `EntityModal/index.tsx:277`.
- **Duplicate**: a near-identical 10-key inline copy shadows the helper name in
  [EntityCard/List.tsx:151-166](../src/components/entity/EntityCard/List.tsx) (called on every
  qty tick :189 and remove :203).
- Rows already optimistic: `SearchEntityRow.tsx` `localQty`, `CollectionGridCard.tsx` `localQty`,
  `EntityCard/List.tsx` `localQuantityOverride` → the follow-up broad refetch is **redundant**.
- **Fix:** collapse the two helpers into one; discovery keys → `refetchType:'none'`; keep the
  two `type:'active'` refetches. De-dupe the EntityCard/List inline copy.

### 2. Scan-commit / Collect tab
- [scanItems.ts:69-91](../src/screens/trade/data/scanItems.ts),
  [CollectTab.tsx:326-332](../src/screens/trade/TradeScreen/CollectTab.tsx) — nearly the same
  9-key set (`Feed`, `entities`, `filteredProductEntities`, `search`, `lists`, `listEntities`,
  `userCollection`, `collectTab`, `UserProfileListings`).
- **Fix:** same convention. Scan-commit is a bulk operation, so a single post-commit refresh of
  the on-screen collect surface + `refetchType:'none'` for the rest.

### 3. Create / Edit listing
- [CreateListing/index.tsx:240-258](../src/components/listings/CreateListing/index.tsx),
  [EditListing/index.tsx:270-303,340-355](../src/components/listings/EditListing/index.tsx).
- Already uses predicates + an optimistic Feed patch — **but then broadly invalidates the same
  roots**, undoing the patch (the patch-then-invalidate anti-pattern). 
- **Fix:** keep the optimistic patch; drop the redundant broad invalidations of the patched roots;
  `refetchType:'none'` for the rest.

### 4. Create / Edit bid
- [CreateBid/index.tsx:133-138](../src/components/bids/CreateBid/index.tsx),
  [EditBid/index.tsx:135-269](../src/components/bids/EditBid/index.tsx). Already the **best**
  cluster — leans on predicates. Still touches `['entities']` broadly.
- **Fix:** minor; convert the broad `entities` hit to a predicate/`refetchType:'none'`.

### 5. Messaging
- `invalidateMessaging` ([mutations.ts:8-13](../src/screens/messages/MessagesScreen/data/mutations.ts))
  → `connections`, `conversations`, `messageFeed`, `inboxCount` from 5 connection mutations.
- **Amplified**: the realtime socket ([socket.ts:126-133](../src/services/realtime/socket.ts))
  invalidates the same roots on the echo push, so a send pays **twice** (local mutation + push).
- **Fix:** `refetchType:'none'` for the roots not currently on-screen; consider de-duping the
  local invalidation when the socket will push the same keys back.

### 6. Group join / create
- Unscoped `['groups']` from `CreateGroup`, `GroupDetailScreen`, `useShareActions`,
  `GroupCircleCarousel` (+ scoped `['groups','mine',accountId]`, `['group',groupId]`).
- **Fix:** prefer the scoped variants; `refetchType:'none'` for the broad list.

---

## Anti-patterns to remove
- **patch-then-invalidate**: `setQueriesData(root, …)` immediately followed by
  `invalidateQueries(root)` — EditListing (:253 patch vs later `entities`/`lists` invalidations),
  EditBid (:194,246). The invalidation refetches what the patch just set.
- **duplicate helper**: EntityCard/List.tsx inline copy of `invalidateCollectionQueries`.
- **double-pay on realtime**: local messaging invalidation + socket echo invalidation.

## Prior art already in the repo (reuse these patterns)
- `type: 'active'` refetch: `invalidateCollectionQueries.ts:22-23`, `EntityCard/List.tsx:165`,
  `BulkTab.tsx:301`.
- `predicate` narrowing: `Feed` predicate in BulkTab/BulkItems/useComments/EditListing/EditBid/
  CreateListing; `filteredProductEntities` predicate across bids/offers/listings;
  `homepageEntities||collectionEntities` predicate in EditListing/EditBid.
- Optimistic UI: `localQty` (SearchEntityRow, CollectionGridCard), `localQuantityOverride`
  (EntityCard/List), `SavedItemsContext` optimistic sets, `fetchRecentSearches` setQueryData.

---

## Suggested rollout order (when we decide to ship)
1. **Convention doc** — land the rule (this file → a short section in the vault
   `documentation/` frontend reference, so it's discoverable).
2. **Central helper** — one `invalidateCollectionQueries` with the 3-bucket split; delete the
   EntityCard/List duplicate. Verify: collection add/qty/remove from grid, search overlay,
   add-to-collection screen, entity modal — badge still correct, no feed/search refetch.
3. **Scan-commit / Collect tab** — apply convention. Verify a scan commit.
4. **Messaging** — `refetchType:'none'` + de-dupe vs socket. Verify send/connect updates inbox.
5. **Listings/bids** — remove patch-then-invalidate; predicates + `refetchType:'none'`.
6. **Groups** — scoped keys.

Each step is independently shippable and independently verifiable — do NOT do them in one commit.

## Risks / things to watch
- `refetchType:'none'` relies on `refetchOnMount: true` (global default) to refresh on next
  mount. Confirm no discovery hook overrides `refetchOnMount: false` (none found in audit).
- A surface that is **mounted but NOT optimistic** and must update live (rare) still needs
  `type:'active'`. Identify these per-flow before switching them to `'none'`.
- Realtime socket keys are **server-controlled** ([socket.ts:132](../src/services/realtime/socket.ts))
  — client-side convention can't scope those; coordinate with the backend push payload if the
  socket fan-out is itself too broad.
- Verify behavior, don't just typecheck — drive each mutation and watch the network panel for the
  before/after refetch count.
