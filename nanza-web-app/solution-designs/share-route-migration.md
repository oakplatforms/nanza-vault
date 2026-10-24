---
tags: [nanza-web-app, nanza-api, nanza-mobile, solution-design, sharing, plan]
status: PLAN — implemented in working trees, pending review/deploy (2026-08-15)
---

# Share Route Migration — type-scoped routes `/<typeSlug>/<code>` (Plan)

## Goal

Move the public share URL to **root-level, type-scoped routes**:
`nanza.app/<typeSlug>/<referenceCode>` (e.g. `/listing/32S392`, `/collection/437C52`).
The **slug in the path — not a letter inside the code — names the record type**, and the
backend looks straight in that type's table. The reference code becomes an opaque token
that can grow to 8–10 chars long-term without another migration.

**Decisions (Skylar, 2026-08-15, voice):**
- Root-level `/<slug>/<code>` — **no `/share` prefix** (supersedes the same-day
  `/share/<code>` migration, which was implemented and then reworked into this design
  before anything deployed).
- The `collection` slug maps to the **List** table.
- New API route `GET /reference/{type}/{code}`; the bare-code route is **removed** —
  old app builds break, accepted (same stance as before: clean cut, no redirects).
- No redirect from any old shape (bare root codes or `/share/<code>`).

## The taxonomy

One slug per shareable type — the cross-repo contract. Source of truth:
`nanza-api/src/constants/shareTaxonomy.ts`; mirrors live in the web app
(`src/helpers/shareTaxonomy.ts`), mobile (`src/utils/referenceCode.ts`), the ogEdge
lambda's inline regex, and mobile's `AndroidManifest.xml` — **all five move together**.

| Slug | Table / record | Old letter |
|------|----------------|-----------|
| `listing` | Listing | S |
| `bid` | Bid | B |
| `collection` | List | C |
| `product` | Product (→ Entity) | P |
| `bulk` | BulkListing | K |
| `group` | Group | G |
| `profile` | Profile (combined bids+listings) | U |
| `post` | Post (root only) | T |
| `tag` | SupportedTagValue (parents+children) | V (retired Y) |

Project sharing stays disabled — no slug until it ships. Codes still mint as 6-char
letter+digits, but **routing never decodes the letter anymore**; lookup-side validation
is a lenient `[A-Z0-9]{4,12}` so longer codes work later. Mobile still derives the slug
from the letter at two choke points (`buildShareUrl`, `referenceService.getByCode`) —
when letterless codes arrive, the slug must be threaded through instead of derived.

## Changes by repo (implemented)

### nanza-api

- **`src/constants/shareTaxonomy.ts` (new)** — slug↔record-type maps, slug guard,
  lenient lookup-code validation.
- **`src/services/referenceResolver.ts`** — `resolveByReferenceCode(prisma, recordType,
  code, includes)`: the caller supplies the record type; the letter switch became a
  record-type switch; strict format validation replaced by the lenient check.
- **`src/routers/universal.ts`** — `GET /reference/:shareType/:referenceCode` replaces
  the bare-code route (400 on unknown slug); OpenAPI rewritten. The legacy
  `GET /username/:username/:referenceCode` route is untouched.
- **`lambdas/metaHandler.ts` / `lambdas/ogHandler.ts`** — routes are now
  `/reference/{type}/{referenceCode}/meta` and `/og.png` (serverless.yml updated);
  slug from the path picks the include set; letter maps deleted.
- **`src/services/og/meta.ts`** — `og:url` = `${webBaseUrl}/<slug>/<CODE>`; og.png URL
  = `${apiBaseUrl}/reference/<slug>/<CODE>/og.png`.
- **`lambdas/ogEdgeHandler.ts`** — matches
  `/^\/(listing|bid|collection|product|bulk|group|profile|post|tag)\/([A-Z0-9]{4,12})\/?$/i`
  and fetches the typed meta route. Slug list is inlined (the edge bundle stays
  import-free).

### nanza-web-app

- **`src/helpers/shareTaxonomy.ts` (new)** — slug list + lenient code check.
- **`src/app/AppLayout.tsx`** — nine routes `/<slug>/:referenceCode` → `ShareDetail`
  (mapped from `SHARE_TYPE_SLUGS`); bare root codes fall through to the catch-all.
