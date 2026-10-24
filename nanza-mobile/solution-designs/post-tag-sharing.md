---
tags: [nanza-mobile, nanza-api, nanza-web-app, solution-design, sharing, posts, tags]
---

# Post & Tag Sharing — Solution Design

> **Status: approved, 2026-08-03 — ready to build** (all open questions resolved with Skylar). Pre-implementation design for the last big feature of the
> chat release train: share reference codes + OG meta + share landings for **individual posts**,
> **supported tag values**, and **secondary tag values**. Spans nanza-api (codes, resolver, OG
> lambdas), nanza-web-app (landing cards), and nanza-mobile (share entries + share cards).
> Builds directly on [[sharing|Sharing]] (the reference-code system) and
> [[../nanza-api/solution-designs/og-sharing|OG / Sharing]] (the three-Lambda OG pipeline) —
> read both first. Depends on the chat release landing first (posts must exist in prod).
>
> **Landed ahead of this design (2026-08-03/04) — the api-side OG work below is already built
> and revised in place; don't re-plan it, build against it:** the T/V/Y templates and meta cases
> exist in `src/services/og/` (post: avatar+username then text, no kicker; tag: banner+wordmark
> only, no on-image name; both titles dropped the `| nanza` suffix), and the OG renderer now
> caches **WebP** (`share/<CODE>-<updatedAtMs>.webp`, q80) instead of PNG — the share-file size
> fix. Remaining work is the mobile share entries + cards and the web landing cards.

## Scope (decided with Skylar, 2026-08-03)

Three new shareable types, plugged into the existing universal reference-code plumbing:

1. **Post** — **root posts only**. Replies are NOT shareable this release (maybe later).
2. **SupportedTagValue** — e.g. the *Warrior* class page.
3. **SecondaryTagValue** — e.g. the *Boltyn* hero page.

Everything reuses the uniform system: a letter per type (for minting),
`GET /reference/{type}/{code}`, the `ShareScreen → ShareDetailView → Share*Card` switch in
mobile, the nine root-level typed routes `/<slug>/:referenceCode` → `ShareDetail` in web
(posts land at `/post/<code>`, tag values at `/tag/<code>` — share-route-migration final
shape, 2026-08), and the og/meta/ogEdge Lambda trio. "Adding a new shareable type is a matter
of registering a letter and wiring one more case at each layer" — this design is those cases, ×3.

Out of scope: sharing replies, share counts/analytics, X/Instagram embeds, any change to the
existing seven share types.

## The letters (proposal — confirm before building)

Taken: `S B C P K G U O`. Proposed additions:

| Letter | Type | Mnemonic |
|---|---|---|
| `T` | Post | the posT / Thread |
| `V` | SupportedTagValue | the supported **V**alue |
| `Y` | SecondaryTagValue | secondar**Y** |

**Capacity note:** a code is 5 digits from `2-9` with the letter at any of 6 positions —
**8⁵ × 6 ≈ 196,608 possible codes per letter**. Fine forever for tag values (admin-curated,
hundreds of rows) and fine for posts *only because minting is lazy* (below): only posts someone
actually shares consume a code. If shared-post volume ever approaches that ceiling, the fix is a
longer code for new types, not a redesign — flagging so nobody mints on post-create later
"for consistency" and burns the space.

## Schema (additive migration — Skylar runs it)

```prisma
// all three models
referenceCode String? @unique
@@index([referenceCode])
```

On `Post`, `SupportedTagValue`, `SecondaryTagValue`. Nullable — same posture as `Profile` was
originally: null means "never shared / not yet minted". No enum changes, no data migration.

## Minting strategy (split by type — decided rationale, confirm)

- **Tag values: mint at admin create + one-time backfill.** Both tag routers
  (`supportedTagValue.ts`, `secondaryTagValue.ts`) wrap create in
  `generateReferenceCodeWithRetry` (as listing/bid/group do). Existing rows get a backfill
  (same treatment as groups: "existing groups get codes backfilled by hand" — a small script or
  SQL pass). Rationale: rows are few, admin-created, and a tag page share should never need a
  write from an arbitrary user's session.
