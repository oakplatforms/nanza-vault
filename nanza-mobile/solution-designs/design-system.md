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
  the palette swap risky; that coupling was removed. A backlog of pre-existing raw-scale refs
  still needs converting — tracked in [[semantic-token-migration|Semantic Token Migration]].
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

### Emoji in running Balto text — inline boxes (2026-10-12)

Post text stays on Figma's `Nanza/postBody` 16/19 (detail root 18/22, `profileBio` 16/19).
That leading is tighter than an emoji is tall, and it holds only because emoji are not drawn
in the text run.

- **Why emoji can't sit in the run (iOS):** Apple Color Emoji is ~1.2× the point size tall and
  RN applies `lineHeight` as a clamp, so in a 19pt box an emoji fills the line and gets its
  bottom cut. And at ANY leading, iOS seats each line's baseline off the deepest descender on
  it — the emoji font hangs ~0.37em below the baseline against Balto's 0.19em — so a line
  holding an emoji is lifted ~3pt toward the line above.
- **The fix:** `EmojiText` (`components/global/EmojiText`) is a `<Text>` that lifts each
  emoji into its own inline `<View>`, which the text system treats as an attachment with no
  descent — the line never sees the emoji font, keeps Balto's baseline, and can't clip the
  glyph. A small `translateY` (layout.ts `inlineEmojiBox`) drops the emoji back to where it
  hangs in running text, and because an inline view isn't part of the string VoiceOver reads,
  the component sets the text as its own `accessibilityLabel`.
- **Where:** post and reply text, the profile bio, tag and group descriptions (via
  `ExpandableText`) and the tag share card. Use it for any new running Balto copy that users
  write. Android renders plain text — the problem doesn't exist there.
- **Replaces** the emoji-only twin slots (`postBodyEmoji`, picked by an `isEmojiOnly` scan),
  which only covered messages that were nothing but emoji.
- **Tried and dropped:** loosening the leading to 16/22 (stops the touching, not the lift, and
  leaves Figma); drawing emoji smaller (only shrinks the lift, and the emoji). Not taken: a
  Balto copy with a deeper descender (no font edits), a native RN patch pinning the baseline.

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

## Proportional sizing — layout adapts, tokens don't

The design comes from a **402pt Figma reference frame** with a 20pt gutter per side, giving a
**362pt content width**. Every absolute dimension in that frame (`w-[189px]`, `h-[500px]`) is a
*proportion* of it, not a pixel count. Translating one literally is the bug this section exists
to prevent: it looks right on the reference device and wrong on every other one.

**The split — what scales and what doesn't:**

- **Design-system tokens are exact and never scaled** — with one carve-out, type. `spacing.*`,
  `radius.*` and `icons.*` are the vocabulary that makes the app feel like one product. An 8pt
  gutter is 8pt on a small phone and on a tablet. Scaling them per-device would destroy the ramp
  the rest of this document defines. The typography ramp (`fonts.size` + `fonts.lineHeight`) is
  the exception since 2026-10-13 — see "Type follows the viewport" below.
- **Container geometry is derived from the viewport.** Widths, columns, and hero/mockup sizes
  come from the live window, never from a constant.

### The four mechanisms

1. **Fraction-of-content-width steps** — `components/cards/thumbSizes.ts` is the canonical
   implementation and the pattern to copy. Semantic steps (`circle` `sm` `smd` `md` `lg`) are
   stored as **fractions with the Figma numerator kept visible** — `md: 189 / 362`,
   `sm: 143 / 362`, `circle: 96 / 362` — so the spec's provenance survives in the code and a
   redesign is a one-line edit. `getThumbWidth(size, windowWidth, theme)` resolves a step against
   `windowWidth - spacing.base * 2`. Two steps are tuned rather than derived (`smd: 0.46`,
   `lg: 0.8`); `smd` exists specifically because md's ~52% cuts the second card at the screen
   edge, while 46% leaves a peek of the third so a rail reads as scrollable.
2. **Computed column widths** — `shelf/CardGrid` derives `cardWidth` from `useWindowDimensions()`
   minus outer padding and `columns - 1` gutters. It `Math.floor`s deliberately: a fractional
   width that sums to exactly the available space still wraps once RN rounds up, so the leftover
   sub-pixel is pushed into the gap instead.
3. **`aspectRatio` for height** — height is a *consequence* of width, not an input. Card thumbs
   use `0.71428571` (5/7), plus `189 / 240`, `143 / 200`, `16 / 9`, `3 / 2`, `1` in `cards.ts`.
   Ratios are written as the division, not the decimal, for the same provenance reason.
   Corollary in `cards.ts`: a row of aspect-ratio children must **top-align** — `alignItems:
   'stretch'` lets the parent dictate height and collapses the child's derived height.
