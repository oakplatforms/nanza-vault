---
tags: [nanza-api, solution-design, infra, database, websocket, dynamodb]
---

# DB Connections & the Socket Registry

> **Status: Phases 0–1 IMPLEMENTED (2026-08-04, uncommitted, pending deploy).** Pivoted with
> Skylar away from the original RDS Proxy + VPC direction: **no VPC move**. The socket path
> stops touching Postgres entirely (DynamoDB registry); the REST path is capped, not
> proxied. The original RDS Proxy design is preserved below as a considered alternative.
> Phase 1 implementation record at the end of the Phases section.

## The incident, and why connections grow

Dev hit `FATAL: remaining connection slots are reserved for roles with privileges of the
"rds_reserved" role` with **71 open connections** — one developer, no leak. The arithmetic:

- Every warm Lambda container holds its own pg `Pool` (`max: 3`, `idleTimeoutMillis: 30s`,
  in `src/utils/prismaHelpers.ts`). Connections = warm containers × 3.
- **Four Lambda families** touch the DB in one app session: `api` (every REST call),
  `websocket` (`$connect` resolves the account + upserts / `$disconnect` deletes a
  `WebSocket` row — `$default` is a DB-free keepalive), `fanout` (per domain event:
  `findMany` + stale-prune on `WebSocket`), and the crons. Each family scales containers
  independently.
- **Nothing caps concurrency** — no `reservedConcurrency` anywhere; dev has
  `provisionedConcurrency: 0`. A burst of parallel requests = that many containers.
- `/link-preview` amplifies: each call holds its container through an external fetch with
  manual redirects and a 5s timeout, so those containers stack instead of recycling.

Postgres `max_connections` on RDS ≈ `LEAST(memoryBytes / 9531392, 5000)` — roughly **~110
on a t3.small, ~200 on a t3.medium** (verify actual class in the console). So **~60–70
concurrent requests exhausts a small instance**. Sockets are the multiplier that makes it
worse than REST alone: every app-on-screen holds a WebSocket, and `$connect`/`$disconnect`
churn plus per-event fanout scale with *sessions*, not request volume.

## The decision: no VPC — remove the socket path from Postgres instead

**RDS Proxy has no public endpoint.** Adopting it forces every DB-touching Lambda into the
DB's VPC, which drags in a NAT Gateway (~$33/mo + data) because Stripe, Shippo, Cognito,
EventBridge, `execute-api`, Rekognition, Anthropic, and `/link-preview`'s
arbitrary-internet fetches all leave the VPC. That is a system-wide infra change to fix
what is mostly a socket-path problem. Rejected.

Instead, the socket lifecycle stops using Postgres at all:

```
$connect / $disconnect ──▶ DynamoDB `oak-api-ws-*` (connectionId → accountId)
fanout (per event) ───────▶ Query GSI by accountId ──▶ PostToConnection
Postgres ─────────────────▶ only the REST/cron paths, capped by reservedConcurrency
```

The `WebSocket` table is ephemeral connection state — exactly what DynamoDB on-demand is
for. No proxy, no new always-on box, no VPC, ~pennies/month. (A public PgBouncer on a
t4g.nano was considered as the literal "lightweight proxy" — viable since RDS is already
publicly accessible, but it's an extra box to patch and it only *thins* connections;
DynamoDB *eliminates* them on this path.)

### DynamoDB table shape

- Table `oak-api-ws-${stage}`, on-demand billing.
- PK: `connectionId` (S). GSI `byAccount`: PK `accountId` (S) — mirrors the current
  `@@index([accountId])` query in fanout.
