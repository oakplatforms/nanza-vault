---
tags: [nanza-web-app, solution-design, sharing]
---

# Sharing — Solution Design

## Overview

Sharing is the **core live purpose** of the deployed web app: it is the public landing
surface for Nanza's shareable reference-code links. When someone opens a share link, the
app either sends them to the native app store (mobile, no app installed) or renders a
read-only card of the shared thing (desktop). This covers every share type —
including profile links (`/profile/<code>?type=sell|bid`) — plus the store-redirect
nudge and the store buttons in each card's header.

The backend contract (nanza-api reference resolver) and the CloudFront/Lambda@Edge OG
injector already fully support every code type; the web SPA is the human-facing renderer
for the same URLs.

## How it works

### Reference-code landing — `/<typeSlug>/:referenceCode`

**Nine root-level type-scoped routes** — `/<slug>/:referenceCode` for `listing`, `bid`,
`collection`, `product`, `bulk`, `group`, `profile`, `post`, `tag` — all render
`app/ShareDetail`, each passing a `shareType` prop (share-route-migration, 2026-08: this
final shape superseded the same-day interim `/share/:referenceCode` prefix; the slug names
the record type and the backend looks straight in that type's table. Old shapes are not
honored, no redirect; anything unmatched `Navigate`s to `/`). The **reference code is an
opaque token**: routing accepts a lenient 4–12 alphanumeric check and never decodes the
type letter — codes are still minted as 6 chars (five digits 2–9 + one letter), but the
old `isReferenceCode()` letter validation is deleted from `ShareDetail/index.tsx` and
`fetchByCode.ts`. The slug↔type taxonomy lives in `src/helpers/shareTaxonomy.ts`,
mirroring the API's source of truth (`nanza-api/src/constants/shareTaxonomy.ts`). Routes
and their cards:

| Route slug | Backing record | Card |
|------------|----------------|------|
| `/listing` | Listing | `FeedDetailCard` |
| `/bid` | Bid | `FeedDetailCard` |
| `/product` | Product (resolves to Entity) | `ProductDetailCard` |
| `/collection` | List / collection | `ListDetailCard` |
| `/bulk` | Bulk listing | `BulkDetailCard` |
| `/group` | Group | `GroupDetailCard` |
| `/profile` | Profile (`?type=sell\|bid`) | `ProfileDetailCard` |
| `/post` | Post (root only) | `PostDetailCard` |
| `/tag` | Supported tag value | `TagDetailCard` |

The post/tag cards (post-tag-sharing design, 2026-08-03) are deliberately **thin** — the
web landing is a fallback pushing visitors to the app. `PostDetailCard`: author header
(avatar/@username/date), body text (falling back to the first `RICH_TEXT` content item),
attachments as a simple image row + bare link rows, and a likes/replies meta line.
`TagDetailCard`: one card for both tag types — banner hero (thumbnail fallback), name,
description. Post/tag shares send **no** client includes: the resolver hydrates posts
server-side and tag values render from their own columns.

`ShareDetail/index.tsx` takes the route's `shareType`, runs the lenient code check,
fetches via `data/fetchByCode.ts` (→ `universalService.getByCode` →
`GET /reference/{type}/{code}{params}` — the typed route that replaced the bare-code
lookup), maps the record into card props, and renders inside `ShareDetailShell`. Invalid
codes `Navigate` to `/`.

### Store redirect for app-less mobile visitors

With Universal Links (iOS) and App Links (Android) configured under
`public/.well-known/` (`apple-app-site-association`, `assetlinks.json`), a visitor who
**already has the app never sees the web page** — the OS opens the app directly. So the
web redirect only ever fires for visitors **without** the app, which is exactly who
should be sent to the store.

At the top of `ShareDetail` — **after** the code passes the lenient reference-code check,
so garbage URLs never trigger a redirect — a device-detection helper
(`src/helpers/getMobilePlatform.ts`) reads `navigator.userAgent` and returns
`'ios' | 'android' | null`. iPadOS 13+ reports as a Mac, so a Mac with
`navigator.maxTouchPoints > 1` is also treated as iOS. On iOS/Android it calls
`window.location.replace(storeUrl)` (using `replace` so Back doesn't trap the visitor on
the share page) and renders nothing. Desktop falls through to the normal card.

