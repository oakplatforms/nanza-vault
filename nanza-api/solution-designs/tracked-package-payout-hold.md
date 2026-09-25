---
tags: [nanza-api, nanza-email, nanza-mobile, solution-design, plan, payments, orders, shipping, stripe]
status: BUILT 2026-09-14, uncommitted, pending Skylar QA — API + email + mobile done. Migration `20260914120000_order_tracking_reminders` (three nullable Order columns + one SystemMessageType value) to run before deploy. TEMP for dev testing: reminder thresholds are 10 min / 15 min and the cron runs every 5 minutes — restore 48h / 72h + daily cron(0 16 * * ? *) before prod.
---

# Tracked Packages — capture on accept, hold until delivery, tracking reminder

## Goal

For a **tracked** (Shippo) order the money should move like this:

1. **Seller accepts** → the buyer's card is **captured** right then.
2. The seller's share sits **on hold** — visible in their wallet but not withdrawable.
3. **Delivery confirmed** (Shippo `DELIVERED` webhook) → the hold lifts and the seller can withdraw.
4. Delivery can only be confirmed if the seller **adds a tracking number**. No tracking number, no
   delivery signal, no payout — ever. So we remind the seller, **clock starting at accept**: a
   **reminder email at 48 h**, then an **in-app system message at 72 h**, each sent once.

The un-accepted sweep is untouched: an order the seller never accepts is still canceled 84 h after
creation. Nothing about tracking runs on an order until it has been accepted (Skylar, 2026-09-14).

Untracked-envelope and in-person orders are unchanged: they complete on accept and the funds are
released immediately (there is nothing to track).

## What already exists (read before touching anything)

Most of this flow is already in place — the ask is narrower than it sounds.

| Piece | Where | State |
| --- | --- | --- |
| Capture on seller accept | `services/order.ts#acceptOrder` — confirms + captures the manual-capture PI for every card order, tracked included; on failure the order is canceled and quantities restored | **Done.** The comment in `routers/shipment.ts` `PUT /shipment/:id` ("DB update first, then payment capture") is stale — capture moved to accept and nothing is captured when the tracking number is entered. |
| Tracked order stays open until delivery | after accept the shipment goes `PRE_TRANSIT`, order stays `PENDING`; `webhooks/shippo.ts` flips shipment + order to `COMPLETED` on `DELIVERED` and runs `syncOrderCollections` | **Done.** |
| Tracking number entry | `PUT /shipment/:id` with `trackingNumber` + `carrier` — validates the format, registers the number with Shippo tracking (`trackingStatus.create`, rejects `UNKNOWN` with no history), moves shipment `CREATED → PENDING`. `PUT /shipment/:id/tracking` is the one-time edit while still `PRE_TRANSIT`. | **Done.** |
| Funds hold | connected accounts are created with `payouts.schedule.interval = 'manual'` (`routers/seller.ts`), so Stripe never pays out on its own. `services/payout.ts#calculateWalletBalance` computes `availableToWithdraw = available − heldFunds`, and `validation/payout.ts` refuses a withdrawal above it. | **Exists, but leaky — see gap 1.** |
| Accepted-but-no-tracking escalation | `lambdas/reviewOrderHandler.ts` condition 0: `PENDING` order, shipment `CREATED` + `PRE_TRANSIT`, **84 h after `createdAt`** → `IN_REVIEW` | Exists; timing is measured from creation, not acceptance — see gap 3. |
| Un-accepted order sweep | `lambdas/cancelOrderHandler.ts`: `PENDING` + tracking `UNKNOWN` (never accepted), 84 h after creation → cancel + PI cancel + quantities restored | Unchanged by this plan. |

There is **no `acceptedAt` on `Order`** (the `acceptedAt` in the schema belongs to `Connection`).
Acceptance is only inferable from `shipment.trackingStatus !== UNKNOWN`.

## The gaps

### Gap 1 — the hold only starts once a tracking number exists (the real bug)

`calculateWalletBalance` holds funds for `PENDING` orders whose SHIPPO shipment has
`status: 'PENDING'`. But the shipment only becomes `PENDING` when the tracking number is entered;
between **accept** and **tracking number** it is still `CREATED`. So the window Skylar is worried
about — seller accepts (money captured and transferred to their connected balance), never ships —
is precisely the window where **nothing is held** and the seller can withdraw.

Also, since the [[buyer-service-fee|buyer service fee]] landed the seller only ever receives the
pre-tax base, but the hold still subtracts `order.total` (which now includes tax + fee), so it
over-holds by the buyer's tax and fee on every open order.

### Gap 2 — no reminder

