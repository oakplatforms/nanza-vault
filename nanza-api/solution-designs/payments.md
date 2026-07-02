---
tags: [nanza-api, solution-design, payments, checkout, orders]
---

# Payments — Solution Design

## Overview

nanza takes a platform cut on every marketplace sale via a Stripe **application fee**, and settles
seller earnings through Stripe Connect. This area covers: the current commission formula
(**3% + $0.50 + tax**), the **complete removal of the 1% group/moderator revenue-share** payout system
that used to sit alongside it, and the **delivery-mode × payment-type model** (Ship vs In Person,
Card vs Cash) — where **in-person cash** is the one path that never touches Stripe.

## How it works

### Application fee (the platform cut)

The application fee is computed in `src/services/invoice.ts` and deducted seller-side at Stripe —
it is **not** shown in either the mobile cart (`ReviewCartScreen`) or the web cart
(`CartOrderSummary`), because the buyer never pays it directly. The formula:

```
applicationFee = round(totalAmount * 0.03) + 50 + taxAmount
```

- The rate is **3%** of the order total (was 5%).
- `+ 50` cents keeps a floor on very small orders.
- `+ taxAmount` passes collected tax through.
- Shipping, capture, and refund flows are untouched by the rate.

The only buyer-facing surface that names the percentage is the **web marketing landing page**
(`nanza-web-app`): `Hero.tsx` copy and `PayoutCalculator.tsx` (whose comparison math is driven by a
`rate: 0.03` constant, so updating the constant keeps the calculator accurate). The mobile app label
reads "Processing Fee (3%)".

### Moderator / group revenue-share — REMOVED

Previously, when an order sourced from a private group completed, the platform paid the group's
moderator a **1% cut** via a Stripe **transfer** out of the platform balance. This was a full
subsystem — a settle-on-completion service, a retry cron Lambda for stuck transfers, a reverse-on-
refund path, a Prisma model, and a transaction-type enum value. **All of it was ripped out.** The
moderator **role** remains; only the payout is gone.

What was removed:

- **Call sites (4):**
  - `src/routers/order.ts` — the `settleModeratorTransfers(id)` call after order acceptance.
  - `src/webhooks/shippo.ts` — the `settleModeratorTransfers(...)` call on the DELIVERED → COMPLETED
    transition.
  - `src/routers/refund.ts` — the `reverseModeratorTransfers(orderId)` call on refund accept.
  - (plus the corresponding imports)
- **Service + Lambda + infra:**
  - `src/services/moderatorTransfer.ts` — deleted.
  - `lambdas/moderatorTransferRetryHandler.ts` — the dedicated retry cron Lambda, deleted.
  - `serverless.yml` — the `moderatorTransferRetry` function block + its cron schedule removed so it
    no longer deploys.
- **Schema / DB:**
  - `prisma/schema.prisma` — the `ModeratorTransfer` model and its back-relations on
    `Order` / `Group` / `Account` (`moderatorTransfers` / `moderatorTransfersReceived`) dropped, and
    the `MODERATOR_COMMISSION` value removed from the `TransactionType` enum. Tables were
    empty/unused, so this was a clean drop (the user runs the migration).

### What deliberately stayed

- `refund.ts` still passes `reverse_transfer: true` to the Stripe refund. With no transfers left it
  is a harmless no-op, and leaving it avoids touching refund behavior.
- `OrderListing.sourceGroupId` / `sourceModeratorAccountId` and the group→listing relationship
  **stay** — they power attribution and the feed, not just the payout. The snapshot of where a sale
  came from is still recorded at purchase time; it just no longer triggers a transfer.

## Delivery modes & payment types (in-person cash/card)

An order is described by **two orthogonal axes** — how it's delivered and how it's paid. Both live
as scalar enums on `Order` with back-compatible defaults, so every pre-existing order reads as a
normal shipped card order with no data change:

```prisma
enum DeliveryType { SHIP IN_PERSON }        // Order.deliveryType @default(SHIP)
enum PaymentType  { CARD CASH }             // Order.paymentType  @default(CARD)
```

- **SHIP** is the existing flow: a shipment is created (SHIPPO tracked or UNTRACKED plain-envelope),
  the buyer is always charged by **card** through Stripe.
- **IN_PERSON** means buyer and seller meet — there is **no shipment**, so no shipping fee and no
  shipping-option add-ons. It can be paid **CASH** or **CARD**.

`PaymentType` is written to the shared `@oakplatforms/types` package (generated from Prisma), so the
DTO's `deliveryType` / `paymentType` flow into mobile and admin automatically.

### The one rule that drives everything

**Only cash skips Stripe. Anything that touches Stripe is taxed and pays the platform fee.** Concretely:

| Mode | Shipment | Stripe PI | Tax | Small-order fee ($1 min) | Shipping/handling |
| --- | --- | --- | --- | --- | --- |
| Ship (tracked/untracked) | yes | yes (capture on accept) | yes | yes | yes |
| In-person **card** | no | yes | **yes** | **yes** | no |
| In-person **cash** | no | **no** | no | no | no |

