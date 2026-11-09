---
tags: [nanza-figma, handoff, nanza-mobile, brand, palette, logo, splash, android]
status: open
date: 2026-11-09
repo: nanza-mobile
scope: tokens + assets + splash fix
---

# Brand re-hue — persian-rose palette, new logo colours, splash edge fix

The brand moves from magenta (hue ~300°, `#FF4BFF`) to **persian-rose** (hue ~329°,
`#ED3594`). Built on a throwaway branch (`throwaway-logo-update`) from a stashed logo update plus
device iteration. The values below are that build, so treat it as the reference implementation.
**Status per change is in the table below: not everything is device-confirmed yet.**

> [!warning] App-wide — read first
> - **One palette swap recolours the whole app.** Every pink in the app derives from
>   `colors.primary` (136 direct refs, mostly `primary['500']`) plus three semantic tokens
>   (`highlight`, `action`, new `link`). No component code hard-codes the old magenta.
> - **Touches shared infra (approved):** one optional key, `link?: string`, in
>   `src/styles/utils/types.ts`.
> - **Native files:** two Android resources change (notification tint, Android 12+ splash).
> - **Raster assets are NOT in this handoff:** the iOS launch logo, the app icons and the Android
>   launcher icons are still old-magenta PNGs. See "Outside the codebase" below.

## Request (verbatim, condensed)
1. "i have a stash for logo updates but its updating our brand color for our logo and then
   respective colors in the palette and throughout the app". The palette was supplied as
   `persian-rose` 50–950.
2. Scope: "logos + palette + android", plus "make it so the android never has a moment where
   theres a white background and a black circle w pink logo inside it or black background with
   white circle and pink logo".
3. Iteration on device: "a darker pink for our actions … like #de247b for links in posts and the
   plus button", then "try the 500 color for the text links", then "try dropping the link font one
   shade … 500 to 400" (kept), and "drop the action color one to 500"; circle rings "the same action pink
   as the plus button" → "nooo, lets try the 300 shade" → "try 400"; "bottom nav active state …
   a bright white instead of pink".
4. Splash: remove the logo's stroke ("theres a stroke on the logo thats causing problems on the
   splash screen"), and "theres a black stroke/shadow around/outside our gray stroke … the gray
   stroke should be the edge, nothing else".

## Changes and status

