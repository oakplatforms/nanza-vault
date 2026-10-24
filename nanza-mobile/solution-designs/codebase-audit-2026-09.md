---
tags: [nanza-mobile, solution-design, plan, audit, tooling, performance, design-system, data-freshness]
status: BUNDLES A–D BUILT 2026-09-16, uncommitted, pending Skylar QA — tsc 60→0 errors, eslint 254→0 errors (31 exhaustive-deps warnings), 41 dead files + ~5.6 MB assets removed, style getters memoized, offer/bulk invalidation keys fixed, StarRating + OrdersList + scrim tokens landed. See "What landed" for deviations and the QA list. Run `cd ios && pod install` before building (two native deps removed).
---

# nanza-mobile codebase audit — September 2026

## Goal

A standardization and efficiency pass over the whole mobile app (`src/`, 580 files, ~100k lines)
before the next release, with **no breaking changes**: find dead code and dead dependencies,
obsolete practices, unneeded network work, render inefficiency, and drift from our own
component/design-system standards — then decide what to fix now versus in a later release.

This document is the synthesis. It ranks findings, proposes fix bundles, and lists the decisions
that are Skylar's to make. The six raw sweep reports (every count, every file:line) live in
[[../audits/2026-09-16/INDEX|audits/2026-09-16]].

## How the audit was run

Orchestrated as one top-level session fanning out six read-only sub-agents, one per area
(dead code · dependencies & size · network/data layer · component standards · runtime
performance · obsolete practices & tooling), each briefed with `CLAUDE.md`, the ui-review
checklist and the relevant solution designs. Mechanical tools ran first and fed the agents:
`tsc --noEmit`, ESLint (JSON), `knip`, `depcheck`. Every number below came from a command
run against `dev` on 2026-09-16; where two agents disagreed, the top level re-ran the check
(noted inline). Pattern recorded in [[../../_shared/capabilities/orchestrations|Orchestration patterns]].

## Scorecard

| Area | Health | One-line verdict |
|---|---|---|
| Network / data layer | **Good** | Freshness model honoured, cold start hits the 5-request diet, no polling, no N+1. A handful of wrong-key invalidations and 51 full-size image fetches. |
| Component standards (P0 rules) | **Good** | 76 static inline styles, 11 live hex, 7 font-prop hits, 21 `any` in 100k lines. Debt is duplication, not rule-breaking. |
| Runtime performance | **Poor** | Style buckets (30k lines) rebuilt on every render at ~500 call sites; `React.memo` used once; ThemeProvider value unmemoized. |
| Dead code & assets | **Fair** | 41 dead files (~6.6k lines), ~3 MB dead fonts, ~4.5 MB unreferenced images, a retired cart bucket. |
| Dependencies | **Fair** | 11 removable packages (2 native), Android R8 off, 290 pods. |
| Tooling & hygiene | **Poor** | tsconfig base preset silently not applied, react-hooks lint absent, ESLint 8 EOL, no CI, two lockfiles, stale root artifacts. |

## The five findings that matter most

1. **Style objects are rebuilt on every render, app-wide.** ~470–520 `getXStyles(theme)` calls
   (the two sweeps used slightly different regexes) across ~300 files; 7 are memoized. The
   buckets total 16.5k runtime properties; `cards.ts` alone is 3,880 and is called by
   `FeedListingCard`, `ListingCard` and `PostItem` — once per list row. 32 of 37 buckets call
   other getters internally (`cards.ts` rebuilds all of `typography.ts` as a side effect). A
   V8 micro-benchmark of one card row's rebuild measured ~1.8 ms and ~550 allocations; ten
   visible rows exceed a 60 fps frame on desktop, and Hermes on mid-range Android is several
   times slower. `ThemeProvider` compounds it by handing a new `value` literal to 313
   `useTheme()` consumers on every provider render, and only one component in the app uses
   `React.memo`. **Fix is small and behaviour-preserving** (see Bundle B) because every getter
   is pure, single-arg `(theme)`, and `theme` is a module-level constant — a `WeakMap` memo at
   the getter definition collapses all calls with zero call-site changes. A memoized
   `useThemeStyles` hook already exists in `src/styles/utils/` with zero consumers.