- **Posts: lazy mint on first share.** New endpoint
  `POST /post/:id/reference-code` — idempotent (returns the existing code or mints one),
  authenticated, **rejects replies** (`parentId != null` → 400) and non-ACTIVE posts. Any
  authenticated user can mint (the sharer is rarely the author). Rationale: post volume ×
  the 196k-per-letter ceiling; only shared posts pay.

Mobile's share flow for posts therefore becomes: tap share → `POST /post/:id/reference-code` →
`guardedShare(buildShareUrl(code))`. Tag pages already have their code on the DTO and share
directly, like every existing type.

## API resolution (`referenceResolver.ts`)

Three new cases in `resolveByReferenceCode`, following the existing shape:

- **`T` → `Post`.** `prisma.post.findUnique({ where: { referenceCode } })` with the lean post
  include (author `account.profile` id/username/avatar, `contentItems` ordered). Then run the
  post through the existing enrichment the `/posts` list uses: **reference expansion**
  (`postReferences.ts` — the embedded listing/bid/entity/link tile needs its `reference`
  object), slim `tags`, `likeCount` / `replyCount` scalars (`postLikes.ts` / `postCount`-style).
  `viewerHasLiked` is always false here — the route is public/unauthenticated (crawlers).
  Resolve `status != ACTIVE` and replies (`parentId != null`) to **null** → "link not found".
  Replies are NOT embedded (payload budget); the mobile share card fetches them through the
  normal public `GET /posts?parentId=` path if it renders them.
- **`V` → `SupportedTagValue`**, **`Y` → `SecondaryTagValue`.** `findUnique` by code; the card
  needs `name/displayName/banner/thumbnail/description` (all on the model) — include-driven
  like the rest. The rails (secondary values / linked heroes) come from the existing public
  tag endpoints client-side, keeping the resolver payload lean.

Also: `VALID_TYPES` grows in **both** API letter lists — `utils/referenceCodeGenerator.ts` and
`validation/referenceCode.ts` (yes, the list is duplicated there too — see the footgun below).

## OG pipeline (nanza-api lambdas) — the three-map footgun, now a seven-file footgun

Per [[../nanza-api/solution-designs/og-sharing|OG / Sharing]], every new letter must land in
**all** of these, and Group sharing has already been bitten twice by missing one:

