---
tags: [nanza-api, nanza-mobile, nanza-email, nanza-web-app, solution-design, plan, payments, checkout, stripe]
status: BUILT 2026-09-11, uncommitted, pending Skylar QA — API + email + mobile done (web copy deferred, admin row not done). Migration `20260911120000_order_service_fee` to run before deploy.
---

# Buyer Service Fee — moving the 3% + $0.50 from the seller to the buyer

## Goal

Today nanza's platform cut (**3% + $0.50**, plus tax pass-through) is a Stripe *application fee*
deducted from the **seller's** payout. The buyer never sees it. This plan flips it: the buyer pays
a visible **Service fee** line at checkout, the seller receives exactly their item price plus
shipping, and the platform's application fee becomes *the buyer's fee + tax*. Nothing downstream of
a `PENDING` order (capture, shipping, Shippo, completion, collection sync) changes.

Decisions from the 2026-09-11 session:

- **Fee base includes shipping** (it's part of what Stripe processes). See the tax question below.
- **No fee on cash.** Cash never touches Nanza's money flow, so there is nothing to collect.
- **Label it "Processing fee."** First pass used "Service fee" (the legally safer marketplace
  wording, see *Naming & the surcharge question*); Skylar switched it to **"Processing fee"** on
  review (2026-09-11): the 3% + $0.50 exists to cover Stripe's 2.9% + $0.30 plus payout/withdrawal
  charges, and that is what customers are paying for. His call, made with the surcharge note in
  hand. Internal names stay `serviceFee` (column, DTO, helpers) — only labels and copy changed.

## Research: can Stripe keep the buyer fee out of its own processing cut?

**No.** Stripe's processing fee (2.9% + $0.30 in the US) is charged on the **full amount charged to
the card**, whatever it is made of. There is no parameter that excludes a platform fee, a surcharge,
or tax from Stripe's percentage. `application_fee_amount` only decides how much of the charge comes
back to the platform; Stripe's fee is then *debited from the platform's balance* ("Your platform pays
the Stripe fee after the application_fee_amount is transferred to your account" — destination
charges docs). `on_behalf_of` would make the seller the fee payer instead, which is the opposite of
what we want.

So the "discrepancy" Skylar was worried about — buyer pays $103.50 but our fee was computed on $100 —
is not a seller-side problem at all. The seller receives exactly $100 either way. The only leakage
is Stripe's 2.9% **on the fee itself**, which the platform absorbs (~$0.10 on a $100 order).

### Worked numbers ($100 item, $5 shipping, 6% tax)

| | Today (seller pays) | Proposed (buyer pays, fee on pre-fee total) |
|---|---|---|
| Buyer is charged | $111.30 | $115.14 |
| Service fee (3% + $0.50) | $3.84 (on $111.30) | $3.84 (on $111.30 — subtotal + shipping + tax) |
| `application_fee_amount` | $3.84 + $6.30 tax = $10.14 | $3.84 + $6.30 tax = $10.14 |
| Seller receives | $101.16 | **$105.00** |
| Stripe fee (2.9% + $0.30, paid by platform) | $3.53 | $3.64 |
| Platform nets after remitting tax | ~$0.31 | ~$0.20 |

The platform's take is already only ~0.1% + $0.20 above Stripe's cost (the "0% commission" model).
Moving the fee to the buyer costs the platform roughly **$0.10 + 0.087% of the order** — Stripe's
percentage applied to the fee. That is the whole cost of the change.

### The tax wrinkle (decision needed — recommendation below)

Today the 3% is computed on the **tax-inclusive** total (`Math.round(totalAmount * 0.03)` where
`totalAmount` already includes tax; mobile and email mirror this with `total * 0.03 + 0.50`). That
detail is what keeps the platform's margin positive: Stripe also takes 2.9% of the tax we collect and
remit, and the 3%-on-tax covers it.

If the buyer fee is computed on the **pre-tax** base (subtotal + shipping only), the platform eats
Stripe's cut on the tax: on the example above it nets ~$0.02, and in a 10% tax state it goes
negative. **Recommendation: keep the base exactly what it is today — everything the buyer pays except
the fee itself (subtotal + shipping + small-order fee + tax).** Copy can still truthfully say
"3% + $0.50 of your order total."

An exact gross-up (`fee = (0.03·T + 0.50) / (1 − 0.029)`) would restore today's margin to the cent
but turns the advertised rate into ~3.09% + $0.52. Not worth the copy. Absorb the dime.

### Naming & the surcharge question (flag, not legal advice)

A fee that is charged **only on card orders and waived for cash** is, to card networks and several
states, a *credit-card surcharge*: Visa caps surcharges at 3%, they may not exceed the merchant's
actual cost of acceptance, debit cards can't be surcharged at all, and CA / CT / ME / MA prohibit them
outright. A flat **marketplace service fee** (eBay, StockX, Mercari, TCGplayer all charge buyers one)
is a different thing and is not governed by those rules. Our cash path is genuinely off-platform for
money, so "no fee on cash" reads as "no fee where Nanza handles no payment," not "cheaper if you avoid
cards." Still: **call it "Service fee," never "Processing fee" or "Card fee," on every buyer-facing
surface**, and have someone with a legal hat glance at it before prod. The seller-facing label can stay
descriptive ("Buyer's service fee").

## How it will work

### Seller view: buyer's rows + Total → "Buyer's taxes & fees" (deduction) → "You receive"

Skylar's first review (2026-09-11) dropped the buyer's Total and the tax / fee deductions from
seller views entirely. His second pass (2026-09-14) brought the platform-bound money back as **one
combined line** so the seller can reconcile the buyer's total without being told they "paid" it:
every seller-facing summary (mobile `OrderSummary` seller variant, `OfferSummary` with
`showSellerReceives`, the seller confirmation email) shows the lines the seller receives (items,
small-order fee, shipping, handling, add-ons), then, with no gap, **the same platform-bound rows
the buyer sees** (Tax / Estimated tax, Processing fee) so the **bold Total** visibly adds up, a gap,
**Buyer's taxes & fees** as a deduction in parentheses (`tax + serviceFee`), then the **bold You
receive**: the full total on cash, `total − tax − serviceFee` on card. Total − deduction = You
receive under both fee models (a pre-move order shows no Processing fee row but its deduction
still carries the seller-paid fee). Buyer views are unchanged. `orderServiceFee` also treats a
never-invoiced order (no total) as fee-less rather than legacy.

### Buyer view: seller-bound lines, a gap, then tax + fee, then total

Skylar (2026-09-11, gap dropped 2026-09-14): the buyer's summary lists what the seller receives —
item lines, small-order fee, shipping rate, handling, add-ons — then the platform-bound
**Estimated tax / Tax** and **Processing fee**, then **Total**, as one unbroken group. Applied to
mobile `OfferSummary`, mobile `OrderSummary`, and the buyer email `OrderCard`.

### Money model

```
base       = subTotal + shipping + smallOrderFee          (what the seller earns, pre-tax)
tax        = isCash ? 0 : round(base × customer.taxRate)   (unchanged)
serviceFee = isCash ? 0 : round((base + tax) × 0.03) + 0.50
total      = base + tax + serviceFee                       (what the buyer pays; PI amount)

application_fee_amount = serviceFee + tax                  (in cents)
seller receives        = base
```

The fee is **not** taxed (tax is computed on `base`, before the fee). Rounding stays in cents via
`roundCents` at each step, same as today. The $1,000 subtotal cap is unchanged.

### Schema — one additive column

```prisma
model Order {
  smallOrderFee  Decimal? @db.Decimal(10, 2)
  serviceFee     Decimal? @db.Decimal(10, 2)
}
```

`serviceFee` is **null on every order that predates this change** and that null is the
legacy discriminator: every client renders a null-fee card order with today's seller-deducted math,
and an order with a value (including `0` for cash) with the new buyer-paid math. New cash orders are
written with `serviceFee = 0` so they are unambiguously "new model."

Follow the types-package steps in `nanza-api/CLAUDE.md` (`prisma generate` → generate/build types →
bump `packages/types` version) so `serviceFee` lands on `OrderDto` for mobile, email and admin.

### nanza-api

- **`services/invoice.ts`** — compute `serviceFee` after `tax`, fold it into `total`, persist it on
  the order (`serviceFee` written even when `0`). `createPaymentIntent` stops recomputing 3% from the
  total and uses `Math.round(Number(order.serviceFee) * 100) + taxAmount` as
  `application_fee_amount`. Keep a guard that throws if `serviceFee` is null on a non-cash order.
- **Refunds — `services/refund.ts`, called from `routers/refund.ts` and `webhooks/shippo.ts`** —
  the buyer gets **everything back but the service fee** (Skylar, 2026-09-11 review; tax *is*
  refunded — there is no taxable sale left, so holding it would be wrong). Mechanics on a
  destination charge: refund `total − serviceFee` with `reverse_transfer: true`, which pulls that
  same amount from the seller (their base **plus** the tax they never held), then hand the tax back
  to the seller with a separate application-fee refund (`applicationFees.createRefund` for exactly
  the tax, on the charge's `application_fee` — the PI is retrieved with `latest_charge` expanded).
  Net: buyer −fee, seller 0, platform +fee minus Stripe's processing fee (never returned). Legacy
  orders (null `serviceFee`) keep the full refund with `refund_application_fee: true`, which also
  fixes the seller-short-on-refund finding for them. Partial refunds are not a feature; the
  transaction row records the actual refunded amount.
- **Cancel paths** (`services/order.ts` cancel, `lambdas/cancelOrderHandler.ts`) — the PI is
  cancelled before capture; no change.
- **Offer engine** — `services/offer.ts` only prices `subTotal`; the invoice path prices the rest, so
  offers pick this up for free. Confirm `PUT /offer/:id/accept` (bid offers) returns the order with
  `serviceFee` populated.
- **OpenAPI JSDoc** on `POST /invoice` / `GET /order/:id` — add the field.
- **In-flight orders at deploy time** — a `PENDING` order already has its PI (and its
  application_fee_amount) fixed at creation; capture on accept uses it. Its `serviceFee` is null, so
  every client shows it the old way. No backfill.

### nanza-mobile

All fee math lives in two places today and both hard-code `total * 0.03 + 0.50`:

- **`utils/offerTotals.ts`** — add `serviceFee` to `OfferTotals` and `computeOfferTotals` (formula
  above; `0` for cash, `null` when tax is unknown because the viewer isn't the buyer). Add one shared
  helper, e.g. `sellerReceives(order | totals)`, that returns `total − serviceFee − tax` when
  `serviceFee != null` and the legacy `total − total × 0.03 − 0.50 − tax` when it is null. Every
  "You receive" reads from it.
- **`components/offers/OfferSummary`** — buyer rows: *Offer price · Estimated tax · Small order fee ·
  shipping rows · **Service fee** · Total*. Seller block: "Buyer's service fee" and "Tax" shown as the
  informational deductions from Total, "You receive" from the helper. Cash unchanged (full total).
- **`components/orders/OrderSummary`** — same rows for stored orders (read `order.serviceFee`);
  seller variant's "Processing Fee (3% + $0.50)" row becomes "Buyer's service fee" and the value
  comes from the stored field (legacy: computed).
- **`screens/offers/OfferMessageDetailScreen`** — the stored-order path that rebuilds totals from
  `order.smallOrderFee` must also pass `order.serviceFee`.
- **`screens/offers/MakeOfferScreen`** — live preview picks the new row up through `OfferSummary`.
- **`components/orders/OrderCard`** — collapsed "incl. … tax and shipping" line: add the fee to the
  wording ("incl. tax, shipping and service fee") or leave the placeholder logic as is; same no-jump
  rule as the in-person work (same rows, zeroed, never unmount).
- **`screens/payouts/PayoutHistoryScreen`** — verify it reads Stripe balance (it does, via
  `/payout`), not a computed fee. No change expected.

### nanza-email

- **`templates/SellerOrderConfirmation.tsx`** — `processingFee = total * 0.03 + 0.50` becomes: read
  `order.serviceFee`; when null fall back to the legacy formula. Rows: *Total · (blank) · Tax ·
  Buyer's service fee · You receive*. Same helper semantics as mobile.
- **`components/OrderCard.tsx`** (buyer's `CustomerInvoiceConfirmation`) — add a **Service fee** row
  between Tax and Total. (While there: it computes tax as `subTotal × seller.taxRate` instead of
  reading `order.tax` — see review findings.)
- The event/dispatcher wiring is untouched; the templates just read one more field off the DTO.

### nanza-web-app (marketing copy only) — DEFERRED

> Skylar, 2026-09-11: the website is being redone, so leave the landing copy alone for now. Kept
> here so the rewrite picks it up.

- `landing/pages/Pricing.tsx` — "0% commission … You only pay the standard card processing fee,
  3% + 50¢" is now wrong on both halves for sellers: sellers pay **nothing**; buyers pay a
  3% + $0.50 service fee. Rewrite as a seller pitch ("Keep 100% of your price and shipping. Buyers
  pay a 3% + $0.50 service fee at checkout.").
- `landing/components/KeepWhatYouEarnSection.tsx` — same message; also fix the stale
  "~3% + $0.30" figure (see findings).

### nanza-admin

- `app/Orders/OrderDetailModal.tsx` — add a Service fee row under Tax so operators can reconcile a
  buyer's charge against Stripe. Cosmetic; do last.

### Docs (vault)

When this lands: fold the money model into [[payments|Payments]] (replace the "fee stays seller-side
/ invisible to buyers" decision), update
[[../../nanza-mobile/solution-designs/payments|nanza-mobile — Payments]] for the new rows, add a
short nanza-email solution design for the order templates (there is none yet), then delete this plan
and its INDEX line.

## What was built (2026-09-11, same session)

- **nanza-api**: `Order.serviceFee` column + migration `20260911120000_order_service_fee`;
  `utils/serviceFee.ts` (rate + `calculateServiceFee`); `services/invoice.ts` computes and stores
  the fee (0 for cash) and the PI's `application_fee_amount` is now `serviceFee + tax`, throwing
  if the fee is missing; `refund_application_fee: true` in `routers/refund.ts` and
  `webhooks/shippo.ts`; Prisma client + types package regenerated (CI bumps the version from the
  registry on deploy). `tsc` + eslint clean.
- **nanza-email**: `utils/serviceFee.ts` (`orderServiceFee` with the legacy fallback,
  `sellerReceives`); seller confirmation reads the stored fee ("Service fee (3% + $0.50)" row);
  buyer `OrderCard` gains a "Service fee" row for buyer-paid orders and now reads `order.tax`
  instead of estimating from the seller's rate. Reads `serviceFee`/`paymentType` through a cast
  until the types package is bumped past 0.1.15. `tsc` clean.
- **nanza-mobile**: `utils/serviceFee.ts` (same helper + `ServiceFeePaidBy`); `OfferTotals`
  carries `serviceFee` + `serviceFeePaidBy`; `OfferSummary` and `OrderSummary` show a buyer
  "Service fee" row (last before Total, always rendered for buyer-paid orders, $0 on cash — the
  no-jump rule) and the seller block deducts the stored fee; the offer detail's stored-order path
  passes the fee through. Legacy orders (null fee) render seller-side exactly as before. eslint
  clean; `tsc` has only pre-existing unrelated errors.
- **Review round (same day)**: seller views reduced to *You receive* (mobile + email); refunds
  moved into `services/refund.ts` with the everything-but-the-fee policy and the tax hand-back;
  refund copy updated in the mobile accept alert and accepted panel, the two refund system
  messages, and the `OrderRefundAccepted` / `OrderFailed` emails ("everything except the service
  fee"). API/email `tsc` + eslint clean; mobile eslint clean, touched files type-clean.
- **Review round 3 (same day)**: customer-facing label renamed to **Processing fee** everywhere
  (20 strings, 11 files; identifiers unchanged); buyer summaries regrouped (seller-bound lines →
  gap → tax + fee → total) in mobile `OfferSummary`, `OrderSummary` and the buyer email card.
- **Review round 4 (2026-09-14)**: seller views regained the platform-bound money — bold Total
  after the seller's lines, then a "Buyer's taxes & fees" deduction in parentheses, then bold
  "You receive" (mobile `OrderSummary`, `OfferSummary`, seller email).
- **Review round 5 (2026-09-14)**: refund surfaces show the amount that moves *for that reader*.
  Customer (`OrderRefundAccepted`, `OrderFailed`, mobile refund panel copy): Total, then
  "Processing fee" in parentheses, then **Your refund** = `total − serviceFee`. Seller
  (`OrderRefundRequest`, `OrderFailed`, the mobile refund-request step's "You refund" line and the
  accept alert): Total, then "Buyer's taxes & fees" in parentheses, then **You refund** =
  `total − tax − serviceFee` — exactly what they received; the tax comes back to the buyer from
  the platform (the application-fee refund in `services/refund.ts`), never from the seller.
  Implemented via a `refund` flag on the shared email `OrderCard`. Pre-move orders refund the
  customer in full.
- **Review round 6 (2026-09-14) — no negative seller balances.** `validation/refund.ts#
  assertSellerCanCoverRefund` reads the connected account's Stripe balance (available + pending,
  USD) and refuses the refund when it is below what the seller would return (`orderRefundPlan(...)
  .sellerReturns` = what they received for the order). `services/refund.ts` now hands the tax back
  (application-fee refund) **before** reversing the transfer so the balance never dips below zero
  in between. Seller-accept surfaces the message to the seller ("Your balance … does not cover
  this refund …"); the mobile accept alert no longer mentions going negative. **Caveat:** the
  Shippo lost/returned-package auto-refund applies the same check and *throws* — the webhook
  transaction rolls back (order stays as it was, no refund row), the error is logged as
  `AUTO_REFUND_SELLER_BALANCE_INSUFFICIENT`, and Shippo will retry. An underfunded seller on a
  lost package therefore needs an operator; decide later whether the platform should front that
  refund instead.
- **Not done**: admin `OrderDetailModal` row (cosmetic); web landing copy (deferred).

## Build order

0. **Refund fix (standalone, ship first).** Add `refund_application_fee: true` beside
   `reverse_transfer: true` in `routers/refund.ts` and `webhooks/shippo.ts`. Independent of the fee
   move; fixes the seller-short-on-refund problem under today's model too. Skylar to confirm the
   symptom in the Stripe dashboard first (see finding 2).
1. **API** — schema column (Skylar runs the migration), invoice math + PI application fee, refunds
   `refund_application_fee`, types package regenerate + version bump. Deployable on its own: clients
   that don't know the field keep working, they just show the old seller-side math on new orders
   until updated (the seller's "You receive" would read ~3% low for those orders in the gap — keep
   the gap short).
2. **Mobile** — `offerTotals` + the shared helper, then the two summaries and the offer detail path.
3. **Email** — seller template + buyer OrderCard row.
4. **Admin row** (web copy deferred — site is being redone).
5. **Docs** — fold into the solution designs above.

## Review findings surfaced while mapping this (fee-related)

1. **Fee math is duplicated five times** with no shared source: `invoice.ts` (authoritative), mobile
   `OrderSummary`, mobile `OfferSummary`, email `SellerOrderConfirmation`, and the web calculator
   constant. Persisting `serviceFee` on the order and reading it everywhere is the fix; the only
   remaining duplicate is the live preview in `offerTotals.ts`, which already mirrors the invoice
   math on purpose.
2. **Refunds probably leave the seller short today.** Both refund paths pass `reverse_transfer:
   true` but not `refund_application_fee`. Mechanics, using the worked example: Stripe transfers the
   **full** charge ($111.30) to the seller's connected account, then pulls the application fee
   ($10.14 = fee + tax) back to the platform, so the seller nets $101.16. A full refund with
   `reverse_transfer` reverses the **whole** $111.30 transfer, and by default the platform keeps the
   application fee. Net: the seller is $10.14 in the hole, the platform is up $10.14, and it still
   owes tax on a sale that no longer exists. Fix: `refund_application_fee: true` on both
   `stripe.refunds.create` calls pushes the fee + tax back to the seller so everyone nets to zero
   (the platform still eats Stripe's processing fee on refunds — Stripe never returns it, and that
   is already true today). Partial refunds aren't used (`refundAmount = paymentIntent.amount`), so
   the proportional case doesn't arise. **Verify first:** open any refunded order's connected
   account in the Stripe dashboard and check whether an *application fee refund* appears next to the
   *transfer reversal*. If it doesn't, the bug is confirmed. Skylar agreed on 2026-09-11 to fold the
   fix into this plan as step 0.
3. **Buyer email tax row can disagree with the app.** `components/OrderCard.tsx` computes
   `tax = subTotal × seller.taxRate`; the API computes tax on subtotal + shipping + small-order fee
   at the **customer's** rate and stores it on `order.tax`. Read `order.tax`.
4. **Landing copy disagrees with itself**: `KeepWhatYouEarnSection` says "~3% + $0.30",
   `Pricing.tsx` says "3% + 50¢". Both get rewritten by this plan anyway.
5. **Small-order fee's rationale goes away.** The $1 minimum exists so a seller doesn't lose money on
   a sub-$1 card sale under the seller-paid fee. With the buyer paying the fee the seller can't lose;
   the platform still nets ~$0.19 on a $0.25 card sale. Candidate for removal in a follow-up — kept
   out of this plan to keep the diff to the fee move.

## Open decisions

- **Fee base: tax-inclusive (recommended) or pre-tax?** See *The tax wrinkle*. Default to
  tax-inclusive unless Skylar objects.
- **Legal glance** at "Service fee" + cash exemption before prod (see *Naming*).
- Whether the seller's summary should keep the informational deduction rows at all now that they no
  longer reduce the payout, or just show *Total · You receive*. Plan keeps them — sellers are used to
  them and they explain why the buyer's total is bigger than what lands.

## Sources

- [Stripe — Create destination charges](https://docs.stripe.com/connect/destination-charges) (collect
  fees, flow of funds, refund application fees)
- [Stripe — Platform pricing tool](https://docs.stripe.com/connect/platform-pricing-tools)
- [Stripe — App fees (marketplace)](https://docs.stripe.com/connect/marketplace/tasks/app-fees)
- [Visa — U.S. Merchant Surcharge Q&A](https://usa.visa.com/content/dam/VCOM/global/support-legal/documents/merchant-surcharging-qa-for-web.pdf)
- [Stax — State-by-state surcharge laws](https://staxpayments.com/blog/credit-card-surcharge-guidance/)

## Related

- [[payments|Payments]] · [[offer-engine|Offer Engine]] · [[../INDEX|nanza-api]]
