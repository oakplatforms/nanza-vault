---
tags: [nanza-api, nanza-mobile, solution-design, plan, push, notifications, firebase, inbox]
---

# Push notifications — native iOS / Android alerts for nine inbox events

**Status:** **plan, written 2026-10-30 (Skylar, voice + pasted specs). Nothing built. All five
open questions answered on the call the same day (foreground suppressed; offer copy written by
Claude; Firebase key in Secrets Manager, no SNS; join requests go to the one moderator; message
previews shown). Decisions: Firebase Cloud Messaging carries both platforms; exactly nine events
(Post Liked and Thread Reply are out for now); the OS-level permission is the only switch in v1.
Ready to build. **API half built 2026-10-30, uncommitted (tsc + eslint clean, push Lambda bundles under esbuild, not deployed): migration `20261031120000_devices` to run, types `0.1.136` to publish (`DeviceDto`), `firebaseServiceAccount` to add to `nanza-credentials-dev`.** Mobile half built 2026-10-31 (uncommitted; tsc + lint clean; iOS simulator + Android debug builds pass). **2026-11-01: first end-to-end push delivered to the iPhone in dev** (message + post reply, `[push] sent successCount 1`); Android registration and the full Testing-in-dev script still to run.**

**First-run fixes (2026-11-01).** The dev secret stored `firebaseServiceAccount` as a nested object, not a string — `utils/firebase.ts` now accepts either and normalises literal `\n` in the key. Firebase project is `nanza-8889`. An iPhone on Do Not Disturb shows nothing — check Focus before debugging delivery.

**API build notes (2026-10-30).** Files: `routers/device.ts`, `services/device.ts`, `validation/device.ts`, `services/notification.ts` (`NotificationOutbox`, `pushName`, `pushPreview`), `services/inboxCount.ts` (route now delegates to it), `utils/firebase.ts` (lazy Admin singleton from the secret), `lambdas/pushHandler.ts`, the `push` block in `serverless.yml` (rule `detail-type: [{ prefix: notification. }]`). Call sites: `message.ts` send, `connection.ts` request (tx now returns the intro conversation id) + accept, `group.ts` join request + accept (queued in the tx, flushed after), `post.ts` reply (gated by `isRootPostViewable(parent, parentAuthor)`; `validateParentPost` now selects `accountId`), offers via an optional `outbox` on `notifyOfferReceived` / `closeOfferForOrder`, passed by `sendOffer`, `declineOffer`, `acceptOrder` and `cancelPendingOrder` (seller decline of a listing offer) — each flushes after its own `$transaction`. Cancel / expire / payment-failed closes pass no outbox, so they stay in-app only. Spans nanza-api (device registry, one notification chokepoint, a `push` Lambda) and
nanza-mobile (Firebase SDK, permission moment, a `PushBridge` that routes a tap). When it lands,
this file becomes the durable *Push notifications* design and [[infra-push]] shrinks to a
"history" appendix of it.

## The ask (Skylar, 2026-10-30, voice)

Closed-app, **system-level** notifications — the ones a phone shows on the lock screen and that
a user turns on or off in the OS settings. Nothing in-app-only: the open-app case is already the
WebSocket ([[realtime-in-app-updates]]). Nine triggers:

| # | Event | Who hears it |
|---|---|---|
| 1 | New message | every other participant of the conversation |
| 2 | Friend request received | the person requested |
| 3 | Friend request accepted | the person who sent the request |
| 4 | Post reply | the author of the parent post |
| 5 | Group join request | the group's moderator (its creator — co-moderators are not a thing, Skylar 2026-10-30) |
| 6 | Group request accepted | the requester |
| 7 | Offer received | the offer's recipient |
| 8 | Offer accepted | the offer's sender |
| 9 | Offer declined | the offer's sender |

Skylar's cross-cutting rules (pasted 2026-10-30) are quoted in §Rules, each with where the code
enforces it.

## How push works, end to end (plain words)

Push is three handshakes, and we own only the middle one.

1. **The phone hands the app an address.** When the app asks for permission and the user says
   yes, the OS gives the app a *device token* — an opaque string that means "this app on this
   phone". On Android it comes from Firebase Cloud Messaging (FCM). On iOS it comes from Apple
   (APNs), and the Firebase SDK on the phone swaps it for an FCM token so **our server only ever
   sees one kind of token**. Tokens change (reinstall, OS restore), so the app re-sends on refresh.
2. **The app tells us the address; we remember it per account.** `PUT /device` stores the token
   against the signed-in account. Sign-out deletes it, so the next person to sign in on that phone
   never receives the previous person's alerts.
