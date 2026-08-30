---
tags: [nanza-api, nanza-mobile, solution-design, plan, offers, orders, trade]
status: API built 2026-08-27 (uncommitted; migration 20260827120000_offer_engine to run); mobile half next
---

# Offer Engine & Trade Removal

## Goal

Retire card-for-card **trading** entirely and replace the current **offers** (a seller's reply to a
bid) with an **offer engine that sits on top of orders**. An offer becomes the single way a buyer
and seller agree on a deal for *both* listings and bids; the cart-based marketplace flow is phased
out afterwards. Nothing downstream of a `PENDING` order (Stripe capture, shipping, Shippo,
refunds, reviews, collection sync) changes.

```
send offer  ──▶  Offer(PENDING) + Order(CREATED)
                        │
          accept ───────┼──────── decline / cancel / expire
            │                              │
   Order → PENDING (invoice/PI)     Order → CANCELED, Offer → DECLINED/CANCELED/EXPIRED
            │
   existing order pipeline: seller accept-order → capture → ship → complete
```

## Part A — Remove trading

Already removed from mobile (2026-08-26, see
[[../../nanza-mobile/solution-designs/trade-screen|Trade tab screen]]). Backend still has all of it.

**Schema (DB tables to confirm empty, then drop):**

| Table | Notes |
|---|---|
| `Trade` | |
| `TradeOfferEntity` | |
| `TradeRequestEntity` | |

- Enum: `TradeStatus`.
- Back-relations to delete: `Account.tradesProposed` / `Account.tradesReceived`,
  `Entity.tradeOfferEntries` / `Entity.tradeRequestEntries`.

**Code:**

- `src/routers/trade.ts` — replace with a stub router (see *Stub policy*).
- `src/validation/trade.ts` — delete.
- `src/services/trade.ts` — **keep the code, rename the file** to `src/services/collection.ts`.
  It holds the collection-move primitives (`addEntitiesToList`, `addCollectionEntities`,
  `transferCollectionEntities`) used by `listing.ts`, `scanItem.ts` and `services/order.ts`
  (see [[data-sync|Data sync]]). Drop `computeTradeFulfillment` (trade-only).
- `all_routes.ts` — mount the stub instead.
- Unrelated, keep: `GET /trade-feed` in `bulkListing.ts` (profile "Trades" rail — listings + bids,
  not the Trade model).

**Docs:** update [[data-sync|Data sync]] (file rename) and `architecture.md` (drop the Trade
model + router lines).

## Part B — Remove offers v1

Offers v1 = a seller's reply to a bid (`Offer{bid, listing, price, quantity}`), accepted by the
bidder into a cart order.

**Schema:**

| Table / column | Action |
|---|---|
| `Offer` | drop table (rebuilt in Part C with a new shape) |
| `Order.offerId` | keep the column — v2 reuses the 1:1 |
| `OrderListing.isOffer` | drop (redundant with `Order.offerId`) |
| `SystemMessage.offerId` | keep — v2 reuses |

- Enums: `OfferStatus` (replaced by the v2 enum), keep `SystemMessageCategory.OFFER` and
  `SystemMessageType.OFFER_RECEIVED` (v2 reuses, and adds more types).
- Back-relations: `Bid.offers`, `Listing.offers` (re-added in v2 shape).

**Code touching v1 offers (all to be removed / rewritten):**

- `src/routers/offer.ts` (990 lines) — rewritten for v2.
- `src/routers/order.ts` — `offerId` on `POST /order` (l.313/323/579), the "restore offers" block on
  `PUT /order/:id` (l.937–965), offer cancel in `accept-order` (l.1239) and `cancel-order` (l.1677).
- `lambdas/cancelOrderHandler.ts` — offer cancel (l.113–121).
- `src/routers/systemMessage.ts` — `VALID_CATEGORIES` stays (OFFER is still a category).
- `packages/types` — regenerate + bump.

**Mobile call sites that still hit v1 (rewired in the mobile half):** `services/api/Offer.ts`,
`hooks/fetchOffer.ts`, `hooks/fetchSellerOffers.ts`, `screens/offers/{CreateOffer,EditOffer,
OfferMessageDetail}Screen`, `screens/bids/{AcceptBidScreen,OfferConfirmationScreen}`,
`components/offers/EditOffer`, `components/entity/EntityCard/Offer.tsx`,
`components/trade/TradeListItem`, `MessagesScreen` Offers tab.

