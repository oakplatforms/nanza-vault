---
tags: [nanza-mobile, solution-design, design-system]
---

# Design System — Colour Tokens, Typography & Theming — Solution Design

## Overview

How colour **and type** work in nanza-mobile: a **two-layer token model** (raw palette scales → semantic
aliases) resolved at runtime into `ThemeTokens` and served through `ThemeProvider`. This design
captures the *model and the decisions*, not the exact values — the token files
(`src/styles/tokens/dark.json`, `src/styles/tokens/light.json`) are the **living source of
truth** for every colour. When you add, rename, or merge a token, the JSON moves; this doc only
changes when the *architecture or conventions* change.

The system was aligned to the Figma design tokens in a staged migration (mid-2026). The durable
outcome: **components reference semantic tokens only** — never raw palette scales.

## The token model (two layers)

1. **Base scales** (`colors.*` in the token JSON) — mode-agnostic palettes: `primary`,
   `secondary`, `neutral`, `tertiary`, plus `system` (overlays, status) and `conditionColors`.
   These mirror the Figma `Mode 1` palette (Primary/Secondary/Neutral + Condition). `neutral` is
   a clean gray ramp; `secondary` is the blue-gray ramp; `primary` is the pink/magenta brand ramp.
2. **Semantic aliases** (`theme.*` in the token JSON) — role-named tokens that *point at* base
   scales via `"scale.shade"` strings (e.g. `"surface": "secondary.950"`). This is the only layer
   components are allowed to touch.

### Resolution path

`ThemeProvider` (`src/styles/utils/ThemeProvider.tsx`) imports a token JSON →
`resolveThemeColors` (`src/styles/utils/resolveThemeColors.ts`) turns each `"scale.shade"`
reference into a concrete hex and re-attaches the raw scales → `parseNumericStrings` coerces
numeric strings → the result is typed as `ThemeTokens` (`src/styles/utils/types.ts`) and handed
to components via `useTheme()`. Style buckets (`src/styles/components/*.ts`) consume
`theme.colors.<semantic>` inside `getXStyles(theme)`; components never inline colours.

Because semantic tokens are the indirection, **the same base value can serve both modes**: a
semantic token flips by pointing at a different shade per mode file, while components stay
unchanged.

## Semantic vocabulary (roles, not colours)

The Figma-defined semantic set: `background`, `surface`, `text`, `overlayText`, `ink`, `border`,
`secondaryText`, `inactive`, `highlight`, `action` (+ condition colours). The app keeps a
**superset** beyond Figma for existing needs (`onSurface`, `card/modalBackground`, `input*`,
`textMuted/Subtle/Disabled`, `overlay*`, status colours, `shadow`). See the JSON for the full
list and each mode's mapping.

Two tokens worth calling out because their *names* don't fully describe their use:

- **`ink`** — a **theme-invariant near-black** (`neutral.950`, identical in light and dark). It is
  *not* "text": it is the darkest ink used both as **foreground** (dark labels/icons on bright
  elements) and as **fill** (buttons, button groups, image wells). Named for the colour's
  identity, not one role.
- **`surface`** — the single elevated-container colour. `well` used to be a second, redundant
  inset token; it was **deprecated and removed** — all former `well` usages are now `surface`.

## Migration conventions (how raw usages were mapped)

When eliminating raw `neutral[...]` references, usages were classified by **role**, and these
conventions are the standing rules for new work:

- **Image containers → `ink`** — anything backing an image or its placeholder: image/product
  wells, avatars, banners, collection/mosaic image cells, profile thumbnails. Near-black backdrop
  so images and placeholders read as deep. (Explicitly confirmed as an app-wide convention.)
- **UI surfaces/cards/boxes/bars/sheets → `surface`**; deepest app-level backgrounds → `background`.
- **Borders/dividers → `border`**; muted control outlines (radio/toggle unchecked) → `inactive`.
- **White / near-white text & icon fills → `text`**; muted body text → `secondaryText`;
  dark-on-bright text → `ink`.
- **Shadows → `shadow`** (was raw `neutral.950` as `shadowColor`).
- **Fallback chains name the target** — a legacy `neutral?.['X'] || theme.colors.Y` was migrated
  by dropping the raw primary and keeping `Y` (rewriting a `|| well` tail to `surface`).

## Key decisions (why it's shaped this way)

- **Semantic-only in components.** Raw palette scales (`neutral`, `secondary`, …) are an
  implementation detail of the semantic layer. Components referencing them directly is what made
  the palette swap risky; that coupling was removed.
- **Swap-first, then clean up.** The Figma palette (neutral ramp + `background`/`surface`/`border`
  repoint) landed as one holistic commit, *then* components were migrated onto semantic tokens.
  The alternative (migrate-first) would have touched each grey surface twice.
- **`ink` over a redundant `well`.** Rather than keep two identical inset tokens, `well` folded
  into `surface`; genuine near-black image wells use `ink`.
- **JSON is the source of truth.** Design docs describe the *system*; exact tokens/values live in
  the token files so streamlining the palette doesn't rot the docs.

## Typography — the enforced scale (July 2026)