2. **The TypeScript base preset is not being applied.** `tsconfig.json` extends
   `@react-native/typescript-config/tsconfig.json`, but the package's `exports` map only
   exposes `"."` and `"./strict"`, so tsc reports TS6053 and silently ignores the preset.
   Verified: changing the line to `"extends": "@react-native/typescript-config"` (and
   dropping the phantom `"jest"` from `types`) takes `tsc --noEmit` from **60 errors to 33** —
   the 27 that vanish are all fallout of the missing preset (`allowImportingTsExtensions`).
   The 33 that remain are genuine (a `Tabs` props mismatch in `Header`/`StickyTabs`,
   `groupIds` → `groups` in `CreateBid`/`EditBid`, implicit `any` in `sanitizeNativeProps`, …).
3. **Hook-dependency linting is off tree-wide.** `eslint-plugin-react-hooks` is not
   installed or configured; the single "exhaustive-deps" hit in lint output is ESLint saying
   the rule definition was not found. 745 `useMemo`/`useCallback` dependency arrays are
   unchecked. Related: the "254 lint errors" headline is 72% noise from `android/app/build/`
   and `amplify/` because `eslint.config.mjs` has no `ignores` for them; real `src/` debt is
   **70 errors**, most auto-fixable.
4. **Offer accept/decline refreshes nothing.** `OfferMessageDetailScreen` and `MakeOfferScreen`
   invalidate `['offers']`, `['orders']`, `['Listings']`, `['Bids']` — keys no query uses.
   The live keys (`BuyerOrders`, `SellerOrders`, `ProductListings`, `ProductBids`,
   `AccountBids`, `listingDetail`) are never marked. It is masked because the order lists use
   `refetchOnMount: 'always'`. Same family: `['UserBulkListings']` invalidated at 7 sites with
   zero queries; `['PaymentMethods', accountId]` declared 3× with two incompatible cached
   shapes on the checkout path; `['group', groupId]` fetched with different `include` sets by
   two screens.
5. **Binary size carries ~8–9 MB of dead weight.** 29 of 35 linked fonts (Figtree, Geist,
   EuclidCircularB, Roobert-Italic; ~3 MB) match no token in `dark.json`/`typography.ts`;
   `profile.png` (3.2 MB), `featuredCard-001.jpg`, `backdrop.svg`, `icon-master.png` and an
   Illustrator source `collected.ai` have zero references in `src/`, `ios/` or `android/`;
   `splash-backdrop.jpg` (2.8 MB) *is* used, on the cold-start path, and is uncompressed.
   Android release builds have `enableProguardInReleaseBuilds = false` and no
   `shrinkResources`. Two dead native deps (`react-native-view-shot`,
   `react-native-floating-label-input`) still link pods.

## Corrections to what we assumed going in

- **i18n is live, not dead.** `src/lang/` holds `en/fr/ja.json` + `i18n.js`, wired in
  `index.js`, used in 9 files (~38 calls, mostly auth). Only ~1% of ~1,000 `<Text>` nodes are
  localized — a scope decision, not a cleanup item. One probable bug: `t(order.name)` in
  `EntityCard/Listing.tsx:83` and `EntityCard/List.tsx:132` passes a product name as a key.
- **`react-native-qrcode-svg` was only used by the dead `QrCode` component** — it went with it.
- **`buffer` is required** — an undeclared dependency of `react-native-svg`; knip and depcheck both mis-flagged it.
- **Reviews are not retired** — `WriteReviewModal` is live in `OrderDetailScreen`.
- **Cart, SavedItems, trade proposals, `/feed`, Top Picks are fully gone** from code; only the
  `styles/components/cart.ts` bucket (28 KB), `calculateCartTotals.ts`, and two
  `setQueriesData` patches against the retired `['Feed']` key remain.
