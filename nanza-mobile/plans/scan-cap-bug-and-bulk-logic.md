# Scan cap bug + cap/bulk design recommendation

## The bug (confirmed)

When you've scanned 10 COLLECTION items, scanning a LISTING item errors with
"You already have 10 scans…". COLLECTION and LISTING scans are supposed to be
separate pools, but the cap counts them together.

**Root cause — one query, `nanza-api/src/validation/scanItem.ts:18`:**

```ts
const count = await prisma.scanItem.count({ where: { accountId } }) // no `type` filter
```

`SCAN_ITEM_MAX = 10` (`scanItem.ts:9`) is then applied to that combined count. So
10 COLLECTION scans fill the cap for LISTING scans too.

Everything else in the system already treats the pools as separate:
- The model has `ScanItem.type` = `LISTING | COLLECTION` (`schema.prisma:184-187`, `1033-1060`).
- The frontend fetches a **type-scoped** pool: `useScanItems(accountId, type)`
  (`nanza-mobile/src/screens/trade/data/scanItems.ts`), with a comment that the
  two flows "keep separate pools so listing scans and collection scans never mix."
- The Scan camera badge `Scans (N)` only counts the current mode
  (`ScanCameraScreen/index.tsx:142-146, 401`) — which is why you can see
  "Scans (0)" in LISTING mode and still get the 409.

Only `enforceScanItemLimit` ignores `type`. It's the odd one out.

## Where the cap is enforced

`nanza-api/src/routers/scan.ts:111`, inside `POST /scan/cards`, before the paid
card-identification call. Note: today `validateScanMode(mode)` runs at line 115,
**after** the cap check at line 111 — so to make the cap type-aware we must move
the mode validation above the cap check.

## Three separate limits — don't conflate

| Constant | Value | Meaning | File |
|---|---|---|---|
| `SCAN_ITEM_MAX` | 10 | Max pending scans held at once (the cap you hit) | `scanItem.ts:9` |
| `SCAN_ITEM_BATCH_CONVERT_MAX` | 10 | Max scans converted/committed in one batch | `scanItem.ts:4` |
| `DAILY_SCAN_LIMIT` | 50 | Scan API calls per 24h (not currently wired into `/scan/cards`) | `services/scan.ts:3` |

## Do scans affect bulk size? No.

Scans never become bulks directly. The flows are:
- LISTING scans → **Listings** via `POST /scan-items/batch/convert` (`scanItem.ts:361-519`).
- COLLECTION scans → **collection List entries** via `POST /scan-items/batch/commit-collection` (`scanItem.ts:537-627`).

A `BulkListing` is a *group of existing Listings* (`Listing.bulkListingId`,
`schema.prisma:1017-1018`, `528-556`). The bulk router has zero scan references.
**Bulk size is driven by how many Listings you group, not by the scan-pool cap.**
Raising the holding cap to 20 does **not** change bulk sizes.

## Recommendation

**Fix the bug as separate per-type pools (Option A), and raise both caps together
if you want more headroom.** Confidence: 0.85.

### Option A — separate pools (recommended)
Make the cap count per `type`: 10 COLLECTION + 10 LISTING, independent.
- Matches the model, the frontend hooks, and the per-mode badge — zero frontend change.
- Smallest, most surgical fix; the badge you already show becomes accurate.
- Change: add a `type` param to `enforceScanItemLimit`, filter the count by it, and
  call it after `validateScanMode` in `scan.ts`.
- Rejected alternative — **Option B (one unified pool of 20):** raise
  `SCAN_ITEM_MAX` to 20, leave the count un-filtered. Simpler diff but keeps the
  two flows competing for one budget and leaves the per-mode badge misleading
  (badge shows one mode's count, cap measures both). Confidence it's the better
  model: 0.4.

### Optional, independent of the above
- **Raise the cap to 20 per pool:** bump `SCAN_ITEM_MAX` to 20. Safe; no bulk impact.
  If you do, also raise `SCAN_ITEM_BATCH_CONVERT_MAX` to 20 so a full pool converts
  in one batch — and update its test `scanItem.test.ts:75-76`
  (`expect(SCAN_ITEM_BATCH_CONVERT_MAX).toBe(10)`). There is no test pinning
  `SCAN_ITEM_MAX`, so raising the holding cap alone breaks nothing.
- **Add `@@index([accountId, type])`** to `ScanItem` to keep the now type-filtered
  count query fast (today only `[accountId]` and `[accountId, createdAt]` exist).
  Requires a migration (you run migrations).

## Proposed change (Option A, cap stays 10 unless you say otherwise)

`nanza-api/src/validation/scanItem.ts`:
```ts
export const enforceScanItemLimit = async (
  accountId: string,
  type: ScanItemType,
): Promise<void> => {
  const prisma = prismaClient()
  const count = await prisma.scanItem.count({ where: { accountId, type } })
  if (count >= SCAN_ITEM_MAX) {
    const err = new Error(
      `You already have ${SCAN_ITEM_MAX} ${type === 'COLLECTION' ? 'collection' : 'listing'} scans. ` +
      'Create your listings or remove a scan before scanning more.',
    )
    ;(err as Error & { statusCode?: number }).statusCode = 409
    throw err
  }
}
```

`nanza-api/src/routers/scan.ts` (reorder so mode is validated first):
```ts
const validatedMode = validateScanMode(mode)
await enforceScanItemLimit(accountId, validatedMode)
```

## Decision (implemented)
- Per-type pools, **20 per pool** (`SCAN_ITEM_MAX = 20`). Counted per `type` so
  collection and listing pools are independent.
- **Batch-convert max left at 10** (`SCAN_ITEM_BATCH_CONVERT_MAX` unchanged), per
  request — a full 20-scan pool converts in two batches.
- Error message is now type-specific ("…20 collection scans" / "…20 listing scans").
- `validateScanMode` reordered above the cap check in `scan.ts` so the mode is known.

### Batch caps also raised to 20 (follow-up decision)
Everything is 20 now, for consistency with the 20-scan pools:
- `SCAN_ITEM_BATCH_CONVERT_MAX` 10 → **20** (`scanItem.ts:4`) — covers both scan→listing
  convert and scan→collection commit (same constant). Test updated
  (`scanItem.test.ts:75-76` value + `:96` regex `/more than 20/`).
- `ENTITY_LIST_BATCH_UPDATE_MAX` 10 → **20** (`entityList.ts:1`) — collection
  batch-update of entity quantities; also bumps the `maxBatchSize` surfaced to the FE
  (`entityList.ts:735, 825`). No tests pinned it.

### Bulk/lot item cap added (20)
Manual lots previously had **no** maximum (only a minimum of 2 to publish). Added a
hard cap of 20 child listings, enforced on both create and update:
- `BULK_LISTING_MAX_ITEMS = 20` (`bulkListing.ts`, near the top of the router).
- Create (`POST /bulk-listing`): rejects when `listingIds.length > 20`.
- Update (`PUT /bulk-listing/:id`): rejects when the **resulting** total
  (`existing + added − removed − deleted`) `> 20` — enforced for all updates, not just
  publish, so a draft can't grow past 20 either.
- Error: "A bulk listing cannot contain more than 20 items."

FE note: this is server-enforced (the error surfaces as an alert). A nicer touch
would be to disable the add-item button at 20 in the lot builder, but the hard cap
holds regardless — optional FE polish, not required for correctness.

### Still optional (not done)
- `@@index([accountId, type])` on `ScanItem` for the type-filtered count query —
  needs a migration (you run those). Low urgency at this scale.