All text styles flow from **one typographic scale** in `src/styles/components/typography.ts`.
`getTypographyStyles(theme)` defines ~11 **slots**, each bundling family (encodes weight) +
size + lineHeight + tracking + a default colour:

`jumbo` `large` `heading` (SemiBold 32/24/20, −0.11 tracking) · `title` `subtitle` `bold`
`label` (SemiBold 16/16/14/14, +0.11) · `caption` (Medium 13.5) · `meta` (Regular 13.5) ·
`micro` (SemiBold 10 — tab labels, tiny badges) · `body` (Regular 14/18) — plus a couple of
glyph one-offs (`closeGlyph`). Exact values live in the file; slot *names* are the stable API.
The ramp follows the July 2026 Figma redesign: **Bold face retired** (SemiBold everywhere),
systematic tracking (+0.11 below 20pt, −0.11 at 20pt+).

### Composition pattern

Style buckets spread a slot, then override colour/spacing **after** the spread:

```ts
timestamp: { ...typography.meta, marginBottom: theme.spacing.xxs }   // colour inherited
rowName:   { ...typography.label, color: theme.colors.textMuted }    // colour overridden
```

Every bucket getter opens with `const typography = getTypographyStyles(theme)`. Inside
typography.ts itself, legacy domain one-offs (~150 names, all still consumed) compose
`...scale.<slot>` — so a slot change propagates to them too.

### Enforcement

