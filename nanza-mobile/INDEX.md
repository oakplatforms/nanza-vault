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

## Plans (in progress)

- [[solution-designs/bid-detail-glass-sheet|BidScreen glass sheet]] — BidScreen (and, since 2026-09-15,
  ListingScreen minus its photo hero) 1:1 with Figma
  1213:213817: info button → linked title chevron, glass condition pill above the title, fair-market
  pill, glass user row / stepper, blurred card backdrop; bid-only opt-in props, plus the app-wide CTA
  bar inset/height. Implemented 2026-09-11; pending on-device QA.

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
- [[solution-designs/bulk|Bulk / Lots]] — items-first Lot builder, the DRAFT model, entry rewiring, and
  the 20-item cap.
- [[solution-designs/profile|Profile]] — read-only user-profile parity: count+share headers and the
  read-only collection grid + detail.
- [[solution-designs/payments|Payments & checkout]] — the 3-way delivery selector (Untracked/Tracked/
  In Person), Pay With + Cash placement, the no-jump zeroed summary, and the in-person order status.
- [[solution-designs/forms|Forms]] — the stacked form language (outline fields, pill pickers,
  help text), the keyboard-sticky action bar with gating + success confirm, and the app-wide
  trailing-arc button loading.
- [[solution-designs/detail-screens|Detail Screens & Thumbs]] — the ListingDetail/BidDetail
  pinned-hero screens (lots included), the card-thumb system (`components/cards/`), the shelf
  domain (`components/shelf/`: ShelfHeading, CardGrid, CardCarousel, the ShelfProps contract),
  and the transitional home thumb grid.
- [[solution-designs/design-system|Design System]] — the two-layer colour token model (base scales →
  semantic aliases), the resolver/provider path, the image→`ink` / surface / border conventions,
  `well`→`surface` deprecation, the enforced typographic scale (slot spreads from
  `typography.ts`, ESLint-banned font props), proportional sizing (402/362 reference frame,
  fraction steps, `aspectRatio`-for-height, parent-owns-width — tokens stay exact), the
  inline emoji in running text (`EmojiText`), the
  floating header as the one chrome for back+title screens, the shared glass chrome (one recipe, no backdrop blur, no native
  shadow, gradient-stroke rim-light), and the pending light/dark switch.
- [[solution-designs/auth|Auth]] — the interstitial/screens/sheets split, the two credential
  paths (Cognito SDK for email+password, hosted-domain OAuth+PKCE for Apple/Google), the
  self-gating username sheet, the App-Review terms notice, and the native legal screens.
- [[solution-designs/semantic-token-migration|Semantic Token Migration]] — PENDING: convert the
  ~475 pre-existing raw palette scale refs (`secondary['400']`, `primary['300']`, …) to their
  semantic tokens, as a dedicated consistent pass (deferred from the Aug-2026 token audit).
- [[solution-designs/post-attachments|Post attachments]] — the ENTITY card row on a post
  (thumb + top-anchored identity block + bottom-anchored fair-market pill), its proportional
  sizing, the "Card Gradient 1" tokens, and the reference-payload gap the number/price
  placeholders stand in for.
- [[solution-designs/ui-fixes|UI / UX fixes]] — consistent "more" ellipsis sizing/color and the on-load
  "Update Available" bottom sheet.
- [[solution-designs/analytics|Analytics]] — the typed Mixpanel catalog, the one `track()` gate,
  the call-site pattern (service functions after their request, screens on mount), automatic
  `surface` read from the navigation ref at track time, People properties, the social-signup
  resolution, and why the S1 rules engine was dropped.
- [[solution-designs/docs-infra|Documentation vault]] — the shared `nanza-vault`: relative symlinks,
  INDEX hierarchy, and how designs land here.

## Related

- [[REFERENCE|Frontend Reference]] — patterns, theme tokens, style pattern catalog
- [[architecture|Architecture]] — app wiring, data/state layer, styling system, services
- [[../_shared/INDEX|Shared brain]] — cross-project architecture & catalog

> Keep this file a map that links out — put deep detail in REFERENCE / architecture.
