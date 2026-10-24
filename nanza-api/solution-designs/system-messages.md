---
tags: [nanza-api, solution-design, messaging, inbox]
---

# System messages — who speaks, what it says, when it leaves

## Overview

A **system message** is a `SYSTEM`-typed `Message` (no conversation) paired 1:1 with a
`SystemMessage` row: `category` (`GROUP` / `ORDER` / `OFFER`), `systemMessageType`, `title`,
one deep-link target (`groupId` / `orderId` / `refundId` / `offerId`) and, since 2026-08-29, an
**`actorId`** — the person the row is *from*. Created through `services/systemMessage.ts`
(`createSystemMessage`, `updateSystemMessage`, `deleteSystemMessage`); the mobile inbox renders
them in the All / Offers / Groups / Orders tabs and deep-links on tap.

## Who the row is from (2026-08-30)

**The Nanza logo is gone.** Every row wears the avatar of whoever it is from:

- **A person** → `actorId` (the `actor` select rides on `GET /system-messages` and
  `GET /message-feed`): the other party of an offer, the requester on a group join request, the
  buyer on a refund request / new order, the seller on a refund answer.
- **A group** → no actor; the `group` relation (`SystemMessage.group`, migration
  `20260830100000_system_message_group_refund`, `{ id, displayName, thumbnail }` on the same
  reads) — the group itself answers a join request, accepted or declined.

It is always the **inverse** of the viewer: what I did to you shows my face on your row, what
you did to me shows yours (or your group's) on mine.

## The title is who did what — nothing else

Titles are `"@username <verb> …"` with the *thing* left to the body: "`@x` sent you an offer",
"`@x` requested to join Flesh and Blood", "`@x` requested a refund", "`@x` placed an order",
"Flesh and Blood accepted your request", "`@y` declined your refund". Bodies carry the card or
lot, the terms, the reason, and what happens next. Per-event offer copy is tabulated in
[[offer-engine|Offer Engine]].

| Type | To | From (avatar) | Title |
|---|---|---|---|
| `GROUP_JOIN_REQUEST` | moderator | requester | "`@x` requested to join *Group*" |
| `GROUP_ACCEPTED` | requester | group | "*Group* accepted your request" |
| `GROUP_DECLINED` (new) | requester | group | "*Group* declined your request" |
| `ORDER_SOLD` | seller | buyer | "`@x` placed an order" (non-offer orders only) |
| `ORDER_REFUND_REQUESTED` | seller | buyer | "`@x` requested a refund" |
| `ORDER_REFUND_ACCEPTED` (new) | buyer | seller | "`@y` accepted your refund" |
| `ORDER_REFUND_DECLINED` (new) | buyer | seller | "`@y` declined your refund" |
| `ORDER_REFUND_*` (seller's row, rewritten) | seller | buyer | "You accepted / declined `@x`'s refund" |
| `ORDER_BUYER_PAYMENT_FAILED` (new, 2026-09-17) | seller | buyer | "`@x`'s payment failed" |
| `ORDER_PAYMENT_FAILED` (new, 2026-09-17) | buyer | seller | "Your payment didn't go through" |

## Declining: it leaves the decliner's inbox

A decline is finished for the person who declined — they don't need to keep seeing it — but the
other side must still hear it. So on **offer declined** and on a **group join request answered either
way** (2026-08-31: accepted too — the moderator doesn't care once it's handled) the actor's own
request row is **deleted** (`deleteSystemMessage`, keyed by type + target, and by `actorId` for
join requests since many requesters share one group) and a fresh row is created for the other
party. An accepted **offer** instead **rewrites** the recipient's row in place
(`updateSystemMessage` on a generic `target`) to "You accepted …". A **declined refund is the
exception**: the seller's row stays, rewritten to "You declined `@x`'s refund" (Skylar,
2026-08-30 — either was acceptable; kept for the order's paper trail).

## Payment failed on accept (2026-09-17)

The seller accepts, Stripe declines the buyer's card, the order is canceled. Before, the only
inbox trace was the offer close narrating it as "`@x` canceled their offer" — wrong on both
counts (the buyer didn't cancel, and cart orders said nothing at all). Now `acceptOrder` closes
any offer **silently** (`closeOfferForOrder(..., { notify: false })`) and
`notifyOrderPaymentFailed` (`services/order.ts`) tells both parties in order terms: the seller's
live row for the deal (`ORDER_SOLD`, or the offer recipient's `OFFER_RECEIVED`) is rewritten in
place to `ORDER_BUYER_PAYMENT_FAILED` — "`@x`'s payment failed … the order was canceled. Your
listings are available again." — and the buyer gets `ORDER_PAYMENT_FAILED` — "Your payment
didn't go through … declined for insufficient funds … You weren't charged. Update your payment
method and try again." Whoever has no live row gets a fresh one; offer orders keep the `OFFER`
category and `offerId` so the row still opens the offer. Two types rather than one because
mobile picks the order-detail variant from the type: `ORDER_PAYMENT_FAILED` opens as the buyer,
everything else as the seller. Email: only the **buyer** is mailed (the cancel template with
`paymentFailed: true`, headlined "Your payment didn't go through"); the seller already heard it
from the accept response and their inbox row. Migration `20260917120000_order_payment_failed`
adds the two enum values (to run).

## Decisions

- **Relation for the group, not a runtime attach.** `groupId` already existed as a bare column;
  the FK (`ON DELETE SET NULL`, orphans nulled in the migration) is the same shape `actorId` uses
  and keeps the select declarative.
- **Delete, don't hide.** A client-side "hidden" flag would need another column and every read
  path to honour it; the row has no value to the decliner, so it goes.
- **Deep links** on mobile: refund answers open the order as the **buyer**; every other order row
  as the seller. A `GROUP_JOIN_REQUEST` opens the group's **members screen on its Requested
  tab** (`GroupMembersScreen`, 2026-08-31); the group's answers open the group detail.

## Related

- [[offer-engine|Offer Engine]] — offer rows and their copy
- [[../../nanza-mobile/solution-designs/inbox|Inbox (mobile)]] — the tabs and how rows render
