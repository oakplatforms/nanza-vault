---
tags: [nanza-api, nanza-web-app, solution-design, plan, profile, sharing]
status: BUILT 2026-10-07/08 — API committed to dev (55db256), web uncommitted; pending QA + dev deploy
---

# Public Profile Page — `nanza.app/<username>` (Plan)

> **Status: BUILT — API 2026-10-07 (committed to `dev` as 55db256), web 2026-10-08 (uncommitted); lint, tsc and the
> CRA production build are clean. Pending Skylar's QA, the username audit query, and a dev
> deploy of both.** Spans
> **nanza-api** (one new endpoint + username hardening) and **nanza-web-app** (a new root-level
> `/:username` surface). No schema change, no migration. Fold into a `public-profile` solution
> design when it lands.
>
> Built in nanza-api: `src/constants/reservedUsernames.ts` (shape regex + reserved set),
> `validateUsername` rewritten in `src/validation/profile.ts`, and `GET /user/:username/page`
> in `src/routers/user.ts`.
>
> Built in nanza-web-app: `src/app/ProfilePage/` (layout, hero, tabs, dark cards, tabs/, screens/,
> data/ hooks), the nested `/:username` route family in `AppLayout.tsx`, services `Post.ts`,
> `Feed.ts`, `BulkListing.ts`, `userService.getPage`, `listService.getEntities`, and the share
> mappers extracted to `ShareDetail/mappers.ts`. Not yet done: the one-off audit of existing
> usernames (Skylar runs it in DBeaver), dev deploy, QA against real profiles.

## Goal

Give every public profile a **link-in-bio style web page** at the root of the web app —
`nanza.app/skybanks` — that mirrors the mobile `UserProfileScreen` **read-only**: hero (banner,
avatar, @username, bio, social links), then **Posts / Collections / Trades** tabs. Every tap stays
on the web and stays read-only — a post opens its own page, a collection opens as a card grid
(tapping a card only enlarges it; **no entity / product page**), a trade opens a read-only listing
or bid screen with **no purchasing** of any kind. There are no writes anywhere on this surface.

The two things Skylar called out as the design's spine:

1. **The existence check must be cheap and must come first.** A visitor typing any word after
   `nanza.app/` must not trigger the expensive profile load. One request does both: it confirms
   the username is real (one indexed row read), and *only then* fans out for the page data; an
   unknown name is a `404` after a single query.
2. **The endpoint lives under the user namespace**, not under `/profile`, because the page is
   "a user's public page" — one call, one response, the whole hero.

## How it works

### The web URL space

Root-level `/:username` is the only new dynamic route family, declared **after** the nine share
routes and before the catch-all:

| Path | Screen | Data |
|------|--------|------|
| `/:username` | the page itself, **Posts** tab (default) | `GET /user/:username/page` + `GET /posts?accountId=` |
| `/:username/collections` | Collections tab (collection cards) | `GET /lists?type=COLLECTION&accountId=` |
| `/:username/trades` | Trades tab (listings + bulks + bids, newest first) | `GET /trade-feed?accountId=` |
| `/:username/post/:postId` | one post, read-only | `GET /post/:id` |
| `/:username/collection/:listId` | one collection as a grid; tap = enlarge card | `GET /list/:id` + `GET /list/:id/entities` |
| `/:username/listing/:listingId` | read-only listing screen | `GET /listing/:id?include=…` |
| `/:username/bid/:bidId` | read-only bid screen | `GET /bid/:id?include=…` |
| `/:username/lot/:bulkId` | read-only lot (bulk listing) screen: cover, totals, child tiles (sold ones dimmed) | `GET /bulk-listing/:id` |

Implemented as **one nested route**: `<Route path="/:username" element={<ProfileLayout />}>` with
`index` / `collections` / `trades` / `post/:postId` / … children, so the page fetch and the hero
live once in the layout and the tabs render into an `<Outlet />`. The address a creator shares
is just `nanza.app/skybanks`; the sub-paths exist so a tab or a post is linkable and Back works. Tabs are real paths (linkable,
back-button friendly), not local state.