| # | File | Change |
|---|---|---|
| 1 | `lambdas/ogHandler.ts` | `TYPE_BY_LETTER` + `OG_INCLUDES_BY_TYPE` + render branches |
| 2 | `lambdas/metaHandler.ts` | `RECORD_TYPE_BY_LETTER` + `META_INCLUDES_BY_TYPE` + `buildOgMeta` cases |
| 3 | `lambdas/ogEdgeHandler.ts` | `REFERENCE_CODE_RE` → `[2-9SBCPKGUTVY]` (today the edge matches the nine type slugs `/(listing\|bid\|…\|tag)/<code>` instead of letters — share-route-migration final shape, 2026-08) — then **manually re-associate the new edge version ARN in CloudFront** (it's version-pinned; nine per-slug path-pattern behaviors, dev + prod) |
| 4 | API `utils/referenceCodeGenerator.ts` + `validation/referenceCode.ts` | `VALID_TYPES` (both) |
| 5 | web `src/app/SmartRouter.tsx` + `src/app/ShareDetail/index.tsx` | both local `isReferenceCode` letter lists |
| 6 | mobile `src/utils/referenceCode.ts` | `ReferenceType` union + `VALID_TYPES` |
| 7 | mobile `src/services/api/Reference.ts` | `buildIncludes` cases for the new letters |

> Post share-route-migration (2026-08) note: routing no longer decodes letters, so the live
> cross-repo sync concern is the **slug taxonomy**, mirrored in five places — API
> `src/constants/shareTaxonomy.ts` (source of truth), web `src/helpers/shareTaxonomy.ts`,
> mobile `src/utils/referenceCode.ts`, the edge slug regex (+ CloudFront behaviors), and
> AndroidManifest's nine `pathPrefix` entries. Letter lists (rows 4–6 above) remain only for
> minting and mobile slug derivation; web's ShareDetail/SmartRouter letter lists (row 5) are gone.

Smoke test per the OG doc: curl `/reference/<type>/<code>/og.png` AND `/meta` separately, then
fetch the share page as a crawler (`curl -A facebookexternalhit`) and grep for `og:image`.

### OG image templates (`src/services/og/templates/`)

Two new templates (server-side Satori — local `COLORS` hex map is the convention there, mobile
token rules do not apply). Keep v1 deliberately simple (Skylar: "gets it working, clean up later"):

- **`post.ts`** — dark field. Circular **avatar + @username** row, then the post's text directly
  beneath it — `Post.body` or the first `RICH_TEXT` content item — **clamped** (the existing
  `clamp()` helper; the OG canvas is small, keep it to a sentence or two). No "Post by" kicker on
  the image (revised 2026-08-03) — the meta title carries that line. nanza wordmark footer
  bottom-left (the shared `footerLogo()`). **No attachment imagery v1** — a later pass can drop
  the embedded image/reference art in.
- **`tag.ts`** — ONE template for both tag types (they're structurally identical). The value's
  **banner** full-bleed (fallback: thumbnail, then a plain dark field), a **bottom drop-shadow
  gradient** reusing the collection template's recipe (transparent → dark), and the nanza
  wordmark bottom-left in it. **No on-image name** (revised 2026-08-03: banner + wordmark only —
  the meta title is the only place the name appears). Both models have `banner` *and*
  `thumbnail`; banner is the shot.

OG cache is keyed `share/<CODE>-<updatedAtMs>.webp` (revised 2026-08-04 — the renderer now
outputs WebP q80, not PNG; see [[../nanza-api/solution-designs/og-sharing|OG / Sharing]]), and
all three models carry `updatedAt` — **post edits bust the image cache for free**. Deleted posts
resolve null → 404 → no unfurl, correct.

### Meta (`buildOgMeta` cases)

- **Post** — title: `Post by @username` (no `| nanza` suffix — revised 2026-08-03, the on-image
  wordmark brands it); description: clamped body text; image: the `og.png` URL as everywhere.
- **Tag values** — title: `displayName` alone (no `| nanza` suffix, same revision — and the only
  place the name appears, since the image dropped it); description: clamped `description`,
  falling back to something generic ("Join the conversation on nanza").

## Web app (nanza-web-app)

- Widen both `isReferenceCode` copies (SmartRouter + ShareDetail).
- `ShareDetail` gains two cards, in the existing landing style (`ShareDetailShell` + app-store
  CTA): **`PostDetailCard`** (avatar/username/time header, body text, attachments as a simple
  image/link row v1) and **`TagDetailCard`** shared by both tag types (banner hero, name,
  description). Web is an unfurl/landing surface — keep these thin; the CTA is the app download,
  as on the other cards.

## Mobile (nanza-mobile)

### Share entry points

- **Post cards** — the footer **share icon that's already drawn in the mocks** (parked
  non-functional in the chat release, top-right of the footer row) goes live: mint-then-share
  (`POST /post/:id/reference-code` → `guardedShare(buildShareUrl(code))`). Root posts only —
  replies render no share affordance (they use the minimal footer already).
- **PostDetail screen** — the floating **share circle top-right** (also already in the mock,
  parked), same mint-then-share flow.
- **Tag screens** (both) — a floating share circle in the header row next to the existing
  floating back (the profile-screen share-2 treatment). Code is already on the DTO (minted at
  create/backfill) → `guardedShare` directly, gated on `referenceCode` presence like the
  collection row is.

### Deep links

No navigator changes. The share-link interception (`/<slug>/<code>` — `parseShareReferenceTarget` →
`Share` route) keys on the share-type slug segment and is letter-agnostic — cold and warm starts
both work.

### Share cards (`ShareDetailView` cases)

Skylar's call: **the share landing should look pretty much identical to the real screens**,
built from the components those screens already use — the same way `ShareListingCard` mirrors
`ListingScreen` ([[share-card-restyle|Share Card Restyle]]). We keep the established pattern —
dedicated chrome-less share cards, NOT re-pointing deep links at the app screens (rejected in
the restyle design; those screens assume the app shell):

- **`SharePostCard`** — reuses **`PostItem` in its detail-root treatment** (flat on the dark
  page, `bodyLg`, full footer) + **`PostContentItems`** for the attachment, so the shared post
  renders exactly like `PostDetailScreen`'s root. **Root post only — no replies on the landing**
  (decided). CTA: auth-gated **reply** (the share screens' `useShareActions` gate → into the app
  at `PostDetail`).
- **`ShareTagValueCard`** — ONE card for both tag types, mirroring the tag screens: banner with
  the collection-detail fade treatment, jumbo title, description, the `TagCarousel` rail where
  applicable. CTA: auth-gated **Join the chat** → into the app at the real tag screen.

`ShareDetailView` switches on `result.type`: `'Post'`, `'SupportedTagValue'`,
`'SecondaryTagValue'`.

## Types (`@oakplatforms/types`)

`referenceCode?: string` on `PostDto`, `SupportedTagValuesDto`, `SecondaryTagValueDto`; the
mint endpoint's response shape. Regenerate in `nanza-api/packages/types`, bump, **Skylar
publishes** — web + mobile install before their FE passes (backend-first rule from
[[sharing|Sharing]]: resolver/OG/meta can ship ahead of the FE cards).

## Rollout order

1. **Chat release lands first** (mobile commit + `ContentItem.parentId`/enum migrations —
   already queued). This feature's migration rides after it.
2. **Schema migration** — the three `referenceCode` columns (additive). Skylar runs it.
3. **nanza-api** — letters (both files), resolver cases, mint endpoint, tag-create minting,
   backfill for existing tag values, OG templates + meta cases + edge regex. Deploy; remember
   the **CloudFront edge version re-association**. Smoke-test with curl per the OG doc.
4. **Types** bump + publish.
5. **nanza-web-app** — letter lists + the two cards.
6. **nanza-mobile** — letter list, `buildIncludes` cases, share entries, the two share cards.

Steps 3–6 are independently shippable; a live `T/V/Y` link before the FE ships just unfurls
correctly and lands on the web card, which is fine.

## Key decisions & rationale

- **Root posts only.** A reply's context is its thread; sharing it out of context is a later
  problem. Enforced server-side at mint AND resolve, not just hidden in the UI.
- **Lazy minting for posts, create-time for tag values.** Post volume vs. the ~196k-per-letter
  code space; tag values are few and admin-owned. The mint endpoint is idempotent so double-taps
  and races are harmless.
- **One tag template / one tag share card for both types.** The two models are structurally
  identical for display (name/banner/description); splitting them is pure duplication. The
  letter still distinguishes them for resolution and navigation.
- **Simple OG v1.** Avatar + name + clamped text for posts; banner + gradient + name for tags.
  Attachment art in the post OG image is a follow-up, not a blocker.
- **Reuse the detail components inside the share cards** rather than re-pointing deep links at
  app screens — same reasoning as the share-card restyle: bring the look to the chrome-less
  landing, not the app shell to the link.
- **Public resolution is content exposure — posture check.** `/reference/:code` is
  unauthenticated by design (crawlers). Post bodies become crawlable/preview-able once shared;
  DELETED posts resolve null, and the updatedAt-keyed OG cache means edits don't serve stale
  cards.

## Open questions — all resolved (Skylar, 2026-08-03)

1. **Letters `T` / `V` / `Y`** — ✅ confirmed (the letter table here + [[sharing|Sharing]] stays
   the index to reference them by).
2. **Replies on the share landing** — ✅ **root post only** in v1; no thread below.
3. **Tag screens' share entry** — ✅ **floating header circle** next to the floating back (the
   profile-screen treatment).
