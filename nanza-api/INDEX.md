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
- [[solution-designs/image-cdn|Image CDN / resizing]] — on-demand `?width=` WebP resizing via CloudFront + a single-origin resizer Lambda over `nanza-static`; no stored thumbnails; client `getImage(id,{width})`/`px()` sizing by tile; the four Function-URL/OAC/ListBucket deploy gotchas.
- [[solution-designs/payments|Payments]] — the 3% + $0.50 + tax application fee, the removed 1% moderator rev-share, and the delivery-mode × payment-type model (in-person cash/card).
- [[solution-designs/data-sync|Data sync]] — best-effort collection sync on listing creation and order completion, reusing the trade move primitive.
- [[solution-designs/query-lists|Query Lists]] — admin-configured *dynamic* collections: multi-select source types (listing/bid/bulk/entity/set) + `QueryCriterion` rows joined by one `AND`/`OR` combinator, resolved live (no stored membership). Design only; first consumer is the homepage.
- [[solution-designs/infra-push|Infra & push]] — push notifications were REMOVED; the APNs / Firebase-WIF setup history kept for a rebuild.
- [[solution-designs/db-connections|DB connections & the socket registry]] — why Lambda×pg connections don't scale (the 71-connection dev incident); pivoted to **no VPC** — the DynamoDB socket registry (`oak-api-ws-*`, built 2026-08-04, pending deploy) takes `$connect`/`$disconnect`/fanout off Postgres, REST is capped via reservedConcurrency; RDS Proxy + VPC kept as appendix/escalation path.
- [[solution-designs/projects|Projects / AI builder]] — the Storefront → Project refactor end state, web builder re-enable, and caps/credits (not subscriptions).
- [[solution-designs/sources-and-insights|Sources & Insights]] — the orchestration judgment rules and legacy price algorithm; the models/runtime moved to [[../oak-cortex/solution-designs/insight-engine|oak-cortex]] (2026-07-05 pivot) — nanza-api keeps only a read-only spec endpoint and receives reviewed price migrations.
- [[solution-designs/oak-api-platform-rescope|oak-api Platform Re-scope]] — nanza-api is really Oak's API: Oak is the platform, tenants are `Client`s (nanza = default row), `clientId` on client roots, non-breaking phasing, and the risky serverless stack rename. Design only.
- [[solution-designs/profile-types|Profile Types]] — `ProfileType` enum (`BASIC`/`AFFILIATE`) on `Profile`, admin-assigned, the gate for later affiliate features. Design only.
- [[solution-designs/posts|Posts]] — comments became **posts** (backend built 2026-07-31): `ContentItem` blocks (`RICH_TEXT`/`IMAGE`/`LINK`, `VIDEO` reserved but rejected), a `BRAND` home target for the brand feed (AFFILIATE-only creation), tagging permissive-within-brand, one-level replies. Responses carry `postCount` plus a deprecated `commentCount` alias. Migration not yet run.
- [[solution-designs/tag-taxonomy-v2|Tag Taxonomy v2]] — `SubTagValue` became **`SecondaryTagValue`** hanging off `BrandTag` with a **many-to-many** link to `SupportedTagValue`, so one hero can belong to several classes; find-or-create-and-link chip endpoint; deletes block (never sweep) when they would orphan one. Backend built 2026-07-31 alongside Posts.
- [[solution-designs/comments|Comments]] — **historical**: what the `Comment` system was before Posts replaced it. Kept because prod mobile still speaks its API through legacy shims.
- [[solution-designs/card-pricing|Card Pricing]] — operator-side price-recommendation orchestration (built 2026-08-20): CSV or zip in (≤50 cards per CSV), parallel `card-pricer` agents web-search current market, one `price_usd` out per card (fair NM midpoint × 0.90); the doc's methodology section is the runtime source of truth the agents read. Every run leaves a postmortem in [[pricing-runs/INDEX|Pricing Runs]]. Distinct from Sources & Insights (in-product engine).
- [[solution-designs/social-eligibility|Social Eligibility]] — all social features (messaging/connections, group create + join, posting) require only a **registered** account (2026-08-17); customer/seller tiers gate commerce only. Messaging eligibility helpers deleted; mobile's seller/customer gate modals removed from social flows; the friend-request payment gate is gone.
- [[solution-designs/account-feeds|Account feeds]] — `/sell-feed` (owner's listings + lots) and `/trade-feed` (those + bids, viewer-filtered; built 2026-08-26 for the profile's Trades rail): one recency-merged, paginated list per account via a shared heads→merge→hydrate page builder.

