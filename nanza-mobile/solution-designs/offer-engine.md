---
tags: [nanza-mobile, solution-design, plan, offers, cart, checkout]
status: built 2026-08-27 (uncommitted) — needs @oakplatforms/types ≥ 0.1.97; API half built the same day
---

# Offer Engine — Mobile

The mobile half of [[../../nanza-api/solution-designs/offer-engine|nanza-api — Offer Engine]].
The cart goes away; every purchase is an **offer**: a buyer offers on a listing, or a seller
offers one of their listings to a bidder. Both land in the inbox's Offers tab, where the
recipient accepts or declines. Requires `@oakplatforms/types` **0.1.92** (new `OfferDto`,
`TradeDto` gone, `OrderListing.isOffer` gone).

## The two screens (from the 2026-08-27 mocks)

Both are one slide-up modal, "Make an Offer", built as a single `MakeOfferScreen` with a
`variant: 'listing' | 'bid'` route param. Tab/row-based, not one long scroll: each concern
(shipping, payment) is a row that opens its own panel.

| | Listing offer (buyer) | Bid offer (seller) |
|---|---|---|
| Header card | **listing card** — image, name, set code, hero line, price pill = listing price | **entity card** — same layout, price pill = bid price |
| Cards | one **line per listing** — price input prefilled with the listing price, stepper capped at listing quantity (full quantity forced when `multiTransactionsEnabled` is off); a lot arrives as all of its cards; **"Add more from this seller"** opens the seller's ACTIVE listings (the group add-listing picker pattern) and appends lines | one line — price input prefilled with the bid price; "N available" = bid quantity; stepper capped at it |
| Shipping row | buyer's address + tap → delivery panel (Untracked / Tracked / In Person via the existing 3-way `ToggleSwitch`, shipping options, address) | seller's own shipping methods (they ship) — same panel, no address (buyer's comes from their customer record) |
| Payment row | saved card (cash only when In Person) — reuses the `CartActions` Pay-With radios | hidden — the buyer pays |
| Summary | Offer price · Shipping · Estimated tax · Total (today's `CartOrderSummary` math) | Offer price · Shipping · "Buyer pays tax at acceptance" · Total before tax |
| Footer | ToS checkbox · "The seller has three days to accept this offer." · **Submit Offer** | "The buyer has three days to accept this offer." · **Review Offer** → step 2 (shipping) → **Send Offer** |
| Submit | `POST /offer { accountId, lines: [{ listingId, quantity, price }], bulkListingIds?, deliveryType, paymentType, shippingMethodId, shippingOptionIds, send: true }` — one seller per offer | `POST /offer { accountId, bidId, listingId?, quantity, price, …, send: true }` |

Copy says **three days** everywhere (never "72 hours"); the API window is 84h.

**No listing yet (bid offer):** the mock's price/quantity step is followed by an **image +
condition panel** (reuse the create-listing camera/picker and condition chips, limited to the
bid's accepted conditions), then `POST /listing { offerOnly: true, entityId, price, quantity,
conditionId, status: 'ACTIVE' }` and the offer is sent with that `listingId`. Everything else is
prefilled from the bid. So the bid variant also grows panels, like the listing one.

**After submit:** an offer-sent confirmation (the existing `OfferConfirmation` screen, re-copied
for both directions — the title names the direction, "You made an offer to **sell** N cards for $X" on a bid, "…to **buy**…" on a listing; since 2026-08-29 it has **no CTA** — the old "View Offers" button went to
`Main` because there is no offers screen in V1, offers live in the inbox, so the header ✕ is the
only way out and `ConfirmationLayout`'s button props became optional), then invalidate `['offers']`, `['systemMessages']`, `['messageFeed']`,
`['inboxCount']`, `['Listings']`, `['Bids']`.

## Inbox

- Offers tab stays and becomes the whole story; the **Orders tab and `ORDER`-category drill-down
  go away** (`MessagesScreen`, `SystemMessageCategoryFilter`). `OFFER_ACCEPTED` deep-links to
  `OrderDetail` for the order; every other type opens `OfferMessageDetail`.
- `OfferMessageDetailScreen` is rewritten around the new `OfferDto`: card + price × quantity,
  who sent it, shipping choice, status pill, and role-aware actions — recipient: **Accept /
  Decline** (`PUT /offer/:id/accept|decline`; a bidder accepting sees their card row first since
  that is when they pay); sender while `PENDING`: **Cancel**. After accept it links to the order.
- **Top line = the other party (2026-08-29).** The "@x offered on your bid" sentence became
  the counterparty's **avatar + username** (no @; the xl `sellerAvatar` + `detailPartyName`, the
  SemiBold body face — sized between the detail user row's title and the label face, to their
  profile), pulled up tighter under the header
  (`detailStatusRow` paddingTop xs), with the **status chiclet** on the right — "Pending" on
  the dark-grey default, "Accepted" green (`success`), "Declined" red (`error`), canceled /
  expired grey; `pill` face, white label. It is the one place the state is stated: the old
  "This offer was declined." footer line is gone.
- **One card for a bid offer (2026-08-29).** A bid offer is one line on the one listing the
  seller made (or updated) for it, so the detail renders a single `OfferListingCard` between
  the top line and the checkout rows — it **replaced** the separate entity/lines card for bid
  offers (listing offers keep the lines card). Top to bottom: the **entity screen's lockup without
  its image** — name, product number, the dot-divided set / primary-tag line (`EntityMetaLine`),
  tap to open the entity (no fair market, no offer-price pills; the offer price lives in the
  summary) — with the **listing price** in the white `lockupPricePill` at the header's far right; then
  the **seller's photo of the actual card**
  (create-listing square preview) with the **condition chiclet** right-aligned beneath it (the
  same pill dress filled with the condition's colour, ink label) and the caption, Delivery (only while pending or once accepted, and
  when the viewer isn't choosing it in the rows below — a declined / canceled / expired offer
  never became an order, so it shows no "Shipped" line) and the note. Open on arrival, not a collapsible, unlabelled. Read-only
  twin of the make-offer screen's `OfferListingRow`. Delivery values say **In Person** (capital
  P) everywhere they name the choice.

## Entry points that become "Make an Offer"

`ListingCard` (all three Buy Now variants), `FeedListingCard`, `ListingScreen` CTA,
`ShareBulkCard` single-listing CTA, `useShareActions.onBuy` → `navigation.navigate('MakeOffer',
{ variant: 'listing', listing })`. `BidScreen` / `useMakeOfferStart` / `FeedBidCard` →
`navigation.navigate('MakeOffer', { variant: 'bid', bid, listing? })`. The auth / customer /
payment-method gates in `useShareActions` stay in front of the navigation.

## Cart retirement

Everything below is deleted or unwired in this pass (the API keeps `/cart` + `/order` for now):

- `contexts/CartContext.tsx`, `contexts/data/fetchCartOrders.ts`, `hooks/useAddToCart.ts`, the
  `account.carts` include on the session user fetch (`SessionContext`).
- `components/cart/*` (`CartPreview`, `CartOrder`, `CartActions`, `CartOrderSummary`) and
  `screens/cart/ReviewCartScreen/*` — **after** the pieces the offer screen reuses are lifted
  into `components/offers/*` (delivery selector, options list, Pay-With radios, summary rows).
- `FloatingTabBar` cart pill slot + `handleReviewPress`, `useBottomPadding` / `Toast` cart
  clearance, `SearchScreen` cart footer variants, `styles/components/cart.ts` classes that
  nothing references afterwards.
- `utils/calculateCartTotals.ts` `isOffer` branch (the order line's `price` is now always the
  agreed price).
- Analytics: stop firing `cart_add` / `checkout_started`; `order_placed` fires on offer accept.
  Add `offer_sent` / `offer_accepted` / `offer_declined`.
- Dead offers-v1 code: `AcceptBidScreen` ("MakeOffer" today), `CreateOfferScreen`,
  `EditOfferScreen`, `components/offers/EditOffer`, `components/bids/BidOffers`,
  `components/orders/SellerOffers`, `EntityCard/Offer.tsx`, `hooks/fetchSellerOffers.ts`,
  `hooks/fetchOffer.ts` (rewritten), the offers section of `TradeListItem`, `OfferPayload` and
  the `Trade*` re-exports in `types/index.ts`.

## Navigation (escalation — P0 rule 6)

`AppNavigator` must change: remove `Cart`, `ReviewCart`, `CreateOffer`, `EditOffer`; repoint
`MakeOffer` at the new screen with `{ variant, listing?, bid? }` params; keep `OfferConfirmation`
and `OfferMessageDetail`. `contexts/CartContext.tsx` removal is also an escalation (P0 rule 7).
Both approved verbally on 2026-08-27 as part of this plan.

## Decisions & confidence

1. **One `MakeOfferScreen` with a `variant` param** instead of two screens — 0.85. Rejected:
   two screens (duplicate header/summary/footer > 15 lines each).
2. **Remove the cart wholesale in this pass** — 0.75. Rejected: leave `CartContext` mounted but
   never fetching (saves a day, leaves a dead request path and the pill geometry everywhere).
3. **Bid-offer step 2 shows no payment row and "buyer pays tax at acceptance"** — 0.6. The
   seller can't know the buyer's tax rate. Alternative: hide tax entirely.
4. **On-the-fly listing condition is picked by the seller** in the image/condition panel,
   limited to the bid's accepted conditions — decided 2026-08-27 (was a question).
5. **Listing offers are multi-line, one seller** — decided 2026-08-27: the API's `Offer` became
   a wrapper over a multi-line order, so a lot's "Purchase all" is one offer with every card as a
   line, and "add more from this seller" appends lines to the draft (`PUT /offer/:id` replace-set).
   Rejected: single-card offers only (would have dropped lots).

## As built (2026-08-27)

- **`screens/offers/MakeOfferScreen`** — the one screen, `route.params.variant: 'listing' | 'bid'`.
  Listing variant: one `OfferLineCard` per card (a lot arrives as every ACTIVE child at full
  quantity), "Add more from this seller" (`OfferSellerListingsPicker`, a modal of the seller's
  ACTIVE listings minus what's already on the offer), Shipping / Payment **accordion rows**
  that expand in place (`OfferShippingPanel` — the cart's 3-way `ToggleSwitch` + add-ons +
  address, `OfferPaymentPanel` — the Pay-With radios), `OfferSummary`, the ToS checkbox, the
  "three days" note, **Submit Offer** → one `POST /offer { lines, …, send: true }`. Bid
  variant: price + quantity → **Review Offer** → (no listing: `OfferNewListingStep` = photo +
  condition limited to the bid's accepted conditions → `POST /listing { offerOnly: true }`) →
  the seller's shipping panel + summary with "buyer pays tax at acceptance" → **Send Offer**
  → `POST /offer { bidId, listingId, quantity, price, …, send: true }`. Both land on
  `OfferConfirmation` (copy now says who has three days).
- **Layout tuning (2026-08-28, voice session).** The whole column steps out to a 14pt inset
  (`inset` in `getMakeOfferStyles`); the first card sits directly under PageLayout's floating-header
  gap (no local top pad); the line card's controls share the lockup's own 8pt inset; summary rows are
  12 apart; the add-more button lost its plus icon. Checkout panels get a 6pt top pad, the address
  renders as two lines (`formatCustomerAddressLines`), shipping services are the same pill tabs as
  the shipping method but multi-select with the (smaller) help text underneath, and the payment
  panel has no "Pay with" title. A listing's own photo sits `contain` on a dark frame in the thumb
  slot; `EntityLockupRow` gained an additive `fadeColor` prop so the tag-line fade matches the
  card's surface instead of painting a page-coloured square over it.
- **Totals** are previewed client-side by `utils/offerTotals.ts` (`computeOfferTotals`,
  `defaultShippingMethodId`) mirroring the API invoice math; the API is the source of truth
  once the offer is invoiced.
- **Entry points** now navigate to `MakeOffer`: `useShareActions.onBuy` / `onBuyBulk`,
  `ListingCard`, `FeedListingCard`, `FeedBulkCard` (per-card and lot), `useMakeOfferStart`
  (bid variant; the "you need a listing" sheet is gone — the screen's photo step replaced it).
- **Inbox**: Orders tab removed; `OfferMessageDetailScreen` rewritten on the new `OfferDto`
  (lines as lockup rows, delivery, summary, status pill; Accept / Decline for the recipient,
  Cancel for the sender, View Order once accepted; a bidder without a card is sent to Wallet
  before accepting).
- **Cart retired**: `CartContext` + provider, `fetchCartOrders`, `useAddToCart`,
  `components/cart/*`, `screens/cart/*`, the tab-bar cart pill, `useBottomPadding` / `Toast`
  cart clearance, `SearchScreen` cart footer variant, `account.carts` on the session fetch,
  `EntityCard` `cart` + `offer` variants. `styles/components/cart.ts` is left in place (dead
  classes) for a later sweep.
- **Deleted offers-v1**: `AcceptBidScreen`, `CreateOfferScreen`, `EditOfferScreen`,
  `components/offers/EditOffer`, `components/bids/BidOffers`, `components/orders/SellerOffers`,
  `hooks/fetchSellerOffers`; `TradeListItem` lost its inline offers section; `types/index.ts`
  lost the `Trade*` re-exports and `OfferPayload` (now `OfferDraftPayload` / `OfferLinePayload`),
  gained `OfferStatus` / `OfferType`.
- **Analytics**: `offer_sent` / `offer_accepted` / `offer_declined` / `offer_canceled` added;
  `cart_add` / `checkout_started` no longer fire (enum values kept).
- **Navigation** (approved escalation): `MakeOffer` → the new screen with the variant params;
  `CreateOffer`, `EditOffer`, `ReviewCart` routes and the `cart` / `review_cart` links removed.

- **Lot cover prices the whole lot (2026-08-29, voice session).** On the lot detail
  (`ListingScreen` with a `bulk`), the cover slide's terms row now shows the price input prefilled
  with the available cards' total, **"N items in lot"** (total quantity across available cards)
  with **no stepper** (`OfferTermsRow` `showStepper={false}`, label `'items in lot'`), and the CTA
  reads "Offer – $total". `onBuyBulk(bulk, price)` carries the named total to `MakeOffer`, where
  `distributeLotPrice` (`utils/offerPrice.ts`) spreads it across the lot's lines in proportion to
  their listing prices — a $2 discount on a $7 lot comes off every card evenly, unit prices
  rounded to cents with the remainder on the first single-quantity line — so the offer still
  goes out as ordinary per-line `lines` and the API is untouched. Child slides are unchanged.

- **Offer line card wears the order Items card thumb (2026-08-29, voice session).** The
  lockup's thumb IS the order screen's card — `panels.ts` `orderCardImageWrapper` (black
  container) + `orderCardImage` + `orderCardQuantityBadge` (pink, on the corner) — for the
  listing photo, or the entity art on a bid offer with no listing yet; `lineThumbSlot` pads the
  badge's overhang. **No fair-market and no price pill** on this card: the price input beneath
  is the number that matters (the offer detail keeps its price pill). Beside a stepper the count
  is what's **left** and counts down as the stepper goes up — "2 left", "1 left", "0 left" — on
  every terms row (listing, bid, make-offer); a single card has no stepper and says "Only 1
  available" / "Only 1 wanted"; a lot keeps "N items in lot". The detail CTAs read **"Offer – $1.50"**
  for one and **"Offer – 3 for $4.50"** for more (a lot's cover uses every available card) on
  both the listing and the bid screen — the WTB / WTS chiclet beside the seller's name already
  says which way. The
  count and stepper sit together on the right (`lineCountStepper`). `EntityLockupRow`
  keeps the additive `pricesAlign: 'left'` option (unused for now).

- **Offer detail: intent chiclet + outline condition (2026-08-29, voice session).** The top row
  keeps avatar + username on the left; on the right sit the `IntentTag` — **WTS** on a bid offer
  (the sender is a seller answering a bid), **WTB** on a listing offer (the sender is buying) —
  and the status pill together in one centred group (`detailStatusGroup`; the tag's own
  `alignSelf: flex-start` is overridden and it takes the pill's xs vertical padding so both stand
  the same height on one line). The status pill wears one dress for every state — dark grey,
  white label; the green / red accepted / declined fills went. The bid offer card's condition
  went back to the detail screens' outline "tab" pill (`detail.ts conditionPill`, condition
  colour on the label) instead of the colour-filled chip.

- **Quantity on the inbox offer (2026-08-29).** The bid offer's `OfferListingCard` shows a `×N`
  pill beside the price (the order line's quantity), and the API's "New offer" notification names
  it too — "3× Card" for one line, "6 cards" (total quantity) for several.

- **Inbox rows wear the other party's avatar (2026-08-29).** `SystemMessageTab` and `AllTab`
  read `systemMessage.actor.profile.avatar` (the API sets `actorId` on offer rows) and fall back
  to the Nanza logo when there is none — so offer rows show the person, true system events the
  brand. Needs `@oakplatforms/types` ≥ 0.1.98 (`SystemMessage.actor`, `Listing.isOffer`).

- **Offer price tolerance (2026-08-29).** An offer may move the reference price by at most
  **±30%** — the listing price on a listing offer (a lot's spread total moves every card by the
  same ratio), the bid price on a bid offer. `utils/offerPrice.ts` (`OFFER_PRICE_TOLERANCE`,
  `offerPriceBounds`, `offerPriceRangeError`) drives the inline check: `OfferTermsRow` gained an
  `error` line under the row (`termsErrorText`), and the listing / bid detail CTAs and the
  make-offer Submit stay disabled while any line is out of range. The API enforces the same
  bound (`assertOfferPriceInRange`), so the client check is a courtesy, not the gate.

- **Submit / Send feedback (2026-08-29).** `MakeOfferScreen` drives its CTA through `useFormBar`
  like the create-listing / create-bid screens: the Button's loading arc while the offer sends,
  a ~700ms checkmark (`success`), then `navigation.replace('OfferConfirmation')` from the
  hook's close callback (params parked in a ref at submit time).

- **Bid offer's listing row: no price, quantity follows the offer (2026-08-29).** `OfferListingRow`
  is photo + condition only; the **offer price is the price** — a fresh offer-only listing is
  created at it (`saveOfferListing({ price })`), an existing listing keeps its own. The bid alone
  caps the stepper: offering more than the seller's listing holds **raises the listing's
  quantity** on send (and flips `multiTransactionsEnabled` on when it goes above 1) instead of
  blocking. The listing / bid detail terms rows reset on focus (`useFocusEffect`) so a stepper
  left at 2 doesn't survive a trip to the offer screen.

- **Accept / Decline row (2026-08-29).** The recipient's actions on a pending offer are the
  listing / bid detail's two-pill row (`detail.ts` `actionBarRow`): the big white **Accept
  Offer** and the small dark **Decline** pill beside it (narrowed with `detailDeclinePill`) —
  replacing the button over a text link. "Buy Now" / "Sell Now" was tried and reverted the
  same day.

- **No Delivery line on the offer detail (2026-08-29).** The "Delivery · Plain envelope" row (and
  the bid offer card's copy of it) is gone: the offer detail is the negotiation, shipping lives
  on the order, reached by **View Order** once accepted. The bidder's own checkout rows still
  appear while they're accepting a bid offer.

- **One offer card lockup (2026-08-29).** `OfferLineLockup` is the shared piece — the order-card
  thumb with the pink quantity badge, name, number, tag line, optional left-aligned white pills —
  used by the make-offer `OfferLineCard` and by the offer detail's listing-offer lines — **no
  price pill on either** (the input / the summary carry the money), no fair-market pill, no ×N.
  The photo is 85% of the order card's, the identity sits near the thumb's top (`lineLockupBody`
  sm inset), the cards run a sm inset (detail cards md) and the summary sits md below them. The bid offer's `OfferListingCard` (photo panel + condition) is unchanged. Accept
  reads **Buy Now** (bidder) / **Sell Now** (seller) — settled.

- **Make-offer polish (2026-08-29, late).** First card flush under the floating header
  (`lineCardFirst`); the identity sits sm+xxs below the thumb's top. **"Remove from offer"** shows
  on every listing-offer card: with several cards it drops that one, with a single card it closes
  the screen (same as the header chevron); styled as a small accent link.

## Phasing

All five phases landed together on 2026-08-27 (see *As built*). Left for a follow-up: deleting
the dead `styles/components/cart.ts` classes, and a saved-offer ("draft") UI if v2 wants it —
the API already supports it.

## Related

- [[../../nanza-api/solution-designs/offer-engine|nanza-api — Offer Engine]]
- [[payments|Payments & checkout]] (the cart-era delivery/payment UI this replaces)
- [[trade-screen|Trade tab screen]]
