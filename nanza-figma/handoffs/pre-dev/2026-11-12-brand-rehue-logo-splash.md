---
tags: [nanza-figma, handoff, pre-dev, nanza-mobile, brand, palette, logo, splash, android, ios]
bucket: pre-dev
status: open
date: 2026-11-12
originally: 2026-11-09 (flat handoffs/ folder, before the buckets)
verified: 2026-11-12 against dev ac083896 (v0.4.3)
repo: nanza-mobile
scope: tokens + shared type + assets (SVG, iOS launch PNGs) + native Android + splash animation fix + plus-button gradient
---

# Brand re-hue: persian-rose palette, logos, links, plus button, splash fixes

The brand moves from magenta (hue ~300°, `#FF4BFF`) to **persian-rose** (hue ~329°, `#ED3594`).
Text links get their own `link` token in the lighter brand pink (`primary.400`), and the
compose plus button picks up the logo's soft gradient. **All values are final and device-tested by design.** Nothing here
is on trial.

> [!info] Pre-dev: why this one needs planning
> Most of this is a token swap. These parts need dev judgement:
> - **Shared infra:** one new optional key, `link?: string`, in `src/styles/utils/types.ts`
>   (approved).
> - **Native Android:** `colors.xml` + `styles.xml`. These need a **native rebuild**. The
>   splash item only shows on Android 12+.
> - **Native iOS assets:** the `LaunchLogo` PNGs are replaced (files supplied, §10).
> - **Animation math:** `LogoBurst` clip matrix fix (§9).
> - **Component changes:**
>   - The bottom nav's active state stops using the brand colour (§6).
>   - The compose plus discs gain a `LinearGradient` layer and a pressed state, via
>     `Pressable`'s children render prop (§11).
>
> No navigation, API or data changes.

> [!warning] App-wide
> One palette swap recolours every pink in the app. All of them derive from `colors.primary`
> (138 refs, 97 of them `primary['500']`) plus the semantic tokens `action` and `highlight`. No
> component hard-codes the old magenta. Outside the tokens, the only old-magenta values are in
> the three logo SVGs, Android's `notification_accent` and the iOS launch PNGs. This handoff
> replaces all of them. The app icons are the one exception (see the end).

