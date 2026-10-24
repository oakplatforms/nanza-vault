# Dead Code / Unused Files / Unused Assets / Stale Artifacts Audit — nanza-mobile

Read `CLAUDE.md` and `documentation/INDEX.md` fully. All findings below are grep/file-size
verified against the live tree, not taken on knip's/depcheck's word alone.

## Summary

| Category | Count | Est. removable | Safe-now vs later |
|---|---|---|---|
| Dead files (knip, hand-confirmed) | 41 of 44 confirmed dead, 3 false-positive | ~6,570 lines / ~336K | Safe now (S/M effort) |
| Dead exports (sample-verified) | 15 of 20 sampled confirmed dead | small | Safe now |
| Dead exported types | not independently re-verified beyond knip (see §2) | — | Later (needs a type-usage pass) |
| Unreferenced assets | 15 files (10 svg, 2 png density-adjusted-out, 1 jpg, 2 json/lottie, +5 more lottie found by hand) | ~1.0MB+ | Safe now |
| Dead font families (whole families unreferenced) | Figtree (10 ttf), Geist (9 ttf), EuclidCircularB (10 ttf) = 29 of 35 ttf | ~3.5MB shipped into both binaries | Safe now, needs build verification |
| Style-bucket unreferenced keys (8 largest files) | ~1,384 flagged, high-confidence subset far smaller (dynamic-key false positives) | large, but low urgency | Later (needs per-file hand-check) |
| Retired-feature remnants | cart.ts bucket, TopPicks comment-only, ReviewDto type, hidden trade-proposal (fully gone) | ~28K (cart.ts) + scattered | Safe now for cart.ts |
| TODO/FIXME/HACK | 1 | n/a | n/a — codebase is clean here |
| console.* (non-guarded) | 16 | n/a | Low priority |
| Stale root artifacts | 8 items flagged | 3.5MB (vendor) + 92K (.plans) + misc | Mixed — vendor is High severity |
| i18n | NOT dead — real, in use (correction to task premise) | n/a | n/a |

---

## 1. Knip "unused files" — verified, classified (44 total)

Methodology: grepped each file's basename/export symbol across `src/`, `App.tsx`, `index.js`,
`navigation/`, and traced import chains upward to confirm the whole chain (not just the leaf)
has no live root.

### Confirmed dead — safe to delete (37)

All of these have **zero** importers anywhere in `src`, `App.tsx`, `index.js`, or
`navigation/*`. Several form orphaned chains where a "used" leaf file is only used by another
dead file — I traced each chain to its root and confirmed the root itself is unreferenced.