3. **Something happens, we tell Firebase, Firebase tells the phone.** The API emits an
   EventBridge event (the same spine the WebSocket nudges and emails already ride). A new
   **`push` Lambda** consumes it, looks up the recipient's devices, and calls Firebase with the
   title, body, and a small `data` map saying what to open. Firebase delivers to Android directly
   and to iPhones through APNs using the APNs auth key uploaded once to the Firebase console.
   The OS draws the banner — **nothing of ours runs on the phone for a closed app**.
4. **A tap opens the app at the right place.** The app reads the `data` map and navigates
   (conversation, post, group, offer); if the thing is gone, it lands in the Inbox.

```
domain change (message / connection / reply / group / offer)
  → API handler: queue a notification, commit the transaction, then emit
      → EventBridge (Source: nanza, DetailType: notification.<kind>)
          → push Lambda: Device rows for the recipient → firebase-admin sendEachForMulticast
              → FCM → Android · APNs → iPhone → OS banner, app badge
                  → tap → PushBridge → navigate to target (or Inbox)
```

The in-app path is untouched and runs beside it: the same change still emits the realtime nudge
for an open app ([[realtime-in-app-updates]]), and the same `createSystemMessage` rows still
feed the Inbox ([[system-messages]]). Push is a third consumer, not a replacement.

## Decisions

- **Firebase Cloud Messaging for both platforms, not Amazon SNS.** One server SDK
  (`firebase-admin`), one token type, one mobile SDK (`@react-native-firebase/messaging`);
  Firebase handles APNs for us. SNS would still need the APNs key and FCM credentials uploaded
  into two "platform applications" and adds a hop that buys nothing. The Firebase project
  (`nanza-a1ff7`) and the APNs setup history already exist ([[infra-push]]). FCM itself is free.
