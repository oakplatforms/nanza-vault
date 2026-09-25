# Runtime Performance & Code Efficiency Audit — nanza-mobile

Scope: render cost, lists, images, animation, startup, memory. Excludes network/React Query
(separate auditor) and style-rule *compliance* (separate auditor). Every count below comes from
a command run against the repo at `dev` (clean tree).

---

## Summary table (ranked by expected impact)

| # | Finding | Expected impact | Effort | Risk | When |
|---|---------|-----------------|--------|------|------|
| 1 | **520 unmemoized `getXStyles(theme)` calls** rebuild 16.5k-property style trees every render; `cards.ts` alone = 3,880 props and is rebuilt per list row | scroll fps (major), memory/GC | S | None | **Fix now** |
| 2 | **`ThemeProvider` value object is a new literal every render** — feeds 313 `useTheme()` consumers across 303 files | scroll fps, render count | XS | None | **Fix now** |
| 3 | **`React.memo` used exactly once app-wide** (1 site / 354 components); no list-row component is memoized | scroll fps (major) | M | Low | **Fix now** |
| 4 | **`eslint-plugin-react-hooks` is not installed and not configured** — zero dep-array / rules-of-hooks enforcement | latent correctness + perf | XS | None | **Fix now** |
| 5 | **`LoadingSpinner` runs 9 JS-thread `Animated` values animating layout `height` in an infinite loop**, mounted on the cold-start critical path | startup, fps | S | Low | **Fix now** |
| 6 | **No image caching library** — 93 plain RN `<Image>`, 0 FastImage/expo-image | memory, scroll fps, re-fetch | M | Low | Later |
| 7 | **Oversized assets shipped**: `profile.png` 3.2 MB, `splash-backdrop.jpg` 2.8 MB, `backdrop.svg` 571 KB | startup, bundle, memory | S | None | Later |
| 8 | **`SPLASH_MIN_MS = 2500`** — a hard 2.5 s floor on every launch regardless of readiness | startup (perceived) | XS | Low | Later |
| 9 | **`getItemLayout` present on 1 of 24 FlatLists**; `windowSize` never tuned; `removeClippedSubviews` on 3 | scroll fps | M | Low | Later |
| 10 | **`ListingQuantityProvider` wraps the whole navigator for 1 consumer**; unmemoized value | render count (minor) | XS | None | Later |
| 11 | Unvirtualized `ScrollView` + `.map` over unbounded data (`CollectionPickerScreen`) | memory, fps | S | Low | Later |
| 12 | Two near-duplicate `CardCarousel` components; neither uses `useCallback` for list props | maintainability, minor fps | S | Low | Later |

**Good news up front** (verified, no action needed): Hermes enabled on both platforms, New Architecture
on for Android, **zero `console.log` in `src/`**, **zero module-level side effects**, listener/timer
cleanup discipline is solid, and `HomeScreen` is genuinely well-engineered (native-driven scroll,
`InteractionManager`-deferred shelves, latched readiness gate).

---

## 1. Style computation cost — the single biggest finding

### Evidence

```
520   getXStyles(theme) call sites          rg 'get[A-Za-z]+Styles\(theme' src/ App.tsx | wc -l
328   files containing them
  7   wrapped in useMemo                    rg 'useMemo\(\(\) => get[A-Za-z]+Styles' src/ | wc -l
  0   consumers of the existing useThemeStyles helper
```

So **513 of 520 call sites (98.7%) rebuild their style object on every single render.**

Per-bucket totals (counted by parsing runtime property lines, excluding the `type` declaration
blocks, via a node script over `src/styles/components/*.ts`):

| bucket | lines | runtime props | call sites |
|---|---|---|---|
| `cards.ts` | 6,377 | **3,880** | 55 |
| `layout.ts` | 3,145 | 1,896 | **94** |
| `inputs.ts` | 2,162 | 1,252 | 38 |
| `modals.ts` | 1,676 | 957 | 36 |
| `buttons.ts` | 1,599 | 873 | 16 |
| `typography.ts` | 1,225 | 606 | **77** |
| … 31 more | | | |
| **TOTAL** | **29,720** | **16,540** | **520** |

### The multiplier: nested rebuilds

32 of the 37 bucket files call *other* bucket getters internally, so one call fans out:

