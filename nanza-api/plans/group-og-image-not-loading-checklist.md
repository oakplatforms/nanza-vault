# Group OG image not loading — ROOT CAUSE FOUND & FIXED (2026-06-21)

## TL;DR
The **image was never broken**. The **`/meta` endpoint was 500ing** for groups, so the link unfurl
never received an `og:image` tag — the image existed but nothing pointed to it.

## How it was diagnosed
Curled the two endpoints separately (they are different Lambdas):
- `GET /reference/G62556/og.png` → **302** → `…/share/G62556-….png` → that S3 object is a real
  **200, image/png, 1.2 MB**. ✅ Image generation works (the banner-resize fix was fine).
- `GET /reference/G62556/meta` → **500 `{"error":"Failed to build meta"}`**. ❌ This is the bug.

## Root cause
`lambdas/metaHandler.ts` has its **own** `RECORD_TYPE_BY_LETTER` map (separate from the one in
`lambdas/ogHandler.ts`). When group support was added, `ogHandler`'s `TYPE_BY_LETTER` got `G: 'Group'`
but **`metaHandler`'s map did not**. So a `G…` code fell through to `|| 'Listing'`, the resolver was
told to load **listing-only relations** (`entity.product`, `entity.entityTags.tag`, `condition`,
`account.profile`) **on a Group**, and Prisma threw on the invalid relations → 500.

## Fix (applied, uncommitted)
Added `G: 'Group'` to `RECORD_TYPE_BY_LETTER` in `lambdas/metaHandler.ts`. tsc + lint clean. Now the
group resolves with `META_INCLUDES_BY_TYPE['Group']` (= `[]`) and `buildOgMeta`'s `Group` case runs.

## ⚠️ Lasting lesson
There are **TWO letter→type maps** — `ogHandler.ts` (image) and `metaHandler.ts` (meta tags). Any new
reference type must be added to **BOTH**. To debug any share-image issue, curl **both**
`/reference/{code}/og.png` and `/reference/{code}/meta` — a working image with a broken preview means
the meta endpoint, not the image.

## UPDATE — there was a SECOND missing-G bug (the edge), found after the meta deploy
After deploying the meta fix, `/meta` returned 200 ✅ but the link preview STILL showed no image.
Fetching the share page as a crawler (`curl -A facebookexternalhit https://d17uau8qojkras.cloudfront.net/G62556`)
showed **zero og tags in the HTML** → the **Lambda@Edge injector wasn't running** for group paths.

**Cause #2:** `lambdas/ogEdgeHandler.ts` `REFERENCE_CODE_RE` was `/^\/([2-9SBCPK]{6})\/?$/i` — **missing
`G`**. So `/G…` paths didn't match → the edge passed them through with no tag injection. **Fixed:**
regex now `[2-9SBCPKG]`.

There were THREE hardcoded type-letter lists; a new reference type must update ALL three:
`ogHandler.ts` TYPE_BY_LETTER (image) · `metaHandler.ts` RECORD_TYPE_BY_LETTER (meta) ·
`ogEdgeHandler.ts` REFERENCE_CODE_RE (edge injection).

## To finish
1. ✅ API (meta) — deployed, `/meta` returns 200.
2. **Deploy + RE-ASSOCIATE the edge function.** Lambda@Edge is version-pinned: publish a new `ogEdge`
   version, then in the CloudFront console edit the distribution's behavior to point at that NEW version
   ARN. (This manual re-association is why edge versions weren't bumping.) Wait for propagation.
3. Re-fetch the page as a crawler and grep for `og:image` — should now be present. Then test the unfurl
   in a fresh context / platform debugger.