- **The old "EventBridge never invoked the Lambda" fear is closed.** That failure was
  root-caused in the WebSocket work: a `eventBus: default` line made the generated Lambda
  permission's `SourceArn` disagree with the rule's real ARN ([[realtime-in-app-updates#Debug log]]).
  The `push` function copies the `fanout` block, which omits that line and has run since.
- **Push rides EventBridge, not an inline call from the request.** The API handler must not wait
  on Firebase (latency, retries, a third-party outage must never fail a message send). The
  emit is best-effort and awaited before `res.json()` — same posture as the realtime emits.
- **Device tokens live in Postgres, as a `Device` model.** They are durable per-account state
  that must cascade on account delete and be visible in admin tooling — unlike the ephemeral
  socket registry, which went to DynamoDB because it churned with sessions. The `push` Lambda
  therefore packages Prisma (like the crons) and gets `reservedConcurrency: 5`, so push can never
  claim more than five Postgres slots; if that cap ever queues visibly, the escalation is a
  `oak-api-devices-<stage>` DynamoDB table mirrored from the socket registry.
- **Generic naming, no vendor columns** (2026-10-24 rule): the model is `Device` with
  `platform` + `token`, not `FcmToken` / `DeviceToken`. The token is the phone's push address,
  the one thing push cannot work without; nothing else of Firebase's is stored.
- **One chokepoint on the API: `services/notification.ts`.** Nine call sites, one function.
  It owns the rules (self, block, visibility), the copy table, the collapse key, and the emit.
  Routers never call `firebase-admin` or `PutEvents` for push directly.
- **Fire after commit, never inside the transaction.** `createSystemMessage` emits its realtime
  nudge inside the caller's `tx` because a rollback only costs a harmless refetch. A push for a
  rolled-back offer is a lie on someone's lock screen. So notifications are **queued** on an
  outbox object the caller passes beside `tx` and **flushed** once `$transaction` resolves.
- **Newer replaces older** via FCM's collapse mechanics: `apns-collapse-id` on iOS and
  `android.notification.tag` on Android, both set to the collapse key (`conversation:<id>`,
  `post:<id>`). One live banner per conversation / per post; the rest stack.
- **Badge = Inbox All unread count.** The `push` Lambda computes it with the same logic as
  `GET /inbox-count` (extracted to `services/inboxCount.ts`, shared by the route and the
  Lambda) and sends it as `aps.badge` / `android.notification.notificationCount`. In the app,
  whenever `useInboxCount` resolves, `setAppBadge(count)` (the `NanzaBadge` module) mirrors it, so reading things
  drops the badge without a push.
- **Foreground = suppressed in v1.** When the app is open, Firebase hands the message to JS
  instead of drawing a banner; the open app already updates through the socket, so v1 shows
  nothing — which also satisfies "never notify about what is already on screen" trivially.
  An in-app banner for *other* screens is a later layer (open question 1).
- **Permission asked with context, never at launch.** The prompt fires after the first
  message sent or the first friend request sent (Skylar's rule), once; a decline sends the user
  to a Settings shelf that deep-links to the OS notification settings.
- **The OS switch is the only switch in v1.** No per-type toggles; the nine events are the
  product. A `Settings → Notifications` shelf shows *On* / *Off — open Settings*.
- **Server credential: Firebase service-account key in Secrets Manager**
  (`nanza-credentials-<stage>` → `firebaseServiceAccount`, read through the existing 30-minute
  cache). Keyless Workload Identity Federation is the cleaner end state but cost a session of
  binding debugging last time ([[infra-push#Firebase Workload Identity Federation]]); the key
  path is the pattern the repo already has. **Not Amazon SNS** (Skylar, 2026-10-30): Android
  needs Firebase regardless (FCM is the only Android transport), so SNS would only move the
  credential into an SNS platform application at the price of one more AWS resource, an
  endpoint ARN per device, and a second token flow — "if we need Firebase anyway, don't add
  more infrastructure; the secret in Secrets Manager is the cleaner solution."
- **Lock-screen previews show the message text** (Skylar, 2026-10-30, on the recommendation):
  iMessage / WhatsApp / Instagram / Discord all do; hiding previews on a locked phone is the
  OS's per-user setting, not ours.

## Rules → where each one lives

| Rule (Skylar) | Enforced by |
|---|---|
| Never notify someone about their own action | `notification.ts` drops any recipient equal to the actor; every call site also passes the actor explicitly |
| Never notify across a block, either direction | `notification.ts` runs `findConnectionBetween(actor, recipient)` and drops `BLOCKED` (symmetric, like `notBlockedFilter`) |
| Only notify if the recipient can see the content | Post reply: `isRootPostViewable(rootId, parentAuthorId)` — covers group membership revoked and blocks. Messages / connections / offers / groups are party-scoped by construction |
| Drafts never trigger | Replies cannot be drafted (`post.ts` refuses), so the reply path only sees live rows; the create path checks `!isDraft` anyway |
| Thread Reply skips the parent's author and the replier | Out of scope (one-level replies only today; Thread Reply is not one of the nine) |
| Don't notify about something open on screen | v1 foreground suppression in `PushBridge` (`onMessage` returns without display) |
| Newer replaces older — one per conversation, one Post Liked per post | collapse key `conversation:<id>`; Post Liked out of scope, the key scheme (`post:<id>`) is reserved for it |
| App badge = Inbox All unread count | `aps.badge` from `services/inboxCount.ts` + `setAppBadge` mirror in `useInboxCount` (iOS) |
| Target gone → open the Inbox | `PushBridge` resolves the target with a GET first; a 404 / handled request falls back to the Inbox tab |
| Ask permission with context, not at launch | `usePushPermissionMoment` fires after the first message / first friend request mutation succeeds |

## The nine, with copy and target

`{name}` is `getProfileName` on the API side (display name, else `@username`, the same helper
the system-message titles use). `{preview}` is the body trimmed to ~100 characters on a word
boundary; a body-less post shows "Sent a photo" / "Sent a link" by its primary item.

| # | Kind | Fires from | Recipient(s) | Title | Body | Opens | Collapse |
|---|---|---|---|---|---|---|---|
| 1 | `message.created` | `POST /message` (`routers/message.ts`) | other participants | `{name}` | `{preview}` | `ConversationThread { conversationId }` | `conversation:<id>` |
| 2 | `connection.requested` | `POST /connection` | recipient | New friend request | `{name}` wants to connect | the intro thread `ConversationThread { conversationId }` (the pending request lives there, with Accept / Decline) | `connection:<id>` |
| 3 | `connection.accepted` | `PUT /connection/:id/accept` | initiator | You're connected | `{name}` accepted your request. Say hi | that 1:1 `ConversationThread` (`findOneToOneConversation`) | `connection:<id>` |
| 4 | `post.replied` | `POST /post` when `parentId` set and not a draft | parent post's author | `{name}` replied to your post | `{preview}` of the reply | the **parent** post: `PostDetail` after `GET /post/:parentId` | `post:<parentId>:reply` |
| 5 | `group.join.requested` | `POST /group/:id/members` | `group.moderatorId` (the one moderator; `GroupMemberRole.MODERATOR` is unused — leave it) | `{group}` | `{name}` asked to join | `GroupMembersScreen { groupId, initialTab: 'requested' }` | `group:<id>:requests` |
| 6 | `group.join.accepted` | `PUT /group/:id/members/:memberAccountId` → `ACTIVE` | the requester | `{group}` | You're in. Welcome to the group | `GroupDetailScreen { groupId }` | `group:<id>` |
| 7 | `offer.received` | `notifyOfferReceived` (`services/offerStatus.ts`) | offer recipient | `{name}` sent you an offer | Wants to buy `{qty}`× `{card}` for `{price}` *(bid offer: Wants to sell you …)*. You have 3 days to answer | `OfferMessageDetail { offerId }` | `offer:<id>` |
| 8 | `offer.accepted` | `closeOfferForOrder(ACCEPTED)` | offer sender | `{name}` accepted your offer | Listing offer: You bought `{qty}`× `{card}` for `{price}`. Your order is on the way · Bid offer: You sold `{qty}`× `{card}` for `{price}`. Time to ship | `OfferMessageDetail { offerId }` | `offer:<id>` |
| 9 | `offer.declined` | `closeOfferForOrder(DECLINED)` | offer sender | `{name}` declined your offer | `{qty}`× `{card}` for `{price}` *(no nudge to re-offer — allowed, not promoted, the inbox rule)* | `OfferMessageDetail { offerId }` | `offer:<id>` |

Rows 7–9 (Claude, 2026-10-30, at Skylar's request) follow the inbox rows in `offerStatus.ts`
so a push and its Inbox row say the same thing. All nine are instant — no batching, no digest.

The mobile targets are exactly what the Inbox rows navigate to today
(`MessagesScreen/index.tsx`: `ConversationThread`, `OfferMessageDetail`, `GroupMembersScreen`
with `initialTab: 'requested'`, `GroupDetailScreen`, `PostDetail`), so a push tap and an inbox
tap land on the same screen.

## nanza-api

### Schema (migration `20261031120000_devices`)

```prisma
enum DevicePlatform {
  IOS
  ANDROID
}

model Device {
  id         String   @id @default(cuid())
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
  lastSeenAt DateTime @default(now())

  account   Account @relation(fields: [accountId], references: [id], onDelete: Cascade)
  accountId String

  platform DevicePlatform
  token    String         @unique

  @@index([accountId])
}
```

Enum in the enums block, model with the models (schema rules). `Account.devices Device[]`.
`token` is unique on its own, not per account: a phone that signs into a second account moves
its row (upsert on `token` re-points `accountId`), which is the "no cross-account leakage" rule
in one constraint.

### Routes (`routers/device.ts`, mounted in `all_routes.ts`)

- `PUT /device` — body `{ accountId, token, platform }`; `validateAccount(..., 'authenticated')`;
  upsert by `token` (set `accountId`, `platform`, `lastSeenAt`). Idempotent, called on every app
  foreground + token refresh. **PUT, not POST** — it's an upsert.
- `DELETE /device/:token` — the signed-in account's own row only; called on sign-out before the
  Cognito session clears. A token the account doesn't own is a 404, never someone else's delete.
- No GET — nothing reads devices but the `push` Lambda.

### Notification chokepoint (`services/notification.ts`)

```ts
type NotificationKind =
  | 'message.created' | 'connection.requested' | 'connection.accepted'
  | 'post.replied' | 'group.join.requested' | 'group.join.accepted'
  | 'offer.received' | 'offer.accepted' | 'offer.declined'

type NotificationTarget =
  | { type: 'conversation'; id: string }
  | { type: 'post'; id: string }
  | { type: 'group'; id: string }
  | { type: 'groupRequests'; id: string }
  | { type: 'offer'; id: string }

type QueuedNotification = {
  kind: NotificationKind
  actorId: string            // whose action — never a recipient
  recipientIds: string[]
  title: string
  body: string
  target: NotificationTarget
  collapseKey: string
}

export class NotificationOutbox { queue(n: QueuedNotification): void; flush(): Promise<void> }
```

`flush()` runs the rules (drop actor, drop blocked pairs via `findConnectionBetween`, the
caller has already applied content visibility) and emits **one EventBridge entry per recipient**,
`Source: nanza`, `DetailType: notification.<kind>`, detail
`{ accountId, kind, title, body, target, collapseKey }` — the same contract shape as
`emitRealtimeEvent` so the `push` Lambda mirrors `fanout`. Best-effort with its own try/catch;
callers `await outbox.flush()` after their `$transaction` and before `res.json()` (un-awaited
work after the response can be frozen by the Lambda runtime — the realtime design's lesson).

Call-site shape, every time:

```ts
const outbox = new NotificationOutbox()
const result = await prisma.$transaction(async (tx) => {
  …
  outbox.queue({ kind: 'offer.received', … })   // inside: cheap, no I/O
  return …
})
await outbox.flush()                             // outside: after commit
res.json(result)
```

`offerStatus.ts` functions (`notifyOfferReceived`, `closeOfferForOrder`) gain an `outbox`
parameter beside `tx`; their callers (offer router send / accept / decline, `acceptOrder`, the
cancel cron) flush after their own transaction. The group and connection routers queue inline.
`message.ts` has no multi-step tx concern but uses the same shape for uniformity.

The copy for each kind lives in a `copy` table in this file (titles / bodies above), next to
the collapse-key scheme — one place to read when wording changes.

### `push` Lambda (`lambdas/pushHandler.ts`)

Copies the `fanout` block in `serverless.yml`:

```yaml
push:
  name: oak-api-push-${sls:stage}
  handler: lambdas/pushHandler.handler
  package:
    patterns: ${self:custom.dbPackagePatterns}
  reservedConcurrency: 5
  events:
    - eventBridge:          # no `eventBus: default` — see the fanout comment
        pattern:
          source: [nanza]
          detail-type:
            - prefix: notification.
```

Per event: load `Device` rows for `detail.accountId` (none → return, logged); compute the badge
with `computeInboxCount(accountId)`; build one multicast message:

```ts
{
  tokens,
  notification: { title, body },
  data: { kind, targetType, targetId, collapseKey },          // strings only
  android: { priority: 'high', collapseKey, notification: { channelId: 'nanza', tag: collapseKey, notificationCount: badge } },
  apns: { headers: { 'apns-collapse-id': collapseKey, 'apns-priority': '10' },
          payload: { aps: { badge, sound: 'default', 'thread-id': collapseKey } } },
}
```

`sendEachForMulticast` returns per-token results; `messaging/registration-token-not-registered`
and `messaging/invalid-registration-token` delete that `Device` row (self-healing, the push
twin of fanout's 410 prune). Other failures log name + message and move on. Firebase is
initialised once per container with the service-account key from `getSecrets()`.

### Serverless / IAM

- `firebase-admin` dependency (server only).
- No new IAM beyond what `fanout` has: `events:PutEvents` already granted; Secrets Manager read
  already granted to the api role.
- `custom.firebaseProjectId` is unnecessary — the service-account JSON names the project.

### Types

`DeviceDto`, `DevicePlatform` land in `@oakplatforms/types` on the usual regenerate + version
bump; mobile only needs the request shape.

## nanza-mobile

### Dependencies

- `@react-native-firebase/app` + `@react-native-firebase/messaging`, **pinned at `26.4.1`** — the
  token, iOS permission, `onNotificationOpenedApp`, `getInitialNotification`, `onTokenRefresh`
  (modular API: `getMessaging()` + free functions).
  - **Why not 27 (built 2026-10-31):** v27 dropped CocoaPods for the Firebase SDK and requires
    Swift Package Manager with `use_frameworks! :linkage => :dynamic` — every pod in the app as a
    dynamic framework. 26.x still supports `$RNFirebaseDisableSPM = true` + static frameworks,
    the smaller change. **Revisit:** Firebase stops publishing CocoaPods releases after October
    2026 (pod install prints the notice), so the next Firebase upgrade means moving to SPM +
    dynamic frameworks — its own tested change.
- **No `@notifee/react-native`** (planned, dropped 2026-10-31): Invertase archived it in April
  2026 and it was never validated on the new architecture; RN 0.83 is new-architecture only. Its
  two jobs are a few native lines instead:
  - the Android channel `nanza` (importance high) is created in `MainApplication.onCreate`;
  - the iOS badge is a tiny `NanzaBadge` module defined inside `AppDelegate.mm` (no Xcode project
    edit), `setCount` → `UNUserNotificationCenter setBadgeCount`. JS: `services/push/badge.ts`.
    Android has no badge-number API; its launcher dot follows the notifications.
- Permission: `requestPermission`/`hasPermission` from messaging on iOS (deprecated in RNFirebase
  but still shipped in 26.x — swap to `react-native-permissions` if a later version removes
  them); `PermissionsAndroid` `POST_NOTIFICATIONS` on Android 13+, implicit below.
- iOS Podfile: `$RNFirebaseAsStaticFramework = true`, `$RNFirebaseDisableSPM = true`, and
  `use_frameworks!` now defaults to `static` (`USE_FRAMEWORKS` still overrides). The first
  `pod install` needs `--repo-update` (the Firebase 12.19 specs).

### Native setup

- **iOS:** `GoogleService-Info.plist` in `ios/nanza/` (client config, not a secret — same class
  as the Airbridge token in `Info.plist`, safe to commit); Xcode capability *Push Notifications*
  (adds `aps-environment` to `nanza.entitlements`) and *Background Modes → Remote notifications*
  (`UIBackgroundModes: remote-notification` in `Info.plist`); `[FIRApp configure]` first thing in
  `AppDelegate.mm didFinishLaunchingWithOptions`. Firebase swizzles the APNs registration
  callbacks, so no further AppDelegate code. Physical device only — the Simulator has no APNs.
- **Android:** `google-services.json` in `android/app/`; `com.google.gms:google-services`
  classpath in `android/build.gradle`, `apply plugin: 'com.google.gms.google-services'` in
  `android/app/build.gradle`; `POST_NOTIFICATIONS` in the manifest (runtime permission on
  Android 13+, auto-granted below); default channel meta-data
  `com.google.firebase.messaging.default_notification_channel_id = nanza`. The channel itself is
  created in `MainApplication.onCreate` on every launch (a no-op once it exists).
- **Android small icon (2026-11-01):** `res/drawable/ic_notification.xml` — the `logo-simple.svg` mark as a white vector — set as `default_notification_icon` in the manifest, tinted by `@color/notification_accent` (primary 500) via `firebase.json` `messaging_android_notification_color`. Without it Android flattens the launcher icon into a blank shape; iOS always uses the app icon.

### `PushBridge` (`components/global/PushBridge`, mounted beside `RealtimeBridge`)

Headless, inside `SessionProvider`, and it does four things:

1. **Register.** When `isSignedIn && accountId` and permission is granted: `getToken()` →
   `deviceService.register({ accountId, token, platform })`. Repeats on `onTokenRefresh` and on
   each foreground (cheap upsert, keeps `lastSeenAt` honest). Deferred behind `isAppReady` like
   the socket — nothing above the fold needs it.
2. **Unregister on sign-out.** `signOut()` in `services/auth/cognito.ts` grows a pre-step: read
   the cached token, `DELETE /device/:token` (best-effort), `messaging().deleteToken()`. Runs
   while the session is still valid, so the delete is authorised.
3. **Route a tap.** `getInitialNotification()` (cold start) and `onNotificationOpenedApp()`
   (background) both yield `data`. The bridge waits for the navigator (`navigationRef.isReady()`)
   and `isAppReady`, then resolves the target:
   - `conversation` → `ConversationThread { conversationId }`
   - `post` → `postService.get(id)` → `PostDetail { post, postableType, postableId }`; 404 →
     Inbox
   - `group` → `GroupDetailScreen { groupId }`; `groupRequests` → `GroupMembersScreen
     { groupId, initialTab: 'requested' }`
   - `offer` → `OfferMessageDetail { offerId }`
   - anything else, or any fetch error → the Inbox tab (`MessagesScreen`)
   A guest or signed-out cold start parks the target until the session resolves to signed-in,
   then routes; signed-out stays on Home (the same latch pattern as `AttributionBridge`).
4. **Foreground.** `onMessage` returns without displaying (v1). The socket's invalidation
   already refreshed the badge and lists; the handler only calls `setBadgeCount` from the
   payload so the icon stays consistent.

Why a `data` target and a bridge rather than `nanza://…` URLs through the existing `linking`
config: `PostDetail` takes a hydrated post object, not an id, and the group members tab is a
param, not a path. Encoding those as URLs means a second resolver anyway; the bridge is that
resolver, and it's the same shape `AttributionBridge` already uses.

### Permission moment (`hooks/usePushPermissionMoment.ts`)

- **Trade moments (Skylar, 2026-11-01):** creating a listing or bid (`CreateListing`, `CreateBid`, `EditBid`'s create path) asks "Know when you get an offer"; sending an offer (`MakeOfferScreen`, listing and bid offers) asks "Know when they answer" over the confirmation screen. Copy lives in `PUSH_PROMPT_COPY` in the hook. Same shared answer, snooze and cap as every moment.
- **Third moment (Skylar, 2026-10-31):** publishing a top-level post (fresh or a published draft; never a reply) in `PostComposer` asks with its own copy — "Know when someone replies". All three moments share the one per-phone flag, so whichever comes first is the only time the sheet shows. **2026-11-01:** the flag is written only when a sheet button is tapped (or the OS already has an answer) — on the Moto it was written before the sheet showed, the sheet never appeared, and the phone was never asked; a swiped-away or unseen sheet now asks again at the next moment. **Snooze (Skylar, 2026-11-01):** "Not now" snoozes the sheet 14 days and it returns at the next moment after that, capped at three "Not now" answers; "Turn on" ends it (the OS permission decides). State is local only (`push.prompt` in AsyncStorage: `done`, `notNowCount`, `lastAnsweredAt`); the old once-only `push.promptShownAt` flag carries over as one "Not now". Buttons: "Not now" / "Turn on". Receiving alone never prompts; opening a conversation as a fourth moment was offered, not built.
- After `useSendMessage` or `useCreateConnection` succeeds for the **first time** (an
  AsyncStorage flag `push.promptShownAt`, per device), show a one-screen sheet: "Know when
  someone writes back" with *Turn on notifications* / *Not now*. *Turn on* →
  `messaging().requestPermission()` (iOS) / `PermissionsAndroid.request(POST_NOTIFICATIONS)`
  (Android 13+), then the bridge registers.
- *Not now* or an OS decline never re-prompts; the **Settings → Notifications** shelf
  (`AccountScreen/Settings`) shows the live status (`hasPermission()`) and *Open Settings* →
  `Linking.openSettings()`. Turning it on there triggers registration on the next foreground.
- Android 12 and below: permission is implicit, so the sheet skips straight to registering.

### Badge mirror

`useInboxCount` gains an effect: whenever `query.data.count` changes, `setAppBadge
(count)`. Sign-out sets it to 0. The server's `aps.badge` and the app's mirror then agree by
construction — both are the Inbox All unread count.

### Mobile build plan (nanza-mobile P1 — confidence scores, 2026-10-30)

Touches more than five files and native config, so it needs Skylar's go-ahead under
nanza-mobile's escalation rules. No navigation file is edited — `navigationRef` is already
exported from `AppNavigator` and `navigateToSearch.ts` uses it the same way.

| Decision | Confidence | Rejected alternative |
|---|---|---|
| `@react-native-firebase/app` + `/messaging` for the token and tap events | 0.9 | `react-native-notifications` + raw APNs from the Lambda — a second server send path for iOS |
| iOS Podfile switches to `use_frameworks! :linkage => :static` (RNFirebase requires it) | **0.7** | Keep dynamic linkage and skip RNFirebase on iOS (raw APNs) — rejected above. Risk: a pod that breaks under static frameworks (Airbridge, vision-camera, Stripe all document static support); verified by a clean `pod install` + device build |
| Guard native init until the config files exist (`[FIRApp configure]` only if the plist is bundled; Android `google-services` plugin applied only if the JSON exists) | 0.9 | Unguarded — every build crashes / fails until Skylar's Step 2 lands |
| ~~`@notifee/react-native` for the Android channel + badge~~ → native channel in `MainApplication` + `NanzaBadge` in `AppDelegate.mm` (2026-10-31: notifee archived) | 0.85 | notifee, or its community fork `react-native-notify-kit` — an unproven dependency for ~20 native lines |
| `PushBridge` headless component beside `RealtimeBridge` in `App.tsx`, routing through the existing `navigationRef` | 0.9 | `linking` URLs — `PostDetail` takes a hydrated post, so a resolver is needed anyway |
| Permission sheet after the first message / friend request, AsyncStorage flag, Settings shelf with *Open Settings* | 0.85 | Ask at launch — rejected by Skylar's rule |
| Badge mirror inside `useInboxCount` | 0.9 | A separate hook — duplicates the query |

Files: `package.json`, `ios/Podfile`, `ios/nanza/AppDelegate.mm`, `android/build.gradle`,
`android/app/build.gradle`, `AndroidManifest.xml`, `App.tsx`, new
`components/global/PushBridge`, new `services/api/Device.ts`, new `services/push/*`,
`services/auth/cognito.ts` (sign-out unregister), `hooks/useInboxCount.ts`, the first-message /
first-request mutation hooks, a sheet component + its style bucket, the Settings screen,
`lang/en.json`.

## What is NOT in v1

- Post Liked and Thread Reply (Skylar, 2026-10-30 — "not right now"). The rules table keeps
  their collapse-key slots so adding them is one `kind` each.
- Per-type toggles, quiet hours, digests. The OS switch is the only control.
- In-app banners while the app is open (open question 1).
- Order and refund events — they already email, and the Inbox has them; add later via the same
  chokepoint if wanted.
- Web (`nanza-web-app`) push.
- **Sender avatar / rich image in the notification** (Skylar, 2026-11-01: "this is fine" for v1). Android's small icon is a one-colour glyph by platform rule (the pink Nanza mark); a full-colour large icon needs data-only messages rendered on the phone (a notification library again), and iOS needs a Notification Service Extension. A v2 item for both platforms together.

## Questions — all answered 2026-10-30 (Skylar, voice)

1. **Foreground:** suppressed entirely in v1 (on the recommendation). An in-app banner when the
   target is not the current screen needs a screen-identity registry — phase 2 if wanted.
2. **Offer copy (7–9):** Claude writes it, mirroring the inbox rows — done above.
3. **Server credential:** service-account key in Secrets Manager, not SNS, not WIF (see
   Decisions). WIF can replace it later with no code change beyond `initializeApp`'s credential.
4. **Group moderators:** co-moderators are not a thing — the join request goes to
   `Group.moderatorId` only.
5. **Message preview:** show the text; the OS owns lock-screen privacy.

## Build order

1. **Firebase console + Apple** (Skylar, human steps, can run in parallel with 2 and 3):
   the numbered checklist with a *Done when* per step is
   [[../../_shared/onboarding/push-notifications|Onboarding — Push notifications]] — Firebase
   project, the two app config files, a new APNs auth key (the old one was revoked) uploaded
   with Key ID + Team ID, the Xcode capabilities, the service-account key stored as
   `firebaseServiceAccount` in `nanza-credentials-dev`, then the first console test message.
2. **API:** `Device` model + migration, `routers/device.ts`, `services/notification.ts` with the
   nine call sites, `services/inboxCount.ts` extraction, `lambdas/pushHandler.ts` +
   `serverless.yml` block, `firebase-admin`. Types regen + bump. Deploy dev; prove one push with a
   hard-coded token from the Firebase console's *Send test message* first (step 1 of the
   realtime plan's lesson — prove the transport before the client).
3. **Mobile:** deps + native setup, `PushBridge`, sign-out unregister, permission sheet +
   Settings shelf, badge mirror, `deviceService`. Device QA on a physical iPhone and the Moto
   test phone (Android 15: `POST_NOTIFICATIONS` runtime prompt).
4. **Prod:** the Firebase project is shared; only the prod secret and a prod deploy are new.

Rough size: API ~1.5 days, mobile ~2 days, console + QA ~0.5 day.

## Testing in dev (Skylar, 2026-10-30: both platforms proven in dev before any prod release)

Two physical phones — an iPhone (the Simulator has no APNs) and the Moto test phone (Android
15, runtime permission) — on debug builds pointed at the dev API, signed in as two test accounts
**A** and **B**. Firebase routes a debug iOS build to the APNs sandbox by itself (it reads the
`aps-environment: development` entitlement), so no separate dev key is needed.

**Stage 0 — transport only, no app code.** Install the app with the Firebase SDK wired, copy
the token from a Metro `console.log`, and send it from the Firebase console (*Cloud Messaging →
Send test message*). Banner on the lock screen on both phones = Firebase + APNs key + native
setup are right. Nothing else is worth debugging until this passes.

**Stage 1 — API path.** `PUT /device` from the app (row appears for A); then
`aws events put-events` is off-limits (CLAUDE.md), so trigger the real route instead: B sends A
a message from the other phone. Watch the dev `oak-api-push-dev` log group for `[push] resolved
devices` → `[push] sent` with per-token results; the banner follows within seconds.

**Stage 2 — the nine, A ↔ B, each phone once as recipient.** Tick each:

| # | Do | Expect on the recipient's locked phone | Tap lands on |
|---|---|---|---|
| 1 | B messages A twice | one banner, the second **replaces** the first (collapse) | that conversation |
| 2 | B sends A a friend request | "New friend request — B wants to connect" | the intro thread with Accept / Decline |
| 3 | A accepts | B: "You're connected — A accepted your request. Say hi" | the A↔B conversation |
| 4 | B replies to A's post | "B replied to your post — {preview}" | A's post (parent) |
| 5 | B requests to join A's group | "{group} — B asked to join" | members screen, Requested tab |
| 6 | A accepts | B: "{group} — You're in. Welcome to the group" | the group |
| 7 | B sends A an offer | "B sent you an offer — Wants to buy …" | the offer |
| 8 | A accepts / 9 declines | B: "A accepted / declined your offer" | the offer |

**Stage 3 — the rules.** A messages B while B is **in that conversation** (foreground): no
banner, thread updates live. A blocks B, B messages / replies: nothing arrives for A. B replies
to A's **group** post after A left the group: nothing. A deletes the post, B taps the old reply
banner: lands in the Inbox. A signs out on the iPhone, B messages: nothing on that phone; sign
A back in: pushes resume (token re-registered). Badge: after 2 unread + 1 offer the icon shows
3; open the Inbox, badge drops to 0 on both phones. Deny permission on the Moto, confirm the
Settings shelf reads *Off — Open Settings* and that turning it on there registers on the next
foreground.

**Done when** every row in stage 2 and every case in stage 3 passes on **both** phones, with the
`oak-api-push-dev` log showing zero `post failed` lines for valid tokens. Only then: prod secret,
prod deploy, same stage-0 test on a TestFlight / internal-track build.

## Related

- [[infra-push]] — the removed 2026-06 prototype and the APNs / WIF setup history this reuses
- [[realtime-in-app-updates]] — the open-app sibling (WebSocket) and the EventBridge permission gotcha
- [[system-messages]] — the Inbox rows the nine events already create
- [[db-connections]] — why the push Lambda is concurrency-capped
- [[../../nanza-mobile/solution-designs/inbox|Inbox (mobile)]] — the screens a tap lands on