- **ESLint** (`no-restricted-syntax` in `eslint.config.mjs`): writing `fontFamily`,
  `fontWeight`, `fontSize`, `lineHeight`, or `letterSpacing` anywhere except typography.ts is
  an error. CLAUDE.md carries the matching P0 rule (#12).
- **Only exception:** truly animated values (the floating-label `interpolate`d
  fontSize/lineHeight in the Form fields) with explicit `eslint-disable-next-line`.
- **Gotcha worth remembering:** react-navigation applies `tabBarLabelStyle` *after* its tint
  colours, so the tab-label style spreads micro's font metrics **minus** `color`
  (see `bottomTabBarLabel` in layout.ts).

### Design ↔ dev sync (typography, extending to all tokens)

The scale is mirrored into Figma by `scripts/figma-token-sync.js` (run inside the Nanza
Figma file via Scripter or a dev "run once" plugin). It creates/updates text styles named
`Nanza/<slot>` and rebuilds a **"Nanza Typography Sheet"** specimen frame. Re-running is
idempotent. The plan is to extend the same script with radius/spacing/colour sheets.

**Direction discipline: code is canonical.** Figma edits are *proposals*, not truth.

- **Code → Figma:** change `typography.ts` (or `dark.json`), regenerate the script's tables
  if needed, re-run the script in Figma. Styles and sheet update in place.
- **Figma → code:** a designer edits the `Nanza/<slot>` styles (the specimen sheet reflects
  them automatically since its rows reference the styles). Then pull via the Figma Dev Mode
  MCP: read the sheet's node, diff each slot against `typography.ts`, apply, commit, and
  re-run the sync script to normalize. Fully automatic write-back is not available
  (Figma's variables write API is Enterprise-only; the plugin sandbox can't push to git),
  and the review step is desirable — slot changes are app-wide by definition.

### Key decisions

- **Slots own colour defaults.** Saves a colour override at most call sites; the cost is that
  layered "modifier" styles must re-override colour after their spread (profile badges, tag
  text variants do this deliberately).
- **Sizes were standardised, not preserved.** The migration snapped ad-hoc sizes
  (`Number(token)+0.5`, `sm+1`, raw `14.5`) to the nearest slot; per-site letterSpacing
  tracking was dropped in favour of slot-level tracking.
- **Roles, not sizes, are the API** — changing what `heading` means is a one-file edit that
  reaches all ~900 migrated sites (proven by the Bold→SemiBold ramp swap landing as a
  two-file commit).

## In-page tabs — the segmented capsule is the standard (2026-10-25)

Two tab vocabularies, deliberately:

- **Inside a page** (section switchers: the entity screen's Details · Market · Trades · Chat,
  the inbox's All · Offers · Friends · Groups · Refunds, Group members' Active · Requests, and
  eventually the profile) → **`SegmentedTabs`** (`src/components/global/SegmentedTabs/`), styled
  by `tabs.ts` `segmentedTabs*`. One 44pt capsule — `background` fill, `glassBorder` hairline,
  3 inner padding — holding equal-width flex-1 segments; the active one sits on the `glassActive`
  fill in `textSecondary`, the rest in `secondaryText`, labels in the `pill` slot. An optional
  `count` renders as a 20pt ink circle beside the label (the inbox's All). **Four segments
  share the track; more than four and the track scrolls horizontally inside the capsule** at
  the four-up width with a sliver of the fifth peeking (Skylar, 2026-10-25: five across read
  too small) — the active segment is scrolled into view when it changes. Figma
  `proposed.rangeTrack`; first built for the entity screen (2026-10-24, as `EntityTabs` in
  `market.ts`), promoted to the shared component when Messages adopted it (Skylar, 2026-10-25:
  "we want to make sure this is the standard now").
- **Search screens** keep the **bigger self-sizing outline pills** (`tabs.ts` `textTabItemSm`,
  the `TabPillsRow` strip) — those are filters over a result set, not sections of one page.

The component is pure: the host owns placement (the entity screen's `entityTabsInline` in
`market.ts`, the inbox's `tabsAndSearchSection` in `messages.ts`, both at the 20 `base` inset so
the capsule is one width everywhere). **Not sticky** anywhere for now (Skylar, 2026-10-25: too
many elements; maybe later — see [[entity-screen]]). The Trade screen's tabs are the one
in-page strip still on the old outline pills; they are slated for their own pass. The Messages
strip's scrolling outline pills (`messages.ts` `tab` / `tabActive` / `tabCountBadge` …) were
deleted with the switch rather than left as a second pattern.

## Haptics (August 2026)

Touch feedback is a **thin wrapper over `react-native-haptic-feedback`** in
`src/utils/haptics.ts`, exposing one intent-named function per cue the app actually uses —
today just `hapticSelection`. Call sites never import the library or its type strings directly,
so the haptic vocabulary (and the library behind it) is a one-file change. Add siblings (impact,
success/error) when a call site needs one, not speculatively.

### Where it fires

**The bottom nav tabs only** — Home, Groups, Trade, Inbox — as a `selection` tick on
**press-in**, in `FloatingTabBar/TabItem.tsx`. Nothing else in the app has haptics yet.

Three decisions worth keeping:

- **`selection`, not `impactLight`.** iOS maps it to `UISelectionFeedbackGenerator`, the tick a
  segmented control gives — semantically "moved between discrete options", which is what a tab
  switch is. An impact reads as "hit something".
- **Press-in, not press.** The tick lands with the finger, alongside the existing scale-down
  `usePressScale` drives. On `onPress` it arrives after the gesture and the `tabPress` event
  resolve, and reads as lag.
- **Fires on the already-active tab too**, where the navigate no-ops. The user pressed a control
  and should feel that it took.

### The shared-hook trap

`usePressScale` (exported from `TabItem.tsx`) is shared by the tabs **and the Search circle** in
`FloatingTabBar/index.tsx`. Putting the haptic in the hook is the obvious move and it is wrong —
it silently gives Search a tick, and haptics are scoped to the tabs (Skylar, 2026-08-23). The
call therefore sits in `TabItem`'s own press handler, which wraps the hook's. There is a comment
on the hook saying so; keep it if the hook is refactored.

### Options policy

Both defaults in `haptics.ts` are deliberate: `enableVibrateFallback: false` (on a device with no
taptic engine the fallback is a blunt motor buzz that reads as an error, so silence is the better
failure) and `ignoreAndroidSystemSettings: false` (a user who disabled haptics in the OS meant
it). Every trigger is wrapped in a `try/catch` — the haptic is decoration and must never cost the
tap.

**Native dependency.** The package ships a TurboModule `codegenConfig`, so it is New-Architecture
safe (the app runs `newArchEnabled=true`). Adding or removing it requires `pod install` and a
full rebuild, not just a Metro reload. Haptics do not fire in the iOS Simulator — verify on device.

## Light / dark theme switching — PENDING (second pass)

`light.json` exists with the full Figma Light mapping and correct base scales, but is **not yet
wired**: `ThemeProvider` still hard-imports `dark.json`, and its `themeMode` state is inert.
Wiring mode selection (and the resolver path for two live modes) is the next architectural step —
**this section, and `architecture.md`, get updated when that lands.** Until then the app is
dark-only and `light.json` is inert-but-ready.


## Button's loading arc — the asset that never drew (fixed 2026-09-11)

`Button`'s `loading` prop keeps the label and spins a Lottie arc at the trailing edge, tinted
to the label color via `colorFilters` on the single layer "Shape Layer 1". It had never been
visible anywhere in the app: the source was `assets/icons/loading-2.lottie`, a **dotLottie v2**
bundle (animation at `a/Main Scene.json`), and the lottie-ios build behind lottie-react-native
7.2 reads the v1 layout, fails silently and draws nothing. The success check is a plain `Icon`,
which is why a save showed a checkmark at the end but no spinner during — Skylar's repro on the
seller-onboarding Submit. `SocialAuthButtons` had already worked around it by pointing at
`assets/lottie/loading-2.json` (the same animation extracted); `Button` now uses that JSON too.
Rule: **Lottie sources are plain `.json` in `assets/lottie/`**, never a `.lottie` bundle.

## DetailActionBar — the fade band passes taps through (2026-09-11)

The bar's gradient band (transparent → solid over `fadeBand`) used to be a touch target at
all times: it declined the responder when the keyboard was closed, but a hit-tested view that
declines still swallows the touch, so a field scrolled under the nearly transparent top of the
band — the listing form's "Add photo", its last field — looked tappable and wasn't. The band is
now `pointerEvents="box-none"` while the keyboard is closed (taps fall through to the content;
the CTA, a child, keeps its own) and `auto` while it's open, where a tap on the band still
dismisses the keyboard in one go. Screens still reserve `useActionBarPadding()` so the last
field can scroll fully clear; this only fixes the case where it hasn't yet.
