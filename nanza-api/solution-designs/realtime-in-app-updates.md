# Plan — Real-time in-app updates (WebSocket) for messages, connections & offers

**Status:** proposed · **Date:** 2026-07-02 · **Repos:** nanza-api, nanza-mobile
**Decision made (voice):** AWS-native — API Gateway **WebSocket API**, no third-party vendor.

---

## Goal (what we're actually building)

When the app is **open**, live-update two things without polling or leaving/re-entering a screen:

1. **Bottom-nav Messages badge** — the unread/unseen count (`useInboxCount`).
2. **Messages tab content** — conversation list & feed auto-refresh (new message → unread at top,
   new connection request, new order/offer surfaces immediately).

**Explicit non-goals:**
- ❌ Closed-app push notifications. Email already covers that. We are **not** rebuilding FCM/APNs.
  See `documentation/solution-designs/infra-push.md` — that push prototype was removed because
  "EventBridge events weren't reliably invoking the sender Lambda." This plan is a *different*
  transport (client WebSocket, app-in-foreground only) and does not touch that history.
- ❌ Interval polling. It was removed for cost; we are not bringing it back.

## Why WebSocket, and why EventBridge alone isn't enough

EventBridge is the event **spine** (the API already emits to it — see below), but EventBridge has
**no client transport** — it can only route between AWS services. So the last hop to the phone needs
a delivery layer. On our stateless Lambda/HTTP-API stack, that hop is **API Gateway WebSocket API**.

**End-to-end flow:**

```
domain change (new message / connection req / offer)
  → API emits PutEvents → EventBridge (default bus)   [extend existing pattern]
      → EventBridge rule (Source: nanza, matched DetailTypes)
          → fanout Lambda: look up user's live connectionIds → post to each
              → API GW WebSocket → 📱 app (foreground)
                  → queryClient.invalidateQueries(['inboxCount'|'conversations'|…])
                      → badge + list update themselves (React Query already reactive)
```

---

## Current-state facts (verified in code — build on these, don't re-derive)

**nanza-api**
- Serverless Framework, Lambda `nodejs20.x` arm64, **HTTP API** (no WebSocket API today).
  `serverless.yml`.
- Auth: Cognito via custom **Lambda request authorizer** (`lambdas/authorizerHandler.ts`), mounted as
  `lambdaAuth` on the HTTP API. Reusable for the WS `$connect` route.
- EventBridge already wired: client `src/utils/eventBridge.ts`; IAM already grants `events:PutEvents`
  on `event-bus/default` (see `provider.iam` in `serverless.yml`). Emit pattern in
  `src/routers/order.ts` (~L1333): `eventBridge.send(new PutEventsCommand({ Entries: [{ Source:
  'nanza', DetailType: 'order.canceled.customer', Detail: JSON.stringify({...}), EventBusName:
  'default' }] }))`.
- **Currently emit events:** orders + refunds + seller/user lifecycle only.
  **Currently silent (need new emits):** `src/routers/message.ts`, `src/routers/conversation.ts`,
  `src/routers/connection.ts`, `src/routers/offer.ts`, `src/services/systemMessage.ts`.
- DB: **Postgres/RDS + Prisma** (`prisma/schema.prisma`, ~55 models). → connections table is just a
  new Prisma model; **no DynamoDB needed**.

**nanza-mobile**
- Bare RN 0.83, **TanStack Query v5**. No push libs, no sockets today.
- Data is already **invalidation-reactive** — the Messages screen invalidates
  `['conversations','connections','systemMessages','messageFeed']` on focus
  (`src/screens/messages/MessagesScreen/index.tsx`). So a socket handler just needs to fire the same
  invalidations on push → **near-zero UI/render work**.
- `useInboxCount` (`src/hooks/useInboxCount.ts`) is `staleTime: Infinity`, fetched **once per
  session** → today it never updates live. Must be invalidated (or cache-patched) on push.
- `App.tsx` already wires `AppState` into React Query's `focusManager` (L21-29) → the natural place
  to open/close the socket on foreground/background.
