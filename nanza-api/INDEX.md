---
tags: [nanza-api]
---

# nanza-api

The marketplace backend for the nanza TCG (trading-card) platform. A Node/Express + Prisma app
packaged as **AWS Lambdas** (Serverless Framework) behind API Gateway, with **Cognito** auth,
**Stripe** payments/Connect, **Shippo** shipping, **S3** storage, and a Satori/sharp **OG
share-image** subsystem. One fat Express app serves the REST surface; dedicated Lambdas handle
auth, share previews, webhooks, and scheduled jobs.

→ Full write-up: [[architecture|Architecture]]

## Structural overview

Everything runs under `src/`; Lambda entry points live in `lambdas/`.

- **`lambdas/`** — Lambda handlers. `apiHandler` (Express app), `authorizerHandler` (JWT →
  IAM policy), `guestTokenHandler`, `ogHandler` / `metaHandler` / `ogEdgeHandler` (share
  previews), `stripeWebhookHandler` / `shippoWebhookHandler`, `cancelOrderHandler` /
  `reviewOrderHandler` (crons).
- **`src/index.ts`** — Express app: reads authorizer headers into `req.user`, mounts the
  aggregate router, terminal error handler.
- **`src/routers/*`** — ~50 feature routers, one per domain (`listing`, `bid`, `order`,
  `group`, `trade`, `cart`, `message`, `feed`, …). `all_routes.ts` composes them;
  `universalRouter` is mounted **last** to catch bare reference codes. Handlers carry inline
  `@openapi` JSDoc.
- **`src/services/*`** — cross-model business logic: `order`/`trade` (collection sync),
  `resolver`/`referenceResolver` (derived pricing + share-code resolution), `og/` (share-image
  templates), `anthropic` (Claude storefront builder), `scan`, `transcribe`, `payout`,
  `invoice`.
- **`src/validation/*`** — per-domain guards; `group.ts` holds listing/bid visibility filters,
  `usageCap.ts` enforces freemium lifetime caps, `referenceCode.ts` decodes share codes.
- **`src/utils/*`** — infra helpers: `prismaHelpers` (pg driver adapter + client init),
  `secretsManager`, `stripe`/`shippo`/`s3Client`/`cognitoClient`, `referenceCodeGenerator`,
  `eventBridge`, image upload/delete/presign.
- **`src/webhooks/*`** — Stripe + Shippo webhook handler bodies.
- **`src/constants/*`** — `usageCaps`, `taxRates`, per-brand `storefrontAssets`.
- **`prisma/`** — `schema.prisma` (~55 models) + full `migrations/` history.
- **`packages/types`** — generated `@oakplatforms/types` DTO package (shared across API,
  mobile, web).
- **`serverless.yml`** + **`.github/workflows/`** — deploy config; GitHub Actions deploy on
  push to `dev`/`prod` via OIDC (`prisma migrate deploy` → `serverless deploy`).

**Dominant patterns:** router-per-domain composed in `all_routes.ts`; handler → validation →
service/Prisma → DTO; lazy singletons (Prisma/Stripe/Shippo) initialized on Lambda cold start;
secrets and DB creds pulled from Secrets Manager and cached in-process; public share links
resolved by a single reference-code resolver shared by the JSON, OG, and meta paths.

## Solution designs

Living overviews of how each backend area works and why. One per theme in `solution-designs/`.

- [[solution-designs/og-sharing|OG / sharing]] — the og/meta/ogEdge Lambdas, reference-code resolution, the three letter→type maps, and share-card style parity.
- [[solution-designs/payments|Payments]] — the 3% + $0.50 + tax application fee, the removed 1% moderator rev-share, and the delivery-mode × payment-type model (in-person cash/card).
- [[solution-designs/data-sync|Data sync]] — best-effort collection sync on listing creation and order completion, reusing the trade move primitive.
- [[solution-designs/infra-push|Infra & push]] — push notifications were REMOVED; the APNs / Firebase-WIF setup history kept for a rebuild.
- [[solution-designs/projects|Projects / AI builder]] — the Storefront → Project refactor end state, web builder re-enable, and caps/credits (not subscriptions).
- [[solution-designs/sources-and-insights|Sources & Insights]] — the orchestration judgment rules and legacy price algorithm; the models/runtime moved to [[../oak-cortex/solution-designs/insight-engine|oak-cortex]] (2026-07-05 pivot) — nanza-api keeps only a read-only spec endpoint and receives reviewed price migrations.
- [[solution-designs/oak-api-platform-rescope|oak-api Platform Re-scope]] — nanza-api is really Oak's API: Oak is the platform, tenants are `Client`s (nanza = default row), `clientId` on client roots, non-breaking phasing, and the risky serverless stack rename. Design only.

## Related

- [[architecture|Architecture]]
- [[../_shared/INDEX|Shared brain]]
- [[../_shared/capabilities/orchestrations|Orchestration patterns]]