Nothing tells the seller "add your tracking number or you won't be paid". The 84-hour review just
silently flips the order to `IN_REVIEW`.

### Gap 3 — `IN_REVIEW` is reached three different automatic ways, all clocked from `createdAt`

An order created day 0 and accepted at hour 80 is flagged `IN_REVIEW` at hour 84 — four hours after
accept, and before any reminder could fire. Skylar's read (2026-09-14): review should be rare and
mean "a human must look" — either a user asked, or a tracked package is genuinely overdue. The
accepted-no-tracking flag is neither; it goes.

## Design

### Schema (one additive migration)

```prisma
model Order {
  …
  acceptedAt                     DateTime?
  trackingReminderEmailedAt      DateTime?
  trackingReminderMessagedAt     DateTime?
}

enum SystemMessageType {
  …
  ORDER_TRACKING_REMINDER
}
```

- `acceptedAt` — set inside `acceptOrder`'s transaction for every order type (in-person and
  untracked too; it is a useful audit field everywhere). Null on all historical rows; the crons
  fall back to `createdAt` when it is null so old open orders are not orphaned.
- `trackingReminderEmailedAt` / `trackingReminderMessagedAt` — the once-only markers for the
  48 h email and the 72 h in-app message. Null = not sent.
- `ORDER_TRACKING_REMINDER` — new `SystemMessageType` (goes in the enums block, no inline comment).