4. **Composed tokens for fixed control geometry** — when a control genuinely has one size,
   it is still *expressed in tokens*, not a raw number: `getFloatingNavGeometry` builds the nav
   height as `spacing.xxl - spacing.md` (66) and the bottom offset as `spacing.base +
   spacing.xxs` (24). One place owns the geometry and the bar, the cart pill, and screens'
   bottom padding all read it.

### Rules that make it hold together

- **Parents own width; children never size themselves.** `ItemThumb` takes a required `width`
  prop and is otherwise width-agnostic; `ListingThumb`/`BidThumb`/`BulkThumb`/`TagThumb`/
  `CircleThumb` take a `size` step *or* an explicit `width` for grid layouts. A section can
  therefore force one step across mixed kinds (listings + bids), and a future view-options or
  filter control changes a single number instead of every thumb.
- **`useWindowDimensions()`, not `Dimensions.get()`.** The hook re-renders on rotation, split
  view, and font-scale changes. The four remaining `Dimensions.get` sites are module-scope
  constants in `BottomSheet`, `Modal`, `CustomFlatList` (which carries a `|| 375` fallback), and
  a keyboard calculation in `PostComposer` — legacy, snapshot-at-import, and not the pattern for
  new work.
- **The computed number is the sanctioned inline style.** CLAUDE.md's P0 #2 bans inline style
  objects; a runtime-computed dimension (`{ width }`, `windowHeight * heightFactor`) is the
  explicit exception. Everything static around it still lives in the bucket class — the idiom is
  `style={[styles.thumbCard, { width }]}`.
- **Per-asset config beats a magic multiplier.** The Android promo mockups (`layout.ts`
  `PROMO_CONFIG`) give each asset its own `{ aspectRatio, heightFactor, topGapFactor }` measured
  against window *height*. It replaced a `transform: scale` on a `flex: 1` base, whose height
  varies per device and so rendered the mockup at wildly different sizes. `topGapFactor` may be
  negative to compensate for Android's extra font padding above the text.

### Type follows the viewport (2026-10-13)

