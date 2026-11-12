---
tags: [nanza-mobile, audit]
---

# Dead code, unreachable surfaces, assets, repo leftovers — nanza-mobile, 2026-11-01

Scope: unused npm packages, directories, components, exports, style classes, assets and
root-level leftovers. Tree: `dev` at `4f710f4d` plus the uncommitted messaging-improvements
edits. Every claim below was re-grepped by hand after `knip` ran; the knip output itself is in
[[knip-report]]. Synthesis and triage in
[[../../solution-designs/codebase-audit-2026-11|Codebase audit — November 2026]].

## 1. npm packages — nothing removable

All 38 `dependencies` and 13 `devDependencies` resolve to at least one import or config site.
Method: grep for `'<pkg>'` / `'<pkg>/…'` across `src/`, `App.tsx`, `index.js`, the babel /
metro / eslint / react-native configs; then hand-check every package with 0–1 hits.

| Package | Where it is used | Note |
|---|---|---|
| `buffer` | nowhere in src | **keep** — undeclared dependency of `react-native-svg`; removing it breaks the Metro resolve (recorded 2026-09-16) |
| `fast-text-encoding` | `App.tsx` side-effect import | `TextEncoder` polyfill for Cognito / Airbridge |
| `react-native-worklets` | `babel.config.js` plugin | required by reanimated 4 |
| `react-native-dotenv` | `babel.config.js` (`module:react-native-dotenv` → `@env`) | why knip reports `@env` as unlisted |
| `react-native-screens` | no import | autolinked native peer of react-navigation; required |
| `@babel/plugin-transform-private-methods` | `babel.config.js` | |
| `@react-native-community/slider` | `WithdrawModal` only | one consumer, live |
| `@chainplatform/react-native-htmlview` | `EntityDescriptionHtml` + `types/htmlview.d.ts` | one consumer, live |
| `react-native-fs` | `IDConfirmScreen`, `ScanCameraScreen` | |
| `react-native-webview` | `InAppBrowserSheet`, `EmbedPlayer`, `OrderTrackingModal` | the third consumer is itself dead (§3) |
| `lottie-react-native` | `SocialAuthButtons`, `LoadingIndicator`, `Button` | |
| `@react-native-clipboard/clipboard` | Settings, `ConversationThread` | |
| `react-native-device-info` | `UpdateGate`, `AnalyticsBridge`, `LegalFooter` | |
| `react-native-app-auth`, `amazon-cognito-identity-js` | `services/auth/cognito.ts` | |
| `mixpanel-react-native`, `airbridge-react-native-sdk` | `services/analytics`, `services/attribution` | |
| `@react-native-firebase/app` + `/messaging` | `services/push` | pulls `firebase@12` (web SDK, 45 MB + 111 MB `@firebase/*` in node_modules) as a declared dependency; nothing in `src/` imports it |
| `rn-emoji-keyboard` | `MessageActionSheet` | added 2026-11-01 |
| dev: `@react-native-community/cli`, `@react-native/typescript-config`, `@types/react`, `typescript` | CLI / `tsconfig.json` `extends` / types | no import expected |

Native side matches: every pod in `Podfile.lock` (AppAuth, BVLinearGradient, Firebase*,
Mixpanel, RNAWSCognito, RNCAsyncStorage, RNCClipboard, RNDeviceInfo, RNFB*, RNFS,
RNGestureHandler, RNReanimated, RNSVG, RNScreens, RNWorklets, Stripe*, VisionCamera,
airbridge, lottie, image-picker, image-resizer, safe-area-context, slider, webview) maps to a
live package. `android/app/build.gradle` declares nothing beyond react-android / hermes.

## 2. Dead source files (knip, hand-verified) — 3 files, 584 lines

| File | Lines | Why dead |
|---|---:|---|
| `src/components/entity/EntityCard/List.tsx` | 296 | zero importers anywhere (`EntityCard/` has no `index`; nothing imports `EntityCard/List`) |
| `src/components/entity/EntityCard/EntityCardBase.tsx` | 112 | only imported by the dead `List.tsx` |
| `src/components/modals/ActionModal/Entity.tsx` | 176 | only imported by the dead `List.tsx`; `ActionModal/index.tsx` and `Feed.tsx` stay |