- TTL attribute `expiresAt` set at `$connect` to now + ~3h. API Gateway WebSockets max out
  at 2h connection duration, so TTL is a pure backstop: it replaces both the Postgres
  `onDelete: Cascade` from `Account` (deleted account's rows just expire) and any rows
  orphaned by missed `$disconnect`s that fanout's 410-prune never touches.

### Code changes (all in nanza-api)

1. `lambdas/websocketHandler.ts` — `prisma.webSocket.upsert` → `PutItem` (PutItem is
   already idempotent-overwrite, matching the upsert's intent); `deleteMany` → `DeleteItem`
   (naturally a no-op on missing keys, matching the deleteMany choice).
2. `lambdas/fanoutHandler.ts` — `findMany({ where: { accountId } })` → `Query` on the
   `byAccount` GSI; stale-prune `deleteMany({ in: stale })` → `BatchWriteItem` deletes.
3. `serverless.yml` — table resource + GSI + TTL, `dynamodb:PutItem/DeleteItem/Query/
   BatchWriteItem` on the table+index for the two functions, table name env var.
4. `prisma/schema.prisma` — **no change in this release.** `model WebSocket` and
   `Account.webSockets` ship as-is (unused once the handlers cut over); they're removed in
   a follow-up release only once we're officially on the new system (Phase 3).

### The `$connect` account-lookup wrinkle

`$connect` also calls `resolveRequesterAccountId` (`src/validation/group.ts`) — one
Postgres `User.findFirst` by Cognito sub. With the registry in DynamoDB this is the socket
path's *only* remaining Postgres touch. Handling, in order:

- **Now:** keep it, behind a warm per-container in-memory cache keyed by `principalId` —
  the `authId → accountId` mapping is immutable, so the cache never needs invalidation.
  Combined with `reservedConcurrency` on `websocket`, worst case is a handful of
  connections, only on cold `$connect`s.
- **Later (fully DB-free):** stamp `accountId` into the token via a Cognito
  pre-token-generation trigger (nanza-auth), read it from the authorizer context. Cross-repo
  change; not required for the win.

## Compatibility — is this a breaking change?

**No. Clients never see the registry.** The client contract is: open
`wss://…execute-api…/${stage}` with the JWT (query-string, via the shared authorizer),
send `{"action":"ping"}` keepalives, receive `{ type, invalidate, … }` nudges. All of that
is API Gateway + handler behavior; where connectionIds are *stored* is invisible.

- **nanza-mobile** (`src/services/realtime/socket.ts`, `RealtimeBridge`) — unchanged. The
  fanout payload is built from the EventBridge detail, not from storage.
- **nanza-web-app** — has no websocket usage at all today (guest-only mode; sockets are
  Cognito-authorized). Nothing to touch.
- **Server-side blast radius** — the Prisma `WebSocket` model has exactly four call sites,
  all inside `websocketHandler.ts` and `fanoutHandler.ts`. Nothing else reads or joins it.
  The `Account.webSockets` relation exists only for cascade delete, which TTL replaces.
- **Rollout window** — sockets already open when the new code deploys have rows only in
  Postgres, so they miss fanout nudges until the client reconnects (≤2h by API Gateway's
  max duration, usually sooner via app background/foreground). The nudge is best-effort by
  design — the app refetches on resume and email covers app-closed — so this degrades
  softly and self-heals. No dual-write/dual-read needed. Rollback = revert the two
  handlers; Postgres table is still there until the Phase-3 cleanup migration.

## The REST path

Unchanged problem, smaller multiplier once sockets are out. Mitigation is caps, not a
proxy:

- `reservedConcurrency` on `websocket` + `fanout` (~5 in dev), and a sane cap on `api` in
  dev.
- `idleTimeoutMillis` 30s → 10s in `src/utils/prismaHelpers.ts` so idle warm containers
  release connections faster.
- `/link-preview` remains the container-stacker to watch; its cap matters most.

If REST concurrency genuinely outgrows the instance later, the RDS Proxy + VPC design
below is the escalation path — adopt it then, not preemptively.

## Phases

- **Phase 0 — IMPLEMENTED 2026-08-03 (pending deploy):** `reservedConcurrency: 20` on
  `api`, dev only (`null` in prod until the prod `max_connections` is confirmed; then size
  a generous backstop ≈ max_connections/3 minus cron headroom) + `idleTimeoutMillis` → 10s.
  **2026-08-11 addendum:** prod cap set to `reservedConcurrency: 30` (≤90 conns — safe even
  if prod is a t3.small; instance class still unconfirmed, so this is the conservative
  backstop, not the sized number). Compatible with prod's `provisionedConcurrency: 3`
  (reserved must stay ≥ provisioned). Resize per the formula once `max_connections` is
  read from the console.
  (both stages, it's code). Caps on `websocket`/`fanout` were **skipped** — the registry
  (Phase 1) is next up and makes them redundant; until it ships, socket-path exhaustion is
  still possible and the interim tool is killing idle sessions via `pg_stat_activity`.
  Trigger: a second dev exhaustion on 2026-08-03 (`GET /brands`, i.e. the REST side).
- **Phase 1 — registry: IMPLEMENTED 2026-08-04 (uncommitted, pending deploy).** Dev first
  (this is where the incident was), then prod. No infra outside nanza-api's own stack.
  What landed, and the deltas from the design:
  - **`src/services/socketRegistry.ts`** (new) — the four ops behind one module:
    `registerConnection` (PutItem with `expiresAt` = now + 3h), `unregisterConnection`
    (DeleteItem), `getConnectionIdsByAccount` (Query on `byAccount`, pages through
    `LastEvaluatedKey`), `pruneConnections` (BatchWriteItem, 25-chunked; unprocessed keys
    are logged and left to TTL — pruning is best-effort). Raw `@aws-sdk/client-dynamodb`
    (new dep), no lib-dynamodb marshalling — three attributes didn't warrant it.
  - **`websocketHandler.ts`** — Prisma `upsert`/`deleteMany` → registry calls. The
    `$connect` account lookup kept as designed, behind the warm per-container
    `Map<principalId, accountId>` cache (immutable mapping, no invalidation); Prisma init
    is now lazy inside that lookup, so `$disconnect` and warm `$connect`s never touch
    Postgres at all.
  - **`fanoutHandler.ts`** — `findMany` → GSI query, stale-prune → batch delete;
    **all Prisma imports removed**, and its `package.patterns` (Prisma client + certs)
    dropped from serverless.yml — the function holds zero Postgres connections.
  - **`serverless.yml`** — `WsRegistryTable` resource exactly per the table-shape section
    (on-demand, `byAccount` GSI **KEYS_ONLY**, TTL on `expiresAt`); one added statement on
    the shared provider role (`PutItem/DeleteItem/Query/BatchWriteItem` on table + index
    ARNs); `WS_REGISTRY_TABLE` env var on both functions; table name in
    `custom.wsRegistryTable` (`oak-api-ws-${stage}`).
  - **Schema** — untouched per Phase 3: `model WebSocket` ships unused as the rollback
    lever.
  - Verified: `tsc --noEmit` + eslint clean; `serverless print --stage dev` resolves
    (with the Cognito `${env:}` pool IDs stubbed — they only exist at deploy time).
- **Phase 2 — verify:** `pg_stat_activity` during a socket connect/disconnect + fanout
  burst — websocket/fanout containers should hold **zero** Postgres connections (except
  cold `$connect` lookups); DynamoDB metrics show the ops; stale rows expire via TTL.
- **Phase 3 — cleanup (separate release, decided 2026-08-03):** the Prisma model and
  Postgres table are kept through the cutover release as the rollback lever; drop
  `model WebSocket` + the relation and delete the table only once we're officially
  running on the new system.
- **Phase 4 (optional, later):** pre-token-generation `accountId` claim → delete
  `resolveRequesterAccountId` from `$connect` entirely.

## Costs (us-east-1, approx)

| Item | Monthly |
|---|---|
| DynamoDB socket table (on-demand + GSI + TTL) | ~pennies at current scale |
| ~~RDS Proxy (2-vCPU floor)~~ | ~~$22~~ — not adopted |
| ~~NAT Gateway~~ | ~~$33 + $0.045/GB~~ — not adopted |

## Open items

1. Confirm the RDS instance class / `max_connections` in the console (still unverified —
   creds weren't available during design).
2. `authorizer` is believed DB-free (JWT verification only) — verify; it stays untouched
   either way now that there's no VPC move.
3. ~~Pick the prod `reservedConcurrency` backstop for `api`~~ — conservative 30 set
   2026-08-11 (see Phase 0 addendum); still resize once item 1 confirms `max_connections`
   (dev is 20; socket-family caps intentionally skipped in favor of the registry).

---

## Appendix — the original RDS Proxy + VPC design (not adopted)

Kept for the record and as the escalation path if REST-side concurrency ever demands it.

The proxy multiplexes many short-lived Lambda connections onto a small pool of long-lived
backend connections; Postgres sees a stable few dozen regardless of burst, and failover is
~66% faster. Per-query latency gains only ~1ms — the real win is warm-pool connects and no
brownouts under load.

Everything is shaped by the VPC prerequisite: Lambdas are not in a VPC today and RDS Proxy
has no public endpoint, so adoption means moving every DB-touching Lambda into the DB's
VPC — NAT Gateway (~$33/mo + $0.045/GB) for Stripe/Shippo/Cognito/EventBridge/
`execute-api`/Rekognition/Anthropic/`/link-preview`, a free S3 gateway endpoint to skip
NAT for images, and ~1s first-invoke ENI setup per subnet/SG combo.

Config notes worth keeping: proxy auth via the same `nanza-credentials-*` secret (own IAM
role with `secretsmanager:GetSecretValue`); **TLS gotcha** — the proxy presents
ACM/Amazon-Trust certs, not the RDS CA, so `certs/global-bundle.pem` +
`rejectUnauthorized: true` would fail the handshake; use the system trust store when the
host is the proxy. `IdleClientTimeout` ~5min, `MaxConnectionsPercent` ~90. Prisma via
`@prisma/adapter-pg` uses unnamed prepared statements (no pinning), but avoid
`SET`/advisory locks/`LISTEN-NOTIFY` and watch
`DatabaseConnectionsCurrentlySessionPinned`. Code-side: `proxyHost` preferred by
`buildDatabaseUrl` with direct host as the rollback lever; keep `max: 3` per container.
