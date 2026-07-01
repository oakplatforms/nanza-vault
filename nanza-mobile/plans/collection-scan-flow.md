# Collection Scan Flow — Solution Design

Add a "scan into collection" flow that mirrors the existing listing scan flow, but on the
review step the user assigns a **quantity** per scanned entity instead of price/condition/etc.
Tapping save adds those entities (with quantities) to the user's active collection.

## Goal (from voice)

Core flow:
- Scanning reuses the **same scan camera** we use for listings.
- After scanning, multi-select works like listings, but the per-item review only asks
  **how many** of each entity to add (qty 2, 1, 3, …).
- Tapping **save/update** adds those entities at their chosen quantities to the collection.
- Conceptually: same as a listing scan, but the scan item `type` is `COLLECTION` (a "list"),
  and the terminal action saves entities to a collection instead of creating listings.

### UI placement (revised 2026-06-20)

**Sell tab** — swap the bottom button and the scan entry point:
- Bottom button: currently a **"Scan"** button (`renderScanSection`, with the caption
  "or scan to create a listing"). Change it to a **"Create Lot"** button (white button,
  `tradeScanButton` style) → navigates to `CreateListings`.
- **Remove** the "or scan to create a listing" caption (`tradeScanCaption`).
- Scan entry point: becomes a **camera icon** in the search row. Today that search-row icon is
  the `collection-2` icon that opens Create Lot (`navigation.navigate('CreateListings')`,
  `index.tsx:343–350`). Replace it with a **camera icon** → `ScanCamera` (`mode: 'listing'`).
  Net effect: the Create-Lot and Scan entry points trade places.