- **`amazon-cognito-identity-js` + `react-native-app-auth` are both needed** (password flows
  vs OAuth federation — the Cognito SDK cannot federate).
- **`polyfill.js` never ships in the app** — it is a `NODE_OPTIONS --require` shim for the
  Metro/CLI process, and its Array polyfills are no-ops on Node ≥ 20.

## Triage — what to do when

### Bundle A · Fix now, zero user-visible risk (config + deletions)

| Item                    | Detail                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Effort |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| tsconfig extends        | `"extends": "@react-native/typescript-config"`; remove `"jest"` from `types`. 60 → 33 errors.                                                                                                                                                                                                                                                                                                                                                                                 | XS     |
| ESLint ignores          | Add `android/**`, `ios/**`, `amplify/**`, `.plans/**`, `**/*.d.ts` to `ignores`; run `--fix` (178 of the errors are indent/semi/quotes).                                                                                                                                                                                                                                                                                                                                      | XS     |
| react-hooks plugin      | Install `eslint-plugin-react-hooks`; `rules-of-hooks: error`, `exhaustive-deps: warn` (745 sites unchecked — `error` would break day one).                                                                                                                                                                                                                                                                                                                                    | XS     |
| Dead dependencies       | Remove `npm`, `install`, `buffer`, `@types/react-native-dotenv`, `react-native-view-shot`, `react-native-floating-label-input`; dev: `@typescript-eslint/parser`, `metro-config`, `openapi-typescript`, `prettier`, `react-dom`. Then `pod install` to drop two pods.                                                                                                                                                                                                         | S      |
| Dead files              | Delete the 37 confirmed-dead files (full list in the dead-code report §1), incl. the `FeedListingCard` → `FeedListingPreview` → `FeedListingPreviewBase` → `SyncedSlider` chain, `FeedBidCard`, `FeedBulkCard`, `ListingCard`, `EntityCard/*`, `CollectTab`, `CollectScanTargetScreen`, `cart.ts`, `calculateCartTotals.ts`, `styles/components/index.ts`, `Invoice.ts`. Keep `ProfileListThumbnails/ListThumbnailImage.tsx` (used directly) and `polyfill.js`. ~6,570 lines. | S      |
| Dead style keys         | With the files above: `bidCard*`/`bidActions*`/`bidCarousel*`/`bidControls*`/`bidList*`, `promoCardFeatured*`, `tradeListItem*`, `shareImageGenerator*` in `cards.ts`; `accountToggleSwitch*` in `buttons.ts`; `profileConnectButton*` in `messages.ts`.                                                                                                                                                                                                                      | S      |
| Dead assets             | Delete `profile.png`, `featuredCard-001.jpg`, `backdrop.svg`, `icon-master.png`, `collected.ai`, `lottie/heart.json`, `lottie/loading.json`, `lottie/splash.json`, `icons/loading-2.lottie` (code comments say do-not-use), 10 orphan SVGs (`sun`, `moon`, `home-depr`, `pencil-2/3`, `more-vertical`, `gear-outline`, `chevron-left-2`, `logo-icon`, …). Keep `fab-logo@2x/@3x` (density variants). ~5.6 MB.                                                                 | S      |
| Dead `['Feed']` patches | Delete the `setQueriesData` blocks in `BulkTab.tsx:305` and `EditBulkScreen/index.tsx:40` (~60 lines, rule-26 violations against a retired key).                                                                                                                                                                                                                                                                                                                              | XS     |
| Repo hygiene            | Delete root `.plans/` (migrate anything still relevant here first), `plan.md`, `compact-summary.md`, empty `solution-designs/`, `scripts/node_with_polyfills.sh`; pick npm and delete `yarn.lock` (README dual-instructions too); `.nvmrc` → `22`; `git rm -r --cached vendor/bundle` (27 tracked files, 3.5 MB, ignore rule added after commit).                                                                                                                             | S      |
| Native leftovers        | Remove `ios/GoogleService-Info.plist` from the Xcode project (no Firebase SDK anywhere; last trace of removed push); remove `NSLocationWhenInUseUsageDescription` (no location code — App Store review flag).                                                                                                                                                                                                                                                                 | S      |