4. **Web cards** — ✅ **thin**. The web landing is deliberately a fallback — the goal is pushing
   people into the in-app share component — so basic info + app-store CTA only, no rails.

## Implementation record — nanza-api (2026-08-03)

The API half is built (uncommitted; **migration not generated** — Skylar runs `prisma migrate`
after committing). What landed, and the deltas from the design:

- **Schema**: `referenceCode String? @unique` + `@@index` on all three models. Additive only.
- **Letters**: `T/V/Y` added to BOTH API letter lists (`utils/referenceCodeGenerator.ts`,
  `validation/referenceCode.ts`).
- **Resolver**: `T` resolves root+ACTIVE posts only (replies/deleted → null) and returns the
  post **exactly as `/posts` serves it** — `postInclude` + reference expansion + like/reply
  counts, viewer always null. To share that shape, **`postInclude` and `hydratePosts` moved
  from `routers/post.ts` into `services/postReferences.ts`** (the payload-budget file); the
  router imports them now. `V`/`Y` are plain `findUnique` by code, include-driven.
- **Mint endpoint**: `POST /post/:id/reference-code` — any signed-in user, 404 on missing/
  non-ACTIVE, 400 on replies, idempotent. The update is guarded
  (`where: { id, AND: [{ referenceCode: null }] }`) so two racing sharers can't clobber each
  other's code — the loser's P2025 re-reads and returns the winner's code.
