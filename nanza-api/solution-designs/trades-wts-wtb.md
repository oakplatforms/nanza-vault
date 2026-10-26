---
tags: [nanza-api, nanza-mobile, solution-design, plan, trade, entity, navigation]
---

# Trades: WTS / WTB — the entity Trades tab and the Trade screen

**Status:** **plan, written 2026-10-25 (Skylar, voice); the five decisions answered the same day. API half built 2026-10-25 (uncommitted, tsc + lint clean, not deployed). Mobile half built 2026-10-25 (uncommitted, tsc + lint clean, NOT run on device). Next: deploy API to dev, device QA.** Spans nanza-api (one trades endpoint grows an entity filter and a kind filter) and nanza-mobile (entity Trades tab, a new WTS / WTB create screen, the Trade tab's pill row). Skylar will keep adding to this file; My Trades gets its own design later.**

When it lands, fold the API half into a "Trades feed" section of [[offer-engine]] (or the
`trades-feed.md` that [[guest-browsing]] already points at), the mobile half into
[[../../nanza-mobile/solution-designs/entity-screen|Entity Screen]] and
[[../../nanza-mobile/solution-designs/trade-screen|Trade tab screen]], then retire this file.

## Vocabulary

Two words carry this design, and they go into the code as written:

| Term | Means | Model |
|---|---|---|
| **WTS** — *want to sell* | someone is selling a card | `Listing` (and a `BulkListing`, a lot of listings) |
| **WTB** — *want to buy* | someone is looking for a card | `Bid` |

A **trade** is either. Bids and listings stay separate tables; the API merges them into one
list and the UI shows them as one thing. On mobile `TradeKind = 'WTS' | 'WTB'` is the type name
and the tab keys. **The API keeps its own nouns** (Skylar, 2026-10-25): the `kind` filter takes
the feed item union's values, `listing` | `bid` | `bulk`, so WTS maps to `kind=listing` and WTB to
`kind=bid` at the call site.

## The ask (Skylar, 2026-10-25, voice)

1. **Reimagine the Trades tab inside the entity screen**, modeled on the Market tab (a headline
   block, then content): a *start a trade* prompt with small buttons under it, a white **Add
   new** pill, and under that a **two-column grid** of the card's existing trades (bids and
   listings mixed). No carousel.
2. **One endpoint for a card's trades.** `GET /trades` already merges bids, listings and lots for
   a brand. Give it an `entityId` filter so one call returns the latest WTS and WTB for a card.
   The entity screen's two list calls (`/listings?entityId`, `/bids?entityId`) go away, and so
   does the "you already have a bid / listing on this card" button state and its two lookups.
3. **Create bid and create listing become one screen.** Their forms are already components;
   wrap them in one new screen with the same capsule tab row the entity screen uses, two tabs:
   **WTS** · **WTB**. No *All* here. Opening it from an entity prefills the card, so a trade is
   one step: pick the tab, fill the price, create.
4. **The Trade tab in the bottom nav** uses the same tab row with **All** · **WTS** · **WTB**,
   all three fed by `GET /trades`: *All* is everything on the market, *WTS* filters to listings,
   *WTB* to bids. More filters later. Two-column grid for all three for now.
