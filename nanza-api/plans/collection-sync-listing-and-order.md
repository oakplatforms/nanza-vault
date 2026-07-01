# Plan: Sync collections on listing creation & order completion

## Goal

Two new behaviors that keep a user's **Collection** (`List` with `type: 'COLLECTION'`) in
sync with their marketplace activity — mirroring what trades already do:

1. **On listing creation** — add the listed card (entity) to the seller's own collection,
   at the listing's quantity.
2. **On order completion** — move the purchased card(s) from the **seller's** collection to
   the **buyer's** collection, at the exact purchased quantity.

Both behaviors are **complimentary / best-effort**: they must NEVER throw an error that
breaks listing creation or order completion. If the seller no longer has the card, or the
quantity is short, we move what we can (or nothing) and silently continue.

---

## What already exists (reuse, don't reinvent)

`src/services/trade.ts` → `transferCollectionEntities(tx, { fromAccountId, toAccountId, entries })`
already does a tolerant, quantity-aware collection move:

- Finds `fromList` (seller COLLECTION) and `toList` (buyer COLLECTION); auto-creates the
  buyer's collection if missing.
- For each `{ entityId, quantity }`: decrements/deletes from `fromList`, increments/creates
  in `toList`. Sender side is tolerant (removes `min(have, want)`, never throws on shortage).

Collection items (`EntityList`) are keyed on `(listId, entityId)` only — **no condition
dimension**. So we sync by `entityId` + quantity. Listing condition is irrelevant to the
collection.

---

## Design

### Helper change (DRY) — `src/services/trade.ts`  · Confidence: 0.88

Extract the **deposit** half of `transferCollectionEntities` into a small reusable helper so
listing-creation can "add to my own collection" without a phantom `from` side:

```ts
export const addCollectionEntities = async (
  tx: Prisma.TransactionClient,
  args: { accountId: string; entries: Array<{ entityId: string; quantity: number }> },
) => {
  // find-or-create the account's COLLECTION list, then for each entry
  // increment-or-create EntityList (same logic as the toList branch today)
}
```

Then `transferCollectionEntities` reuses `addCollectionEntities` for its deposit side
(withdrawal logic stays inline). Net: no behavior change to trades, one shared deposit path.

