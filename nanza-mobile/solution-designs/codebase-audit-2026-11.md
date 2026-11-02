---
tags: [nanza-mobile, solution-design, plan, audit, tooling, design-system]
status: AUDIT COMPLETE 2026-11-01, nothing removed yet — findings + triage below, awaiting Skylar's go on the fix-now bundle and the 5 decisions. Follow-up to codebase-audit-2026-09.
---

# nanza-mobile codebase audit — November 2026 (unused packages, directories, components)

## Goal

The app is growing (617 files / 104k lines, +4.2k lines since 2026-09-16) because features are
landing — composer, messaging, trades, market tab, push. This pass asks one narrow question:
**what is in the repo that nothing uses any more?** npm packages, directories, components,
exports, style classes, assets, root-level leftovers. It does not re-run the September
performance / network / standards sweeps; the open items from those are still listed in
[[codebase-audit-2026-09#Later release · needs its own QA pass or a visual regression sweep|September's Later list]].

Raw per-item tables (every file, line count and the command behind each number) live in
[[../audits/2026-11-01/INDEX|audits/2026-11-01]].

## How it was run

One session, no sub-agents. `npx knip@5` first; then every package, directory and component
was re-grepped by hand (import-site counts per module, multiline JSX prop scans for the
Header/PageLayout tabs path, a node script for style-bucket class references, the i18n key
set against `t('…')` calls, font family names against `dark.json` + `Info.plist` + the Android
assets copy, pods against `package.json`). Numbers are from `dev` at `4f710f4d` on 2026-11-01.

## Scorecard

| Area | Health | Verdict |
|---|---|---|
| npm packages | **Clean** | 51 of 51 used (38 deps + 13 dev). Nothing to remove. `buffer` stays (react-native-svg's undeclared dep). Native pods and Gradle match. |
| Source files & components | **Good** | 3 dead files (584 lines) + 3 components hidden behind barrels (395 lines) + 1 unreachable screen + 1 unread context + 1 dead Header feature path (~340 lines). ~1.4k lines total. |
| Dead exports | **Good** | 13 genuinely dead functions (~230 lines); 35 more exports only need the `export` keyword dropped. |
| Style buckets | **Poor** | ~1,760 of 4,019 classes (~9.3k of 30.9k lines, 30% of `styles/`) are referenced nowhere. `cards.ts` is 545 of 729 — the `feed*Card` / `listingCard<Size>` / `productCard*` / `listCard*` families whose components were deleted in September. |
| i18n | **Stale** | 21 of 127 `en.json` keys used; `fr.json` / `ja.json` loaded but `lng: 'en'` is hard-coded. |
| Assets | **Unchanged since Sept** | 30 of 35 fonts dead (~3.9 MB per binary), `activity.svg`, the 2.9 MB splash JPEG. |
| Repo root | **Stale** | `amplify/` (dead), `vendor/bundle` still tracked, a `package.json` script pointing at a missing file, `.cursorrules`. |

## The findings that matter

1. **Style buckets carry ~9.3k dead lines.** When the September pass deleted 41 files
   (FeedListingCard, ListingCard, EntityCard, CollectTab, …) their classes stayed behind in
   `cards.ts`, `layout.ts`, `inputs.ts`, `modals.ts`, `scan.ts`, `share.ts`, `entity.ts`,
   `filters.ts`. Per-bucket lists are in
   [[../audits/2026-11-01/style-bucket-classes|style-bucket-classes]]. This is safe to do
   mechanically **because the buckets are typed**: every `styles.foo` access is checked
   against the bucket's `*Styles` type, so deleting a class that is still used fails
   `tsc --noEmit`. The two runtime-composed key families are excluded from the lists —
   `Button/index.tsx` builds `button<Size><Variant>[Text|Description]` and
   `AvatarPlaceholder` builds `circle_<size>` — and nothing else in `src/` indexes a bucket
   with a computed key. Style getters are memoized (Sept Bundle B), so this is a source-size
   and bundle-size win, not a render-time one.
2. **One whole feature path is dead: Header / PageLayout sticky tabs.** `Tabs` (154 lines),
   `StickyTabs` (105), Header's `showActivityTabs`/`tabs`/scroll-to-active branch and
   PageLayout's `headerProps.tabs` branch have no caller — every `tabs={…}` in the app is
   `SegmentedTabs` or `TabPillsRow` (the in-page tabs standard
   since 2026-10-25). September saw only the scroll branch as dead.
3. **`Favorites` is registered but unreachable.** Root-stack screen + deep link + param type,
   nothing navigates to it, the Account list has no row for it, and it renders a permanent
   empty state since Saved items were retired (2026-08-29). Removing it takes
   `EmptyStateMessage` (sole consumer) and the `accountScreen.favorites.*` strings with it.
4. **`ListingQuantityProvider` wraps the app for nobody.** The hook has zero callers.
5. **Packages are clean.** This is the question the audit was asked, and the answer is that
   `package.json` has no dead weight; the September removals held. The only package-level
   note is that `@react-native-firebase/app` declares `firebase@12` (web SDK) as a
   dependency — 156 MB in `node_modules`, nothing in `src/` imports it, Metro only bundles
   what is imported.

## Triage

### Bundle E · Fix now — deletions and config, no behaviour change, `tsc` + lint as the gate

| Item | Lines / size | Touches |
|---|---:|---|
| Delete `src/components/entity/EntityCard/` (2 files) + `modals/ActionModal/Entity.tsx` | 584 | — |
| Delete `OrderTrackingModal/`, `Form/NumericField/`, `Form/ImageField/` + their barrel lines | 395 | `orders/index.ts`, `Form/index.tsx` |
| Delete `ListingQuantityContext.tsx`, unwrap the provider in `App.tsx` | 55 | `src/contexts/*` (P0 rule 7 — deletion only, approve) |
| Delete `Tabs/`, `Header/StickyTabs.tsx`, the tabs branches in `Header` and `PageLayout`, then the orphaned `tabs.ts` classes | ~340 | `Header`, `PageLayout` |
| Delete the 13 dead functions; drop `export` on the 35 in-file-only exports | ~230 | 20 files |
| Style-bucket sweep: delete the unreferenced classes bucket by bucket, `tsc` after each | ~9,300 | `src/styles/components/*` (P0 rule 7 is `styles/utils/*`; buckets are fair game) |
| Remove the two no-op `['collectTab']` invalidations | 2 | |
| `git rm -r --cached vendor/bundle`; delete `amplify/`; delete `.cursorrules`; drop the `cognito-setup` script; delete `assets/icons/activity.svg`; fix the `metro.config.js` lottie comment / assetExt | 3.5 MB tracked | repo root |
| Trim `en.json` to the live keys, delete `fr.json` + `ja.json`, drop the two `resources` lines | 361 | `src/lang` (needs decision 1 first) |

Order: files → exports → buckets (largest first: `cards`, `layout`, `inputs`, `modals`,
`scan`, `share`, `entity`, `filters`) → root. Run `npx tsc --noEmit && npm run lint` after each
bucket; a device smoke of Home · Entity · Trade · Messages · Make Offer at the end.

### Needs Skylar (navigator / native)

| Item | Why it is not in the bundle |
|---|---|
| Remove the `Favorites` screen + `EmptyStateMessage` | edits `AppNavigator.tsx` and `types/navigation.ts` (P0 rule 6) |
| Remove the 30 dead fonts (Figtree, Geist, EuclidCircularB, Roobert-RegularItalic) | native: `Info.plist` UIAppFonts, pbxproj resources, Android assets copy; needs a build on each platform. ~3.9 MB per binary. Still the single biggest *shipped* saving |
| `useBottomPadding` tab list | names routes that do not exist and omits `Groups` — check Groups-tab bottom padding on device before "cleaning" it |

### Still open from September (not re-audited here)

Splash JPEG re-export (2.9 MB), Android R8, image caching, `fetchData` abort, ESLint 9, CI,
`LoadingSpinner` over `ActivityIndicator` (now 31 files; `LoadingSpinner` has 2 importers
vs `LoadingIndicator` 52), `PostComposer.tsx` split (now 1,817 lines), the two
`CardCarousel`s (shelf 1 importer / global 4), `TopHeader` (4) vs `Header` (16),
`HomeTagCarousel` vs `TagCarousel` (2 each), `TabPillsRow` (2) vs `SegmentedTabs` (6).

## Decisions needed (Skylar)

1. **i18n scope** (Sept #4, still open) — auth-only is the reality. Trim to the 18 live keys
   and drop `fr`/`ja`, or keep the files as the seed of a real localization pass?
2. **`amplify/`** (Sept #2) — confirm no Amplify backend is deployed from it; then it goes.
3. **`.cursorrules`** (Sept #3) — the Figtree "$" price-font rule it carries was never wired
   (no `price*` tokens). Delete the file, or port the rule into `CLAUDE.md` and implement it?
4. **Style-bucket sweep scope** — all 1,760 classes in one bundle, or `cards.ts` first
   (545 classes, 3.3k lines) as a trial?
5. **`Favorites`** — delete the screen (nav edit), or is it the placeholder for the
   "favorites comes later" note in [[profile]]?

## What is already in good shape (do not "fix")

- Every npm package, pod and Gradle dependency maps to a live import.
- The 41 September deletions did not leave orphaned *components* behind — the only
  cascade is in the style buckets.
- No retired-feature code paths survive (cart, Top Picks, trade proposals, CollectTab,
  SavedItems): only comments and the two no-op query keys.
- `vendor/`, `android/app/build`, `ios/build` and the 6.8 GB of NDK output inside
  `node_modules/react-native-reanimated|worklets/android` are build artefacts, not shipped;
  `./gradlew clean` + `rm -rf ios/build` reclaims ~12 GB of disk.

## Related

- [[codebase-audit-2026-09]] — the broad September pass this follows.
- [[design-system]] · [[ui-fixes]] · [[../audits/2026-11-01/INDEX|Raw reports (2026-11-01)]]
