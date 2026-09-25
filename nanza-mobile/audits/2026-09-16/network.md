# Network & Data-Layer Audit — nanza-mobile

Read-only sweep of React Query, `src/services/api`, realtime and analytics.
Scope: `src/` + `App.tsx`. Baseline docs: `CLAUDE.md` rules 26–30,
`documentation/solution-designs/data-freshness.md`,
`documentation/solution-designs/app-load-performance.md`.

**Headline:** the freshness architecture is genuinely well-built and the
cold-start diet is being honoured (5 requests for a guest, 6 for a signed-in
user — matching the design). The real problems are (a) a cluster of
**invalidation keys that match zero queries**, so several mutations silently
refresh nothing, (b) **51 of 96 image URLs fetching full-size originals**, and
(c) **no request timeout or cancellation anywhere in `fetchData`**.

---

## 1. Summary table (ranked by user-visible impact)

| # | Finding | Requests saved / risk | Effort | Risk | When |
|---|---|---|---|---|---|
| F1 | **51/96 `getImage()` calls fetch full-size originals** — feed cards, listing cards, every avatar, group/collection banners. `src/components/feed/FeedListingCard/FeedListingCardSimple.tsx:66,154`, `src/components/listings/ListingCard/index.tsx:167,282,382` | Largest data-usage item in the app. A feed scroll pulls originals where `IMG_WIDTH.CARD/THUMB` would do. Battery + cellular. | M | Low | **Fix now** |
| F2 | **Offer accept/decline invalidates 4 dead keys and none of the live ones.** `src/screens/offers/OfferMessageDetailScreen/index.tsx:110–116` and `src/screens/offers/MakeOfferScreen/index.tsx:253–258` invalidate `['offers']`, `['orders']`, `['Listings']`, `['Bids']` — **no query uses any of those keys**. The real keys (`['BuyerOrders']`, `['SellerOrders']`, `['ProductListings',…]`, `['ProductBids',…]`, `['AccountBids',…]`, `['listingDetail',…]`) are never marked. | Stale orders/listings after accepting an offer — the latent bug data-freshness.md § Gotchas flagged, still live. | S | Low | **Fix now** |
| F2b | **`['PaymentMethods', accountId]` declared 3× with incompatible cached shapes.** `screens/account/data/fetchPaymentMethods.ts:14` caches `{data: ExternalAccount[]}`; `MakeOfferScreen/index.tsx:189` and `OfferMessageDetailScreen/index.tsx:91` cache a bare `ExternalAccount[]`. One key, two types — whichever populates first is what the other reads. | Wallet→offer (or offer→wallet) navigation reads the wrong shape: empty card list or a render throw on the **checkout** path. | S | Med | **Fix now** |
| F3 | **`['UserBulkListings', accountId]` invalidated at 7 sites; zero queries use it.** `EditBulkScreen:54`, `BulkTab:323`, `CreateListingScreen:70,87`, `CreateListing:242`, `EditListing:292,326` | Bulk create/edit/delete refreshes nothing on the surfaces that show bulks. | S | Low | **Fix now** |
| F4 | **`['group', groupId]` shared by two screens with *different* include sets.** `GroupDetailScreen/index.tsx:66` adds `&include=socialLinks.socialLinkType`; `GroupMembersScreen/index.tsx:46` does not, and its comment claims they match. | Members→Detail navigation renders a group with no social links (whoever mounted first owns the entry). | S | Low | **Fix now** |
| F5 | **No timeout, no `AbortController`, no `AbortSignal` anywhere.** `src/services/api/index.ts:105` `fetch()` has no signal; `rg AbortController src` → 0 hits. React Query's `queryFn`s ignore the `signal` argument. | A stalled request hangs until the OS gives up; abandoned screens keep sockets open. Affects perceived startup on bad networks. | M | Med | Later release |
| F6 | **3 `setQueriesData` cache patches — direct rule-26 violations**, and 2 of the 3 patch `['Feed']`, a **retired key no query uses** (dead code). `BulkTab.tsx:305`, `EditBulkScreen/index.tsx:40` (dead); `CollectionDetailScreen/data/fetchNanzaSearch.ts:76` (live, documented rationale). | Two dead patch blocks (~60 lines) to delete; one live violation to re-evaluate. | S | None | **Fix now** (dead ones) |
| F7 | **`['FeedGrid', …]` (the "See all" grid, `FeedGridScreen/index.tsx:91`) is never invalidated by any mutation.** | Editing a listing/bid leaves the See-all grid stale until its 5-min window lapses. | S | Low | Later |
| F8 | **Homepage shelves use `STALE_TIME['5_MINS']`, design says `30_SECS`.** `QueryListShelves/index.tsx:69` vs data-freshness.md §3 and CLAUDE.md rule 30. `30_SECS` has exactly **one** user in the whole app (`fetchRecentSearches.ts:32`) — the tier is effectively unused. | Either the code or the design is wrong; decide and make them agree. | S | Low | Later |
| F9 | **`['profileSocialLinks', id]` fetched by two screens with identical key/params but no shared hook** (`AccountScreen/Profile/index.tsx:55`, `ProfileScreen/index.tsx:104`), both **without `staleTime`** (fall to the 5-min default). | No extra request (key dedupes), but a copy-paste pair that will drift — exactly the failure mode `useHomeReady` documents. | S | None | Later |
| F10 | **`AddressAutocomplete` typeahead has no in-flight cancellation.** `src/components/global/AddressAutocomplete/index.tsx:76,108` — debounced 300ms but a slow response can overwrite a newer one. Also the only raw `fetch()` outside `services/`, so it bypasses every error/cap convention. | Wrong suggestions during checkout address entry. | S | Low | Later |
| F11 | **4 unbounded `usePagination=false` requests.** `fetchBrands.ts:14`, `fetchShippingMethods.ts:15`, `scanItems.ts:25`, `OfferSellerListingsPicker/index.tsx:42`. `['brands']` is on the cold-start path. | Grows with the dataset; brands is small today, `scanItems` and seller listings are not bounded. | S | Low | Later |
| F12 | **`invalidateQueries()` with no filter** — `AccountScreen/Profile/index.tsx:101`. | Documented and deliberate (identity is denormalised across the cache; lazy default makes it cheap). **No action** — recorded so a future reader doesn't "fix" it. | — | — | Leave |

