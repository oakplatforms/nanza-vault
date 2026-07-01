# Commission change: 5% → 3% + remove moderator/group rev-share

**Goal:** Lower the platform application fee from 5% to 3% (keep the `+ $0.50` and `+ tax`), and completely remove the 1% group/moderator revenue-share payout system. Keep the moderator *role* — just stop paying them a cut.

---

## Part 1 — Application fee 5% → 3%

### Backend (nanza-api)
- `src/services/invoice.ts:52` — change `Math.round(totalAmount * 0.05)` to `* 0.03`. Everything else (`+ 50 + taxAmount`) stays.

**Confidence: 0.98** — single-line, unambiguous. Keeps small-order fee, tax, shipping, capture, refund flow untouched.

### Web marketing copy (nanza-web-app) — display only, no payment logic
- `src/landing/components/Hero.tsx:43` — "5% + 50¢" → "3% + 50¢"
- `src/landing/components/PayoutCalculator.tsx`
  - L8: `{ name: 'Nanza', rate: 0.05, ... }` → `rate: 0.03`
  - L19: "Flat 5% commission — no hidden fees" → "Flat 3% commission — no hidden fees"
  - any other "5%"/"50¢" copy in the same component

**Confidence: 0.95** — cosmetic; calculator math is driven by the `rate` constant so updating it keeps the comparison accurate.

> Note: Mobile cart (`ReviewCartScreen`) and web cart (`CartOrderSummary`) do **not** display the platform fee — it's deducted seller-side at Stripe — so no buyer-facing checkout copy needs to change. Only the marketing landing page references the percentage.

---

## Part 2 — Remove moderator / group 1% rev-share

The moderator commission is a Stripe **transfer** paid out of the platform after an order
is settled, retried by a cron lambda if stuck, and reversed on refund. Removing it means
stop calling the settle/reverse functions, delete the service + retry lambda, and (chosen)
drop the DB table + enum value.

### Call sites to remove (4 — original map missed 2)
- `src/routers/order.ts:13, 1292` — remove import + `settleModeratorTransfers(id)` call after order acceptance.
- `src/webhooks/shippo.ts:6, 111-115` — remove import + `settleModeratorTransfers(...)` call on DELIVERED (this is the webhook you mentioned).
- `src/routers/refund.ts:10, 642` — remove import + `reverseModeratorTransfers(orderId)` call on refund accept.

### Service + lambda + infra to delete
- `src/services/moderatorTransfer.ts` — delete the file entirely.
- `lambdas/moderatorTransferRetryHandler.ts` — delete the dedicated retry cron lambda.
- `serverless.yml:206-214` — remove the `moderatorTransferRetry` function block + its cron schedule so it's no longer deployed.

**Confidence: 0.92** — clean once all 4 call sites + lambda + serverless entry are gone.
Nuance: `refund.ts` passes `reverse_transfer: true` to the Stripe refund; with no transfers
it's a no-op, so I'll **leave it** to avoid touching refund behavior (low risk).

### Schema / DB — DROP (your call: tables empty/unused → drop)
- `prisma/schema.prisma` — remove the `ModeratorTransfer` model, its back-relations on
  `Order`/`Group`/`Account` (`moderatorTransfers` / `moderatorTransfersReceived`), and the
  `MODERATOR_COMMISSION` value from the `TransactionType` enum.
- Generate the migration (drop table + relations). **You run the migration** — I'll generate
  the SQL/migration files but not execute `prisma migrate`.

**Confidence: 0.8** on the enum drop — Postgres can't easily remove an enum value if any
historical `Transaction` row uses `MODERATOR_COMMISSION`. Since you said the table's empty,
this should be safe, but the migration will need the standard "recreate type without the
value" dance. I'll flag if any rows reference it before finalizing.

> Note: `orderListing.sourceGroupId` / `sourceModeratorAccountId` fields and the group→listing
> relationship stay — they're used for attribution/feed, not just the payout. Not touching those.
> The moderator **role** stays — only the payout goes.

---

## Files touched
1. `nanza-api/src/services/invoice.ts` (fee 5→3)
2. `nanza-api/src/routers/order.ts` (remove import + settle call)
3. `nanza-api/src/webhooks/shippo.ts` (remove import + settle call)
4. `nanza-api/src/routers/refund.ts` (remove import + reverse call)
5. `nanza-api/src/services/moderatorTransfer.ts` (DELETE)
6. `nanza-api/lambdas/moderatorTransferRetryHandler.ts` (DELETE)
7. `nanza-api/serverless.yml` (remove retry cron function)
8. `nanza-api/prisma/schema.prisma` (drop model + enum value + relations) → migration
9. `nanza-web-app/src/landing/components/Hero.tsx` (copy 5%→3%)
10. `nanza-web-app/src/landing/components/PayoutCalculator.tsx` (copy + rate 0.05→0.03)

No nanza-mobile changes needed.

## Verification
- `tsc` / lint nanza-api and nanza-web-app.
- Grep to confirm zero remaining refs to `moderatorTransfer` / `settleModeratorTransfers` /
  `reverseModeratorTransfers` / `MODERATOR_COMMISSION` (outside generated json-schema, which
  regenerates).
- Regenerate prisma client + json-schema after schema edit.
- User runs the migration.
