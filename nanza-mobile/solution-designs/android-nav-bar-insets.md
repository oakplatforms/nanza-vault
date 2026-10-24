---
tags: [nanza-mobile, solution-design, plan, android, modals, ui-fixes]
---

# Android three-button navigation covers sheet buttons

**Status:** **built 2026-10-23, uncommitted, pending device QA.** Plan written 2026-10-23 (Skylar,
voice), architect cross-review folded in the same day, built the same evening: 4 files
(`App.tsx`, `Modal`, `BottomSheet`, `modals.ts`), tsc + lint clean on them. Remaining: the QA
list below — the Android ≤14 emulator check decides whether the nested-provider fallback is needed.
nanza-mobile only — no API change. When it lands, fold the durable part (the bottom-inset rule
for sheets, the root `SafeAreaProvider`, the Android keyboard-height asymmetry) into
[[ui-fixes|UI / UX Fixes]] and [[design-system|Design System]], then delete this file.

## Why

Android users running **three-button navigation** (the back / home / recents bar instead of
gesture navigation) report that the buttons at the bottom of our sheets — the **Add to
Collection** quantity sheet off the card scan, and the same sheet off search and collection
detail — are cut off and sit under the system bar. Gesture-navigation users don't see it.

## Root cause

1. **The app is already edge-to-edge on Android 15+ whether we opted in or not.**
   `android/build.gradle` targets SDK 36. Android 15 forces every window of an app targeting
   SDK 35+ to draw under the system bars, and Android 16 removes the opt-out for SDK 36. We never
   set React Native's `edgeToEdgeEnabled` gradle property, so RN's own
   `isEdgeToEdgeFeatureFlagOn` is false and RN isn't managing the bars for us; the OS simply
   stopped reserving space for them. The gesture bar is ~24dp and translucent, so content under
   it goes unnoticed. The three-button bar is **~48dp** and scrimmed, so anything in that band is
   hidden and un-tappable.

2. **Screens cope; sheets don't.** Screens read `useSafeAreaInsets()` (react-navigation's
   `SafeAreaProviderCompat` inside each navigator) and the bottom bars pad by the inset —
   `getBottomBarInset(theme, insets.bottom)` in the search footers, collection detail and
   `JoinChatFooter`; `insets.bottom + gutter` in `DetailActionBar`; `insets.bottom + sm` in the
   composer. The shared sheet primitives do not:
   - `Modal` (`src/components/modals/Modal`) reads the inset context but only uses `insets.top`
     (to clamp the keyboard push). The float above the screen edge is the fixed
     `modalTouchContent.paddingBottom = theme.spacing.base` (20) in `styles/components/modals.ts`.
     With a 48dp bar, the bar covers the bottom **28dp of the sheet**: all of its 28pt bottom
     padding (`base + sm`), so the xxl action button's lower edge lands exactly on the bar and
     the 44pt glass corners are clipped.
   - `BottomSheet` (in-tree, used by `ModalStackRenderer`) shares the same `modalTouchContent`
     float and never reads insets at all.
   - The RN `<Modal>` under `Modal` is rendered with `statusBarTranslucent`. In RN 0.83 that
     means: no `fitsSystemWindows` on the dialog's root `FrameLayout`, `disableEdgeToEdge()`
     (`setDecorFitsSystemWindows(true)`) on the dialog window, and an inset listener that zeroes
     only the **top** inset. So bottom padding for the dialog is the **OS's job on Android
     11–14** (the decor still consumes the nav-bar inset), **nobody's on 15+** (the OS ignores
     `setDecorFitsSystemWindows(true)` for target 35+), and on **API 24–29** the only RN-side
     bottom padding was that `fitsSystemWindows`, which `statusBarTranslucent` removes. The
     Android ≤14 behaviour therefore rests on platform conduct, not on anything we control —
     see QA.

3. **There is no root `SafeAreaProvider`** (latent, not part of the reported chain). `App.tsx`
   never mounts one; only react-navigation's per-navigator Compat providers exist. The reported
   sheets mount inside a navigator and already see `insets.bottom ≈ 48` — the bug is purely that
   `Modal` never reads it. But `ModalStackRenderer` and `UpdateGate` mount outside any navigator
   (`AppNavigator.tsx` ~884), where `Modal`'s `useContext(SafeAreaInsetsContext) ?? zeros`
   fallback silently yields 0, and `BottomSheet` has no read at all. Without a root provider the
   fix below would not reach those sheets. (The comment in `Modal` blaming the RN `<Modal>`
   portal is wrong — React context crosses RN's `<Modal>` fine; it's the mount point.)