## Stub policy (trade + offer v1 routes)

Keep every old route mounted so an old app build never hits a 404/HTML from API Gateway. Each
stub returns the same shape the app already understands for errors:

```
410 Gone  { errorMessage: 'Trading is no longer available. Please update the app.' }
```

Routes stubbed: `GET /trades`, `GET /trade/unread-count`, `GET /trade/:id`, `POST /trade`,
`PUT /trade/:id/{accept,reject,cancel,seen}`, `POST /trade/:id/counter`, and the v1-only offer
routes that v2 doesn't keep (`PUT /offer/:id/update`, `GET /offers/seller/:accountId`,
`DELETE /offer/:accountId/:id`).

Decided (2026-08-27): 410 + `errorMessage`, matching the app's existing error contract. The stubs
are temporary and can be deleted once old builds age out.

## Part C — Offers v2 design

### Concepts

- An **Offer** is a negotiation wrapper around exactly one **Order**. Roles on the order are fixed
  by ownership regardless of who initiated: `Order.customer` = the buyer, `Order.seller` = the
  listing owner.
- **The buyer always checks out.** The sender fills in the cards, quantity, price and note; the
  delivery type, payment type and shipping method + options are the **buyer's** choice — made on
  the draft for a listing offer (buyer → seller) and **on accept** for a bid offer (the bidder is
  the buyer). The buyer address is snapshotted from the buyer's `Customer` when the order is
  invoiced. A seller answering a bid therefore only names price + quantity (and adds a photo +
  condition if they have no listing). Reversed 2026-08-28 from "sender fills in everything" —
  see decision 2.
- **Gates to send:** a listing offer requires a customer account with a card on file
  (`Customer.hasPaymentMethod`); a bid offer requires a seller account.
- **Saved (draft) offer** = `Offer(DRAFT)` + `Order(CREATED)` — the cart's "sitting there" state,
  nothing sent yet. **Backend-only in v1:** the model, `DRAFT` status, `POST /offer` without
  `send`, `PUT /offer/:id` editing, and draft-aware reads all ship now, but the v1 mobile UI has
  no "save for later" — it always calls `POST /offer` with `send: true`. Drafts are excluded from
  the recipient's views and from expiry, so the unused path is inert until a later release adds
  the UI.
- **Two entry points, one model, different pending moments:**

  | | Listing offer (buyer → seller) | Bid offer (seller → bidder) |
  |---|---|---|
  | On **send** | `POST /invoice` path runs: quantity decremented, PI **authorized** on the buyer's card, order **`PENDING`**. Seller notified. | Order stays **`CREATED`** (can't authorize the bidder's card before they agree). Bidder notified. |
  | On **accept** | = today's `accept-order` (capture, shipment, tracking). | Invoice runs (order `PENDING`, PI authorized) **and then `accept-order` runs automatically on the seller's behalf** — the seller's send *was* their acceptance. Capture + shipment + tracking start in one step. |
  | On **decline** | = today's `cancel-order` (PI canceled, quantities restored). | Order `CANCELED`, offer `DECLINED`. Nothing to unwind. |
  | On **cancel** (sender) | `cancel-order` semantics, offer `CANCELED`. | Order `CANCELED`, offer `CANCELED`. |

  After `PENDING` nothing changes: seller `accept-order` → capture → ship → complete (tracked
  orders complete on the Shippo DELIVERED webhook; untracked/in-person complete on accept).
- **Bid offers and listings:** if the seller already has an ACTIVE public listing of the bid's
  entity it's reused; otherwise a listing is **created on the fly** — the seller only adds an
  image, everything else (entity, price = offer price, quantity, condition) is prefilled. That
  listing is flagged **`isOffer`** (and forced `isPublic: false`, no groups) and the offer path
  skips `POST /order`'s source-group requirement for private listings. Bid → `INACTIVE` on
  accept (as v1).
- **Counter** (later release): a new Offer row with `parentId` pointing at the countered one; the
  countered offer's order is canceled and a fresh order is created for the counter. Only the
  latest offer in a chain is actionable.

### Data model (as built)

The Offer is a **pure negotiation wrapper**; the cards, quantities and agreed prices live on the
order's `OrderListing` rows (`price` = agreed unit price). That is what lets a listing offer carry
**many cards from one seller** — several singles, a whole lot, or both — using the multi-line
order machinery that already exists.