### Bundle B · Fix now, small code changes, behaviour-preserving (needs P0-rule-7 approval: touches `src/styles/utils/*`)

| Item | Detail | Effort |
|---|---|---|
| Memoize style getters | New `src/styles/utils/memoizeByTheme.ts` (WeakMap keyed on theme identity); wrap each of the 37 `export const getXStyles = memoizeByTheme((theme) => ({…}))`. No call-site changes. Delete or adopt the orphan `useThemeStyles.ts`. | S |
| Memoize `ThemeProvider` value | `useMemo(() => ({ theme, themeMode, setThemeMode }), [theme, themeMode])`. | XS |
| `React.memo` the list rows | `FeedListingCard`, `ListingCard`, `PostItem`, `FeedBidCard`, `FeedBulkCard` (only effective once getters are memoized — land together); `useMemo` the two `.filter` calls in `PostItem`. | S |
| `LoadingSpinner` off the JS thread | It animates `height`/`y` with `useNativeDriver: false` in an infinite loop on the cold-start path. Drive `scaleY`/`translateY` with the native driver instead. | S |
| `ListingQuantityProvider` | Memoize value + `useCallback` setters, or move the provider down to its single consumer. | XS |

### Bundle C · Fix now, data-layer correctness (small, contained)

| Item | Detail | Effort |
|---|---|---|
| Offer invalidation keys | Replace the four dead keys in `OfferMessageDetailScreen:110–116` and `MakeOfferScreen:253–258` with `BuyerOrders`/`SellerOrders` (lazy), `ProductListings`/`ProductBids`/`listingDetail`/`bidDetail` (`refetchType:'active'`), `AccountBids`. | S |
| `UserBulkListings` | Point the 7 invalidations at the keys the bulk surfaces actually read (`UserSellFeed`, `UserProfileListings`, `queryLists` lazily) or delete them. | S |
| `PaymentMethods` shape | One `useFetchPaymentMethods` hook; the two offer screens stop redeclaring the query with a different return shape. | S |
| `['group', groupId]` include | Make `GroupMembersScreen` request the same `include` set as `GroupDetailScreen` (or share the hook). | XS |
| Image widths | Pass `IMG_WIDTH.CARD` / `THUMB` / `BANNER` at the 51 width-less `getImage()` sites that render tiles, avatars and banners (feed cards, listing cards, inbox rows, `Badge/Profile`, group member rows, share banners, friend avatars). Detail heroes stay width-less by design. | M |
| `profileSocialLinks` | One shared hook for the two identical declarations; give it a `staleTime`. | XS |

### Bundle D · Fix now, design-system cleanup (~7 files, ~255 lines)

The orders/reviews cluster (`OrderCard`, `WriteReviewModal`, `OrderDetailScreen`,
`BuyerOrders`/`SellerOrders`) holds nearly every remaining P0 violation: all 8 component-level
hex values (star amber/grey), 4 of 5 raw icon sizes, 6 `as any`, 3 copies of the same
star-rating block.

| Item                                                         | Detail                                                                                                                                                                                                                                                              | Effort |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| `theme.colors.rating.{filled,empty}` tokens + `<StarRating>` | Replaces 3 hand-rolled blocks; kills the hex, the raw sizes, the inline styles.                                                                                                                                                                                     | S      |
| `OrderDto.review`                                            | The 6 `(order as any).review` casts are a missing relation in `@oakplatforms/types` — bump the package (nanza-api side), don't patch locally (rule 11).                                                                                                             | S      |
| Merge `BuyerOrders`/`SellerOrders`                           | 78-line twins with a 4-line diff → `<OrdersList role>`; clears 2 `display-name` lint errors.                                                                                                                                                                        | S      |
| Gradient scrim tokens                                        | `scrimTop`/`scrimBottom` in `gradients.ts` replace the 9 live `rgba()` literals repeated in 7 files.                                                                                                                                                                | S      |
| Remaining singles                                            | `posts.ts:1282` hex fallback; `posts.ts:1230` `#FFFFFF`; `cards.ts:6260` `shadowColor`; `QuantityStepper` inline `StyleSheet.create`; `AccordionTagSection` 4 inline font props; `messages.ts:521–530` font props; `App.tsx` `StatusBar backgroundColor="#000000"`. | S      |
| `Page<T>` type                                               | Replaces 7 duplicate `OrderPage`/`BidPage`/… interfaces.                                                                                                                                                                                                            | XS     |
| `LoadingSpinner` adoption                                    | 32 files use raw `ActivityIndicator` while the primitive exists.                                                                                                                                                                                                    | S      |
| Fix the 33 real type errors                                  | `Tabs` props, `groupIds`→`groups`, implicit `any`s.                                                                                                                                                                                                                 | S–M    |