The modal comment says the 20pt float is "the 20 the auth card sits at". The auth card actually
floats `Math.max(insets.bottom, theme.spacing.base)` (`AuthScreenLayout.tsx`), so the sheet has
already drifted from the thing it claims to mirror.

### A second symptom the same fix cures (Android)

RN reports the Android keyboard height as `ime.bottom − systemBars.bottom` — measured **above
the nav bar** (`ReactRootView.java` ~920). Today a sheet pushed by that amount with a 20 float
ends **28dp under the keyboard** on three-button (4dp on gesture). The composer picker layers
already document and compensate for this asymmetry (`CardPickerLayer.tsx` ~482,
`PostComposer.tsx` ~1510: `+ insets.bottom` on Android only). With the inset float the sheet
lands 8dp above the keys. On iOS the keyboard height **includes** the home-indicator region, so
the same float would double-count there — handled in step 2.

## Proposed change

### 1. Mount one root `SafeAreaProvider`

`App.tsx` wraps the tree — **above `AppRoot`, so the splash is inside it** — in
`SafeAreaProvider` from `react-native-safe-area-context`. Why it is safe next to
react-navigation: with a provider already in context, `SafeAreaProviderCompat` renders a plain
`View` instead of its own native provider, so there is no doubling — net fewer native inset
listeners.

`initialMetrics`: `initialWindowMetrics` is **null on Android** when there is no current
activity or the decor hasn't measured yet, and a `SafeAreaProvider` with null metrics renders
**no children** until the first native inset event — a blank first frame for the whole app.
Guard it the way react-navigation does:

```ts
initialMetrics={initialWindowMetrics ?? { frame: { x: 0, y: 0, width, height }, insets: { top: 0, right: 0, bottom: 0, left: 0 } }}
```

(`width`/`height` from `Dimensions.get('window')`.) `Modal` then drops the defensive
`useContext(SafeAreaInsetsContext) ?? zeros` read for a plain `useSafeAreaInsets()`, and the
stale portal comment goes with it. `BottomSheet` gains the same read. **Sequencing:** step 1
lands before step 2 — `BottomSheet` is mounted under `ModalStackRenderer` outside the
navigator and `useSafeAreaInsets()` throws with no provider in scope.

### 2. Sheets float by the bottom inset, through the one existing helper

`modalTouchContent` loses its static `paddingBottom`. `Modal` and `BottomSheet` set it inline:

```ts
const sheetFloat = Math.max(theme.spacing.base, getBottomBarInset(theme, insets.bottom))
```

`getBottomBarInset` (`styles/components/glass.ts`) is `insets.bottom + spacing.sm` — the rule
the footers use, so a sheet clears the system bar by the same 8pt gap and stays at today's 20
wherever there's no inset. Resulting floats:

| Device | `insets.bottom` | float today | float after |
|---|---|---|---|
| Android 15+, three-button | ~48 | 20 (28dp hidden) | 56 |
| Android 15+, gesture | ~24 | 20 (4dp under) | 32 |
| Android ≤ 14 (not edge-to-edge) | 0 | 20 | 20 |
| iOS, home indicator | 34 | 20 | 42 |

**Keyboard push.** Android: unchanged — the push is measured above the bar, so the float is
*required* (see the second symptom above); expected gap above the keys becomes 8. iOS: the
keyboard height already spans the indicator region, so subtract the float's inset share from
the push: `effectiveKeyboardOffset = keyboardOffset − (sheetFloat − theme.spacing.base)` on iOS
only, keeping the 20pt gap above the keys the sheet has today. The existing Android clamp on
`insets.top` stays.

`bare` sheets (`EntityModal`) now float 56 on three-button with zero internal padding — fine by
the rule, confirm on device. `position="center"` currently has no users in `src`; it is
untouched either way.

### 3. Sweep the direct `react-native` `Modal` users

Eleven components build on the shared `Modal` and inherit the fix. Every other file importing
`Modal` from `react-native`, graded:

- **No change** — already pad `insets.bottom` in the host, or have no bottom controls:
  `JoinChatFooter`, `InAppBrowserSheet`, `GroupAddItemsSheet`, `AvatarViewer`, `ImageViewer`
  (reads `insets.top` only, no bottom controls), `EntitySearchOverlay` (top-anchored 33%
  positioner), `PostComposer` (its own translucent Modal hosts `TagPickerLayer`, which pads
  `insets.bottom`), `PostDetailScreen` (hosts `PostComposer`, whose icon row pads
  `insets.bottom + sm`). Verify on device, don't edit.