```
src/styles/components/cards.ts:901       const typography = getTypographyStyles(theme)
src/styles/components/layout.ts:524      const typography = getTypographyStyles(theme)
src/styles/components/layout.ts:2178     ...getGlassStyles(theme).searchPill,
src/styles/components/postComposer.ts:95-98  typography + glass + cards   (3 nested)
src/styles/components/offers.ts:56,377,378   typography x2 + glass        (3 nested)
```

`getCardStyles(theme)` therefore rebuilds the whole 606-property typography tree as a side effect
of building its own 3,880 properties.

### The hot path

`getCardStyles` — the largest bucket — is called by the components that render **once per list row**:

```
src/components/feed/FeedListingCard/index.tsx:45-46
  const { theme } = useTheme()
  const cardStyles = getCardStyles(theme)      // 3,880 props + nested typography, every render

src/components/listings/ListingCard/index.tsx:53-54   (identical)
src/components/posts/PostItem.tsx:128-129             getPostStyles → 483 props + typography
```

None of these three is wrapped in `React.memo` (see §4), so every parent render rebuilds them all.

### Measured cost

Benchmark simulating the `cards` + nested `typography` rebuild shape (470 classes × ~8 props +
80 × 7), run on node/V8:

```
per card-row render (cards+typography rebuild): 1.847 ms
=> 10 visible rows: 18.47 ms per render pass     (60fps budget = 16.7 ms)
objects allocated per card row: ~552
```

Desktop V8 already blows the frame budget at 10 visible rows. Hermes on a mid-range Android device
is typically 3–10× slower, and this allocates ~5,500 short-lived objects per scroll frame, driving
constant GC pressure.

### Fix (least invasive — zero component changes)

I verified this is safe:

- **All 37 `get*Styles` exports have the identical signature `(theme: ThemeTokens)`** — the check
  `rg -o 'export const get[A-Za-z]+Styles = \([^)]*\)' | rg -vc 'theme: ThemeTokens\)$'` returns 0.
  (The only multi-arg helpers — `getGradient`, `getGradientInverted`, `getBottomBarInset` — are not
  named `*Styles` and are untouched.)
- **The getters are pure**: no `Date.now`, `Math.random`, `new Date`, or `Dimensions.get` anywhere
  in `src/styles/components/*.ts`.
- **`theme` identity is stable**: `src/styles/utils/ThemeProvider.tsx:17-21` builds it once at module
  scope (`const fallbackTheme = parseNumericStrings(resolvedTheme)`) and hands back the same
  reference (`const theme = fallbackTheme`); there is exactly 1 `<ThemeProvider` mount.

So a WeakMap keyed on theme identity is a drop-in memo with no semantic change. Add one helper:

```ts
// src/styles/utils/memoizeByTheme.ts
import { ThemeTokens } from './types'

export const memoizeByTheme = <T>(fn: (theme: ThemeTokens) => T) => {
  const cache = new WeakMap<ThemeTokens, T>()
  return (theme: ThemeTokens): T => {
    let hit = cache.get(theme)
    if (hit === undefined) {
      hit = fn(theme)
      cache.set(theme, hit)
    }
    return hit
  }
}
```

Then wrap each getter at its definition — a one-line change per bucket file, and **no call site or
component changes at all**:

```ts
// src/styles/components/cards.ts
export const getCardStyles = memoizeByTheme((theme: ThemeTokens): CardStyles => ({
  /* body unchanged */
}))
```

This collapses all 520 calls (and every nested fan-out) to one build per bucket for the app's
lifetime. Because the returned object identity also becomes stable, it makes `React.memo` on list
rows (§4) actually effective — the two fixes compound.

> Note this touches `src/styles/utils/*`, which CLAUDE.md P0 #7 / the escalation triggers put
> behind approval. Worth escalating explicitly, but the change is additive and behaviour-preserving.

**Secondary cleanup:** `src/styles/utils/useThemeStyles.ts` exists but has **zero consumers**
(`rg 'useThemeStyles' src/` returns only its own definition). Either adopt or delete it.

---

## 2. Contexts

