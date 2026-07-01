---
tags: [nanza-api, solution-design, og-sharing]
---

# OG / Sharing — Solution Design

## Overview

Every public entity in nanza carries a short **reference code** (`referenceCodeGenerator.ts`) and
can be shared as a link. When that link is pasted into iMessage, Slack, Discord, or a social feed,
the crawler needs an unfurl: a title, description, and a preview image. nanza-api serves this with a
small **OG / share-preview subsystem** — three purpose-built Lambdas that resolve the code, render a
per-type share card, and splice Open Graph tags into the static web app's HTML so the link previews
correctly. Crawlers send no auth token, so these paths are all public and unauthenticated.

The share cards themselves are rendered server-side with **Satori → sharp** from React-like template
functions in `src/services/og/templates/*`, cached as PNGs in S3, and served via redirect.

## How it works

### The three Lambdas (and the resolver they share)

- **og** — `GET /reference/{referenceCode}/og.png` (`lambdas/ogHandler.ts`). Resolves the code via
  `src/services/referenceResolver.ts`, renders the matching card template with Satori → sharp,
  caches the PNG in S3 (`nanza-static-{stage}` under `/share/`), and 302-redirects to the object
  URL. Runs on the `sharp-arm64` Lambda layer (WebP → PNG decode).
- **meta** — `GET /reference/{referenceCode}/meta` (`lambdas/metaHandler.ts`). Returns the Open
  Graph payload — `title` / `description` / `image` / `url` — built by `src/services/og/meta.ts`
  (`buildOgMeta`). The image field points back at the `og.png` URL above.
- **ogEdge** — a **Lambda@Edge** origin-response function (`lambdas/ogEdgeHandler.ts`, pinned
  `x86_64` with its own edge-trust IAM role). On a share-page request it calls `/meta` and injects
  the resulting tags into the static web app's `<head>` so the link unfurls. It is **version-pinned**:
  publishing a new `ogEdge` version requires manually re-associating the CloudFront behavior with
  the new version ARN before it takes effect.

Reference codes are a 5-digit + type-letter scheme: `S`=Listing, `B`=Bid, `C`=List/collection,
`P`=Product, `K`=BulkListing, `G`=Group, `U`=Profile, `O`=Project. The resolver loads the right
Prisma relations per type (e.g. a Listing pulls `entity.product`, `entity.entityTags.tag`,
`condition`, `account.profile`; a Group pulls none).

### The three-map footgun

The single most important operational rule of this subsystem: **the letter→type mapping is
duplicated in three places, and a new share type must be added to ALL THREE:**

1. `ogHandler.ts` — `TYPE_BY_LETTER` (drives which card **image** renders).
2. `metaHandler.ts` — `RECORD_TYPE_BY_LETTER` (drives which **meta** relations load + which
   `buildOgMeta` case runs).
3. `ogEdgeHandler.ts` — `REFERENCE_CODE_RE` (the regex that decides whether the **edge** injects
   tags for a given path).

When Group sharing (`G`) was added, only `ogHandler` was updated. The result was a textbook
three-map failure, diagnosed by curling the endpoints separately (they are different Lambdas):

