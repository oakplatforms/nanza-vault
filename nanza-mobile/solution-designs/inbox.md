---
tags: [nanza-mobile, solution-design, inbox, messaging]
---

# Inbox — tabs, friends and friend requests

## Overview

`MessagesScreen` is four tabs over one unified feed: **All**, **Friends**, **Groups**, **Offers**.
Friend requests have **no tab of their own** (Skylar, 2026-08-29): they arrive in **All** as
pending intro threads, and the **Friends** tab surfaces them as one row.

| Tab | Source | Rows |
|---|---|---|
| All | `GET /message-feed` | conversations (incl. pending intros) + every system message, newest first; the only tab with a numeric unread count |
| Friends (was "Messages") | `GET /conversations` (ACCEPTED connections only) + `GET /connections?status=PENDING` | a **"Friend requests (N)"** row when people are waiting on you, then your friends' conversations |
| Groups / Offers | `GET /system-messages?category=` | system rows; offer rows wear the other party's avatar (`systemMessage.actor`) |

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
