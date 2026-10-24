---
tags: [nanza-api, solution-design, saved-items, posts]
---

# Saved items → post likes only

## Overview

**Saving listings, bids and lots is retired (2026-08-29).** A "favorites" feature will replace it
in a later release with its own architecture. The one thing riding on `SavedItem` — **post
likes** — moved to its own `PostLike` model (2026-08-30), and `SavedItem` itself is dropped.

## How it works (2026-08-30 — `SavedItem` is gone)

- **Schema** — the `SavedItem` model and table are deleted. Post likes moved to their own
  **`PostLike`** model (see [[posts|Posts]] § Likes). Migration `20260830090000_post_likes`
  creates `PostLike`, copies every `SavedItem` row that had a `postId`, and drops `SavedItem` —
  the listing / bid / lot rows simply go with the table. It is written to tolerate a database
  where `SavedItem` was already dropped by hand (guarded copy, `IF EXISTS` / `IF NOT EXISTS`
  throughout) after the dev deploy failed on exactly that (2026-08-30). `Listing`, `Bid`, `BulkListing`,
  `Account` and `Post` lost their `savedItems` relations (`Account.postLikes`, `Post.likes`
  replace the last two).
- **Routes stay as a compatibility shim** (`routers/savedItem.ts`, over `PostLike`) so shipped
  builds keep working, on the retired-feature contract the trading and comments removals set:
  - `GET /saved-items?type=post` — the account's liked posts (group-visibility filtered, rows
    shaped like the old SavedItem). Any other `type`, including the unfiltered call old builds
    make for the profile's Saved section, returns an **empty page** rather than an error,
    because a profile hits it on every visit.
  - `GET /saved-items/ids` — a **static** `{ listingIds: [], bidIds: [], bulkListingIds: [],
    postIds: [] }`, no query at all (Skylar, 2026-08-30): the new build never calls it, and old
    builds only used it to fill hearts — cards are retired and posts carry `viewerHasLiked`.
  - `POST /saved-item` and `POST /saved-item/toggle` — post targets like / toggle a `PostLike`
    (`toggleLike`, the same helper `POST /post/:id/like` uses); a listing, bid or lot target
    answers **410 Gone** `{ errorMessage: 'Saving listings, bids and lots is no longer
    available. Please update the app.' }`.
  - `DELETE /saved-item/:accountId/:id` removes a like by row id.
  - Remove the shim once every build in the field is on `POST /post/:id/like`.
- **Home feed** (`routers/feed.ts`) no longer reads saved items; `isSaved: false` stays on
  every feed item so shipped builds keep parsing.

## Key decisions

- **Own `PostLike` model, not a shrunken `SavedItem`** — decided by Skylar 2026-08-30 (the
  first cut kept the table for posts). A like is a per-post fact; modelling it as a saved set
  forced an account-wide id fetch on app load. Favorites, when it comes, is a separate design.
- **Empty page, not 410, on the list reads** — 0.75. Old builds call `GET /saved-items` from
  the profile on every visit; a 410 there would surface an error repeatedly, while the write
  paths for cards get the honest 410.

## Related

- [[posts|Posts]] — likes ride on the same rows
- [[offer-engine|Offer Engine & Trade Removal]] — the 410 retired-feature contract
- [[../../nanza-mobile/solution-designs/profile|nanza-mobile Profile]] — the Saved section that went
