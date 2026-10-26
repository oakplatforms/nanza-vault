---
tags: [nanza-api, nanza-mobile, nanza-email, nanza-web-app, solution-design, plan, offers, payments, legal]
status: plan, 2026-10-25 (Skylar, voice). All three parts built 2026-10-25 across api + mobile + email + web (all uncommitted; migration `20261025130000_order_service_fee_paid_by` to run, types `0.1.134` to publish then mobile `types:update`); pending Skylar QA + legal read of the policy. Decisions 2–6 answered; only the row wording (1) is open, defaulting to the spoken words.
---

# Offers — who pays the processing fee, the Seller & Refund Policy, and the inbox detail restyle

Three changes to the offer UX, all scoped to this release (Skylar, 2026-10-25, voice). Listed
big-first because that is the build order.

1. **Who pays the processing fee** — a new row in the offer's checkout card, under Payment:
   *Who pays the processing fee?* with two choices, **I'll pay** (default) and **You'll pay**.
   Buyer pays = today's model (the 3% + $0.50 rides on the buyer's charge). Seller pays = the
   original design (the fee comes out of the seller's payout), and the offer carries a note
   telling the seller they are covering it. The fee is a *term of the offer*: a buyer can
   strengthen an offer by covering it, a seller can decline and message "I'll accept if you
   cover the fee", mirroring how in-person deals go.
2. **Seller & Refund Policy** — the "I agree to Nanza's Terms of Service" checkbox at the end
   of the offer becomes "I agree to Nanza's Seller & Refund Policy", linking to a new legal
   document (web page + native screen, like the Terms) that says refunds are at the seller's
   discretion, Nanza is not liable for a bad trade, what Nanza can and cannot see per delivery
   type (tracked / untracked / in person), and how to report a bad actor (zero tolerance).
3. **Inbox offer detail restyle** — the offer detail opened from the inbox's Offers tab adopts
   the Make Offer line card's new look (the 2026-10-23 Figma redesign: hero thumb, "/" between
   tags, no pink badge, no surface card) instead of the older order-card lockup it still uses.
   Reverses the handoff's Q3 answer ("make-offer card only").

Builds on [[buyer-service-fee|Buyer Service Fee]] (the fee model and the stored
`Order.serviceFee`) and [[offer-engine|Offer Engine]] (the draft → send → accept flow).

---

## Part 1 — Who pays the processing fee

### Money model

Today (buyer pays), from `services/invoice.ts`:

```
base       = subTotal + shipping + smallOrderFee
tax        = isCash ? 0 : round(base × customer.taxRate)
serviceFee = isCash ? 0 : round((base + tax) × 0.03 + 0.50)
total      = base + tax + serviceFee            ← the buyer's charge (PI amount)
application_fee_amount = serviceFee + tax
seller receives        = base
```

With the payer as a term:

```
serviceFee = isCash ? 0 : round((base + tax) × 0.03 + 0.50)      (same amount either way)
total      = base + tax + (paidBy === BUYER ? serviceFee : 0)
application_fee_amount = serviceFee + tax                         (unchanged)
seller receives        = total − tax − serviceFee
                       = base            when BUYER pays
                       = base − fee      when SELLER pays
```

Seller-pays is *exactly* the pre-2026-09 model, so the numbers line up with every legacy order
in history (fee on the tax-inclusive total, deducted from the transfer). Nothing about
`createPaymentIntent` changes except that `total` no longer always contains the fee: the
`application_fee_amount` formula is already `serviceFee + tax`. Payouts (`services/payout.ts`
computes `total − tax − serviceFee`) are unchanged — that identity holds under both models.

Cash is fee-less and the payer is meaningless; the order stores the default (`BUYER`) and the
row reads "None (cash)".

### Schema — one enum, one column

```prisma
enum ServiceFeePayer {
  BUYER
  SELLER
}

model Order {
  serviceFee        Decimal?        @db.Decimal(10, 2)
  serviceFeePaidBy  ServiceFeePayer @default(BUYER)
}
```

- Lives on **`Order`**, not `Offer`: the invoice prices the order, refunds/payouts read the
  order, and the term must survive the offer (an accepted offer is just an order afterwards).
  Same home as `deliveryType` / `paymentType`, which are also offer terms stored on the draft.