The small-order fee is kept for in-person **card** deliberately: the 3% + $0.50 processing fee would
otherwise make the seller lose money on a sub-$1 card sale. Cash never touches Stripe, so it's waived.
The predicate everywhere is `isCash`, **not** `isInPerson`.

### Backend (`nanza-api`)

- **`POST /order` + `PUT /order/:id`** accept & persist `deliveryType` / `paymentType`. Validation:
  `CASH` is only valid when the resulting order is `IN_PERSON`; switching an order to `IN_PERSON`
  **disconnects any shipping method** (so stale tracked/untracked data never lingers), and switching
  to a shipping method flips `deliveryType` back to `SHIP` + `paymentType` to `CARD`. Orders are also
  capped at a **$1,000 subtotal** (enforced on create, update, and again at invoice time).
- **`services/invoice.ts`** (`createInvoiceWithTransactions`) branches on the two flags: in-person ⇒
  `shipping = 0` and **no shipment created**; cash ⇒ `tax = 0`, `smallOrderFee = 0`, `total = subTotal`,
  **no Stripe payment intent**, and the transaction row is recorded with `paymentMethodType = CASH`
  (a new value on the `PaymentMethodType` enum). Cash orders send **no emails** — the seller/customer
  eventBridge sends are gated on `paymentType !== CASH` — but they still fire the in-app `ORDER_SOLD`
  system message so the seller knows to accept.
- **Mixed-cart guard:** cash is in-person-only across the *whole* checkout. If any order in an invoice
  is cash, every order must be in person, else the buy is rejected (checked in both
  `validation/invoice.ts#validateOrdersForInvoice` — the up-front, clear-error gate — and again inside
  `createInvoiceWithTransactions`).
- **`accept-order`:** in-person orders complete **on accept**, same as untracked. The handler branches
  on `isInPerson` *before* the shipment lookup (in-person has no shipment); cash skips the Stripe
  capture, card captures as normal, and both then set the order `COMPLETED` and run
  `syncOrderCollections`. That collection transfer (seller→buyer cards, via `transferCollectionEntities`,
  the same helper trades use) depends only on the two account IDs and the order's listings — **no**
  dependency on shipment, payment intent, or payment type — so it behaves identically for cash, card,
  and untracked.
- **`cancel-order`:** likewise guarded — in-person orders don't require a shipment to cancel, and cash
  orders skip the payment-intent cancel.

### Consuming surfaces

The two enums flow to the clients via the generated `@oakplatforms/types` DTO. Each consuming repo
documents its own slice:

- **Checkout UI (mobile):** [[../../nanza-mobile/solution-designs/payments|nanza-mobile — Payments & checkout]] — the 3-way delivery selector, the Pay With / Cash placement, the no-jump summary math, and the in-person order-status timeline.
- **Operator visibility (admin):** [[../../nanza-admin/solution-designs/payments|nanza-admin — Payments visibility]] — the Delivery/Payment badges on the Orders list and detail.

## Key decisions & rationale

- **3% flat, keeping the $0.50 floor + tax pass-through.** A single-line rate change in
  `invoice.ts`; the floor still protects tiny orders and tax still flows through. The web
  calculator's constant-driven math means marketing copy stays truthful automatically.
- **Rev-share removed entirely, role kept.** The 1% moderator payout added a Stripe transfer, a
  retry cron, a reversal path, a model, and an enum value — real operational surface for a
  monetization mechanic being dropped. Removing the payout while keeping the moderator role and the
  source-attribution fields is the minimal, reversible-in-spirit end state.
- **Enum-value drop needs the recreate-type dance.** Postgres can't remove an enum value in place if
  any historical `Transaction` row references `MODERATOR_COMMISSION`. Because the table is empty the
  migration is safe, but it still uses the standard "recreate the type without the value" pattern.
- **Fee stays seller-side / invisible to buyers.** Since Stripe deducts the application fee from the
  seller, no checkout copy changes; only the marketing landing page names the percentage.
- **Delivery & payment are orthogonal enums on `Order`, defaulted for back-compat.** Modeling In Person
  as a `deliveryType` rather than a `ShippingMethod` keeps the tracked/untracked machinery untouched and
  lets an order carry a payment axis independently. Non-null defaults (`SHIP`/`CARD`) backfill every
  existing row, so the migration is additive and safe.
- **Cash is the only Stripe-free path; the gating predicate is `isCash`, never `isInPerson`.** In-person
  *card* is a normal Stripe order minus shipping, so it keeps tax and the $1 small-order fee (which
  covers the 3% + $0.50 the seller would otherwise eat on a sub-$1 sale). Only cash — which never
  reaches Stripe — waives tax, fee, and emails.
- **In-person reuses the untracked accept flow.** Completing on seller-accept (and firing the same
  `syncOrderCollections` transfer) means the card-movement, connection, and completion logic is shared,
  not forked — the only in-person-specific branch is "no shipment / cash skips capture."
- **No layout jump = same rows, disabled-not-removed.** In the cart, in-person renders the identical
  summary rows (zeroed) and keeps the shipping-options / address / fee-warning elements present but
  disabled, rather than unmounting them — unmounting was the source of the visible "jump"/fade.

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