**Collect tab** — keep the bottom **Save** button exactly as-is (no change). Add a **camera
icon next to the filter icon** in the search row → `ScanCamera` (`mode: 'collection'`). This is
the scan-into-collection entry point.
- Note the filter icon only renders when `hasMultipleCollections`; the camera icon should show
  regardless (it's the primary scan entry). Place the camera icon in the same right-cluster of
  the search row, beside where the filter icon appears.

## What already exists (reuse, don't rebuild)

| Piece | File | Notes |
|---|---|---|
| Scan camera + variant picker | `src/screens/trade/ScanCameraScreen/index.tsx` | Hardcodes `type: 'LISTING'`, navigates to `ScanItems`. Shared with the Sell flow. |
| Scan item review list | `src/screens/trade/ScanItemsScreen/index.tsx` | Listing-oriented (price/condition/group readiness, Sell/Lot footer). |
| Per-item edit | `src/screens/trade/ScanItemEditScreen/index.tsx` | Listing fields. |
| Scan item service + `type` field | `src/services/api/ScanItem.ts` | `CreateScanItemPayload.type?: 'LISTING' \| 'COLLECTION'` already exists. |
| Scan item hooks | `src/screens/trade/data/scanItems.ts` | `useScanItems` lists **all** account scan items (no type filter today). |
| Collection write endpoint | `src/services/api/EntityList.ts` → `batchUpdate` | `POST /entity-list/batch` with `{ listId, accountId, items: [{entityId, quantity}] }`. **This is exactly what the Collect Save button uses.** |
| Active collection context | `src/contexts/CollectionSelectionContext.tsx` | `selectedCollectionId`; primary fallback. |
| Collect tab (Save button) | `src/screens/trade/TradeScreen/CollectTab.tsx` | `handleSave` already calls `entityListService.batchUpdate`. |

**Key reuse:** the save action is already built — `entityListService.batchUpdate`. The collection
scan flow's terminal step is the same call CollectTab's Save already makes. No new collection
endpoint is needed on the frontend.

## The one real design decision: how to carry "mode" through the shared scan screens

The scan camera and review list are shared. We must tell them whether this run is a LISTING
scan or a COLLECTION scan.

**Decision: thread a `mode: 'listing' | 'collection'` route param through the scan screens.**
Confidence: **0.85**

- `ScanCamera` param gains `mode` (default `'listing'`). On Add, it sets `type` accordingly
  in the FormData. For collection mode it navigates to the collection review screen.
- Rejected alt (Confidence 0.55): infer mode from where the user came from / store in context.
  Worse — implicit, and context bleeds across unrelated navigations. Explicit param is clearer
  and matches how the rest of the app passes intent (e.g. `initialScanItemIds`, `initialTab`).

### Scan-item pool isolation (must-handle, Confidence 0.8)

`useScanItems` lists **all** of the account's scan items regardless of `type`. If a user has
pending LISTING scans and starts a COLLECTION scan, the two pools mix. Two options:

1. **Filter client-side by `type`** in each review screen (collection review shows only
   `type === 'COLLECTION'`, Sell review shows only `LISTING`). Smallest change, no API work.
   The header "Scans (N)" count on the camera should also filter by the active mode.
2. Filter server-side via a `type` query param on `/scan-items`. Cleaner but needs an API change.

Recommend **option 1** for the frontend pass; flag option 2 to backend as a follow-up so the
list endpoint can scope by type. **Open question for Skylar / backend:** does `batchConvert`
or any collection-save endpoint already exist server-side for `type: 'COLLECTION'`, or do we
save collection scans purely through `entity-list/batch` on the frontend? (See "Backend" below.)

## Proposed implementation

### 1. Navigation types — `src/types/navigation.ts`
- `ScanCamera: { clearScan?: boolean; mode?: 'listing' | 'collection' } | undefined`
- Add `CollectScanItems: undefined` (new review screen) — register in `AppNavigator`.
  - NOTE: P0 rule #7 says don't touch the navigator. Registering a brand-new screen is additive
    (no reorder, no changes to existing registration). Per memory, nav changes are permitted when
    cleaner — but I'll **confirm with Skylar** before editing `AppNavigator.tsx`.

### 2a. Sell tab — `src/screens/trade/TradeScreen/index.tsx`
- **Search-row icon (line ~343–350):** replace the `collection-2` icon (which opens
  `CreateListings`) with a **camera icon** → `navigation.navigate('ScanCamera', { mode: 'listing' })`.
- **`renderScanSection` (line ~501–519):** change the bottom button from "Scan" to **"Create Lot"**,
  `onPress` → `navigation.navigate('CreateListings')`, keep the white `tradeScanButton` style.
- **Remove** the `tradeScanCaption` text ("or scan to create a listing").

### 2b. Collect tab — `src/screens/trade/TradeScreen/CollectTab.tsx`
- Keep the bottom **Save** button unchanged.
- In the search row's right cluster (`renderSearchRow`), add a **camera icon** beside the
  filter icon → `navigation.navigate('ScanCamera', { mode: 'collection' })`. The camera icon
  shows regardless of `hasMultipleCollections`.
- Styles go in `src/styles/components/layout.ts`. No inline styles (P0 #1/#2).

### 3. Scan camera — `src/screens/trade/ScanCameraScreen/index.tsx`
- Read `route.params?.mode` (default `'listing'`).
- In `handleAdd`, set `formData.append('type', mode === 'collection' ? 'COLLECTION' : 'LISTING')`.
  - For collection mode the listing image is irrelevant; still harmless to send, or skip the
    `file` append. (Minor — confirm whether collection scans want the photo at all.)
- "Scans (N)" chip count and the screen it opens depend on mode:
  - collection → `navigation.navigate('CollectScanItems')`
  - listing → `navigation.navigate('ScanItems')` (unchanged)

### Collection scan flow shape (revised — this is what differs from listing scan)

The collection scan flow has an extra step the listing flow does not. **Collection select comes
FIRST, quantities SECOND** (corrected 2026-06-20 — was originally drafted the other way):

```
Scan (mode=collection)
  → Collection select   (multi-select which collection(s) to add to; single collection preselected)
  → Quantities screen   (set how many of each scanned card; default 0; ≥1 highlighted) → commit
  → Commit              (add each card at its quantity into EVERY selected collection)
```

- **Collection select is first.** Multi-select list of the account's COLLECTION lists (mirrors the
  group-targeting chips look for now). A single-collection account has it preselected so they just
  tap Next. Nothing is written here — it only records which lists the next step targets.
- **Quantities screen is second** and runs the commit. Quantities default to **0** (scanned but not
  yet added); the user steps each card up. A card with quantity ≥ 1 is **highlighted**. This is the
  add/subtract UX from CollectTab with its Save button. Whatever quantity is set is added to **every**
  collection selected on the first screen — quantity can't be varied per collection. 0 = added to
  nothing.
- The **listing** scan flow is unchanged — straight to the listings review, no collection select.

#### 4a. Collection select screen (FIRST) — `src/screens/trade/CollectScanTargetScreen/index.tsx`
- Route param: none. Multi-select list of the account's COLLECTION lists (from
  `getAllListsQueryConfig` filtered to `type === 'COLLECTION'`). Single collection is preselected.
