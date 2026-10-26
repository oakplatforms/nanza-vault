# nanza-mobile

React Native app (iOS + Android) for the Nanza trading-card marketplace: browsing a
brand-filtered feed, buying and bidding on cards, building collections, running trades,
messaging, and sharing listings/bids/collections/profiles via deep links. It is the most
evolved codebase in the family — its styling architecture (ThemeProvider, style buckets,
`dark.json` tokens) is the reference implementation ported to the other repos. All app work
is scoped to `src/`; domain types come from `@oakplatforms/types` re-exported through
`src/types/index.ts`.

See [[REFERENCE|Frontend Reference Guide]] for the deep style rules, theme-token tables, and
pattern catalog, and [[architecture|Architecture]] for how the app is wired together.

## Structural overview

Everything lives under `src/`. Dominant patterns are called out per directory; the exact
rules (no inline styles, no hardcoded colors/sizes, DTO-only types) live in
[[REFERENCE|Frontend Reference Guide]].

- **`screens/`** — feature screens grouped by domain (`home/`, `buy/`, `bids/`, `trade/`,
  `collections/`, `bulk/`, `messages/`, `profile/`, `account/`, `cart/`, `search/`,
  `share/`, `orders/`, `offers/`, …). Each domain keeps its React Query data hooks in a
  local `data/` folder (`useFetchX` naming).
- **`components/`** — reusable UI. `global/` holds cross-domain primitives (Button, Icon,
  Header, Alert, ModalStack); the rest are domain-scoped (`feed/`, `entity/`, `share/`,
  `cart/`, `collections/`, …). One folder per component, `index.tsx` entry.
- **`styles/`** — the theme system. `utils/ThemeProvider.tsx` resolves `tokens/dark.json`
  into `ThemeTokens`; `components/*.ts` are the style-bucket files (`layout`, `typography`,
  `buttons`, `cards`, …) that return classes from `getXStyles(theme)`. Components never
  write inline styles.
- **`services/`** — `api/` wraps every backend call through `fetchData` (one thin module per
  resource); also `analytics/` and `auth/` (Cognito).
- **`contexts/`** — cross-cutting React state: `SessionContext` (current user/access),
  `AuthContext`, `CartContext`, `SavedItemsContext`, `ModalStackContext`,
  `CollectionSelectionContext`, `ListingQuantityContext`, `ShareGuardContext`.
- **`navigation/`** — `AppNavigator`, `BottomTabNavigator`, `ProfileNavigator` (React
  Navigation stacks + tabs, deep-link config). Treated as **read-only**.
- **`hooks/`** — shared hooks not tied to one screen (`useFetchByReference`, `useAppVersion`,
  `useUpdateSheet`, `useCapSheet`, `useShareActions`, `useAddToCart`, `useInboxCount`, …).
- **`types/`** — `index.ts` re-exports DTOs from `@oakplatforms/types`; `navigation.ts` holds
  route param types; generated schemas live alongside.

## Solution designs

Living designs of how each feature area works and why, in `solution-designs/`. Each file
synthesizes the durable decisions across that theme; frontmatter carries a `nanza-mobile`
tag, a `solution-design` tag, and its theme tag.

- [[solution-designs/profile|Profile]] § Saved items retired — hearts and the Saved section are gone (2026-08-29); `SavedItemsContext` deleted (2026-08-30), post likes are `useLike` over `POST /post/:id/like`; favorites comes later.
- [[solution-designs/inbox|Inbox]] — All / Friends / Groups / Offers (2026-08-29): friend requests have no tab — they arrive in All and the Friends tab leads with a "Friend requests (N)" row that opens an accept/decline sheet.
- [[solution-designs/offer-engine|Offer Engine — Mobile]] — **built 2026-08-27, uncommitted (needs `@oakplatforms/types` ≥ 0.1.97)**: one `MakeOfferScreen` (listing + bid variants from the mocks, multi-card from one seller, on-the-fly listing step), Offers-tab-only inbox with a rewritten offer detail, every Buy Now → Make an Offer, and the cart (context, pill, review screen) retired.
- [[solution-designs/app-load-performance|App-load performance]] — the five-request cold-start
  diet (2026-08-14), plus the 2026-10-10 sign-in swap (`HomeViewerSwap` holds Home's viewer-keyed
  sections on the old viewer, prefetches the new viewer's rail/groups/posts, swaps once): `/feed` + Top Picks retired (Query List 1 takes the slot, homepage lists
  `limit=3`), posts `limit=9` with two-item cards + a by-id detail fetch, the lean `/cart-preview`
  at load with full `/orders` on cart open, and `view=session` skipping the user request's
  derived aggregates.