The typography ramp is drawn for the 402 frame, and 14/20 that reads right there reads cramped
on a Pro Max and loose on an SE. So `ThemeProvider` scales **`fonts.size` and
`fonts.lineHeight` only** by the device's portrait width over 402, clamped to 0.9–1.1 (SE 375 →
0.93, Pro Max 430 → 1.07, iPad stops at 1.1), rounded to half points. Decisions: width only
(the shorter window side, so rotation doesn't resize type); line height scales with size so
every slot keeps its ratio; spacing, radius and icons stay exact. Mechanics: one resolved theme
object per factor, cached, so `memoizeByTheme`'s identity-keyed buckets rebuild once per factor
and factor 1 is the base theme itself; the factor is private to the provider, so no call site
can reach for a `scale(x)`. The JSON keeps the reference values (what the Figma sync pushes).
Because every slot and every bucket that derives a control height from a line height reads the
ramp through the theme, nothing else changed — the two raw leadings in `typography.ts` (jumbo's
38, the payouts balance's 50) became tokens so they follow too. Not handled: the OS
accessibility text size still multiplies on top (React Native's default); cap it with
`maxFontSizeMultiplier` if the two ever compound badly.

### Key decisions

- **Fractions over a global scale factor.** A single `scale(x)` helper (the common RN approach)
  would drag spacing, radius, and type along with layout — exactly what must stay fixed. Scoping
  proportionality to *container width only* keeps the token ramp intact. Type is the one ramp
  that does scale — by its own clamped factor inside the provider, never through a helper a
  call site can reach (see "Type follows the viewport").
- **Keep the numerator.** `189 / 362` costs nothing at runtime and tells the next reader which
  Figma value it came from; `0.522` tells them nothing.
- **New code only.** ~218 numeric `width:` and ~234 numeric `height:` values remain across the
  style buckets. Many are legitimate fixed control geometry (the 44pt glass circle, icon buttons,
  hairlines); the rest are pre-existing. This is a standard for new and touched code, not a
  completed migration — same policy as the raw-token backlog in
  [[semantic-token-migration|Semantic Token Migration]].

## Floating header — the one chrome for back+title screens (July 2026)

Every screen with a back button and centered title uses the Header's **floating** variant
(the ShelfScreen pattern): a glass dark-circle back button (`chevron-left-3`,
`secondaryText` glyph) and right icons (`variant: 'darkCircle'`), centered
`typography.heading` title capped at 61% width, RadialGlow halos behind buttons and title.
All styles live in `headers.ts` (`headerFloating*`); screens opt in with props only. The
circles' fill and rim come from the shared glass recipe — see **Glass chrome** below.
Right-hand darkCircle glyphs render at `icons.sm` (16); the back chevron is drawn larger
(10×18 explicitly) and is deliberately exempt from that step-down.

Two wirings, one look:

- **Inside PageLayout** — `headerProps={{ floating: true, title, showBackButton: true,
  backIcon: 'chevron-left-3' }}`. PageLayout's animated wrapper adds pin/hide-on-scroll-down/
  reveal-on-scroll-up, measures the header for content padding, and adds a 32pt content gap
  itself — no local spacer needed.
- **Bare Header (screens that own their ScrollView/keyboard handling — create/edit forms,
  scan flows, bulk)** — `<Header floating floatingInFlow title=... showBackButton
  backIcon="chevron-left-3" />`. `floatingInFlow` keeps the same visuals but renders
  in normal flow (no absolute overlay, no hide-on-scroll); chosen over migrating these
  forms into PageLayout to avoid keyboard/scroll regressions. These screens add
  `layoutStyles.floatingContentTop` (the same 32pt) to their first content container.

The content gap and the glow are tuned together: `effects.glow.spread` was tightened
80→56 (the 80 halo washed ~100pt over panels on plain screens) and content starts 32
below the measured band, so only the glow's faint outer fringe can touch content at
rest. The glow-centering offsets in `headers.ts` derive from the spread token — keep
them token-driven if the spread is retuned.

Conventions that rode along: titles are sentence-cased ("Edit profile", "Create offer");
the listing create/edit screen is titled **Sell** and the bid create/edit screen **Buy**
(product identity stays inline in the form body). Deliberately *not* floating-with-title:
detail screens with hero imagery (Listing/Bid/GroupDetail/UserProfile — floating but
title-less), Search/ConversationThread (centerContent), camera capture, and cart review.

**Title-less screens opt out of hide-on-scroll.** PageLayout's `staticHeader` prop pins the
band and keeps it fully opaque. It exists because a header that is *only* always-available
circles has no title bar worth reclaiming space for — there the hide/reveal swing reads as
the circles drifting and flashing as you scroll. `SupportedTagValueScreen` uses it; the
other title-less floating screens (Listing/Bid/GroupDetail/UserProfile) are candidates and
have not been converted yet. Apply it to **every** PageLayout in a file (loading *and*
content states) or the header jumps when data lands.

## Glass chrome — the translucent surface language (Aug 2026)

One recipe in `styles/components/glass.ts` backs every translucent surface. Before it, the
same two constants were re-derived in three buckets (`postComposer`, `searchFooter`,
`collectionSearch`) with a fourth opaque variant in `headers.ts` — four edits per change and
guaranteed drift.

- **Tokens.** `glass` (fill) and `glassBorder` (rim) are semantic aliases over
  `system.glass` / `system.glassBorder`, so alphas are retunable in JSON. `spacing.page`
  (20) is the page-gutter alias — deliberately distinct from `base` (also 20) so gutters can
  move without dragging every other 20.
- **`glassSurface`** — fill + rim, for callers owning their own geometry.
  **`glassCircle`** — the `spacing.control` (44pt) floating control, fill only.
  **`glassPill`** — the full-width bar (chat entry, search footer, collection search):
  `spacing.control` tall, `radius.24`, `spacing.page` padding. Fill only in the bucket; the
  rim-light is a `<GlassRim pill />` child at each of the three call sites. Its `radius.24`
  clamps to half the 44 height in both RN and `GlassRim`, so the bar is an exact stadium.

### One control size, one control recipe (Aug 2026)

Every floating control — circles, pills, bars, tabs — is **44pt**, held by
**`spacing.control`**. Figma resized the glass circles 50 → 44 and the decision was taken
app-wide, sweeping the strays at 48 and 52 with it.

- **`control` is deliberately a separate token from `hit`, though both are 44.** `hit` names
  the *touch target*, `control` the *drawn size*. Step 4 below makes them differ in practice,
  and collapsing them would stop the two diverging later. Same rationale as `page` vs `base`
  (both 20) above.
- **The size lives in one place; so does the recipe — with no fill forks.** `glassCircle` owns
  control-box geometry (size, `radius.max`, centring, fill) and `glassPill` owns the bar. All
  eight control circles are now a bare `...glass.glassCircle` spread; the only properties added
  after it anywhere are `marginTop: -spacing.min` optical nudges on two header circles. Before
  this, six circles duplicated the box, so anything but a resize meant a 12-site edit.
  - **The fill forks were audited away rather than preserved.** Three of the four "different
    fill" circles turned out not to justify themselves: `searchHeaderBackCircle` (inputs) and
    `searchFooter closeCircle` were **dead code** — declared and typed but never rendered —
    and `headerSearchIconCircle` painted an opaque `background` fill, which resolves to the
    *same colour as the surface behind it* (`#010101` dark / `#FFFFFF` light), so the circle
    read as nothing. It now takes the glass fill like every other control. Lesson: an override
    that has drifted into invisibility looks identical to an intentional one — resolve the
    token to a colour before assuming a fork is deliberate.
  - The one deliberate non-adopter is `headerFloatingButtonWrap`: it is a bare positioning
    wrapper, not a filled circle (the button inside it is the `glassCircle`, and a `RadialGlow`
    is its sibling). It takes the size token only — spreading the recipe would paint a second
    61%-alpha fill directly behind the real one, stacking into a visibly darker blob. It needs
    the size because the glow is absolutely positioned against it.
- **Touch expansion is `hitSlop`, not layout.** `getControlHitSlop(theme)` (exported from
  `glass.ts`) returns `spacing.min` on all four sides — a 48×48 target around a 44pt control,
  expanding the tap area **without changing anything the user sees**, and staying under the
  tightest neighbour gap (the composer's icon row at `spacing.xs` = 6) so it can't steal a
  neighbour's taps. `hitSlop` is a **prop, not a style**, so it can never ride inside a
  `...glassCircle` spread — the value is shared, the call sites pass it.
  - Bars, pills and full-width tabs get none: already ≥44 on both axes.
  - The ~37 other `hitSlop` sites keep their own larger `sm`/`xs`/`md` values — those are
    tuned per affordance (text links, chips, close glyphs) and are not part of this rule.
- **44 already satisfies the iOS HIG and WCAG 2.5.5 unaided.** The slop is headroom, not a fix.

### The bars carry the rim too (Aug 2026)

The bars originally shipped **rim-less**, because a flat 16%-alpha `borderWidth: 1` read as a
hard edge rather than a lit one at that width. That verdict was about the **border**, not the
rim: the gradient stroke fades out along the sides, which is precisely what a uniform border
cannot do — the same reason the circles moved to `GlassRim` in the first place. So the rim was
extended to all three bars at once, keeping one recipe one look rather than letting the chat
entry drift from search.

`GlassRim` grew a `pill` shape to do it, and the viewBox trick that makes the circle
resolution-independent **does not carry over**: scaling a 100×100 viewBox non-uniformly
(`preserveAspectRatio="none"`) stretches both the corner radius and the stroke horizontally, so
a 24pt corner comes out a flat-topped smear. The pill therefore draws in real points and
measures itself with `onLayout` — one extra frame on mount, invisible on static chrome. Letting
the SVG canvas clip half a centred stroke (which would need no measurement) was rejected:
out-of-bounds clipping differs between iOS and Android, the same class of platform trap that
sank the per-side border. Both shapes still read one falloff curve from `effects.glassRim.stops`.

### No backdrop blur — and the fill is a prerequisite, not a substitute

RN has no `backdrop-filter`. Figma sets *both* a translucent fill and a blur on the same
node: the blur softens what's behind, the tint gives colour identity and a contrast floor
for text. Shipping the fill alone is ~90% of the look and costs no native dependency; if
blur is added later it wraps these classes unchanged.

### No native shadow or elevation on the circles

Removed deliberately, and this is the trap worth remembering: `shadowColor`/`shadowOffset`/
`shadowOpacity`/`shadowRadius` are **iOS-only** while Android honours only `elevation` — so
the two platforms had silently diverged. Worse, Android clips an elevation shadow to the
view outline *only when the background is opaque*; against a translucent fill the shadow
composites straight through the control. RadialGlow (react-native-svg, identical on both
platforms) already does the darkening these controls need. Verified on device: removing
`elevation` did not break Android z-order or tap reception — the parent's `zIndex` carries it.

### The rim-light is a gradient stroke, not a border

Figma draws the circle's rim on the top and bottom edges only. Two border approaches were
tried on device and both rejected:

- `borderTopWidth`/`borderBottomWidth` alone → renders **nothing**. On a shape whose whole
  perimeter is curve there is no straight top/bottom edge to occupy, so the rim silently
  collapses to invisible. No artifact, no warning.
- `borderWidth: 1` + transparent side colours → renders, but RN paints a rounded border as
  four discrete arcs, so it cuts hard at the 45° diagonals. Visibly sharp at 50pt.

**No border-based approach can fade.** A drop shadow can't either — shadows render outside
the shape, giving a halo rather than a lit edge. So the rim is `components/global/GlassRim`:
a `react-native-svg` circle with a vertical gradient stroke, same approach as RadialGlow.
It mounts as the first child of any glass circle — absolute, `pointerEvents="none"`, so call
sites keep their own style and press handling — and is sized from a `100×100` viewBox so one
component serves every diameter (its `pill` shape is the exception, above). The falloff lives
in `effects.glassRim.stops`, retunable in JSON alongside `effects.glow.stops`.

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