| # | Change | Status |
|---|---|---|
| 1 | `primary` scale → persian-rose (dark + light) | device-tested |
| 2 | Logo SVGs: new fill + gradient (from the stash), **stroke removed** | device-tested |
| 3 | `action` → `primary.500` (+ button, send, step dots, request dot…) | 600 device-tested, **500 now on trial** |
| 4 | New `link` token → `primary.400` (post #tags / mentions, Show more, composer tag runs) | **decided: 400** (500 tried first) |
| 5 | Bottom nav active tab → `textSecondary` (bright white) | device-tested |
| 6 | Circle-thumb brand ring + play badge → `primary.400` | **trial, decision pending** (see Open questions) |
| 7 | Android `notification_accent` → `#ED3594` | built, needs an Android push to verify |
| 8 | Android 12+ system splash → app background, no icon disc | built, needs an Android 12+ native build |
| 9 | Splash burst: black band outside the grey edge (clip/zoom bug) | built, **verify on device** |

### 1. Palette (`src/styles/tokens/dark.json` + `light.json`, `colors.primary`)

| | 50 | 100 | 200 | 300 | 400 | **500** | 600 | 700 | 800 | 900 | 950 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Old | FFF3FE | FEE6FF | FFCCFF | FFA3FD | FF6DFA | FF4BFF | E316DE | BD0EB6 | 9A0E93 | 7E1176 | 55004F |
| New | FDF2F8 | FCE7F3 | FCCEE9 | FAA7D6 | F670B9 | **ED3594** | DE247B | C11560 | 9F1550 | 851645 | 510625 |

### 2. Logos (`assets/images/logo.svg`, `logo-simple.svg`, `logo_full.svg`)
- From the stash: base fill `#FF4BFF` → **`#ED3594`**; gradient `#FF6DFA → #E415DE` →
  **`#FF89C6 → #ED3594`** (the 0.6-opacity overlay path).
- **Stroke removed:** each file had a third `<path>` redrawing the mark's outline in
  `stroke="#FF6DFA"` (0.86 / 0.8 / 0.15). Deleted outright. No old magenta remains in the SVGs.
  Consumers: JS splash (`LogoBurst`), `Header`, `LoginModal`, `DrawerModal`.
- The stash's `ios/Podfile.lock` / `project.pbxproj` changes are **not** part of this.

### 3–5. Semantic tokens and the nav

| Token (`theme` section) | Old | New | Used by |
|---|---|---|---|
| `highlight` | `primary.300` | unchanged | "Buy" intent tags |
| `action` | `primary.400` | **`primary.500`** `#ED3594` (600 `#DE247B` tried first) | `floatingNavFab` / `floatingNavPlus` (+ button), `sendIconButtonActive`, `audienceMenuRowTextSelected`, `stepIndicatorDotCompleted`, `requestedUnreadDot` |
| **`link`** (new) | — | **`primary.400`** `#F670B9` (500 `#ED3594` tried first) | `tagMention` (+ pressed at 0.7), `showMoreText`, composer `inlineTagRun`; each falls back to `action` |

- `src/styles/utils/types.ts`: add `link?: string` beside `action?`. The resolver
  (`resolveThemeColors`) maps every `theme` key generically, so nothing else is needed.
- **Bottom nav active state:** `layout.ts floatingNavLabelActive` colour and `FloatingTabBar`
  `activeColor` → `theme.colors.textSecondary || theme.colors.text` (`#F8F6F4`). It was `action`.
  Inactive stays `secondaryText`.

### 6. Circle-thumb ring + play badge (`cards.ts`)
`circleThumbRingBrand.borderColor` and `circleThumbPlayBadge.backgroundColor` →
`theme.colors.primary?.['400'] || theme.colors.highlight || theme.colors.border` (`#F670B9`).
Tried on device: 600 (rejected), 300 (rejected), 400 (current). **Scale ref on purpose for the
trial.** If 400 sticks, add a semantic token (e.g. `brandRing: "primary.400"`, the same way as
`link`) and point both classes at it. The badge must always match the ring.

### 7–8. Android (`android/app/src/main/res/values/`)
- `colors.xml`: `notification_accent` `#FF4BFF` → **`#ED3594`** (the push icon / app-name tint
  via `firebase.json`; mirrors `primary.500`).
- `styles.xml` `AppTheme`: Android 12+ draws its own launch splash (app icon in a circle on a
  system background) before the app's theme applies. Unset, the background follows light/dark
  mode and the circle takes the launcher icon's `#000000` background: a black disc on white, or
  the reverse. Pin both to the app background:
```xml
<resources xmlns:tools="http://schemas.android.com/tools">
  …
  <item name="android:windowSplashScreenBackground" tools:targetApi="31">@color/app_window_background</item>
  <item name="android:windowSplashScreenIconBackgroundColor" tools:targetApi="31">@color/app_window_background</item>
```
  Ignored below API 31. No Play Console setting is involved.

### 9. Splash burst edge (`src/components/global/SplashAnimation/LogoBurst.tsx`)
**Symptom:** a black band outside the grey border during the burst, and a dark ring round the mark
at rest (thicker the longer the splash holds).

**Cause:** the artwork's logo-shaped clip (`ClipPath`, `userSpaceOnUse`) resolves in the image's
own user space, which **includes the image's `matrix`**: the slow push-in `artZoom`, 1 → 1.12 over
10s. So the artwork's hole was up to 12% larger than the underlay's hole and the border, and the
dark underlay showed between them.

**Fix:** divide the push-in back out of the clip's matrix. Everything pivots on `PIVOT`, so it
composes back to exactly the burst scale:
```ts
const clipProps = useAnimatedProps<AnimatedPathProps>(() => {
  const k = scale.value / artZoom.value
  return { matrix: [k, 0, 0, k, PIVOT * (1 - k), PIVOT * (1 - k)] }
})
```
The old comment claiming the image's zoom "leaves the hole untouched" is corrected.

## Code map (local `throwaway-logo-update`; match these on your branch)
- `src/styles/tokens/dark.json`, `light.json`: `colors.primary`; `theme.action` → `primary.600`; new `theme.link`.
- `src/styles/utils/types.ts`: `link?: string` (approved shared-infra edit).
- `src/styles/components/posts.ts`: `tagMention`, `tagMentionPressed`, `showMoreText` → `link`.
- `src/styles/components/postComposer.ts`: `inlineTagRun` → `link`.
- `src/styles/components/layout.ts`: `floatingNavLabelActive` → `textSecondary`.
- `src/components/global/FloatingTabBar/index.tsx`: `activeColor` → `textSecondary`.
- `src/styles/components/cards.ts`: `circleThumbRingBrand`, `circleThumbPlayBadge` → `primary['400']` (trial).
- `src/components/global/SplashAnimation/LogoBurst.tsx`: `clipProps` divides by `artZoom`.
- `assets/images/logo.svg`, `logo-simple.svg`, `logo_full.svg`: colours + stroke path removed.
- `android/app/src/main/res/values/colors.xml`, `styles.xml`.

## QA
- **Palette:** + button and send icon `#ED3594`; post #tags, mentions and "Show more" `#F670B9`;
  "Buy" tags `#FAA7D6`; no old magenta anywhere (check rings, badges, step dots, request dot,
  audience menu).
- **Nav:** active tab label + icon bright white; inactive unchanged.
- **Logos:** header, login, drawer, splash: new pink, no outline. Reset the Metro cache if stale.
- **Splash (iOS + Android):** at rest, the pink mark with no dark ring, even after holding a few
  seconds. On the burst, only the grey edge, with no black band outside it.
- **Android 12+** (native rebuild), light **and** dark system mode: no white or black screen, no
  disc behind the logo.
- **Android push:** notification tint is the new pink.
- **Known mismatch until the PNGs are replaced:** the iOS native launch logo shows the old magenta
  for a moment before the JS splash takes over.

## Open questions
1. **Circle ring shade:** keep `primary.400`? If yes, add the semantic token (see §6).
2. **Action shade:** keep `action` on `primary.500` (the current trial), or back to 600?

## Outside the codebase (not in this handoff's code)
Raster and store assets still carry the old magenta; they need re-exporting from the new
stroke-less logo. Tracked here so they aren't lost; the list is in the session notes / next
handoff.

## Prompt
```text
Implement the handoff nanza-vault/nanza-figma/handoffs/2026-11-09-brand-rehue-logo-splash.md
(read it in full, including Status and Open questions).

1. PALETTE: replace colors.primary 50–950 in dark.json AND light.json with the persian-rose
   values in §1; theme.action -> primary.500; add theme.link = primary.400 and `link?: string`
   to ThemeTokens (types.ts, approved).
2. LINKS: tagMention / tagMentionPressed / showMoreText (posts.ts) and inlineTagRun
   (postComposer.ts) read theme.colors.link (fallback action).
3. NAV: floatingNavLabelActive + FloatingTabBar activeColor -> textSecondary.
4. RINGS: circleThumbRingBrand + circleThumbPlayBadge per §6 (check the open question first).
5. LOGOS: logo.svg / logo-simple.svg / logo_full.svg: new fill + gradient, delete the stroke path.
6. ANDROID: notification_accent #ED3594; AppTheme windowSplashScreenBackground and
   windowSplashScreenIconBackgroundColor = @color/app_window_background (tools:targetApi="31").
7. SPLASH: LogoBurst clipProps scale = scale.value / artZoom.value.

House rules (CLAUDE.md): plan first with confidence scores; theme tokens only. Run /ui-review
before finishing, then set this handoff to built and add a Log line.
```

## Log
- 2026-11-09: handoff written. Palette, logos, action / link tokens and the nav are device-tested on `throwaway-logo-update`; the ring shade is a trial; the Android notification tint, the Android 12 splash and the splash edge fix are built and awaiting device checks
- 2026-11-09: `link` moved 500 → 400 on trial (device check pending)
- 2026-11-09: `link` kept on 400; `action` moved 600 → 500 on trial