- `GET /reference/G62556/og.png` → 302 → a real 200 `image/png` in S3 (**image was always fine**).
- `GET /reference/G62556/meta` → **500 `{"error":"Failed to build meta"}`** — a `G…` code fell
  through to `|| 'Listing'`, so the resolver tried to load listing-only relations on a Group and
  Prisma threw. Fixed by adding `G: 'Group'` to `RECORD_TYPE_BY_LETTER` (Group resolves with an
  empty include set and `buildOgMeta`'s `Group` case).
- Even after `/meta` returned 200, the preview stayed blank because `ogEdge`'s
  `REFERENCE_CODE_RE` (`/^\/([2-9SBCPK]{6})\/?$/i`) was **missing `G`**, so `/G…` paths passed
  through with no tag injection. Fixed by widening the regex to `[2-9SBCPKG]`.

**Debugging rule:** a working `og.png` with a broken unfurl means the bug is in `meta` or the
`edge`, not the image. Always curl both `/og.png` and `/meta`, and fetch the share page as a
crawler (`curl -A facebookexternalhit …`) and grep for `og:image` to confirm the edge injected.

### Card templates (`src/services/og/templates/`)

The **listing** and **bid** cards are the canonical reference; product, lot (bulk), and collection
were brought to parity with them. Note: this renderer uses a local `COLORS` map with hardcoded hex
by existing convention — it is server-side Satori, **not** the RN app, so the mobile theme-token
rules do not apply here. Satori rasterizes inline base64 SVG data URIs fine; only WebP source
images need the sharp decode.

The settled visual spec:

- **Global wrapper:** one outer padding of **56** on all sides (no per-element / per-card padding).
- **Title** (entity name): Roobert **SemiBold 600**, white, single line, **pixel ellipsis**
  (`whiteSpace:nowrap` + `overflow:hidden` + `textOverflow:ellipsis`) — not a character clamp.
- **Subcode** (product # / subcode): Roobert **Regular 400**, **white**.
- **Hero image:** **400 × 518**, `objectFit:contain`, `borderRadius:18`.
- **Pills** (tags / condition): Roobert 600. **Price** pill: Roobert 600; the "$" stays Figtree
  (brand rule) at one weight above the number.
- **nanza wordmark** footer, bottom-left, on listing and bid cards. Inlined as a base64 SVG data
  URI (`logo_full.svg`) via a shared `footerLogo()` helper. The card body uses a bottom-anchored
  footer row so the logo sits consistently regardless of how much content is above it. A shared
  `statusPill()` helper and the inlined `NANZA_LOGO_SVG` live in `listing.ts` and are exported for
  `bid.ts`. (A WTS/WTB status pill was considered and **dropped** — no status pill ships.)

Per-type notes:

- **product.ts** — brought to parity: 56 padding, 400×518 hero, title 600/white pixel-ellipsis,
  subcode 400/white. Keeps a Fair-Market block on the right under the name. **No** nanza logo on
  product (intentional).
- **bulk.ts** (lot) — 56 padding, title 600/white pixel-ellipsis, shared `pricePill`. The card fan
  is a **centered front card** with the other two peeking only at the edges (one tilted slightly
  left, one slightly right — symmetric and subtle, not a wide splay), cards sized larger.
- **collection.ts** — full-bleed mosaic, plus a short deep **drop-shadow gradient** along the
  bottom of the grid (transparent → dark) to seat the nanza wordmark legibly bottom-left.

Verification is a local scratch harness (`.og-preview/preview.ts`, Satori → SVG → qlmanage PNG with
placeholder heroes) that renders the templates side by side to eyeball parity; it is gitignored /
removed before commit.

## Key decisions & rationale

- **Three separate Lambdas, one shared resolver.** Image, meta, and edge injection are genuinely
  different runtimes (sharp layer for image, edge-trust role for `ogEdge`), so they stay separate,
  but they resolve reference codes through the same `referenceResolver.ts` so the code→entity
  mapping stays consistent.
- **Listing/bid as the canonical card standard.** Rather than styling each template independently,
  every other card was measured against the listing/bid cards (56 padding, 600 title, 400×518 hero,
  pixel ellipsis) so the whole share surface reads as one system.
- **Hardcoded hex in the OG renderer is acceptable.** This is server-side rendering, not the RN app;
  a local `COLORS` map is the existing convention and the mobile CLAUDE.md theme-token rules do not
  apply.
- **The three-map duplication is a known, documented footgun.** It has bitten Group sharing twice.
  Until the maps are unified, every new reference type must update `ogHandler`, `metaHandler`, and
  `ogEdgeHandler` together, and be smoke-tested by curling both endpoints plus a crawler fetch.

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
