# Share a Profile — Profile Reference Code + Sell/Bid Share Cards

## Goal

Let a user share **their listings** and **their bids** as dedicated share cards, both
backed by **one profile reference code** per user, distinguished by a **query parameter**:

- `https://nanza.app/<profileCode>?type=sell` → **Listings card** (their listings + a "X listings · $TOTAL" summary)
- `https://nanza.app/<profileCode>?type=bid` → **Bids card** (their bids + a "X bids · $TOTAL" summary)

The bare profile code (no query param) is **not supported** — only the two parameterized
forms render a card. Both cards share a common header: profile **banner**, **avatar**,
**username**, and a two-line summary (count line + total-value line).

This reuses the existing universal reference-code system (`/reference/:code`,
`ShareDetailView`, deep-link interception) and the existing share-card chrome.

---

## Current state (verified)

- Reference types today: `S` Listing, `B` Bid, `C` List, `P` Product, `K` BulkListing, `G` Group.
  Type is derived **from the code's letter** by `extractTypeIdentifier` (API) /
  `extractReferenceType` (mobile). No `referenceCode` exists on `User` or `Profile`.
- The displayable identity lives on **Profile** (`username`, `avatar`, `banner`, `description`),
  one-to-one with `Account`. `Profile` also already has `listingCount` and `bidCount` Int fields.
- `GET /reference/:referenceCode` (universal.ts) reads only `?include=`. Resolver returns `{ type, data }`.
- Mobile deep links: `parseShareReferenceCode()` extracts the first path segment **and strips query params**.
  `getStateFromPath` + a `Linking` listener route to `Share` with `{ referenceCode }`.
- `ShareDetailView` switches on `result.type` to pick a card. `ShareScreen` reads `route.params.referenceCode`.
- Web app is a React Router SPA; `ShareDetail` reads only `useParams().referenceCode`, no query params.
- No pre-aggregated "total value of a user's listings/bids" exists; sell-feed/bids compute per-page in memory.

---

## Key design decision — how to model "one code, two views"

**Decision: a new `U` (profile/user) reference type, with `?type=sell|bid` selecting the view.**
Confidence: **0.82**

- `Profile.referenceCode` gets a `U`-letter code (e.g. `4U2839`), generated like every other code.
- The resolver, on a `U` code, reads the `type` query param to decide whether to attach
  listings+total or bids+total, and returns a discriminated payload.
- One code per profile (matches the request). The two share cards are picked by `type`, not by the letter.

**Rejected alternative 1 — two separate codes (one Listing-list code, one Bid-list code).**
Confidence: 0.55. Rejected: the user explicitly wants **one** reference code (the profile), with query
params selecting the view. Two codes contradicts the request and doubles generation/storage.

**Rejected alternative 2 — encode the view in the path (`/<code>/sell`).** Confidence: 0.5.
Rejected: the existing deep-link interceptor keys on the **first path segment only** and would need
deeper surgery; query params are the user's stated mechanism.

---

## Plan A — API (nanza-api)

### A1. Schema — add `referenceCode` to Profile  · Confidence 0.9
`prisma/schema.prisma`, `Profile` model:
```prisma
referenceCode String? @unique
```
Add an index consistent with the other models. **User runs the migration** (per memory:
user handles DB migrations — do not run `prisma migrate`). Also requires bumping
`@oakplatforms/types` so `ProfileDto.referenceCode` exists on the frontends.

### A2. Register `U` in the reference-code generator + validators · Confidence 0.9
- `src/utils/referenceCodeGenerator.ts`: `U` is just a type identifier passed to
  `generateRandomReferenceCode('U')`; confirm no allow-list needs editing. If there's a
  shared "valid type letters" constant on the API, add `U`.
- Backfill: generate codes for existing profiles (one-off script the user runs, or lazy-assign
  on first share — see A4). **Recommend lazy-assign** to avoid a migration-time backfill.

### A3. Assign a profile code · Confidence 0.85
Lazy approach: add `POST /profile/:id/reference-code` (or fold into existing profile fetch) that,
if `profile.referenceCode` is null, calls `generateReferenceCodeWithRetry({ typeIdentifier: 'U', ... })`
and persists it. Mobile/web request this when the user taps "Share my listings / bids".
Keeps parity with how Group/List got codes (generated on demand at create time).

### A4. Resolver — handle `U` + `type` param · Confidence 0.78
`src/services/referenceResolver.ts`: the resolver currently takes `(prisma, code, includes)`.
Add an optional `params` argument (or a dedicated `resolveProfileReference`) so the `U` case can read `type`:
- `type === 'sell'` → load the profile + that account's **listings** (reuse the sell-feed query
  shape; ACTIVE singles + PUBLISHED bulks, newest-first, paginated) **plus** an aggregate
  `{ count, totalValue }` (SUM(price*quantity)).
- `type === 'bid'` → load the profile + that account's **ACTIVE bids** plus `{ count, totalValue }`.
- Missing/invalid `type` → return `null` (renders the "link not found" state; bare code unsupported).

Return shape (new discriminated type): `{ type: 'ProfileListings' | 'ProfileBids', data: { profile, items, summary } }`.

Aggregation: add `count` + `_sum`/reduce for `totalValue`. Confidence 0.75 — needs care that
"total" matches what the user means (sum of price×quantity for listings; sum across active bids).
**Confirm the exact total definition with the user before finalizing.**

### A5. Route — thread the query param · Confidence 0.85
`src/routers/universal.ts` `GET /reference/:referenceCode`: pass `req.query.type` into the resolver
alongside `include`. No new route needed.

---

## Plan B — Mobile (nanza-mobile, `src/` only)

