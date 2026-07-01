---
tags: [nanza-api, architecture]
---

# nanza-api — Architecture

The marketplace backend for the nanza TCG (trading-card) platform. Node + Express + Prisma,
packaged as AWS Lambdas behind API Gateway (HTTP API v2) via the Serverless Framework. One
"fat" Express app serves the whole REST surface; a handful of purpose-built Lambdas handle
auth, OG/share previews, webhooks, and scheduled jobs.

See also: [[INDEX|Project map]] · [[../_shared/INDEX|Shared brain]] · [[../_shared/capabilities/orchestrations|Orchestration patterns]]

---

## Request lifecycle

```
Client → API Gateway (HTTP API v2)
       → lambdaAuth (authorizerHandler)         # verifies JWT, returns IAM policy + context
       → api Lambda (apiHandler → Express app)  # /{proxy+} for GET/POST/PUT/DELETE/OPTIONS
       → router (all_routes.ts)                 # mounts ~50 feature routers
       → route handler                          # validation → service/prisma → DTO response
       → Prisma (pg adapter) → PostgreSQL (RDS)
```

1. **API Gateway** matches every path to the catch-all `/{proxy+}` route defined in
   [`serverless.yml`](../../nanza-api/serverless.yml) and invokes the `lambdaAuth` request
   authorizer first.
2. **Authorizer** — [`lambdas/authorizerHandler.ts`](../../nanza-api/lambdas/authorizerHandler.ts)
   extracts the bearer token, decodes it, and branches on the JWT `alg`:
   - `HS256` → a **guest** token (minted by the `guestToken` Lambda, signed with `tempJwtSecret`
     from Secrets Manager). Guests are read-only: any non-`GET` method is denied.
   - `RS256` → a **Cognito** token. The issuer decides which user pool (admin vs consumer), the
     JWKS public key verifies the signature, and `cognito:groups` maps to a role
     (`admin` / `seller` / `customer` / `registered`).
   It returns an IAM Allow/Deny policy plus a `context` of `{ role, userPool, principalId }`.
3. **api Lambda** — [`lambdas/apiHandler.ts`](../../nanza-api/lambdas/apiHandler.ts) wraps the
   Express app with `@vendia/serverless-express`. On cold start it initializes the Prisma,
   Stripe, and Shippo clients in parallel. It copies the authorizer `context` into
   `x-authorizer-*` request headers.
4. **Express app** — [`src/index.ts`](../../nanza-api/src/index.ts) reads those headers into
   `req.user`, mounts the aggregate router at `/`, and has a terminal error handler that maps
   thrown errors to `400` (with message) or `500`.
5. **Router layer** — [`src/routers/all_routes.ts`](../../nanza-api/src/routers/all_routes.ts)
   `router.use()`s every feature router. The **`universalRouter` is mounted last** so it can
   catch bare 6-char reference codes that nothing else matched.

---

## Layering & code organization

Everything runs under `src/`. Entry points live in `lambdas/`.

- **`src/routers/*`** — one Express `Router` per domain concept (`listing`, `bid`, `order`,
  `group`, `trade`, `cart`, `message`, `feed`, …). Each file exports a named router
  (`export const listingRouter`). Handlers read `req.user`, run validators, call into Prisma
  (often directly) or a service, and return DTO-shaped JSON. Routes carry inline `@openapi`
  JSDoc for spec generation. `all_routes.ts` composes them.
- **`src/services/*`** — cross-cutting business logic that spans multiple models or needs its
  own client: `order.ts` (collection sync on completion), `trade.ts` (quantity-aware
  collection moves), `resolver.ts` / `referenceResolver.ts` (derived pricing + share-code
  resolution), `og/` (share-image rendering), `anthropic.ts` (Claude client for the storefront
  builder), `transcribe.ts` (voice → text), `scan.ts` (card scanning), `payout.ts`,
  `invoice.ts`, `connection.ts`, `systemMessage.ts`.
- **`src/validation/*`** — pure validators / guards per domain (`seller`, `group`, `listing`,
  `usageCap`, `referenceCode`, …). `group.ts` holds the listing/bid **visibility filters**
  (public marketplace vs private-group access). `usageCap.ts` enforces freemium lifetime caps.
