---
tags: [nanza-api, solution-design, data-sync]
---

# Data Sync — Solution Design

## Overview

A user's **Collection** is a `List` with `type: 'COLLECTION'`, holding `EntityList` join rows
(entity + quantity). This area keeps that collection in sync with the user's marketplace activity —
so a card you list shows up in your collection, and a card that sells moves out of the seller's
collection and into the buyer's — mirroring what card-for-card **trades** already do. All of this
sync is **best-effort**: it must never throw in a way that breaks listing creation or order
completion.

## How it works

### The shared collection-move primitive

`src/services/trade.ts` already owns the tolerant, quantity-aware collection move that trades use:

- `transferCollectionEntities(tx, { fromAccountId, toAccountId, entries })` — finds the sender's
  `COLLECTION` list and the receiver's (auto-creating the receiver's if missing), then for each
  `{ entityId, quantity }` decrements/deletes from the sender and increments/creates on the
  receiver. The **sender side is tolerant**: it removes `min(have, want)` and never throws on a
  shortage.

Collection items are keyed on `(listId, entityId)` only — there is **no condition dimension** — so
sync happens purely by `entityId` + quantity; a listing's condition is irrelevant to the collection.

To let listing-creation deposit into a collection without a phantom `from` side, the **deposit half**
of `transferCollectionEntities` is extracted into a reusable helper:

- `addCollectionEntities(tx, { accountId, entries })` — find-or-create the account's `COLLECTION`
  list, then increment-or-create each `EntityList` row (the same logic the receiver branch already
  used). `transferCollectionEntities` now reuses this for its deposit side, so there is one shared
  deposit path and no behavior change to trades.

### 1. On listing creation

After `prisma.listing.create` succeeds in `src/routers/listing.ts`, the listed card is added to the
**seller's own** collection at the listing quantity, inside its own tiny transaction wrapped in a
`try/catch`. `entityId` and the parsed quantity are already in scope. Because listing creation has
no surrounding transaction, the isolated transaction + catch guarantees the response still sends
even if the sync fails.

### 2. On order completion

Both completion paths add a sibling best-effort call next to the existing settle logic, via a new
service function `syncOrderCollections(orderId)` (in `src/services/order.ts`):

- It loads the order with `seller.accountId`, `customer.accountId`, and
  `orderListings.listing.entityId`, builds `{ entityId, quantity }` entries from purchased lines
  with a positive quantity, and calls `transferCollectionEntities` (seller → buyer) inside a single
  transaction. If either account or any entity is missing, it returns without moving anything.

The two completion paths that invoke it:

- **`src/routers/order.ts`** — in the accept-order handler, beside the settle call, wrapped in
  `try/catch`.
- **`src/webhooks/shippo.ts`** — in `handleShippoTrackingUpdated`, right after the DELIVERED →
  COMPLETED block, also wrapped in `try/catch`.

Both calls sit **outside** the order-completion transaction, so a collection failure can never roll
back or block completion.

## Key decisions & rationale

- **Reuse the trade primitive, don't reinvent.** The tolerant move already exists in `trade.ts`;
  inlining a separate add block in the listing router would duplicate the increment-or-create logic
  and risk drift if collection semantics change. Extracting `addCollectionEntities` keeps one shared
  deposit path (additive export, no signature change to the existing `transferCollectionEntities`).
- **Best-effort, always isolated.** Each sync runs in its **own** transaction guarded by
  `try/catch` (logged with `console.warn`). A collection error is never allowed to break listing
  creation or roll back order completion — this is a hard requirement, so calling the sync inside the
  completion transaction was explicitly rejected.
- **Sync by entity + quantity, condition-agnostic.** `EntityList` has no condition column, so the
  collection tracks how many of a card you hold, not its condition — sync mirrors that.
- **Tolerant of missing/short inventory.** If the seller no longer holds the card (or holds fewer
  than sold), the move takes `min(have, want)` and continues; a missing buyer collection is
  auto-created. No new DTOs or types — Prisma-inferred shapes only.

### Known edge case

On the tracked (webhook) path, the completion block only fires on the real
`previousStatus !== mappedStatus` transition to DELIVERED, so sync runs once per genuine transition.
A rare duplicate webhook delivery could over-move at most once — accepted and flagged for review.

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
