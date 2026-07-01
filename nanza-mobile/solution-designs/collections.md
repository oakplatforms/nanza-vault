---
tags: [nanza-mobile, solution-design, collections]
---

# Collections — Solution Design

## Overview

A **Collection** is a user-owned list of trading cards (a `List` of `type === 'COLLECTION'`)
with per-entity quantities. Unlike listings (things for sale) or bids (things wanted),
a collection is a personal inventory — "what I own" — carrying a live **portfolio value**
(`entityListValue`), an **item count** (`entityListCount`), a **privacy** flag, and
optional **thumbnail** / **banner** imagery. Collections are the home of the "Collect"
experience: search-to-add, scan-to-add, and per-card quantity editing.

The feature area spans two places in the app:

- **The user's own Collections tab** (on `ProfileScreen`) — the primary surface for
  building and browsing your collections.
- **A read-only view of another user's public collections** (on `UserProfileScreen`) —
  browse and search, but never edit.

Collections are also **shareable** via the universal reference-code system (a `C` code,
resolved to a mosaic OG card — see [[sharing|Sharing]]).

## How it works

### Where collections live: items-first, off the Profile

The Collect workflow was deliberately **moved out of the Trade tab and onto the Profile
Collections tab**. The Trade screen no longer renders a `Collect` tab at all. Instead:

- The Profile Collections tab shows a **count header** ("N Collections") over a paginated
  (`useInfiniteQuery`) grid of **`CollectionTargetCard`s**.
- Tapping a card opens **`CollectionDetailScreen({ listId })`**, which renders the existing
  `CollectTab` scoped to that one collection via a `lockedCollection` prop. Locked mode
  forces the single-collection endpoint (`listService.getEntities(listId)`), hides the
  multi-collection switcher, and ignores `CollectionSelectionContext`.

This makes the **items screen the primary view** — you land in a collection's contents, not
a settings form. The same items-first pattern was later mirrored for Lots (see [[bulk|Bulk / Lots]]).

### The `CollectionTargetCard`

The collection card mirrors the **OG collection share graphic**: a full-bleed mosaic of the
collection's card images (`entityList[].entity.image` via `getImage`) with a dark bottom
gradient, a **privacy pill** top-left (renders "Private" only when `isPrivate`; public shows
nothing), an **item count** top-right, and a connected info bar below carrying the
**collection name** and the **fair-market value** (`entityListValue`, rendered with
`PriceText` so the "$" uses Figtree per P0 rule #6 — same number shown as "Portfolio Value"
on the profile, no recomputation).

Mosaic sizing uses a **fixed cell size, floating column count**: the cell (~52×73, gap 3)
stays constant and `columns = floor(cardWidth / (cellWidth + gap))` is computed from the
measured card width via `onLayout`, so a wider card shows more columns rather than bigger
cells. This same card is reused as the picker row in the collection-scan "choose collections"
step and in the read-only user-profile grid.

### Adding cards: the single-vs-multi rule

How you add a card to collections depends on **how many collections you own** — a recurring
rule in this feature:

- **1 collection** → keep the inline UX: quantity stepper + "Update collection" header and a
  "Remove from {name}" row directly in the entity action sheet. No extra screen.
- **2+ collections** → the action sheet instead shows a single **"Add to Collection"** action
  that opens a dedicated **`AddToCollectionScreen`**. There, the entity is fixed and each row
  is one of your collections with a `+`/`−` stepper; you set how many of *this* card go into
  *each* collection, then commit **one batch** (`entityListService.batchUpdateByEntity` →
  `POST /entity-list/batch-by-entity`, sending only changed rows; `quantity: 0` removes).
  Quantity is the single source of truth (0 = not in it, `> 0` = highlighted outline), so no
  checkbox is needed.

`EntityActionModal` branches on `collections.length` to pick which path to render. **Share
stays present in both branches.**

### Scanning into a collection

Collections reuse the **same scan camera** as listings, differentiated by a
`mode: 'listing' | 'collection'` route param threaded through the shared scan screens. In
collection mode the scan item's `type` is set to `COLLECTION`. The collection scan flow has
one extra step the listing flow doesn't, and the order matters:

```
Scan (mode=collection)
  → CollectScanTargetScreen   (multi-select which collections to add to; single is preselected)
  → CollectScanItemsScreen    (set qty per scanned card; default 0, ≥1 highlighted) → commit
```

- **Collection-select is first** and writes nothing — it only records the target `listIds`.
  Its rows render `CollectionTargetCard`.
