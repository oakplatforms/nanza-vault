---
tags: [nanza-api, solution-design, payments]
---

# Payments — Solution Design

## Overview

nanza takes a platform cut on every marketplace sale via a Stripe **application fee**, and settles
seller earnings through Stripe Connect. This area covers two settled decisions: the current
commission formula (**3% + $0.50 + tax**), and the **complete removal of the 1% group/moderator
revenue-share** payout system that used to sit alongside it.

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

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
