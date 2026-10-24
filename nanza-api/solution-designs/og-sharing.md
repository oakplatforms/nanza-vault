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
functions in `src/services/og/templates/*`, cached as WebP in S3, and served via redirect.
(Revised 2026-08-04: the cached objects were PNG — 1.5MB+ per photographic card, the largest
files in the bucket — and are now `share/<CODE>-<updatedAtMs>.webp` at quality 80, matching the
bucket-wide WebP convention. The route path stays `/og.png`; crawlers follow the 302 regardless.
Old `.png` share objects are orphaned, not overwritten — purge `share/*.png` manually if bucket
size matters.)

## How it works

### The three Lambdas (and the resolver they share)

- **og** — `GET /reference/{type}/{referenceCode}/og.png` (`lambdas/ogHandler.ts`). Resolves the
  code via `src/services/referenceResolver.ts` (the path's type slug picks the table), renders the
  matching card template with Satori → sharp, caches the WebP in S3 (`nanza-static-{stage}` under
  `/share/`), and 302-redirects to the object URL. Runs on the `sharp-arm64` Lambda layer (WebP
  decode for heroes, WebP encode for output).
- **meta** — `GET /reference/{type}/{referenceCode}/meta` (`lambdas/metaHandler.ts`). Returns the
  Open Graph payload — `title` / `description` / `image` / `url` — built by
  `src/services/og/meta.ts` (`buildOgMeta`). The image field points back at the `og.png` URL
  above; the `url` field is the typed share path `https://nanza.app/<slug>/<code>`.
- **ogEdge** — a **Lambda@Edge** origin-request function (`lambdas/ogEdgeHandler.ts`, pinned
  `x86_64` with its own edge-trust IAM role). On a share-page request — the root-level type-scoped
  paths since the share-route-migration final shape (2026-08); `REFERENCE_CODE_RE` is now
  `/^\/(listing|bid|collection|product|bulk|group|profile|post|tag)\/([A-Z0-9]{4,12})\/?$/i` — it
  calls the typed `/meta` route and returns a **complete response** with the tags injected into
  the static web app's `<head>`, short-circuiting the origin. It is **version-pinned**: publishing
  a new `ogEdge` version requires manually re-associating the CloudFront behaviors with the new
  version ARN before it takes effect — and because CloudFront path patterns can't express
  alternation, the association is **nine path-pattern behaviors** (one per slug: `/listing/*`,
  `/bid/*`, … `/tag/*`), manually created on both the dev and prod distributions; the old
  root/`/share/*` behaviors must be removed.

The URL slug picks the table: `listing`=Listing, `bid`=Bid, `collection`=List, `product`=Product
(resolving to Entity), `bulk`=BulkListing, `group`=Group, `profile`=Profile (combined
bids+listings), `post`=Post, `tag`=SupportedTagValue. Project sharing stays disabled — no slug.
The reference code itself is an **opaque token** (lenient 4–12 alphanumeric on lookup; still
minted as 6 chars, five digits + a type letter, but routing never decodes the letter). The
resolver loads the right Prisma relations per type (e.g. a Listing pulls `entity.product`,
`entity.entityTags.tag`, `condition`, `account.profile`; a Group pulls none).

### The letter-map footgun (historical) → the slug-taxonomy sync

The single most important operational rule of this subsystem used to be that **the letter→type
mapping was duplicated in three places** (`ogHandler.ts`'s `TYPE_BY_LETTER`, `metaHandler.ts`'s
`RECORD_TYPE_BY_LETTER`, `ogEdgeHandler.ts`'s `REFERENCE_CODE_RE`) and a new share type had to
land in all three. Since the share-route-migration final shape (2026-08) routing never decodes
the letter — the path's type slug drives all three Lambdas — and the sync concern moved to the
**slug taxonomy, duplicated in five places that move together:**

1. `nanza-api/src/constants/shareTaxonomy.ts` — the **source of truth**.
2. web `src/helpers/shareTaxonomy.ts`.
3. mobile `src/utils/referenceCode.ts`.
4. `ogEdgeHandler.ts`'s slug regex — plus the **nine per-slug CloudFront behaviors**.
5. mobile `AndroidManifest.xml`'s nine `pathPrefix` entries.

(Letter lists survive only for code minting and mobile's slug derivation at the share choke
point.) The Group incident below is the canonical story of why this class of duplication bites.

When Group sharing (`G`) was added, only `ogHandler` was updated. The result was a textbook
three-map failure, diagnosed by curling the endpoints separately (they are different Lambdas):

- `GET /reference/G62556/og.png` → 302 → a real 200 `image/png` in S3 (**image was always fine**).
- `GET /reference/G62556/meta` → **500 `{"error":"Failed to build meta"}`** — a `G…` code fell
  through to `|| 'Listing'`, so the resolver tried to load listing-only relations on a Group and
  Prisma threw. Fixed by adding `G: 'Group'` to `RECORD_TYPE_BY_LETTER` (Group resolves with an
  empty include set and `buildOgMeta`'s `Group` case).
- Even after `/meta` returned 200, the preview stayed blank because `ogEdge`'s
  `REFERENCE_CODE_RE` (then `/^\/([2-9SBCPK]{6})\/?$/i`; today the edge matches the nine type
  slugs, not letters) was **missing `G`**, so `G…` share paths passed
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
- **The slug taxonomy is the new documented sync concern** (replacing the retired letter-map
  footgun, which bit Group sharing twice). Every new share type must land in all five taxonomy
  mirrors — API `src/constants/shareTaxonomy.ts` (source of truth), web `shareTaxonomy.ts`,
  mobile `referenceCode.ts`, the edge slug regex (+ a new CloudFront behavior per slug), and
  AndroidManifest — and be smoke-tested by curling both endpoints plus a crawler fetch.

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
- [[../nanza-mobile/solution-designs/post-tag-sharing|Post & Tag Sharing]] — proposed: three new
  letters (T=Post, V=SupportedTagValue, Y=SecondaryTagValue) through this pipeline, incl. two new
  templates (post, tag)
