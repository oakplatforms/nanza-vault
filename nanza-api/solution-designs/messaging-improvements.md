---
tags: [nanza-api, nanza-mobile, solution-design, plan, inbox, messaging, realtime, reactions]
---

# Messaging improvements — typing indicator, reactions, edit / delete, long-press sheet

**Status:** **plan, written 2026-11-01 (Skylar, voice + two Discord screenshots). API + mobile built the same day, uncommitted (tsc + eslint clean in both repos; migration `20261101120000_message_reactions` to run, types `0.1.138` to publish then `npm run types:update` in mobile, dev WebSocket redeploy for typing); pending device QA.**
Decisions taken on the call: hard delete (no "message deleted" placeholder); reactions are
multiple per user per message (toggle per emoji, Discord style) because it costs nothing over
one; six quick emoji in the sheet **plus a full searchable picker in this round** ("it makes the
best UX"); no push notification for reactions, in-app live only; edit shows an *edited* marker
and is unlimited in time. Skylar's partner sends Figma styling tomorrow — this round is
functionality; the sheet, chips and typing row are plain and get restyled then.

## Goal

Bring the 1:1 inbox thread up to the Discord / Instagram baseline:

1. **Typing indicator** — "<name> is typing…" at the bottom of the open thread while the other
   participant types.
2. **Reactions** — double-tap any bubble (theirs or yours) to ❤️ it; long-press opens a sheet with
   a quick emoji row (👍 ❤️ 😂 🔥 ‼️ 👋) and a *more* button that opens the full emoji picker.
   Reaction chips (emoji × count, viewer's own highlighted) render under the bubble; tapping a chip
   toggles that emoji for the viewer. Both participants see reactions live.
3. **Long-press sheet** — emoji row, then the hint *Double tap a message to ❤️*, then actions:
   own message → **Edit message · Copy text · Delete message**; other's message → **Copy text**
   only. If the keyboard is up it drops first, then the sheet rises.
4. **Edit / delete** — server-side edit with an *edited* marker; hard delete.

Out of scope this round: reply / forward / pin / mark unread (Discord rows not requested), group
chats (the API loops over "other participants" so it carries over, but no group UI), read
receipts, presence ("online"), reaction push notifications.

## Current state (verified in code 2026-11-01 — build on this)

- **Realtime transport exists and works.** API Gateway WebSocket API, `lambdas/websocketHandler.ts`
  (`$connect` Cognito-authorized via `?token=`, `$disconnect`, `$default` answers `{"action":"ping"}`),
  a **DynamoDB socket registry** (`src/services/socketRegistry.ts`: `registerConnection`,
  `unregisterConnection`, `getConnectionIdsByAccount`, `pruneConnections`), and
  `lambdas/fanoutHandler.ts` fed by an EventBridge rule (`Source: nanza`, detail-types listed in
  `serverless.yml` under `fanout`). Emitters call `emitRealtimeEvent` (`src/utils/realtimeEvent.ts`)
  with `{ type, accountId | accountId[], invalidate, data }`; the phone's
  `src/services/realtime/socket.ts` turns `invalidate` into `queryClient.invalidateQueries`
  (`refetchType: 'active'`) and ignores everything else. The IAM policy already grants
  `execute-api:ManageConnections` to the whole service and the `fanout` function already has
  `WS_API_ENDPOINT`. The `websocket` function does **not** yet have `WS_API_ENDPOINT`.
- **Messages.** `Message { type USER|SYSTEM|INTRO, body, conversationId?, senderId?, systemMessage? }`
  with `@@index([conversationId, createdAt])`. `GET /conversation/:id/messages` (paginated, newest
  first, `include: { sender: { include: { profile } } }`), `POST /conversation/:id/messages`
  (participant + ACCEPTED-connection checks, rate limit, lifetime cap, `Conversation.lastMessageAt`
  bump, `message.created` realtime emit invalidating `messages · conversations · messageFeed ·
  inboxCount`, push via `NotificationOutbox`). No edit, delete or reaction endpoints exist.
- **Inbox preview + unread** both read "the newest message": `conversationInclude` in
  `src/routers/conversation.ts` projects `messages: { take: 1, orderBy: createdAt desc }`, and
  `computeInboxCount` (`src/services/inboxCount.ts`) treats a conversation as unread when the newest
  message is from the other party and `lastMessageAt > my lastReadAt`. **Delete must keep
  `lastMessageAt` consistent with the surviving newest message** or the badge lies.
- **Mobile thread.** `src/components/messages/ConversationThread.tsx` renders `useMessages`
  (`['messages', conversationId, accountId]`, `limit=50`, newest-first reversed) in a plain
  `ScrollView` of `View` bubbles (no gesture handling), a `TextInput` + send button, and the
  pending-intro accept/decline variant. `ActionModal` (`src/components/modals/ActionModal`) already
  takes `rows: ActionRow[]` + `headerContent` — the long-press sheet is an `ActionModal` with the
  emoji row as its header. `@react-native-clipboard/clipboard`, `react-native-gesture-handler` 2.29
  and `reanimated` 4 are installed; `EmojiText` exists for emoji in text runs.

## Design

### 1. Typing indicator — over the socket, no REST, no Postgres on the hot path

Typing is ephemeral and best-effort, exactly what the socket is for. The REST → EventBridge →
fanout chain would work but adds a Lambda + Postgres hit per keystroke burst and ~0.5–1 s of
latency; the socket path is ~100 ms and free.

**Client → server.** While the composer text changes, the thread sends
`{ "action": "typing", "conversationId": "<id>" }` on the open socket, throttled to **one frame
per 3 s** (leading edge). Nothing is sent on an empty input or when the socket is closed. No
explicit "stopped typing" frame — the indicator expires client-side (below), same as Discord's
~10 s typing window.

**Server.** `websocketHandler` `$default` gains a `typing` action beside `ping`:

1. Resolve the sender's `accountId` from the connection. `$default` carries no authorizer
   context, so the registry lookup is by `connectionId` — add `getAccountIdByConnection` to
   `socketRegistry.ts` (a `GetItem` on the PK; the row already stores `accountId`).
2. Verify the sender is a participant of `conversationId` and collect the other participants.
   This is the one Postgres read on the path; cache it per container in a
   `Map<conversationId, { participantIds, cachedAt }>` with a **10-minute TTL** (membership is
   near-immutable; a blocked pair stops typing frames at worst 10 min late, which is harmless
   because the recipient's thread for a blocked conversation is gone anyway). Same shape as the
   existing `accountIdByPrincipal` cache. Unknown conversation or non-member → drop silently.
3. `getConnectionIdsByAccount` for each other participant, `PostToConnection`
   `{ "type": "typing", "conversationId", "accountId": <typer>, "expiresAt": now + 5000 }` to
   each, prune on 410 (lift the post + prune loop out of `fanoutHandler.ts` into
   `src/services/socketPush.ts` so both Lambdas share it). Needs `WS_API_ENDPOINT` on the
   `websocket` function in `serverless.yml` (same `Fn::Sub` as `fanout`).

**Client ← server.** `socket.ts` grows a tiny subscription API — `subscribe(type, handler)` —
and routes non-`invalidate` frames to it. A `useTypingIndicator(conversationId)` hook subscribes
to `typing` frames for its conversation and holds `typingAccountIds` with a per-account expiry
timestamp; it clears an account **5 s after its last frame** or immediately when a
`message.created` invalidation for that conversation arrives (the message landed). The thread
renders `"<getProfileName(other)> is typing…"` as a muted row between the message list and the
input (the three-dot animation is the partner's styling pass). The hook is a timer that *expires
a remote signal*, not UI sequencing, so it does not conflict with the no-timer rule.

### 2. Reactions

**Schema** (enums block untouched; model after `Message`):

```prisma
model MessageReaction {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())

  message   Message @relation(fields: [messageId], references: [id], onDelete: Cascade)
  messageId String

  account   Account @relation(fields: [accountId], references: [id], onDelete: Cascade)
  accountId String

  emoji String

  @@unique([messageId, accountId, emoji])
  @@index([messageId])
  @@index([accountId])
}
```

`Message.reactions MessageReaction[]`, `Account.messageReactions MessageReaction[]`. One row per
(message, account, emoji): multiple reactions per user fall out of the unique key, and a single
`createMany … skipDuplicates` / `deleteMany` is the whole toggle. Migration
`20261101120000_message_reactions`.

**Validation** (`src/validation/message.ts`, new): `validateReactionEmoji(emoji)` — a non-empty
string of **at most 16 code units** that is **one grapheme** whose first code point matches
`\p{Extended_Pictographic}` (keycaps and flags count), so any emoji the picker offers is accepted
and arbitrary text is not. Nothing stores a fixed list: the six quick emoji live in the mobile
sheet only, so the full picker needs no API change.

**Endpoints** (both: participant check as `GET /conversation/:id/messages` does, `USER` or `INTRO`
messages only, own or other's message alike):

- `POST /message/:id/reactions` body `{ accountId, emoji }` → upsert (ignore duplicate), returns
  the message with `reactions`.
- `DELETE /message/:id/reactions/:emoji` (`?accountId=`) → `deleteMany` for the viewer, returns
  the message with `reactions`.

`GET /conversation/:id/messages` and the two new endpoints include
`reactions: { select: { emoji: true, accountId: true, createdAt: true } }` — rows, not counts; a
1:1 thread has at most a handful per message and the client aggregates to chips in insertion
order (first reaction of an emoji fixes the chip's position, Discord style).

**Realtime.** Both endpoints emit a new type `message.updated` to the *other* participants,
`invalidate: ['messages']`, `data: { conversationId, messageId }`. Add `message.updated` to the
`fanout` rule's `detail-type` list. The actor's own phone updates optimistically (below). No
`NotificationOutbox` call: reactions don't push.

**Double-tap = heart.** The client toggles ❤️: if the viewer already has a ❤️ row on that message
→ `DELETE`, else `POST`. Hint row in the sheet reads *Double tap a message to ❤️*.

### 3. Edit and delete

- `PUT /message/:id` body `{ accountId, body }` — sender only, `type === 'USER'` only (INTRO
  greetings and SYSTEM rows are not editable), `validateMessageBody`, sets `body` and a new
  `Message.editedAt DateTime?`. Returns the message. Emits `message.updated`
  (`invalidate: ['messages', 'conversations']` — the inbox preview may show it). Same migration.
- `DELETE /message/:id` (`?accountId=`) — sender only, `USER` only, **hard delete** (reactions
  cascade). In the same transaction, re-point `Conversation.lastMessageAt` to the surviving newest
  message's `createdAt` (or `null` when none) so the inbox order, preview and `computeInboxCount`
  stay truthful. Emits `message.updated` with `invalidate: ['messages', 'conversations',
  'messageFeed', 'inboxCount']`. Returns `{ ok: true }`.
- No edit window, no edit history (Discord has none either).
- The bubble shows a small *edited* suffix when `editedAt` is set.

### 4. Mobile — thread, sheet, picker

- **Bubble gestures.** Wrap each bubble in a `GestureDetector` with
  `Gesture.Exclusive(longPress, doubleTap)`: double-tap → heart toggle; long-press (≈350 ms) →
  `Keyboard.dismiss()` then open the sheet. Plain taps keep doing nothing. The thread stays a
  `ScrollView`; RNGH gestures inside it work without changes.
- **Reaction chips.** Under the bubble, aligned to its side: one chip per emoji
  (`emoji count`), viewer's own chips outlined in the accent. Tap → toggle that emoji.
  Component `MessageReactions` beside `ConversationThread.tsx`.
- **Sheet.** `MessageActionSheet` = `ActionModal` whose `headerContent` is the quick row
  (👍 ❤️ 😂 🔥 ‼️ 👋 + a smiley *more* button) and the hint line. Rows by ownership: mine →
  `edit`, `copy`, `delete` (danger); theirs → `copy`. Picking a quick emoji toggles it and closes.
- **Full picker.** `rn-emoji-keyboard` (pure JS, searchable, categories, recent; no native
  rebuild). The *more* button closes the sheet and opens the picker; a pick toggles that emoji.
  Any emoji it returns passes the API's grapheme check.
- **Copy text** → `Clipboard.setString(body)`, toast via `alert`.
- **Edit** → the composer switches to edit mode: input prefilled with the body, a small
  "Editing message · Cancel" strip above it, send icon becomes a check; submit → `PUT`. Cancel
  restores the draft. (Discord's in-place edit; keeps one `TextInput`.)
- **Delete** → `ConfirmSheetContent` ("Delete message?") then `DELETE`.
- **Mutations** (`MessagesScreen/data/mutations.ts`): `useToggleReaction`, `useEditMessage`,
  `useDeleteMessage`. All three patch `['messages', conversationId]` optimistically
  (`setQueryData`) and roll back on error; delete/edit also invalidate `['conversations']`
  (+ `['inboxCount']` for delete). Services in `src/services/api/Message.ts`: `react`,
  `unreact`, `edit`, `remove`.
- **Types.** `MessageDto` gains `editedAt` + `reactions: MessageReactionDto[]` → publish
  `@oakplatforms/types` **0.1.138**; mobile `npm run types:update`.
- **Typing row** as in §1; name via `getProfileName`.

## Work breakdown

**nanza-api**
1. `schema.prisma`: `MessageReaction` model, `Message.editedAt`, relations; migration
   `20261101120000_message_reactions` (Skylar runs it).
2. `src/validation/message.ts`: `validateReactionEmoji`, `assertMessageSender`,
   `assertReactableMessage`.
3. `src/services/message.ts`: `toggleReaction` helpers, `editMessage`, `deleteMessage`
   (with the `lastMessageAt` re-point).
4. `src/routers/message.ts`: the four endpoints; add `reactions` to `messageInclude`;
   `message.updated` in `realtimeEvent.ts` + `serverless.yml` rule.
5. `src/services/socketPush.ts` (post + prune shared by fanout and websocket),
   `socketRegistry.getAccountIdByConnection`, `typing` action in `websocketHandler.ts`,
   `WS_API_ENDPOINT` on the `websocket` function.
6. `npx prisma generate && npm run generate:types && npm run build:types`, bump to 0.1.138.

**nanza-mobile**
7. `socket.ts`: `subscribe()` + `sendTyping()`; `useTypingIndicator` hook.
8. `ConversationThread.tsx`: gestures, chips, typing row, edit mode in the composer.
9. `MessageActionSheet`, `MessageReactions`, picker wiring (`rn-emoji-keyboard`).
10. Mutations + services + types update.

Order: 1–4 (API, testable with curl) → 7–10 (mobile) → 5 (typing, both sides together).
Roughly two days; typing is the only piece that needs the dev WebSocket redeployed.

## Build log

- **2026-11-01 — API built.** `MessageReaction` + `Message.editedAt` (migration `20261101120000_message_reactions`), `validation/message.ts` (`validateReactionEmoji` accepts pictographic, keycap and flag graphemes; `assertReactableMessage`, `assertEditableMessage`), `services/message.ts` (`messageInclude` with reaction rows, add/remove reaction, edit, delete with the `lastMessageAt` re-point), the four `/message/:id` routes in `routers/message.ts`, `message.updated` in `realtimeEvent.ts` + the fanout rule, `services/socketPush.ts` (post + 410-prune shared by `fanoutHandler` and the new `typing` action in `websocketHandler`, which gets `WS_API_ENDPOINT`), `socketRegistry.getAccountIdByConnection`. Types package regenerated and bumped to `0.1.138` (`0.1.137` had already shipped with push).
- **2026-11-01 — mobile built.** `socket.ts` gains `subscribe(type, handler)` / `send` / `sendTyping`; `hooks/useTypingIndicator.ts` (5 s expiry, cleared on `message.created`); `services/api/Message.ts` `react / unreact / edit / remove`; `useToggleReaction / useEditMessage / useDeleteMessage` with optimistic `setQueriesData` patches + rollback; `MessageReactions` chips, `MessageActionSheet` on `ActionModal` (whose `headerContent` may now be a render-prop receiving the after-dismiss runner), `rn-emoji-keyboard@1.7.0` picker; `ConversationThread` wraps bubbles in `Gesture.Exclusive(longPress, doubleTap)` (`runOnJS`), composer edit mode with an *Editing message · Cancel* strip, typing row, 3 s leading-edge typing ping. `ThreadMessage` is a local intersection on `MessageDto` until `0.1.138` installs.

- **2026-11-01 (night, device QA round 1 — Skylar):** (1) *"The bottom input jumps after I send"*
  — the push permission moment fired on send success and had to `Keyboard.dismiss()` to get its
  sheet above the keys, so the input row dropped and came back. The ask moved out of
  `useSendMessage` into `ConversationThread`: a `sentThisVisit` ref is set on send and the sheet
  is asked on the screen's `blur` / unmount, when the keyboard is already down and the sheet lands
  on the list behind. (2) *"Not seeing the typing indicator"* — the client side checks out
  (`sendTyping` → `$default` → `pushToAccounts` → `useTypingIndicator`); the relay lives in the
  websocket Lambda from commit `5c3673d`, so if dev was last deployed before it, typing frames
  never arrive — redeploy dev and look for `ws-typing` in the websocket Lambda's logs. Same
  round: the indicator now also shows in the conversations LIST (All + Friends-active tabs) as a
  "Typing…" preview in the text colour — `useTypingConversations()` (one subscription, every
  conversation) with `useTypingIndicator(id)` layered on it for the thread.
- **2026-11-01 (night, round 2):** still no indicator → root cause found in `serverless.yml`: the
  socket-registry IAM statement granted Put / Delete / Query / BatchWrite but not **`GetItem`**,
  which `getAccountIdByConnection` (the typing relay's reverse lookup) uses — AccessDenied, caught
  by the handler's catch-all, 200 returned, frame dropped, nothing on the client. `dynamodb:GetItem`
  added; needs a dev deploy (push `dev`, release the pipeline). Verify with `ws-typing` in the
  websocket Lambda's logs.
- **2026-11-01 (night, round 3):** typing state moved out of the hooks into a module-level store
  in `hooks/useTypingIndicator.ts` (socket subscribed once at load, `useSyncExternalStore`
  readers), so opening a thread from a row that says "Typing…" shows the indicator at once
  instead of waiting for the next 3 s ping (Skylar: "weird to see typing… then click in and see
  not typing").

- 2026-11-08 — Thread UX fixes (mobile, uncommitted, pending device QA). (a) Message input sat
  higher after sending a multi-line message: clearing the draft programmatically doesn't reliably
  re-measure the native multiline input, so the box kept its grown height with the placeholder at
  its top (input fill is secondary.950 on a neutral.950 page, so the box itself is near-invisible
  and only the placeholder shows the jump). The EMPTY input is now pinned to its one-line height,
  measured once on first layout (`restingInputHeight`). (b) Long threads: the list now sticks to
  its end on content-size and layout changes (keyboard open shrinks the viewport) unless the reader
  has scrolled up more than `spacing.xl`; sending always re-pins. Replaced the `messages.length`
  effect, which scrolled a frame before the new row laid out. Content gets `base` bottom padding.

## Open questions

None blocking. Styling (sheet, chips, typing dots, edited marker) comes from the partner's Figma
tomorrow and lands as a `/figma-handoff` pass.

## Related

- [[realtime-in-app-updates|Realtime in-app updates]] — the socket + fanout this builds on.
- [[db-connections|DB connections & the socket registry]] — why the typing path stays off Postgres.
- [[push-notifications|Push notifications]] — reactions deliberately do not push.
- [[../../nanza-mobile/solution-designs/inbox|Inbox (mobile)]]