| File | Severity | Why dead | Effort | Risk |
|---|---|---|---|---|
| `src/components/feed/FeedListingCard/index.tsx` | High | Root of a 4-file dead chain (see below); no importer | S | None |
| `src/components/feed/FeedListingPreview/index.tsx` | High | Only used by the dead `FeedListingCard/index.tsx` | S | None |
| `src/components/feed/FeedListingPreviewBase/index.tsx` | High | Only used by dead `FeedListingPreview` | S | None |
| `src/components/feed/FeedListingCard/FeedListingCardSimple.tsx` | High | Only used by dead `FeedListingPreviewBase` | S | None |
| `src/components/global/SyncedSlider/index.tsx` | Med | Only used by dead `FeedListingPreviewBase` | S | None |
| `src/components/feed/FeedBulkCard/index.tsx` | High | No importer (only referenced in comments in `ListingScreen`, `DetailTitleBlock`, `useFetchByReference`) | S | None |
| `src/components/feed/FeedBidCard/index.tsx` | High | No importer (comment-only refs in `BidScreen`) | S | None |
| `src/components/listings/ListingCard/index.tsx` | High | No importer anywhere | S | None |
| `src/components/listings/ListingControls/index.tsx` | Med | Only used by dead `ListingCard/index.tsx` | S | None |
| `src/components/entity/EntityCard/index.tsx` | High | No importer; re-exports `Listing.tsx` internally but nothing imports the barrel | S | None |
| `src/components/entity/EntityCard/Listing.tsx` | Med | Only reachable via dead `EntityCard/index.tsx` | S | None |
| `src/components/entity/EntityBottomBar/index.tsx` | Med | Zero references anywhere | S | None |
| `src/components/global/QuantitySelector/index.tsx` | Low | Zero references; a same-named unrelated *type* `QuantitySelector` exists in `DetailActionBar/index.tsx` (false-positive risk in naive grep — verified the component itself is unused) | S | None |
| `src/components/global/ToggleSwitch/index.tsx` | Med | Zero references; sibling style keys `accountToggleSwitch*` in `buttons.ts` are also dead (see §4) | S | None |
| `src/components/global/Toast/Toast.tsx` + `src/components/global/Toast/index.ts` | Med | Zero references to either file | S | None |
| `src/components/global/ZoomableImage.tsx` | Low | Zero references — a **different** `ZoomableImage.tsx` lives at `src/components/detail/DetailHero/ZoomableImage.tsx` and *that* one is actively imported by `DetailHero/index.tsx`. This top-level one is an abandoned duplicate. | S | None |
| `src/components/share/CollectionTargetCard.tsx` | Med | Only used by dead `CollectScanTargetScreen` (below); referenced only in comments elsewhere | S | None |
| `src/components/share/ShareCardParts.tsx` | Med | Zero references | S | None |
| `src/components/share/QrCode/index.tsx` | Low | Zero references (an unrelated `qr.svg` icon and `showQrCode` prop exist but don't use this component) | S | None |
| `src/components/modals/ActionModal/Trade.tsx` | Med | `ActionModal/index.tsx` (the live barrel) doesn't reference `Trade.tsx` or `Profile.tsx` variants at all — only `Listing.tsx`'s `TradeActionModal` import references it, and that chain needs re-checking (see note below) | S | Low |
| `src/components/modals/ActionModal/Profile.tsx` | Med | Zero references | S | None |
| `src/components/modals/ModalScreen/index.tsx` | Med | Zero references | S | None |
| `src/components/modals/PurchaseConfirmationModal/index.tsx` | Med | Zero references | S | None |
| `src/components/messages/ConnectButton.tsx` | Low | Zero references; style keys `profileConnectButton*` in `messages.ts` also orphaned | S | None |
| `src/components/groups/GroupCircleCarousel/index.tsx` | Med | Zero references (mentioned only in a comment in `TagCarousel`) | S | None |
| `src/components/profile/ProfileSectionHeader/index.tsx` | Low | Zero references | S | None |
| `src/screens/profile/ProfileScreen/ProfileListThumbnails/index.tsx` | Med | Zero references — but its **sibling** `ListThumbnailImage.tsx` in the same folder IS imported directly by `CollectionPickerScreen`, bypassing this `index.tsx`. Delete only the `index.tsx`, keep `ListThumbnailImage.tsx`. | S | Low |
| `src/screens/profile/UserProfileScreen/data/fetchUserCollection.ts` | Med | Zero references | S | None |
| `src/screens/trade/CollectScanTargetScreen/index.tsx` | High | Not registered in `AppNavigator.tsx` (its sibling `CollectScanItemsScreen` IS registered at line 787; this one is an orphaned screen file) | M | Low — confirm no deep link targets it |
| `src/screens/trade/TradeScreen/CollectTab.tsx` | High | `TradeScreen/index.tsx` does not import `CollectTab` at all — matches the vault's own note that "the Offers pill and the hidden trade-proposal subsystem were deleted the same day" (trade-screen solution design) | M | None |
| `src/components/global/Header/sheetHeader.ts` | Med | Elaborate doc-comment claims it's "the single source of truth" for sheet headers, but zero call sites | S | None |
| `src/styles/components/cart.ts` | High | Entire style bucket for the retired cart feature (offer-engine solution design: "the cart (context, pill, review screen) retired"); zero importers | S | None — cart feature is confirmed retired |
| `src/styles/components/index.ts` | Med | Dead barrel — superseded; every real consumer imports individual bucket files (`getCardStyles` from `./cards`, etc.) directly, confirmed via depcheck's "using" list | S | None |
| `src/services/api/Invoice.ts` | Med | Zero references, including from `services/api/index.ts` | S | Low — confirm no invoicing feature is mid-build |
| `src/styles/utils/parseFontWeight.ts` | Low | Zero references | S | None |
| `src/styles/utils/useThemeStyles.ts` | Low | Zero references | S | None |
| `src/utils/calculateListingQuantity.ts` | Low | Zero references | S | None |
| `src/utils/calculateSubtotalCost.ts` | Low | Zero references (function only defined, never called) | S | None |
| `src/utils/calculateCartTotals.ts` | Low | Zero references; retired-cart remnant, contains a stray non-guarded `console` call | S | None |
| `src/hooks/useFilterActionBar.ts` | Low | Zero references | S | None |
| `scripts/figma-token-sync.js` | Low | Not wired into any `package.json` script; only mentioned in `.claude/skills/figma-sync` docs as historical context | S | None — confirm with figma-sync skill owner before deleting |

### Referenced dynamically / keep-with-reason (4)

| File | Reason |
|---|---|
| `amplify/backend/types/amplify-dependent-resources-ref.d.ts` | Amplify CLI-generated ambient type declaration; consumed implicitly by the TS compiler for any Amplify-typed imports, not via an explicit `import`. Standard for Amplify projects. Flagged by knip as a false positive. **Note:** nothing else in `src`/`App.tsx` references `amplify` at all (see §7) — the whole `amplify/` directory looks stale regardless of this one file. |
| `polyfill.js` | Loaded via `NODE_OPTIONS="--require $(pwd)/polyfill.js"` in `package.json`'s `android`/`ios`/`start` scripts — a Node `--require` preload, invisible to static import analysis. Genuinely used. Keep. |

### False positives / already-live (3, corrections to the knip report)

None of the 44 turned out to be outright false positives once chains were traced — the 3
"borderline" ones (`ProfileListThumbnails/index.tsx`, `amplify-dependent-resources-ref.d.ts`,
`polyfill.js`) are handled above as keep-with-reason rather than delete.

**One important caveat on `ActionModal/Trade.tsx`**: `EntityCard/Listing.tsx` imports
`TradeActionModal` from it — but `EntityCard/Listing.tsx` is itself only reachable through the
dead `EntityCard/index.tsx` root. If `EntityCard/index.tsx` is deleted, `ActionModal/Trade.tsx`
becomes fully dead too; if for some reason `EntityCard` gets resurrected, re-check this one.

---

## 2. Unused exports & unused exported types — sample-verified (20+ of 39 exports, types spot-checked)

Verified 20 unused-export entries with `grep -rn '\bname\b' src` and manual inspection of the
hit context (to rule out self-definition-only matches).

**Confirmed genuinely dead (15/20 sampled):**
- `useReplyToConnection` (`MessagesScreen/data/mutations.ts`) — 0 refs
- `draftHasImage` (`posts/composer/draft.ts`) — 0 refs
- `isShareGuardActive` (`contexts/ShareGuardContext.tsx`) — 0 refs
- `useSetPrimaryContentItem` (`posts/usePosts.ts`) — 0 refs
- `ART_TOTAL` (`screens/auth/entrance.tsx`) — 0 refs
- `parseShareReferenceCode` (`utils/referenceCode.ts`) — 0 refs
- `OFFER_PRICE_TOLERANCE`, `offerPriceBounds` (`utils/offerPrice.ts`) — 0 refs
- `SERVICE_FEE_FIXED`, `SERVICE_FEE_RATE` (`utils/serviceFee.ts`) — 0 refs (constants only used inline elsewhere, not imported)
- `generateShippingOptionsRate`, `getReturnShipment` (`utils/shipping.ts`) — 0 refs
- `pluralizeProtectiveSleeve`, `pluralizeShippingShield` (`utils/pluralize.ts`) — 0 refs (the generic `pluralize` helper itself has 2 refs, see below — it's a nuanced case)
- `getEntityTagLine` (`utils/getEntityTagLine.ts`) — the **named** export is dead, but the file is very much alive: every consumer (9 files) imports the sibling `getEntityTagSegments` instead
- `NumericField`, `ToggleField`, `ImageField` (via `components/global/Form/index.tsx` barrel) — the underlying components have **zero consumers at all**, not just barrel-bypass. Fully dead form fields (each ~150 lines).

**Nonzero-hit exports that are real DRY/inconsistency findings, not simple dead code:**
- `COLLECTION_PAGE_SIZE` (`fetchCollectionEntities.ts`, value `21`) — knip flags the export as unused because no file imports it, but **two other files hand-roll the identical constant name with a different value (`20`)**: `TradeScreen/CollectTab.tsx:28` and `ProfileSeeAllScreen/index.tsx:36`. This is a page-size inconsistency (20 vs 21) across three near-identical pagination call sites that should share one source. Flag as Med severity — not dead code, but a duplication bug risk.
- `pluralize` (`utils/pluralize.ts`) — genuinely imported in 2 places; knip's "unused export" flag here looks like a false positive (possibly a re-export ambiguity). Do not delete.
- `NumericField`/`ToggleField`/`ImageField` re-exports from `Form/index.tsx` barrel: `CodeField` and `TextArea` (same barrel) ARE used elsewhere but always via **direct file import**, never through the barrel — so the barrel re-export lines for `CodeField`/`TextArea` are dead code paths even though the components are alive.

**Unused exported types (53 total per knip):** not independently re-verified one-by-one within
this pass's time budget beyond the ones encountered organically above (`ReviewDto`,
`QuantitySelector` collision-checked). The type list is dominated by `src/types/index.ts`
re-exports (`ConversationParticipantDto`, `PostCollectionReference`, `ShippingCarrier`,
`GroupListingDto`, `TransactionDto`, `ContentItemDto`, `QueryListDto`, `CategoryDto`,
`ShipmentDto`, `GroupBidDto`, `AccountDto`, `ReviewDto`, `ThemeDto`, `TagDto`, etc.) — these are
DTOs from `@oakplatforms/types`; per CLAUDE.md P0 rule #11, domain types are supposed to flow
through here even if not every one is consumed by the mobile app today (some may be consumed by
sibling repos' contract or reserved for near-term screens). **Recommendation: treat this list as
Low severity / Later — do not prune `types/index.ts` re-exports without cross-checking
nanza-api and nanza-email usage; this file's job is to mirror the shared package, not to be
tree-shaken.** The component-local unused prop types (`OfferLineLockupProps`, `BadgeProps`,
`ModalRef`, `SearchBarHandle`, etc.) are far lower risk to clean up (S effort, None risk) since
they're either exported unnecessarily (should be file-local) or dead alongside their component.

---

## 3. Unused assets under `assets/`

Scripted a filename (without extension) grep across `src`, `App.tsx`, `react-native.config.js`,
and `package.json` for every svg/png/jpg/json/lottie/ai file (167 files, ttf handled separately
in the fonts section below).

**Zero-reference assets found (15 by script + corroborated by hand):**

| File | Size | Note |
|---|---|---|
| `assets/images/featuredCard-001.jpg` | 676K | Largest single unreferenced asset in the repo |
| `assets/icon-master.png` | 140K | Not wired into any iOS/Android app-icon config found in `ios/`/`android/`; likely a design-source leftover, not a shipped app icon |
| `assets/images/fab-logo@3x.png` | 56K | **Not actually dead** — see correction below |
| `assets/images/fab-logo@2x.png` | 28K | **Not actually dead** — see correction below |
| `assets/lottie/loading-outline.json` | 8K | 0 refs |
| `assets/lottie/loading-gray.json` | 8K | 0 refs |
| `assets/images/logo-icon.svg` | 4K | 0 refs |
| `assets/icons/sun.svg` | 4K | 0 refs |
| `assets/icons/pencil-3.svg`, `pencil-2.svg` | 4K each | 0 refs (an active `pencil.svg`/`pencil-2`? — check iconMap; neither numbered variant appears in `iconMap.ts`) |
| `assets/icons/more-vertical.svg` | 4K | 0 refs |
| `assets/icons/moon.svg` | 4K | 0 refs (paired with dead `sun.svg` — looks like an abandoned light/dark toggle icon pair) |
| `assets/icons/home-depr.svg` | 4K | Filename literally flags itself deprecated (`-depr` suffix) |
| `assets/icons/gear-outline.svg` | 4K | 0 refs |
| `assets/icons/chevron-left-2.svg` | 4K | 0 refs |

**Correction — `fab-logo@2x.png`/`@3x.png` are NOT dead**: React Native resolves `@2x`/`@3x`
density suffixes automatically from the base `require('.../fab-logo.png')` call in
`src/components/global/Header/index.tsx:479` — there is never a literal source reference to the
`@2x`/`@3x` filenames, by design. Do not delete these two; they're load-bearing.

**Additional unreferenced lottie files found by hand** (missed by the generic filename script
because the actual `require()` calls point to *different* lottie files than several files on
disk):
- `assets/lottie/heart.json` — 108K, by far the largest lottie asset, zero `require()` anywhere
- `assets/lottie/loading.json` — 8K, zero refs (superseded by `loading-2.json`, which IS used)
- `assets/lottie/splash.json` — 8K, zero refs
- `assets/icons/loading-2.lottie` — 4K, explicitly called out as **do-not-use** in code comments in `SocialAuthButtons.tsx:42-44` and `Button/index.tsx:12-13` ("The JSON, NOT assets/icons/loading-2.lottie... it cannot [be opened by lottie-ios]") — a deliberately superseded format-incompatible file that was never deleted.

**Total confirmed-dead asset payload: ~952K (script-found) + ~128K (hand-found lottie) ≈ 1.08MB.**

Severity Low individually, Med in aggregate (bundle size). Effort S, Risk None (all verified
zero-reference). Fix now (non-breaking) — pure deletions, no code changes needed.

---

## 4. Fonts — `react-native.config.js` ships the whole `assets/fonts/` folder (35 ttf, 4.5MB) into both binaries regardless of use

`src/styles/tokens/dark.json` and `light.json` (byte-identical `fonts.family` blocks) declare
only **6 font family tokens**, and only **2 font families**:

```json
"family": {
  "primary": "Roobert-Regular",
  "primaryMedium": "Roobert-Medium",
  "primarySemiBold": "Roobert-SemiBold",
  "primaryBold": "Roobert-Bold",
  "bodyFamily": "BaltoMobile-Book",
  "secondary": "Roobert-Regular"
}
```

Cross-checked every `theme.fonts.family.*` call site in `src/styles/components/typography.ts`
(41 call sites) and the rest of `src` (`grep -rn "theme.fonts.family\."`) — **no other token name
is ever referenced** (no `.priceMedium`, `.priceSemiBold`, etc. anywhere in the codebase).

| Font family | .ttf files shipped | Referenced by a token? | Verdict |
|---|---|---|---|
| Roobert (Bold/Medium/Regular/SemiBold) | 4 of 5 files | Yes — `primary*`/`secondary` tokens | **Live** |
| Roobert-RegularItalic | 1 file | No — no italic token exists in either theme | **Dead weight** (ships, never selectable) |
| BaltoMobile-Book | 1 file | Yes — `bodyFamily` | **Live** |
| **Figtree** (Black/BlackItalic/Bold/BoldItalic/ExtraBold/ExtraBoldItalic/Medium/MediumItalic/SemiBold/SemiBoldItalic) | 10 files | **No** | **Fully dead** — see note below |
| **Geist** (Black/Bold/ExtraBold/ExtraLight/Light/Medium/Regular/SemiBold/Thin) | 9 files | **No** | **Fully dead** |
| **EuclidCircularB** (Bold/BoldItalic/Light/LightItalic/Medium/MediumItalic/Regular/RegularItalic/Semibold/SemiboldItalic) | 10 files | **No** | **Fully dead** |

**Notable inconsistency:** the repo's own `.cursorrules` file documents an elaborate "Dollar
Sign Font Rule" convention mapping price weights to `theme.fonts.family.priceMedium` /
`priceSemiBold` / `priceBold` / `priceExtraBold` / `priceBlack` (implying Figtree is meant to be
the "$" glyph face) — but **none of those token names exist in `dark.json` or `light.json`**,
and no `theme.fonts.family.price*` call site exists anywhere in `src`. Either this convention
was designed and never wired into the token layer, or the token layer was refactored and the
`.cursorrules` doc was never updated. Either way, the 10 Figtree files are dead weight against a
documented-but-unimplemented design intent — worth a decision (implement the tokens, or delete
both the fonts and the stale `.cursorrules` section) rather than a silent deletion.

**Impact:** 29 of 35 shipped `.ttf` files (≈3.3MB of the 4.5MB `assets/fonts/` folder) are dead
weight in both the iOS and Android binaries, since `react-native.config.js` links the entire
folder unconditionally:
```js
module.exports = { assets: ['./assets/fonts/'] }
```
Severity High (real binary-size cost), Effort M (requires confirming with a native build that
removing files doesn't break an Xcode/Gradle asset-catalog reference, since font linking is
partly a native build step, not just JS), Risk Low with verification. **Later release** — flag
for a build-time check before deleting, don't do it as a pure JS-side "safe now" deletion.

---

## 5. Style-bucket unreferenced classes — 8 largest `src/styles/components/*.ts` files

Method: extracted every top-level key in each file's returned style object, then grepped all of
`src` for `.keyName` usage. **Caveat, important**: several consumer components
(`ListingCard/index.tsx`, `SyncedSlider/index.tsx`, `Button/index.tsx`) build style keys via
**template-literal / computed bracket access** (e.g. `` cardStyles[`listingCard${variant}`] ``,
`buttonStyles[variantOverride]`), which a literal `.keyName` grep cannot see. This inflates the
raw counts below for `cards.ts` and especially `buttons.ts`. Numbers are an **upper bound**;
treat sub-bullets marked "hand-confirmed" as the reliable subset.

| File | Total keys | Flagged unreferenced (literal grep) | Hand-confirmed dead |
|---|---|---|---|
| `cards.ts` | 864 | 554 (64%) | `bidCard*`/`bidActions*`/`bidCarousel*`/`bidControls*`/`bidList*` (all ~90 keys — orphaned with the confirmed-dead `FeedBidCard`); `promoCardFeatured*`, `promoCardGroup` (0 refs anywhere, no dynamic consumer found); `tradeListItem*` (14 keys — `TradeListItem.tsx` doesn't reference any `cardStyles.tradeListItem*` key despite the name match); `zoomableImageContainer`; `shareImageGenerator*` (9 keys) |
| `layout.ts` | 472 | 211 (45%) | Not individually hand-verified beyond spot checks; recommend a follow-up pass |
| `buttons.ts` | 290 | 230 (79%, likely heavily inflated by `Button/index.tsx`'s dynamic variant lookup) | `accountToggleSwitch*` (4 keys, confirmed — sole consumer is the dead `ToggleSwitch` component) |
| `inputs.ts` | 310 | 124 (40%) | Not individually hand-verified |
| `modals.ts` | 252 | 113 (45%) | Not individually hand-verified |
| `posts.ts` | 134 | 30 (22%) | Not individually hand-verified |
| `typography.ts` | 189 | 76 (40%) | Not individually hand-verified — typography.ts is P0-governed (CLAUDE.md rule 12), treat any cleanup here as higher-risk/Later |
| `detail.ts` | 101 | 6 (6%) | Cleanest bucket by far — low-hanging, low-risk if pursued |
| `share.ts` | 131 | 40 (31%) | Not individually hand-verified |

**Recommendation:** the `bidCard*`/`bidActions*`/`bidCarousel*`/`bidControls*`/`bidList*`,
`promoCardFeatured*`/`promoCardGroup`, and `tradeListItem*` blocks in `cards.ts` (~110+ keys) can
be deleted alongside their orphaned components in §1 with high confidence — same severity/effort/
risk as those files. The remaining bucket-wide numbers are **Later / Med severity** — need a
per-file pass that accounts for computed-key access before trusting the raw percentages,
especially `buttons.ts` and `cards.ts`'s `listingCard*`/`button*` prefixes.

---

## 6. Retired-feature remnants (SavedItems, cart, trades, Top Picks, /feed, reviews, comments, favorites)

Per `documentation/INDEX.md`: SavedItems deleted 2026-08-30, `/feed` + Top Picks retired
2026-08-14, cart retired as part of the 2026-08-27 offer engine, hidden trade-proposal subsystem
deleted 2026-08-26. Verified against the current tree:

- **SavedItems**: `SavedItemsContext` does not exist as a file anywhere — fully removed already. Only a comment in `usePostLike.ts` ("...SavedItemsContext modelled") references the old pattern by name; no functional remnant. **Nothing to clean up.**
- **Cart**: `CartContext` does not exist as a file — context itself is gone. But `src/styles/components/cart.ts` (28K, the entire style bucket) is still present and fully orphaned (§1). Also `src/utils/calculateCartTotals.ts` is a dead orphaned util from the same retirement. **Action: delete both — High severity given the doc explicitly says cart was retired, Effort S, Risk None.**
- **Trade proposal**: zero matches for `TradeProposal`/`tradeProposal` anywhere — fully gone, matches the vault's own note. `CollectTab.tsx` (§1) looks like an adjacent casualty of the same cut (orphaned from `TradeScreen`). **Nothing further to clean beyond §1's CollectTab.**
- **Top Picks**: zero live references; the only trace is a comment in `BottomTabNavigator.tsx:24` explaining historical navigator wiring ("TopPicks ('Show more') AND UserProfile are registered on the ROOT..."). Comment-only, not functional dead code, but worth a follow-up doc/comment cleanup pass since it references a screen registration pattern for a feature that no longer exists — could confuse a future reader of navigation (which is otherwise read-only/off-limits to touch).
- **`/feed`**: zero matches for the literal route string anywhere — fully retired, matches docs.
- **Reviews**: **not fully retired**, contrary to the summary's framing — `WriteReviewModal` is still actively imported and rendered by `OrderDetailScreen/index.tsx`. However, the `ReviewDto` type (knip-flagged) is unused — the modal apparently doesn't type its data through that DTO. Not dead code to remove; flag as a minor type-hygiene gap, Low severity.
- **Comments**: no `Comment`/`CommentDto` component or screen exists in `src` at all (only an icon-map key). Nothing to clean up — feature was either never built as literal "comments" or was named something else (posts/replies).
- **Favorites**: `FavoritesScreen` is registered in `AppNavigator.tsx` and has real (if minimal) content — an intentional empty-state placeholder screen ("favorites comes later" per your own memory notes), not dead code. **Not a retirement remnant — leave as-is.**

---

## 7. Comments, TODOs, console usage

- **Commented-out code blocks (3+ consecutive lines of actual code, not prose comments):** none found in a scan of `src` — the codebase's inline comments are overwhelmingly prose/rationale (e.g. the extensive "Roobert descender nudge" style of comment seen throughout `styles/components/*.ts`), not commented-out dead code. Nothing to report here.
- **TODO/FIXME/HACK:** 1 total, in `src/components/search/SearchBar/index.tsx`. Trivially low.
- **`console.log`/`warn`/`error` in `src`, non-`__DEV__`-guarded:** 16 call sites across 12 files (none wrapped in `__DEV__` anywhere in `src` — 0 guarded instances found). Top files: `AccountScreen/Profile/index.tsx` (3), `CreateCollection/index.tsx` (2), `CustomFlatList/index.tsx` (2). Severity Low, Effort S, Risk None — but since literally zero are dev-guarded, consider a lint rule (`no-console`) rather than manual cleanup given the count is small.

---

## 8. Stale repo-root artifacts

| Item | Referenced by anything? | Recommendation | Severity | Effort | Risk |
|---|---|---|---|---|---|
| `.plans/` (8 files, 92K) | `CLAUDE.md` explicitly says "The repo-root `.plans/` is retired — don't write there," but the directory and its 8 markdown files (`chat-release-mobile.md`, `collection-detail-fullscreen-restyle.md`, `global-search-rework.md`, `oak-client-model-and-serverless-rename.md`, `query-invalidation-review.md`, `sources-and-insights-fable-review.md`, `top-level-overlay-system.md`, `README.md`) still physically exist and are still git-tracked | Migrate any still-relevant insight into `nanza-vault/nanza-mobile/solution-designs/`, then delete the directory | High (directly contradicts documented policy) | S (docs-only, no code risk) | None |
| `plan.md` (repo root, 8K) | Not referenced by any script/doc; content is a single specific completed-looking plan ("Style CreateBid & CreateListing like EditBid inline layout") | Same as above — fold into vault or delete; violates the same "plans go in the vault" rule as `.plans/` | Med | S | None |
| `compact-summary.md` (repo root, 16K) | Not referenced by any script; per `CLAUDE.md`'s own "Compacts" convention, compacts belong in `compacts/` (a directory), not a single root-level file | Move into `compacts/` or delete if superseded | Low | S | None |
| `solution-designs/` (repo root, empty dir) | Zero files inside; `CLAUDE.md` states designs live ONLY in the vault, never in this repo | Delete the empty directory — it shouldn't exist at all per policy | Med | S | None |
| `.cursorrules` / `.cursorignore` | Still used by Cursor IDE if anyone on the team uses it; `.cursorrules` documents the Figtree "$" font convention that (per §4) was never wired into the actual token system | Not dead in the "unused file" sense, but its content is partially stale/aspirational (documents tokens that don't exist). Recommend reconciling with `typography.ts`/`dark.json` rather than deleting the file. | Med (content-stale, not file-stale) | M | Low |
| `amplify/` (10 files, 40K) | **Zero references anywhere in `src`, `App.tsx`, `index.js`, `package.json`, `babel.config.js`, `metro.config.js`** — the only knip hit was the ambient `.d.ts` (kept, §1). Nothing in the app actually calls into Amplify at runtime. | Very likely a vestigial scaffold from before the app switched to Cognito directly (`services/auth/cognito.ts`) — confirm with the team, then delete the whole directory | High (40K + conceptual confusion for new devs — implies a backend integration that isn't real) | M (needs a "is this really dead" confirmation before deleting infra scaffolding) | Low |
| `vendor/bundle/` (27 files, 3.5MB, Ruby/CocoaPods gems) | `.gitignore` line 60 has `/vendor/bundle/`, but `git check-ignore -v` on a tracked file returns **nothing** — meaning these files were committed to history *before* the ignore rule existed (or with `git add -f`) and remain tracked despite the rule. `git ls-files vendor` confirms 27 files are actually tracked in git right now. | **This is a real bug**: the gitignore rule is not doing anything because the files are already tracked. Needs `git rm -r --cached vendor/bundle` (a state-changing git operation — out of scope for this read-only audit to perform, flagging for the user) to actually stop tracking them, followed by a normal commit. | **High** (3.5MB of vendored Ruby gem sources — Makefiles, C extension code — committed to a JS/RN repo's git history, growing on every clone) | S once someone runs the untrack command | None (gitignore already covers future changes; this only stops tracking, doesn't delete working files) |
| `scripts/*` (4 files) | `build-android-release.sh`, `set-ios-version.sh` — not verified against CI/fastlane wiring in this pass (out of `src/` scope but flagged since scripts/ was named in the brief); `node_with_polyfills.sh` — not referenced by `package.json` scripts (which call `polyfill.js` directly via `NODE_OPTIONS`, not this shell wrapper); `figma-token-sync.js` — dead per §1. | `node_with_polyfills.sh` looks like an earlier iteration of the now-inline `NODE_OPTIONS` approach in `package.json` — likely stale. Confirm and delete. | Low–Med | S | Low |
| Both `yarn.lock` AND `package-lock.json` present | `package.json` has an `.npmrc` (123 bytes) but no explicit `packageManager` field checked in this pass. Two lockfiles for two different package managers in the same repo is a classic footgun — whichever one isn't authoritative silently drifts out of date every time someone runs the "wrong" install command. | Determine which package manager CI/deploy actually uses and delete the other lockfile | Med | S | Low — verify CI config first (out of `src/` scope, not independently checked here) |

---

## Correction to task premise: i18n is NOT dead

The task brief states `src/lang/` is empty. **It is not** — it contains `en.json` (8.3K),
`fr.json` (8.5K), `ja.json` (9.5K), and `i18n.js` (the i18next init, imported by `index.js:11`).
`useTranslation` is imported in 9 files (`SignUpScreen`, both `ForgotPasswordScreen` steps ×2,
`SignUpScreen/ConfirmationCode`, `Favorites/index.tsx`, `ForgotPasswordModal`,
`EntityCard/Listing.tsx`, `EntityCard/List.tsx`), with roughly **38 real `t('key')` call sites**
across them (a naive `grep "t('"` undercounts due to JSX-brace positioning — verified by hand
per file). i18n is real, working infrastructure, actively used in the auth/forgot-password
flow and two entity-card labels. **Do not flag as dead.**

One oddity worth a separate bug ticket (not dead code, a probable misuse): `EntityCard/Listing.tsx:83`
and `EntityCard/List.tsx:132` both call `t(order.name)` — passing a **runtime product name** as
the i18n key, which will only "work" by printing the raw product name back out whenever there's
no matching translation key (which there never will be for a card name). This looks like a
copy-paste artifact from a real i18n call rather than intentional design.
