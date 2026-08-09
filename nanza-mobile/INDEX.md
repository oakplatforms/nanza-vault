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
- [[solution-designs/profile|Profile]] — read-only user-profile parity: count+share headers and the
  read-only collection grid + detail.
- [[solution-designs/payments|Payments & checkout]] — the 3-way delivery selector (Untracked/Tracked/
  In Person), Pay With + Cash placement, the no-jump zeroed summary, and the in-person order status.
- [[solution-designs/social-auth|Social Sign-In]] — native Apple/Google sign-in (no browser bounce)
  feeding the consumer Cognito pool via token federation; the `services/auth` seam and UI surface.
- [[solution-designs/design-system|Design System]] — the two-layer colour token model (base scales →
  semantic aliases), the resolver/provider path, the image→`ink` / surface / border conventions,
  `well`→`surface` deprecation, the enforced typographic scale (slot spreads from
  `typography.ts`, ESLint-banned font props), and the pending light/dark switch.
- [[solution-designs/ui-fixes|UI / UX fixes]] — consistent "more" ellipsis sizing/color and the on-load
  "Update Available" bottom sheet.
- [[solution-designs/data-freshness|Data Freshness]] — lazy-invalidate + refetch-on-page-visit: the
  `AppQueryClient` `refetchType:'none'` default, the navigator `onStateChange` stale-refetch, the
  `staleTime` tiers, and the price-values-active vs discovery-surfaces-lazy split.
- [[solution-designs/tag-taxonomy|Tag Taxonomy (mobile surface)]] — the `TagCarousel` circle-thumb
  rail (addressable by BrandTag id *or* tag name resolved server-side in one request), the home
  "class" shelf between groups and Top Picks, and the two detail pages. Shipped; the second-level
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

- [[solution-designs/whats-new|What's New]] — **plan**: affiliate-only pink plus-circle in the top
  header opening the existing post composer in BRAND mode; brand posts surface in a "What's new"
  homepage section (3 cards between Trending and Top Picks) with See all → paginated screen.

## Related

- [[REFERENCE|Frontend Reference]] — patterns, theme tokens, style pattern catalog
- [[architecture|Architecture]] — app wiring, data/state layer, styling system, services
- [[../_shared/INDEX|Shared brain]] — cross-project architecture & catalog

> Keep this file a map that links out — put deep detail in REFERENCE / architecture.