**Why it doesn't collide with what exists.** React Router v6 ranks a static segment above a
dynamic one, so `/create`, `/get`, `/support/:article` and the nine `/<slug>/:referenceCode`
share routes always beat `/:username…` regardless of order. One subtlety: `/listing/:code` and
`/:username/collections` tie on score (one static + one dynamic each), and a tie resolves by
**declaration order** — hence "declare the profile family after the share routes". The
consequence is that a handful of words can never be a reachable username (see *Reserved
usernames*). The dormant `SmartRouter` (`/:param1/:param2`) and the commented
`/:username/listings|bids|lists` marketplace routes are **not revived** — this family replaces
them; delete the dormant files in the same change or right after.

**Static hosting needs nothing new for v1.** Marketing paths like `/create` already resolve on
the S3 + CloudFront SPA through the error-document fallback to `index.html`, and `/skybanks` takes
the same path. Today it reaches the catch-all and bounces to `/`; after this change it renders the
page. No CloudFront behavior, no AASA entry (deliberately — see *Not doing*).

### The page endpoint — `GET /user/:username/page`

Lives in `src/routers/user.ts`. Public (guest token is enough — the web app only ever has one).
`/user/:authId` is unaffected: Express matches it on exactly one segment.

```
GET /user/:username/page
  400  username fails the shape check          — no DB hit
  404  no such public profile                  — exactly ONE indexed query
  200  { profile, accountId, socialLinks, counts }
```

Handler, in order:

1. **Shape check before any query.** `^[A-Za-z0-9_]{3,20}$` (the charset mobile's
   `CompleteProfileModal` already enforces) and not a reserved word → otherwise `400`. Garbage
   and probes cost nothing.
2. **Existence — the cheap step.** One `profile.findFirst` on `username` with
   `mode: 'insensitive'`, selecting only the hero columns plus `isPublic`, `accountId`,
   `referenceCode` and `account.status`. Missing, `isPublic: false`, or a non-`ACTIVE` account →
   **`404 { errorMessage: 'Profile not found' }`**. Private is a 404, not a 403: the page must not
   disclose that a hidden profile exists (same stance as `GET /post/:id`).
3. **Fan-out — only now.** `Promise.all` over the page's small reads, all scoped to the resolved
   `accountId` and all using the **anonymous-viewer** rules the tab endpoints already apply (the
   web has no signed-in viewer):
   - `affiliateLink` (string | null) — the creator's Airbridge tracking link for AFFILIATE
     profiles, built server-side (`utils/affiliateLink.ts`); the nav's Join button opens it.
   - `socialLinks` with `socialLinkType` (id, name, displayName, logo) — the hero chips.
   - `counts.posts` — ACTIVE root posts; `BRAND` rows only when `profile.type === 'AFFILIATE'`,
     `SUPPORTED_TAG_VALUE` rows always (mirrors `GET /posts` author mode for a stranger).
   - `counts.collections` — `type: 'COLLECTION'` lists that are `isPrivate: false` or `isPrimary`
     (mirrors `GET /lists` for a non-owner) **and hold at least one entity** — so an owner with
     only the empty default collection gets no Collections tab (mobile's empty-hide rule).
   - `counts.listings` — ACTIVE, `isOffer: false`, `bulkListingId: null`, public — plus PUBLISHED
     public bulks; `counts.bids` — ACTIVE public bids (mirrors `/trade-feed` for a stranger).
   - `ensureProfileReferenceCode(profile)` — so the page can carry the app share link
     `nanza.app/profile/<code>` for "Open in Nanza".
4. **Respond with a public projection.** Never the raw Prisma row: no `qrCode`, no account
   email, no derived `deals`.

```json
{
  "profile": { "id": "…", "username": "SkyBanks", "avatar": "…", "banner": "…",
               "description": "…", "type": "AFFILIATE", "referenceCode": "U28472" },
  "accountId": "…",
  "socialLinks": [ { "id": "…", "url": "https://…", "socialLinkType": { "id": "…", "name": "x", "displayName": "X", "logo": "…" } } ],
  "counts": { "posts": 12, "collections": 2, "listings": 5, "bids": 1 }
}
```

The counts exist so the web can apply mobile's **public-profile empty-hide rule** (a tab with
nothing in it is not shown; a posts-only profile shows no pill row) **without** fetching every
tab up front. `username` comes back in its canonical stored casing so the page can render
`@SkyBanks` whatever the visitor typed.