- The enum goes in the enums block (house rule), no inline comments in the schema.
- **Legacy discriminator stays `serviceFee == null`.** Every pre-2026-09 order reads as
  seller-paid-but-unstored exactly as today; the new column's `BUYER` default on those rows is
  never consulted because the null check comes first in every client helper. A new seller-paid
  order is `serviceFee = <amount>` + `serviceFeePaidBy = SELLER` — the fee is stored either way.
- Migration `20261025130000_order_service_fee_paid_by` (additive; Skylar runs it). Types
  package → `0.1.132` after `prisma generate` + `generate:types` / `build:types`.

### nanza-api

- **`validation/offer.ts#parseCheckoutFields`** — parse `serviceFeePaidBy` (`BUYER` | `SELLER`,
  default `BUYER`) on `POST /offer` and `PUT /offer/:id` beside `deliveryType` / `paymentType`.
  Cash normalises to `BUYER` silently (no fee to assign). **`validateOfferAcceptBody` ignores
  it**: the bidder's checkout on accept sets delivery, payment and shipping, never the fee
  payer — that is the sender's term, accepted or declined as sent.
- **`services/offer.ts`** — `createOfferDraft` / `updateOfferDraft` write it onto the order
  draft; `sendOffer` and `acceptOffer` need no change (the invoice reads it off the order).
  `offerInclude` already returns the order, so the DTO carries it.
- **`services/invoice.ts`** — `total` excludes the fee when `SELLER`; store `serviceFee` as
  today. `createPaymentIntent` untouched (guard + formula already right).
- **`services/refund.ts`** — see *Refund policy* below (decision 3).
- **`services/offerStatus.ts`** — the OFFER_RECEIVED body gains one sentence when the sender
  asked the recipient to cover the fee: listing offer → "You'd cover the $3.84 processing fee."
  (the seller is being asked), bid offer with SELLER → nothing extra for the bidder (the seller
  sweetened: say "They'll cover the processing fee."). Buyer-pays is the default and gets no
  sentence. Mobile `utils/offerCopy.ts` mirrors this file — update both.
- **OpenAPI JSDoc** on `POST /offer`, `PUT /offer/:id`, `GET /order/:id` — add the field.
- **`validation/refund.ts#assertSellerCanCoverRefund`** — `orderRefundPlan.sellerReturns`
  already equals what the seller received; with the stored fee it is exact for seller-paid
  orders (the `legacyFeeCents` approximation only remains for null-fee history).

### nanza-mobile

- **`utils/serviceFee.ts`** — `orderServiceFee(order)`: `serviceFee != null` →
  `paidBy = order.serviceFeePaidBy === 'SELLER' ? 'seller' : 'buyer'` (the `ServiceFeePaidBy`
  type already exists); null → legacy `'seller'` computed, as today. `sellerReceives` unchanged.
- **`utils/offerTotals.ts`** — `OfferTotalsInput.serviceFeePaidBy`; `computeOfferTotals` keeps
  the fee out of `total` when `'seller'` and reports `serviceFeePaidBy` instead of the
  hard-coded `'buyer'`.
