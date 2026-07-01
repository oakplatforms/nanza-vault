---
tags: [nanza-mobile, solution-design, scan]
---

# Scan & Camera — Solution Design

## Overview

Scanning lets a user point the camera at a physical card, have it identified server-side,
and turn the result into either a **listing** (something to sell) or a **collection entry**
(something they own). Both flows share one **scan camera** and one server-side identification
call; the difference is carried by a **mode** and materializes at the terminal commit step.

A **scan item** (`ScanItem`) is the intermediate record between "I scanned a card" and "I
committed it." Each scan item has a `type` of `LISTING` or `COLLECTION`. Scan items are held
in a pending pool until the user reviews and commits them.

## How it works

### One camera, two modes

The scan camera (`ScanCameraScreen`) and the card-identification endpoint (`POST /scan/cards`)
are shared between listing and collection scanning. Intent is threaded explicitly as a
**`mode: 'listing' | 'collection'`** route param (default `'listing'`). On add, the camera
sets the scan item's `type` accordingly (`LISTING` / `COLLECTION`) and routes to the matching
review screen:

- **listing** → `ScanItemsScreen` (price/condition/group-readiness review, Sell/Lot footer).
- **collection** → the collection quantity flow (`CollectScanTargetScreen` →
  `CollectScanItemsScreen`), where the only per-item field is **how many** (see [[collections|Collections]]).

Explicit mode was chosen over inferring it from navigation history or stashing it in context —
implicit routing bleeds across unrelated navigations, whereas an explicit param matches how the
rest of the app passes intent.

### Entry points

- **Sell tab**: the scan entry point is a **camera icon** in the search row →
  `ScanCamera({ mode: 'listing' })`. The bottom button is **"Create Lot"** → the lot builder.
  (These two traded places: the camera used to be the bottom button and Create-Lot the search
  icon.)
- **Collect tab / collection detail**: a **camera icon** next to the filter icon in the search
  row → `ScanCamera({ mode: 'collection' })`. Shown regardless of how many collections exist.

### Separate scan pools (per type)

Listing scans and collection scans are **independent pools**. The list endpoint filters by
type (`/scan-items?type=…`), the mobile hook is type-scoped (`useScanItems(accountId, type)`),
and the camera's "Scans (N)" badge counts only the current mode. This keeps a pending listing
scan from showing up mid-collection-scan and vice versa.

### The scan cap (and the bug that motivated fixing it)

There are three distinct limits — do not conflate them:

| Constant | Value | Meaning |
|---|---|---|
| `SCAN_ITEM_MAX` | **20 per pool** | Max pending scans held at once |
| `SCAN_ITEM_BATCH_CONVERT_MAX` | **20** | Max scans committed in one batch (listing convert **and** collection commit) |
| `DAILY_SCAN_LIMIT` | 50 | Scan API calls per 24h |

The settled design is **per-type pools**: the holding cap is counted **filtered by `type`**,
so 20 collection scans and 20 listing scans coexist independently. This fixed a real bug where
`enforceScanItemLimit` counted **all** scan items regardless of type — filling the collection
pool to 10 would spuriously block listing scans with "You already have 10 scans," even though
the per-mode badge read "Scans (0)." The cap check now runs **after** `validateScanMode` so the
mode is known, and the error message is type-specific ("…20 collection scans" / "…20 listing
scans"). Raising the pool from 10→20 has **no effect on bulk/lot size** — scans never become
lots directly; a lot is a grouping of existing listings, capped separately at 20 items.

Rejected alternative: one unified pool of 20 (leave the count unfiltered). It's a simpler diff
but keeps the two flows competing for one budget and leaves the per-mode badge misleading —
the model doesn't match the app's own separation.

### Mode-aware dedup (motivating a backend change)

`POST /scan/cards` originally excluded any card the user had an ACTIVE listing for **or** any
existing scan item for, regardless of mode — so a card you'd listed was hidden when scanning
into a collection, and vice versa. The correct behavior is mode-scoped: a **LISTING** scan hides
only cards you have an active listing (or LISTING scan item) for; a **COLLECTION** scan hides
only cards already in the collection / COLLECTION scan items, ignoring listings. This required
passing the mode to `/scan/cards` (the client already knows it) so the endpoint can branch the
exclusion query — a backend change flagged as out of the mobile `src/` scope.

## The Image Viewer (shared component)

The full-screen, pinch-to-zoom, swipeable image viewer (`ImageViewer`) is used by feed cards,
order items, and the share cards. Because it's shared, all changes stay **backwards-compatible**
(single-image callers keep passing one `source`). Three durable behaviors:

- **Uniform image height** — a fixed-ratio centered frame with `resizeMode="cover"` so every
  image fills the same box (uniform even if cropped), instead of `contain` letting height vary
  by aspect ratio.
- **Edge-to-edge backdrop on Android** — the `Modal` uses `statusBarTranslucent` (and
  `navigationBarTranslucent` where supported) so the backdrop reaches the top/bottom on tall
  phones; the close button stays on `insets.top`.
- **Carousel for multi-image items** — the viewer optionally accepts an **array of sources +
  an initial index**, rendering a horizontal paged carousel (each page keeps
  pinch/pan/double-tap) with a page indicator. Lots open a swipeable viewer aligned to the
  carousel order, starting on the tapped slide; single listings/bids stay single. Share cards
  wire the same viewer (see [[sharing|Sharing]]).

## Known caveats

- **Android carousel gesture conflict.** On the scan-results screen, the horizontal
  `ScrollView` competes with react-native-gesture-handler's screen-level swipe-back on Android.
  The fix is to use the `ScrollView` from `react-native-gesture-handler` (already a dependency),
  which coordinates with the screen gesture handler — but it needs on-device confirmation.

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
- [[collections|Collections]] — the collection scan quantity flow and commit
- [[bulk|Bulk / Lots]] — lots are groupings of listings, capped independently of scans
- [[sharing|Sharing]] — share cards wire the same `ImageViewer`