- Footer **"Next"** → `navigation.navigate('CollectScanItems', { listIds })` with the selected ids.
  Writes nothing.

#### 4b. Quantities screen (SECOND, commits) — `src/screens/trade/CollectScanItemsScreen/index.tsx`
- Route param: `{ listIds: string[] }` (the collections chosen on 4a).
- Lists scan items where `type === 'COLLECTION'`. `/scan-items` **already filters by `type`** —
  pass `&type=COLLECTION` via a typed `useScanItems`.
- Each row: entity card + quantity stepper (default **0**, highlighted when ≥ 1). Reuse
  `TradeCardSelectionRow` (same component CollectTab uses).
- Footer **"Add N to collection(s)"**:
  - Persist each chosen quantity onto its scan item via `PUT /scan-item/:id`, then call
    `POST /scan-items/batch/commit-collection` with `{ accountId, scanItemIds, listIds }`.
  - On success: invalidate the same keys CollectTab invalidates (`collectTab`, `lists`,
    `listEntities`, `entities`, `Feed`, `search`, `userCollection`) plus `scanItems`; `popToTop()`
    back to the Collect tab. (Invalidations live in `useBatchCommitCollectionScans`.)

---

## Backend (nanza-api) — new collection-scan commit endpoint

**State of the backend (verified):**
- `ScanItem.type` enum already has `LISTING | COLLECTION` (no migration needed).
- `GET /scan-items` **already** supports a `?type=` filter (`scanItem.ts` list handler).
- `addCollectionEntities(tx, { accountId, entries })` in `src/services/trade.ts:90` does
  **increment-or-create** per `(list, entity)` — but it writes only to the account's *single*
  COLLECTION list (`findFirst where type=COLLECTION`, auto-creates if absent). **It does not take
  a target list id.** Multi-collection requires writing to *specific, multiple* list ids, so we
  need a list-id-targeted variant (see below). Still don't touch `entity-list/batch` (SET-based).
- `POST /scan-items/batch/convert` (`scanItem.ts:360`) ignores `type` and always creates Listings;
  it also requires `validateSeller`. We do **not** reuse it for collection.

**New service helper: `addEntitiesToList(tx, { listId, entries })`** in `src/services/trade.ts`.
Same increment-or-create loop as `addCollectionEntities`, but targets a **given** `listId` instead
of the account's default COLLECTION list. (Refactor `addCollectionEntities` to resolve the list id
then delegate to `addEntitiesToList`, to avoid duplicating the loop. Confidence 0.85.)

**New endpoint: `POST /scan-items/batch/commit-collection`** in `src/routers/scanItem.ts`.
Mirrors `batch/convert` structure but simpler, and writes to multiple collections:

Request: `{ accountId, scanItemIds: string[], listIds: string[] }`
- `listIds` are the selected COLLECTION lists. (If the FE single-collection path wants, it can send
  one id; or omit `listIds` to mean "the primary/default COLLECTION" — **decide**: recommend the FE
  always sends explicit `listIds` so the server never guesses. Confidence 0.8.)

1. Validate: `accountId` present; `scanItemIds` non-empty string array ≤ `SCAN_ITEM_BATCH_CONVERT_MAX`;
   `listIds` non-empty string array.
2. Auth: `validateAccount(req.user, accountId, 'seller')` — **but NOT `validateSeller`**
   (any account can have a collection). Confidence 0.8 — confirm the `validateAccount` role arg.