- **`components/offers/OfferCheckoutRows`** — fifth row, last, after Payment: label
  **Processing fee**, value **I'll pay** / **You'll pay** (muted **None** on cash). Expanded
  panel: `panelTitle` "Who pays the processing fee?", two `RadioField`s —
  *I'll pay* ("3% + $0.50 added to your total") and *You'll pay* ("Comes out of what the
  seller receives — ask before you assume"). Switching to cash resets to I'll pay, the way
  leaving In Person resets cash to card. Pure local state, written on send like the rest.
- **`screens/offers/MakeOfferScreen`** — listing variant: new `serviceFeePaidBy` state →
  `computeOfferTotals` + `POST /offer` body. **Bid variant (decided 2026-10-25, Skylar):** the
  seller offering to sell on a bid sets it too — a one-row card with the same row, worded from
  the seller's seat, *You'll pay* (bidder pays, default) / *I'll pay* (seller covers) — the
  term belongs to whoever sends, in both directions.
- **`components/offers/OfferCheckoutSummary`** (read-only, the recipient's view) — fifth row
  **Processing fee**: from the viewer's seat, "You pay" / "Buyer pays" for a seller, "You pay" /
  "Seller pays" for a buyer; expanded help "The 3% + $0.50 comes out of what you receive." /
  "…is added to your total." This row is *the note* Skylar asked for — it sits where the seller
  already reads the buyer's terms before Sell Now.
- **`components/offers/OfferSummary`** — buyer-mode rows: the **Processing fee** row renders
  only when `serviceFeePaidBy === 'buyer'` (already the case). Seller block when the *seller*
  pays: the deduction can't be called "Buyer's taxes & fees" any more — show two parenthesised
  lines, **Tax** and **Processing fee**, then **You receive**. Buyer-paid rendering unchanged.
- **`components/orders/OrderSummary`**, `OrderStatus`, `OrderRefund` — read `paidBy` through
  the helper; same two-line deduction when seller-paid. `OfferMessageDetailScreen`'s stored-
  order path already passes `serviceFeePaidBy` from the helper.
- Mobile offer copy (`utils/offerCopy.ts`) — mirror the API sentence.

### nanza-email

- **`utils/serviceFee.ts`** — `orderServiceFee` returns `paidBy` from `serviceFeePaidBy`
  (legacy null stays `isLegacy`); `sellerReceives` unchanged.
- **`templates/SellerOrderConfirmation.tsx`** — seller-paid: no "Processing fee" row among the
  buyer's lines (the buyer didn't pay it); deduction block becomes *Tax* + *Processing fee* in
  parentheses, then **You Receive**. The offer-received variant adds the one-line note under
  the respond-within-3-days panel: "The buyer has asked you to cover the processing fee."
- **`components/OrderCard.tsx`** (buyer) — `showServiceFee` = buyer-paid and > 0; the refund
  variant's **Your refund** = full total when the seller paid the fee (see decision 3).
- **`templates/BidOfferReceived.tsx`** — one line when the seller is covering the fee.

### Refund policy under seller-pays (decision 3)

Today's rule (buyer pays): *everything back but the processing fee* — the platform keeps the
buyer's fee because Stripe never returns its cost; the tax is handed back to the seller first,
then the transfer is reversed, and `assertSellerCanCoverRefund` refuses when the seller's
balance can't cover what they received.

Seller-paid orders have two consistent options:

| | A — everyone nets to zero (recommended) | B — the payer loses the fee (symmetric) |
|---|---|---|
| Mechanics | the legacy path: full refund, `reverse_transfer: true`, `refund_application_fee: true` | tax hand-back, then refund the full charge with `reverse_transfer: true`, fee **not** refunded |
| Buyer | full total back | full total back |
| Seller | 0 | −fee |
| Platform | −Stripe's processing cost (~2.9% + $0.30) | +fee − Stripe's cost |
| Balance guard | balance ≥ base − fee (what they received) — passes on a lone sale | balance ≥ base — **fails for a seller whose only balance is this sale**, needs an operator |

Recommend **A**: it is the already-built legacy path, and B blocks the most common refund (a
seller's first and only sale) behind an operator. The cost to the platform is a few dimes per
refund on seller-paid orders only. Skylar to confirm.

### Why not on the Offer, and why not a free-form split

- A percentage split or "split 50/50" was not asked for; two choices are what in-person trades
  actually negotiate ("you cover shipping, I'll cover the fee"). Keep the enum two-valued.
- Putting it on `Offer` would mean the invoice reaches through `order.offer` for pricing and
  would leave cart-era orders (still possible until the cart is retired) without a payer. On
  `Order` with a default it is uniform.

---

## Part 2 — Seller & Refund Policy

### What changes in the product

- **`MakeOfferScreen`** footer checkbox → "I agree to Nanza's **Seller & Refund Policy**",
  linking to the new native screen. (Found while mapping: the current link navigates to
  `ConditionsOfUse`, whose screen renders the **Privacy Policy**, not the Terms — the Terms are
  `TermsOfService`. The new link fixes that by replacing it.) Listing offers only, as today
  (decision 4 asks about the seller's side).
- **nanza-mobile** — `components/legal/SellerRefundPolicyBody` (the document, no chrome, the
  `TermsOfServiceBody` pattern) + `screens/legal/SellerRefundPolicyScreen` (thin host), route
  `SellerRefundPolicy` in `types/navigation.ts` + `AppNavigator` with deep link
  `seller-refund-policy`. `LegalFooter` gains the third link.
- **nanza-web-app** — `pages/legal/SellerRefundPolicyCopy.tsx` inside `LegalArticle`, path
  `/seller-refund-policy` in `pages.ts`'s legal list and `AppLayout`'s copy map, beside
  `/terms` and `/privacy-policy`. The web page is the source of truth; the native body mirrors
  it, same as the Terms.
- **Terms of Service** — one cross-reference sentence in §3 (Service description) pointing at
  the policy, web + native. No other Terms change.
- No API change. No acceptance record (decision 5).

### The document — first draft

Written to match what the code actually does today (84 h accept window, capture on accept,
the tracked-package hold, the Shippo lost/returned auto-refund, `IN_REVIEW` after 10 days
undelivered, the refund flow where the fee is not returned). Legal review before prod; this is
not legal advice.

> **Seller & Refund Policy**
> Effective date: [release date]
>
> This policy explains how buying and selling works on Nanza, who is responsible for what,
> and what happens when a trade goes wrong. It is part of our Terms of Service. By sending an
> offer, accepting an offer, or completing a sale on Nanza you agree to it.
>
> **1. Nanza is the venue, not the seller.** Nanza (Oak Platforms, Inc.) provides the place
> where buyers and sellers find each other, agree on a price, and pay. We are not a party to
> any trade. We do not own, inspect, grade, store, or ship the cards. The seller is
> responsible for describing the card accurately and shipping what was agreed; the buyer is
> responsible for reading the listing, the condition, and the delivery method before offering.
>
> **2. Refunds are at the seller's discretion.** Unless this policy says otherwise, whether a
> refund is given is the seller's decision. A buyer requests a refund in the app; the seller
> accepts or declines. Nanza does not issue refunds on a seller's behalf and does not
> guarantee any trade.
>
> **3. What Nanza can see.** How much we can verify depends on how the order was delivered.
> - *Tracked packages.* The carrier reports every scan to us. We can confirm when a package
>   shipped, where it is, and when it was delivered. Payment to the seller is held until the
>   package is on its way with a tracking number, and a package the carrier reports as lost or
>   returned is refunded automatically.
> - *Untracked (plain envelope).* We see only that the seller marked it shipped. We cannot
>   confirm delivery, and a lost envelope cannot be verified by anyone. Untracked is the
>   cheapest option and the buyer accepts that risk by choosing it.
> - *In person.* We see nothing about the handoff. In-person trades are between the two of
>   you; Nanza only records the order. Cash trades never touch Nanza's payment system.
>
> **4. Orders Nanza steps into.** We review an order ourselves when a buyer or seller asks us
> to, and automatically when a tracked order is still undelivered 10 days after the seller
> accepted it. An order the seller has not accepted within 3 days is cancelled and the buyer
> is not charged. Review means we look at the order, the tracking, and the messages between
> you; it does not mean we promise a refund.
>
> **5. How a refund is paid.** When a refund is accepted, the buyer receives everything they
> paid except the processing fee (Nanza's payment provider does not return its cost), and any
> sales tax is returned. If the seller agreed to cover the processing fee on that order, the
> buyer is refunded in full. Refunds go back to the card that paid. Cash trades cannot be
> refunded through Nanza.
>
> **6. Reporting a problem.** If something is wrong with a trade — the wrong card, a
> misdescribed condition, no shipment, no payment at a meet-up — message the other party
> first; most problems are a misunderstanding. If that fails, report the order or the
> account in the app or email support@nanza.app. Tell us the order, what you expected, and
> what you received, with photos where you can.
>
> **7. Bad actors.** We take every report seriously and look into it. Every order, offer,
> shipment, and message on Nanza is on record, so a pattern is easy to see. We have zero
> tolerance for fraud: an account that takes payment and never ships, knowingly sends the
> wrong or a misdescribed card, or abuses refunds can be suspended or removed from Nanza.
> Honest mistakes are not fraud — we look at the whole picture, including how both sides
> handled it, before acting.
>
> **8. What we are not responsible for.** To the fullest extent the law allows, Nanza is not
> liable for losses from a trade between users — including cards that arrive damaged, late,
> misdescribed, or not at all, counterfeit or altered cards, lost untracked mail, and
> anything that happens at an in-person meeting. The limitation of liability in our Terms of
> Service applies to every trade.
>
> **9. Changes.** We may update this policy. The effective date at the top is the version you
> are agreeing to; material changes will be posted in the app.

---

## Part 3 — Inbox offer detail restyle

`OfferMessageDetailScreen` renders a listing offer's lines as `OfferLineLockup` in its
default look — the order-card portrait thumb with the pink quantity badge — inside a surface
card (`offerStyles.card`). `MakeOfferScreen`'s `OfferLineCard` moved to the 2026-10-23 Figma
look: `EntityHeroThumb` landscape tile with blurred backdrop, "/" between tags, no badge, on
the page gutter (no card), the glass terms row beneath. Skylar now wants the detail to match.

- Pass `heroThumb` from the detail's line loop and drop the `offerStyles.card` wrapper (the
  handoff's Q5: the line sits on the 20 pt gutter). The note under the lines keeps its text
  style, left-aligned under the last line.
- **The quantity has to go somewhere** once the badge goes (decision 6): the make-offer card
  carries it in the stepper. Proposed: a read-only twin of the glass terms row — the hugging
  price box showing the agreed unit price as static text, and the count on the right as
  "×2" in the same meta type — so the detail reads like the card the sender just filled in.
  Alternatively, keep only the count ("2 of 3 available" is meaningless here) and leave the
  price to the summary as today.
- Bid offers keep `OfferListingCard` (one card, photo + condition): it is already its own
  design and the request was about matching the line card.
- Update the "offer detail keeps the order-card thumb and badge" comments in `OfferLineLockup`
  / `OfferLineCard` and flip the 2026-10-23 handoff's Q3 answer in its log.
- `/ui-review` before finishing; no new styles beyond what `OfferLineCard` already uses.

---

## Decisions needed (Skylar)

1. **Row wording.** "I'll pay" / "You'll pay" as spoken, with the row label *Processing fee*
   and the panel question *Who pays the processing fee?* — or the row label as the question?
2. ~~Bid offers too?~~ **Decided 2026-10-25 (Skylar, typed mid-session): yes.** A seller
   offering to sell on a bid picks I'll pay / You'll pay with their offer, same as a buyer on a
   listing. Default is always "the buyer pays".
3. ~~Refunds on seller-paid orders~~ **Decided 2026-10-25 (voice): A.** Everyone nets to
   zero on a seller-paid refund (the legacy path); the platform eats Stripe's cost.
4. ~~Seller-side agreement~~ **Decided 2026-10-25 (voice): buyer-only**, as today — the
   Terms already bind the seller and the policy's refund discretion is in their favour.
5. ~~Record acceptance?~~ **Decided 2026-10-25 (voice): client gate only**, same as the
   Terms; the `Agreement` / `AgreementAcceptance` model stays available if that changes.
6. ~~Quantity on the restyled detail line~~ **Decided 2026-10-25 (voice): read-only price
   box + "×N"** under each line.

## What was built (2026-10-25)

- **nanza-api (Part 1)**: `ServiceFeePayer { BUYER SELLER }` enum + `Order.serviceFeePaidBy`
  (default `BUYER`), migration `20261025130000_order_service_fee_paid_by`; `parseCheckoutFields`
  parses it on the draft (cash normalises to `BUYER`), `acceptOffer` never writes it; draft
  create/update store it; `createInvoiceWithTransactions` keeps the fee out of `total` when
  `SELLER` (application fee formula unchanged); `orderRefundPlan` treats `SELLER` like legacy
  (full refund, application fee back to the seller, `sellerReturns` uses the stored fee);
  `OFFER_RECEIVED` body gains "You’d cover the processing fee." (listing) / "I’ll cover the
  processing fee." (bid) when the seller pays; OpenAPI docs on create + accept. Prisma client
  and types regenerated, `packages/types` → `0.1.134` (the registry already holds `0.1.133`). `tsc` + eslint clean.
- **nanza-mobile (Part 1)**: `ServiceFeePayer` type + `OfferDraftPayload.serviceFeePaidBy`;
  `orderServiceFee` reads the payer off the order (`feePayerFromOrder`), `computeOfferTotals`
  takes `serviceFeePaidBy` and keeps the fee out of `total` when the seller pays; new
  `OfferFeePayerPanel` ("Who pays the processing fee?", I'll pay / You'll pay from the viewer's
  seat, read-only + cash variants) under a fifth **Processing fee** row in `OfferCheckoutRows`
  (`feeOnly` renders that row alone for the bid-offer send; the bidder's accept shows it
  read-only); `OfferCheckoutSummary` gains the same row from the viewer's seat (the seller's
  note before Sell Now); `OfferSummary` / `OrderSummary` split the seller's deduction into
  *Sales tax (buyer pays)* + *Processing fee (you pay)* when the seller covers it; `MakeOfferScreen` holds
  the term for both variants and resets it on cash; `offerCopy` mirrors the API sentence for
  sender and recipient. Local typecheck used the API's built `dist` copied into
  `node_modules/@oakplatforms/types` — run `types:update` once `0.1.134` is published.
  `tsc` + eslint clean (two pre-existing hook warnings).

- **nanza-email (Part 1)**: `orderServiceFee` returns `paidBy` (`feePayerFromOrder` reads the
  column through the existing cast); seller confirmation shows the buyer's *Processing fee* row
  only when the buyer pays, splits the deduction into *Sales tax (buyer pays)* + *Processing fee (you pay)*
  when the seller covers it, and the offer-received panel adds "The buyer has asked you to cover
  the 3% + $0.50 processing fee…"; buyer `OrderCard` hides the fee row on seller-paid orders
  (refund math already right) and splits the seller's refund deduction the same way;
  `BidOfferReceived` adds "The seller is covering the … processing fee" when so. `tsc` clean.
- **Part 2 — Seller & Refund Policy**: the web page lives at the **existing**
  `/refund-and-return-policy` URL (it was a placeholder "Refund and Return Policy" kept routed
  for links in the wild; now titled *Seller & Refund Policy*, back in the footer, indexed) —
  `pages/legal/SellerRefundPolicyCopy.tsx` in `LegalArticle`, wired in `AppLayout`'s body map;
  Terms §3 cross-links it (web + native). Mobile: `components/legal/SellerRefundPolicyBody`
  (mirror of the page, `TermsOfServiceBody` pattern) + `screens/legal/SellerRefundPolicyScreen`,
  route `SellerRefundPolicy` with deep link `refund-and-return-policy` (navigator edit, approved
  by the "continue" instruction); the Make Offer checkbox now reads "I agree to Nanza's Seller &
  Refund Policy" and opens it (it used to open `ConditionsOfUse` = the privacy policy). Effective
  date set to 2026-10-25 — change on release. No acceptance recorded (decision 5).
- **Part 3 — inbox detail restyle**: `OfferMessageDetailScreen` renders each listing-offer line
  as the make-offer card — `OfferLineLockup heroThumb` + a new **read-only** glass
  `OfferTermsRow` (`readOnly`: agreed unit price in the box, "×N" count, no stepper) in
  `lineCard` / `lineTerms` on the page gutter; the surface card is gone (`detailLines` now
  carries the lg bottom gap, new `detailLinesNote` for the sender's note). Bid offers keep
  `OfferListingCard`. Lockup comment updated; handoff log flipped. `tsc` + eslint clean.

## Build order

1. **API** — enum + column + migration, `parseCheckoutFields`, invoice total, refund branch
   (per decision 3), offer-received sentence, JSDoc; `prisma generate` → types `0.1.132`;
   deploy dev.
2. **Mobile, fee** — helpers, `OfferCheckoutRows` row + panel, `MakeOfferScreen` state (listing +
   bid variants), `OfferCheckoutSummary` row, `OfferSummary` / `OrderSummary`
   seller-paid deduction lines, `offerCopy` mirror; `types:update`.
3. **Email** — helper `paidBy`, seller template rows + note, buyer card, bid-offer line.
4. **Policy** — web page + route, native body + screen + route + footer link, checkbox swap,
   Terms §3 cross-reference.
5. **Detail restyle** — `heroThumb` on the detail lines, card wrapper off, read-only terms row,
   comments + handoff log.
6. **Docs** — fold Part 1 into [[buyer-service-fee|Buyer Service Fee]] → [[payments|Payments]]
   and [[offer-engine|Offer Engine]]; Part 2 into a short legal/policies note in `_shared`;
   Part 3 into the mobile offer-engine design; then delete this plan and its INDEX line.

## Related

- [[buyer-service-fee|Buyer Service Fee]] · [[payments|Payments]] · [[offer-engine|Offer Engine]]
- [[../../nanza-mobile/solution-designs/offer-engine|Offer Engine — Mobile]] ·
  [[../../nanza-figma/handoffs/2026-10-23-makeoffer-listing-card|Make Offer line card handoff]]
- [[../INDEX|nanza-api]]
