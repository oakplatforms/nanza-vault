---
tags: [nanza-api, nanza-mobile, solution-design, plan, auth, guest, trade, navigation]
---

# Guest Browsing: the nav for everyone, plus the Trades market feed

**Status:** **plan, written 2026-10-23 (Skylar, voice). All four questions answered the same day. API `GET /trades` built 2026-10-23 (uncommitted, tsc + lint clean, not deployed). Mobile built the same day (uncommitted, tsc clean, lint clean on touched files, NOT run on device). No migration, no types bump. Next: deploy API to dev, then device QA.**

Spans nanza-mobile (guest nav, Trade/Groups/Inbox guest states) and nanza-api (a new
market-wide `GET /trades` feed). This reverses the "signed out = no nav" call in
[[../../nanza-mobile/solution-designs/open-posting-and-nav-swap#Sixth pass — the signed-out nav (2026-08-22, Skylar)|nav-swap § Sixth pass]].
When it lands, fold the mobile half into that design and
[[../../nanza-mobile/solution-designs/trade-screen|Trade tab screen]], fold the API half into a
"Trades feed" section of [[offer-engine]] (or a new `trades-feed.md`), then retire this file.

## The ask (Skylar, 2026-10-23, voice)

- Today a guest lands on Home with a **"Join the chat" input and a search circle** at the
  bottom instead of the tab bar. Remove that.
- Guests get **the normal bottom navigation**:
  - **Home**: works as it does today.
  - **Trade**: guests see it as a **read-only grid** of what's on the market. Phase 1 is one plain
    GET. The end goal is a **trades endpoint** that merges the latest **bids, listings and lots**
    into one request, with **filters** (most recently added, price range, card type, …).
  - **Groups**: stays as it is, but read-only. Tapping **Join** does what Home does today and
    opens the sign-up prompt.
  - **Inbox**: the **only members-only tab**. First asked for as a tap-to-sign-up gate; after
    seeing it built, Skylar chose to **hide the tab for guests** instead (same day).
- Everything else works as it does now.

## Today (what changes)

| Area | Today | Where |
|---|---|---|
| Bottom bar, guest | `FloatingTabBar` returns early on `!isRegistered` and renders `JoinChatFooter placement="fixed"` with a trailing Search circle. No tabs. | `components/global/FloatingTabBar/index.tsx:215-259` |
| Header search | `TopHeader.hideSearch` defaults to `true` for everyone since 2026-09-15; guests reached search through the footer's trailing circle. | `TopHeader` |
| Trade tab | `if (!viewerAccountId) return null`. Pills show *your* bids (`GET /bids?accountId`) and *your* listings (`GET /sell-feed`). | `screens/trade/TradeScreen/index.tsx:465-471` |
| Groups tab | `if (!isRegistered) return null`. `handleJoin` silently no-ops without `accountId`. | `screens/groups/GroupsListScreen/index.tsx:102-108, 130-138` |
| Group detail | Guests can reach it from the Home banner, but get **no Join pill** (`if (!isRegistered) return null`). | `GroupDetailScreen/index.tsx:366-369` |
| Inbox tab | `if (!viewerAccountId) return null`. | `screens/messages/MessagesScreen/index.tsx:108-115` |
| Sign-up prompt | `AuthModal` reads "Ready to join in?", with **Keep browsing** / **Create account** and an "Already have an account?" link to `LoginModal`. | `components/modals/AuthModal` |

The guest API path already works: `fetchData` falls back to the guest token from
`GET /user/guest-token`, so every read endpoint already serves guests through the authorizer.

## Mobile design

### 1. One nav for everyone

- **Delete the guest early return** in `FloatingTabBar`. Guests get the same pill, tabs, search
  circle, jelly and glow as signed-in users. The guest `JoinChatFooter` mount and its trailing
  search circle go with it.
- **Search is the bottom circle for everyone.** The header already hides it; the guest footer's
  trailing search circle goes with the footer, and the nav circle takes over.
- **No post FAB for guests.** `showPostFab` keeps its `isRegistered` check. The point of this
  change is that guests stop seeing a chat CTA in the bottom band.
- **No Inbox badge for guests.** `useInboxCount` is already keyed on the account, so guests get
  zero. Just confirm it renders no dot.
- **Auth-settled guard.** The nav gates on `isRegistered`, which is `false` while a signed-in
  user's `currentUser` is still loading. With the same nav for both states, the only visible
  effect is the Inbox gate. A tap during hydration must not open the sign-up prompt for a
  signed-in user, so the Inbox gate uses the existing settled check
  (`isSignedIn === false || currentUser !== undefined`) and ignores taps until auth has settled.

### 2. The guest pill: Search where Inbox was

- A settled guest's pill is **Home · Groups · Trade · Search** (Skylar, 2026-10-23, third pass):
  Inbox is members-only and simply isn't there, and its slot carries a **Search tab** — the
  `search-2` glyph with a "Search" label, a `TabItem` like the others, opening the Search screen.
  The **search circle beside the pill is dropped for guests** so search isn't offered twice. The
  pill keeps the **member width** (tab width is always cluster minus circle ÷ 4) and the cluster
  shrinks to it, so the guest pill sits centred rather than stretching over the circle's slot
  (a stretched version read "extended" — Skylar, same day). Members keep four tabs + circle.
  `FloatingTabBar` renders from `visibleTabs` (`GUEST_TABS` or `MAIN_TABS`); the Search tab has
  no navigator route, so it is never the focused tab and the jelly never lands on it.
- **Sign-in and sign-up always land on Home** (`navigation/finishAuthNavigation.ts`:
  `popTo('Main', { screen: 'Home', params: { screen: 'HomeScreen' } })`), from all three
  completion points — `LoginModal`, the sign-up confirmation step, `SocialAuthButtons`. The guest
  bar and the member bar differ, so the session flip is a nav change, and Home is the one tab both
  have. The old `returnTo` destination is no longer consulted: `useConfirmationCodeStep` and
  `SocialAuthButtons` lost the param; `AuthModal` / `LoginModal` / the `SignUp` route still carry
  it (share cards and `useShareActions` set it), which is now dead plumbing — remove it if
  always-Home sticks.
- **Why hidden, not gated (Skylar, 2026-10-23):** a four-tab version whose Inbox tap opened
  `AuthModal` was built first, looked at, and removed the same day — hidden read cleaner; then a
  three-tab pill, then the Search tab in the slot. Nothing of the gate remains in the bar.
- The guest state is read only once auth has settled, so a member never sees the pill reflow on
  cold start; a guest may see four tabs collapse to three during the initial auth check.
- `MessagesScreen` keeps its `return null` guard; it never mounts for a guest.
- `MessagesScreen` keeps its `return null` guard. Everything below it narrows on
  `viewerAccountId`. It is now unreachable for guests, not removed.

### 3. Trade tab: the market grid for guests

- **Guests** (`!viewerAccountId` once auth is settled) render a new `TradeMarketGrid` in place of
  the `return null`: a **2-column grid**, brand-scoped to `selectedBrandId`, paging on scroll, fed
  by `GET /trades` (below).
  - Tiles: one tile recipe per kind (listing / bid / lot), each with image, title, price and a
    small kind label ("Listing", "Bid", "Lot"). Reuse the existing tile/card visuals where they
    fit: the entity hero thumb for bids, `BulkSaleCard` imagery for lots.
  - Tap → the existing detail screens (`ListingScreen`, `BidScreen`, bulk detail). Guests can
    already open these, and their buy/sell/offer actions already open `AuthModal`.
  - No pills, no search input, no Create-lot CTA, no scan shortcut for guests. Those all act on
    *your own* market.
- **Signed-in users get three pills: `Nanza` · `My Bids` · `My Listings`** (answered 2026-10-23).
  `Nanza` is the same market grid, with the viewer's own items excluded, and it is the default
  pill. `My Bids` / `My Listings` are today's Bids / Listings pills, unchanged apart from the
  label. `initialTab` accepts `'Nanza' | 'Bids' | 'Listings'`, so existing `initialTab: 'Listings'`
  callers still land on their own listings. The search input, scan shortcut and Create-lot CTA show
  only on the two `My` pills. Guests see the grid with no pill row.
- React Query key: `['Trades', brandId, filters]`, staleTime in the discovery tier from
  [[../../nanza-mobile/solution-designs/data-freshness|Data Freshness]] (lazy, refetch on visit).

### 4. Groups tab: read-only for guests

- **Drop `if (!isRegistered) return null`** in `GroupsListScreen`. The list query already runs
  for everyone.
- **`handleJoin` for guests opens `AuthModal`** instead of silently returning, with
  `returnTo` → the Groups tab.
- **Group detail gets the same treatment.** Show the Join pill to guests and route the tap to
  `AuthModal`. This replaces the current "no pill at all" so the two entry points behave the same.
- "Same as Home" means: what a guest gets today on Home when they hit a gated action, which is
  this `AuthModal` sheet.
- **Create-group entry** (if the list shows one) is hidden for guests.

### 5. Home

Unchanged, apart from the bottom bar. Search moves from the header to the circle. The groups shelf
`GroupCircleCarousel` stays dimmed and inert for guests. The one join prompt it was deferring to
(the bottom chat footer) is gone, and the Groups tab now carries that prompt.

## API design: `GET /trades`

### Phase 1: the market feed (ship with the mobile change)

`GET /trades?brandId=&page=&limit=` returns one paginated, **newest-first** list of everything on
the market for a brand: ACTIVE single **listings**, PUBLISHED **lots** and ACTIVE **bids**.

- **Same response shape as `/trade-feed`:**
  `{ data: ({ kind: 'listing', listing } | { kind: 'bulk', bulk } | { kind: 'bid', bid })[], page, limit, total }`.
  Mobile already has this type for the profile Trades rail, so this needs no new DTO.
- **Reuse `buildAccountFeedPage`** (`routers/bulkListing.ts`). It already takes per-kind `where`
  clauses, merge-sorts by `createdAt`, slices the page and hydrates only that slice with comment
  counts. **Built 2026-10-23:** `/trades` sits next to `/trade-feed` in `bulkListing.ts` and calls
  the builder in place. The extraction to `utils/` was skipped, because the builder shares that
  file's include constants and moving them would be pure churn. Bulks also get the
  public-author rule (`account.profile` public or null) that `/feed` applies.
- **Filters (the per-kind `where`s):**
  - Listing: `status: 'ACTIVE'`, `bulkListingId: null`, `isOffer: false`, `quantity > 0`,
    `entity.brandId = brandId`
  - Bid: `status: 'ACTIVE'`, `entity.brandId = brandId`
  - Lot: `status: 'PUBLISHED'`, with at least one ACTIVE, `quantity > 0` child whose entity is in
    the brand (the same transitive-brand rule `/feed` uses)
  - Visibility: AND with `buildListingVisibilityFilter` / `buildPublicAuthorListingFilter`,
    `buildBidVisibilityFilter` / `buildPublicAuthorBidFilter`, and the bulk `isPublic` / group rule
    from `/trade-feed`. For a guest (`requesterAccountId = null`) these collapse to *public items
    by public authors*. For a signed-in viewer they widen to their groups. Private and group items
    never reach a guest.
  - Signed-in viewers: exclude their own items (`accountId: { not: viewer }`), the same as `/feed`.
    This matters only if signed-in users ever see the grid.
- **Route:** `GET /trades` is currently a 410 stub in `routers/trade.ts` for pre-removal builds.
  Repurpose that path, delete just the `/trades` stub, and keep the other `/trade/*` stubs. See
  open question 1.
- Validation (`brandId` required, `limit ≤ 100`) is inline in the route, matching its `/sell-feed`
  and `/trade-feed` siblings. Error tag `GET_TRADES_ERROR`.

### Phase 2: filters (later, same endpoint)

Add query params. Each one narrows the per-kind `where`, so the merge stays unchanged:

| Param | Meaning | Notes |
|---|---|---|
| `kind=listing\|bid\|lot` (repeatable) | which kinds to include | Omitting a kind skips its count + head query entirely. |
| `sort=recent\|price_asc\|price_desc` | order | `recent` is the default. Price sorts select `price` into the head rows and merge-sort on it instead of `createdAt`. |
| `minPrice`, `maxPrice` | price band | Listings and bids filter on `price`. **Lots have no stored price**: `totalPrice` is derived at runtime. Either filter lots on the sum of their ACTIVE children in raw SQL, or leave lots out when a price band is set. Recommend the latter for v1. |
| `tagValueId` (repeatable) | "type of card" | `entity.entityTags.some({ supportedTagValueId in … })`, the same tag taxonomy the search filter uses. Lots match when any child matches. |
| `conditionId` | condition | Listing `conditionId` / bid `conditions.some`. Lots use their children. |

Mobile gets a filter pill row above the grid, reusing the search-filter sheet pieces from
[[../../nanza-mobile/solution-designs/tag-taxonomy|Tag Taxonomy]] (the trending + tag search pills) where they fit.

### Performance notes

- The builder over-fetches `(page+1) × limit` heads per kind, so deep pages cost more. That's fine
  for an infinite grid at `limit` 20. If it ever matters, move to a `(createdAt, id)` cursor.
- `Listing` and `Bid` have no index that leads with `createdAt`. A brand-wide newest-first scan
  will want `@@index([status, createdAt])` on both. That's an optional migration, which Skylar
  runs. Measure on dev data first.

## Build order

1. **API:** add `GET /trades` (phase 1) on the existing builder. No schema change, no types bump
   (the response reuses the trade-feed item type). Deploy to dev.
2. **Mobile:** FloatingTabBar guest swap removed + Inbox gate + Groups list/detail guest join →
   `AuthModal` + `TradeMarketGrid` on `GET /trades`.
3. Device QA as a guest and as a signed-in user, including cold start while signed in (check for
   the hydration flash and that Inbox doesn't misfire).
4. Later: phase 2 filters, plus indexes if needed.

Rough size: API half a day, mobile 1 to 1.5 days.

## Implementation record (2026-10-23)

- **API:** `GET /trades` is in `routers/bulkListing.ts` next to `/trade-feed`. The `/trades` 410 stub was removed from `routers/trade.ts`.
- **Mobile:**
  - `FloatingTabBar`: the guest `JoinChatFooter` branch is deleted; settled guests get `GUEST_TABS` = Home · Groups · Trade · Search and no circle. The tap-to-prompt variant was built and removed the same day.
  - `navigation/finishAuthNavigation.ts`: every auth completion → Home. `returnTo` dropped from `useConfirmationCodeStep` and `SocialAuthButtons`; the Groups `AuthModal`s pass none. The navigator stays untouched apart from a comment. The header search was already off for everyone (2026-09-15), so `TopHeader` needed no change.
  - `TradeScreen`: pills `Nanza` · `My Bids` · `My Listings`, with `TAB_LABELS` over the unchanged `'Bids' | 'Listings'` keys. The market grid is the new `TradeScreen/TradeMarketGrid.tsx`: the `ProfileSeeAllScreen` trade recipe (`CardGrid` + `ListingThumb` / `BidThumb` / `BulkThumb`) in a gesture-handler `ScrollView` with end-reached paging, fed by `screens/trade/data/fetchTrades.ts` (`useTrades`, key `['Trades', brandId, viewer|'guest']`, `30_SECS`).
  - `GroupsListScreen`: the guest guard is gone, Join opens `AuthModal` for guests, and "New group" is hidden for guests.
  - `GroupDetailScreen`: a settled guest sees an emphasis Join pill that opens `AuthModal`.
- **Join prompts land on Home like every other sign-in** (the earlier return-to-Groups was
  superseded by always-Home the same day).
- **Grid gutter:** `TradeMarketGrid` passes `inset="page"` so the grid's left edge meets the pill
  row's page gutter.
- **Review (Fable, 2026-10-23):** architecture confirmed as designed; the three fixes above
  (return-to, grid inset, this doc's structure) were the only changes.
- **QA watch:** three pills at `paddingHorizontal: lg` may be tight on small phones. Check "My
  Listings" on an SE-width device. Also: signing out while on the Inbox tab leaves the navigator
  on Activity with no pill for it (jelly fades, `MessagesScreen` renders null) — check what
  sign-out does to the tab state.

## Decisions (answered 2026-10-23)

1. ~~**Endpoint path.**~~ **Answered 2026-10-23: use `GET /trades`.** It isn't in use, so the 410
   stub for `/trades` is deleted and the other `/trade/*` stubs stay.
2. **Answered:** signed-in users get the grid too, as the pills `Nanza` · `My Bids` · `My Listings`.
3. **Answered:** the post FAB stays hidden for guests.
4. **Answered:** all three kinds are mixed in the grid from phase 1.

## Related

- [[../../nanza-mobile/solution-designs/open-posting-and-nav-swap|Open posting & nav swap]]: the
  signed-out nav this reverses
- [[../../nanza-mobile/solution-designs/auth-screens|Auth screens]]: the `AuthModal`
  interstitial every gate opens
- [[../../nanza-mobile/solution-designs/trade-screen|Trade tab screen]]
- [[../../nanza-mobile/solution-designs/groups|Groups]]
- [[offer-engine]]: what a guest's tap on a grid tile eventually leads to