### B1. Preserve query params through the deep link · Confidence 0.75
`src/utils/referenceCode.ts` + `src/navigation/AppNavigator.tsx`:
- Extend `parseShareReferenceCode` to also return the `type` query value (or add a sibling
  `parseShareReferenceTarget(url) → { referenceCode, type }`).
- `getStateFromPath` and the `Linking` listener pass `{ referenceCode, type }` into the
  `Share` route params.
- `'U'` added to `VALID_TYPES` / `ReferenceType`.

> ⚠️ This touches `AppNavigator.tsx`, normally off-limits (P0 #7). Per memory
> ([[feedback_nav_changes_permitted_when_cleaner]]) the user permits nav edits when cleaner —
> still flag it explicitly at approval. The change is additive (extra param), no route reordering.

### B2. Reference service — send `?type=` and request profile includes · Confidence 0.85
`src/services/api/Reference.ts`: `buildIncludes` gains a `U` branch
(`account.profile`, listings/bids relations) and `getByCode` appends `&type=sell|bid`.
`useFetchByReference` gains the `type` arg and a new query-key segment.

### B3. New share cards · Confidence 0.85
`src/components/share/ShareProfileListingsCard.tsx` and `ShareProfileBidsCard.tsx`
(or one `ShareProfileCard` with a `mode` prop — **lean to one component, two modes**, to share
the banner/avatar/summary header). Built from the `ShareGroupCard` template (banner + gradient +
text overlay) plus avatar + the two summary lines + the item list. Styles in
`src/styles/components/share.ts` (no inline styles; theme tokens only; `$` via `PriceText`).

### B4. Type switch · Confidence 0.9
`src/components/share/ShareDetailView.tsx`: add `ProfileListings` / `ProfileBids` cases.
`ShareScreen.tsx` + the `ReferenceResult` union in `useFetchByReference.ts` extended.

### B5. Share entry points · Confidence 0.7
Add "Share my listings" / "Share my bids" actions (profile screen ellipsis or the existing
share sheet) that build `https://nanza.app/<profileCode>?type=sell|bid`, requesting/lazy-creating
the code via A3 first. **Confirm where the share buttons live with the user.**

---

## API status — BUILT (this session)

- `Profile.referenceCode String? @unique` + index added (schema.prisma). **Migration is user-run.**
- `U` registered in `src/validation/referenceCode.ts` + `src/utils/referenceCodeGenerator.ts`.
- `resolveByReferenceCode` gained optional 4th arg `params:{view}`; new `resolveProfileReference`
  returns `{ type: 'ProfileListings'|'ProfileBids', data: { profile, items, summary:{count,totalValue} } }`.
  totalValue = Σ(price×quantity); bids are ACTIVE+public only; listings ACTIVE+non-bundled.
- `/reference/:code` forwards `?type` → `{ view }`.
- **OG graphic BUILT** (not blank): `src/services/og/templates/profile.ts` — 5-across card mosaic,
  HARD left gradient (only right 3-4 cols visible), avatar + @handle header, big "Listings"/"Bids"
  title + "N listings · $TOTAL" summary, nanza wordmark bottom-left. `mapProfileCard` +
  `OG_PROFILE_INCLUDES`/`PROFILE_GRID_SIZE` in fields.ts; `renderProfileOg` in og/index.ts
  (fetches avatar+heroes, summary-based cache key). `lambdas/ogHandler.ts` reads `?type`, maps
  `U`→view, dispatches to renderProfileOg. `meta.ts`: title "Listings from @username" /
  "Bids from @username", desc "7 listings · $154.00", og.png URL carries `?type=`.
- Edge: `lambdas/ogEdgeHandler.ts` regex now `[2-9SBCPKGU]` and forwards the querystring to /meta;
  `lambdas/metaHandler.ts` maps `U`→view and passes it through.
- tsc + eslint clean across all touched API files; Prisma client regenerated locally.

## Plan C — Web app (nanza-web-app) · Confidence 0.7

`src/app/ShareDetail`:
- Read `useSearchParams()` for `type` and pass it to `fetchByCode` (append `&type=`).
- `useParams().referenceCode` validation: allow the `U` letter.
- Add `ProfileListings` / `ProfileBids` render branches (reuse `FeedDetailCard` for the items grid,
  new header for banner/avatar/summary).
- OG/meta: out of scope here unless requested (web is a client SPA; per-share meta needs the
  Lambda@Edge injector already built in the shareability OG phase — separate effort).

---

## Decisions (confirmed with user)

1. **Total definition** — Σ(price × quantity) over ACTIVE listings; Σ(bid price × quantity) over
   ACTIVE bids. (A4)
2. **Share entry** — on the profile screen, reuse the spot where the existing **share-2 icon**
   lives; bump that icon **+1pt** larger. (B5)
3. **Order** — start with the **API** work, then mobile, then web.

## Still to confirm later
- **Code assignment** — lazy-create the profile code on first share (recommended) vs. backfill.

## Risk / files touched
- API: `schema.prisma`, `referenceResolver.ts`, `referenceCodeGenerator.ts` (maybe), `universal.ts`, profile route.
- Mobile: `referenceCode.ts`, `AppNavigator.tsx` (flagged), `Reference.ts`, `useFetchByReference.ts`,
  `ShareDetailView.tsx`, `ShareScreen.tsx`, new card(s), `styles/components/share.ts`, share entry point.
- Web: `ShareDetail/index.tsx`, `fetchByCode.ts`, new card(s).
- Blocked on: `@oakplatforms/types` bump for `ProfileDto.referenceCode` + the new resolver payload types
  (no custom DTOs per [[feedback_no_custom_dtos]]).
- Migration is **user-run** ([[feedback_user_handles_db_migrations]]).