- **`src/utils/*`** — infrastructure helpers: `prismaHelpers.ts` (client init + error
  mapping), `secretsManager.ts` (cached Secrets Manager fetch), `stripe.ts` / `shippo.ts`
  (SDK clients), `s3Client.ts` + `uploadImage.ts` / `deleteImage.ts` / `generatePresignedUrl.ts`,
  `cognitoClient.ts` + `promoteUserToSeller.ts` / `promoteUserToCustomer.ts`,
  `referenceCodeGenerator.ts`, `paginatePrisma.ts`, `generateIncludes.ts`, `eventBridge.ts`.
- **`src/webhooks/*`** — the handler bodies for the Stripe and Shippo webhook Lambdas
  (dispatched from `lambdas/stripeWebhookHandler.ts` / `lambdas/shippoWebhookHandler.ts`).
- **`src/constants/*`** — `usageCaps.ts` (freemium limits), `taxRates.ts`, and
  `storefrontAssets/` (per-brand asset manifests for the AI builder).
- **`src/generated/json`** — Prisma JSON-schema output (`prisma-json-schema-generator`).
- **`packages/types`** — the `@oakplatforms/types` DTO package generated from the Prisma schema;
  the single source of truth for shared domain types across API, mobile, and web.
- **`prisma/`** — `schema.prisma` + the full `migrations/` history.
- **`migration-scripts/`** — one-off data-backfill scripts (the card-data import pipeline).

### Prisma usage

- Client is created lazily in [`src/utils/prismaHelpers.ts`](../../nanza-api/src/utils/prismaHelpers.ts)
  using the **`@prisma/adapter-pg`** driver adapter over a `pg` `Pool` (`max: 1`,
  `allowExitOnIdle` — tuned for Lambda). SSL uses the bundled RDS CA (`certs/global-bundle.pem`).
- Connection string comes from `DATABASE_URL` locally, else from Secrets Manager
  (`getDatabaseCredentials` → `buildDatabaseUrl`).
- The client applies field-level **`omit`** rules (e.g. hides `User.authId` / `isAdmin`) so
  sensitive columns never leave the DB by default.
- Handlers use `prismaClient()` for the singleton; `generatePrismaError` normalizes Prisma
  errors into user-facing messages.

---

## Data model (high level)

`prisma/schema.prisma` (~1600 lines, ~55 models). The spine:

**Identity & accounts**
- **User** (1:1 Cognito `authId`) → **Account** (1:1). `Account.type` is `REGISTERED` /
  `CUSTOMER` / `SELLER`. An Account fans out to role sub-models **Customer** (buyer, Stripe
  customer) and **Seller** (Stripe Connect account), plus a **Profile** (public username,
  avatar, `referenceCode`, `isPublic`) and **Cart**(s). **Admin** is a separate 1:1 off User
  and is the `createdBy`/`lastModifiedBy` on catalog models.
- Account carries backend-managed counters: `scanCount*` and lifetime usage counters
  (`listingsCreatedCount`, `bidsCreatedCount`, `messagesSentCount`, `collectionsCreatedCount`)
  enforced by [`validation/usageCap.ts`](../../nanza-api/src/validation/usageCap.ts).

**Catalog (admin-curated)**
- **Brand** → **Category**, **Set**, **Tag** / **BrandTag** / **SupportedTagValue** (the
  card-attribute taxonomy). **Entity** is a card (name, image, `EntityType`) with a 1:1
  **Product** (SKU, weight, dimensions), many **EntityTag** values, and a **Set**.
- **List** (`ListType` HOMEPAGE / COLLECTION / FAVORITE / CUSTOM) holds **EntityList**
  join rows (entity + quantity) — the model behind homepage rails **and** user collections.

**Marketplace (user-generated)**
- **Listing** (WTS — price, quantity, condition, entity) and **Bid** (WTB). Both have a unique
  `referenceCode`, `isPublic`, and optional group scoping via **GroupListing** / **GroupBid**.
- **BulkListing** ("lots") groups many child **Listing**s; joined to groups via
  **BulkListingGroup**. **ScanItem** is a staged pre-listing produced by card scanning.
