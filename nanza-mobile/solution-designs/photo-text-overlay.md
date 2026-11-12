---
tags: [nanza-mobile, solution-design, plan, posts, composer, images]
---

# Photo text overlay — Stories-style captions burned into post images

**Status:** **plan, 2026-11-07 (Skylar, voice + four Facebook Stories screenshots); built the
same day and reworked across ~15 live device rounds that evening (uncommitted, tsc + lint clean,
`react-native-view-shot` 6.2.1 installed + `pod install` run).** Mobile only — the text is
rasterised into the JPEG before upload, so nanza-api, the Post model and the upload contract do
not change. When it lands, fold the durable part into
[[../../nanza-api/solution-designs/photo-galleries|Photo galleries]] (the composer's image step)
and delete this plan.

### Where the device rounds landed (supersedes the design below where they differ)

- **Editing model (Instagram's):** *Aa* at rest starts a centered block; while a block has the
  keyboard, a tap on the photo starts the next block **under the finger** (`origin` on the layer),
  the header swaps the counter + Save for one glass **Done**, *Aa* hides, and the keyboard stays up
  through every tap until Done — the old input lingers one frame so focus hands across without the
  keys dipping. The toolbar (plus a strip above it) swallows stray taps so clicking between colours
  never starts a block. Backspace on an empty block removes it; a fresh block is just a caret (no
  placeholder, no slab until text).
- **Home while editing:** editing springs the block to its origin (pulled up to ≤ 30 % of the canvas
  if it was placed low), upright; the block **grows upward** as lines are added (bottom edge pinned
  above the keys); on a new block's first commit the lifted spot becomes its placement, an existing
  block springs back to where it was. A 60 % scrim veils the photo while editing (never captured).
- **Slab:** ONE rectangle, square corners, wide side padding (`lg` / `xs`), identical while typing and
  committed. The per-line Stories slabs from `onTextLayout` were cut (lagged a frame → flashed), as was
  a Reanimated layout transition on the box (flashed the glyphs). The input runs at the block's full
  width so its frame never changes while typing; the slab wraps an invisible measurement twin.
- **Type:** 32 / 38 (`xxl` on the `'38'` leading), two presets **Bold** (Roobert Bold — the heaviest
  linked cut) and **Normal** (Roobert Regular); Light was tried and dropped. Default pair **white on
  black**, then black on white, then the accent-on-cream pairs.
- **Size slider** (right edge, `sm` from it, centred at 36 % of the stage, shown only while
  editing): an SVG wedge track (skinny at the bottom, `md` wide at the top) with an `mlg` white
  knob; drives a shared travel value the editing block maps onto scale 0.4–2.5. Pinch still works.
- **Chrome:** Save is the composer's own white `postPill`; the bar floats over the stage at the
  app's sheet float and rides the measured keyboard (`usePickerKeyboardPad`, both platforms — the
  composer Modal freezes the Android window); a gradient scrim behind it was tried and removed. The
  photo sits `xxl` under the header (flush-top read too high, centred too low).

## What Skylar asked for

Players at a meetup or event photograph their opponent and post it with a line of text on the
picture — "Round 1 vs Gabriel from Québec, playing Fang", "0-1" — the way Facebook / Instagram
Stories do it. Any time images are added to a post (library gallery of up to five, one loose
library photo, or a camera snap) a step sits **between picking and attaching**: a full-screen
editor where you place text on each image, swipe to the next, and one **Save** attaches them all.

Decisions (voice, 2026-11-07):

- **Burn it in.** The text becomes pixels in the uploaded JPEG. No overlay metadata, no API
  change, readers see a flat photo. Rejected: a separate editable layer — needs a schema, a
  renderer in every reader and the web, for no reader-side gain today.
- **Instagram-level text, images only.** Place, drag, pinch to scale, rotate, a few fonts, a
  text/background colour pair, a background behind the text or none. More than one text block per
  image. Emoji in the text. No stickers, no drawing, no video — "if we get this working, it should
  be all we need."
- **Posts only.** Not binders, not scan, not profile or listing photos. It shows for the gallery
  pick, the single library pick and the camera capture.
- **No skip.** Save is always there; with no text added it just attaches the originals untouched.
- **Counter + Save in the header**, like the multi-card scan queue: `1 of 4` in a glass pill,
  Save to its right, swipe left/right between photos.

## How it looks (from the screenshots)

Bold centered text, one colour on a soft contrasting background that **hugs each line** (not one
box — each wrapped line gets its own rounded slab, the "Stories" look), placed anywhere on the
photo. Example pairs: orange on cream, red on blush. Two blocks per image is common (a caption
top, a result bottom).

## Design

### Where it plugs in

`PostComposer.handleAttachImage` already picks (library ≤ 5 / camera 1) and resizes each asset to
1200×1200-contain JPEG q85 (`post.jpg`). The editor opens **after the resize and before
`pushAttachment`** with the resized `{uri,type,name}[]`. On **Save** it resolves with the same
shape — a burned file for every photo that has text, the original file for every photo that
doesn't — and the existing gallery / loose-photo / camera logic runs unchanged. On **close (X)** it
resolves `null` and the pick is dropped (nothing attached), the same as cancelling the picker.

The composer lives inside a native `<Modal>`, so the editor is **its own full-screen `<Modal
statusBarTranslucent>` + `GestureHandlerRootView`** mounted by the composer (the `ImageViewer`
shape), not a navigator route — no `AppNavigator` change.

### The editor screen — `PhotoTextEditor`

```
┌──────────────────────────────────────────┐
│ (v)                     [ 2 of 4 ] [Save]│  glass circle chevron-down · glass pill · Button
│                                          │
│   ┌──────────────────────────────────┐   │
│   │                                  │   │  canvas: photo at its own aspect,
│   │        ▐ Round 1 vs Gabriel ▌    │   │  full width, centered; text layers
│   │        ▐  from Québec…      ▌    │   │  absolutely positioned on top
│   │                                  │   │
│   │                                  │   │
│   │               ▐ 0-1 ▌            │   │
│   └──────────────────────────────────┘   │
│                                          │
│        ( Aa )   ◐   ●●●●●●   🗑          │  toolbar: add text · bg on/off · colour pairs · delete
└──────────────────────────────────────────┘
```

- **Pager:** horizontal paging `ScrollView` of window-width pages (the ImageViewer carousel),
  `scrollEnabled` off while a layer is being dragged (the viewer's `isZoomed` trick). The header
  pill reads `${index + 1} of ${count}`; a single image hides the pill.
- **Canvas:** per page, a `View` sized to the photo's aspect (`Image.getSize`, capped to the
  space between header and toolbar), the photo `contain`, and the page's text layers inside it.
  The canvas is what gets captured, so nothing outside it (chrome, selection outline) can leak
  into the JPEG.
- **Add text (Aa):** appends a layer centered on the canvas and puts it straight into edit mode
  (keyboard up). Tap an existing layer → edit mode for that layer. Tap empty canvas → deselect /
  commit. Committing an **empty** layer removes it.
- **Edit mode:** the layer itself turns into a multiline, center-aligned `TextInput` (Instagram
  edits in place). The chips live in the toolbar, which rides above the keyboard
  (`KeyboardAvoidingView`): font (4 presets), colour pair (8 presets), background on / off,
  trash. Done = tap outside or the keyboard's return-to-dismiss.
- **Gestures (per layer, RNGH):** `Pan` (drag, 1:1), `Pinch` (scale 0.5–4, applied as a transform
  so the font stays one token size), `Rotation`, run `Simultaneous`, `Race`d against a `Tap`
  that selects / edits. Shared values + saved* values, the `ZoomableImage` pattern. Layers are
  not clamped to the canvas edge (Stories lets text bleed off; the capture crops it).
- **Line-hugging background:** `Text.onTextLayout` returns each wrapped line's `x / y / width /
  height`; a rounded `View` per line is drawn behind the text from those boxes, inset by the
  text padding. In edit mode a measurement twin `Text` (same font / size / width, invisible)
  supplies the lines under the `TextInput`.
- **Save:** for every page with ≥ 1 layer, `captureRef(canvasRef, { format: 'jpg', quality:
  0.85, result: 'tmpfile' })` with the selection outline hidden; pages without layers keep their
  original file. Capture is at device pixel ratio, so a 390pt canvas is ~1170px — the same size
  class the resize targets, and the server resizes again to ~600. The result array goes back to
  the composer. Save shows the Button's loading arc while capturing.

### Presets (all from tokens — no literals)

- **Fonts** (families already in `theme.fonts.family`): `primaryBold` (Roobert Bold, the
  default — the screenshot look), `bodyFamilyMedium` (Balto Medium), `primary` (Roobert
  Regular), `primarySemiBold`. They live as `caption*` slots in `typography.ts` (the only file
  allowed font properties); the one base size is `fonts.size.lg`, scale comes from the pinch.
- **Colour pairs** (text / background): `wanted`/`overlayLight` (orange on cream),
  `error`/`overlayLight` (red on blush), `accent`/`overlayLight`, `info`/`overlayLight`,
  `success`/`overlayLight`, `text`/`scrim` (white on black), `scrim`/`overlayLight` (black on
  white). Background off → text keeps its colour with the `overlayButton` text shadow for
  legibility. No new colour tokens, so `styles/utils/types.ts` stays untouched.

### Files

| File | Change |
|---|---|
| `package.json`, `ios/Podfile.lock` | **new native dep `react-native-view-shot`** (approved by voice 2026-11-07) — `pod install` + native rebuild both platforms |
| `src/components/posts/composer/PhotoTextEditor/index.tsx` | new — modal, header, pager, canvases, toolbar, save/capture |
| `src/components/posts/composer/PhotoTextEditor/TextLayer.tsx` | new — one draggable / pinchable / rotatable text block with the per-line background and in-place edit |
| `src/components/posts/composer/PhotoTextEditor/layers.ts` | new — layer model, font / colour preset tables, `nextLayerKey` |
| `src/styles/components/photoTextEditor.ts` | new style bucket `getPhotoTextEditorStyles(theme)` |
| `src/styles/components/typography.ts` | + `caption*` slots (font family per preset) |
| `src/components/posts/composer/PostComposer.tsx` | `handleAttachImage` awaits the editor between resize and `pushAttachment`; mounts `<PhotoTextEditor>` |

Seven files — over the five-file line, flagged here as the escalation; the native dependency is
the other one. Nothing under `src/contexts/`, `src/styles/utils/` or `src/navigation/`.

### Confidence

- **Insertion in `handleAttachImage`, burned file in / original file shape out:** 0.95 — the
  composer, draft serialisation and upload already treat the file as opaque.
- **`react-native-view-shot` for the flatten on RN 0.83 / new arch:** 0.85 — 4.x supports Fabric;
  capture of a page that is off-screen in a non-virtualised `ScrollView` works because every page
  stays mounted. Rejected: `@shopify/react-native-skia` — a far bigger dependency and a second
  rendering of the text that would never match the on-screen layout exactly.
- **In-place `TextInput` editing with a measurement twin for the line slabs:** 0.8 — same font /
  size / padding / max width on both so wraps match; Android `TextInput` needs `padding: 0` and
  `textAlignVertical: 'top'` to line up. Rejected (fallback if the twin drifts on device): a
  docked input above the keyboard with the layer as a live `Text` preview — no twin, slightly
  less Instagram.
- **Rotation + pinch + pan simultaneous on one layer:** 0.85 — RNGH `Gesture.Rotation` composes
  the same way the existing pinch/pan does; rotation is new to the app.

## Device QA checklist

- Gallery of 5, loose library photo, camera snap — editor shows for each, Save attaches, X drops.
- Text with emoji, two blocks on one photo, a block dragged half off the edge — capture matches
  the screen, text crisp at the post's size.
- No text → originals attach byte-for-byte (no re-encode).
- Keyboard: toolbar stays above the keys on both platforms; dismiss commits; empty commit removes.
- Android three-button nav bar doesn't cover the toolbar (`getBottomBarInset`).