```prisma
enum OfferStatus { DRAFT PENDING ACCEPTED DECLINED CANCELED COUNTERED EXPIRED }
enum OfferType   { LISTING BID }

model Offer {
  id / createdAt / updatedAt
  status      OfferStatus @default(DRAFT)
  type        OfferType
  note        String?  @db.Text
  sentAt / respondedAt / expiresAt

  sender      Account   (senderId)      // who made this offer
  recipient   Account   (recipientId)   // who must respond
  bid         Bid?      (bidId)         // set for BID offers
  order       Order?                    // 1:1 via Order.offerId (kept, unique)
  parent      Offer?    (parentId)      // counter chain, self-relation
  children    Offer[]

  @@index([senderId, status]) @@index([recipientId, status]) @@index([bidId]) @@index([parentId])
  @@index([status, expiresAt])
}
```

- **One seller per offer.** Every line must belong to the same seller (`assertSingleSeller`); an
  offer is one conversation with one person about one order. Multi-seller carts are gone for good.
- **Bid offers are single-line** (the bid is for one card).
- `Listing` gains **`isOffer Boolean @default(false)`** (migration `20260829120000_listing_is_offer`,
  2026-08-29). `isPublic` alone couldn't mark an offer-only listing: `isPublic: false` already means
  "group-only" *and* "owner's profile went private", and in both of those the owner (and group
  members) must still see the listing. `isOffer` is the one flag that hides a listing from its
  owner's own lists too. The on-the-fly listing is `isOffer: true`, `isPublic: false`, no groups,
  and the offer path **skips the private-listing source-group check** that `POST /order` enforces.

### Endpoints

| Route | Body / notes |
|---|---|
| `POST /offer` | Listing offer: `{ accountId, lines: [{ listingId, quantity, price? }], bulkListingIds?, … }` (or the single-card shorthand `listingId` + `quantity` + `price?`); a missing `price` means the listing price. Bid offer: `{ accountId, bidId, quantity, price, listingId? }`. Both add `note?, deliveryType?, paymentType?, shippingMethodId?, shippingOptionIds?, send?`. Returns the offer with its order draft. `send: true` sends in the same call. |
| `PUT /offer/:id` | `{ accountId, …same draft fields }` — edit a DRAFT. `lines` / `bulkListingIds` are a **replace-set** (mobile holds the draft and PUTs the whole list after "add more from this seller"); a bid offer keeps its listing and only changes quantity/price/shipping. |
| `PUT /offer/:id/send` | `{ accountId }` (sender) — DRAFT → PENDING. Listing offer: runs the invoice path (order `PENDING`, PI authorized). Bid offer: order stays `CREATED`. Notifies recipient. |
| `GET /offers` | `?accountId=&role=sent\|received&status=` paginated |
| `GET /offer/:id` | includes `order`, `listing`, `bid`, `parent` |
| `PUT /offer/:id/accept` | `{ accountId }` (recipient). Listing offer: = `accept-order`. Bid offer: the body also carries the bidder's checkout — `deliveryType`, `paymentType`, `shippingMethodId`, `shippingOptionIds` (method required unless in person) — written onto the order draft, then invoice → `PENDING`, then `accept-order` on the seller's behalf. Returns the offer with its order. |
| `PUT /offer/:id/decline` | `{ accountId }` (recipient) — listing offer: `cancel-order` semantics; bid offer: order `CANCELED`. |
| `PUT /offer/:id/cancel` | `{ accountId }` (sender) — same unwinding as decline, offer `CANCELED`. |
| `POST /offer/:id/counter` | later — `{ accountId, price, quantity }` |

`PUT /offer/:id` exists in v1 too; it is rewritten rather than stubbed.

### Order interplay