**Cost.** One cheap query on the miss path; on the hit path one cheap query plus seven small
parallel ones — the same order of work as today's `GET /profile/:id` — and nothing until the
name is confirmed. The existing `GET /profile/username/:username` stays as it is (the dormant
marketplace screens call it); the new route doesn't touch it.

### Tabs reuse existing read endpoints — no second new API

Every tab and detail screen is served by an endpoint that already exists and is already
**guest-safe**: the authorizer stamps guest requests with `principalId: 'guest'`,
`resolveRequesterAccountId('guest')` resolves to `null`, and each endpoint then applies its
public-viewer filter.

| Need | Endpoint | Guest behaviour today |
|------|----------|-----------------------|
| Posts tab | `GET /posts?accountId=&page=&limit=` | author mode; `BRAND` only for affiliates, tag posts always; blocked-author rule n/a |
| Post page | `GET /post/:id` | 404 unless the root is viewable to an anonymous viewer |
| Collections tab | `GET /lists?type=COLLECTION&accountId=` | non-owner sees `isPrivate:false` or the primary |
| Collection grid | `GET /list/:id`, `GET /list/:id/entities?page=&limit=&search=` | private → rejected |
| Trades tab | `GET /trade-feed?accountId=&page=&limit=` | the public-visibility filters of `/listings` + `/bids`; bulks public-only |
| Listing screen | `GET /listing/:id?include=…` | `canAccessListing` → 404 when not visible |
| Bid screen | `GET /bid/:id?include=…` | public read |

So the API half of this feature is **one endpoint plus validation hardening** — the "second
request for a post" Skylar anticipated is `GET /post/:id`, which exists.

### Username hardening (API, `validation/profile.ts`)

The route only works if usernames are URL-safe and unambiguous, and today the API enforces
neither — `validateUsername` is a uniqueness check that loads **every profile's username** and
compares in JS. Three pure-validation changes, no schema change:

- **Charset + length** — server-side `^[A-Za-z0-9_]{3,20}$`, matching mobile. Spaces, dots,
  slashes and emoji can no longer be minted into a path segment.
- **Reserved usernames** — a constant list in `src/constants/reservedUsernames.ts`, rejected on
  create and update: the web app's root words that would shadow the page (`create`, `collect`,
  `trade`, `support`, `get`, `about`, `terms`), the nine share slugs (`listing`, `bid`,
  `collection`, `product`, `bulk`, `group`, `profile`, `post`, `tag`), the dormant marketplace
  words (`cart`, `build`, `sign-in`-family is hyphenated and already unmintable), and the usual
  platform words (`admin`, `api`, `app`, `nanza`, `oak`, `user`, `users`, `settings`, `help`,
  `login`, `logout`, `static`, `assets`). Hyphenated routes (`fair-market`, `privacy-policy`,
  `refund-and-return-policy`) are excluded by the charset alone. Compare case-insensitively.
- **One query instead of a table scan** — `findFirst` on `username` with
  `mode: 'insensitive'` (excluding the profile being edited). Same semantics, O(1) rows.

**Case.** The DB unique index is case-sensitive, uniqueness is already enforced
case-insensitively, and lookups here are insensitive — so `/SkyBanks` and `/skybanks` both land
on the one profile. Postgres can't use the plain btree for an `ILIKE`/`lower()` match; with the
Profile table at its current size that's a non-issue, and if it ever matters the fix is a
hand-run `CREATE UNIQUE INDEX … ON "Profile" (lower(username))` (Skylar runs migrations manually
anyway), not a Prisma change.

**Before shipping:** a one-off read query for existing usernames that fail the new rules or sit
on the reserved list, so nobody's page silently 400s. (Skylar runs it; it's SQL, not a
migration.)

### The web page (nanza-web-app)