5. **My Trades** (today's *My Bids* / *My Listings* pills) will get a different design. Out of
   scope here, and **its components stay in the code**, unmounted: the lists, the search, the
   add-a-listing / add-a-bid rows. They come back as My Trades + *Add new trade*, two buttons
   at the bottom of the Trade tab, in a later step.
6. **Lots are hidden this release.** The backend keeps running; every way into a lot on mobile is
   hidden (Create lot, add to lot, lot tiles). Lots come back later and will live under WTS.

## Today (what changes)

| Area | Today | Where |
|---|---|---|
| Market feed | `GET /trades?brandId=&page=&limit=`: ACTIVE listings + PUBLISHED lots + ACTIVE bids for a brand, newest-first, viewer-filtered, **the viewer's own items excluded**. Built on `buildAccountFeedPage` with the `/trade-feed` item union `{ kind: 'listing' \| 'bulk' \| 'bid', … }`. | `nanza-api/src/routers/bulkListing.ts:512-579`, [[account-feeds]] |
| Entity Trades tab | Four requests once the tab opens: `/listings?entityId&status=ACTIVE`, `/bids?entityId&status=ACTIVE` (merged client-side into `feedItems`, newest-first), plus `/listings?entityId&accountId` and `/bids?entityId&accountId` for the viewer's own trade, which only feed the CTA labels (*Listed at $X* / *Wanted at $X*). Two xxl CTAs (**Place bid** / **Sell**) above a two-column grid of `ListingThumb` / `BidThumb`; the empty state is `MarketEmptyBlock` with the same two CTAs. | `nanza-mobile/src/screens/entity/EntityScreen/index.tsx:167-209, 393-396, 650-700` |
| Create screens | `CreateBidScreen` (139 lines) and `CreateListingScreen` (275 lines) each render `Header` + `ScrollView` + `DetailActionBar` around a form component: `CreateBid` / `EditBid`, `CreateListing` / `EditListing` (forwardRef handles: `submit`, `deleteNow`, `requestShare`). The screen picks create vs edit by looking up the viewer's existing trade on that card. | `screens/bids/CreateBidScreen`, `screens/listings/CreateListingScreen`, `components/bids/*`, `components/listings/*` |
| Who opens them | `EntityScreen` (both CTAs), `TradeListItem` (search rows on the Trade tab), `ActionModal/Feed` (pencil on your own bid / listing), `BidScreen`, `ListingScreen`, `BulkTab` (add a listing to a lot). Route params: `{ productEntity?, initialImageUri?, bulkListingId? }`. | `types/navigation.ts:44-54` |
| Trade tab | Three pills `Nanza` · `My Bids` · `My Listings` (keys `'Nanza' \| 'Bids' \| 'Listings'`); the Nanza pill is `TradeMarketGrid` over `useTrades(brandId, viewerAccountId)`; the other two are the viewer's own lists plus search. Guests get the grid alone. | `screens/trade/TradeScreen/index.tsx`, `TradeMarketGrid.tsx`, `data/fetchTrades.ts` |
| Tab row | `EntityTabs`: one 44pt capsule of equal-width segments, `EntityTabId = 'details' \| 'market' \| 'trades' \| 'chat'`. | `components/entity/EntityTabs` |

## API design

### `GET /trades` grows two filters

```
GET /trades?brandId=&entityId=&kind=&page=&limit=
```

| Param | Required | Effect |
|---|---|---|
| `brandId` | one scope required | brand-wide market (today's behaviour) |
| `entityId` | one scope required | one card's trades: `entityId` on listings and bids. With `entityId` set, `brandId` is ignored (the card already belongs to a brand). |
| `accountId` | one scope required | one account's trades (the profile's Trades rail, 2026-10-25). The requester's own account skips the viewer filters, as `/trade-feed` does for its owner. |
| `groupId` | one scope required | trades targeted to a group (the group's rail, 2026-10-25). A PRIVATE group requires the requester to be its moderator or an ACTIVE member. |

Scopes AND together when several are sent. **The market is public-only (Skylar, 2026-10-25,
fourth pass):** with no `accountId` or `groupId`, listings and bids need `isPublic: true`, so a
group-only trade never reaches the Trade screen or a card's Trades tab, not even for its owner
or a fellow member. It shows inside its group (`groupId`) and on its owner's profile rail
(`accountId`, viewer-filtered).
| `kind` | no | `listing` (the app's WTS), `bid` (WTB) or `bulk`. Absent = all. `bulk` is accepted and returns an empty page until lots return. |

Everything else stays: same item union, same `{ data, page, limit, total }` page, same
visibility rules (`buildListingVisibilityFilter` + public-author filters for listings and bids),
same `buildAccountFeedPage` builder. The builder's three `where`s are all optional now; a kind
whose `where` is omitted is skipped (no count, no heads), which is how `/sell-feed` already left
bids out and how `/trades` now leaves lots out or honours `kind`.

**Bulked children.** On a card (`entityId`) the listing author filter runs with
`includeBulked`, so a lot's children show as individual listings, exactly as `GET /listings`
does for an entity. Brand-wide they stay inside their lot as before.

**Own items are in (Skylar, 2026-10-25).** The `notOwn` exclusion comes off `GET /trades`
entirely: brand-wide and per card, the viewer's own listings and bids appear in the grid like
anyone else's, and tapping one opens its detail screen, where edit already lives. On the entity
tab that is what lets the four-request CTA-label dance go: the grid *is* the "you already have
one" signal. (The home feed keeps its own exclusion; this is the market, where you want to see
your trade sitting among the others.)

**Lots are out of the feed this release.** The handler passes no `bulkWhere` to the builder,
so no lot tile reaches a grid. The old `/trades` lot `where` (PUBLISHED, an available child in
the brand, `isPublic OR viewer's groups`, public author) is in git history and described in the
handler comment; the `BulkThumb` branch stays in the mobile grid. Bringing lots back is a
`bulkWhere` plus a `kind=bulk` branch on the API and nothing on mobile.

### What the mobile app stops calling

- `GET /listings?entityId&status=ACTIVE` and `GET /bids?entityId&status=ACTIVE` from the entity
  screen. The `/listings` and `/bids` routes themselves stay (detail screens, profile, the
  create screens' edit lookup, the bid-offer flow, lots).
- `GET /listings?entityId&accountId` and `GET /bids?entityId&accountId` from the entity screen.
  They remain in use by `CreateBidScreen` / `CreateListingScreen` to decide create vs edit
  until the new screen replaces them, and by `ActionModal/Feed`.

**Old app builds keep working (Skylar, 2026-10-25).** Nothing here deprecates or changes an
existing route: `GET /listings`, `GET /bids`, `POST|PUT|DELETE /listing`, `POST|PUT|DELETE /bid`
and the bulk routes stay as they are, so a shipped build that still creates listings and bids
through the old screens is unaffected. `GET /trades` itself is only in the uncommitted
2026-10-23 build (nothing in the store calls it), so its two behaviour changes (own items in,
lots out) break no shipped client. The 410 `/trade/*` stubs are untouched.

No schema change. No types bump (the response reuses the client-typed `TradeFeedItem`).
Optional later: `@@index([entityId, status, createdAt])` on `Listing` and `Bid` if the
per-card scan shows up on dev data; Skylar runs migrations.

## Mobile design

### 1. Entity Trades tab

Layout mirrors the Market tab's two-part structure (headline block, then content):

```
┌──────────────────────────────────────────┐
│ Start a trade                            │   ← title, marketPriceLabel weight
│ Buy or sell this card on Nanza.          │   ← one line of body
│ [ Add new ]                              │   ← small white pill (the market range-pill
│                                          │     size, not the xxl CTA)
├──────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐                │   ← two-column grid, newest first
│ │ WTS $12  │ │ WTB $9   │                │     ListingThumb / BidThumb / BulkThumb
│ └──────────┘ └──────────┘                │     (the TradeMarketGrid recipe)
│ ┌──────────┐                             │
│ │ WTS $11  │                             │
│ └──────────┘                             │
└──────────────────────────────────────────┘
```

- **Data:** one `useTrades({ entityId })` call (the existing hook grows an options object:
  `{ brandId?, entityId?, kind? }` and keys on all three). Gated on `tradeReady` exactly as the
  four requests are today, so nothing fires until the tab opens.
- **Headline block:** title + one-line body + the **Add new** pill. The pill opens the new
  Trade create screen with `{ productEntity: entity }`. Guests get the same pill; it opens
  `AuthModal` (the existing `useAuthGate` pattern on the detail screens).
- **Grid:** the `CardGrid` + thumb recipe from `TradeMarketGrid` (listing and bid tiles; the lot
  branch stays but gets no items this release), extracted into a shared `TradesGrid` component (items in, thumbs out, end-reached paging) so the entity tab and the
  Trade screen render the same thing. The entity tab passes the page gutter the Market tab uses.
- **Empty:** headline block alone, body reads *Be the first to make an offer*. No second copy of
  the pill inside an empty block.
- **Removed:** the two xxl CTAs, `bidCtaLabel` / `sellCtaLabel`, `handleBidPress` /
  `handleSellPress`, the four hooks' imports, `FeedItem` and the client-side merge,
  `paneTradeCtas*` and `entityModalTradeButtonWrapper` styles once nothing else uses them.

### 2. The Trade create screen (`CreateTrade`)

One new route, `CreateTrade: { productEntity?, kind?: TradeKind, initialImageUri?, bulkListingId? }`,
replacing `CreateBid` and `CreateListing` in the navigator.

```
┌──────────────────────────────────────────┐
│ ˅                               🗑  ⤴    │   ← Header, floating, chevron-down back
│ ┌──────────────┬──────────────┐          │
│ │     WTS      │     WTB      │          │   ← EntityTabs capsule, two segments
│ └──────────────┴──────────────┘          │
│                                          │
│   (CreateListing | EditListing form)     │   ← WTS
│   (CreateBid     | EditBid     form)     │   ← WTB
│                                          │
├──────────────────────────────────────────┤
│ [        Create listing / Place bid    ] │   ← DetailActionBar, stickToKeyboard
└──────────────────────────────────────────┘
```

- **Tabs:** `EntityTabs` (its `EntityTabId` union widens to a generic string key, or the
  component is generalised to `TabsCapsule<T extends string>`; the four entity ids keep working).
  Default tab: `kind` param if given, else **WTS**. Switching tabs swaps the form; unsaved input
  on the other form is dropped (the forms are uncontrolled behind refs today, and a trade is a
  few fields).
- **Forms unchanged.** `CreateBid` / `EditBid` / `CreateListing` / `EditListing` mount exactly as
  the two screens mount them now, with the same `useFormBar` wiring (busy, canSubmit, held
  checkmark close). Each tab resolves create vs edit with the existing per-card lookup
  (`useFetchUserProductBid` / `useFetchUserProductListing`), so landing on a card you already
  listed shows **Update listing** on WTS and **Place bid** on WTB. The trash and share circles
  follow the active tab's edit state. The camera's `initialImageUri` rides on the
  WTS tab. The lot params (`bulkListingId`, *Add to lot*) are carried through so `BulkTab` keeps
  compiling, but no mounted surface reaches them this release; when lots return, a set
  `bulkListingId` hides the WTB tab (a lot has no bids).
- **From an entity:** `productEntity` is set, so the card is fixed and the form is the price,
  quantity and condition fields it already has. That is the one-step trade.
- **Without a card** (future: the Trade screen's own Add new): the forms' existing card search
  shows, as `CreateBid` / `CreateListing` already do when `productEntity` is undefined.
- **Callers move:** `TradeListItem` (`variant === 'buy'` → `kind: 'WTB'`), `ActionModal/Feed`
  (`type === 'bid'` → `'WTB'`), `BidScreen`, `ListingScreen`, `BulkTab` (→ `kind: 'WTS'`,
  unmounted this release but kept compiling). The
  deep-link names `create_listing` / `create_bid` map to `CreateTrade` with the matching `kind`
  so old links keep resolving.
- **Deleted:** `CreateBidScreen`, `CreateListingScreen`, their navigator entries and
  `RootStackParamList` keys.

### 3. The Trade tab (bottom nav)

```
┌──────────────────────────────────────────┐
│ TopHeader                                │
│ ┌─────────┬─────────┬─────────┐          │
│ │   All   │   WTS   │   WTB   │          │   ← EntityTabs capsule, three segments
│ └─────────┴─────────┴─────────┘          │
│ ┌──────────┐ ┌──────────┐                │   ← TradesGrid over GET /trades
│ │          │ │          │                │     ?brandId=&kind=
│ └──────────┘ └──────────┘                │
└──────────────────────────────────────────┘
```

- **Pills → capsule.** The pill row becomes the same `EntityTabs` capsule the entity screen
  uses, keys `'all' | 'WTS' | 'WTB'`. *All* is today's Nanza grid unchanged; *WTS* and *WTB*
  are the same grid with `kind=listing` / `kind=bid` set. The query key carries the kind, so each tab keeps its own
  pages.
- **Guests** see the same three tabs (all three are read-only market views). The guest
  early-return that renders the grid without pills goes.
- **Search** is off this screen for now (it only ever searched your own lists). The
  **Create lot** footer is unmounted with the *My Listings* pill (lots hidden this release).
- **`initialTab`** accepts `'all' | 'WTS' | 'WTB'`; the stale `'Bids'` / `'Listings'` values are
  ignored as any unknown value is today. With lots hidden nothing navigates back with
  `initialTab: 'Listings'`.
- **My Bids / My Listings leave this screen but not the code.** `renderMyBidsList`,
  `renderMyListingsList`, `renderSearch`, `TradeListItem`, `BulkSaleCard` and the Create lot
  footer stay in place, unmounted (`// kept for My Trades`), so the My Trades + *Add new trade*
  step wires them back rather than rewriting them. Their lists are still reachable from the
  profile's Trades rail and See-all (`/trade-feed`).
- **Later:** the phase-2 filters from [[guest-browsing#Phase 2: filters (later, same endpoint)|Guest Browsing § Phase 2]]
  (sort, price band, tag values, condition) sit under the capsule as a pill row.

## Build order

1. **API** (half a day): `entityId` + `kind` on `GET /trades`, own-items rule, deploy to dev. **Built 2026-10-25**, see the implementation record.
2. **Mobile — shared pieces** (half a day): `useTrades` options object, `TradesGrid` extracted
   from `TradeMarketGrid`; the tab capsule is Skylar's shared `SegmentedTabs`. **Built 2026-10-25.**
3. **Mobile — entity Trades tab** (half a day): headline block + Add new + grid; delete the four
   hooks' use and the CTA pair. **Built 2026-10-25.**
4. **Mobile — `CreateTrade`** (1 day): new screen, callers moved, old screens deleted, deep links
   mapped. **Built 2026-10-25.**
5. **Mobile — Trade tab** (half a day): All · WTS · WTB over the grid; My Bids / My Listings and
   Create lot unmounted, not deleted; guest branch simplified; lot entry points hidden
   (Create lot, add to lot, lot tiles). **Built 2026-10-25.**
6. Device QA as guest and member: entity tab with none / some / your own trades; create from an
   entity on both tabs; edit an existing one through the grid; lot add-to-lot path; stale
   `create_bid` deep link.

Rough size: API half a day, mobile 3 days.

## Implementation record

### API (2026-10-25, uncommitted)

- `routers/bulkListing.ts`: `buildAccountFeedPage` takes optional `listingWhere` / `bulkWhere` /
  `bidWhere`; an omitted kind contributes nothing. `/sell-feed` and `/trade-feed` are unchanged.
- `GET /trades` reads `brandId`, `entityId`, `kind`, `page`, `limit`. One of `brandId` /
  `entityId` is required (`entityId` wins). `parseFeedKind` accepts `listing` | `bid` | `bulk`
  and throws on anything else (400 through `generatePrismaError`). The `notOwn` exclusion and
  the lot `where` are gone; `getActiveGroupIds` is no longer called here. Listing author filter
  gets `includeBulked = isEntityScope`.
- tsc and eslint clean. No migration, no types bump. Not deployed.
- **Fourth pass, same day:** market scopes (brand / card) swap the viewer-widened visibility
  filters for `isPublic: true`; account and group scopes keep theirs.
- **Second pass, same day:** `accountId` and `groupId` scopes (Skylar: the profile and group
  Trades carousels move onto the one trades endpoint). `/trade-feed` and `/group/:id/items` stay
  mounted for old builds. The group scope re-uses the items route's PRIVATE-group membership
  check inline.

### Mobile (2026-10-25, uncommitted)

- **Vocabulary:** `constants/tradeKinds.ts` holds `TradeKind = 'WTS' | 'WTB'`, the tab labels
  (`TRADE_KIND_LABELS`, the acronyms for now, one place to flip) and
  `FEED_KIND_FOR_TRADE_KIND` (WTS → `listing`, WTB → `bid`).
- **Data:** `bulkListingService.trades({ brandId, entityId, kind }, page, limit)` and
  `useTrades(query, viewerAccountId, enabled)` keyed `['Trades', scope, kind, viewer]`. Every bid /
  listing create, edit and delete now also invalidates `['Trades']` (one line beside each
  `UserTradeFeed` invalidation in the four form components).
- **Tiles and grid:** `components/trade/TradeFeedThumb` is the one trade tile (the thumb trio
  behind a `kind` switch) with `tradeFeedItemKey` / `tradeFeedItemAccountId`; the group
  detail / See-all screens render through it. `components/trade/TradesGrid` is the pure
  two-column grid over it with a spinner footer while paging; `TradeMarketGrid` wraps it in the
  Trade screen's scroll + end-reached paging and takes `kind`. New style bucket
  `styles/components/trade.ts`.
- **Rails on the one endpoint (second pass):** `fetchUserTradeFeed` (profile Trades rail +
  See-all) now calls `/trades?accountId`, `fetchGroupItems` (group rail + See-all) calls
  `/trades?groupId` and returns `TradeFeedItem`s; both keep their query keys so existing
  invalidations still hit.
- **Entity Trades tab:** `components/entity/EntityTradesTab` — one quiet body line (*Buy or sell
  this card on Nanza.*) over the white `ActionPill`, no title (Skylar, 2026-10-25 after seeing
  the build: the button carries the words), then the same **Trades `CardCarousel`** the profile
  and group pages show (second pass: carousel, not a grid) on `useTrades({ entityId })`, gated
  on the tab being open. **Third pass (Skylar, 2026-10-25, voice):** no trade of the viewer's →
  white pill **Trade** under *Buy or sell this card on Nanza.* (or *Be the first to trade this
  card.* when the card has none); one of the loaded trades is theirs (read off the items, no
  extra lookup) → the body describes it (*You're selling this card for $12.* / *You're looking
  to buy this card for $9.*) and the pill is the default ink one, **Edit my trade**. The
  carousel keeps its *Trades* heading and is hidden when the card has no trades.
  `EntityScreen` lost the four hooks, the client-side merge, the measured thumb width, the CTA
  labels and the Place bid / Sell pair; `handleAddNew` gates guests with the auth window and
  opens `CreateTrade`. `market.ts` gained `tradeStart*` + `paneTradeGrid`, lost
  `paneTradeCtas*` / `emptyBlockTrade`; `modals.ts` lost `entityModalFeedSection` /
  `entityModalTradeButtonWrapper`. The tab row is Skylar's shared `SegmentedTabs` (2026-10-25).
- **`CreateTradeScreen`** (`screens/trade/CreateTradeScreen/`): `index.tsx` owns the chrome
  (floating header with trash / share circles in edit mode, the WTS · WTB `SegmentedTabs`, the
  `DetailActionBar`, one `useFormBar`) and drives the mounted pane through a
  `TradePaneHandle` (`submit` / `confirmDelete` / `requestShare`); `ListingPane.tsx` and
  `BidPane.tsx` are the two old screens' bodies with the chrome removed, reporting
  `{ isEditMode, canShare, submitLabel }` up (`tradePane.ts`). Default tab WTS, `kind` param
  picks; with `bulkListingId` only WTS shows. **Second pass (Skylar, 2026-10-25, voice, after
  the device look):** the screen runs the viewer's two per-card lookups itself and, when a
  listing or bid already exists, opens **that** trade in edit mode with **no capsule** (the
  capsule also stays hidden while the lookups resolve, so it never flashes); the CTA reads
  plain **Create** / **Update** (lot add keeps *Add to lot*); with the capsule showing the
  content starts `gutter` under the floating header (`tradeCreateContentWithTabs`) instead of
  the 32 `floatingContentTop`, closer to the chevron; and all four forms render
  `DetailTitleBlock` in `lockupMeta` mode with a set-only `tagLine` (`utils/getSetTagLine`),
  so the tag chips that navigated away from the form are gone. In edit mode the trade tiles'
  **WTS / WTB chiclet** (`IntentTag`, mint for sell, pink for buy) sits above the card name
  (`tradeCreateKindRow`), since there is no capsule to say which kind this is. Route `CreateTrade: { productEntity?, kind?,
  initialImageUri?, bulkListingId? }`, deep link `create_trade`; `CreateBid` / `CreateListing`
  routes, screens and deep links deleted (nothing generated those links). Callers moved:
  `TradeListItem`, `ActionModal/Feed`, `BidScreen` (WTB), `ListingScreen` (WTS), `BulkTab`
  (WTS + lot id).
- **Trade tab, fourth pass (Skylar, 2026-10-25, voice):** members get two buttons floating above
  the nav bar in the old Create lot footer's recipe (`tradeScanSection` + gradient,
  `tradeFooterRow`): **My trades** (ink, `ctaDark`) → the `MyTrades` route, and **Trade** (white)
  → the `TradeSearch` route. Guests see neither. `MyTradesScreen` is no longer parked: it is the
  dashboard of the viewer's own trades — the standard pushed-screen chrome (floating down
  chevron, no title; Skylar, fifth pass), the shared capsule WTS · WTB over the old My Listings /
  My Bids `TradeListItem` rows (each opens `CreateTrade` to edit), the Nanza market pill removed,
  the Create lot footer behind a new `LOTS_ENABLED = false` flag (`constants/featureFlags.ts`).
  **The old search-to-create flow is deleted** (fifth pass): the entity search, its result rows,
  the price-hint maps and `tradeSearchSignal.ts` (with its two callers in the create forms) are
  gone, plus the dead `tradeSearch*` / `tradeResults*` layout styles. Skylar: a first cut, "we're
  gonna have to update that a bit". `TradeSearchScreen` hosts the composer's `CardPickerLayer`
  (Nanza · My Library) with a new `onPickEntity` single-pick prop, **headerless** like the Search
  screen (the picker's footer X closes; status-bar inset + xs above its tabs); tapping a card
  **pushes** `CreateTrade` so the chevron there returns to the search with its query and tab
  intact (fifth pass; a first cut replaced the screen). The picker's second tab reads **My Library** (was *My Collections*) — the one place that
  label lived. Old `tradePillTab*` styles deleted.
- **Trade tab:** `TradeScreen/index.tsx` rewritten: `SegmentedTabs` All · WTS · WTB in the shared
  `tabs.ts segmentedTabsSection` (xxs under the header, page inset — the Trade tab, the inbox and
  Group members now share one placement; Skylar, second pass) over `TradeMarketGrid` with `kind`; guests and members share it; the
  pre-hydration null frame stays. The previous screen is **parked, unmounted** as
  `TradeScreen/MyTradesScreen.tsx` (My Bids / My Listings lists, search, add rows, Create lot
  footer) for the My Trades step; nothing imports it. `initialTab` takes `'all' | 'WTS' | 'WTB'`;
  the lot flow's stale `'Listings'` is ignored and lands on All.
- tsc clean; eslint clean on every touched file (two pre-existing warnings). Not run on a
  device.
- **QA watch:** the WTS / WTB labels read as acronyms in the capsules; the entity grid's gutter
  (CardGrid 8 + wrapper 12 = the pane's 20) against the headline's 20; the create screen's
  capsule spacing under the floating header; tab switch mid-form drops the other form's input
  by design; *Load more* pill on a card with 20+ trades.
- **Still reachable lot entry points** (outside this plan): the profile Trades rail / See-all
  still render `BulkThumb` tiles from `/trade-feed`, and `BulkItems` from a lot's detail. Hiding
  those is a profile change Skylar hasn't asked for yet.

## Decisions (answered 2026-10-25, Skylar, voice)

1. **Own trades.** **In, everywhere.** The viewer's own listings and bids show in the grid on
   the entity tab *and* on the Trade tab's All / WTS / WTB; the `notOwn` exclusion comes off
   `GET /trades`. The tile replaces the CTA-label lookups and is the way into edit.
2. **Lots.** **Hidden this release**, lot tiles included (confirmed). Backend untouched; no lot
   tiles in the feed, no Create lot, no add-to-lot. They return later and will live under WTS.
3. **Where the lot flows land.** Moot while lots are hidden; decide when they return.
4. **Add new on the Trade tab.** **Later.** It arrives with My Trades as two buttons at the
   bottom of the Trade tab (*My trades* · *Add new trade*). This step only gets the filters right.
   The My Bids / My Listings code stays for that step.
5. **Copy.** Revised 2026-10-25 after the build: the entity tab has **no title**; the body line
   (*Buy or sell this card on Nanza.*, or *Be the first to make an offer on this card.* when
   empty) sits over a white pill reading **Start a trade** (was *Add new*). Action bar keeps
   *Create listing* / *Place bid*.

6. **API `kind` values.** `listing` / `bid` / `bulk`, the item union's own names, not WTS / WTB
   (Skylar, 2026-10-25, typed mid-build). WTS / WTB stay the mobile words.
7. **One trade per card per person** (Skylar, 2026-10-25, voice). If the viewer already has a
   listing or a bid on a card, *Start a trade* (and every other way into `CreateTrade` without
   an explicit kind) opens that existing trade to edit, capsule hidden, CTA *Update*. The
   create screen never offers the other kind alongside an existing trade. The API does not yet
   enforce this; a server-side guard is a follow-up if it matters.
8. **Create-screen chrome** (same session): capsule sits closer under the chevron; CTA is
   *Create* / *Update*; the card block shows name, number and set only, no tag chips; the tiles'
   WTS / WTB `IntentTag` chiclet above the name when editing (no capsule to say so). Entity tab:
   the pill block and the Trades carousel sit lg apart (`tradesTabStack`). Entity tab, third pass: pill
   *Trade* (white) or *Edit my trade* (ink) with a body line about the viewer's own trade;
   empty copy *Be the first to trade this card.*; carousel hidden when empty.
9. **Capsule everywhere** (same session, settled on the third pass): `SegmentedTabs` text moves
   from `pill` (13) to `label` (14) on every capsule; **four** segments share the track and a
   fifth or sixth scrolls in at the same width and text size (a five-up try read small). One
   shared section placement (`tabs.ts segmentedTabsSection`) for the Trade tab, inbox and Group
   members, and the **inbox is pinned like the Trade tab**: `MessagesScreen` left `PageLayout`
   (which scrolled the whole page) for the `pageContainer` + inner `ScrollView` layout, so the
   header and capsule stay put and only the list scrolls.
11. **Public-only market** (same session, fourth pass): group-only trades never show on the Trade
   screen or a card's Trades tab, owner included; only in their group and on the owner's profile.
12. **Trade screen footer** (same session): *My trades* (ink) and *Trade* (white) float above the
   nav bar for members. *My trades* opens the own-trades dashboard (first cut); *Trade* opens the
   card picker (Nanza · My Library) and a pick lands on `CreateTrade`. "My Library" replaces "My
   Collections" on search toggles.
13. **My trades and the picker, fifth pass** (same session): My trades keeps only the two lists,
   standard down-chevron chrome, no title — the search-to-create flow is deleted outright, not
   parked. The picker screen has no header (its tabs lead, its X closes) and a pick pushes the
   create screen so back returns to the search as it was.
10. **Rails on one endpoint** (same session): profile and group Trades carousels fetch
   `/trades?accountId` / `/trades?groupId`; the entity tab shows the same carousel with a *Trades*
   title. Lots are gone from those rails with it (hidden this release).

**Next steps Skylar named:** more filters on the Trade tab, then My Trades + Add new trade on the
Trade tab, then lots back under WTS.

## Related

- [[guest-browsing]]: where `GET /trades` and `TradeMarketGrid` came from (2026-10-23)
- [[account-feeds]]: the `buildAccountFeedPage` builder and the trade-feed item union
- [[offer-engine]]: what a tap on a WTS or WTB tile leads to
- [[tcg-api-pricing]]: the Market tab whose layout the Trades tab mirrors
- [[../../nanza-mobile/solution-designs/entity-screen|Entity Screen]]
- [[../../nanza-mobile/solution-designs/trade-screen|Trade tab screen]]
- [[../../nanza-mobile/solution-designs/bulk|Bulk / Lots]]: the WTS-only lot paths on the create screen

## Trade search rows — 2026-10-25 (Skylar, voice; built api + mobile, uncommitted)

- **Dismiss-first taps.** In the Trade screen's card search (`CardPickerLayer` in single-pick
  mode, `onPickEntity` set), a tap while the keyboard is up only puts the keyboard away; the
  next tap opens `CreateTrade`. The composer's multi-select picker keeps its tap-through.
- **"You already trade this."** `GET /search` and `GET /list/collection/:accountId` attach
  `viewerListing` / `viewerBid` to every entity — the requester's own ACTIVE listing / bid on
  the card as `{ id, price, quantity }` or null (`src/utils/viewerTrades.ts`, two batched
  queries over the page's ids; nulls for guests). The trade search rows draw them bottom-right
  as the My Trades pills (`×qty` when > 1, then `$price` — `TradeListItem`'s recipe, listing
  wins over bid, the same precedence `CreateTrade` opens with). A pencil mark was considered
  and dropped for consistency with My Trades.
- **My Trades empty tabs** say "You're not looking to buy anything yet" / "You're not selling
  anything yet", a step below the capsule; the Trade screen footer (My Trades · Trade) floats
  over the grid with no fade behind it.