- **Offer** (a bid/listing negotiation) → an **Order** on acceptance.
- **SavedItem** is a polymorphic bookmark over Listing / Bid / BulkListing.

**Orders, payments, fulfillment**
- **Order** (`ProcessStatus`) ← Cart, Customer, Seller; carries subtotal/tax/shipping/total,
  `paymentIntentId`, buyer/seller JSON snapshots. **OrderListing** join rows record purchased
  quantity/price and snapshot the `sourceGroup` + `sourceModeratorAccountId` at purchase time.
- **Invoice** groups orders. **Transaction** records payments/refunds/payouts
  (`TransactionType`, Stripe ids). **Payout** rolls transactions up to a seller. **Refund**
  (1:1 Order). **Review** (1:1 Order).
- Shipping: **ShippingMethod** / **ShippingOption**, seller-scoped via
  **SellerShippingMethod** / **SellerShippingOption**, **OrderShippingOption**, and **Shipment**
  (Shippo carrier + `TrackingStatus`).

**Social & community**
- **Group** (moderator Account + brand, `GroupPrivacy`) with **GroupMember**
  (`GroupMemberRole`, `GroupMembershipStatus`). **Connection** (initiator/recipient,
  `ConnectionStatus`, `intro`) models the follow/connect graph.
- **Conversation** ↔ **ConversationParticipant** ↔ **Message** (`MessageType` USER / SYSTEM).
  **SystemMessage** hangs off a Message (1:1) with `SystemMessageCategory` /
  `SystemMessageType` — the notification backbone.
- **Comment** — self-referential threads, polymorphic over `CommentableType`.
- **Trade** (`TradeStatus`) with **TradeOfferEntity** / **TradeRequestEntity** — card-for-card
  swaps that move entities between collections.

**AI builder**
- **Project** (`ProjectType`, default STOREFRONT) holds a JSON `tree` page + free-text
  `context`; **ProjectEdit** logs each AI edit (prompt, before/after tree, token counts).
  Currently **disabled** — the `projectRouter` is intentionally not mounted in `all_routes.ts`.

Reference codes: a 5-digit + type-letter scheme (`referenceCodeGenerator.ts`) — `S`=Listing,
`B`=Bid, `C`=List/collection, `P`=Product, `K`=BulkListing, `G`=Group, `U`=Profile,
`O`=Project. These power all public share links.

---

## Auth flow (Cognito)

- Two Cognito user pools: **consumer** and **admin** (`CONSUMER_USER_POOL_ID` /
  `ADMIN_USER_POOL_ID`). Roles come from Cognito **groups** (`admin` / `seller` / `customer` /
  `registered`), resolved in the authorizer.
- **Guest access**: `GET /user/guest-token` (its own unauthenticated `guestToken` Lambda) mints
  a short-lived HS256 JWT (`role: guest`). Guests can only make `GET` requests — enforced in the
  authorizer.
- The API can mutate Cognito group membership (`AdminAddUserToGroup` /
  `AdminRemoveUserFromGroup` / `AdminDeleteUser`) via `utils/cognitoClient.ts` when promoting a
  user to seller/customer (e.g. after Stripe onboarding completes) — IAM-scoped in
  `serverless.yml`.

---

## Background jobs & crons

Two EventBridge-scheduled Lambdas (defined in `serverless.yml`):

- **cancelOrder** — [`lambdas/cancelOrderHandler.ts`](../../nanza-api/lambdas/cancelOrderHandler.ts):
  daily `cron(0 17 * * ? *)`. Cancels still-`PENDING`, unshipped orders older than ~72–84h,
  refunding via Stripe.
- **reviewOrder** — [`lambdas/reviewOrderHandler.ts`](../../nanza-api/lambdas/reviewOrderHandler.ts):
  daily `cron(0 18 * * ? *)`. Flags stalled orders (long-pending with `UNKNOWN`/`PRE_TRANSIT`
  tracking, or not completed/delivered after 14 days) for manual review.