## Reference build
- **Branch `throwaway-hue-change`** (designer's machine, uncommitted) = dev `ac083896` plus
  exactly this handoff. Nothing else is in it, apart from a local-only Figma devtools hook in
  `App.tsx`.
- **[[assets/2026-11-12-brand-rehue-logo-splash/reference.patch|reference.patch]]** in this
  handoff's assets folder is that branch's diff against dev: 17 files, +127 / −70. It
  excludes `App.tsx` and the launch PNGs, which are separate files in the same folder.
  - Use it to check your work, or apply it with `git apply` if your branch is close to dev.
  - Either way, this document is the spec.
- Verified 2026-11-12: every target class still exists on dev under the same name, at its
  pre-re-hue value.
- An older stash ("local handoff builds") also carries an early version of this work, mixed
  with other handoffs. Ignore it. Its ring change to `cards.ts` and its splash-backdrop
  artwork swap are **not** part of this handoff.

## Changes

| # | Change | Kind | Verify |
|---|---|---|---|
| 1 | `primary` scale → persian-rose (dark + light) | token | device-tested |
| 2 | Logo SVGs: new fill + gradient, **stroke path removed** | asset | device-tested |
| 3 | `action` → `primary.500` | token | device-tested |
| 4 | `highlight` → `primary.400` (ring, play badge, circle badge, Buy tags) | token | device |
| 5 | New `link` token → `primary.400` pink (post #tags / mentions, Show more, composer tag runs) | token + **shared type** | device-tested |
| 6 | Bottom nav active tab → `textSecondary` (bright white) | **component** | device-tested |
| 7 | Android `notification_accent` → `#ED3594` | **native** | Android push |
| 8 | Android 12+ system splash → app background, no icon disc | **native** | Android 12+ build, light + dark mode |
| 9 | Splash burst: black band outside the grey edge (clip/zoom bug) | **animation** | iOS + Android device |
| 10 | iOS native launch logo PNGs → new pink, no stroke | **native** asset | iOS cold launch |
| 11 | Compose plus discs (nav + Home FAB): subtle logo gradient, flat when pressed | **component** | device-tested |

### 1. Palette (`src/styles/tokens/dark.json` + `light.json`, `colors.primary`)

Make the same change in both files.

| | 50 | 100 | 200 | 300 | 400 | **500** | 600 | 700 | 800 | 900 | 950 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Old | FFF3FE | FEE6FF | FFCCFF | FFA3FD | FF6DFA | FF4BFF | E316DE | BD0EB6 | 9A0E93 | 7E1176 | 55004F |
| New | FDF2F8 | FCE7F3 | FCCEE9 | FAA7D6 | F670B9 | **ED3594** | DE247B | C11560 | 9F1550 | 851645 | 510625 |

### 2. Logos (`assets/images/logo.svg`, `logo-simple.svg`, `logo_full.svg`)
- Base fill `#FF4BFF` → **`#ED3594`**.
- Gradient stops `#FF6DFA → #E415DE` → **`#FF89C6 → #ED3594`**. This is the 0.6-opacity overlay
  path.
- **Delete the stroke path.** Each file ends with a `<path>` that redraws the outline with
  `stroke="#FF6DFA"` (width 0.86 / 0.8 / 0.15). It caused a visible edge on the splash.
- Consumers: `LogoBurst` (JS splash), `Header`, `LoginModal`, `DrawerModal`.

### 3–5. Semantic tokens (`theme` section of both token files)

| Token | Old | New | Used by (dev today) |
|---|---|---|---|
| `action` | `primary.400` | **`primary.500`** `#ED3594` | `floatingNavFab`, `floatingNavPlus` (+ button), `sendIconButtonActive`, `audienceMenuRowTextSelected`, `stepIndicatorDotCompleted`, `requestedUnreadDot` |
| `highlight` | `primary.300` | **`primary.400`** `#F670B9` | `circleThumbRingBrand`, `circleThumbPlayBadge`, `circleThumbBadge` (cards.ts); `intentTagBuy`, `intentTagBuyCompact` (badges.ts) |
| **`link`** (new) | — | **`primary.400`** `#F670B9` (dark + light) | `tagMention`, `tagMentionPressed` (at 0.7), `showMoreText` (posts.ts); `inlineTagRun` (postComposer.ts) |

- **`highlight` is a token change only.** Its consumers already read it, so `cards.ts` and
  `badges.ts` don't change. The play badge keeps matching the ring.
- **`link`:**
  - Add `link?: string` beside `action?` in `ThemeTokens`. `resolveThemeColors` maps
    `theme` keys generically, so no other infra changes.
  - Point the four classes at `theme.colors.link || theme.colors.action`. In
    `tagMentionPressed`, that goes inside the existing `hexToRgba(…, 0.7)`.
- **Why `link` is pink 400:**
  - Links stay in the brand pink, a step lighter than `action` (500), so they read as links
    against the warm body text without the weight of the + button's fill.
  - A cool grey (`neutral.300`) was tried on device and walked back: links read better pink.
  - It shares a value with `highlight`. They stay separate tokens so either can move alone.

### 6. Bottom nav active state
- `layout.ts` `floatingNavLabelActive.color` and `FloatingTabBar` `activeColor` →
  `theme.colors.textSecondary || theme.colors.text` (`#F8F6F4`). Both used to read `action`.
- Inactive tabs stay `secondaryText`.

### 7–8. Android (`android/app/src/main/res/values/`)
- **`colors.xml`:** `notification_accent` `#FF4BFF` → **`#ED3594`**. It is the push icon and
  app-name tint via `firebase.json` `messaging_android_notification_color`, and mirrors
  `primary.500`.
- **`styles.xml` `AppTheme`:** Android 12+ draws its own launch splash before the app's theme
  applies: the launcher icon in a circle on a system background.
  - **Why it shows a disc:** unset, the background follows the system light/dark mode, and
    the circle takes the icon's `#000000` background. The result is a black disc on white,
    or the reverse.
  - **Fix:** pin both to the app background, which `windowBackground` already uses:
  ```xml
  <resources xmlns:tools="http://schemas.android.com/tools">
    …
    <item name="android:windowSplashScreenBackground" tools:targetApi="31">@color/app_window_background</item>
    <item name="android:windowSplashScreenIconBackgroundColor" tools:targetApi="31">@color/app_window_background</item>
  ```
  API levels below 31 ignore these items. No Play Console setting is involved.

### 9. Splash burst edge (`src/components/global/SplashAnimation/LogoBurst.tsx`)
**Symptom:** a black band outside the grey border during the burst. At rest there is a dark
ring round the mark, and it gets thicker the longer the splash holds.

**Cause:**
- The artwork's logo-shaped `ClipPath` uses `userSpaceOnUse`, so it resolves in the image's
  own user space.
- That space **includes the image's `matrix`**, which carries the push-in `artZoom`
  (1 → 1.12 over 10 s).
- So the hole grew up to 12% larger than the underlay's hole and the border, and the dark
  underlay showed between them.

**Fix:** in `clipProps`, divide the push-in back out. Both matrices pivot on `PIVOT`, so the
result composes back to exactly the burst scale:
```ts
const clipProps = useAnimatedProps<AnimatedPathProps>(() => {
  const k = scale.value / artZoom.value
  return { matrix: [k, 0, 0, k, PIVOT * (1 - k), PIVOT * (1 - k)] }
})
```
Also correct the push-in comment. It currently claims the image's zoom "leaves the hole, the
border and the mark untouched", which is wrong.

### 10. iOS native launch logo (`ios/Assets.xcassets/LaunchLogo.imageset/`)
`LaunchScreen.storyboard` shows `LaunchLogo` before JS loads. Without this change, iOS
flashes the old magenta before the JS splash.
- **Replace** `nanza_logo_1x.png` (55×72), `_2x` (110×144) and `_3x` (165×216) with the files
  in [[assets/2026-11-12-brand-rehue-logo-splash/|this handoff's assets folder]]
  (`nanza-figma/handoffs/pre-dev/assets/2026-11-12-brand-rehue-logo-splash/`).
- Keep the same filenames, so `Contents.json` doesn't change.
- They are rendered from the new `logo.svg`, aspect-fit on a transparent background, with the
  old framing.
- iOS caches launch screens: delete the app before reinstalling to see the change.

### 11. Compose plus discs: subtle logo gradient (`JoinChatFooter.tsx` + `layout.ts`)
- **Look:** the disc keeps its flat `action` `#ED3594` fill. Over it goes a `primary.400`
  `#F670B9` overlay, **75% opacity at the top, fading to 0 at 85% down.**
  - The result is a soft lift at the top, echoing the logo.
  - This was tuned on device: a full 400 → 500 ramp was too much, and 50% → 0 by 70% was
    too flat.
- **Pressed:** the gradient drops and the disc shows flat `action`. Size is unchanged, so the
  existing "no press animation" rule (Skylar, 2026-08-14) still holds.
- **Where:** both compose discs, kept identical. `placement="navPlus"` is the disc inside the
  nav's glass circle (`floatingNavPlus`); `placement="fab"` is the Home corner FAB
  (`floatingNavFab`). Both render in `JoinChatFooter`.
- **How (reference build):**
  - **The gradient:** one `react-native-linear-gradient` element, `plusGradient`:
    colours `[hexToRgba(primary['400'], 0.75), hexToRgba(primary['400'], 0)]`,
    `locations={[0, 0.85]}`, start `{x:.5,y:0}`, end `{x:.5,y:1}`, `pointerEvents="none"`.
  - **Placement:** it is the first child of each disc, so it sits under the plus icon.
  - **Pressed:** both `Pressable`s switch to the children render prop
    (`{({ pressed }) => …}`) and render `pressed ? null : plusGradient`.
  - **Style:** new class `floatingNavPlusGradient` in `layout.ts`: an absolute fill with
    `radius.max`.
  - **Keep the discs' `action` `backgroundColor`.** The overlay and the pressed state both
    rely on it.

## Code map (dev @ `ac083896`)
- `src/styles/tokens/dark.json`, `light.json`: `colors.primary`. In `theme`: `action` →
  `primary.500`, `highlight` → `primary.400`, new `link` → `primary.400`.
- `src/styles/utils/types.ts`: `link?: string` (approved shared-infra edit).
- `src/styles/components/posts.ts`: `tagMention`, `tagMentionPressed`, `showMoreText` → `link`.
- `src/styles/components/postComposer.ts`: `inlineTagRun` → `link`.
- `src/styles/components/layout.ts`: `floatingNavLabelActive` → `textSecondary`; new
  `floatingNavPlusGradient`.
- `src/components/global/FloatingTabBar/index.tsx`: `activeColor` → `textSecondary`.
- `src/components/posts/composer/JoinChatFooter.tsx`: `plusGradient` in the navPlus and fab
  discs, dropped while pressed.
- `src/components/global/SplashAnimation/LogoBurst.tsx`: `clipProps` divides by `artZoom`.
- `assets/images/logo.svg`, `logo-simple.svg`, `logo_full.svg`: new colours, stroke path
  removed.
- `android/app/src/main/res/values/colors.xml`, `styles.xml`.
- `ios/Assets.xcassets/LaunchLogo.imageset/nanza_logo_{1x,2x,3x}.png`: from the assets folder.
- **Not touched:** `cards.ts`, `badges.ts`. They follow the `highlight` token.

## QA
- **Pinks:**
  - + button, send icon, step dots, request dot and audience-menu selected row are
    `#ED3594`.
  - Circle-thumb rings, play badges, circle badges and "Buy" tags are `#F670B9`.
  - No old magenta anywhere.
- **Links:** post #tags, mentions, "Show more" and composer tag runs are pink `#F670B9`,
  a step lighter than the + button. Pressed tags dim to 70%.
- **Nav:** the active tab label and icon are bright white. Inactive tabs are unchanged.
- **Plus discs (nav + Home FAB):** a soft lighter top easing into `#ED3594`, with the white
  plus icon on top and clean circle edges. Held down, they go flat `#ED3594` at the same size.
- **Logos:** header, login, drawer and splash show the new pink, with no outline. If stale,
  reset the Metro cache.
- **Splash (iOS + Android):** at rest, no dark ring round the mark, even after holding. On the
  burst, only the grey edge shows, with no black band.
- **Android 12+** (native rebuild), with the system in light **and** dark mode: no white or
  black screen, and no disc behind the logo.
- **Android push:** the notification tint is the new pink.
- **iOS cold launch** (delete the app first): the native launch logo is the new pink, with no
  old-magenta flash.

## Not in this handoff
The **app icons** still carry the old magenta. Design will re-export them from the new
stroke-less logo, and they will come as a separate handoff:
- iOS: `ios/nanza/Images.xcassets/AppIcon.appiconset/`.
- Android: `res/mipmap-*/ic_launcher*.png` (incl. `_foreground`, `_round`).
  `ic_launcher_background` stays `#000000`.

## Prompt
```text
Implement the pre-dev handoff
nanza-vault/nanza-figma/handoffs/pre-dev/2026-11-12-brand-rehue-logo-splash.md
(read it in full first; all values are final). reference.patch in its assets folder is the
designer's build of exactly this, against dev ac083896 — use it to check your work.

1. PALETTE: replace colors.primary 50–950 in dark.json AND light.json with §1.
   theme.action -> primary.500; theme.highlight -> primary.400; add theme.link
   (primary.400, both themes) and `link?: string` to ThemeTokens (types.ts, approved).
2. LINKS: tagMention / tagMentionPressed / showMoreText (posts.ts) and inlineTagRun
   (postComposer.ts) read theme.colors.link (fallback action).
3. NAV: floatingNavLabelActive + FloatingTabBar activeColor -> textSecondary || text.
4. LOGOS: logo.svg / logo-simple.svg / logo_full.svg: new fill + gradient, delete the stroke path.
5. ANDROID: notification_accent #ED3594; AppTheme windowSplashScreenBackground and
   windowSplashScreenIconBackgroundColor = @color/app_window_background (tools:targetApi="31").
6. SPLASH: LogoBurst clipProps k = scale.value / artZoom.value; fix the push-in comment.
7. iOS LAUNCH LOGO: copy nanza_logo_{1x,2x,3x}.png from the handoff's assets folder over
   ios/Assets.xcassets/LaunchLogo.imageset/ (same names; Contents.json unchanged).
8. PLUS DISCS: navPlus + fab in JoinChatFooter get a LinearGradient overlay
   primary['400'] at 0.75 -> 0, locations [0, 0.85], top to bottom, under the icon, not rendered
   while pressed (Pressable children render prop); new floatingNavPlusGradient (absolute fill,
   radius.max) in layout.ts. Keep the discs' action backgroundColor.
Do NOT edit cards.ts or badges.ts — they follow the highlight token.

House rules (CLAUDE.md): plan first with confidence scores; theme tokens only. Run /ui-review
before finishing, then set this handoff to built and add a Log line.
```

## Log
- 2026-11-09: handoff written from the `throwaway-logo-update` build. The palette, logos,
  `action` and the nav were device-tested; the ring shade, Android and the splash fix were
  pending.
- 2026-11-12: moved to **pre-dev** and re-verified against dev `ac083896`. Decisions:
  - `action` stays 500.
  - The ring is solved by moving `highlight` to `primary.400`.
  - The stash's backdrop swap is excluded.
- 2026-11-12: added the iOS launch PNGs (#10) and the plus-disc gradient (#11).
  - Gradient tuned on device to 75% → 0 by 85% down, and flat when pressed.
  - `link` moved from pink to `neutral.300` grey.
- 2026-11-12: **ready for dev.** Rebuilt as `throwaway-hue-change` on dev `ac083896`;
  `reference.patch` exported. TypeScript and ESLint are clean on the touched files.
- 2026-11-12: `link` set back to pink, `primary.400` `#F670B9` in both themes (was `neutral.300` grey). Token value only; `reference.patch` updated and re-checked against dev.
