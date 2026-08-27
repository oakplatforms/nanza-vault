---
tags: [nanza-mobile, solution-design, sharing]
---

# Sharing — Solution Design

## Overview

Any shareable object in Nanza is addressed by a **universal reference code**: a short code
that maps to a domain record, resolves to a deep-linkable share card in-app, and unfurls to a
rich link preview (OG image + meta) on the web. A shared link (`https://nanza.app/<slug>/<code>`
— root-level type-scoped, the slug naming the record type) opens the app to a dedicated
**share card** for that object — with buy/sell/join actions gated behind auth — or, on a device
without the app, a web landing page and social preview.

The system is deliberately uniform: every shareable type plugs into the same reference-code
plumbing (`GET /reference/{type}/{code}`), the same in-app `ShareDetailView` switch, the same
OG/meta pipeline, and the same share-sheet entry points. Adding a new shareable type is a matter
of registering a slug (plus a letter for minting) and wiring one more case at each layer.

## How it works

### The reference-code scheme

Codes are minted as **6 characters: 5 digits (2–9) + 1 type letter** placed at a random
position, with uniqueness enforced per model by a `@unique` column. Since the
share-route-migration final shape (2026-08) the code is an **opaque token** — the share type
travels in the URL's slug segment, and routing never decodes the letter. The letter survives
for minting and for deriving the slug in `buildShareUrl` (until letterless codes ship):

| Letter | Type |
|---|---|
| `S` | Listing |
| `B` | Bid |
| `C` | List / Collection |
| `P` | Product (catalog) |
| `K` | BulKlisting (Lot) |
| `G` | Group |
| `U` | User / Profile |
| `T` | Post (root posts only) |
| `V` | SupportedTagValue |
| `Y` | SecondaryTagValue |

Codes are minted like the rest — generated on record creation (listing/bid/lot/group create,
tag-value admin create), or **lazily on first share** for profiles and posts (posts:
`POST /post/:id/reference-code`, idempotent). The web landing routes no longer validate
letters (a lenient 4–12 alphanumeric check); what must stay in sync instead is the slug
taxonomy — mobile's copy lives in `src/utils/referenceCode.ts`, mirroring the API's
`nanza-api/src/constants/shareTaxonomy.ts`.

### Resolution and rendering

`GET /reference/{type}/{code}` (API) takes the type from the path slug, looks straight in that
type's table with the type's includes, and returns `{ type, data }`. In the app,
`useFetchByReference` fetches by code and `ShareDetailView` switches on `result.type` to pick
the card; `ShareScreen` reads `route.params.referenceCode`. Deep links route to the `Share`
route via `getStateFromPath` + a `Linking` listener — the parser matches `<slug>/<code>`, and
AndroidManifest carries nine matching `pathPrefix` entries. Each shareable type has a dedicated
dark share card in `src/components/share/` built from shared parts in `ShareCardParts.tsx`.

### The buy/sell action matrix

Each share card carries the primary action that fits its object, all funneled through
`useShareActions` and **auth-gated first** (a guest tapping the CTA hits the auth wall, then
returns to the share):

- **Listing** → **Buy** (`onBuy`)
- **Bid** → **Make offer**
- **Lot (Bulk)** → **Buy** the lot
- **Group** → **Join** (`onJoinGroup`; request-to-join for private groups)
- **Collection** → browse (share/view, no purchase)
- **Product** → catalog view
- **Profile listings / bids** → browse the shared set

### Share entry points

Sharing is triggered from an **ellipsis / "more" menu** on each object's detail screen (built
with the standard `ActionModal` icon+label+description rows), calling
`guardedShare(buildShareUrl(referenceCode))` to open the native RN share sheet.
`buildShareUrl` mints `/<slug>/<code>` links, deriving the slug from the code's type letter at
this one choke point (until letterless codes ship).

- **Group detail** ellipsis rows are role-based: non-moderator gets *Leave group* / *Share
  group*; moderator gets *Edit group* / *Share group* / *Manage members*.
- **Collection detail** (own + read-only user) exposes *Share collection*, gated on the
  collection having a `referenceCode`.
- **Profile** shares reuse the existing **share-2 icon** spot (bumped +1pt).

### Sharing a Group (`G`)

