---
tags: [nanza-mobile, solution-design, bulk]
---

# Bulk / Lots — Solution Design

## Overview

A **Lot** (a `BulkListing`) is a group of the user's existing **Listings** sold together as one
unit. It is not a new kind of card and it is not built from scans — it is a *grouping* of
listings the seller already has (`Listing.bulkListingId`). A lot carries its own details (name,
cover image, where-to-post targeting) and its own child-listing membership. The owner-facing Lot
experience is **items-first**: you manage which listings are in the lot as the primary view, with
the details form demoted to a secondary step. This mirrors the Collections flow.

## How it works

### Two variants of one builder

The lot builder (`BulkTab`) has two variants that both back the create and edit flows:

- **`picker`** — the **items screen**: a search field ("Add to lot"), member rows, a summary,
  and a primary button. This is the primary owner view.
- **`form`** — the **details screen**: Name, Cover image, group/where-to-post targeting, a
  "Lot items (N)" row that pushes the picker, and a Create/Update button.

### Items-first entry

- **Tapping a lot you own** opens the **items screen** (`BulkItemsScreen`, a thin wrapper that
  renders `BulkTab variant="picker"` against the published lot id) — not the details form. Its
  header ellipsis carries **Edit lot / Share lot / Delete lot**:
  - *Edit lot* → the details form (`EditBulk`), which reached this way has **no ellipsis** of
    its own (the actions now live on the items screen).
  - *Share lot* → `guardedShare` with the lot's `K` reference code (see [[sharing|Sharing]]).
  - *Delete lot* → the existing "keep listings vs delete listings" choice sheet.
- **Creating a lot** starts on the **items screen** too (`BulkItems({ create: true })` against a
  fresh draft). The bottom button reads **"Next"** (not "Done") → the details form, whose final
  action creates the lot and returns to the Sell screen (`handlePublish`, unchanged).

Entry rewiring: the owner tap on `FeedBulkCard` / `BulkSaleCard` / `FeedActionModal` and the Sell
tab's "Create Lot" button all route to `BulkItems` (items-first) instead of the old form-first
`EditBulk` / `CreateListings`.

### The draft model

Create mode get-or-creates a **DRAFT** `BulkListing` on mount; items attach to it server-side as
you pick them; publishing flips DRAFT→PUBLISHED. To keep `BulkTab`'s picker draft-agnostic,
`BulkItemsScreen` owns creating the draft and passes an `existingBulkId` down. Edit mode uses the
existing published id.

### Scan-deferred create stays form-first

The **scan-deferred** create path (arriving with `initialScanItemIds`, i.e. converting scans into
a lot) **keeps form-first** — there are no items to pick before conversion, so the details form
(which hides the items row in this case) is the right landing.

### Lot item cap

A lot is capped at **20 child listings**, enforced server-side on both create
(`POST /bulk-listing`) and update (`PUT /bulk-listing/:id`, checking the *resulting* total so a
draft can't grow past 20 either). Previously lots had only a **minimum** of 2 to publish and no
maximum. The cap is server-enforced (surfaces as an alert); disabling the add button at 20 in the
builder is optional polish, not required for correctness. This 20 is **independent of the scan
pool cap** — lots are groupings of listings, not scans (see [[scan|Scan & Camera]]).

### Buyer flow — buy-all semantics, line-total pricing, straight to cart

The buyer's view of a lot is the **full-screen lot detail** (`ListingScreen`, registered as the
`ListingDetail` route — it renders both single listings and lots via `bulkListingId`), plus the
compact `FeedBulkCard` in feeds and `ShareBulkCard` on share landings. All three share one swipe
model: a **cover** slide (the whole lot) followed by one slide per child listing.

Three rules govern buying a lot, and every surface obeys them identically:

- **Lot children are buy-all.** A child listing in a lot always has
  `multiTransactionsEnabled = false` (forced off at create — bulk children can't be partially
  purchased). So a child with `quantity = 2` is added to the cart as **`quantityInOrder = 2`**, and
  its buy label reads **"Buy 2 for …"**. The add-to-cart quantity is taken straight from
  `listing.quantity` (never collapsed to 1, and the old `multiTransactionsEnabled ? 1 : quantity`
  branch is gone — for a lot child the flag is invariantly false, so keying off it was fragile
  against mis-set data).
- **Prices are line totals.** `Listing.price` is the **per-unit** price. Every lot buy label shows
  **unit price × quantity** (a qty-2 card at $0.50 reads "Buy 2 for **$1.00**", not $0.50). Lot
  totals come from `getBulkTotalPrice` (`utils/bulkTotals.ts`), which sums `price × quantity`
  across children; per-child labels multiply inline.
- **Every buy goes straight to the cart.** Tapping buy on the **cover** batches every available
  child into one same-seller order (`useShareActions.onBuyBulk`) and navigates to `ReviewCart`.
  Tapping buy on a **single child** adds that card (its full quantity) and also navigates to
  `ReviewCart`. There is no "review then confirm on the feed" — the feed card used to stay put; it
  now routes to the cart like the detail and share surfaces.

**"N items" counts total quantity, not listing count.** The lot's item metric is
`Σ child.quantity` (3 cards of qty 2 → "6 items"), falling back to a summed count when the server
doesn't send `totalQuantity` — never the raw `listings.length`.

**The old select-which-ones "Buy Now" screen (`BulkDetailScreen` / the `BulkDetail` route) was
deleted.** With buy-all semantics the cover simply buys the whole lot, so a per-card selection step
was redundant. Its shared data hook (`BulkDetailScreen/data/fetchBulkListing` → `useBulkListing`)
stays — it's imported by `ListingScreen`, `BulkItemsScreen`, `BulkTab`, and `CreateListingScreen`;
only the screen component and its route were removed.

## Key decisions & rationale

- **Items-first, mirroring Collections.** People manage a lot by its contents, not by its
  metadata form. Reusing the two existing `BulkTab` variants and just rewiring entry points +
  transitions kept the change low-risk versus rebuilding screens.
- **"Next" vs "Done" as the mode signal.** Create needs to advance to the details form to
  actually publish; edit's items screen is terminal (Done closes back). The button label encodes
  which flow you're in.
- **Draft ownership pulled up to the wrapper.** Having `BulkItemsScreen` create the draft and pass
  `existingBulkId` keeps `BulkTab`'s picker draft-agnostic and avoids duplicating get-or-create
  logic across the two variants.
- **Scan-deferred create kept form-first.** No items exist pre-conversion, so items-first would
  land on an empty picker — the form is the correct entry there.
- **20-item cap enforced on the resulting total, not just at publish.** Checking existing + added
  − removed − deleted on every update prevents a draft from ballooning past the cap between edits.

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
- [[collections|Collections]] — the items-first pattern this mirrors
- [[scan|Scan & Camera]] — why the lot cap is separate from the scan cap
- [[sharing|Sharing]] — the Lot (`K`) share card and share action