3. Verify every `listId` belongs to this account and is `type=COLLECTION` (reject otherwise — don't
   let a user write into someone else's or a non-collection list).
4. Fetch the scan items (`id in scanItemIds`, `accountId`), select `id, type, entityId, quantity`.
   - Reject if any id is missing/foreign (same 404-style message as batch/convert).
   - Reject if any item has `type !== 'COLLECTION'`.
   - Collection readiness = `entityId` present and `quantity ≥ 1`. New small helper (do NOT reuse
     `findScanItemReadinessIssues`, which demands price/condition/groups).
5. In one `prisma.$transaction`:
   - For each `listId`: `addEntitiesToList(tx, { listId, entries })` where entries are
     `{ entityId, quantity }` from the scan items. (Same quantities into every selected list.)
   - `tx.scanItem.deleteMany({ where: { id: { in: ids }, accountId } })`.
6. Respond `{ successCount, listIds }`.

**Quantity source:** persisted onto each scan item via `PUT /scan-item/:id` as the user steps it,
then read server-side at commit — consistent with how listing scans carry their own quantity, keeps
the commit body small, and survives app backgrounding. Confidence 0.8.

**FE service:** add `scanItemService.batchCommitCollection({ accountId, scanItemIds, listIds })` in
`src/services/api/ScanItem.ts` + a `useBatchCommitCollectionScans` hook in
`src/screens/trade/data/scanItems.ts`.

**Notes/risks:**
- `SCAN_ITEM_MAX = 10` caps *total* scans per account across both types — listing and collection
  scans share that budget. Acceptable for v1; flag if it becomes annoying.
- Follow batch/convert's atomicity pattern (pre-flight validation outside the tx; create+delete
  inside). Use `generatePrismaError` for the catch, matching the file's convention.

### 5. Styles
- New/extended classes in `src/styles/components/layout.ts` and/or
  `src/styles/components/scanItems.ts`. All theme tokens, no hex, no inline (P0 #1–#5).

## Decisions (confirmed with Skylar 2026-06-20)

1. **Add vs. set quantity → ADD.** Scanning a card already in the collection increments the
   existing count, not replace it.
2. **Scan pool isolation → APPROVED.** Filter the scan pool by `type` so listing scans and
   collection scans don't mix (client-side for the FE pass; backend `type` filter on
   `/scan-items` as a follow-up).
3. **Navigator edit → APPROVED.** OK to register the new `CollectScanItems` review screen in
   `AppNavigator.tsx` (additive, no reordering).
4. **Overall approach → APPROVED.** Same camera + multi-select, quantity-only review, commit
   via `entity-list/batch` into the active collection.

### Backend resolved (verified in nanza-api 2026-06-20)

- No dedicated collection-scan commit endpoint exists yet → **add** `POST
  /scan-items/batch/commit-collection` (see Backend section above).
- `entity-list/batch` is SET-based; the additive `addCollectionEntities` service already exists
  and is what the new endpoint uses. No `entity-list` changes needed.
- `/scan-items` already filters by `type` → FE filters server-side via `&type=COLLECTION`,
  not client-side.
- No Prisma migration needed (`ScanItemType.COLLECTION` already in schema).

## File touch list (estimate)

**Backend (nanza-api) — BUILT 2026-06-20, tsc + eslint clean:**
- `src/services/trade.ts` — added `addEntitiesToList(tx, { listId, entries })`; refactored
  `addCollectionEntities` to resolve the COLLECTION list then delegate to it. ✅
- `src/validation/scanItem.ts` — added `findCollectionScanReadinessIssues` (entityId + quantity ≥ 1). ✅
- `src/routers/scanItem.ts` — added `POST /scan-items/batch/commit-collection`
  (body `{ accountId, scanItemIds, listIds }`): validates account + that every listId is an
  owned COLLECTION + every scan is owned/COLLECTION/ready, then in one tx deposits each card's
  quantity into every selected list and deletes the scans. No seller gate. Router already mounted. ✅

**Frontend (nanza-mobile):**
- `src/services/api/ScanItem.ts` (`batchCommitCollection` with `listIds`)
- `src/screens/trade/data/scanItems.ts` (`useBatchCommitCollectionScans`; typed `useScanItems` by type)
- `src/types/navigation.ts` (params: `ScanCamera.mode`, `CollectScanItems`, `CollectScanTarget`)
- `src/navigation/AppNavigator.tsx` (register the two new screens — approved)
- `src/screens/trade/TradeScreen/index.tsx` (Sell: bottom button → Create Lot; search icon → camera; drop caption)
- `src/screens/trade/TradeScreen/CollectTab.tsx` (Collect: camera icon next to filter)
- `src/screens/trade/ScanCameraScreen/index.tsx` (mode → type + destination)
- `src/screens/trade/CollectScanItemsScreen/index.tsx` (NEW — quantities screen)
- `src/screens/trade/CollectScanTargetScreen/index.tsx` (NEW — collection multi-select)
- `src/styles/components/layout.ts` (+ maybe `scanItems.ts`) (styles)

Navigator edit approved (additive screen registration only).
</content>
</invoke>