| Provider | file | `value` memoized? | consumers (`rg` call sites / files) |
|---|---|---|---|
| `ThemeProvider` | `src/styles/utils/ThemeProvider.tsx:25` | **NO** — `value={{ theme, themeMode, setThemeMode }}` | **313 calls / 303 files** |
| `SessionProvider` | `src/contexts/SessionContext.tsx:102` | YES (`useMemo`, with a good explanatory comment) | 123 / 120 |
| `AppReadyProvider` | `src/contexts/AppReadyContext.tsx:53` | YES | 5 / 5 |
| `ModalStackProvider` | `src/contexts/ModalStackContext.tsx:85` | YES | 27 / 27 |
| `AuthProvider` | `src/contexts/AuthContext.tsx:45` | **NO** — `value={{ userData, setUserData }}` | 6 / 6 |
| `CollectionSelectionProvider` | `src/contexts/CollectionSelectionContext.tsx:40` | **NO** | 2 / 2 |
| `ListingQuantityProvider` | `src/contexts/ListingQuantityContext.tsx:43` | **NO** | **1 / 1** |
| `ShareGuardProvider` | `src/contexts/ShareGuardContext.tsx` | n/a — module-level listener set, no context | 0 |

### 2a. `ThemeProvider` — the amplifier (Finding #2)

```tsx
// src/styles/utils/ThemeProvider.tsx:20-28
export const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const theme = fallbackTheme
  const [themeMode, setThemeMode] = useState<ThemeMode>('user')
  return (
    <ThemeContext.Provider value={{ theme, themeMode, setThemeMode }}>
```

`theme` never changes, but the **wrapper object is new on every ThemeProvider render**, so any
`setThemeMode` call (or a parent re-render) invalidates all 303 consuming files at once — each of
which then re-runs its unmemoized `getXStyles(theme)` (§1). This is the mechanism that turns
finding #1 from "per-component waste" into "app-wide render storm".

Fix — 3 lines, no API change:

```tsx
const value = useMemo(() => ({ theme, themeMode, setThemeMode }), [theme, themeMode])
return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
```

### 2b. `ListingQuantityProvider` — whole-tree provider for one consumer

`App.tsx:76-78` wraps the **entire `AppNavigator`** in it, yet `useListingQuantity` has exactly
**1 call site in 1 file**. Its value is unmemoized *and* its three functions
(`setListingQuantity`, `getListingQuantity`, `clearListingQuantities`,
`ListingQuantityContext.tsx:27-41`) are re-created every render, so each quantity change
re-renders the whole navigator subtree. `listingQuantities` is exactly the kind of
frequently-changing value flagged in the brief. Memoize the value and `useCallback` the setters;
better still, move the provider down to the one subtree that uses it.

`AuthContext` and `CollectionSelectionContext` are unmemoized too but have small consumer counts
and low-churn state, so they are cheap follow-ups rather than priorities.

---

## 3. Lists

```
 24  <FlatList  /  Animated.FlatList          0  SectionList
  0  FlashList  — the app does not use FlashList at all
 65  <ScrollView / <Animated.ScrollView
 74  keyExtractor occurrences
  1  getItemLayout                            3  removeClippedSubviews
  1  initialNumToRender                       1  maxToRenderPerBatch
  0  windowSize                               0  updateCellsBatchingPeriod
```

**keyExtractor discipline is good** — every one of the 17 FlatList-bearing files has at least one.
**No oversized `initialNumToRender` / `windowSize` overrides exist** (the opposite problem: they are
never tuned at all).

Per-file `renderItem` audit (inline `renderItem={(` vs `useCallback` present):

```
inlineRenderItem=3  useCallback=6  src/screens/trade/TradeScreen/index.tsx
inlineRenderItem=3  useCallback=4  src/components/posts/composer/CardPickerLayer.tsx
inlineRenderItem=2  useCallback=3  src/components/collections/CollectionSearchOverlay/index.tsx
inlineRenderItem=1  useCallback=5  src/screens/trade/TradeScreen/CollectTab.tsx
inlineRenderItem=1  useCallback=9  src/screens/collections/CollectionDetailScreen/index.tsx
inlineRenderItem=1  useCallback=3  src/screens/bulk/CreateListingsScreen/BulkTab.tsx
inlineRenderItem=1  useCallback=4  src/components/global/AddressAutocomplete/index.tsx
inlineRenderItem=0  useCallback=0  src/components/shelf/CardCarousel/index.tsx
inlineRenderItem=0  useCallback=0  src/components/global/CardCarousel/index.tsx
```