**Non-findings worth stating** (these were checked and are clean): no N+1 per-item
queries in any list renderer; no `refetchInterval` polling anywhere in the app; no
API responses cached in AsyncStorage; no `retry:` overrides (the global `retry: 0`
holds everywhere); no reads inside `useEffect` outside `SessionContext`; realtime
and analytics volumes are low and well-gated; the cart is fully retired (zero
`CartContext`/`cart-preview` references remain).

---

## 2. Cold-start trace

Provider order (`App.tsx:112–135`): `QueryClientProvider` → `AppReadyProvider` →
`AuthProvider` → `SessionProvider` → `AnalyticsBridge` + `RealtimeBridge` →
`CollectionSelectionProvider` → `ThemeProvider` → `AppRoot` → `ShareGuardProvider`
→ `ListingQuantityProvider` → `AppNavigator`.

`AuthProvider` (`src/contexts/AuthContext.tsx`) holds only local form state — **no
requests**. `CollectionSelectionContext`, `ListingQuantityContext`,
`ShareGuardContext`, `ModalStackContext` likewise fire nothing.

### (a) Signed-in user — 6 requests to first paint

| # | Request | Source | Gate |
|---|---|---|---|
| 1 | `POST`/auth token refresh (Cognito) | `SessionContext.tsx:80` `fetchAuthSession()` | none |
| 2 | `GET /user/:authId?include=account.profile&…&view=session` | `SessionContext.tsx:68` — **outside React Query** (raw `useState`, no cache, no dedupe) | `isSignedIn && !currentUser` |
| 3 | `GET /brands?usePagination=false` | `useFetchBrands` via `BottomTabNavigator:81` **and** `HomeScreen:47` — one key, dedupes | none (`enabled` defaults true) |
| 4 | `GET /profiles?…` (affiliate circles) | `useHomeReady.ts:49` → `AFFILIATES_QUERY_KEY` | `!!selectedBrandId && isAuthReady` |
| 5 | `GET /groups?brandId&viewerAccountId&limit=20` | `useHomeReady.ts:59` | same |
| 6 | `GET /supported-tag-values?isPrimary=true&limit=10` (Trending) | `useHomeReady.ts:74` | same |
| — | `GET /app-version` | `UpdateGate` at `AppNavigator.tsx:838` → `useAppVersion` (`staleTime: Infinity`, **no `enabled`**) | fires unconditionally, races the gate |
| — | `GET /inbox-count?accountId=` | `FloatingTabBar:93` → `useInboxCount` (`staleTime: Infinity`) | `!!accountId` |

**After the gate opens** (`markAppReady` → `InteractionManager`): `GET /posts?limit=9`
(`WhatsNewSection:53` → `usePosts`), `GET /query-lists/homepage?limit=3`
(`HomeScreen:137` → `useHomepageQueryLists(deferredActive)`), then the realtime
WebSocket `$connect` (`RealtimeBridge`, gated on `isAppReady` + interactions, 10s
fallback).

**Verdict vs the diet:** the documented *three-request gate* is met exactly —
affiliates, groups, Trending, nothing else blocks paint. Counting everything on
the wire before paint gives **6** (auth, session, brands, ×3 gate) plus two
ungated strays (`app-version`, `inbox-count`). That is consistent with the
"five-request diet" as written, since the doc counts API reads and excludes the
Cognito token call.

### (b) Anonymous user — 5 requests to first paint

Identical minus #1/#2 (`SessionContext` sets `isSignedIn=false`, never calls
`userService.get`) and minus `inbox-count` (`enabled: !!accountId`). So: brands,
affiliates, groups, Trending, app-version = **5**.

