---
tags: [nanza-api, nanza-mobile, solution-design, image-cdn]
---

# Image CDN / On-Demand Resizing — Solution Design

## Overview

Uploaded images (entities, listings, group banners, avatars, …) live in the **`nanza-static-{stage}`**
S3 bucket, one resized-at-upload copy per asset. Clients used to load these **full-size straight from
S3** — a grid of small thumbnails downloaded the same bytes as a full-screen hero. The image CDN fixes
that: a **CloudFront distribution in front of an on-demand resizer Lambda** returns a right-sized WebP
for any `?width=` on the fly, edge-cached per size, with **no stored thumbnails and no external
services** (no Cloudinary/imgix). The client (`getImage`) sends **only resized requests** to the
resizer distribution; **width-less requests go straight to the plain S3 CDN** (`cdn.nanza.app`) so
full-size assets and logos never pay the resizer Lambda's cold-start (see Client side).

The heavy lifting is on the *thumbnails*: a 190pt tile now pulls a ~600px image instead of a 1050px
original. Heroes barely shrink (they want near-full-size) but still gain edge caching and one URL path.

## How it works

### The resizer Lambda + its CloudFront distribution

- **imageResize** — `lambdas/imageResizeHandler.ts`, exposed as a **Lambda Function URL** and used as
  the **single origin** of a dedicated CloudFront distribution (`ImageCdnDistribution` in
  `serverless.yml`). It parses the S3 key from the request path and `width`/`height`/`quality` from the
  query, `GetObject`s the original from `nanza-static-{stage}`, resizes with **sharp** to WebP, and
  returns the bytes (base64, `isBase64Encoded`). CloudFront caches each `(path, width)` variant, so a
  given URL only ever runs the Lambda once. Runs arm64 on the shared **`sharp-arm64` layer** — same
  layer as `og`/`api` (see [[og-sharing]]).
- **Single-origin is deliberate.** CloudFront routes by *path pattern*, not query string, so a
  "S3-for-plain, Lambda-for-resizes" split would need Lambda@Edge. Instead every image request goes
  through the resizer; when there's no `?width=` (or the asset is an SVG) it passes the original
  through unchanged. The extra work is one cache-miss per URL; after that it's pure edge cache.
- **Guardrails.** Widths snap to an allowlist (`100…1600`) so the cache can't fragment across arbitrary
  sizes or be driven to generate unbounded variants. **SVGs are never rasterized** — logos stay vector.
  `withoutEnlargement: true` means asking for a width above the stored original just returns the
  original (heroes never upscale).

### Client side (`nanza-mobile`)

- **`src/utils/getImage.ts`** is the one URL builder, and it **splits traffic across the two
  distributions by whether a resize is requested** — not everything through the resizer.
  - `getImage(id, { width })` → **`IMAGE_CDN_BASE_URL`** (the resizer distribution). Appends
    `?width=`; the sharp Lambda runs once per `(path, width)` on a cold miss, then the edge caches
    every variant. The cold-start only ever costs the *first* viewer of each size.
  - `getImage(id)` with **no width** → **`S3_BASE_URL`** (the plain CloudFront-over-S3 CDN,
    `cdn.nanza.app`). No Lambda in the path, so full-size/un-resized assets — listing & bid detail
    heroes, avatars, SVG logos — skip the resizer's cold-start entirely. Routing everything through
    the resizer made these *slower* than plain S3 for zero benefit (a no-width request only streams
    the original through); the split fixes that.
  - If `IMAGE_CDN_BASE_URL` is unset, resized calls fall back to `S3_BASE_URL` full-size (params
    dropped) so the app still works without the resizer.
- **Sizing is by fixed bucket, not device pixels.** Every resized call passes one of the
  `IMG_WIDTH` constants (`THUMB 400`, `CARD 600`, `HERO 800`, `BANNER 1000`) so the URL — and thus
  the edge cache key — is identical across devices; a device-derived `px()` width fragments the cache
  per screen size / pixel ratio so the edge almost never has a warm copy. The small-thumbnail surfaces
  (EntityCard rows, trade/cart/order rows, search chips, collection grid cards / thumbs / mosaic cells,
  the profile-list strip, group circles/banners) all pass `IMG_WIDTH.THUMB`. Heroes and wide banners
  pass `HERO`/`BANNER`.
- **Do not add a width where `getImage()` feeds an equality check.** `EditListing` and
  `ScanItemEditScreen` compare `selectedImage !== getImage(stored)` to detect an unchanged image;
  appending `?width=` there silently breaks change detection. Those call sites stay width-less.

### Upload sizes (source of truth)

Originals are still resized once at upload with sharp (`src/utils/uploadImage.ts`): **entities 750px**,
**listings 1050px**, group thumbnails 600px — kept generous so the CDN has a crisp source to downscale
from. The CDN never upscales, so the stored width is the ceiling for any `?width=`.

## Deploy gotchas (learned the hard way)

Standing up the distribution surfaced four traps, all now encoded in `serverless.yml`:

1. **`Outputs:` vs `Resources:` nesting.** Adding an `Outputs:` block mid-`resources` silently swallowed
   the existing `OgEdgeExecutionRole` into Outputs, breaking `ogEdge`'s `Fn::GetAtt`. Keep `Outputs:` as
   its own top-level block *after* all `Resources:`.
2. **Don't forward `Authorization` to a Function URL.** The managed `AllViewerExceptHostHeader` origin
   policy forwards the app's bearer token; a Function URL reads it as a bad SigV4 signature → `403
   AccessDeniedException`. Use a **custom origin request policy** forwarding only `width`/`height`/`quality`.
3. **Oct 2025 CloudFront→Function-URL auth change.** Public `AuthType: NONE` + `Principal:*` no longer
   works. The Function URL must be **`AuthType: AWS_IAM`**, reached via a CloudFront **Origin Access
   Control** (type `lambda`, SigV4), plus **two** permissions for `cloudfront.amazonaws.com` scoped to
   the distribution: `lambda:InvokeFunctionUrl` **and** `lambda:InvokeFunction`.
4. **`s3:ListBucket` masks 404s.** Without it, a `GetObject` on a missing key returns a misleading
   `AccessDenied` naming `s3:ListBucket` instead of `NoSuchKey`. The shared role now grants
   `s3:ListBucket` on the bucket ARN (separate statement — it acts on the bucket, not `/*`).

## Rollout

- Per stage: `sls deploy --stage <stage>` stands up its own resizer + distribution over its own bucket.
  The distribution domain is a stack output (`ImageCdnDomain`); set it as mobile `IMAGE_CDN_BASE_URL`
  (prefixed `https://`). First deploy takes ~15–20 min to propagate.
- **`cdn.nanza.app`** is a *separate, older* CloudFront-over-S3 distribution (plain passthrough, no
  resize). Consolidating the new resizer onto that custom domain is a follow-up (needs an ACM cert +
  alias + DNS); until then the resizer uses its `*.cloudfront.net` domain.

## Related

- [[og-sharing]] — the other sharp-on-Lambda subsystem; shares the `sharp-arm64` layer and the
  `nanza-static` bucket.
- [[infra-push]] — Serverless/CloudFront deployment patterns.