- **`src/app/ShareDetail/index.tsx`** — takes a `shareType` prop; the 6-char letter
  validation (`isReferenceCode`) is deleted; code shape checked leniently. Card
  branching still keys off the API envelope's record type.
- **`src/app/ShareDetail/data/fetchByCode.ts`** — includes keyed by slug, not letter;
  query key + fetch include the slug.
- **`src/services/api/Universal.ts`** — `getByCode(shareType, code, params)` →
  `/reference/<slug>/<code>`.
- **`public/.well-known/apple-app-site-association`** — one `"/<slug>/*"` component per
  type.
- Note: this **subsumed the uncommitted O/Y letter-roster edits** in
  `ShareDetail/index.tsx` + `fetchByCode.ts` (the letter lists no longer exist on web);
  `SmartRouter.tsx` (dormant) still carries its letter list untouched.

### nanza-mobile

- **`src/utils/referenceCode.ts`** — `ShareTypeSlug` type, letter→slug map,
  `shareSlugFromReferenceCode`, and `parseShareReferenceCode` now matches
  `<slug>/<code>` (slug at segment 0 or 1, lenient code shape). Near-miss config paths
  (`listing/:listingId`, `list/:listId`, `user/:profileId`) fall through: their ids are
  25-char cuids, rejected by the 4-12 code shape.
- **`src/utils/shareUrl.ts`** — `buildShareUrl`/`buildShareLabel` mint
  `/<slug>/<code>`; slug derived from the letter, optional explicit `shareType` param.
- **`src/services/api/Reference.ts`** — `getByCode` calls `/reference/<slug>/<code>`.
- **`android/app/src/main/AndroidManifest.xml`** — nine `pathPrefix` entries (one per
  slug) in the App Links intent filter.
- Navigator interception (`getStateFromPath` + warm-start listener) kept as-is — only
  the parser behind it changed.

## Manual infra step (highest risk — outside any repo)

`ogEdge` is **version-pinned and hand-associated in the CloudFront console** on the
web-app distributions (serverless.yml declares no association). After deploying:

1. Create behaviors (or one behavior per slug pattern — CloudFront has no alternation,
   so that's **nine path patterns**: `/listing/*`, `/bid/*`, `/collection/*`,
   `/product/*`, `/bulk/*`, `/group/*`, `/profile/*`, `/post/*`, `/tag/*`) with the new
   `ogEdge` version as the **origin-request** trigger, on **both** dev and prod
   distributions.
2. **Remove** any old root-level or `/share/*` behavior/association.
3. Invalidate `/.well-known/*`.

## Rollout order & accepted consequences

1. web app (routes + AASA) → 2. API (typed routes + edge) + manual CloudFront →
3. mobile release (force-update via the existing `useUpdateSheet`).

- Every already-shared root link **dies silently** (bounces to the marketing page); the
  server-persisted profile QR codes (`nanza.app/<username>` — username URLs, not codes)
  deserve their own review.
- Old mobile builds break both directions (intent filters stop matching; parser returns
  null; their API calls hit the removed bare-code route → 404).
- Apple's AASA CDN caches the old pattern for a while after deploy.
- No automated tests on web or mobile for these paths — verification is manual: cold +
  warm deep links per type on both platforms, OG unfurl per type, bare codes bouncing
  home.
- Known pre-existing drift (not caused by this change): web `ShareDetail`'s profile
  branch expects `ProfileListings`/`ProfileBids` envelope types with `data.items`, but
  the resolver returns `type: 'Profile'` with `{profile, bids, listings, summary}` —
  the web profile share card needs its own fix.

## Docs to update when this lands (fold into designs, then delete this plan)

Same set as before — web `sharing.md`/`architecture.md`/`INDEX.md`, api `og-sharing.md`
/`architecture.md`, mobile `sharing.md`/`share-card-restyle.md`/`architecture.md`
/`post-tag-sharing.md` — now saying `/<slug>/<code>` (updated 2026-08-15).

## Related

- [[sharing|Web Sharing design]] · [[../architecture|Web architecture]]
- nanza-api: `og-sharing.md` · nanza-mobile: `sharing.md`