- **No auth-requiring request fires for guests.** `fetchData` (`src/services/api/index.ts:79`)
  falls back to a shared, de-duplicated `/user/guest-token` (`:43–72`, with an
  in-flight promise so concurrent gate queries don't each mint one) — so a guest
  cold start actually costs **6** including that token mint.
- **Cart is gone.** No `CartContext`, `CartProvider`, `CartOrders`, `cart-preview`
  or `CartPreview` reference exists anywhere in `src/` or `App.tsx`. The retirement
  in INDEX.md is fully landed; nothing to remove.
- `useHomeReady`'s `enabled` correctly waits for auth to be *determined*
  (`isSignedIn === false || currentUser !== undefined`) so a signed-in user never
  flashes guest state, and guests aren't blocked behind a session they'll never get.

---

## 3. Duplicate & redundant fetching

**Mirrored keys are correct.** `useHomeReady` deliberately duplicates the
queryKey/params/staleTime of `AffiliateCircleCarousel` (`AFFILIATES_QUERY_KEY`,
shared constant), `GroupBannerCarousel`/`GroupCircleCarousel`
(`['groups','home',brandId,accountId]`) and `TagCarousel`
(`['supportedTagValues','carousel','primary',`${brandId}:all`]`). I verified each
pair — limits (`GROUPS_COUNT=20`, `TAGS_COUNT=10`) match the components'
`DISPLAY_COUNT`, so React Query dedupes and the children add no request. The
`usePosts`-not-copied lesson from the design is also honoured (`WhatsNewSection:53`
calls the hook rather than re-declaring the query).

- **Same endpoint, different include (F4).** `['group', groupId]` — `GroupDetailScreen:66`
  fetches `?include=moderator&include=members.account.profile&include=socialLinks.socialLinkType`;
  `GroupMembersScreen:46` fetches the same key **without** socialLinks. One cache
  entry, two payload shapes. `GroupPeopleScreen:39` correctly shares
  `['group', groupId, 'members', 'ACTIVE']` with an identical query string — that
  one is right, and shows the pattern F4 should follow.
- **`['PaymentMethods', accountId]` declared three times** with three different
  queryFn shapes: `fetchPaymentMethods.ts:13` returns `{data}`, while
  `MakeOfferScreen:188` and `OfferMessageDetailScreen:90` unwrap to `ExternalAccount[]`.
  Same key, **incompatible cached types** — whichever populates first decides what
  the other reads. Latent crash/empty-list risk on the checkout path.
- **`['profileSocialLinks']` copy-pair (F9)** — identical, no shared hook, no staleTime.
- **`['linkPreview', url]`** — correctly shared between `LinkPreviewCard.tsx:31` and
  `InAppBrowserSheet.tsx:74` (same key, same `1_DAY` staleTime, documented). Good.
- **`getPayoutHistory` under two keys** — `['PayoutHistory', accountId]`
  (`fetchPayoutHistory.ts:13`) and `['sellerPayoutHistory', accountId]`
  (`fetchSellerPayoutHistory.ts:13`). Worth confirming these are genuinely
  different reads; if not, it is a double fetch on the wallet surface.
- **No N+1.** I checked every component under `src/components/cards`, `/share`,
  `/shelf`, `/orders`, `/feed` and every file containing `renderItem` (38 files):
  **zero** `useQuery`/`useInfiniteQuery` inside a list-item component. List rows
  render from the parent's payload throughout.
- **No sequential waterfalls in queryFns** — no `await`-chained multi-request
  `queryFn` found. The one dependent pair is intentional and correct:
  `MakeOfferScreen:173` (`['Listing', id, 'seller']`) gates `:181`
  (`['OfferSeller', sellerId]`) only when the feed card didn't carry the seller
  (`enabled: !isBid && !knownSellerId && !!firstListingId`) — it's a fallback, not
  a default waterfall.
- **No `refetchInterval` polling anywhere in `src/`.** Confirmed by grep across all
  97 query hooks.
- **`refetchOnMount: 'always'` on 4 hooks** — `fetchOffer.ts:10`, `fetchBids.ts:34`,
  `fetchSellerOrders.ts:27`, `useBuyerOrders.ts:36`. These bypass staleTime on every
  mount. Defensible for money surfaces, but note it means `['BuyerOrders']`'s
  `STALE_TIME['LIVE']` is decorative — and it partly masks F2 (orders look fresh
  because they always refetch, not because the invalidation works).
- **`refetchOnWindowFocus: true` on 5 messaging queries** (`fetchConnections`,
  `fetchConversations`, `fetchMessageFeed`, `fetchMessages`, `fetchSystemMessages`)
  — these fire on every app foreground *in addition to* the realtime socket's
  pushes and the navigator's focus refetch. Three overlapping refresh mechanisms on
  one surface; worth trimming to two.

---

## 4. Freshness-rule violations

**Rule 26 (no cache patching) — 3 violations:**
- `src/screens/bulk/CreateListingsScreen/BulkTab.tsx:305` — `setQueriesData` over
  `queryKey[0] === 'Feed'`. **Dead**: no query uses `['Feed']` (verified). Also
  preceded by `refetchQueries` + `getQueryData` at `:302–303`, so the whole block
  is a refetch-then-patch round trip against nothing.
- `src/screens/bulk/EditBulkScreen/index.tsx:40` — same predicate, same dead key.
- `src/screens/collections/CollectionDetailScreen/data/fetchNanzaSearch.ts:76` —
  live and heavily justified in-comment (mounted overlay + `refetchType:'none'`
  default leaves the badge stale). This is a genuine gap in the lazy model rather
  than carelessness, but it *is* the rule-26 pattern; the model-consistent fix is
  `refetchType: 'active'` on that key, per rule 28.

`setQueryData` (5 sites) is fine — 4 are `['PostDraft']` local draft state
(`PostComposer.tsx:852,878,925,929`), 1 is an optimistic clear of recent searches
(`fetchRecentSearches.ts:71`). Neither is server-data patching.

**Rule 29 (discovery surfaces stay lazy) — clean.** All 11 `['queryLists']`
invalidations use the lazy default. No `refetchType:'active'` on `['queryLists']`,
`['entities']` or `['Feed']` anywhere. The 50 `refetchType:'active'` sites are all
on same-screen surfaces the rule permits: messages/connections, group members,
scan items, post threads/replies/detail, price values
(`['ProductListings']`/`['ProductBids']`/`['entityDetail']`/`['listingDetail']`/`['bidDetail']`),
payment methods, and the realtime socket. `EditListing/index.tsx:259–266` and
`EditBid/index.tsx:192–197` implement the values-vs-surfaces split exactly as the
design specifies.

**Missing / wrong-key invalidations:**
- **F2** — offers (dead keys `['offers']`, `['orders']`, `['Listings']`, `['Bids']`).
- **F3** — `['UserBulkListings']`, 7 sites, zero queries.
- **F7** — `['FeedGrid']` never invalidated.
- `src/screens/trade/data/scanCards.ts:5` — `useMutation` with no `onSuccess` at
  all. Likely fine (pure scan RPC, no server state changed), but unverified.
- `src/screens/posts/ReportPostScreen/index.tsx:36` — no invalidation,
  explicitly justified in-comment (the queue lives in admin). Correct.

**Over-broad invalidation:** exactly one — `AccountScreen/Profile/index.tsx:101`
`invalidateQueries()` unfiltered. Deliberate and documented (F12).

---

## 5. Realtime + analytics volume

**Realtime (`src/services/realtime/socket.ts`, `src/components/global/RealtimeBridge/index.tsx`) — no issues found.**
- One WebSocket for the app. Frames carry no data, only `{invalidate: string[]}`
  key nudges (`socket.ts:126–134`).
- Each nudge is `invalidateQueries({queryKey:[key], refetchType:'active'})` —
  correct per rule 28, and **scoped to one key per frame**, so no refetch storm.
  The storm risk would be server-side (a frame listing many broad keys); the
  client is not the amplifier.
- Connection is gated on `isAppReady` + `runAfterInteractions` with a 10s fallback
  (`RealtimeBridge:45–58`), so the handshake is off the cold-start path — matching
  the app-load design.
- Heartbeat 4 min (under API Gateway's 10-min idle), exponential backoff with full
  jitter capped at 30s, disconnect on background. Textbook.

**Analytics (`src/services/analytics/*`) — low volume, no issues found.**
- **21 `track()` call sites total**, across 19 discrete named events
  (`events.ts:11–29`). All are explicit user actions (offer sent/accepted, listing
  published, search executed, order placed, scan completed…).
- **No impression, scroll, or render-time tracking.** I grepped every `track(`
  against `onScroll`/`renderItem`/`onViewableItemsChanged`: zero matches. The one
  `track` inside a `useEffect` is `EntityScreen/index.tsx:93` — a single
  `product_view` on screen open, which is correct.
- `identify()` / `setUserProfile()` fire from exactly two places:
  `AnalyticsBridge/index.tsx:63,67` (once per session change) and
  `CompleteProfileModal/index.tsx:93,98` (once on profile completion). No
  per-event people calls.
- A per-user/day rate limiter with a kill switch backs all of it
  (`rateLimiter.ts`), hydrated once at launch so the hot path is synchronous and
  AsyncStorage writes are fire-and-forget. Well designed.

---

## 6. Payload / transfer efficiency

**Images — the biggest finding (F1).** `src/utils/getImage.ts` is well designed:
fixed `IMG_WIDTH` buckets (BLUR 200 / THUMB 400 / CARD 600 / HERO 800 / BANNER 1000)
chosen precisely so the CDN cache key doesn't fragment per device, with a documented
rule that width-less calls skip the resizer Lambda entirely.

But the rule is only half-applied: **96 `getImage()` calls, 45 pass a width, 51 do not.**
The doc's rationale for going width-less ("listing & bid detail heroes, avatars, SVG
logos") covers some, but not these:

| Site | What it renders | Should be |
|---|---|---|
| `components/feed/FeedListingCard/FeedListingCardSimple.tsx:66,154` | feed card art, `resizeMode contain`/`cover` | `IMG_WIDTH.CARD` |
| `components/listings/ListingCard/index.tsx:167,282,382` | three listing-card variants | `IMG_WIDTH.CARD` |
| `components/feed/FeedListingCard/index.tsx:520` | full-screen `ImageViewer` | width-less is correct |
| `components/messages/MessagesListRow.tsx` | inbox row avatars (long list) | `IMG_WIDTH.THUMB` |
| `components/global/Badge/Profile.tsx:60,74,90` | avatar badge, 3 sizes | `IMG_WIDTH.THUMB` |
| `screens/groups/GroupMembersScreen` , `components/groups/GroupSearchRow` | member/search row avatars | `IMG_WIDTH.THUMB` |
| `components/share/ShareGroupCard.tsx:47`, `CollectionTargetCard.tsx:56` | banners | `IMG_WIDTH.BANNER` |
| `screens/profile/ProfileScreen/index.tsx:166` | friend avatar row | `IMG_WIDTH.THUMB` |

Feed and listing cards are the highest-traffic surfaces in the app; every scroll
pulls originals. `ListThumbnailImage.tsx`, `CollectionGridCard.tsx:124` and
`EntitySearchBlock/index.tsx:53` already do it right — the fix is mechanical.

**`include=` relations.** Sampled the ten heaviest screens. Include sets are
consistently tight and justified in-comment — `view=session` on the session read
(`SessionContext.tsx:68`) skips the derived aggregates per the app-load design;
group detail's `members.account.profile` is what the facepile actually renders;
`scanItems.ts:25`'s `entity.product`/`entity.set`/`condition` are all displayed.
I found **no screen requesting a relation it doesn't read**. The only include
problem is F4's *inconsistency* between two callers of one key.

**Response storage / re-parsing.** No API response is written to AsyncStorage —
the only two persisted keys are `INSTALL_FLAG` (`analytics/lifecycle.ts`) and the
analytics rate-limiter counters. No manual re-parsing of cached payloads; React
Query holds the parsed objects. Clean.

---

## 7. Error & retry behaviour

- **Global `retry: 0`** for both queries and mutations (`App.tsx:58,64`), and
  **zero per-hook `retry:` overrides in `src/`** — no hook retries aggressively.
  Confirmed by grep.
- **No timeout and no cancellation (F5).** `fetchData` (`src/services/api/index.ts:105`)
  calls `fetch(fullUrl, options)` with no `signal` and no timer. `rg 'AbortController|AbortSignal|signal:' src` → **0 matches**. Consequences:
  - A hung request blocks its query indefinitely — on the cold-start path, one of
    the three gate queries stalling means the page rides `SPLASH_MAX_MS` (6s) and
    then shows a spinner, with no recovery until the OS times the socket out.
  - React Query passes an `AbortSignal` to every `queryFn`; not one of the 97
    hooks forwards it, so navigating away from a screen never cancels its
    in-flight requests. On a fast tab-switch the app keeps paying for reads
    nobody will render.
  - The fix is contained: thread `signal` through `FetchProps` → `fetch(url, {signal})`,
    add an `AbortSignal.timeout(n)` default, and pass `({signal}) => …` in the
    queryFns that matter. **Note `fetchData` is P0 rule 7 shared infrastructure —
    this needs escalation before anyone touches it.**
- **Error translation is solid.** 429 → `RateLimitError`; `errorCode:'CAP_REACHED'`
  → `CapError` carrying the feature, so the cap sheet renders instead of a generic
  failure; non-JSON error bodies fall back to text then to the status code
  (`index.ts:110–137`). Nothing swallows an error silently.
- **Guest-token minting is correctly de-duplicated** (`index.ts:43–72`): a shared
  in-flight promise means the three gate queries enabling on the same render mint
  one token, not three. 10-minute cache. Good.
- `AddressAutocomplete` (F10) is the one path outside `fetchData` — raw `fetch`,
  `catch {}` that silently empties the list, no cancellation.

---

## 8. Catalog

97 read hooks (75 `useQuery`, 21 `useInfiniteQuery`, 1 `useQueries` running 3
queries) + 33 `useMutation`. `queryClient` call sites: 288 `invalidateQueries`,
5 `setQueryData`, 3 `setQueriesData`, 4 `refetchQueries`, 1 `getQueryData`,
1 `fetchQuery`. No `removeQueries`, `resetQueries`, `cancelQueries` or
`prefetchQuery` anywhere.

**staleTime distribution:** `LIVE`/`5_MINS` 75 · default-5m 8 · `INSTANT`/`0` 5 ·
`1_DAY` 3 · `Infinity` 2 · `1_HOUR` 2 · `30_SECS` 1 · literal `60_000` 1.
The `30_SECS` tier the design reserves for homepage discovery has exactly one
user, and it isn't the homepage (F8).

**`enabled` guard:** present on 82 of 97. The 15 without are mostly static
reference data (`['Conditions']`, `['ShippingMethods']`, `['ShippingOptions']`,
`['socialLinkTypes']`, `['brands']`) which is fine; `['appVersion']`
(`useAppVersion.ts:12`) is the one that fires on every cold start ahead of the
gate.

### Requests made outside React Query (bypass caching)

| Site | Call | Note |
|---|---|---|
| `src/contexts/SessionContext.tsx:68` | `userService.get(userId, '?…&view=session')` | Plain `useState`+`useEffect`. No cache, no dedupe, no retry, no abort. Called from **23 `refreshCurrentUser()` sites**, each a full session refetch — the app-load design already names this the next optimisation target. |
| `src/contexts/SessionContext.tsx:80` | `fetchAuthSession()` | Cognito, cached by Amplify. Fine. |
| `src/components/global/AddressAutocomplete/index.tsx:76` | raw `fetch` Places autocomplete | F10 |
| `src/components/global/AddressAutocomplete/index.tsx:108` | raw `fetch` Places details | one-shot on select; fine |

Everything else routes through a `queryFn` or `mutationFn`. I scanned every
`useEffect` in `src/` for direct service reads: **zero hits** outside
`SessionContext`. That is unusually clean for a codebase this size.

### Full query catalog

| # | file:line | hook | queryKey | staleTime | enabled | overrides |
|---|---|---|---|---|---|---|
| 1 | src/components/groups/GroupBannerCarousel/index.tsx:47 | Query | ['groups', 'home', selectedBrandId, accountId] | STALE_TIME['LIVE'] | yes | — |
| 2 | src/components/groups/GroupCircleCarousel/index.tsx:46 | Query | ['groups', 'home', selectedBrandId, accountId] | STALE_TIME['LIVE'] | yes | — |
| 3 | src/components/groups/GroupTargetingChips/index.tsx:34 | Query | ['groups', 'mine', accountId, 'forTargeting'] | STALE_TIME['LIVE'] | yes | — |
| 4 | src/components/home/QueryListShelves/index.tsx:51 | Query | ['queryLists', 'homepage', selectedBrandId] | STALE_TIME['5_MINS'] | yes | — |
| 5 | src/components/modals/ListModal/data/fetchExistingEntityLists.ts:60 | Query | ['ExistingEntityLists', accountId, entityId] | STALE_TIME['LIVE'] | yes | — |
| 6 | src/components/offers/OfferSellerListingsPicker/index.tsx:51 | Query | ['SellerOfferableListings', sellerAccountId] | STALE_TIME['LIVE'] | yes | — |
| 7 | src/components/posts/composer/CardPickerLayer.tsx:98 | Query | *(inline cfg)* | *default 5m* | **no** | — |
| 8 | src/components/posts/composer/PostComposer.tsx:447 | Query | ['groups', 'mine', accountId, 'forTargeting'] | STALE_TIME['LIVE'] | yes | — |
| 9 | src/components/posts/composer/PostComposer.tsx:472 | Query | ['PostDraft'] | 0 | yes | — |
| 10 | src/components/posts/InAppBrowserSheet.tsx:74 | Query | ['linkPreview', url] | STALE_TIME['1_DAY'] | yes | — |
| 11 | src/components/posts/LinkPreviewCard.tsx:31 | Query | ['linkPreview', url] | STALE_TIME['1_DAY'] | yes | — |
| 12 | src/components/posts/usePosts.ts:34 | InfiniteQuery | *(inline cfg)* | STALE_TIME['LIVE'] | yes | — |
| 13 | src/components/posts/usePosts.ts:86 | Query | ['PostDetail', postId, useViewerKey()] | STALE_TIME['LIVE'] | yes | — |
| 14 | src/components/posts/usePosts.ts:111 | InfiniteQuery | *(inline cfg)* | STALE_TIME['LIVE'] | yes | — |
| 15 | src/components/profile/AffiliateCircleCarousel/index.tsx:40 | Query | AFFILIATES_QUERY_KEY | STALE_TIME['LIVE'] | yes | — |
| 16 | src/components/profile/ProfileSections/index.tsx:93 | Query | *(inline cfg)* | *default 5m* | **no** | — |
| 17 | src/components/profile/SocialLinksEditor/index.tsx:68 | Query | ['socialLinkTypes'] | STALE_TIME['5_MINS'] | **no** | — |
| 18 | src/components/tags/TagCarousel/index.tsx:86 | Query | [ | STALE_TIME['LIVE'] | yes | — |
| 19 | src/hooks/fetchConditions.ts:13 | Query | ['Conditions'] | STALE_TIME['1_DAY'] | **no** | — |
| 20 | src/hooks/fetchOffer.ts:10 | Query | ['offer', offerId] | STALE_TIME['LIVE'] | yes | refetchOnMount='always' |
| 21 | src/hooks/fetchProductBids.ts:23 | Query | ['ProductBids', entityId] | STALE_TIME['INSTANT'] | yes | — |
| 22 | src/hooks/fetchUserProductListing.ts:16 | Query | ['UserProductListing', entity?.id, currentUser?.acco… | STALE_TIME['LIVE'] | yes | — |
| 23 | src/hooks/useAppVersion.ts:12 | Query | ['appVersion'] | Infinity | **no** | refetchOnWindowFocus=false |
| 24 | src/hooks/useFetchByReference.ts:59 | Query | ['reference', referenceCode] | STALE_TIME['LIVE'] | yes | — |
| 25 | src/hooks/useInboxCount.ts:17 | Query | ['inboxCount', accountId] | Infinity | yes | refetchOnWindowFocus=false |
| 26 | src/hooks/useUserGroupIds.ts:21 | Query | ['groups', 'mine', accountId] | STALE_TIME['LIVE'] | yes | — |
| 27 | src/screens/account/AccountScreen/Profile/index.tsx:55 | Query | ['profileSocialLinks', profileId] | *default 5m* | yes | — |
| 28 | src/screens/account/data/fetchPaymentMethods.ts:13 | Query | ['PaymentMethods', accountId] | STALE_TIME['LIVE'] | yes | — |
| 29 | src/screens/account/data/fetchSellerById.ts:17 | Query | ['Seller', sellerId] | STALE_TIME['LIVE'] | yes | — |
| 30 | src/screens/account/data/fetchShippingMethods.ts:13 | Query | ['ShippingMethods'] | STALE_TIME['LIVE'] | **no** | — |
| 31 | src/screens/account/data/fetchShippingOptions.ts:13 | Query | ['ShippingOptions'] | STALE_TIME['LIVE'] | **no** | — |
| 32 | src/screens/bids/BidScreen/data/fetchBid.ts:21 | Query | ['bidDetail', bidId] | STALE_TIME['LIVE'] | yes | — |
| 33 | src/screens/bids/data/fetchBids.ts:34 | InfiniteQuery | ['AccountBids', accountId] | STALE_TIME['LIVE'] | yes | refetchOnMount='always' |
| 34 | src/screens/bulk/BulkDetailScreen/data/fetchBulkListing.ts:7 | Query | ['BulkListing', idOrCode] | STALE_TIME.LIVE | yes | — |
| 35 | src/screens/buy/data/fetchProductListings.ts:18 | Query | ['ProductListings', entity?.id] | STALE_TIME['INSTANT'] | yes | — |
| 36 | src/screens/buy/data/fetchUserProductBid.ts:17 | Query | ['UserProductBid', entity?.id, currentUser?.account?… | STALE_TIME['LIVE'] | yes | — |
| 37 | src/screens/collections/AddToCollectionScreen/index.tsx:36 | Query | *(inline cfg)* | *default 5m* | **no** | — |
| 38 | src/screens/collections/CollectionDetailScreen/data/fetchCollectionEntities.ts:57 | InfiniteQuery | ['collectionDetail', listId, accountId, search ?? nu… | STALE_TIME['LIVE'] | **no** | — |
| 39 | src/screens/collections/CollectionDetailScreen/data/fetchNanzaSearch.ts:42 | InfiniteQuery | ['collectionNanzaSearch', term, accountId ?? null] | STALE_TIME['LIVE'] | **no** | — |
| 40 | src/screens/collections/CollectionDetailScreen/index.tsx:70 | Query | *(inline cfg)* | *default 5m* | **no** | — |
| 41 | src/screens/collections/CollectionPickerScreen/index.tsx:32 | Query | *(inline cfg)* | *default 5m* | **no** | — |
| 42 | src/screens/entity/EntityScreen/data/fetchEntityDetail.ts:20 | Query | ['entityDetail', entityId, accountId] | STALE_TIME['INSTANT'] | yes | — |
| 43 | src/screens/feedGrid/FeedGridScreen/index.tsx:90 | InfiniteQuery | ['FeedGrid', source, selectedBrandId, accountId] | STALE_TIME['LIVE'] | yes | — |
| 44 | src/screens/groups/data/fetchGroupItems.ts:29 | InfiniteQuery | ['group', groupId, 'items', 'seeAll'] | STALE_TIME.LIVE | yes | — |
| 45 | src/screens/groups/data/fetchGroupItems.ts:43 | Query | ['group', groupId, 'items', 'preview'] | STALE_TIME.LIVE | yes | — |
| 46 | src/screens/groups/GroupDetailScreen/index.tsx:66 | Query | ['group', groupId] | STALE_TIME['LIVE'] | yes | — |
| 47 | src/screens/groups/GroupMembersScreen/index.tsx:46 | Query | ['group', groupId] | STALE_TIME['LIVE'] | yes | — |
| 48 | src/screens/groups/GroupMembersScreen/index.tsx:58 | Query | ['group', groupId, 'members', 'PENDING'] | STALE_TIME['LIVE'] | yes | — |
| 49 | src/screens/groups/GroupMembersScreen/index.tsx:65 | Query | ['group', groupId, 'members', 'ACTIVE'] | STALE_TIME['LIVE'] | yes | — |
| 50 | src/screens/groups/GroupPeopleScreen/index.tsx:39 | Query | ['group', groupId, 'members', 'ACTIVE'] | STALE_TIME['LIVE'] | yes | — |
| 51 | src/screens/groups/GroupsListScreen/index.tsx:45 | InfiniteQuery | ['groups', 'list', selectedBrandId] | STALE_TIME['LIVE'] | yes | — |
| 52 | src/screens/home/data/fetchBrands.ts:17 | Query | ['brands'] | STALE_TIME['LIVE'] | **no** | refetchOnWindowFocus=false |
| 53 | src/screens/home/data/useHomeReady.ts:49 | Queries | AFFILIATES_QUERY_KEY | STALE_TIME['LIVE'] | **no** | — |
| 54 | src/screens/listings/ListingScreen/data/fetchListing.ts:23 | Query | ['listingDetail', listingId] | STALE_TIME['LIVE'] | yes | — |
| 55 | src/screens/messages/MessagesScreen/data/fetchConnections.ts:17 | Query | ['connections', accountId, status ?? 'all'] | STALE_TIME['LIVE'] | yes | refetchOnWindowFocus=true |
| 56 | src/screens/messages/MessagesScreen/data/fetchConversation.ts:12 | Query | ['conversation', conversationId, accountId] | STALE_TIME['LIVE'] | yes | — |
| 57 | src/screens/messages/MessagesScreen/data/fetchConversations.ts:12 | Query | ['conversations', accountId] | STALE_TIME['LIVE'] | yes | refetchOnWindowFocus=true |
| 58 | src/screens/messages/MessagesScreen/data/fetchMessageFeed.ts:19 | Query | ['messageFeed', accountId] | STALE_TIME['LIVE'] | yes | refetchOnWindowFocus=true |
| 59 | src/screens/messages/MessagesScreen/data/fetchMessages.ts:12 | Query | ['messages', conversationId, accountId] | STALE_TIME['LIVE'] | yes | refetchOnWindowFocus=true |
| 60 | src/screens/messages/MessagesScreen/data/fetchSystemMessages.ts:15 | Query | ['systemMessages', accountId, category] | STALE_TIME['LIVE'] | yes | refetchOnWindowFocus=true |
| 61 | src/screens/offers/MakeOfferScreen/index.tsx:173 | Query | ['Listing', firstListingId, 'seller'] | STALE_TIME['LIVE'] | yes | — |
| 62 | src/screens/offers/MakeOfferScreen/index.tsx:181 | Query | ['OfferSeller', sellerId] | STALE_TIME['LIVE'] | yes | — |
| 63 | src/screens/offers/MakeOfferScreen/index.tsx:188 | Query | ['PaymentMethods', accountId] | STALE_TIME['LIVE'] | yes | — |
| 64 | src/screens/offers/OfferMessageDetailScreen/index.tsx:83 | Query | ['OfferSeller', order?.sellerId] | STALE_TIME['LIVE'] | yes | — |
| 65 | src/screens/offers/OfferMessageDetailScreen/index.tsx:90 | Query | ['PaymentMethods', accountId] | STALE_TIME['LIVE'] | yes | — |
| 66 | src/screens/orders/data/fetchOrder.ts:27 | Query | ['order', orderId] | STALE_TIME.LIVE | yes | — |
| 67 | src/screens/orders/data/fetchSellerOrders.ts:27 | InfiniteQuery | ['SellerOrders', accountId] | STALE_TIME.LIVE | yes | refetchOnMount='always' |
| 68 | src/screens/orders/data/useBuyerOrders.ts:36 | InfiniteQuery | ['BuyerOrders', accountId] | STALE_TIME.LIVE | yes | refetchOnMount='always' |
| 69 | src/screens/payouts/data/fetchPayoutHistory.ts:13 | Query | ['PayoutHistory', accountId] | STALE_TIME['LIVE'] | yes | — |
| 70 | src/screens/payouts/data/fetchSellerPaymentMethods.ts:13 | Query | ['sellerPaymentMethods', accountId] | STALE_TIME['LIVE'] | yes | — |
| 71 | src/screens/payouts/data/fetchSellerPayoutHistory.ts:13 | Query | ['sellerPayoutHistory', accountId] | STALE_TIME['LIVE'] | yes | — |
| 72 | src/screens/payouts/data/fetchWalletBalance.ts:13 | Query | ['WalletBalance', accountId] | STALE_TIME['LIVE'] | yes | — |
| 73 | src/screens/posts/PostDetailScreen/index.tsx:69 | Query | ['group', postableId] | STALE_TIME['LIVE'] | yes | — |
| 74 | src/screens/posts/PostDetailScreen/index.tsx:79 | Query | ['supportedTagValue', postableId] | STALE_TIME['LIVE'] | yes | — |
| 75 | src/screens/profile/ProfileScreen/index.tsx:104 | Query | ['profileSocialLinks', profile?.id] | *default 5m* | yes | — |
| 76 | src/screens/profile/ProfileSeeAllScreen/index.tsx:63 | InfiniteQuery | ['lists', 'COLLECTION', accountId, 'seeAll'] | STALE_TIME['LIVE'] | yes | — |
| 77 | src/screens/profile/UserProfileScreen/data/fetchUserBids.ts:35 | InfiniteQuery | ['UserProfileBids', accountId] | STALE_TIME.LIVE | yes | — |
| 78 | src/screens/profile/UserProfileScreen/data/fetchUserListings.ts:26 | InfiniteQuery | ['UserProfileListings', accountId] | STALE_TIME.LIVE | yes | — |
| 79 | src/screens/profile/UserProfileScreen/data/fetchUserPosts.ts:37 | InfiniteQuery | ['UserProfilePosts', accountId, viewerKey] | STALE_TIME.LIVE | yes | — |
| 80 | src/screens/profile/UserProfileScreen/data/fetchUserPosts.ts:58 | Query | ['UserProfilePosts', accountId, viewerKey, 'preview'] | STALE_TIME.LIVE | yes | — |
| 81 | src/screens/profile/UserProfileScreen/data/fetchUserProfile.ts:13 | Query | ['userProfile', profileId] | STALE_TIME.LIVE | yes | — |
| 82 | src/screens/profile/UserProfileScreen/data/fetchUserSellFeed.ts:23 | InfiniteQuery | ['UserSellFeed', accountId] | STALE_TIME.LIVE | yes | — |
| 83 | src/screens/profile/UserProfileScreen/data/fetchUserTradeFeed.ts:24 | InfiniteQuery | ['UserTradeFeed', accountId] | STALE_TIME.LIVE | yes | — |
| 84 | src/screens/profile/UserProfileScreen/data/fetchUserTradeFeed.ts:43 | Query | ['UserTradeFeed', accountId, 'preview'] | STALE_TIME.LIVE | yes | — |
| 85 | src/screens/search/data/fetchBrandSets.ts:30 | InfiniteQuery | ['brandSets', brandId] | STALE_TIME['1_HOUR'] | yes | refetchOnWindowFocus=false |
| 86 | src/screens/search/data/fetchBrandTags.ts:42 | Query | ['brandTags', brandId, options?.valuesIsPrimary ?? f… | STALE_TIME['LIVE'] | yes | refetchOnWindowFocus=false |
| 87 | src/screens/search/data/fetchFilteredProductEntities.ts:58 | Query | *(inline cfg)* | STALE_TIME['LIVE'] | yes | — |
| 88 | src/screens/search/data/fetchGroupSearch.ts:24 | InfiniteQuery | ['groupSearch', trimmed, brandId, viewerAccountId] | STALE_TIME['LIVE'] | yes | — |
| 89 | src/screens/search/data/fetchRecentSearches.ts:22 | Query | recentSearchesKey(accountId) | STALE_TIME['30_SECS'] | yes | — |
| 90 | src/screens/search/data/fetchSearch.ts:83 | Query | *(inline cfg)* | STALE_TIME['LIVE'] | yes | — |
| 91 | src/screens/search/data/fetchSupportedTagValues.ts:45 | InfiniteQuery | ['supportedTagValues', brandTagId, optionsKey] | STALE_TIME['1_HOUR'] | yes | refetchOnWindowFocus=false |
| 92 | src/screens/tags/SupportedTagValueScreen/index.tsx:57 | Query | ['supportedTagValue', supportedTagValueId] | STALE_TIME['LIVE'] | yes | — |
| 93 | src/screens/trade/CollectScanTargetScreen/index.tsx:34 | Query | *(inline cfg)* | *default 5m* | **no** | — |
| 94 | src/screens/trade/data/scanItems.ts:20 | Query | SCAN_ITEMS_KEY(accountId, type) | 0 | yes | — |
| 95 | src/screens/trade/ScanCameraScreen/index.tsx:206 | Query | ['scanVariants', entityIdsCsv] | 60_000 | yes | — |
| 96 | src/screens/trade/TradeScreen/CollectTab.tsx:120 | InfiniteQuery | ['collectTab', 'collection', accountId, activeCollec… | STALE_TIME['LIVE'] | yes | — |
| 97 | src/screens/trade/TradeScreen/CollectTab.tsx:135 | InfiniteQuery | ['collectTab', 'search', debouncedSearchText, accoun… | STALE_TIME['LIVE'] | yes | — |