*Rejected alternative (Confidence 0.6): inline a separate add block in the listing router.*
*Why rejected: duplicates the increment-or-create logic that already lives in trade.ts —
violates DRY (P1 #14), and we'd drift if collection semantics change.*

> P0 #8 note: `src/services/*` is not on the "shared infrastructure off-limits" list
> (that's `contexts/`, `styles/utils/`, `fetchData`). This is an additive export + internal
> refactor of an existing service, no signature change to the existing export.

### 1. Listing creation — `src/routers/listing.ts` (~after line 385)  · Confidence: 0.9

After the listing is successfully created (and **outside** any failure path), add the card to
the seller's collection at the listing quantity:

```ts
// Best-effort: mirror the new listing into the seller's collection.
// Never let a collection failure break listing creation.
try {
  await prisma.$transaction(async (tx) => {
    await addCollectionEntities(tx, {
      accountId,
      entries: [{ entityId, quantity: parsedQuantity }],
    })
  })
} catch (err) {
  console.warn(`Collection sync (create listing) failed for entity ${entityId}:`, err)
}
```

- Runs after `prisma.listing.create` returns, before/after `res.json(listing)` — placed
  before the response is fine; the try/catch guarantees the response still sends.
- `parsedQuantity` and `entityId` are already in scope.
- Listing creation has no surrounding transaction today, so wrapping our own tiny
  transaction (helper needs a `TransactionClient`) is correct and isolated.

### 2. Order completion — both flows  · Confidence: 0.85

Both completion paths converge on calling `settleModeratorTransfers(orderId)` **after** the
order is marked `COMPLETED` and outside the main DB transaction. We add a sibling
best-effort call right next to it, using the same isolation pattern.

New service function `src/services/order.ts` (or colocated) — `syncOrderCollections(orderId)`:

```ts
// Best-effort: move purchased cards from seller's collection to buyer's.
export const syncOrderCollections = async (orderId: string): Promise<void> => {
  const prisma = prismaClient()
  const order = await prisma.order.findUnique({
    where: { id: orderId },
    include: {
      seller: { select: { accountId: true } },
      customer: { select: { accountId: true } },
      orderListings: { include: { listing: { select: { entityId: true } } } },
    },
  })
  if (!order?.seller?.accountId || !order?.customer?.accountId) return

  const entries = order.orderListings
    .filter((ol) => ol.listing?.entityId && (ol.quantity ?? 0) > 0)
    .map((ol) => ({ entityId: ol.listing!.entityId!, quantity: ol.quantity! }))
  if (entries.length === 0) return

  await prisma.$transaction(async (tx) => {
    await transferCollectionEntities(tx, {
      fromAccountId: order.seller!.accountId!,
      toAccountId: order.customer!.accountId!,
      entries,
    })
  })
}
```

**Untracked** — `src/routers/order.ts`, in the `/order/:id/accept-order` handler, right
beside the existing untracked `settleModeratorTransfers` call (~line 1291):

```ts
try { await syncOrderCollections(id) }
catch (err) { console.warn(`Collection sync (order ${id}) failed:`, err) }
```

**Tracked** — `src/webhooks/shippo.ts`, in `handleShippoTrackingUpdated`, right after the
DELIVERED → COMPLETED block calls `settleModeratorTransfers` (~line 111):

```ts
try { await syncOrderCollections(shipment.order.id) }
catch (err) { console.warn(`Collection sync (order ${shipment.order.id}) failed:`, err) }
```

Both are **outside** the order-completion transaction → a collection failure can never roll
back or block completion. Uses `customer.accountId` as buyer and `seller.accountId` as seller
(both already loaded/loadable on the order).

*Rejected alternative (Confidence 0.5): call inside the completion transaction.*
*Why rejected: violates the hard requirement — a collection error would roll back the order.*

---

## Files touched (3)

| File | Change |
|------|--------|
| `src/services/trade.ts` | Add exported `addCollectionEntities`; refactor `transferCollectionEntities` deposit side to use it (additive, no signature change). |
| `src/services/order.ts` | New `syncOrderCollections(orderId)` (or colocate in trade.ts if preferred). |
| `src/routers/listing.ts` | Best-effort `addCollectionEntities` call after create. |
| `src/routers/order.ts` | Best-effort `syncOrderCollections` call beside untracked settle. |
| `src/webhooks/shippo.ts` | Best-effort `syncOrderCollections` call beside tracked settle. |

(That's 5 files — under the escalation threshold but worth flagging. All additive.)

---

## Edge cases & guarantees

- **Seller no longer has the card / short quantity** → `transferCollectionEntities` removes
  `min(have, want)` and never throws; buyer still receives what we attempted. ✅ matches
  "keep 8 with seller, move 2 to customer" example.
- **Buyer has no collection yet** → auto-created by the helper. ✅
- **Any DB/Stripe-adjacent failure in sync** → caught, logged with `console.warn`, completion
  unaffected. ✅
- **Duplicate webhook delivery (tracked)** → completion block only runs on the
  `previousStatus !== mappedStatus` transition to DELIVERED, so sync fires once per real
  transition. (Acceptable; a rare double would over-move at most once — note for review.)
- **No new types / DTOs**; uses Prisma-inferred shapes only (P0 #11, #12). ✅

## Confidence summary

- Reuse existing tolerant helper: **0.95**
- Best-effort isolation (separate txn + try/catch, never breaks completion): **0.92**
- Helper extraction `addCollectionEntities` (DRY): **0.88**
- Listing add at full listing quantity: **0.9** (per your confirmation)
- Order move at purchased quantity: **0.9** (per your confirmation)

No score below 0.8 → proceed on approval.