- **Tag minting**: both create routes wrap `generateReferenceCodeWithRetry` — including the
  admin chip flow, where the **whole `$transaction` sits inside the retry** (safe: the lookup
  and link-upsert are idempotent; the code is only consumed on the create branch).
- **Backfill**: `migration-scripts/backfill_tag_value_reference_codes.sql` — idempotent
  PL/pgSQL (NULL-code rows only, per-row collision retry), DBeaver-run, dev then prod.
- **OG images**: `templates/post.ts` (dark field, avatar + @handle, muted "Post by @user"
  kicker, 4-line clamped preview text, wordmark) and `templates/tag.ts` (one template for V+Y:
  banner cover, collection-style bottom shadow, 2-line name, wordmark). The post card reuses
  `renderCardOg` by **riding the avatar on the single-hero `imageKey` slot** — no bespoke
  render path. Post preview text = `body` else first RICH_TEXT item, whitespace-collapsed,
  200-char clamp in the map + line clamp in the template.
- **Meta**: `Post` → title `Post by @user | nanza`, description = the preview text;
  tag values → title `<name> | nanza`, description = the value's description; both fall back
  to "Join the conversation on Nanza".
- **Footgun maps**: all three lambda maps updated (`TYPE_BY_LETTER`, `RECORD_TYPE_BY_LETTER`,
  edge regex → `[2-9SBCPKGUTVY]`). **Deploy reminder: re-associate the new ogEdge version ARN
  in CloudFront.**
- **Types**: `@oakplatforms/types` regenerated (referenceCode on Post/SupportedTagValue/
  SecondaryTagValue DTOs), bumped **0.1.85**, built clean. Skylar publishes.

Verified: `tsc --noEmit` clean, eslint clean on all touched files. Still to do: migration +
backfill + deploy + `@oakplatforms/types` publish (Skylar). Web and mobile passes landed
2026-08-03 (below).

## Implementation record — nanza-web-app + nanza-mobile (2026-08-03)

