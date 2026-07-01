# Plan — Profile (U) share views in nanza-web-app

## Goal
Make `https://<share-base>/<U-referenceCode>?type=sell|bid` render a real landing page in the
web app. The backend (nanza-api) and the CloudFront/Lambda@Edge OG injector already fully support
profile (`U`) codes and the `?type=sell|bid` selector. The web SPA is the only missing piece: it
explicitly rejects `U` codes and never reads the `?type` param, so a human who taps a shared
profile link is bounced to `/`.

## Evidence (current state)
- `src/app/ShareDetail/index.tsx:38` — `validTypes = ['S','B','C','P','K','G']` (no `U`); invalid
  codes `Navigate to="/"` at line 82–84.
- `src/app/SmartRouter.tsx:17` — same `validTypes` array, no `U`.
- `src/app/ShareDetail/data/fetchByCode.ts:9` — `extractTypeIdentifier` loops over
  `['S','B','C','P','K','G']` (no `U`); `buildIncludes` has no `U` branch and never appends `?type`.
- `src/services/api/Universal.ts:9` — `getByCode(referenceCode, parameters)` → `GET /reference/{code}{parameters}`.
- API (confirmed in nanza-api): `referenceResolver.resolveProfileReference` **requires** `?type=sell|bid`
  and returns `null` (→ 404) without it; profile must be `isPublic`. Response shape:
  `{ type: 'ProfileListings' | 'ProfileBids', data: { profile, items, summary } }`.
- Mobile reference points: `ShareProfileView.tsx` (render shape) and `services/api/Reference.ts:61`
  (`if (view) params.append('type', view)`).

## Design
Mirror the mobile `ShareProfileView`: avatar + username header → `N listings|bids · $TOTAL`
summary → vertical stack of the existing `FeedDetailCard` per item. Reuse `FeedDetailCard` and the
existing `mapEntityCard` so listing/bid items render identically to single-listing shares.

### Files

1. **`src/app/ShareDetail/index.tsx`** (edit, ~25 lines)
   - Add `'U'` to `validTypes` (line 38).
   - Read `?type` via `useSearchParams()`; pass `view: 'sell' | 'bid'` down.
   - Handle `type === 'ProfileListings'` / `'ProfileBids'`: render a new `ProfileDetailCard`
     (or inline) using `data.profile`, `data.summary`, and `data.items` mapped through
     `mapEntityCard(item, kind, conditions)`.
   - Default a missing/invalid `?type` to `'sell'` (matches the share buttons' default).

2. **`src/app/SmartRouter.tsx`** (edit, 1 line)
   - Add `'U'` to `validTypes` (line 17) so the `/:username/:referenceCode` path also accepts it.
   *(Confidence this path is exercised for profiles: 0.6 — the bare `/:referenceCode` route in
   AppLayout is the primary share target. Including it is low-risk parity; see Q1.)*

3. **`src/app/ShareDetail/data/fetchByCode.ts`** (edit, ~10 lines)
   - Add `'U'` to `extractTypeIdentifier`.
   - New signature `useFetchByCode(referenceCode, view?)`: for `U`, append `type=<view>` to the
     query and **omit client includes** (the API attaches `OG_PROFILE_INCLUDES` server-side).
   - Add `view` to the `queryKey` so sell/bid don't collide.

4. **`src/app/ShareDetail/ProfileDetailCard.tsx`** (NEW, ~70 lines)
   - Props: `username, avatarUrl, mode ('sell'|'bid'), count, totalValue, items` (already
     `mapEntityCard`-shaped). Tailwind, light theme, mirrors `GroupDetailCard` header + a stacked
     list of `FeedDetailCard`. Empty state when `items.length === 0`.

## Confidence
- Add `U` to validators + read `?type` + render via existing `FeedDetailCard` — **0.9**.
- `data.items` already carry `entity` + condition data from `OG_PROFILE_INCLUDES` so `mapEntityCard`
  works unchanged — **0.78** (verify the resolver's profile includes match `mapEntityCard`'s reads:
  `entity`, `account.profile`, `condition`/`conditions`, `price`). If they differ, add a small
  profile-specific mapper. Rejected alternative: re-fetch each item by code (N calls) — wasteful.
- SmartRouter `U` parity is needed — **0.6** (the bare route is the real target; including it is
  harmless). See open question Q1.

## Open questions
- **Q1**: Is the `/:username/:referenceCode` (SmartRouter) path ever used for profile shares, or is
  the bare `/:referenceCode` → `ShareDetail` the only entry? (Affects whether SmartRouter needs `U`.)
- **Q2**: "Version 8" — not the web app (`package.json` is `0.1.0`). Likely the edge function /
  CloudFront behavior version in nanza-api. Confirm separately; not a web-app code change.
- **Q3**: Deploy: `deploy-dev.yml` does **not** invalidate CloudFront; prod does. A dev test of the
  human landing page may need a manual invalidation.

## Out of scope
- No nanza-api changes (backend + OG already done).
- No CloudFront/edge changes (already accepts `U` + forwards `?type`).