New folder `src/app/ProfilePage/` — the dormant `UserProfile/`, `ProfileProvider` and
`SmartRouter` are the old *storefront*, built around `/profile/username/:username?include=account.lists`
(the exact expensive call this design avoids); they are not the starting point.

- **`ProfileLayout`** — fetches the page data once (`userService.getPage(username)` →
  `GET /user/:username/page`, TanStack Query keyed `['userPage', username.toLowerCase()]`,
  **no retry on 404** — the QueryClient already skips 4xx retries), renders the hero and the
  tab pills, and an `<Outlet />`. Hero = the mobile `ProfileHero` recipe in Tailwind: banner
  (edge-to-edge, bottom fade) → avatar overlapping → `@username` → bio → social-link chips
  (logo circles linking out, `rel="noopener"`), then the store buttons as the "Get Nanza" call to
  action. Document title `@username · Nanza` via the existing `useDocumentTitle`.
- **Tabs** — pills from `counts` with empty-hide; the default child is Posts (mobile's order:
  Posts → Collections → Trades). Each tab is an infinite query over its existing endpoint.
- **Cards** — reuse the share pages' *data mappers* (`mapEntityCard`, the post body /
  attachment mappers, `mapListEntries`, `mapBulkListings`) and primitives (`Avatar`,
  `buildImageUrl`, `formatPriceAsDollar`), but render dark-theme card variants (see *Theme*):
  a listing/bid row, a lot carousel, a post card, an entity tile. New: a collection-card tile
  for the Collections tab (mosaic, name, count) and a lightbox that enlarges one entity tile.
- **Detail screens** are the share cards without the share chrome: post page, listing page, bid
  page — every button that would write (buy, offer, like, reply, follow) is simply absent. Links
  inside them stay inside `/:username/…`.
- **Not-found redirects home** (Skylar, 2026-10-07) — a 404 or a 400 from the page call
  `Navigate`s to `/` with `replace`, exactly like the catch-all does for every other unknown path
  today. One rule for "this address isn't a thing" across the whole site; no bespoke empty
  screen. While the page call is in flight the layout renders nothing but the dark ground, so a
  bad name is a brief blank then the landing page, never a flash of a half-built profile.
- **No store redirect for phone visitors.** `ShareDetail` bounces app-less phones straight to
  the store because its page is a fallback; this page *is* the destination — it's what a creator
  puts in a bio and what a phone opens. The way into the app is the **Join the chat** button in the nav
  (2026-10-16, replaced the hero store buttons): white pill far right; the nav on this route shows
  only the logo and Join (`Navigation minimal` + `action` slot). It opens the creator's Airbridge affiliate
  link on any device when `page.affiliateLink` is set (AFFILIATE profiles — click and install
  attributed to the creator); otherwise the platform store on a phone, `/get` on desktop. The desktop QR card scans to the
  same affiliate link for AFFILIATE profiles (`QrOverlay url` prop, live `qrcode.react` code);
  other profiles keep the general code. See
  [[airbridge-affiliate-attribution|Airbridge affiliate attribution]] → "Web Join button".
- **Theme — the dark marketing theme** (Skylar, 2026-10-07). The layout calls `usePublicSite()`
  so `.public-site` and the `page` / `ink` / `surface` / `hairline` tokens apply, the same ground
  as the landing page and the content pages: a creator's bio link should feel like Nanza's own
  site, not like a share fallback. Consequence: the share cards are styled light and can't be
  dropped in unchanged — the Trades rows, post cards and collection tiles get dark-theme variants
  built on the same mappers (`mapEntityCard`, the post body/attachment mappers) rather than the
  light components themselves. The share pages keep their light shell untouched.

### Not doing (deliberately), and what's phase 2

- **No AASA / App Links entry for `/*`.** Making root paths universal links would hijack
  `/create`, `/support` and every marketing page into the app. Root username URLs open on the
  web, everywhere; the app is reached by the store buttons and the `/profile/<code>` link.
