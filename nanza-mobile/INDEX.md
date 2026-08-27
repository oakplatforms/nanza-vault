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

- [[solution-designs/app-load-performance|App-load performance]] — the five-request cold-start
  diet (2026-08-14): `/feed` + Top Picks retired (Query List 1 takes the slot, homepage lists
  `limit=3`), posts `limit=9` with two-item cards + a by-id detail fetch, the lean `/cart-preview`
  at load with full `/orders` on cart open, and `view=session` skipping the user request's
  derived aggregates.
- [[solution-designs/collections|Collections]] — the Collect experience on Profile, items-first
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
- [[solution-designs/trade-screen|Trade tab screen]] — the Trade tab's two pills, **Bids** and
  **Listings** (renamed from Buy/Sell, 2026-08-26); the Offers pill and the hidden trade-proposal
  subsystem were deleted the same day.
- [[solution-designs/payments|Payments & checkout]] — the 3-way delivery selector (Untracked/Tracked/
  In Person), Pay With + Cash placement, the no-jump zeroed summary, and the in-person order status.
- [[solution-designs/social-auth|Social Sign-In]] — native Apple/Google sign-in (no browser bounce)
  feeding the consumer Cognito pool via token federation; the `services/auth` seam and UI surface.
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
  [[../nanza-api/solution-designs/posts|nanza-api Posts]] § Chat release.
- [[solution-designs/docs-infra|Documentation vault]] — the shared `nanza-vault`: relative symlinks,
  INDEX hierarchy, and how designs land here.

## Plans (in progress)

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