- [[solution-designs/collections|Collections]] — the Collect experience on Profile, items-first; **2026-10-25: private collections retired (isPrivate stored, ignored), `List.isValueVisible` show-value toggle on every collection incl. My Library (pencil back), portfolio total = My Library's value — api + mobile + web built, uncommitted, migration + types 0.1.131 pending**
  detail, `CollectionTargetCard` mosaic, the single-vs-multi add rule, and scan-into-collection.
- [[solution-designs/scan|Scan & Camera]] — one camera two modes, per-type scan pools + cap, mode-aware
  dedup, and the shared `ImageViewer`.
- [[solution-designs/sharing|Sharing]] — the reference-code letter scheme (S/B/C/P/K/G/U), the
  buy/sell/join action matrix, group + profile share cards, and tap-to-expand/cropped images.
- [[solution-designs/share-card-restyle|Share Card Restyle]] — bring the new detail-screen look
  (dark hero, jumbo title, "Sold by" row, priced CTA) to the share cards without re-pointing links.
- [[solution-designs/post-tag-sharing|Post & Tag Sharing]] — proposed: reference codes + OG meta +
  share cards for posts (lazy-minted, root-only) and supported/secondary tag values (new letters
  T/V/Y), across api + web + mobile.
- [[solution-designs/bulk|Bulk / Lots]] — items-first Lot builder, the DRAFT model, entry rewiring, and
  the 20-item cap.
- [[solution-designs/groups|Groups]] — the groups surface; detail header on the shared
  collection/tag banner recipe (jumbo title below the fade), the member facepile + read-only
  People screen, the "Buy & Sell" thumb rail (Add new → Trade), and the loading-frame-parity
  rule that killed the cold-open flash.
- [[solution-designs/profile|Profile]] — read-only user-profile parity: count+share headers and the
  read-only collection grid + detail.