The whole `src/components/entity/EntityCard/` directory goes. Every module these three import
(`ConfirmSheetContent`, `EntityLockupRow`, `extractEntityListData`,
`invalidateCollectionQueries`, `getCardStyles`, …) has other live importers — no cascade.
knip's other three "unused files" are false positives: `polyfill.js` (NODE_OPTIONS),
`scripts/figma-token-sync.js` (Figma Scripter script documented by the figma-sync skill),
`amplify/backend/types/amplify-dependent-resources-ref.d.ts` (dead with the rest of
`amplify/`, §8).

## 3. Components hidden behind barrels — 3 files, 395 lines

knip reports the *default export* unused but not the file, because a barrel re-exports it.
Nothing imports the barrel name either.

| Component | Lines | Barrel | Consumers |
|---|---:|---|---|
| `components/orders/OrderTrackingModal/index.tsx` | 47 | `components/orders/index.ts` | none (`OrderTracking` is the live one) |
| `components/global/Form/NumericField/index.tsx` | 155 | `components/global/Form/index.tsx` | none |
| `components/global/Form/ImageField/index.tsx` | 193 | `components/global/Form/index.tsx` | none (`CreateCollection` has its own `ImageField` *type*, unrelated) |

`CodeField` and `TextArea` are also re-exported from the Form barrel without consumers of the
barrel name, but both are imported directly (`SignUpScreen`, `ForgotPasswordScreen`,
`ConnectRequestSheetContent`) and stay. Deleting `OrderTrackingModal` removes one of the three
`react-native-webview` consumers; the other two keep the package.

## 4. Unreachable screen — `Favorites`

`src/screens/account/AccountScreen/Favorites/index.tsx` (34 lines) is registered on the root
stack (`AppNavigator.tsx:701`), typed in `RootStackParamList`, and has a deep-link path
(`favorites`), but **no `navigate('Favorites')` exists anywhere** and the Account screen's
section list (Orders / Wallet / Selling / Settings) has no Favorites row. It renders a
permanent empty state ("Saved items retired", 2026-08-29). Last meaningful touch 2026-08-14.

Removing it also removes its only dependencies:
- `components/empty/EmptyStateMessage/` — Favorites is its single importer (`EmptyState` with
  19 importers is the live primitive).
- the `accountScreen.favorites.*` i18n keys (§7).

Needs an `AppNavigator.tsx` + `types/navigation.ts` edit → P0 rule 6, escalate.

## 5. Mounted but never read — `ListingQuantityContext`

`src/contexts/ListingQuantityContext.tsx` (55 lines). `ListingQuantityProvider` wraps the tree
in `App.tsx:102`; `useListingQuantity` has zero callers (knip + grep). The only other hits for
"ListingQuantity" are unrelated style-class names. Delete the file and unwrap the provider.
Touches `src/contexts/*` → P0 rule 7, escalate (deletion only, no behaviour).

## 6. Dead feature branch — Header / PageLayout sticky tabs

`Header` accepts `showActivityTabs` + `tabs` and renders `Tabs`; `PageLayout` accepts
`headerProps.tabs` and renders `StickyTabs`. **No caller passes either** — all eight `tabs={…}`
sites in `src/` are `SegmentedTabs` (6) or `TabPillsRow` (2); a multiline scan of every
`<Header …>` / `<PageLayout …>` opening tag finds no `tabs`, `stickyTabs` or
`showActivityTabs` prop, and no screen builds a `tabs:` object for `headerProps`.
2026-09-16 flagged the scroll-to-active-tab branch as dead; the whole path is now dead.

| Item | Lines |
|---|---:|
| `components/global/Tabs/index.tsx` | 154 |
| `components/global/Header/StickyTabs.tsx` (+ its re-export at `Header/index.tsx:770`) | 105 |
| `Header/index.tsx` tabs branch (`tabConfig`, `showActivityTabs`, scroll-to-active, lines ~270–300 and 659–680) | ~60 |
| `PageLayout/index.tsx` `StickyTabs` branch (~line 205) + `headerProps.tabs/activeTab/onTabPress/tabsOnly` | ~25 |
| `styles/components/tabs.ts` classes only these consume | 17 flagged now, more once the above goes |

## 7. i18n — 106 of 127 keys and two whole locales unused

