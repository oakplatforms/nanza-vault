---
tags: [nanza-mobile, nanza-api, solution-design, plan, scan, collections]
---

# Scan Quick-Add: tap a match, set how many, it's in your collection

**Status:** **plan, written 2026-10-23 (Skylar, voice), architect-reviewed the same day and the
review folded in. API + mobile built 2026-10-23 (both uncommitted, tsc + lint clean, not
deployed, pending device QA). No migration, no types bump. Still to do by hand: delete the seven
dead mobile files listed under "Built (mobile)", and run the one-off COLLECTION scan-item
delete in DBeaver on dev and prod.**
Spans nanza-mobile (camera result UI, a new swipeable quantity modal, entry points) and
nanza-api (the scan cap, one additive add-to-collection endpoint, owned quantities on the scan
response). When it lands, fold the result into [[scan|Scan & Camera]] and the scan section of
[[collections|Collections]], then delete this file.

## Why

Scanning is in the hands of testers (AFFILIATE + VERIFIED). Right now a collection scan takes
two steps: pick a match and press **Add**, then a separate review screen
(`CollectScanItemsScreen`) to set quantities and commit. We want testers to go from scanning a
card to having it in their collection as quickly as possible. We also want to limit how many
paid vision calls each tester can make until premium scan packs exist. Listings and bids are
being redesigned as their own project, so the listing camera comes out for now.

## Decisions

Skylar, 2026-10-23 (voice):

1. **Scan cap: 300, one-time, lifetime.** Replaces the current lifetime cap of 500. It never
   resets. Paid / premium scan counts come later.
2. **No camera on listings for now.** Remove the scan entry from the Sell tab search row and
   from the lot builder. Bids never had one.
3. **Collection scan result = match images only.** Remove the top `VariantDetail` highlight
   and the **Add** button. The bottom strip of possible matches gets **bigger** tiles.
4. **Tap a match → quantity modal.** Large card, a −/count/+ stepper that starts at **1**, an
   **Add to collection** button. Swipe left/right inside the modal through the other matches.
   Closing the modal goes back to the strip.
5. **Add commits immediately**, then the modal closes and the **camera stays open** for the
   next card. No review screen, no pending pool for collections.
6. **"You have N"** shows in the modal when the match is already in that collection.
7. **Price is in (2026-10-24).** `SHOW_FAIR_MARKET` was `false` at build and flipped to `true`
   on 2026-10-24 (label now "Market price"). The match tile draws amount + "Market price"
   under the name (`scanVariantCardPriceRow`, baseline-aligned); the modal page draws the
   standard `FairMarketPill` under its subtitle (`scanAddCardPill`, centred). En dash when
   unpriced.

From the architect review (Fable, 2026-10-23), adopted by Skylar:

8. **Collection mode skips the 20-item scan pool check**, and existing COLLECTION scan items are
   deleted by a one-off SQL, so nobody is stranded at the pool limit with no screen to clear it.
9. **Unreachable listing-scan mobile screens are deleted**, not left dormant. Git keeps them;
   the listings/bids redesign rebuilds that UX anyway. The API scan-item endpoints stay.
10. **Collection scanning no longer requires a seller profile.** `/scan/cards` validates
    `'authenticated'` in COLLECTION mode (still `'seller'` in LISTING mode). Scanning into a
    collection isn't a selling action, and the AFFILIATE/VERIFIED gate still applies.
11. **`ScanAddModal` is its own component**; the stepper is extracted into a shared
    `QuantityStepper` used by both it and `QuantityModal`.

## How it works

### API (nanza-api)

**Cap.** `DEFAULT_USAGE_CAPS.scan` goes from 500 to **300** in `src/constants/usageCaps.ts`.
Enforcement is unchanged: the cap is checked before the vision call and `Account.scanCount`
is incremented after a successful one. Accounts already past 300 get `403 CAP_REACHED` on their
next scan — intended. Two parallel scans at 299 can both succeed; that's an acceptable
freemium race and not worth wrapping a paid external call in a transaction. There is no
`CAP_SCAN` override in `serverless.yml`, `buildspec.yml` or `nanza-pipelines`; only SSM could
set one, so confirm in the console that 300 is live after deploy.