### Later release · needs its own QA pass or a visual regression sweep

| Item | Why later | Effort |
|---|---|---|
| Style-bucket dedupe | 621 byte-identical classes (~3,150 lines, 10.5% of style code) + 2,555 near-duplicates; the `bidCard*`/`listingCard*` twin family (72 twins, 59 identical) and 35 base names × 5 size suffixes (222 classes → ~20 with lookup maps). Touches `cards.ts`, which backs every feed and detail surface. Run `react-native/no-unused-styles` first so the dedupe does not preserve dead classes. ~4,650 lines removable. | L |
| Remove dead fonts | Drop 29 `.ttf` files; needs a native build to confirm the asset link step. Also resolve the `.cursorrules` "Figtree `$` price font" convention whose tokens never existed (implement or delete the section). ~3 MB per binary. | M |
| Image assets | Re-export `splash-backdrop.jpg` (2.8 MB, cold path) and `auth-backdrop.jpg` as optimized JPEG/WebP. | S |
| Android R8 | `enableProguardInReleaseBuilds = true` + `shrinkResources true`; test a release build for over-stripping. Confirm distribution is AAB-only (all 4 ABIs are built, no splits). | M |
| Image caching | 93 plain `<Image>`, no `expo-image`/FastImage; list tiles re-decode on scroll-back. | M |
| `fetchData` timeout + abort | No `AbortController` anywhere; React Query's `signal` is never forwarded. Add `AbortSignal.timeout` default and thread `signal` through `FetchProps`. P0 rule 7 escalation (shared infra). | M |
| Lint stack upgrade | ESLint 9 + typescript-eslint 8 (TS 5.9 support) + `eslint-plugin-react-native` (`no-color-literals` can go straight to `error` after Bundle D; `no-inline-styles` as `warn`). Extend `no-restricted-syntax` for raw bucket sizes (579), raw icon sizes, raw scale refs in components (90). | M |
| CI + pre-commit | No `.github/`, no husky. Add lint + `tsc --noEmit` gates once the baseline is green. | S |
| `LoadingSpinner` over raw `ActivityIndicator` (27 files) | Visible change from platform spinner to the three-bar animation — design call first. | S |
| `Page<T>` / `PaginatedResponse<T>` | Shared type has nullable `page`/`total`; adopting it means null-guarding every pager. | S |
| `Header` scroll-to-active-tab | The logic exists but Tabs never reports layouts; either expose `onLayout` from Tabs or delete the dead branch. | S |
| `SPLASH_MIN_MS = 2500` | Hard 2.5 s floor on every launch; gate on the burst animation instead, or drop to ~1.2–1.5 s. Product call. | XS |
| List tuning | `getItemLayout` on the fixed-width carousels (1 of 24 lists has it); `CollectionPickerScreen` maps unbounded data in a `ScrollView`; two near-duplicate `CardCarousel` components. | M |
| Version drift | `@stripe/stripe-react-native` 0.55 → 0.76 (payment-critical), `react-native-vision-camera` 4.7 → 5.2, RN 0.83.1 → 0.83.10 patch, reanimated 4.3 → 4.6 with worklets. | M |
| Legacy API tidy | 2 files import `SafeAreaView` from `react-native`; `UIManager.setLayoutAnimationEnabledExperimental` in `DetailPager` (no-op on Fabric); `CardStyleInterpolators` in 3 navigators (nav is read-only — escalate); 22 `as unknown as` casts; 34 untyped `useNavigation()`; `Touchable*` outnumbers `Pressable` 10:1. | M–L |
| `PostComposer.tsx` split | 1,639 lines, 4× the next-largest component, 8 inline styles. | L |
| `SessionContext` session read | Outside React Query, no cache/dedupe, called from 23 `refreshCurrentUser` sites — already named the next target in [[app-load-performance]]. | M |
| Duplicated utils | `formatPrice` ×3 (`formatPriceAsDollar` is canonical), ad-hoc `clamp` ×6, `COLLECTION_PAGE_SIZE` hardcoded 20 vs 21 across three pagers. | S |
| Freshness tier mismatch | Homepage shelves use `5_MINS`; [[data-freshness]] says `30_SECS` (which has one user app-wide). Decide which is right. Messaging queries stack three refresh mechanisms (socket + focus refetch + `refetchOnWindowFocus`). | XS |