## Plans (in progress)

Day-to-day implementation plans live alongside the solution designs in `solution-designs/` (in the
vault, so they're visible in Obsidian — not in the repo's gitignored `.plans/`, which is retcd ired).
When work lands, fold the plan's durable insight into its solution design and remove the plan.

- [[solution-designs/blocking-and-reporting|Blocking Visibility & Content Reporting]] — **built,
  uncommitted (2026-08-23, Apple release blockers; migration still to run)**: blocking must hide posts/replies everywhere (home,
  groups, profile tabs, tag pages) via a symmetric `notBlockedFilter` — no schema change; plus a
  new `Report` model + reason enums, a mobile flag → "What are you reporting?" screen, and a
  reported-post queue in nanza-admin (a new admin read mode on `GET /posts`). Spans API +
  mobile + admin. See also [[solution-designs/apple-review-response|the Apple review response]].
- [[solution-designs/affiliate-social-links|Affiliates: Banners & Social Links]] — **plan, API
  half implemented (2026-08-15)**: admin-curated `SocialLinkType` catalog + one-URL-per-type
  `SocialLink` rows on affiliate profiles and groups (replace-set PUTs, moderator/affiliate
  gated); profile banner already existed. Admin + mobile halves next.
- [[solution-designs/reference-item-chat-surfacing|Reference-Item Chat Surfacing]] — **plan
  (2026-08-14, build tonight)**: posts that EMBED a listing/bid/entity/lot surface in that item's
  chat, via the tag-union pattern (`referencedPostsFilter` mirroring `taggedTagValuePostsFilter`)
  + a ContentItem `referenceId` index. API-only; mobile unchanged.
- [[solution-designs/post-editing-and-drafts|Post Editing & the Single Draft]] — **plan (design
  review)**: close PUT's gallery/image gaps for a composer edit mode; one persistent DRAFT-status
  post per account (additive Status value, reads filtered). Planned 2026-08-14; mobile half in
  nanza-mobile. NOT implemented.
- [[solution-designs/open-posting-and-friend-visibility|Open Home Posting & Friend-Scoped Visibility]] —
  **plan**: the BRAND affiliate create-gate becomes a read-side visibility rule — everyone posts to
  the homepage feed; affiliates reach everyone, basic users reach self + accepted Connections. No
  schema change. Planned 2026-08-14; mobile half (nav swap) in nanza-mobile.
- [[solution-designs/post-carousel-and-tagging|Posts — Multi-Content Carousel & Open Tagging]] — **plan**:
  5 content items per root post (`isPrimary` + swap endpoint), replies gain one attachment / lose
  tag rows, 10 supported tags flat (parents+children) on every home incl. GROUP (tag-page union gains the group exclusion
  filter). Planned 2026-08-11; mobile half in nanza-mobile.
- [[solution-designs/tag-taxonomy-v3-is-child|Tag Taxonomy v3 — isChild]] — collapse `SecondaryTagValue` into `SupportedTagValue` via `isChild` + a `SupportedTagValueParent` self m:n; wipe dev secondary data; supersedes v2's two-model design. Planned 2026-08-04.
- [[solution-designs/realtime-in-app-updates|Realtime in-app updates]]
- [[solution-designs/idempotent-post-user|Idempotent POST /user]]

## Related

- [[architecture|Architecture]]
- [[../_shared/INDEX|Shared brain]]
- [[../_shared/capabilities/orchestrations|Orchestration patterns]]
