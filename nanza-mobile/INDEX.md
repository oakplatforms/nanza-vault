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
- [[solution-designs/bulk|Bulk / Lots]] — items-first Lot builder, the DRAFT model, entry rewiring, and
  the 20-item cap.
- [[solution-designs/profile|Profile]] — read-only user-profile parity: count+share headers and the
  read-only collection grid + detail.
- [[solution-designs/ui-fixes|UI / UX fixes]] — consistent "more" ellipsis sizing/color and the on-load
  "Update Available" bottom sheet.
- [[solution-designs/docs-infra|Documentation vault]] — the shared `nanza-vault`: relative symlinks,
  INDEX hierarchy, and how designs land here.

## Related

- [[REFERENCE|Frontend Reference]] — patterns, theme tokens, style pattern catalog
- [[architecture|Architecture]] — app wiring, data/state layer, styling system, services
- [[../_shared/INDEX|Shared brain]] — cross-project architecture & catalog

> Keep this file a map that links out — put deep detail in REFERENCE / architecture.