## What landed (2026-09-16, same day)

Skylar approved Bundles A–D; all four are built in the mobile repo and left uncommitted for review.
End state: `npx tsc --noEmit` **60 → 0 errors**, `npm run lint` **254 → 0 errors** (31
`react-hooks/exhaustive-deps` warnings remain, expected on first enablement), `assets/` 14 MB → 8.4 MB,
222 files changed (+1.1k / −24.6k lines).

**Deviations from the plan, and why**
- `buffer` stays: `react-native-svg` imports it without declaring it — Metro failed to resolve it the
  moment it was uninstalled. knip/depcheck false positive. `react-native-qrcode-svg` went instead (its
  only consumer was the dead `QrCode` component).
- `LoadingSpinner` adoption over the 27 raw `ActivityIndicator`s was **deferred**: it swaps small inline
  spinners for the three-bar animation, which is a visible design change, not a cleanup. Needs a
  design call (moved to Later).
- `Page<T>` consolidation **deferred**: the shared `PaginatedResponse<T>` types `page`/`total` as
  nullable, so adopting it ripples null checks through every pager. The 5 local page interfaces stay.
- `amplify/` and `scripts/figma-token-sync.js` untouched — both need Skylar's answer (Decisions 2, and
  the figma-sync skill still references the script).
- The `fetchX` React Query hooks (17) were renamed `useFetchX` to satisfy `rules-of-hooks`; files keep
  their `fetchX.ts` names per the REFERENCE convention.
- `Header`'s tab-layout handler was removed (Tabs never called it); the scroll-to-active-tab logic
  it fed was already inert at runtime and stays that way, now with a comment. Wiring it up is a
  Later item.
- `GroupDetailScreen` was added to `RootStackParamList` (type-only; the navigator files are untouched)
  so typed navigators can reach an already-registered screen.
- Bundle B touched `src/styles/utils/*` (memoizeByTheme, ThemeProvider value memo) and Bundle D added
  `rating`/`scrim` tokens through `resolveThemeColors.ts`/`types.ts` — P0 rule 7 areas, done under
  Skylar's bundle approval.
- `git rm --cached vendor/bundle` is **staged** (the only way to untrack it); everything else is an
  unstaged working-tree change.

**QA checklist before release** (behaviour is meant to be identical; these are the spots to eyeball)
1. Cold start + home feed scroll — style getters are memoized and `PostItem` is `React.memo`; expect
   smoother scroll, no visual change.
2. `LoadingSpinner` — now `scaleY`/`translateY` on the native driver; bars should look the same.
3. Status bar — moved inside `AppRoot`, colour from `theme.colors.background` (was `#000000`).
4. Orders: `OrdersList` replaces `BuyerOrders`/`SellerOrders`; star ratings on order cards, order detail
   and the write-review modal now come from `<StarRating>` with `rating.filled/empty` tokens.