- **`getItemLayout` on 1 of 24 lists.** The card carousels are fixed-width horizontal rows
  (`entity.ts` exports `LOCKUP_THUMB_WIDTH_ROW`, imported by `cards.ts:7`) — ideal `getItemLayout`
  candidates, giving instant scroll-to-index and skipping per-row measurement.
- **Two near-identical carousels**: `src/components/global/CardCarousel/index.tsx` and
  `src/components/shelf/CardCarousel/index.tsx` share the same header comment and structure, both
  taking `renderItem`/`keyExtractor` as props (so the caller controls memoization) and **neither uses
  `useCallback` internally**. Both use `react-native-gesture-handler`'s `FlatList` — deliberate and
  well documented (lines 3-7), not a bug.
- **Horizontal-in-vertical**: 123 `horizontal` props; the home feed nests horizontal carousels inside
  the vertical `Animated.FlatList`. Without memoized rows (§4) each vertical scroll tick re-renders
  every horizontal shelf's items.
- **Unvirtualized `ScrollView` + `.map` over unbounded data** —
  `src/screens/collections/CollectionPickerScreen/index.tsx:72-104` maps all `collections` inside a
  plain `ScrollView`. Most other `.map` sites are bounded (tabs, slides, attachments) and are fine:
  `SearchScreen:581` (`searchTabs`), `MessagesScreen:145` (`tabs`), `FeedBulkCard:238` (`slides`).

---

## 4. Memoization hygiene

### 4a. `React.memo` is used **once** in the entire app

```
rg 'React\.memo|\bmemo\(' src/  →  src/components/posts/EmbedPlayer.tsx:207  export default memo(EmbedPlayer)
```

That is **1 memoized component out of 354 `.tsx` files**. Every list-row component —
`FeedListingCard` (528 lines), `ListingCard` (532), `PostItem` (533), `FeedBidCard` (490),
`FeedBulkCard` (483) — is unmemoized *and* rebuilds a multi-thousand-property style object per
render (§1).

Hook usage overall is otherwise healthy: **284 `useMemo`, 461 `useCallback`** across 354 components.

Minimal fix for the top rows (no behaviour change, and only effective *after* §1 stabilises the
style object identity):

```tsx
// src/components/feed/FeedListingCard/index.tsx:528
export default React.memo(FeedListingCard)
```

Apply to `ListingCard`, `PostItem`, `FeedBidCard`, `FeedBulkCard`. Then pass `renderItem` via
`useCallback` in the 7 files that currently inline it.

### 4b. `eslint-plugin-react-hooks` is **not installed** (Finding #4)

```
ls node_modules/eslint-plugin-react-hooks   →  (does not exist)
```

`eslint.config.mjs` loads `@eslint/js`, `typescript-eslint` and `eslint-plugin-react` only — there is
**no `react-hooks` plugin and no `exhaustive-deps` / `rules-of-hooks` rule anywhere**. So the single
`react-hooks/exhaustive-deps` mention in lint output cannot be coming from a configured rule, and
dependency arrays across 745 `useMemo`/`useCallback` sites are entirely unverified. This is both a
correctness and a performance gap (stale closures; deps that silently defeat memoization).

Fix: add the plugin as a devDependency and append to `eslint.config.mjs`:

```js
import reactHooks from 'eslint-plugin-react-hooks'
// …
{ plugins: { 'react-hooks': reactHooks },
  rules: { 'react-hooks/rules-of-hooks': 'error', 'react-hooks/exhaustive-deps': 'warn' } },
```

Land it as `warn` first — with 745 sites unchecked, `error` would almost certainly break the build.

---

## 5. Images

```
93  <Image                  0  ImageBackground
 0  FastImage               0  expo-image        ← no caching library at all
89  resizeMode              148 SVG components (react-native-svg-transformer)
```

- **No image caching/decoding library.** All 93 usages are plain RN `<Image>`. RN's built-in cache is
  memory-only on iOS and limited on Android, so list tiles re-decode on every scroll-back and
  full-resolution bitmaps stay resident. `react-native-fast-image` or `expo-image` would cut both
  decode cost and steady-state memory in the feed. (The network auditor owns the URL width params;
  this is the decode/memory half.)