- **Leave** — non-translucent full-screen Modals that RN pads itself via `fitsSystemWindows`:
  `OfferSellerListingsPicker`, `OrderTrackingModal` (header + WebView; its
  `presentationStyle="pageSheet"` is iOS-only).

## Alternatives weighed

- **`navigationBarTranslucent` on the RN Modal.** Forces `enableEdgeToEdge()` on the dialog on
  *every* Android version while the host provider still reports 0 on ≤14 — makes ≤14 worse
  unless the whole app opts in. Rejected.
- **Whole-app `edgeToEdgeEnabled` / `react-native-edge-to-edge` now.** Forces both RN Modal
  translucency getters true and gives host and dialog identical geometry on all Android
  versions — it is the change that would retire the ≤14 uncertainty in root cause 2. It also
  flips the status bar and `StatusBar` module behaviour everywhere, so it is its own pass. The
  fix above is correct with the flag on or off because it keys off the measured inset. Deferred
  (open question 1), with the dependency noted.
- **`SafeAreaView edges={['bottom']}` inside the sheet.** Pads *inside* the glass; the bar still
  clips the 44pt corners. Wrong tool.
- **Reading `initialWindowMetrics` directly in `Modal`.** Raw decor insets — wrong on ≤14. No.
- **A nested `SafeAreaProvider` as the first child inside `RNModal`.** Measures the *dialog's
  own* window (its root view is the dialog `DecorView`) and seeds from the parent's insets, so
  it is correct in all four regimes without assuming host/dialog geometry parity — the host
  insets only provably match the dialog on 15+. Cost: one native view per open sheet and the
  inset read moves into a child component. **This is the fallback:** adopt it if the Android ≤14
  QA below shows the dialog under the bar (where the host inset is 0 and the float can't help).
  Otherwise keep the plan as written.

## Non-goals

- RN edge-to-edge opt-in (above).
- `KeyboardLift`, the composer picker layers, and the footer bars — they already do this.

## QA

Toggle the bar on the device (Motorola: Settings → Gestures → System navigation; it only
reproduces on **Android 15+**) and run both ways:

1. Collection detail → camera → scan → tap a match → quantity sheet: button fully visible and
   tappable, glass corners intact, 8pt gap above the bar.
2. Search → hold a card → quantity sheet. Collection detail → `QuantityModal`.
3. `ActionModal` rows (post/entity "more" sheets), `WithdrawModal`, `WriteReviewModal`,
   `CompleteProfileModal`, `AuthModal`, `CustomerGateModal`, and a `bare` `EntityModal`.
4. `ModalStack` sheets at two nested levels (`getMaxHeightForLevel`) — both adopt the float.
5. **Keyboard open inside a sheet with a text input**, expected gaps above the keys:
   Android three-button **8**, Android gesture **8**, iOS **20** (same as today). No double
   offset on iOS; the Android `insets.top + xl` headroom clamp still holds on a tall (84%) sheet
   on a small device.
6. iOS: the 20 → 42 float next to the auth card (34).
7. **Android ≤ 14, mandatory, not optional:** API 29 and API 34 emulators with three-button nav
   — the sheet must sit above the bar with nothing moving. If it is under the bar, switch to the
   nested-provider fallback.
8. Cold start after the root provider: no blank first frame, splash intact.
9. Through the Android 15 three-button scrim: `modalOverlay` and `modalBottomExtension` read
   correctly (`showBottomExtension` defaults true; `AuthModal`, `LoginModal`,
   `ForgotPasswordModal`, `CompleteProfileModal` keep it on).
10. Direct-Modal sweep (section 3) spot-checked on three-button: `JoinChatFooter`,
    `GroupAddItemsSheet`, `TagPickerLayer` in the composer.

## Decisions (Skylar, 2026-10-23, voice)

1. **Float rule: `getBottomBarInset`, everywhere.** iOS sheets move from 20 to 42 above the
   screen edge (indicator + 8). The alternative — the auth card's `Math.max(insets.bottom, base)`,
   34 on iOS — would leave three-button Android sheets flush on the bar with no gap. Skylar's
   brief: keep everything inside the safe area of the page; sheets and footers share one bottom
   rhythm.

## Open questions

1. Opt into RN edge-to-edge now, or as a separate pass (recommended)? Separate pass also
   closes the Android ≤14 dialog-geometry uncertainty for good.

## Related

- [[ui-fixes|UI / UX Fixes]] — where the durable rule lands.
- [[design-system|Design System]] — glass sheet recipe, `getBottomBarInset`.
- [[scan|Scan & Camera]], [[collections|Collections]] — the reported flow.
- [[scan-quick-add|Scan Quick-Add]] — the new swipeable quantity modal in that plan must adopt
  the same float.