- [[solution-designs/seller-onboarding|Seller onboarding]] — the three-step "Get ready to sell"
  flow; since 2026-09-11 every screen in it is a vertical sheet with the glass down-chevron
  (the ID screens' horizontal-push overrides are gone) and every busy CTA spins.
- [[solution-designs/trade-screen|Trade tab screen]] — the Trade tab's two pills, **Bids** and
  **Listings** (renamed from Buy/Sell, 2026-08-26); the Offers pill and the hidden trade-proposal
  subsystem were deleted the same day.
- [[solution-designs/payments|Payments & checkout]] — the 3-way delivery selector (Untracked/Tracked/
  In Person), Pay With + Cash placement, the no-jump zeroed summary, and the in-person order status.
- [[solution-designs/social-auth|Social Sign-In]] — native Apple/Google sign-in (no browser bounce)
  feeding the consumer Cognito pool via token federation; the `services/auth` seam and UI surface.
- [[solution-designs/figma-handoff|Figma handoff]] — designer edits a frame → `/figma-handoff` writes one implementation prompt to the vault's [[../nanza-figma/INDEX|nanza-figma]] folder → a mobile session implements it; open questions answered inline in the handoff (2026-10-23).
- [[solution-designs/design-system|Design System]] — the two-layer colour token model (base scales →
  semantic aliases), the resolver/provider path, the image→`ink` / surface / border conventions,
  `well`→`surface` deprecation, the enforced typographic scale (slot spreads from
  `typography.ts`, ESLint-banned font props), and the pending light/dark switch.
- [[solution-designs/ui-fixes|UI / UX fixes]] — consistent "more" ellipsis sizing/color, the on-load
  "Update Available" bottom sheet, and the inbox drill-downs (conversation/offer/order, plus
  direct group opens) joining the vertical slide-up family.
- [[solution-designs/data-freshness|Data Freshness]] — lazy-invalidate + refetch-on-page-visit: the
  `AppQueryClient` `refetchType:'none'` default, the navigator `onStateChange` stale-refetch, the
  `staleTime` tiers, and the price-values-active vs discovery-surfaces-lazy split.
- [[solution-designs/tag-taxonomy|Tag Taxonomy (mobile surface)]] — the `TagCarousel` circle-thumb
  rail (addressable by BrandTag id *or* tag name resolved server-side in one request), the home
  "class" shelf between groups and the first Query List shelf (Top Picks retired — see
  [[solution-designs/app-load-performance|App-load performance]]), and the two detail pages.
  Shipped; the second-level
  rail and its screen are reshaped by
  [[../nanza-api/solution-designs/tag-taxonomy-v2|v2]] + [[../nanza-api/solution-designs/posts|Posts]].
- [[solution-designs/posts-chat|Posts Chat]] — the "Join the chat" era: fixed glass pill (above the
  buy/sell bars on listings/bids), snap-open in-place composer (no slide), primary-secondary-tag
  chicklets, reply summary card, post cards with hearts-as-SavedItems, and the ordered content
  renderer incl. LISTING/BID/ENTITY reference cards. Design only; model in
  [[../nanza-api/solution-designs/posts|nanza-api Posts]] § Chat release. 2026-10-18: the
  ENTITY card's thumb is the bid hero tile at thumb scale (`EntityHeroThumb`), card info to its
  right, one branch for stacked row and rail tile.
- [[solution-designs/docs-infra|Documentation vault]] — the shared `nanza-vault`: relative symlinks,
  INDEX hierarchy, and how designs land here.

## Plans (in progress)

- [[../nanza-api/solution-designs/trades-wts-wtb|Trades: WTS / WTB — entity Trades tab + Trade screen]] — **plan, 2026-10-25, API + mobile built same day (uncommitted, tsc + lint clean, pending device QA); lives in the nanza-api vault folder**: entity Trades tab = Market-style headline (*Start a trade*, white **Add new** pill) over a two-column `TradesGrid` on one `GET /trades?entityId` call (the four per-card hooks and the Place bid / Sell CTAs go); new `CreateTrade` screen with a WTS · WTB `EntityTabs` capsule wrapping `CreateListing`/`EditListing` and `CreateBid`/`EditBid`, replacing `CreateBidScreen` + `CreateListingScreen`; Trade tab pills become All · WTS · WTB over the same grid (My Bids / My Listings + Create lot unmounted, not deleted, for the later My Trades + Add new trade step; lots hidden this release).
- [[solution-designs/my-library|My Library — the locked primary collection]] — **plan, 2026-10-24, design review, nothing built**: the primary collection is renamed *My Library* (Skylar, by hand in the DB) and becomes immutable — no rename, no banner/thumbnail, no delete, never private — with API guards on `PUT /list/:id`, the image routes and delete; the card **mosaic becomes the primary's identity** (primary-only, everywhere a tile shows, including add-to-collection), non-primary collections show their banner or a plain tile; the edit form hides Title/Banner for the primary; camera-for-banner parked. Pencil hidden on the primary (answered); two open questions (lock `thumbnail` too, error copy).
- [[solution-designs/stories-post-replies|Stories pager — replies on post pages, light scroll vs hard swipe]] — **plan, 2026-10-23, built same day, uncommitted, pending device QA (mobile only)**: `StoryPostPage` renders the post + `usePostReplies` rows + a pinned *Reply to post* pill inside the pager; a long page scrolls under a light drag (gesture-handler ScrollView) and a hard fling up turns the page through the existing `scrollToIndex`, a hard fling down at the top goes back / closes; short pages page natively as before; **phase 2 (same day): `PostPager` route — What's New posts (home bands + screen) open in the same pager over their own feed cache via `PostItem onOpenDetail` / `PostThread onOpenPost`**. **Phase 3: `PostThread` opens the pager by default, so every chat area (product, listing, bid, group, tag, set) swipes through its own posts; sideways paging + `PagerLockContext` for in-post sliders.** Open: replies on Short pages.
- [[../nanza-api/solution-designs/profile-display-name|Profile Display Name]] — **built 2026-10-23, uncommitted, pending types `0.1.124` install + device QA; lives in the nanza-api vault folder**: `src/utils/profileName.ts getProfileName` (display name, else username) applied to ~25 name slots (post author, hero, badges, inbox rows, threads, rail, Stories), a *Display name* field under Username in profile settings; @-handles and the completion sheet stay on username. Same day: `CircleThumb showPlayBadge` for live-Short circles, and `galleryRail.ts` tiles at one 16:9 height with a seeded four-shape width cycle.
- [[solution-designs/android-nav-bar-insets|Android three-button navigation covers sheet buttons]] — **plan, 2026-10-23, awaiting review**: targetSdk 36 makes the app edge-to-edge on Android 15+ and the shared `Modal` / `BottomSheet` float a fixed 20pt with no bottom inset, so the ~48dp three-button bar covers the quantity sheet's button; add a root `SafeAreaProvider` and float sheets by `getBottomBarInset(theme, insets.bottom)`; direct RN `Modal` users swept. One iOS float decision open.
- [[solution-designs/scan-quick-add|Scan Quick-Add]] — **plan, 2026-10-23, architect-reviewed, ready to build**: scan cap 500 → 300 (lifetime, one-time), listing camera removed (Sell tab + lot builder) and the listing-scan screens deleted, collection scan result loses the top highlight + Add button and gets bigger match tiles; tapping a match opens a swipeable `ScanAddModal` ("You have N" from `ownedQuantity` on `/scan/cards`) whose **Add to collection** calls a new atomic, additive `POST /entity-list/add-by-entity` and returns to the live camera; collection mode skips the scan pool and the seller check; `CollectScanItemsScreen` deleted.
- [[../nanza-api/solution-designs/guest-browsing|Guest Browsing & the Trades feed]] — **plan, 2026-10-23, built 2026-10-23, uncommitted, pending device QA; lives in the nanza-api vault folder**: one nav for guests and members (guest `JoinChatFooter` swap deleted), `TradeMarketGrid` for guests on `GET /trades`, Groups list/detail Join → `AuthModal`, guest pill Home · Groups · Trade · Search with no circle, `finishAuthNavigation` sends every sign-in to Home.
- [[../nanza-api/solution-designs/photo-galleries|Photo Galleries & Image Gates]] — **API deployed to dev 2026-10-18; mobile built 2026-10-19, uncommitted, pending device QA (no migration / types bump)**: a `GROUP` of up to five IMAGE children picked from the camera roll in one go (multi-select library picker, one gallery per post, the existing slot-width rail + one lightbox set), homogeneous groups only; and *images in posts* — picker, camera, gallery — become a public-author (AFFILIATE + VERIFIED) feature enforced in `validateContentItems`, with BASIC composers losing the two image circles. Posting itself stays open. Upload pairing flattens the item tree. ~2–3 days api + mobile. All questions answered 2026-10-18 (one gallery OR one photo, no camera into galleries, gallery tiles a bit smaller than a lone photo) — ready to build.
- [[../nanza-api/solution-designs/reply-likes|Reply Likes]] — **mobile built 2026-10-19, uncommitted, pending device QA (no API change)**: the reply card's `'minimal'` footer becomes heart + count (no reply / share / report); API already hydrates `likeCount`/`viewerHasLiked` on thread reads and `POST /post/:id/like` already takes a reply id, so this is a half-day mobile change in `PostItem`. Heart only, count from one.
- [[../nanza-api/solution-designs/affiliate-featured-shorts|Affiliate Rail — Featured Shorts & the Stories pager]] — **API deployed to dev 2026-10-18; mobile built 2026-10-18, uncommitted, pending device QA (types `0.1.118` installed, tsc + lint clean); lives in the nanza-api vault folder**: own circle → `MyShortScreen` (your live Short with an X that hard-deletes it, or a centred *Paste a YouTube Shorts link* input → `POST /post` with `isFeatured`; one live Short at a time), a new ring rule (unseen Short or unseen latest post), the rail now lists only `Profile.isFeatured` affiliates, and a full-screen vertical `AffiliateStoriesScreen` pager fed by `GET /profiles/affiliates/recent-posts` — autoplaying 9:16 `EmbedPlayer` pages and flat `PostItem` pages, Instagram-style progress bar, hold-to-pause, seen stamped per page, swipe-up past the last page closes.
- [[../nanza-api/solution-designs/agreements|Agreements — affiliate terms before the link]] — **built 2026-10-17, uncommitted; lives in the nanza-api vault folder**: `hasSignedAgreement` on `SessionContext`, Settings swaps the Affiliate link shelf for an *Affiliate agreement* shelf (Review) until signed, new `AffiliateAgreement` screen in `ProfileNavigator` renders `GET /agreement/current?type=AFFILIATE` with one **I agree** → `POST /agreement/:id/accept` → session refresh. Needs `@oakplatforms/types ^0.1.115`.
- [[../nanza-api/solution-designs/airbridge-affiliate-attribution|Airbridge affiliate attribution]] — cross-repo plan (api + mobile built 2026-10-16, uncommitted): Airbridge SDK, `AttributionBridge` parks the link's `ref` and claims `PUT /user/affiliate`; affiliate own-profile share uses the `go.nanza.app/<username>` link.
- [[../nanza-api/solution-designs/affiliate-rail-latest-post-and-seen|Affiliate Rail — latest post, seen state, recency order]] — **built 2026-10-09, uncommitted, pending QA; lives in the nanza-api vault folder**: rail swaps to `GET /profiles/affiliates/latest-posts` (renamed from `/rail` 2026-10-10; viewer-scoped key, still in the first-paint gate), `CircleThumb` gains `ring="quiet"`, tap fetches the latest post and pushes `PostDetail`, and `PostDetailScreen` stamps `POST /post/:id/seen` on every open so reading the post from What's New clears the circle too.
- [[../nanza-api/solution-designs/verified-profiles-and-affiliate-rail|Verified Profiles + Affiliate Rail Restore]] — **built 2026-10-08, uncommitted, pending QA; lives in the nanza-api vault folder**: bring the affiliate circles rail back to the top of Home (component deleted in `de3023ff`, restore from `de3023ff^`, back into the `useHomeReady` gate) and widen the share gate for the new API `VERIFIED` profile type, whose posts reach guests on What's New.
- [[solution-designs/codebase-audit-2026-09|Codebase audit — September 2026]] — **bundles A–D built 2026-09-16, uncommitted, pending QA (tsc 60→0, lint 254→0 errors); Later list + 8 decisions open**: six-agent read-only sweep (dead code · deps & size · network · component standards · performance · tooling). Headline: style buckets rebuilt on every render at ~500 sites (memoize the getters — zero call-site changes), the tsconfig base preset silently not applied (one-line fix, 60→33 type errors), react-hooks lint absent, offer accept/decline invalidating dead keys, ~8–9 MB of dead fonts/images, 41 dead files. Fix-now bundles A–D vs a later-release list; eight decisions for Skylar. Raw reports in `audits/2026-09-16/`.
- [[solution-designs/auth-screens|Auth: modal → full screens + social sign-in]] — **plan,
  NOT STARTED**: the auth modal shrinks to a single marketing interstitial (logo, copy,
  Sign up / Keep browsing); the whole sign-up flow moves to full screens and collapses
  7 steps → 4 (email+password merged, Agreement to fine print, MobileNumber dropped).
  Apple/Google buttons sit above the email fields on screen 1, gated on the Cognito IdP
  work in nanza-auth. 2026-08-22.
- [[solution-designs/profile-sections|Profile Sections]] — **plan, IMPLEMENTED
  (uncommitted)**: tabs off both profile screens; stacked limit-trimmed sections (posts 3,
  collections/listings/bids 2×2, own-profile Saved last), Reviews retired from the profile
  UX, one `ProfileSeeAllScreen` (limit 20, paginated) behind every See all,
  posts+collections gate the first paint, own-profile friends row (facepile + Add
  collection) on the GroupDetail recipe. 2026-08-17.
- [[solution-designs/post-editing-and-drafts|Post Editing & the Single Draft]] — **plan,
  IMPLEMENTED (uncommitted)**: composer edit mode behind the detail screen's edit pencil
  (delete = trash circle beside the dirty-gated Save), and the single restorable server
  draft (DRAFT-status row; silent save on dismiss, restore on open, publish in place).
  2026-08-14; api half in nanza-api.
- [[solution-designs/open-posting-and-nav-swap|Open Home Posting & the Plus/Profile Nav Swap]] —
  **plan**: the top-header plus circle and the bottom-nav "You" avatar swap roles — profile avatar
  up top, a pink plus (avatar-sized) in the tab bar opening the composer, no affiliate gate.
  Planned 2026-08-14; api half (friend-scoped visibility) in nanza-api.
- [[solution-designs/post-carousel-and-tagging|Post Composer — Multi-Content Carousel & Tagging]] —
  **plan**: composer goes to 5 attachments (circles disable at cap), a new multi-select tagging
  screen (10 parent values, removable chips), primary-item cards + detail carousel with
  promote-to-primary swap, replies gain one attachment. Planned 2026-08-11; api half in nanza-api.
- [[solution-designs/entity-group-chat|Entity & Group Chat]] — **plan**: chat on the entity
  screen (ENTITY home already live api-side) and in groups via a new `GROUP` postable target
  with membership-gated writes/reads, no-tag scoping, and no share codes; entity feed becomes a
  "Buy & Sell" carousel with See-all expanding to the grid.
- [[solution-designs/entity-screen|Entity Screen]] — **plan**: convert the `EntityModal` bottom
  sheet into a full-screen slide-up detail screen (chevron-down + share floating header, hero →
  shared `ImageViewer`, same `Entity` route re-registered on the default vertical interpolator).
- [[solution-designs/whats-new|What's New]] — **plan**: affiliate-only pink plus-circle in the top
  header opening the existing post composer in BRAND mode; brand posts surface in a "What's new"
  homepage section (three 3-card bands threaded between the query-list shelves) with See all →
  paginated screen.

## Related

- [[REFERENCE|Frontend Reference]] — patterns, theme tokens, style pattern catalog
- [[architecture|Architecture]] — app wiring, data/state layer, styling system, services
- [[../_shared/INDEX|Shared brain]] — cross-project architecture & catalog

> Keep this file a map that links out — put deep detail in REFERENCE / architecture.