- **No OG tags for `/<username>` in v1.** The edge injector (`ogEdgeHandler`) only runs on the
  nine slug behaviors. Giving usernames an unfurl means: a `GET /user/:username/meta` route (the
  `buildOgMeta` Profile case + the existing by-code `og.png`), the edge regex learning a
  `^/[A-Za-z0-9_]{3,20}/?$` shape minus the reserved words, and — the real cost — associating the
  version-pinned edge with the **default** CloudFront behavior on both distributions so it runs on
  every HTML miss. Worth doing (link-in-bio previews matter) but as its own slice, after the page
  exists; until then a shared `nanza.app/skybanks` unfurls with the site-wide default tags.
- **Mobile unchanged.** Later the own-profile share sheet can offer `nanza.app/<username>`
  alongside the `/profile/<code>` link — `buildShareUrl` needs a non-code variant for that.
- **No types-package bump.** The page response composes existing DTOs; the web types it locally
  (as mobile does for `TradeFeedItem`).

## Decisions (Skylar, 2026-10-07)

- **Not-found → redirect home**, same as the catch-all. (Rendering a message in place was
  proposed and rejected.)
- **Dark marketing theme**, not the light share-card shell.
- Still open: the page call returns identity + counts and the tabs fetch lazily (proposed);
  the alternative is inlining each tab's first page into that response.

## Build order

1. **API** — ~~`reservedUsernames.ts`; `validateUsername` (charset, reserved, single insensitive
   query); `GET /user/:username/page` in `user.ts` with `@openapi` JSDoc; lint + `tsc`~~ **done
   2026-10-07.** Still to do: run the username audit query (below). Deploy dev.

   ```sql
   -- Existing usernames that would now fail validation (bad shape or reserved word)
   SELECT id, username FROM "Profile"
   WHERE username IS NOT NULL
     AND (username !~ '^[A-Za-z0-9_]{3,20}$'
       OR lower(username) IN ('create','collect','trade','support','get','about','terms',
         'listing','bid','collection','product','bulk','group','profile','post','tag',
         'cart','build','admin','api','app','nanza','oak','user','users','settings','help',
         'login','logout','static','assets'));
   ```
2. **Web** — ~~routes (nested, after share routes); `userService.getPage`; `ProfilePage/`
   layout + hero + tabs; post / collection / listing / bid / lot screens; not-found~~ **done
   2026-10-08.** Dormant storefront files (`UserProfile/`, `ProfileProvider`, `SmartRouter`,
   `UserListings|Bids|Lists|Saved`) left in place — delete in a follow-up once this ships.

   Notes from the build:
   - The post page shows the post only (every content item, GROUP galleries flattened,
     embedded listing/bid/entity/collection references as slim tiles, tombstone when the
     target is gone). **Replies are not fetched yet** — a follow-up (`GET /posts?parentId=`).
   - Collections tab requests `/lists?type=COLLECTION&accountId=&include=entityList.entity`
     (the API caps the embedded entities at 20) and builds the four-tile mosaic from them;
     lists with nothing in them are filtered client-side too, so pill and grid agree.
   - Trades rows carry the mobile feed-card accents (teal For sale, gold Wanted, violet Lot).
   - Detail screens swap the full hero for a compact `‹ @username` row that is the way back.
3. **Phase 2** — username OG/meta + default-behavior edge; mobile share-sheet link;
   `lower(username)` index if lookups ever show up in slow logs.

## Related

- [[profile-types|Profile Types]] — `AFFILIATE` decides whether `BRAND` posts are public here.
- [[affiliate-social-links|Social links]] — the chips in the hero.
- [[account-feeds|Account feeds]] — `/trade-feed`, the Trades tab's source.
- [[posts|Posts]] — author mode of `GET /posts` and the single-post read.
- [[og-sharing|OG / sharing]] — the edge injector phase 2 would extend.
- [[../../nanza-web-app/solution-designs/sharing|nanza-web-app Sharing]] — the share cards this
  page reuses; [[../../nanza-web-app/solution-designs/share-route-migration|Share Route Migration]]
  — the nine slug routes this family sits beside.
- [[../../nanza-mobile/solution-designs/profile|nanza-mobile Profile]] — the screen being mirrored
  (hero recipe, tab order, empty-hide rule).