Because mobile redirects away, the in-card store buttons are desktop-only in practice —
intended and fine.

### Store URLs — single source of truth

Store URLs are centralized in `src/constants.ts` and consumed by both the landing site
and the share cards:

```ts
export const APP_STORE_URL = 'https://apps.apple.com/us/app/nanza/id6757965694'
export const PLAY_STORE_URL =
  'https://play.google.com/store/apps/details?id=com.oakplatforms.nanza&hl=en_US'
```

The landing page's App Store / Google Play links (`landing/index.tsx`,
`landing/components/LandingLayout.tsx`) point at these constants — earlier placeholder
`href`s were replaced.

### Share-card header

Every share card carries a header rendered once inside `ShareDetailShell`
(`ShareDetail/ShareCardHeader.tsx`), sitting inside the card container: the Nanza logo
top-left (links to `/`) and the two store buttons on the far right, reusing the same
store-button markup as the landing page. Rendering it in the shell rather than in each of
the five variant layouts means every variant (Listing/Bid/Product/List/Bulk/Group/Profile)
gets it for free — this replaced the earlier per-variant centered `ShareFooterLogo`.

### Profile share views

Profile links render `/profile/<code>?type=sell|bid` as a real landing page. The backend
resolver **requires** `?type=sell|bid` (returns `null` → 404 without it) and the profile
must be `isPublic`; the response is
`{ type: 'ProfileListings' | 'ProfileBids', data: { profile, items, summary } }`.

On the web side, the `/profile/:referenceCode` route hands `ShareDetail` the profile
`shareType` (no letter validation anymore), the `?type` param is read via
`useSearchParams()` and passed down as `view: 'sell' | 'bid'` (missing/invalid defaults
to `'sell'`, matching the share buttons' default). `data/fetchByCode.ts` appends
`type=<view>` to the query for profile shares, omits client includes (the API attaches
its profile includes server-side), and includes `view` in the query key so sell/bid don't
collide.

`ProfileDetailCard` mirrors the mobile `ShareProfileView`: an avatar + username header, an
`N listings|bids · $TOTAL` summary from `data.summary`, and a vertical stack of the
existing `FeedDetailCard` per item — each item mapped through the shared `mapEntityCard`
so profile items render identically to single-listing shares, plus an empty state when
there are no items.

### SmartRouter (dormant)

`app/SmartRouter.tsx` keeps its own reference-code check to disambiguate `/:a/:b` between
a profile-reference view and a brand/entity slug, but the nine `/<slug>/:referenceCode` →
`ShareDetail` routes are the primary share target; the SmartRouter path is part of the
dormant marketplace routing.

## Key decisions & rationale

- **Redirect only after code validation.** Gating on a plausible reference code (the
  lenient 4–12 alphanumeric check — routing never decodes the letter) guarantees we never
  bounce someone off a mistyped or marketing URL to the store.
- **`window.location.replace`, not `assign`.** Keeps the store redirect out of history so
  Back returns to wherever the visitor came from, not to the share page.
- **UA sniff for platform.** A pragmatic choice for a redirect-to-store nudge, with the
  iPadOS-reports-as-Mac + `maxTouchPoints` correction; acceptable because the cost of a
  wrong guess is only which store button shows.
- **Header in the shell, not per-variant.** One render point covers all card variants and
  avoids repeating the same markup five-plus times (DRY).
- **Reuse `FeedDetailCard` + `mapEntityCard` for profile items.** Profile items already
  carry the `entity` + condition + price data the mapper reads, so listing/bid rows render
  identically to single-listing shares without re-fetching each item by code (N calls,
  rejected as wasteful).
- **Web is renderer-only for profiles.** No nanza-api or CloudFront/edge changes — the
  backend resolver and OG injector already accept `U` and forward `?type`.

## Related

- [[../INDEX|nanza-web-app]]
- [[../architecture|Architecture]]
- [[../fe_spec|Frontend Spec]]
</content>