**`POST /scan/cards`, COLLECTION mode.**
- `validateAccount(..., mode === 'COLLECTION' ? 'authenticated' : 'seller')`.
  `validateScanAccess` (AFFILIATE / VERIFIED) still runs for both modes.
- Skip `enforceScanItemLimit` in COLLECTION mode. It exists only for the scan-item workflow,
  which collections no longer use. LISTING mode keeps it.
- New optional body field **`listId`**. When present: verify the list belongs to `accountId`
  **and** is `type: COLLECTION` (otherwise 403 — without this check any scanner could probe
  what another user owns). After the match list is built, one query —
  `entityList.findMany({ where: { listId, entityId: { in: matchEntityIds } }, select: { entityId, quantity } })`
  — and each `productMatch` gains **`ownedQuantity`** (`quantity ?? 1` for a row, `0` if
  absent). No per-match queries.

**Commit = SET, not add (Skylar, 2026-10-23, revised the same afternoon).** The modal opens on
the count you already hold (`ownedQuantity`, or 1 for a card not in the list), so what you see
is what gets saved. The existing **`POST /entity-list/batch-by-entity`** does exactly that
(creates the row if missing, updates it if present, counts new rows toward the voucher) — the
scan sends one item `{ listId, quantity }`. The stepper never goes below 1: removing a card is
not a scan-screen action. Last-write-wins is accepted (a re-scan reopens on the fresh count).
The additive `POST /entity-list/add-by-entity` built that morning was **removed again** —
no extra endpoint, validator or service function.
- Unchanged: `POST /scan-item`, `batch/convert`, `batch/commit-collection` stay for the
  listing flow.

**Collection order (added 2026-10-23):** the collection grid's default sort moves from
`createdAt desc` to **`updatedAt desc`** on both `GET /list/:id/entities` and the account
collection endpoint in `routers/list.ts`, so a quick-add onto a card you already hold (or any
quantity edit) brings it to the front. The raw increment in `addEntityToList` stamps
`updatedAt` itself since it bypasses Prisma's `@updatedAt`. Price sorts are unchanged.

**One-off SQL (DBeaver, dev then prod):** `DELETE FROM "ScanItem" WHERE type = 'COLLECTION'`.

**Types:** no `@oakplatforms/types` bump. That package is generated from the Prisma schema, which
doesn't change; the scan request/response shapes are hand-written in nanza-mobile's
`services/api/Scan.ts`, so `ownedQuantity` and the `addByEntity` call are added there.

**Built (2026-10-23):** `usageCaps.ts` scan → 300; `validation/scan.ts` `validateScanListId`;
`validation/entityList.ts` `parseAddByEntityInput` + `assertOwnedCollectionList` (403);
`services/collection.ts` `addEntityToList` (createMany skipDuplicates → raw COALESCE update);
`routers/scan.ts` (role by mode, pool check LISTING-only, `listId` → `ownedQuantity` on every
`productMatch`); `routers/entityList.ts` `POST /entity-list/add-by-entity` returning the row
plus `created` and `maxQuantity`.

### Mobile (nanza-mobile)

- **Entry points.** Remove the camera icon from `TradeScreen` (Sell search row) and from
  `BulkTab` (lot builder). The collection detail camera (`CollectionSearchFooter`) stays.
- **`ScanCameraScreen`** becomes collection-only.
  - Drop the `mode` route param and the listing branch; the screen always has a `listId`.
  - Send `listId` on `/scan/cards`.
  - Result view: remove `VariantDetail`, the Add button and the "Scans (N)" header chip. The
    match strip gets bigger tiles (about 120px wide, up from 86px; strip slot sized to fit).
    Drop the "Picked" badge, keep `#N`.
  - Tapping a tile opens `ScanAddModal` at that index.