- **Oversized bundled assets** (Finding #7):

  ```
  3.2M  assets/images/profile.png
  2.8M  assets/images/splash-backdrop.jpg   ← required at SplashAnimation/LogoBurst.tsx:95 (cold path)
  673K  assets/images/featuredCard-001.jpg
  571K  assets/images/backdrop.svg
  382K  assets/images/auth-backdrop.jpg     ← required at AuthScreenLayout.tsx:126
  139K  assets/icon-master.png
  ```

  `splash-backdrop.jpg` at 2.8 MB is decoded on the **cold-start critical path**. A 3.2 MB PNG decodes
  to tens of MB of uncompressed bitmap in memory.

  Only **3 `assets/images` requires exist in all of `src/`** (`fab-logo.png`, `splash-backdrop.jpg`,
  `auth-backdrop.jpg`) — so `profile.png` (3.2 MB), `featuredCard-001.jpg` (673 K), `backdrop.svg`
  (571 K) and `icon-master.png` appear to be **unreferenced dead weight in the bundle**. Worth
  confirming against the native projects before deleting, but that is ~4.5 MB of likely-removable
  bundle.
- **SVGs are fine**: total SVG payload across `src/` + `assets/` is 1.2 MB, and only
  `backdrop.svg` (571 K) exceeds 50 KB — and it is not referenced from `src/`. No large SVG is
  rendered inside a list.

---

## 6. Animation

```
17  files importing react-native-reanimated     41  files referencing Animated
24  useNativeDriver: true                       11  useNativeDriver: false
 7  LayoutAnimation                             18  lottie references
 0  useAnimatedScrollHandler                     9  onScroll=   /   9  scrollEventThrottle
```

Reanimated 4 + `react-native-worklets` 0.8 are installed and used in 17 files (splash, modals,
bottom sheet, keyboard lift, zoomable images, form fields) — the harder animations are already on
the UI thread.

**Scroll handlers are correct despite 0 `useAnimatedScrollHandler`**: all 9 `onScroll` sites pair
with `scrollEventThrottle`, and the home feed uses the native-driven legacy path properly —

```tsx
// src/screens/home/HomeScreen/index.tsx:150-153
const handleScroll = useRef(
  Animated.event([{ nativeEvent: { contentOffset: { y: scrollY } } }], { useNativeDriver: true })
).current
```

with the title driven by `transform` + `opacity` only. That is the right call; no change needed.

### The one real animation problem (Finding #5)

`src/components/global/LoadingSpinner/index.tsx` runs **9 `Animated.Value`s (3 bars × opacity +
height + translate) in an infinite `Animated.loop`, all with `useNativeDriver: false`**, and two of
the three animated properties per bar are **layout properties**:

```tsx
// LoadingSpinner/index.tsx:42-50
Animated.timing(height, { toValue: barMaxHeight, duration: 300, useNativeDriver: false }),
Animated.timing(y,      { toValue: barLift,      duration: 300, useNativeDriver: false }),
```

Animating `height` forces a layout pass per frame on the JS thread, and the component is mounted
during the cold-start gate (`HomeScreen` renders `<LoadingIndicator />` in both the pre-ready branch
and the cross-fade overlay) — exactly when the JS thread is busiest.

Minimal fix, visually identical: drive `scaleY` + `translateY` instead of `height` + `y` and flip to
`useNativeDriver: true`. Give the bar a fixed `barMaxHeight` and animate `scaleY` from
`barMinHeight/barMaxHeight` → `1` with `transformOrigin` handled by the existing translate.

The other three `useNativeDriver: false` files are lower priority: `SyncedSlider` (:93, :122),
`Tabs/index.tsx` (:68, :75 — animates `x`/`width` of the active indicator), `Tabs/Text.tsx` (:95).
The `Tabs` indicator is a good `scaleX`/`translateX` candidate later.

`LayoutAnimation` (7 sites) is unrelated to Reanimated and can conflict with it on the New
Architecture — worth a look, but not a measured problem here.

---

## 7. Startup work

**Engine config is correct:**

```
ios/Podfile:24              :hermes_enabled => true
android/gradle.properties:37  newArchEnabled=true
android/gradle.properties:41  hermesEnabled=true
ios/Podfile.lock:16         hermes-engine (0.14.0)
```

Hermes on both platforms; New Architecture on for Android. (iOS `newArchEnabled` is not set in the
Podfile — RN 0.83 defaults it on, but worth confirming it is intentional.)

**`index.js` is minimal (13 lines)** — `react-native-get-random-values`, `gesture-handler`,
`reanimated`, then `./src/lang/i18n`. `polyfill.js` is **not loaded into the app at all**: it is a
36-line `NODE_OPTIONS --require` shim used only by the `ios`/`android`/`start` npm scripts for the
Metro/CLI process. Zero runtime cost in the app. Good.

**No module-level side effects in `src/`** — a scan for column-0 calls and top-level `await` across
all `.ts`/`.tsx` found **zero** hits. Everything is inside components or exported functions.

**`App.tsx` pre-paint work** is deliberately sequenced and non-blocking (`App.tsx:92-108`):
`initAnalytics()` → `trackInstallOnce()` → `trackAppOpen()` run inside an async IIFE in `useEffect`,
so they do not block first paint. `RealtimeBridge` is explicitly gated behind `AppReadyProvider` to
keep the WebSocket handshake off the cold path (comment at `App.tsx:113-118`). `SessionContext`'s
Cognito `fetchAuthSession()` runs in an effect, not at import. This is all well done.

**The one startup cost worth questioning (Finding #8):**

```
src/components/global/SplashAnimation/index.tsx:20   const SPLASH_MIN_MS = 2500
src/components/global/SplashAnimation/index.tsx:41   const SPLASH_MAX_MS = 45000
```

A **hard 2.5 s minimum splash on every launch**, enforced by `setTimeout(() => setMinElapsed(true),
SPLASH_MIN_MS)` at :268. Even when the home feed is ready in 600 ms the user waits 2.5 s. If the
2.5 s exists to cover the burst animation, gate on the animation's completion rather than wall-clock;
otherwise consider 1,200–1,500 ms. The 45 s `SPLASH_MAX_MS` backstop is fine (it is only a ceiling).
Also note `splash-backdrop.jpg` (2.8 MB, §5) is decoded here.

---

## 8. Hot-path waste

```
  0   console.log in src/          16  console.* total (all warn/error)
 28   JSON.parse / JSON.stringify
 11   AsyncStorage.* call sites
```

- **Zero `console.log` in `src/`** — genuinely clean, no action.
- **No `JSON.parse`/`stringify` or `AsyncStorage` reads found on render paths** — the 28 JSON and 11
  AsyncStorage sites are in services and effects, not component bodies.
- **Array work in render**: `src/components/posts/PostItem.tsx` (a list row) filters twice in the
  component body without `useMemo`:

  ```tsx
  PostItem.tsx:200   .filter((tag) => tag.id && tag.name)
  PostItem.tsx:232   const attachments = contentItems.filter((item) => item.type !== 'RICH_TEXT')
  ```

  Small arrays, so the allocation is minor next to §1 — but it also produces a **new array identity
  every render**, which will defeat `React.memo` on any child once §4 lands. Wrap both in `useMemo`
  when memoizing `PostItem`.
- `src/components/posts/composer/PostComposer.tsx:259` builds `new Set(...)` in the component body
  (1,639-line component), again a fresh identity per render.
- No regex construction, date parsing, or large-array sorting found in `renderItem` bodies.

---

## 9. Memory / leaks

```
Keyboard.add*             22        AppState.addEventListener   4 (+1 in App.tsx)
Linking.addEventListener   1        Dimensions.addEventListener 0
.remove() / removeEventListener    24
setInterval  2  /  clearInterval  1
setTimeout  55  /  clearTimeout  41
```

**Cleanup discipline is good overall.** `App.tsx` handles both of its subscriptions correctly —
`focusManager` returns `() => subscription.remove()` (:28) and the analytics effect returns
`() => subscription?.remove()` (:107). `ShareGuardContext.tsx:41-45` properly adds/deletes its
listener from a module-level `Set`. `HomeScreen:124` cancels its `InteractionManager` handle.

Two items to verify manually:

1. **`setInterval` 2 vs `clearInterval` 1** — one interval may not be cleared. Worth locating.
2. **`setTimeout` 55 vs `clearTimeout` 41** — a 14-call gap. Many are legitimately fire-and-forget
   (the splash's own backstops at `SplashAnimation/index.tsx:188, 213` are intentional per the
   documented safety contract), but timers that `setState` after unmount would warn and retain.

No realtime subscription without unsubscribe was found; `RealtimeBridge` is gated by `AppReadyProvider`.

---

## Top 5 — concrete minimal fixes

### 1. Memoize the style buckets (biggest win, zero component changes)

```ts
// NEW src/styles/utils/memoizeByTheme.ts
import { ThemeTokens } from './types'
export const memoizeByTheme = <T>(fn: (theme: ThemeTokens) => T) => {
  const cache = new WeakMap<ThemeTokens, T>()
  return (theme: ThemeTokens): T => {
    let hit = cache.get(theme)
    if (hit === undefined) { hit = fn(theme); cache.set(theme, hit) }
    return hit
  }
}
```

```ts
// src/styles/components/cards.ts  (repeat for the other 36 buckets — one line each)
export const getCardStyles = memoizeByTheme((theme: ThemeTokens): CardStyles => ({ /* unchanged */ }))
```

Safe because all 37 `get*Styles` are single-arg, pure, and `theme` is a stable module-level constant.

### 2. Memoize the ThemeProvider value

```tsx
// src/styles/utils/ThemeProvider.tsx:20-28
const value = useMemo(() => ({ theme, themeMode, setThemeMode }), [theme, themeMode])
return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
```

### 3. Memoize the five list-row components

```tsx
// src/components/feed/FeedListingCard/index.tsx:528
export default React.memo(FeedListingCard)
```

Same for `ListingCard:533`, `PostItem:533`, `FeedBidCard`, `FeedBulkCard`. Only effective after
fix #1 stabilises the style object identity — land them together.

### 4. Turn on react-hooks linting

```js
// eslint.config.mjs — add to the rules block
import reactHooks from 'eslint-plugin-react-hooks'
{ plugins: { 'react-hooks': reactHooks },
  rules: { 'react-hooks/rules-of-hooks': 'error', 'react-hooks/exhaustive-deps': 'warn' } },
```

Start at `warn` — 745 unchecked hook sites make `error` a build-breaker on day one.

### 5. Take LoadingSpinner off the JS thread

```tsx
// src/components/global/LoadingSpinner/index.tsx — replace height/y with scaleY/translateY
Animated.timing(scaleY, { toValue: 1, duration: 300, useNativeDriver: true }),
Animated.timing(y,      { toValue: barLift, duration: 300, useNativeDriver: true }),
// render: style={[ ..., { height: barMaxHeight, transform: [{ scaleY }, { translateY: y }], opacity } ]}
```

Visually identical, removes a per-frame layout pass from the cold-start critical path.

---

## Commands used for every count

```bash
rg -n 'get[A-Za-z]+Styles\(theme' src/ App.tsx | wc -l                    # 520
rg -l 'get[A-Za-z]+Styles\(theme' src/ App.tsx | wc -l                    # 328
rg -n 'useMemo\(\(\) => get[A-Za-z]+Styles' src/ | wc -l                  # 7
rg -o 'get[A-Za-z]+Styles\(theme\)' src/ App.tsx --no-filename | sort | uniq -c | sort -rn
rg -o 'export const get[A-Za-z]+Styles = \([^)]*\)' src/styles/components/*.ts --no-filename | rg -vc 'theme: ThemeTokens\)$'   # 0
rg -c 'useTheme\(' src/ --no-heading | awk -F: '{s+=$2} END{print s}'     # 313
rg -n '<FlatList' src/ | wc -l;  rg -n 'FlashList' src/ | wc -l           # 24 ; 0
rg -n 'getItemLayout' src/ | wc -l                                        # 1
rg -n 'React\.memo|\bmemo\(' src/                                         # 1 site
rg -n '<Image\b' src/ | wc -l;  rg -n 'FastImage|expo-image' src/ | wc -l # 93 ; 0
rg -c 'useNativeDriver: false' src/ --no-heading | awk -F: '{s+=$2} END{print s}'  # 11
rg -c 'console\.log' src/ --no-heading | awk -F: '{s+=$2} END{print s}'   # 0
ls node_modules/eslint-plugin-react-hooks                                 # not found
rg -n 'hermes|newArchEnabled' android/gradle.properties ios/Podfile
find src assets -type f \( -name '*.png' -o -name '*.jpg' \) -size +100k -exec ls -lh {} \;
node <scratchpad>/measure.js    # per-bucket runtime property counts
node <scratchpad>/bench.js      # 1.847 ms per card-row style rebuild
```