- Extract the order-draft validation from `POST /order` (access/block checks, quantity + seller
  max-quantity, `$1,000` cap, delivery/payment validation, snapshots) into a service
  (`services/order.ts#createDraftOrder`) so cart-add (until it's removed) and offers share one path.
- `Order.cartId` stays null for offer orders; while the cart still exists, cart reads
  (`/cart-preview`, `GET /orders` by cart) must exclude offer orders and offer reads must exclude
  cart orders — i.e. filter on `offerId`.
- `cancelOrderHandler` (cron) gains a second sweep: bid-offer orders still `CREATED` with the
  offer `PENDING` for > 72h → offer `EXPIRED`, order `CANCELED`. Listing offers are `PENDING`
  orders and already fall under the existing unshipped-order sweep (which also flips the offer to
  `EXPIRED`). DRAFT offers never expire.
- Bid offers on accept: bid → `INACTIVE` (same as v1). Partial-quantity handling: open.
- Collection sync, payouts, refunds, reviews: untouched.

### Messaging

- Every lifecycle event is a `SystemMessage` in category `OFFER` that deep-links to the offer
  (no connection required). `SystemMessageType` gains `OFFER_ACCEPTED`, `OFFER_DECLINED`,
  `OFFER_EXPIRED`, `OFFER_CANCELED` (and `OFFER_COUNTERED` later) alongside `OFFER_RECEIVED`.
  Realtime `offer.received` already exists; accept/decline reuse `order.updated`.
- **One inbox row per offer for the recipient (2026-08-29).** `closeOfferForOrder` rewrites the
  recipient's `OFFER_RECEIVED` row in place (`updateSystemMessage`: type, title, `Message.body`,
  `orderId`) instead of adding a second row — "New offer" becomes "Offer accepted · You accepted
  @x's offer for your bid on Y" / "Offer declined · You declined …" (stays read: they did it), or
  "Offer canceled" / "Offer expired" (resurfaced unread + realtime nudge: done to them). Falls
  back to creating a row if the original is gone. The sender still gets a fresh row per outcome
  (they had none). Body wording is from the recipient's side: "your bid on X" / "your listing
  for X".
- The `ORDER` category is no longer emitted for offer-originated orders (`ORDER_SOLD` on invoice
  becomes `OFFER_ACCEPTED`/`OFFER_RECEIVED` as appropriate); mobile drops the Orders tab.

### Cart retirement (later pass, not this one)

`Cart` model + router + `validation/cart.ts`, `Order.cartId`, `/cart-preview`, mobile
`CartContext` / `ReviewCartScreen` (its layout becomes the offer screen). Once carts are gone,
every `CREATED` order is an offer order.

## As built (API, 2026-08-27)

**Files.** `services/offer.ts` (draft → send → accept / decline / cancel → expire),
`services/offerStatus.ts` (`closeOfferForOrder` + the OFFER-category system messages),
`validation/offer.ts` (draft shape, party/status asserts, the one-open-offer-per-stranger rule),
`routers/offer.ts` (v2 routes + three 410 stubs), `routers/trade.ts` (all 410 stubs),
`services/trade.ts` → `services/collection.ts` (`computeTradeFulfillment` dropped),
`validation/trade.ts` deleted. `services/order.ts` now owns `acceptOrder` and
`cancelPendingOrder` (moved out of the router so the offer engine can call them; the
`accept-order` / `cancel-order` handlers are thin wrappers) plus the snapshot / max-quantity
helpers `POST /order` and the offer draft share.

**Where offer state is synced.** Every order transition that matters closes the offer *inside
the same transaction* via `closeOfferForOrder`: `acceptOrder` → `ACCEPTED` (capture failure →
`CANCELED`), `cancelPendingOrder` by the seller → `DECLINED`, by the buyer → `CANCELED`, the
cron's unshipped-order sweep → `EXPIRED`. It is a no-op unless the offer is `PENDING`, so
replays and double-taps are harmless. A seller who accepts a listing offer from the *order*
screen (`PUT /order/:id/accept-order`) closes the offer exactly the same way.

**Send.** `POST /offer` with `send: true` (what mobile v1 does) or `PUT /offer/:id/send`. On
send: listing still ACTIVE with enough quantity, blocks, the stranger rule, a shipping method
chosen for SHIP orders, then — listing offer — card on file + seller Stripe account,
`validateOrdersForInvoice` + `createInvoiceWithTransactions` (order → `PENDING`); bid offer —
seller `isAdminVerified` + Stripe account, order stays `CREATED`. `sentAt`/`expiresAt` are set
and the order's `createdAt` is reset to the send time so the unshipped-order sweep's clock
starts at send, not at draft creation. The invoice service **skips its `ORDER_SOLD` system
message** for offer orders (the engine sends `OFFER_RECEIVED` instead); the seller/customer
confirmation **emails are unchanged** (follow-up: offer-specific templates in nanza-email).