- **New `QuantityStepper`** (`components/global/`): the −/count/+ control lifted out of
  `QuantityModal`, which then uses it too.
- **New `ScanAddModal`** (`components/modals/`), built on `Modal` + `ProductCardList` +
  `QuantityStepper`, not on `QuantityModal`.
  - A horizontal paging list of the scan's matches. Each page: card image, name,
    number • set, **"You have N"** when `ownedQuantity > 0`, and the market-price pill.
  - One stepper (min 1, max 99) that **resets to 1 when you swipe to another page**.
  - **Add** (or **Update** for a card already held) →
    `entityListService.batchUpdateByEntity({ accountId, entityId, items: [{ listId, quantity }] })`.
    The button is disabled while the mutation is in flight. On success: invalidate the
    collection's queries, close the modal, `handleResetScan()` so the camera is live again.
- **Delete:** `CollectScanItemsScreen`, `ScanCollectRow`, the `CollectScanItems` route, and
  the listing-scan screens `ScanItemsScreen`, `ScanItemEditScreen` plus their routes, services
  and types that nothing else imports.
- **Cap UX.** The existing `CAP_REACHED` handling stays. Check that the copy doesn't hard-code
  "500".

**Built (mobile, 2026-10-23):** `ScanCamera` route param is now `{ listId }` only;
`ScanCameraScreen` rewritten collection-only (one 2x capture, `mode: 'COLLECTION'` + `listId`,
tiles-only result view, `#N` badge kept, tap → modal, header camera button still re-arms the
camera); new `components/modals/ScanAddModal` (measured-width `ScrollView` pager; each page
draws the card image itself at `xxl+xxl+xl` wide — the 89pt `EntityLockupRow` the
long-press sheet uses read too small for a preview — with name + number • set under it,
"You have N" row, page dots, the same glass-circle −/count/+ as `QuantityModal`, and an Add
that disables while in flight; `QuantityModal` itself is untouched); new `typography.labelLg`
(label one size up, 14/16) for the card name on tiles and in the sheet; `scan.ts` styles:
tiles `xxl + xl + sm` wide, `scanAdd*` keys; `services/api/Scan.ts` gains
`listId` / `ownedQuantity`; `services/api/EntityList.ts` gains `addByEntity`; the Sell-tab and
lot-builder cameras are gone, and the lot builder's scan-deferred branch
(`initialScanItemIds`, `batchConvert`) with them. Dead files **deleted 2026-10-23** (Skylar approved by voice;
nothing imported them, tsc 0 errors after): `src/screens/trade/ScanItemsScreen/`, `ScanItemEditScreen/`,
`CollectScanItemsScreen/`, `ScanReviewScreen/`, `src/components/collections/ScanCollectRow/`,
`src/screens/trade/data/scanItems.ts`, `src/services/api/ScanItem.ts`. `styles/components/scanItems.ts`
stays (collections use it).

## Out of scope / later

- Premium / paid scan packs that raise the 300 cap.
- The listings + bids redesign, including whether scanning comes back to listings.
- Scanning several cards in one shot.

## Build order

1. API: cap to 300; `/scan/cards` COLLECTION changes (`'authenticated'`, skip pool check,
   `listId` → `ownedQuantity`); `POST /entity-list/add-by-entity`; types bump. No migration.
   Run the one-off COLLECTION scan-item delete on dev.
2. Mobile: remove the entry points → camera result changes → `QuantityStepper` + `ScanAddModal`
   → delete the review and listing-scan screens → `types:update`.
3. Deploy API to dev, QA on device, then prod (run the one-off delete there too).
4. Vault: fold into [[scan]] + [[collections]] (fix the stale 50/24h limit and the
   `CollectScanTargetScreen` mention while there), then delete this plan.