- **Quantities are second** and run the commit: the chosen quantity is added to **every**
  selected collection (quantity can't vary per collection). The commit is
  `scanItemService.batchCommitCollection({ accountId, scanItemIds, listIds })` →
  `POST /scan-items/batch/commit-collection`, which deposits each card's quantity into every
  selected list and deletes the scans, then invalidates the collection query keys.
- **Add, not set**: scanning a card already in a collection **increments** the existing count.
- Listing scans and collection scans keep **separate pools** (scoped by `?type=` on
  `/scan-items`), so pending listing scans never bleed into a collection scan. See
  [[scan|Scan & Camera]] for the cap details.

### Creating and editing collection metadata

`CreateCollection` uses an **inline bordered-row** form (the same layout as Create/Edit Bid):
a **Title** row, a **Thumbnail** row and a **Banner** row (each "Add thumbnail"/"Add banner"
with preview + remove), and a **Visibility** row. The two image rows map to the `thumbnail`
and `banner` fields (the old single `logo` row was dropped from the UI). `Group`'s create form
was later rebuilt to match this same inline style (Title, Thumbnail, Banner, taller
Description; no visibility since groups are always private).

The **"Add to collection" affordance** is a standard full-width `Button` (plus icon + label,
`variant="secondary"`, `size="xxl"`, `fullWidth`) placed at the **end of the collections list
on the own profile only** — it scrolls with content and navigates to `CreateCollection`. The
old header `+` circle button was removed. The read-only user profile gets **no** add button
(you can't add to someone else's collections).

### Read-only parity on another user's profile

`UserProfileScreen` shows the **same `CollectionTargetCard` grid** (driven by `accountId`,
public collections only). Tapping a card opens **`UserCollectionDetailScreen`** — a read-only
detail: a header, a **debounced search input** (server-side, via
`listService.getCollection(accountId, '?search=…&page=…')` — the same endpoint the
propose-trade flow uses), and a read-only `EntityCardList` (`gestures={false}`). There is **no**
add/edit/save/delete and no edit menu; **share is allowed** because these collections are
public. Deliberately **not** built by threading a `readOnly` flag through the editable
`CollectTab` — a thin dedicated screen keeps the live editable flow untouched.

### Sharing a collection

Both the own-profile `CollectionDetailScreen` and the read-only `UserCollectionDetailScreen`
expose **"Share collection"** through the ellipsis menu. The read-only share is gated on the
collection having a `referenceCode`. See [[sharing|Sharing]] for the reference-code scheme.

## Key decisions & rationale

- **Collect moved to Profile, items-first.** The Collect experience belongs to "my stuff,"
  not the Trade tab. Landing directly in a collection's items (rather than a form) matches how
  people think about their inventory, and set the precedent later reused for Lots.
- **Add-to-Collection as a screen, not a bottom sheet.** A scrollable multi-collection
  quantity editor is genuinely a screen's worth of interaction; cramming it into a sheet was
  rejected. This reintroduced the previously-deleted screen intentionally, accepting the
  navigation registration cost.
- **Quantity is the only state (no checkboxes).** `0` = not in the collection, `> 0` =
  in it and highlighted. One mental model, reused across the add screen, the collect steppers,
  and the scan quantities step — matching `TradeCardSelectionRow`.
- **One batch write, not per-collection loops.** All collection quantity commits go through a
  single batch endpoint (`batch-by-entity` for the add screen, `commit-collection` for scans),
  keeping the client simple and the writes atomic.
- **Scan quantity persists server-side.** Each scanned card's quantity is written onto its
  scan item as you step it, then read at commit — so the commit body stays small and survives
  app backgrounding.
- **Separate `CollectionTargetCard` rather than generalizing `ShareGroupCard`.** The group
  card is a single banner; the collection card is a multi-image mosaic with a different info
  bar. A shared abstraction would have been conditional-heavy, so they stay distinct.
- **Read-only detail is a dedicated screen, not a `readOnly` prop on `CollectTab`.**
  `CollectTab` is large and stateful; threading a flag through it risked the working editable
  flow. A thin read-only screen reuses the proven server-side collection-search endpoint.
- **`List.thumbnail` was backend-gated.** Adding the thumbnail field required a
  `@oakplatforms/types` bump before the mobile UI could type it — no custom DTO stand-in was
  allowed. Group's thumbnail/banner shipped first because its types already existed.

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
- [[scan|Scan & Camera]] — the shared scan camera and per-type scan pools
- [[sharing|Sharing]] — the reference-code scheme and collection share card
- [[bulk|Bulk / Lots]] — the same items-first pattern applied to Lots
- [[profile|Profile]] — read-only user-profile parity