Both FE passes are built (uncommitted), against `@oakplatforms/types` **0.1.86** (mobile
installed it; web stayed on 0.1.54 — its ShareDetail maps the resolver payload as `any`, so
the bump isn't needed there and a 30-version jump was not worth the risk in this pass).

**nanza-web-app** (per the design: thin, app-store CTA is the sell):

- All **three** local letter lists widened (`ShareDetail/index.tsx`, `SmartRouter.tsx`, *and*
  `ShareDetail/data/fetchByCode.ts` — the design's map said two; fetchByCode's
  `extractTypeIdentifier` was a third copy).
- `PostDetailCard` — avatar/@username/date header, body text (falls back to first RICH_TEXT
  item), attachments as a simple image row (≤3, square) + bare link rows, likes/replies meta
  line. `TagDetailCard` — one card for V+Y: banner hero (thumbnail fallback), name,
  description. Both in the existing `ShareDetailShell` style; T/V/Y send **no** client
  includes (the resolver hydrates posts itself; tag values render from their own columns).
- Verified: `tsc --noEmit` + eslint clean.
- ⚠️ **Drift found (pre-existing, not fixed here):** web's `U` handling still expects
  `ProfileListings`/`ProfileBids` + `?type=sell|bid`, but the API resolver now returns the
  unified `{ type: 'Profile', data: { profile, bids, listings, summary } }` shape mobile
  uses — so web profile shares currently fall through to the landing redirect. Needs its own
  pass.

**nanza-mobile** (against the uncommitted chat-release tree on `tag-screens`):

- `utils/referenceCode.ts`: `ReferenceType` + `VALID_TYPES` grew T/V/Y — the deep-link
  interceptor is letter-agnostic from here. `services/api/Reference.ts`: T/V/Y documented as
  no-include types. `useFetchByReference`: envelope union grew
  `Post → HydratedPostDto`, `SupportedTagValue`/`SecondaryTagValue` DTOs.
- `postService.mintReferenceCode(postId)` → `POST /post/:id/reference-code`; wrapped by a new
  **`useSharePost`** hook (mint-then-`guardedShare`, isSharing guard, error alert).
- Share entries: **PostItem footer** share-2 icon (root posts only, full footer only, hidden
  for guests via `useCanShare` like every in-app entry; sits `marginLeft:auto` beside the
  owner trash — new `footerShare` style in `styles/components/posts.ts`). **PostDetailScreen**
  floating share-3 darkCircle in the header. **Both tag screens**: floating share-3 circle
  next to the floating back, gated on `referenceCode` presence + `useCanShare` — no lazy
  minting for tag values.
- Share cards: **`SharePostCard`** = PostItem in its `detailRoot` treatment (full footer, no
  replies below — decided) + auth-gated **Reply** CTA into the app at PostDetail; the owner's
  trash works and invalidates the reference query so a deleted post flips the landing to
  "link not found". **`ShareTagValueCard`** = one card for V+Y mirroring the tag screens
  (banner fade treatment, jumbo title, description, secondary-value rail for V only) +
  auth-gated **Join the chat** CTA into the real tag screen. Both CTAs live in
  `useShareActions` as `onReplyToPost` / `onJoinTagChat` (guest → auth modal with
  `authReturnTo` back to the Share landing). `ShareDetailView` switches on the three new
  types.
- Verified: eslint clean on all touched files; `tsc --noEmit` adds no errors beyond the
  pre-existing ones in the chat-release working tree.

## Related

- [[sharing|Sharing]] — the reference-code system this extends.
- [[share-card-restyle|Share Card Restyle]] — the "share card mirrors the detail screen"
  precedent the new cards follow.
- [[posts-chat|Posts Chat]] — the post cards/detail screen whose parked share affordances go
  live here.
- [[../nanza-api/solution-designs/og-sharing|OG / Sharing]] — the three-Lambda pipeline + the
  letter-map footgun.
- [[../nanza-api/solution-designs/posts|Posts (API)]] — the Post model, content items, and the
  "Share: deferred" note this design picks up.
- [[../nanza-api/solution-designs/tag-taxonomy-v2|Tag Taxonomy v2]] — the two tag-value models.