**EventBridge (default bus)** is also used for fire-and-forget domain events — the api and
webhook Lambdas hold `events:PutEvents` permission and dispatch via `utils/eventBridge.ts`
(e.g. after a Stripe seller-account update completes onboarding).

> Push notifications (FCM) were prototyped and **removed** — there is no `services/fcm.ts` and
> no push Lambda. See the "Infra & push" plans in [[INDEX]].

---

## External integrations

- **AWS Cognito** — identity (two pools), JWT verification via JWKS.
- **AWS Secrets Manager** — `nanza-credentials-{stage}` holds DB creds + all app secrets
  (Stripe, Shippo, Google Places, Anthropic, guest-JWT secret). Cached in-process by
  `utils/secretsManager.ts`.
- **Stripe** — payments, Connect seller onboarding, refunds. Webhook Lambda
  (`lambdas/stripeWebhookHandler.ts` → `src/webhooks/stripe.ts`) verifies signatures with
  `stripeWebhookSecret` and reacts to `account.updated` (seller verification), payment, and
  refund events.
- **Shippo** — shipping rates, label purchase, tracking. Webhook Lambda
  (`lambdas/shippoWebhookHandler.ts` → `src/webhooks/shippo.ts`) drives `Shipment.trackingStatus`.
- **Amazon S3** (`nanza-static-{stage}`) — user/catalog images and the rendered OG share PNGs.
  Presigned uploads via `utils/generatePresignedUrl.ts`.
- **Amazon Rekognition** (`DetectText`) + **Amazon Transcribe** — power card scanning
  (`services/scan.ts`) and voice input (`services/transcribe.ts`).
- **Anthropic (Claude)** — `services/anthropic.ts` drives the (currently disabled) storefront
  builder, generating a `Project.tree` from prompts.
- **Google Places** — address validation (`utils/addressValidation.ts`).

### OG / share-preview subsystem

Three dedicated Lambdas serve public, unauthenticated share links (crawlers send no token):

- **og** — `GET /reference/{referenceCode}/og.png`
  ([`lambdas/ogHandler.ts`](../../nanza-api/lambdas/ogHandler.ts)). Resolves the code via
  `services/referenceResolver.ts`, renders a per-type card with **Satori → sharp**
  (`src/services/og/templates/*`), caches the PNG in S3, and 302-redirects to the object URL.
  Uses the `sharp-arm64` Lambda layer (WebP → PNG decode).
- **meta** — `GET /reference/{referenceCode}/meta`
  ([`lambdas/metaHandler.ts`](../../nanza-api/lambdas/metaHandler.ts)). Returns the Open Graph
  payload (title/description/image/url).
- **ogEdge** — a **Lambda@Edge** origin-response function
  ([`lambdas/ogEdgeHandler.ts`](../../nanza-api/lambdas/ogEdgeHandler.ts), pinned `x86_64` with
  its own edge-trust IAM role) that calls `/meta` and splices the tags into the static web app's
  `<head>` so links unfurl.

> The og and meta handlers each keep their **own** letter→type map — a known footgun; adding a
> new share type means updating both (see the group-OG plan in [[INDEX]]).

---

## Deployment

- **Serverless Framework v3** ([`serverless.yml`](../../nanza-api/serverless.yml)), `nodejs20.x`,
  **arm64** (except `ogEdge`), region `us-east-1`, bundled with `serverless-esbuild`. Stages:
  `dev` and `prod`, each with its own S3 bucket, base URLs, memory/storage, and provisioned
  concurrency (prod api = 3).
- **GitHub Actions** deploy on branch push:
  [`.github/workflows/deploy-dev.yml`](../../nanza-api/.github/workflows/deploy-dev.yml) (branch
  `dev`) and `deploy-prod.yml` (branch `prod`). Each job authenticates to AWS via **OIDC**
  (no static keys), pulls DB creds from Secrets Manager, runs **`prisma migrate deploy`** +
  `prisma generate`, then `serverless deploy --stage <stage>`. Pool ids and
  `LATEST_APP_VERSION` are injected from GitHub `vars`.
- Local dev: `npm run dev` (`ts-node src/localhost.ts`) with a `DATABASE_URL` env; a
  `docker-compose.yml` Postgres backs the Jest test suite.