5. Gradient scrims on tag/group/collection/profile heroes and share cards use the new
   `scrimTop`/`scrimBottom` tokens.
6. Offers: accept/decline/send now refresh orders, bids and price surfaces (they refreshed nothing).
7. Bulk create/edit/delete: the dead `UserBulkListings` invalidations are gone; profile listings, sell
   feed and trade feed still refresh.
8. Group detail ↔ members: one shared `useFetchGroup` with the full include set (social links show
   from either entry point).
9. Payment methods on Wallet and both offer screens share one hook (same key, one shape).
10. Avatars, row thumbnails, member rows, share banners and scan variant tiles now request a CDN width
    bucket — check they render crisp at their sizes.
11. iOS: `GoogleService-Info.plist` removed from the Xcode project; location usage string removed.
    Run `pod install`.
12. Fonts were NOT touched (still Later).

## Decisions needed (Skylar)

1. **Approve Bundle B** — it edits `src/styles/utils/*` (P0 rule 7) and `ThemeProvider`.
   Additive, no API change, but it is the shared infrastructure the rule protects.
2. **`amplify/`** — zero references from the app; is an Amplify backend still deployed from
   this config? If not, delete the folder.
3. **Figtree "$" price font** — the `.cursorrules` convention was designed but never wired
   (no `price*` tokens exist). Implement it, or delete the fonts and the rule text.
4. **i18n scope** — auth-only localization is the current reality. Intentional, or a backlog
   of ~1,000 strings?
5. **`OrderDto.review`** — needs a `@oakplatforms/types` bump from nanza-api before the
   orders cleanup can drop its `as any` casts.
6. **`enableProguardInReleaseBuilds`** — flip it on for the next release build and QA, or
   defer.
7. **Splash minimum** — keep the 2.5 s floor as a brand moment, or shorten.
8. **`fetchData` timeout/abort** — approve the shared-infra change for a later release.

## Enforcement — stopping the drift

The authored code is close to green. After Bundles A + D the lint baseline is essentially
zero in `src/`, which is the precondition for gating. Recommended order:

1. Bundle A config fixes (tsconfig, ignores, react-hooks `warn`).
2. Bundles B–D.
3. ESLint 9 + typescript-eslint 8 + `eslint-plugin-react-native`; `no-color-literals: error`,
   `no-inline-styles: warn`, `no-unused-styles: warn`, `react-hooks/exhaustive-deps: warn` →
   `error` as sites are cleaned.
4. A CI workflow running `npm run lint` and `npx tsc --noEmit`; husky + lint-staged locally.

## What is already in good shape (do not "fix")

- Cold start: 6 requests signed-in, 5 anonymous (+1 guest-token mint), matching
  [[app-load-performance]]; the three-request gate is exact; no auth-only request fires for
  guests; mirrored query keys in `useHomeReady` dedupe correctly.
- Freshness: rule 29 (discovery surfaces lazy) is clean across all 50 `refetchType:'active'`
  sites; the one unfiltered `invalidateQueries()` in `AccountScreen/Profile:101` is
  deliberate and documented — leave it.
- Realtime: one socket, key-nudge frames, gated off the cold path, jittered backoff.
- Analytics: 21 explicit user-action events, no impression/scroll tracking, rate-limited.
- Zero `console.log`, zero module-level side effects, zero class components, zero named
  colors, gesture-handler fully on the `Gesture` API, listener/timer cleanup solid, Hermes +
  New Architecture on both platforms, `HomeScreen` scroll natively driven.
- Component structure: 163 folder-components, uniform default exports, `*Props` naming,
  `components/global/` well stocked (49 primitives) — the gap is adoption, not absence.

## Related

- [[design-system]] · [[data-freshness]] · [[app-load-performance]] · [[docs-infra]]
- [[../audits/2026-09-16/INDEX|Raw audit reports (2026-09-16)]]
- nanza-api side: `OrderDto.review` types bump; the `t(order.name)` i18n bug is mobile-only.
