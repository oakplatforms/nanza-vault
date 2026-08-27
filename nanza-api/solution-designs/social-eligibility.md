---
tags: [nanza-api, nanza-mobile, solution-design, messaging, groups, connections]
---

# Social Eligibility — Registered Is Enough

> **Status: implemented 2026-08-17 (Skylar, voice session), uncommitted.** All social
> features — messaging/connections, group create *and* join, posting — require only a
> **registered account** (any `AccountType`). The old customer/seller tiers gate
> commerce (buy/sell/trade), not social.

## The rule

An account's type (`REGISTERED | CUSTOMER | SELLER`) reflects **payment capability**,
not social standing. Social surfaces stopped consulting it:

| Action | Old requirement | Now |
|---|---|---|
| Connections + messaging (send/receive) | CUSTOMER or SELLER, both parties | any registered account |
| Send friend request (mobile) | saved payment method (PaymentGate sheet) | none |
| Post to home feed | AFFILIATE profile (retired 2026-08-14) | any registered account ([[open-posting-and-friend-visibility|open posting]]) |
| Create group | seller **front-end only** — the API never gated it | any registered account |
| Join group | CUSTOMER or SELLER (API `customerOrSeller` + mobile CustomerGateModal) | any registered account |

Buy/sell/trade flows keep their gates: the payment-method gate stays on purchase
(`useShareActions`), bids (`CreateBid`), and trade proposals; groups' *why*: the
original seller-only group creation existed for moderator payouts, which were removed
([[payments|Payments]] — the 1% moderator rev-share is gone), so the gate lost its
reason.

## Backend (nanza-api)

- `validation/connection.ts`: `MESSAGING_ACCOUNT_TYPES` / `isMessagingEligible` /
  `validateMessagingEligibleAccount` **deleted** (same posture as the deleted
  `validateBrandPostPermission`). Every messaging route already runs
  `validateAccount(..., 'authenticated')`, which is the whole requirement now. Call
  sites removed in `connection.ts`, `conversation.ts`, `inboxCount.ts`,
  `systemMessage.ts`, `messageFeed.ts` — including the recipient/other-party type
  checks (existence check remains).
- `routers/group.ts` POST `/group/:id/members`: role `customerOrSeller` →
  `authenticated`. Group **create** was already `authenticated`-only.

## Mobile (nanza-mobile)

- Messaging: `useInboxCount` + `useConnections` fire for any signed-in account;
  `ConnectButton` (dead code — no callers) and `useProfileConnect` dropped their
  SELLER/CUSTOMER eligibility checks.
- Payment gate removed from the friend-request flow in `useProfileConnect`,
  `useMessageUser`, and `ActionModal/Feed` (the `PaymentGateSheetContent` connect
  arm); the component itself survives for buy flows.
- Groups: `HeaderCreateMenu` routes any registered user to `CreateGroupScreen`
  (`SellerGateModal` deleted); the join `CustomerGateModal` arm removed from
  `GroupsListScreen`, `GroupDetailScreen`, `GroupCircleCarousel` (component survives
  for bid/buy gates). Signed-out users still get `AuthModal` / placeholder screens.
