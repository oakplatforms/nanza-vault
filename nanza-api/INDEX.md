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
- **`serverless.yml`** + **`buildspec.yml`** — deploy config; the `nanza-api-<stage>` CodePipeline
  (see [[../_shared/architecture/deploy-pipelines|Deploy pipelines]]) runs the buildspec on manual
  release (`prisma migrate deploy` → types publish → `serverless deploy`). `.github/workflows/`
  is the retired GitHub Actions version.

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
- [[solution-designs/saved-items|Saved items retired]] — saving listings/bids/lots retired 2026-08-29 (favorites later); `SavedItem` dropped 2026-08-30 in favour of a `PostLike` model + `POST /post/:id/like` (migration `20260830090000_post_likes` copies the likes over); `/saved-item*` stays as a compat shim over `PostLike` (410 for card targets).
- [[solution-designs/system-messages|System messages]] — who a system row is *from* (actor or group avatar; the Nanza logo is gone, 2026-08-30), "@x did what" titles, and the decline convention: the decliner's row is deleted, the other side gets a fresh one (refunds stay, rewritten). New types `GROUP_DECLINED` / `ORDER_REFUND_ACCEPTED` / `ORDER_REFUND_DECLINED` + `SystemMessage.group` (migration `20260830100000_system_message_group_refund`, to run).
- [[solution-designs/account-feeds|Account feeds]] — `/sell-feed` (owner's listings + lots) and `/trade-feed` (those + bids, viewer-filtered; built 2026-08-26 for the profile's Trades rail): one recency-merged, paginated list per account via a shared heads→merge→hydrate page builder.

## Plans (in progress)

Day-to-day implementation plans live alongside the solution designs in `solution-designs/` (in the
vault, so they're visible in Obsidian — not in the repo's gitignored `.plans/`, which is retcd ired).
When work lands, fold the plan's durable insight into its solution design and remove the plan.

