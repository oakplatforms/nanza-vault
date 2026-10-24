---
tags: [nanza-mobile, solution-design, inbox, messaging]
---

# Inbox — tabs, friends and friend requests

## Overview

`MessagesScreen` is four tabs over one unified feed, in this order (Skylar, 2026-08-30):
**All**, **Offers**, **Friends**, **Groups** — plus a fifth, **Refunds**, that only appears once
the viewer has a refund row (`GET /system-messages?category=ORDER` with a `refundId`; the
`orders` tab id is kept, the label reads "Refunds"). Friend requests have **no tab of their
own** (Skylar, 2026-08-29): they arrive in **All** as pending intro threads, and the **Friends**
tab surfaces them as one row. Refunds is the one order event that needs answering — requested
(to the seller), accepted / declined (to the buyer) — so it earns a tab only when it exists.

| Tab | Source | Rows |
|---|---|---|
| All | `GET /message-feed` | conversations (incl. pending intros) + every system message, newest first; the only tab with a numeric unread count |
| Friends (was "Messages") | `GET /conversations` (ACCEPTED connections only) + `GET /connections?status=PENDING` | a **"Friend requests (N)"** row when people are waiting on you, then your friends' conversations |
| Offers / Groups / Refunds | `GET /system-messages?category=` | system rows; every row wears whoever it's from — the other party (`systemMessage.actor`) or, when a group answers, the group (`systemMessage.group.thumbnail`). No Nanza logo any more (2026-08-30); the fallback is the first letter. |

Row rules (who speaks, what the title says, what leaves the inbox on decline) are the API's:
[[../../nanza-api/solution-designs/system-messages|System messages]]. `SystemMessageDto` is the
package type as-is (`@oakplatforms/types` 0.1.102 ships `group` and the new types); a refund
answer deep-links to the order as the **buyer**, other order rows as the seller.

## Friends tab

`FriendsTab` composes the requests row + `ActiveTab` (the friends' conversation list; empty
state "No friends yet"). The row is a `MessagesListRow` — first requester's avatar, name
"Friend requests", the count as its badge, a preview like "@a, @b and 2 others want to be your
friend", unread while any request is unseen (`recipientSeenAt` null). Tapping it marks them seen
and opens **`FriendRequestsSheetContent`** in the modal stack (a sheet, so no new route): one row
per requester — avatar, username, intro note — with **Accept / Decline** inline
(`useAcceptConnection` / `useDeleteConnection`, the same mutations the intro thread uses); the
row itself opens the intro thread (`conversationId` attached by the connections endpoint). The
sheet reads its own connections query so it stays live as requests are answered.

## Decisions

- **Sheet, not a screen** — 0.85. A new route means `AppNavigator` (P0 rule 6); the list is short
  and the actions are inline, so the modal stack fits. Rejected: a `FriendRequests` screen.
- **Requests stay in All** — decided by Skylar 2026-08-29; the Friends row is the second entry
  point, not the only one.

## Related

- [[offer-engine|Offer Engine — Mobile]] — the Offers tab and inbox avatars
- [[profile|Profile]] — the connect / friend-request actions that create these requests

- **Thread text +1 (2026-08-31).** Conversation bubbles (`bubbleText*`) render at 15
  (`fonts.size.input`) over the body face's 14 — the thread read small; line height 22 stays.