- Auth: Cognito JWT in AsyncStorage (`src/services/auth/cognito.ts`) → sent to WS `$connect`.

---

## Work breakdown

### A. nanza-api — WebSocket API + connection registry

1. **WebSocket API in `serverless.yml`** — new `websocketApi` events on a new function
   (`websocketHandler`) with routes `$connect`, `$disconnect`, `$default` (+ optional `ping`).
   Put the **Cognito authorizer on `$connect`** (reuse `authorizerHandler` logic; WS authorizers read
   the token from a query param, e.g. `?token=<jwt>`, since RN can't set WS headers reliably).
2. **Prisma model** `WsConnection { connectionId String @id, accountId String @index, createdAt }`
   + migration. (Harmless to keep small; rows are ephemeral.)
3. **`$connect`** → insert `{connectionId, accountId}` (accountId from authorizer context).
   **`$disconnect`** → delete by `connectionId`. **`ping`/`$default`** → no-op / pong (keepalive).
4. **IAM**: add `execute-api:ManageConnections` for the WS API ARN so the fanout Lambda can post.

### B. nanza-api — emit the missing events

Add `PutEvents` (mirror the order.ts pattern) at the create/update points that today are silent:
- new **message** → `message.created` (Detail: `{ recipientAccountId, conversationId, … }`)
- new **connection request** / accept → `connection.requested` / `connection.accepted`
- new **offer** → `offer.received`
- relevant `systemMessage` writes → matching DetailType
Each Detail **must carry the target `accountId`** so the fanout Lambda can look up connections.
Keep emits best-effort (try/catch, non-blocking) — same posture as existing order emits.

### C. nanza-api — fanout Lambda

New `fanoutHandler` Lambda, triggered by an **EventBridge rule** (`Source: nanza`, DetailTypes from
B). For each event: read target `accountId` from Detail → query `WsConnection` for that account →
`PostToConnection` the payload to each live connectionId via `ApiGatewayManagementApi`. On `410 Gone`,
delete that stale row (self-healing cleanup). Payload = `{ type, invalidate: ['conversations'|
'inboxCount'|…], ...minimalData }`.

### D. nanza-mobile — socket client

1. `src/services/realtime/socket.ts` — connect to `wss://…?token=<jwt>`, reconnect w/ backoff,
   heartbeat ping every ~4 min (under API GW's 10-min idle timeout), handle the 2-hr max-connection
   cap with a transparent reconnect.
2. **Lifecycle** — open on foreground / after login, close on background / logout. Hook into the
   existing `AppState` wiring in `App.tsx` (next to `focusManager`).
   **Cold-load deferral (2026-08-11):** the FIRST connect additionally waits for the one-shot
   `isAppReady` latch (the same signal the splash uses) + `runAfterInteractions`, with a 10s
   fallback timer so a missing signal can't strand the socket closed. Rationale: the socket only
   carries invalidation nudges — nothing above the fold needs it — but connecting mid-startup put
   the WS handshake in contention with the first screen's fetches on the device and added the
   API-side `$connect` (authorizer + cold account lookup) to the startup burst, which showed up as
   slow prod first loads. `AppReadyProvider` was hoisted in `App.tsx` to span `RealtimeBridge`.
   Same deferral pattern as HomeScreen's below-the-fold fetches (`deferredActive`).
3. **On message** — `queryClient.invalidateQueries` for the keys named in the payload
   (`['inboxCount', accountId]`, `['conversations']`, `['connections']`, `['systemMessages']`,
   `['messageFeed']`). Badge + list update themselves. (Optional later: direct `setQueryData` patch
   to skip the refetch entirely and drop even the invalidation fetch cost.)

---

## Connection lifecycle (answers "what closes it?")

**One socket per device**, not per conversation. It closes on: (1) app backgrounded (main saver —
wired to AppState), (2) network drop (auto-reconnect on return), (3) AWS limits — API GW force-closes
after **10 min idle** (keepalive ping prevents) and hard-caps at **2 hr** (transparent reconnect),
(4) logout (explicit). Every close fires `$disconnect` → row deleted, so we never post to a dead
connection; `410 Gone` handling in the fanout Lambda is the backstop.

## Cost (verified reasoning, us-east-1)

API GW WebSocket: **$1.00 / million messages** + **$0.25 / million connection-minutes**; fanout
Lambda invocations negligible.
- ~1k active users, ~2 h/day connected, ~1M pushed msgs/mo → **≈ $2–3 / month total**.
- 10× that → **low tens of dollars**. Both dimensions scale linearly, no cliff.
- Strictly cheaper than the removed minute-polling (which billed Lambda+Postgres every tick whether
  or not anything changed); WS bills only on real connection-time + real events.

## Risks / watch-items
- **WS auth in RN** — token via query param on `$connect` (headers unreliable on RN WS). Keep the JWT
  short-lived; re-auth on reconnect.
- **The old failure mode** was "EventBridge → sender Lambda not reliably invoked" (infra-push.md).
  De-risk early: unit-test the rule→fanout wiring with a probe event **before** building the client.
- **Multi-device** — a user may have >1 live connection; fanout iterates all rows (already handled).
- **Prod** — WS API + rule + IAM must deploy to prod stage too; redeploy is what creates/destroys the
  CloudFormation resources.

## Suggested build order
1. C-probe: WS API + connect/disconnect + a hardcoded test push (prove GW→device works).
2. Fanout Lambda + one real emit (`message.created`) end-to-end (prove EventBridge→device).
3. Mobile socket client + invalidation on that one event (prove badge/list update).
4. Fan out remaining emits (connection, offer, systemMessage).
5. Harden: reconnect/backoff, 410 cleanup, keepalive, prod deploy.

## Implementation status (2026-07-02)

Model renamed `WsConnection` → **`WebSocket`** at the user's request (before migration).

**nanza-api (done, code only — user runs the migration):**
- `prisma/schema.prisma` — `WebSocket` model + `Account.webSockets` relation. **Migration NOT run** (user's).
- `lambdas/authorizerHandler.ts` — reads JWT from `?token=` query param; denies guests on `$connect`.
- `lambdas/websocketHandler.ts` — `$connect`/`$disconnect`/`$default`(ping); resolves account via
  `resolveRequesterAccountId`; upsert/deleteMany on `webSocket`.
- `lambdas/fanoutHandler.ts` — EventBridge-triggered; looks up `webSocket` rows for `detail.accountId`,
  `PostToConnection` to each, prunes on 410.
- `src/utils/realtimeEvent.ts` — `emitRealtimeEvent()` helper (best-effort PutEvents; carries
  `accountId` + `invalidate`).
- `src/routers/message.ts` — emits `message.created` to other participants on send.
- `serverless.yml` — `websocket` + `fanout` functions, `websocketApi` routes with the Cognito
  authorizer (`identitySource: route.request.querystring.token`), EventBridge rule (message.created,
  connection.requested/accepted, offer.received), IAM `execute-api:ManageConnections`,
  `WS_API_ENDPOINT` env (`Fn::Sub` on `WebsocketsApi`).
- Added dep `@aws-sdk/client-apigatewaymanagementapi`. tsc + eslint + `serverless print` clean.

**nanza-mobile (done):**
- `env.d.ts` + `.env` — `WS_BASE_URL` (blank until WS API deployed; separate endpoint from API_BASE_URL).
- `src/services/realtime/socket.ts` — single WS: auth'd `?token=` URL, backoff+jitter reconnect,
  4-min heartbeat, `onmessage` → `queryClient.invalidateQueries([key])`. No-op if `WS_BASE_URL` unset.
- `src/components/global/RealtimeBridge/index.tsx` — headless; opens on foreground+auth, closes on
  background/logout (mirrors AnalyticsBridge). Mounted in `App.tsx`. Since 2026-08-11 the first
  connect is deferred behind `isAppReady` (see Lifecycle above).

**Still TODO (not yet done):**
- User: run migration; deploy WS API; set `WS_BASE_URL` in mobile `.env`; regenerate `@oakplatforms/types`.
- Remaining emitters: `connection.requested`/`connection.accepted` (connection router),
  `offer.received` (offer router), relevant `systemMessage` writes — each ~2 lines via `emitRealtimeEvent`.
- Prod: WS API + rule + IAM deploy to prod stage; set prod `WS_BASE_URL`.

## Debug log (2026-07-02) — "connections work but nothing pushes"

Symptom: WebSocket connections recorded in DB, but no live messages; fanout Lambda had **zero logs**.

Trace (source-side, not target-side):
1. Added emit diagnostics → API log showed `[realtime] emit result { failedEntryCount: 0, entries:[{EventId}] }`
   → the event **was** published and accepted by EventBridge. So the break was EventBridge → fanout.
2. Compiled CFN (`serverless package`) and compared ARNs:
   - Rule ARN: `…:rule/nanza-api-fanout-dev-rule-1` (no bus segment).
   - Fanout invoke-permission `SourceArn`: `…:rule/**default**/nanza-api-fanout-dev-rule-1`.
   - **Mismatch** (caused by `eventBus: default` in the event source) → EventBridge not authorized to
     invoke fanout → silent no-op, zero logs. This is the `infra-push.md` failure, root-caused.

**Fix:** removed `eventBus: default` from the `fanout` `eventBridge` event in `serverless.yml`. Re-packaged;
permission `SourceArn` now matches the rule ARN. Requires **redeploy**. (Also: `message.ts` now awaits the
emit before `res.json()`; fanout + realtimeEvent have diagnostic logs.)

Gotcha saved to memory: `eventbridge-default-bus-arn-gotcha`.

## Coverage expansion (2026-07-02, second session) — beyond messages

After messages worked end-to-end, wired the remaining live-update sources:
- **Open conversation thread** now updates live: added `'messages'` to the `message.created` invalidate
  list (thread uses queryKey `['messages', conversationId, accountId]`; client invalidates by root).
- **Connections** (`src/routers/connection.ts`): `connection.requested` → recipient on request create;
  `connection.accepted` → initiator on accept.
- **All system-message notifications** (offers, orders, groups) covered at one chokepoint —
  `createSystemMessage` (`src/services/systemMessage.ts`) now emits per recipient, category-mapped to
  `offer.received` / `order.updated` / `group.updated`, invalidating
  `['systemMessages','messageFeed','inboxCount']`. This automatically covers the offer router
  (`OFFER_RECEIVED`), order/refund flows, and group join requests without per-router wiring.
- `serverless.yml` rule pattern extended with `order.updated`, `group.updated`. Re-packaged; rule has
  no `EventBusName` and the permission ARN has no `/default/` segment (fix from first session holds).
- All tsc + eslint + `serverless package` clean.

**Badge ("footer indicator not updating") — investigated, no code bug.** `inbox-count` counts UNREAD
conversations (last message from the other party, newer than my `lastReadAt`) + unread system messages;
`useInboxCount` is an always-mounted active observer, and `invalidateQueries(['inboxCount'])` refetches
it regardless of `staleTime: Infinity`. Most likely the count reads 0 because viewing the messages
screen marks things read. Verify by receiving while on a different tab. (Emit already includes
`inboxCount`.)

**Caveat noted in code:** the `createSystemMessage` emit fires inside the caller's `tx` (pre-commit).
It's best-effort/awaited-with-internal-catch, so a rare rollback only causes a harmless client refetch.
If that ever matters, move the emit to after each caller's `$transaction` commits.

**Needs redeploy** to take effect. Still TODO: prod deploy + prod `WS_BASE_URL`; typing-indicator/presence
(separate larger feature — user requested for later).

## Durable-doc follow-up (when it lands)
Fold the lasting design into a vault solution design — likely a **new `solution-designs/realtime.md`**
(distinct from `infra-push.md`, which stays the record of the removed closed-app push) + an `INDEX.md`
link + an `architecture.md` note for the new WS Lambda + rule.