All three columns are nullable so the migration is `ALTER TABLE … ADD COLUMN` ×3 plus
`ALTER TYPE "SystemMessageType" ADD VALUE`. Regenerate the Prisma client and `packages/types`
(`acceptedAt` on `OrderDto` is handy for mobile's timeline copy).

### Hold fix (`services/payout.ts#calculateWalletBalance`)

Hold every order that is still open and tracked, regardless of whether the tracking number has
been entered yet:

```
where: {
  sellerId,
  status: { in: ['PENDING', 'IN_REVIEW'] },
  shipments: { some: { shipmentAccountType: 'SHIPPO', status: { in: ['CREATED', 'PENDING'] } } },
}
```

and hold what the seller actually received, not the buyer's total:

```
heldPerOrder = total − tax − (serviceFee ?? 0)      // = subTotal + shipping + smallOrderFee
```

(Legacy orders with a null `serviceFee` were charged the fee seller-side, so `total − tax` slightly
over-holds by that fee — acceptable, and those orders are draining out.)

`IN_REVIEW` is included on purpose: an order under review has by definition not been delivered, so
its funds must stay locked until an admin resolves it.

The Swagger text on `GET /seller/:id/wallet` ("balance = PENDING + COMPLETED orders …") describes an
older formula; refresh it to say *Stripe balance minus open tracked orders*.

### Reminder cron (`lambdas/trackingReminderHandler.ts`, new)

Daily, alongside the other two. Two sweeps over the same base predicate — an accepted tracked
order the seller has not shipped:

```
status: PENDING
deliveryType: SHIP
shipments: some { shipmentAccountType: SHIPPO, status: CREATED, trackingStatus: PRE_TRANSIT, trackingNumber: null }
```

| Sweep | Extra predicate | Action |
| --- | --- | --- |
| **Email, 48 h** | `trackingReminderEmailedAt: null` and `acceptedAt < now − 48h` (fallback `createdAt` when `acceptedAt` is null) | stamp `trackingReminderEmailedAt = now`, then put EventBridge event `order.tracking.reminder.seller` `{ orderId, type }` — same shape as `sendOrderCanceledEvents` |
| **In-app, 72 h** | `trackingReminderMessagedAt: null` and `acceptedAt < now − 72h` (same fallback) | stamp `trackingReminderMessagedAt = now` and `createSystemMessage` to the seller in the same transaction: category `ORDER`, type `ORDER_TRACKING_REMINDER`, title "Add tracking to get paid", body "Your payout for this order is on hold until the carrier confirms delivery. Add the tracking number and carrier to release it.", `orderId` set, actor = the buyer (the system-message convention is "who this is about", see [[system-messages]]) |

For each hit re-check the active shipment via `getActiveShipment` inside the transaction (the
seller may have added tracking between the query and the write). Stamp *before* the email event
goes out so a Lambda retry cannot double-send; a lost event costs one reminder, a duplicate costs
seller trust. The in-app message and its stamp are one transaction, so they cannot diverge.

Why a third Lambda rather than a branch in `reviewOrderHandler`: the review cron *changes order
status*; this one only *notifies*. Keeping them separate keeps each handler's failure modes and
schedule independent (and matches the existing one-job-per-Lambda pattern in `serverless.yml`).
Schedule it at `cron(0 16 * * ? *)` (11am EST), an hour before the cancel sweep.

Because the cron is daily, "48 h" really means "the first run at least 48 h after accept", so the
email lands 2–3 days after accept and the message 3–4 days after; fine for a reminder. Put the
selection + stamping in `services/order.ts#sendTrackingReminders` (it writes, so it is a service)
and keep the Lambda thin.

### Consolidate `IN_REVIEW` (gap 3, widened by Skylar)

Skylar's rule: **an order goes to `IN_REVIEW` only when a user asks for it, or automatically when a
tracked order still hasn't been delivered 10 days after accept.** Nothing else. The
accepted-but-no-tracking escalation is dropped — the reminders above are the whole response to a
slow seller, and the funds hold is the enforcement.

Every `IN_REVIEW` touchpoint today, so the consolidation is checked against reality:

| Where | What it does today | After |
| --- | --- | --- |
| `PUT /order/:id/request-review` (`routers/order.ts`) | customer **or** seller flips any order to `IN_REVIEW`; no status guard server-side. Mobile gates the button: buyer 3 days after a refund request or after a declined refund (`OrderRefund`), either party 7 days after the last refund change (`OrderStatus`). | **Keep.** Add the missing guard: refuse when the order is `CANCELED`/`FAILED`/`CREATED`, or already `IN_REVIEW`. |
| `reviewOrderHandler` condition 0 — accepted, no tracking, 84 h after creation | → `IN_REVIEW` | **Delete.** Replaced by the 48 h / 72 h reminders. |
| `reviewOrderHandler` condition 1 — tracking added but still `PRE_TRANSIT` 7 days after creation | → `IN_REVIEW` | **Merge into one rule** (below). "Never scanned by the carrier" is just one way of "not delivered". |
| `reviewOrderHandler` condition 2 — not `DELIVERED` 10 days after creation | → `IN_REVIEW` | **Becomes the one automatic rule**, anchored on `acceptedAt`. |
| `POST /order`, `PUT /order/:id` | refuse `IN_REVIEW` as a target status | Keep. |
| `webhooks/shippo.ts` `DELIVERED` | sets `COMPLETED` without checking the current status — so a late delivery *does* pull an order out of review | Keep; it is the only automatic exit and the right one. |
| `services/payout.ts` | `IN_REVIEW` orders are **not** held today | Held (this plan). |
| nanza-admin `Orders` list / `OrderDetailModal` | orange badge, status filter, shows `order.resolution` | Unchanged. Note `resolution` and `inReview` are columns nothing writes and the API omits (`prismaHelpers` omit list), so the admin's "Resolution:" line can never render. Dead — candidate for a later drop, not this plan. |
| nanza-mobile `OrderStatus` | "This order is currently under review by the platform." row; `PayoutHistoryScreen` label | Unchanged. |

**There is no admin exit from `IN_REVIEW`.** No endpoint moves an order from `IN_REVIEW` to
`COMPLETED`/`CANCELED`/refunded; the admin app only reads. Today the ways out are the Shippo
`DELIVERED` webhook or a refund accepted through the normal refund flow (which sets its own
statuses). That is outside this plan, but it is why `IN_REVIEW` must stay rare — every one is a
manual Stripe-dashboard job.

The single automatic rule, in `reviewOrderHandler`:

```
status: PENDING
deliveryType: SHIP
shipments: some { shipmentAccountType: SHIPPO, trackingStatus: { notIn: [DELIVERED, RETURNED, FAILURE] } }
acceptedAt < now − 10 days     (fallback createdAt when acceptedAt is null)
```

`RETURNED`/`FAILURE` are excluded because the Shippo webhook already auto-refunds those. Untracked
and in-person orders never qualify (they complete on accept). Un-accepted orders never qualify
(the 84 h cancel sweep gets them first; `acceptedAt` is null and `trackingStatus` is `UNKNOWN`).

Per-order timeline after this plan:

```
accept ──48h──▶ reminder email ──72h──▶ in-app message ──10d──▶ IN_REVIEW if still undelivered
```

### Email (`nanza-email`)

- `serverless.yml` — add `order.tracking.reminder.seller` to the EventBridge rule's detail-type list.
- `handler/emailDispatcher.tsx` — new case: load the order, resolve the **seller's** email, render
  `OrderTrackingReminder`. Subject: **"Add tracking to get paid for order #…"**.
- `templates/OrderTrackingReminder.tsx` (new) — seller-only, built on `OrderCard` like
  `OrderInTransit`. Callout copy (plain, no hedging — this is the whole point of the email):

  > **Your funds are on hold until you add a tracking number.**
  > You accepted this order 2 days ago but it has no tracking yet. Nanza releases your **$X.XX**
  > for this order only after the carrier confirms delivery, and we can only see delivery through
  > the tracking number. Open the order in the Nanza app and add the tracking number and carrier.

  `$X.XX` is `sellerReceives(order)` from `utils/serviceFee.ts` (base, not the buyer's total).
  No "decline" escape hatch: an accepted order cannot be canceled by the seller (`cancel-order`
  requires tracking `UNKNOWN`), and Skylar confirmed that stays. A seller who cannot ship ends up
  in the admin review step.

### Mobile (`nanza-mobile`, small)

- `components/orders/OrderStatus` — seller's **Awaiting Tracking** step description gains one
  sentence: "Your payout for this order is on hold until the carrier confirms delivery."
- `screens/payouts/PayoutsScreen` — under "$X available now" show "$Y on hold for undelivered
  orders" (`balance − availableToWithdraw`). Optional; it explains the number the reminder
  email refers to.
- Accept-order alert for tracked orders (`getAcceptDescription`, `SHIPPO` branch) — today it says
  "you may be liable for shipping fees"; add "The buyer is charged now; your payout is released
  when the package is delivered."

No other client change. The API already returns everything mobile needs.

## Files (nanza-api)

| File | Change |
| --- | --- |
| `prisma/schema.prisma` | `Order.acceptedAt`, `Order.trackingReminderEmailedAt`, `Order.trackingReminderMessagedAt` (all `DateTime?`); `SystemMessageType.ORDER_TRACKING_REMINDER` |
| `prisma/migrations/2026091xxxxxxx_order_tracking_reminders/migration.sql` | three `ADD COLUMN` statements + one enum `ADD VALUE` |
| `services/order.ts` | `acceptOrder` writes `acceptedAt`; new `sendTrackingReminders` (both sweeps, stamps, event + system message) |
| `services/payout.ts` | hold predicate (`CREATED`+`PENDING` shipments, `PENDING`+`IN_REVIEW` orders) and held amount = seller's base |
| `lambdas/trackingReminderHandler.ts` | new daily cron |
| `lambdas/reviewOrderHandler.ts` | delete condition 0; replace conditions 1 + 2 with the single "undelivered 10 days after `acceptedAt`" rule |
| `routers/order.ts` | `request-review` refuses `CREATED`/`CANCELED`/`FAILED`/already-`IN_REVIEW` orders |
| `serverless.yml` | `trackingReminder` function + schedule |
| `routers/shipment.ts` | delete the stale "then payment capture" comment |
| `routers/seller.ts` | refresh the wallet endpoint's `@openapi` description |
| `packages/types` | regenerate (`acceptedAt`, `trackingReminderSentAt` on `OrderDto`) |

## Not doing

- **Moving the money later** (separate charges + transfer on delivery, or `on_behalf_of`
  + delayed transfer). The manual payout schedule plus the wallet math already gives the "not
  withdrawable" guarantee, funds stay in the seller's connected balance where a refund reversal
  can reach them, and it needs no change to `createPaymentIntent`. Revisit only if Stripe
  balance timing ever lets a seller withdraw held money.
- **Repeat reminders.** One email at 48 h, one in-app message at 72 h, then silence. The review
  step is the follow-up, and the admin can reach out from there.
- **Blocking accept until a tracking number exists.** Skylar's call: sellers accept first, ship
  within a couple of days. The hold + reminder is the enforcement, not the accept button.

## Decisions (voice session 2026-09-14)

- **Clock starts at accept, never before.** Un-accepted orders keep the 84 h-from-creation cancel.
- **Email at 48 h, in-app system message at 72 h**, each once. (Skylar chose the split so the
  two nudges arrive on different days.)
- **No decline path** in the copy — an accepted order cannot be canceled by the seller.

- **`IN_REVIEW` only via user request or one automatic undelivered-after-accept rule.** The
  accepted-no-tracking flag is deleted; the 7-day `PRE_TRANSIT` and 10-day not-delivered
  conditions collapse into one.

- **Automatic review at 10 days after accept.** Today's code has 7 (never scanned) and 10 (not
  delivered), both from creation. One number, from accept: 10 — domestic tracked mail rarely takes
  a week, and a shorter window mostly generates reviews that resolve themselves when the
  delivery scan arrives. Review is a manual job, so it should stay rare.

## Still open

Nothing — ready to build once Skylar has read this.

## Related

- [[payments]] — delivery-mode × payment-type model, application fee.
- [[buyer-service-fee]] — why the held amount is the pre-tax base.
- [[../INDEX|nanza-api]]