`src/lang/i18n.js` registers `en`, `fr`, `ja` with `lng: 'en'` hard-coded and no language
switch anywhere, so `fr.json` (125 lines) and `ja.json` (130 lines) never render. Of the 127
keys in `en.json`, **21 are used** — all under `emailScreen`, `confirmationCode`,
`passwordScreen`, `forgotPassword`, `global.label.{next,submit,browse}` and
`accountScreen.favorites.*` (the dead screen, §4). The other 106 describe a retired
onboarding/account flow (`splashScreen.*`, `mobileNumber.*`, `saveToiCloud.*`,
`nameScreen.*`, `usernameScreen.*`, `profilePicture.*`, `agreementScreen.*`,
`welcomeScreen.*`, `accountScreen.buying/selling/settings/signOut.*`). The 2026-09-16
decision #4 ("auth-only localization — intentional, or a backlog?") is still open; the cheap
end is: keep `react-i18next`, trim `en.json` to the 18 live keys, delete `fr.json` / `ja.json`.

## 8. Root-level directories and files

| Item | Size | Finding | Action |
|---|---|---|---|
| `amplify/` | 10 tracked files, 40 KB | zero references from app, `package.json`, Podfile or Gradle; config names `tcgxconsumer`; last touched 2025-12-29 by an unrelated commit | delete (2026-09-16 decision #2, still open) |
| `vendor/bundle/` | 27 tracked files, 3.5 MB | ruby 2.7 `ffi` gem build tree from 2024-11, tracked **despite** `/vendor/bundle/` in `.gitignore` — the 2026-09-16 `git rm --cached` never landed | `git rm -r --cached vendor/bundle` |
| `scripts/setup-local.sh` | — | missing, but `npm run cognito-setup` points at it | drop the script entry |
| `scripts/figma-token-sync.js` | 124 lines | Figma Scripter script, not app code; knip flags it | keep; add to a knip ignore or move it under the skill's references |
| `.cursorrules` | 2.6 KB, 2026-02 | Cursor-era rules; duplicates `CLAUDE.md`; its "Figtree `$` price font" rule references `theme.fonts.family.priceMedium/priceSemiBold`, which do not exist in `dark.json` | delete (2026-09-16 decision #3, still open) |
| `.claude/commands/plan.md` | — | live | keep |
| `polyfill.js` | 36 lines | live (`NODE_OPTIONS --require`), knip false positive | keep |

## 9. Assets

> **2026-11-07:** rows below marked ✅ were removed in the app-size pass — 30 unused `.ttf` (assets, Android copy, pbxproj, both Info.plists + 4 stale Volte plist entries), `activity.svg` plus 60 unregistered-in-use icon keys/SVGs, `avatar.jpg`, `logo.png`; `splash-backdrop.jpg` (2.9 MB) deleted and the splash now draws `auth-backdrop.jpg` (new 1080×1350 art, 364 KB) — one image for splash + auth.

| Asset | Size | Finding |
|---|---|---|
| ✅ `assets/fonts/` Figtree ×10, Geist ×9, EuclidCircularB ×10, `Roobert-RegularItalic` | 30 of 35 `.ttf`, ~3.9 MB | no `fontFamily` token references them (`dark.json` names only Roobert Regular/Medium/SemiBold/Bold + BaltoMobile-Book); still linked into both binaries (`Info.plist` UIAppFonts lists all 35; `android/app/src/main/assets/fonts` = 35 files, 4.5 MB). 2026-09-16 Later item, untouched. Removing them needs `npx react-native-asset` (or hand-editing `Info.plist`, the pbxproj and the Android assets copy) + a build on each platform |
| ✅ `assets/icons/activity.svg` | 4 KB | unreferenced |
| ✅ `assets/images/splash-backdrop.jpg` | 2.9 MB | referenced, but still the unoptimized export (2026-09-16 Later item) |
| `assets/images/fab-logo@2x/@3x.png` | 81 KB | referenced implicitly via `fab-logo.png` density resolution — keep |
| `assets/lottie/*.json` ×2 | 7 KB | both referenced |
| `metro.config.js` comment | — | says "`assets/icons/loading-2.lottie`" exists; it does not (`Button` notes lottie-ios cannot open `.lottie` anyway). The `lottie` assetExt entry is dead config |

## 10. Dead exports (knip, 48) — 13 functions truly dead (~230 lines), the rest just over-exported

Used nowhere, not even in their own file (definition is the only hit):
`useFetchExistingEntityLists` (53 lines, `ListModal/data/fetchExistingEntityLists.ts`),
`getConditionLabels` (21) + `getConditionPills` (16, `constants/conditionLabels.ts`),
`useSetPrimaryContentItem` (20, `posts/usePosts.ts`), `generateShippingOptionsRate` (17) +
`getReturnShipment` (7, `utils/shipping.ts`), `getEntityHeroBackdropStyle` (10,
`styles/components/posts.ts`), `useReplyToConnection` (9, `MessagesScreen/data/mutations.ts`),
`isShareGuardActive` (9, `ShareGuardContext.tsx`), `getEntityTagLine` (9,
`utils/getEntityTagLine.ts`), `getListsQueryConfig` (7, `profile/data/fetchLists.ts`),
`formatFetchTimeUtc` (7, `utils/dateHelpers.ts`), `draftHasImage` (2, `composer/draft.ts`).

Used only inside their own file (drop the `export`, keep the code): `buildListQueryParams`,
`isPostSeenTracked`, `parseShareReferenceCode`, `hasEmoji`, `fetchAffiliateRail`,
`railItemHasUnseen`, `offerPriceBounds`, `isPushAvailable`, `pluralize*`, `TAG_LINE_SEPARATOR`,
and the constants `SCAN_ADD_MAX_QUANTITY`, `QUICK_REACTIONS`, `AFFILIATE_LINK_HOST`,
`ART_TOTAL`, `SERVICE_FEE_RATE/FIXED`, `OFFER_PRICE_TOLERANCE`, `COLLECTION_PAGE_SIZE`,
`NANZA_PAGE_SIZE`, `BRAND_TAG_VALUES_LIMIT`, `LIST_ENTITY_INCLUDES`, `ENTITY_HERO_CARD_ASPECT`,
`PUBLIC_AUTHOR_PROFILE_TYPES`. Barrel names: `Form/index.tsx` `NumericField`, `ImageField`,
`CodeField`, `TextArea`; `orders/index.ts` `OrderTrackingModal`; `lang/i18n.js` default.

83 unused exported *types*: 20 are DTO aliases in `src/types/index.ts` nobody imports
(`TagDto`, `CategoryDto`, `ShipmentDto`, `TransactionDto`, `ReviewDto`, `ContentItemDto`,
`QueryListDto`, `ScanItemDto`, `GroupListingDto`, `GroupBidDto`, …); the rest are `*Props`
and local helper types. Harmless; prune when touching the file.

## 11. Stale references that are no-ops today

- `invalidateQueries({ queryKey: ['collectTab'] })` at `utils/invalidateCollectionQueries.ts:22`
  and `bulk/CreateListingsScreen/BulkTab.tsx:281` — no query declares that key (CollectTab was
  deleted 2026-09-16).
- `hooks/useBottomPadding.ts:31` lists tab routes `'Saved'`, `'Agent'`, `'CenterButton'`,
  `'Account'`, `'AccountHome'` that no navigator registers, and **omits** `'Groups'` /
  `'GroupsListScreen'`, which do exist. Worth a behaviour check on the Groups tab, not just a
  cleanup.
- Comments only (no code): "Top Picks" ×4, `FeedListingCard`/`FeedBidCard`/`FeedBulkCard` ×5,
  "CollectTab" ×1.

## 12. Disk, not binary — ~12 GB of build output under the repo

Untracked and gitignored, but it is why the folder feels heavy: `android/app/build` 2.9 GB,
`ios/build` 2.1 GB, `node_modules/react-native-reanimated/android` 4.1 GB,
`node_modules/react-native-worklets/android` 2.7 GB (CMake/NDK outputs from local builds),
`ios/Pods` 0.3 GB. `cd android && ./gradlew clean` + `rm -rf ios/build` reclaims most of it.

## 13. Size since the last audit

| | 2026-09-16 | 2026-11-01 |
|---|---:|---:|
| `src/` files | 580 | 617 |
| `src/` lines | 100,028 | 104,254 |
| `components/` | 256 files / 38,857 | 253 / 39,161 |
| `screens/` | 152 / 22,940 | 163 / 24,130 |
| `styles/` | 43 / 30,056 | 45 / 30,948 |

+37 files / +4.2k lines in six weeks despite the 41-file September deletion — growth is real
feature work (posts composer, messaging, trades, market tab, push), not drift.
