---
tags: [nanza-api, solution-design, feeds]
---

# Account feeds — `/sell-feed` and `/trade-feed`

## Overview

Two endpoints in `src/routers/bulkListing.ts` serve one account's tradeable items as a
single newest-first, paginated list instead of the client stitching `/listings`, `/bulk-listings`
and `/bids` pages together (which double-showed bulked children and sorted across
un-aligned pages):

| Endpoint | Kinds | Scope | Consumer |
|---|---|---|---|
| `GET /sell-feed?accountId=&page=&limit=` | ACTIVE listings (un-bulked) + PUBLISHED bulks | owner-only surfaces, no viewer filter | mobile Trade screen "My listings", post composer ELB picker |
| `GET /trade-feed?accountId=&page=&limit=` | the above **plus ACTIVE bids** | viewer-filtered | the profile's **Trades** rail + its See-all grid (2026-08-26) |

Each item is a discriminated union — `{ kind: 'listing', listing } | { kind: 'bulk', bulk } |
{ kind: 'bid', bid }` — and the response is `{ data, page, limit, total }` with `total` the sum
across kinds, so the client's "See all" gate and `getNextPageParam` work unchanged.

## How it works

Both handlers call one page builder, `buildAccountFeedPage({ listingWhere, bulkWhere,
bidWhere?, page, limit })`:

1. **Heads, not rows.** For each kind it runs a `count` plus a `findMany` selecting only
   `id, createdAt` ordered `createdAt DESC, id DESC`, taking `(page + 1) * limit` — the page
   window's upper bound. Over-fetching ids is cheap and makes the merge exact.
2. **Merge-sort + slice.** The head streams are concatenated, sorted by `createdAt`, and the
   `[page*limit, (page+1)*limit)` window is sliced.
3. **Hydrate the slice only.** Listings with `sellListingInclude`, bulks with the sold-aware
   `detailChildInclude` → `withDetailData` (children marked `isSold`, totals over available
   children), bids with `feedBidInclude` (the same entity/account shape + `conditions`). Comment
   counts ride along via `getPostCountMap` / `withPostCount`, mirroring the home feed.
4. **Re-project** in merged order, dropping rows that vanished mid-request.

`/sell-feed` passes the owner wheres and no `bidWhere`. `/trade-feed` resolves the requester
and, when they are **not** the account owner, ANDs the list endpoints' visibility rules onto
each kind: `buildListingVisibilityFilter` + `buildPublicAuthorListingFilter` for listings,
`buildBidVisibilityFilter` + `buildPublicAuthorBidFilter` for bids, and for bulks (which have no
shared helper) `isPublic OR groups.some(groupId in viewer's ACTIVE groups)` via
`getActiveGroupIds`. The owner sees everything of theirs, including group-only trades.

## Key decisions & rationale

- **One builder, two routes** rather than a `?includeBids` flag: the sell feed is an
  owner-only surface with no viewer filter and must stay that way (it feeds the seller's own
  tooling); the trade feed is a public surface. Separate routes keep the visibility contract
  obvious at the call site.
- **Viewer filtering on `/trade-feed` even though `/sell-feed` has none.** The profile rail is
  visible to anyone, so it inherits exactly the rules `/listings` and `/bids` already enforce;
  the earlier profile preview (which called `/sell-feed`) leaked private/group listings onto
  public profiles.
- **Recency interleave, not the home feed's 4:4 pattern.** A profile is a chronological
  ledger of what one person is trading; diversification only matters across sellers.
- **No schema change, no types-package bump.** The item union is a composition of existing
  DTOs, typed client-side (`SellFeedItem` / `TradeFeedItem` in mobile's `types/index.ts`).

## Related

- [[../INDEX|nanza-api]]
- [[../../nanza-mobile/solution-designs/profile|nanza-mobile Profile]] — the Trades rail that consumes `/trade-feed`
- [[../../nanza-mobile/solution-designs/groups|nanza-mobile Groups]] — `/group/:id/items`, the same mixed-rail idea scoped to a group