**Accept.** Listing offer → `acceptOrder` as the seller. Bid offer → the bidder's checkout
(`validateOfferAcceptBody` → `resolveShippingSelection`, the same seller-offers-it / add-ons-exist
checks the draft path uses) is written onto the `CREATED` order — delivery type, payment type,
shipping method, replace-set of shipping options — a shipping method is required for `SHIP`,
then card-on-file check, invoice (which prices shipping/tax/total from those relations),
`acceptOrder` on the seller's behalf, then the bid's quantity is reduced by the offer quantity
(INACTIVE at zero — partial fills keep the bid open for the remainder). Send no longer requires
a shipping method on a bid offer (2026-08-28); it still does on a listing offer.

**Expiry.** `OFFER_EXPIRY_HOURS = 84`, matching the existing sweep exactly (the plan said
"72h / same window" — the cron's window is 84h, so 84h wins for consistency). Listing offers
are expired by the existing PENDING sweep; `expireUnansweredBidOffers()` runs after it for
`CREATED` bid-offer orders. Drafts never expire.

**On-the-fly listing.** Mobile creates it first with `POST /listing` + `offerOnly: true`
(forces `isPublic: false`, no groups, bypasses the "public or at least one group" rule), then
`POST /offer { bidId, listingId, quantity, price }`. The seller picks the listing's **condition**
in that step (decided 2026-08-27); the bid's accepted conditions are the allowed choices.

**Offer-only listing visibility (2026-08-29).** `offerOnly: true` sets `Listing.isOffer`, and an
`isOffer` listing surfaces **nowhere except inside its offer** — not the homepage/query lists/entity
page/search (already excluded by `isPublic: false`), and not the owner's own surfaces either:
`buildListingVisibilityFilter` and `buildPublicAuthorListingFilter` AND `isOffer: false`,
`/sell-feed`, `/trade-feed` and the profile `listingCount` exclude it, and `GET /listings` in owner
scope hides it unless the query is entity-scoped (`?entityId&accountId`, the lookup the bid-offer
flow uses to find and reuse the listing it created). It can't be promoted: `PUT /listing` rejects
`isPublic: true` or any `groupIds`, `POST /group/:id/listing` rejects it, and lots refuse to bundle
it. `canAccessListing` lets its owner and the other party of any offer whose order carries it read
it (`GET /listing/:id`); everyone else gets 404. `resolveListings` (bid price-conflict) and the
project builder's listing picker ignore it.

The one-ACTIVE-listing-per-entity rule is **per kind**: a real (public/group) listing and an
offer-only listing of the same card may coexist, so answering a bid on card X never blocks the
seller from listing X for real, and vice versa. Every duplicate check carries `isOffer` — `POST`
/ `PUT /listing`, the scan duplicate filters (`isOffer: false`, scans only make real listings),
and the cancel-order restore (`restoreOrderListingQuantities` re-activates against the same
kind). The owner's entity lookup orders `isOffer asc` so "your listing" surfaces pick the real one
when both exist.

**Multi-line listing offers (added the same day).** `Offer` lost `listingId` / `price` /
`quantity`; the resolver (`resolveOfferContext`) turns `lines` + `bulkListingIds` into
`OrderListing` rows (a lot expands to one full-quantity line per ACTIVE child at the listing
price — what "Purchase all" used to do), enforces one seller, and sums quantity against the
seller's max and the $1,000 cap. Send/edit re-derive the lines from the order so they always
re-validate exactly what was drafted.

**Price rules (2026-08-29).** The market-conflict checks are gone: creating or editing a listing
no longer fails because a public bid sits higher, and a bid no longer fails because a public
listing sits lower ("A bid already exists for this product at a higher price…" and its mirror,
in `POST`/`PUT /listing` and `/bid`; `services/resolver.ts` deleted). With offers, buyers and
sellers price against the **entity** as they like and negotiate from there — a $20 listing next to
a $10 one is fine (condition, trust). In their place, an offer must stay within **±30%** of its
reference — `assertOfferPriceInRange(price, reference, 'listing' | 'bid')` in
`validation/offer.ts` (`OFFER_PRICE_TOLERANCE`), run per line in `validateOfferContext`: the
listing price for listing offers (a lot's spread total moves every line by one ratio), the bid
price for bid offers. Mobile mirrors the bound inline; the API is the gate.

**Notification copy (rewritten 2026-08-29, twice).** One voice throughout: the **title** names
who did what on whose bid / listing, the **body** carries the details line — what was wanted,
the card, the terms — then what happens next.

| Event | Title | Body |
|---|---|---|
| New offer (recipient) | "`@x` sent an offer for your bid / listing" | "Wants to sell 3× Card for $45.00 · Near Mint. You have 3 days to accept or decline." |
| Accepted (sender) | "`@y` accepted your offer for their bid / listing" | "Wanted to sell … Ship the order to complete the sale." / "… Your order is on the way." |
| Accepted (recipient row) | "You accepted `@x`'s offer for your bid / listing" | details + "They will ship your order." / "Ship the order …" |
| Declined | "`@y` declined your offer for their …" / "You declined `@x`'s offer for your …" | details (+ "You can send a new offer.") |
| Canceled (recipient row) | "`@x` canceled their offer for your …" | details |
| Expired | "Your offer for `@y`'s bid / listing expired" / "`@x`'s offer for your … expired" | details + "didn't respond within 3 days" |

Helpers: `cardName` ("3× Card" for one line, "6 cards" total for several), `offerTerms`
("$45.00 for 3 · Near Mint" — agreed total, quantity when > 1, condition when one listing),
`wantsLine` ("Wants / Wanted to sell|buy `card` for `terms`"), `yourThing` / `theirThing`.
`offerNotifyInclude` therefore carries line `price` and the listing's `condition`. Offer rows
also set **`SystemMessage.actorId`** (new nullable `Account` relation, migration
`20260829130000_system_message_actor`): the sender on the recipient's row, the recipient on the
sender's, so the inbox shows that person's avatar instead of the Nanza logo. `GET
/system-messages` and `/message-feed` include `actor.profile { username, avatar }`.

**Reads.** `GET /offers?accountId&role=sent|received|all&status=` (drafts only ever visible to
their sender), `GET /offer/:id` (parties only). Default include: both parties as
`{ id, profile }` (never the Account row), bid + entity, and the order with shipping method /
options / lines (each with listing → condition + entity **with `entityTags.tag`, `set`, `brand`,
`product`** — the full lockup the mobile offer detail renders, 2026-08-29) / shipments.

**Emails (2026-08-28).** nanza-email keeps its existing templates and rewords them by reading
the offer off the order (`?include=offer`); the buyer's purchase-confirmation template is
untouched. The rule: *send offer → the recipient is emailed; accept → the order confirmations go
out.*

| Moment | Who | Event | Rendered as |
|---|---|---|---|
| Listing offer sent | seller | `order.confirmation.seller` (from the invoice, as before) | "You've received an offer!" + respond-within-3-days panel |
| Listing offer sent | buyer | — | nothing: the invoice service skips `invoice.confirmation.customer` for offer orders |
| Listing offer accepted | buyer | `invoice.confirmation.customer`, now emitted by `acceptOrder` | the unchanged "Thank you for your purchase" |
| Bid offer sent | bidder | **new** `offer.confirmation.customer` (from `sendOffer`) | new `BidOfferReceived` template: cards + offer total, no checkout yet |
| Bid offer accepted | seller / bidder | `order.confirmation.seller` (invoice) / `invoice.confirmation.customer` (`acceptOrder`) | "Your item has been sold!" / purchase confirmation, both unchanged |
| Declined / canceled / expired (either type) | see below | `order.canceled.{customer\|seller}` | cancel template headlined by `offer.status`: "Your offer was declined", "Offer canceled", "Your offer wasn't accepted" / "An offer expired" |

`offerClosedEmailEntries` (`services/offerStatus.ts`) decides the audience and the offer-worded
reason: a decline reaches the sender, a cancel the recipient, expiry both. `cancelPendingOrder`
and the cron use it only when `closeOfferForOrder` actually closed a PENDING offer — an accepted
offer that is then never shipped stays a plain order cancellation. Bid offers, which never had
cancel events, now emit them from `declineOffer` / `cancelOffer` / `expireUnansweredBidOffers`.
`sendOrderCanceledEvents` (exported from `services/order.ts`) is the single cancel emitter.
Follow-up: bump nanza-email's `@oakplatforms/types` once ≥ 0.1.92 is published; until then it
reads the offer through a local `OrderWithOffer` shim.

**Types.** `@oakplatforms/types` regenerated and bumped to **0.1.92**: `OfferDto` has the new
shape, `TradeDto` & co. are gone, `OrderListing.isOffer` is gone — mobile/admin must stop
reading them.

## Phasing

1. ~~**Teardown**~~ — done 2026-08-27 (migration written, not yet run).
2. ~~**Offer engine core**~~ — done 2026-08-27.
3. ~~**Bid path**~~ — done 2026-08-27 (`offerOnly` listings + bidder-side accept).
4. ~~**Expiry + feeds**~~ — done 2026-08-27 (cron sweep, `GET /offers`, realtime via the
   existing system-message emit).
5. **Mobile** — offer screen (from the cart design), inbox drill-down, listing/bid CTAs. Single
   create-and-send step; no saved-offer UI in v1.
6. **Counter** — chain + swap-order semantics.

## Decisions so far (2026-08-27 voice session)

1. **Payment timing** — resolved: see the table above. Listing offers authorize on send; bid offers
   authorize on the bidder's accept and auto-run `accept-order` for the seller.
2. **Shipping for bid offers** — **reversed 2026-08-28** (Skylar, voice session): the seller
   does *not* pick shipping. Shipping methods belong to the seller, but choosing among them —
   and delivery type, add-ons and payment — is the buyer's decision, exactly as on a listing
   offer; on a bid offer the buyer is the bidder, so they choose **on accept**
   (`PUT /offer/:id/accept` carries the checkout). The seller's offer screen shows price +
   quantity only. Original 2026-08-27 call was "sender picks up front"; it made a seller send
   the bidder shipping choices that weren't theirs to make.
3. **Hidden listing** — resolved: `isOffer: true` + `isPublic: false`, offer path skips the group
   check; see "Offer-only listing visibility" below.
4. **Send gates** — resolved: customer + card on file for listing offers; seller account for bid
   offers.
5. **Expiry** — resolved: same window as unshipped orders (**84h** in code — see *As built*). Listing offers are already
   covered by the existing cron (they're `PENDING` orders); `cancelOrderHandler` gains one extra
   sweep for `CREATED` bid-offer orders older than 72h → order `CANCELED`, offer `EXPIRED`. That
   is the only cron change; everything downstream of `PENDING` stays untouched.
6. **Inbox surface** — resolved: offers stay **system messages in the Offers tab** for this
   release; direct-message delivery is a later evolution. The **Orders tab and `ORDER`-category
   messages are retired on the client** — the whole lifecycle is narrated under the `OFFER`
   category (received → accepted "your order is on the way" → declined / expired, and later
   shipping/refund events). Enum values (`SystemMessageCategory.ORDER`, `ORDER_SOLD`, …) stay in
   the schema; the API just stops emitting them for offer-originated orders.

8. **Multi-card listing offers** — resolved (2026-08-27, later in the session): a listing offer
   carries any number of that seller's listings, including whole lots; the buyer can browse the
   seller's other listings and add them to the draft. One seller per offer. Bid offers stay
   single-line. This is why the Offer model carries no listing/price/quantity of its own.
9. **On-the-fly listing condition** — resolved: the seller picks it (an extra panel in the bid
   variant of the offer screen), constrained to the bid's accepted conditions.
7. **Multiple pending offers** — resolved: the cap is **per sender→recipient pair**, not per
   listing. **Non-connected** accounts may have only **one `PENDING` offer** open toward a given
   recipient at a time (the recipient must respond before another can be sent — spam guard).
   **Accepted connections** may have any number open. The check (`assertCanSendOffer`, in
   `validation/offer.ts`) uses `findConnectionBetween` from `services/connection.ts` and runs at
   send time, not draft time. Blocked in either direction always rejects. Quantity is decremented
   on send for listing offers, so a second buyer can only offer on what's left.

## Open questions

_None left as of 2026-08-27._

## Related

- [[../INDEX|nanza-api]] · [[../architecture|Architecture]]
- [[payments|Payments]] · [[data-sync|Data sync]] · [[account-feeds|Account feeds]]
- [[../../nanza-mobile/solution-designs/trade-screen|Mobile — Trade tab]] ·
  [[../../nanza-mobile/solution-designs/payments|Mobile — Payments & checkout]]