- [[solution-designs/profile-display-name|Profile Display Name + username hardening]] — **plan, 2026-10-23, built api + mobile 2026-10-23, uncommitted, pending migration `20261023130000_profile_display_name` + types `0.1.124` publish + device QA**: optional free-form `Profile.displayName` (≤30 graphemes) shown in place of the username across mobile (`getProfileName` helper, settings field under Username), username stays the URL-safe identity; `PUT /user/:id` now runs the same shape + reserved-word check as the profile router (400) before the uniqueness 409. Same day, mobile-only: the rail's Short play badge (in [[solution-designs/affiliate-featured-shorts|Featured Shorts]]) and shorter mixed-width gallery tiles (in [[solution-designs/photo-galleries|Photo Galleries]]).
- [[solution-designs/guest-browsing|Guest Browsing & the Trades feed]] — **plan, 2026-10-23, built api + mobile 2026-10-23, uncommitted, pending dev deploy + device QA (Trade pills Nanza · My Bids · My Listings)**: guests get the normal bottom nav (the "Join the chat" footer goes, search returns to the circle); Trade shows guests a read-only market grid on a new brand-wide `GET /trades` (listings + lots + bids newest-first, reusing the `/trade-feed` builder; filters for price, kind, card type in phase 2); Groups read-only with Join → `AuthModal`; guest pill = Home · Groups · Trade · Search (no Inbox, no circle), every sign-in lands on Home. Uses `/trades`; guest FAB stays hidden.
- [[solution-designs/photo-galleries|Photo Galleries & Image Gates]] — **API deployed to dev 2026-10-18; mobile built 2026-10-19, uncommitted, pending device QA (no migration / types bump)**: a `GROUP` of up to five IMAGE children picked from the camera roll in one go (multi-select library picker, one gallery per post, the existing slot-width rail + one lightbox set), homogeneous groups only; and *images in posts* — picker, camera, gallery — become a public-author (AFFILIATE + VERIFIED) feature enforced in `validateContentItems`, with BASIC composers losing the two image circles. Posting itself stays open. Upload pairing flattens the item tree. ~2–3 days api + mobile. All questions answered 2026-10-18 (one gallery OR one photo, no camera into galleries, gallery tiles a bit smaller than a lone photo) — ready to build.
- [[solution-designs/reply-likes|Reply Likes]] — **mobile built 2026-10-19, uncommitted, pending device QA (no API change)**: the reply card's `'minimal'` footer becomes heart + count (no reply / share / report); API already hydrates `likeCount`/`viewerHasLiked` on thread reads and `POST /post/:id/like` already takes a reply id, so this is a half-day mobile change in `PostItem`. Heart only, count from one.
- [[solution-designs/affiliate-featured-shorts|Affiliate Rail — Featured Shorts & the Stories pager]] — **API deployed to dev 2026-10-18 (types `0.1.118`); mobile built 2026-10-18, uncommitted, pending device QA**: a new admin-set `Profile.isFeatured` decides who has a rail circle at all (unflagged affiliates stay in the program, just no circle); featured affiliates post a YouTube **Short** from their own circle as a *featured* post (`Post.isFeatured`, one EMBED item, `youtube.com/shorts/<id>` only, no body/tags, affiliates only, **one live Short per affiliate** — 409 while one is under 24 h); featured posts are excluded from every feed and surface only in the rail for 24 h (read-time window, never deleted). Rail rows gain `featuredPostId`/`viewerHasSeenFeatured`; ring = any unseen activity; tap opens a full-screen vertical **Stories pager** (`GET /profiles/affiliates/recent-posts`, first consumer of `withPostSeen`) mixing autoplaying 9:16 Shorts and flat post pages with a top progress bar (video duration / 5 s), hold-to-pause, swipe-up-past-the-end closes. Own circle → `MyShort` screen (live Short + X delete, or the input). All six questions answered on the call.
- [[solution-designs/agreements|Agreements — affiliate terms before the link]] — **built 2026-10-17 in api (committed) + admin + mobile (uncommitted); types 0.1.116 published + installed; pending migration + QA**: `Agreement` (typed document, `AFFILIATE`, `displayName` + plain-text `body`, authored in nanza-admin, text locked once anyone accepts → 409) + `AgreementAcceptance` (one per agreement × profile, `acceptedAt`) + a denormalized `Profile.hasSigned`; `POST /agreement/:id/accept` runs the two writes in one transaction; `GET /profile/:id/affiliate-link` and the public page's `affiliateLink` require `hasSigned`. Admin `/agreements` page; mobile Settings shows an *Affiliate agreement* shelf → `AffiliateAgreement` screen with **I agree** until signed. Migration `20261019120000_agreements`, types 0.1.115. Three open questions.
- [[solution-designs/airbridge-affiliate-attribution|Airbridge — Affiliate Install Attribution]] — **plan, design review (2026-10-15)**: one Airbridge tracking link per affiliate carrying the affiliate `Account.id` (`sub_id` + deep-link `ref`), deferred deep link via the RN SDK, a post-sign-up `PUT /user/affiliate` claim (covers email and Apple/Google; Cognito metadata rejected) that writes `Account.affiliateId` once, 7-day window, self-referral blocked. Mixpanel and web-SDK work out of scope. Uses `/trades`; guest FAB stays hidden.
- [[solution-designs/affiliate-vouchers|Affiliate Vouchers]] — **plan (2026-10-17)**: `Voucher` model (referred account, affiliate, type `AFFILIATE_POST|OFFER|LISTING|COLLECTION`, status `PENDING|ACCEPTED|DECLINED`), issued by hooks on a referred user's 5th post / 5th sent offer to 5 distinct recipients / 5th listing / 10th card added to a collection (lifetime counters for posts, listings + collection cards, live distinct-recipient count for offers), once per user per type (`@@unique`); admin `GET /vouchers` + `PUT /voucher/:id` and a nanza-admin Vouchers dashboard with accept/decline. `amount` on the row ($5, up to $20 per referred user), payout manual, no notifications. **API + admin built 2026-10-17, both uncommitted** (migration `20261017120000_affiliate_vouchers` to run; types 0.1.113 published); **fourth tier built 2026-10-20** (migration `20261020120000_affiliate_collection_voucher` to run; types 0.1.120 to publish); QA next.
- [[solution-designs/sets-and-tags-parity|Sets & Tags Parity]] — **2026-10-23: `isVisible` added beside `isPrimary` on Set + QueryList (alias kept, both columns written, migration `20261023120000_add_is_visible`, types `0.1.122`), separate hide/show `isVisible` on SupportedTagValue, `Listing.isPrimary` dropped; built api + mobile + admin, uncommitted.** API built 2026-10-17, uncommitted, pending QA (migrations `20261018120000_set_tag_value_parity` + `20261018130000_post_sets` to run; types `0.1.114`); mobile built 2026-10-17 (uncommitted, pending types install + QA); admin next**: `Set` stays its own model (replaces the retired *Sets as Tags* plan); both `Set` and `SupportedTagValue` get the same three images (`banner` / `thumbnail` / `avatar`, `Set.logo` renamed; circles read avatar → thumbnail), sets become post associations (`PostSet`, ten tags + sets combined) and a post home (`PostableType.SET`) with a mobile `SetScreen` + chat mirroring the tag value page, admin Sets grows a detail editor, and `GET /search` matches set names to return the set's cards. Three phases, ~4 days. Five open questions.
- [[solution-designs/affiliate-rail-latest-post-and-seen|Affiliate Rail — latest post, seen state, recency order]] — **built 2026-10-09 in api + mobile, uncommitted, pending QA (migration `20261009120000_post_seen` to run, types `0.1.107`)**: the home affiliate circles become a Stories-style rail — tap opens the affiliate's latest brand post, ordered newest-first, pink ring = unseen (no ring when signed out, since 2026-10-10), seen only clears the ring (the move-to-end went 2026-10-10). New `PostSeen` (mirrors `PostLike`), `POST /post/:id/seen` stamped on every post-detail open, `viewerHasSeen` on reads, and one light `GET /profiles/affiliates/latest-posts` (renamed from `/rail` 2026-10-10; profile + latestPostId + seen, server-ordered by recency). Three open questions.
- [[solution-designs/verified-profiles-and-affiliate-rail|Verified Profiles + Affiliate Rail Restore]] — **built 2026-10-08 in api + mobile, uncommitted, pending QA (migration `20261008120000_profile_type_verified` to run; types `0.1.106`; nanza-admin picker + ToS wording still open)**: a third `ProfileType`, `VERIFIED` (community member, not an affiliate), whose brand posts are public like an affiliate's — guests included — via one `PUBLIC_AUTHOR_PROFILE_TYPES` constant replacing six hard-coded `=== 'AFFILIATE'` reads; additive enum migration, types bump. Rail stays `?type=AFFILIATE`. Mobile half: restore the deleted `AffiliateCircleCarousel` (commit `de3023ff^`) to the top of Home + the first-paint gate, widen `useCanSharePost`. Uses `/trades`; guest FAB stays hidden.
- [[solution-designs/public-profile-page|Public Profile Page — `nanza.app/<username>`]] — **built 2026-10-07/08 in api + web, uncommitted, pending QA**: a read-only link-in-bio web page per public profile (hero + Posts / Collections / Trades, mirroring mobile). One new endpoint `GET /user/:username/page` that confirms the username with a single query *before* fanning out (404 otherwise), `validateUsername` hardened (charset, reserved words, one query); every tab reuses an existing guest-safe read. Web half: nested `/:username` routes after the share routes. No schema change.
- [[solution-designs/tracked-package-payout-hold|Tracked Packages — payout hold & tracking reminder]] — **built 2026-09-14, uncommitted, pending QA (migration `20260914120000_order_tracking_reminders` to run)**: capture-on-accept already existed; the wallet hold leaks between accept and tracking-number entry (shipment still `CREATED`) and over-holds by tax + fee — fix the predicate/amount; add `Order.acceptedAt` + two once-only reminder stamps, a daily `trackingReminder` cron that, clock starting at accept, emails the seller at 48 h (`order.tracking.reminder.seller`, new nanza-email template) and posts an `ORDER_TRACKING_REMINDER` system message at 72 h; `IN_REVIEW` consolidated to user-requested + one automatic "undelivered 10 days after accept" rule (accepted-no-tracking flag deleted). Un-accepted 84 h cancel unchanged. Spans API + email + small mobile copy.
- [[solution-designs/buyer-service-fee|Buyer Service Fee]] — **built 2026-09-11, uncommitted, pending QA (migration `20260911120000_order_service_fee` to run)**: move the 3% + $0.50 platform cut from the seller's payout to a visible buyer-paid **Service fee**; Stripe cannot exclude it from its own 2.9% + $0.30 (platform absorbs ~$0.10/order); one additive `Order.serviceFee` column (null = legacy seller-paid), `refund_application_fee` on refunds, fee on the tax-inclusive pre-fee total, waived on cash. Spans API + mobile + email + web copy.
- [[solution-designs/offer-engine|Offer Engine & Trade Removal]] — **API built 2026-08-27, uncommitted (migration `20260827120000_offer_engine` still to run; mobile half next)**: trading dropped entirely (3 tables + enum, 410 stubs), offers v1 torn out, and offers rebuilt as a negotiation layer over orders for both listings and bids — listing offers invoice on send, bid offers invoice on the bidder's accept and auto-accept the order for the seller; drafts modeled but no UI; cart retired later, counters later. Offer-only listings carry `Listing.isOffer` (2026-08-29, migration `20260829120000_listing_is_offer`) and surface nowhere but their offer.
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
  the homepage feed; guests see affiliates only (2026-10-23), signed-in viewers see affiliates + verified + self + accepted Connections. No
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
