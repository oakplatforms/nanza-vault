---
tags: [nanza-mobile, solution-design, profile]
---

# Profile — Solution Design

## Overview

There are two profile surfaces: **`ProfileScreen`** (your own profile, editable) and
**`UserProfileScreen`** (viewing someone else, read-only). Both present the same three tabs —
**Listings**, **Bids**, and **Collections** — over a paginated feed. The design goal is
**read-only parity**: a visited profile should look and feel like your own, minus any editing
affordances, so browsing another user is a first-class experience rather than a stripped-down
one.

## How it works

### Listings & Bids tabs — count + share header

On your own profile, each of the Listings and Bids tabs renders a **`ProfileSectionHeader`**: a
count label ("X Listings" / "X Bids") plus a **share-2** action. The user profile renders the
**same header** above each list, with:

- **Counts** computed from the already-fetched page totals (`sellFeedData` / `bidsData`) — the
  same expression the own-profile tabs use, no extra fetch.
- **Share** targeting the **viewed user's** `profile.referenceCode` via
  `buildShareUrl(code, 'sell' | 'bid')` → `guardedShare`. The share is guarded when the code is
  absent (the user-profile DTO must expose `profile.referenceCode`, same field the own profile
  reads from `currentUser.account.profile.referenceCode`). See [[sharing|Sharing]] for the
  profile-code (`U`) scheme.

### Collections tab — same card grid, read-only detail

Both profiles show the same **`CollectionTargetCard` grid** (the mosaic collection card). On the
own profile, tapping a card opens the editable `CollectionDetailScreen`; on a user profile it
opens a **read-only `UserCollectionDetailScreen`**:

- A header, a **debounced search input** (250ms), and a read-only `EntityCardList`
  (`gestures={false}`).
- Search is **server-side**, reusing the propose-trade collection-search endpoint
  (`GET /list/collection/{accountId}?search=…&page=…`) — the same endpoint `UserCollection`
  already calls, just with `&search=` added, so it handles collections with more than 20 items.
- **No** plus-circle/add, no edit, no save, no delete, and no edit-menu actions. **Share** is
  allowed (these are public collections).

The read-only detail is a **thin dedicated screen**, deliberately not built by threading a
`readOnly` flag through the large, stateful editable `CollectTab` — that would risk the working
own-profile flow. See [[collections|Collections]] for the full collection design.

### Header spacing

The count-header's bottom spacing was bumped to **18px** via a new **`theme.spacing.smd = 18`**
token (there was no 18 between `md`=14 and `lg`=24), keeping it a named token rather than a
hardcoded size, and repointing `collectionCountHeaderTight.marginBottom` at it.

## Key decisions & rationale

- **Reuse, don't fork.** Parity is achieved by reusing existing pieces — `ProfileSectionHeader`,
  `CollectionTargetCard`, `EntityCardList`, `buildShareUrl`/`guardedShare` — with only two thin
  new files (the user-profile collection grid variant and the read-only detail screen).
- **Read-only detail as its own screen.** Cleaner separation and zero risk to the editable Collect
  flow; reuses a proven server-side search endpoint instead of inventing client-side filtering.
- **Server-side collection search.** Handles collections larger than one page (>20 items),
  debounced and paginated — the same behavior the trade flow already relies on.
- **A named `smd` token, not an inline 18.** Honors the no-hardcoded-sizes rule; the token is
  additive to `dark.json` and changes no existing value.
- **Guard the share when the code is missing.** The user-profile fetch must surface
  `profile.referenceCode`; if it's absent for a given profile, the share action is hidden rather
  than shipping a broken link.

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
- [[collections|Collections]] — the collection grid + read-only detail in depth
- [[sharing|Sharing]] — the profile (`U`) reference code and share cards