A group shares as its **banner** image, full-bleed, with a bottom drop-shadow gradient
carrying the group **name** + **description** and the nanza wordmark bottom-left (styled like
the collection OG card but single-image). The in-app `ShareGroupCard` adds the member count and
a colored **Join** CTA. Shipping the group card was **gated** on a Prisma migration
(`Group.referenceCode` didn't exist) plus a `@oakplatforms/types` bump — the backend
resolver/OG/meta could be built immediately, but the frontend card was blocked until types
landed. Existing groups get codes backfilled by hand.

### Sharing a Profile (`U`)

A profile shares via **one `U` reference code per user** (minted lazily on first share), with a
**`?type=sell|bid` query parameter** selecting which view renders:

- `…/profile/<code>?type=sell` → **Listings card**: the user's active listings + a
  "N listings · $TOTAL" summary.
- `…/profile/<code>?type=bid` → **Bids card**: the user's active bids + a "N bids · $TOTAL" summary.
- The **bare code (no `?type`)** is unsupported — it renders the "link not found" state.

Both cards share a header: profile **banner**, **avatar**, **username**, and a two-line summary
(count line + total-value line). **Total** = Σ(price × quantity) over ACTIVE listings /
Σ(bid price × quantity) over ACTIVE bids. On the API, the resolver reads the `type` param and
returns a discriminated `{ type: 'ProfileListings' | 'ProfileBids', data: { profile, items,
summary: { count, totalValue } } }`. The OG graphic is a 5-across card mosaic with a hard left
gradient, avatar + @handle header, a big "Listings"/"Bids" title, and the summary line. Because
the query param must survive to the OG/meta pipeline, the **Lambda@Edge injector forwards the
querystring** to `/meta`, and the edge regex was widened to include `U` (today the edge matches
the nine type slugs, not letters). This required touching
`AppNavigator.tsx` to preserve the `type` query through the deep link — an approved, additive
nav change.

> A later refinement (tracked in project memory) collapsed the two `U` views into **one** card
> showing both bids and listings without a `?type` param, to fix an OG-unfurl edge case where
> CloudFront didn't forward the query. The two-view `?type` design above is the shipped mobile
> behavior; the unified single-card version is the intended end state.

### Tap-to-expand + cropped share images

Share cards route their imagery through one building block, **`ShareImageWell`** in
`ShareCardParts.tsx`, which gained two optional props (default behavior unchanged):

- **`onPress`** → the image becomes tap-to-expand, opening the shared `ImageViewer`. Single
  cards (Listing, Bid, Product) open one image; carousel cards (Lot, Collection) open the
  swipeable viewer aligned to carousel order, starting on the tapped slide (the `sources` +
  `initialIndex` pattern from `FeedBulkCard`).
- **`fill`** → renders `resizeMode="cover"` full-width (a cropped capture) instead of the
  default tiny `contain` fit. **Listings and Lots** use `fill` to match `FeedListingCard` /
  `FeedBulkCard`; Bids, Collections, and catalog Products keep `contain`.

Centralizing on `ShareImageWell` avoided duplicating the TouchableOpacity + viewer wiring
across five cards.

## Key decisions & rationale

- **Type lives in the path slug (final shape, 2026-08).** The deep-link interceptor keys on a
  share-type slug segment (`listing|bid|collection|product|bulk|group|profile|post|tag`) plus the
  code after it — superseding both the original bare-first-segment shape and the same-day interim
  literal-`share` segment. The code itself is opaque: routing never decodes its letter, which
  survives only for minting and `buildShareUrl`'s slug derivation. This is why the Profile view is
  selected by `?type=`, not by `/profile/<code>/sell`.
- **One profile code, two views (via `?type`).** The user wanted a single shareable identity
  per person; two separate codes was rejected as contradicting that and doubling
  generation/storage.
- **Lazy code minting for profiles.** Avoids a migration-time backfill — the code is generated
  the first time a user actually shares.
- **Auth-gated actions, return-to-share.** Every card CTA (Buy / Make offer / Join) hits the
  auth wall for guests, then returns them to the share — so a shared link converts a logged-out
  visitor without losing context.
- **Centralize image behavior on `ShareImageWell`.** Additive `onPress`/`fill` props on one
  shared part beat editing five cards' image JSX independently (DRY).
- **New shareable types are backend-first, then FE.** Reference plumbing + resolver + OG + meta
  can ship before the migration and `@oakplatforms/types` bump; the in-app card is blocked until
  types regenerate (no custom DTO stand-ins).

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
- [[post-tag-sharing|Post & Tag Sharing]] — proposed extension: three new letters (T/V/Y) for
  posts and supported/secondary tag values
- [[scan|Scan & Camera]] — the shared `ImageViewer` that share cards reuse
- [[collections|Collections]] — collection share card + the mosaic OG style
- [[profile|Profile]] — profile share entry points and read-only parity
- [[bulk|Bulk / Lots]] — the Lot (`K`) share card
