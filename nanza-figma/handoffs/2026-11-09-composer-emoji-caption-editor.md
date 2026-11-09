---
tags: [nanza-figma, handoff, nanza-mobile, posts, composer, captions]
status: open
date: 2026-11-09
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3395-211411&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3395:211411
frame: Post-Compose-ImageCaption / default · Seller · edit me (PhotoTextEditor, editing state)
repo: nanza-mobile
scope: styles-only
---

# Composer — emoji no longer clipped, caption editor restyle, Aa moved to the header

Three changes to the post composer, all built and tested on device on a throwaway branch
(`throwaway-composer-edits`). The values below are that build, so treat it as the reference
implementation. Independent of the other 2026-11-09 handoffs.

> [!note] Scope
> - **Change 1** touches only the composer's text fields; nothing outside the composer moves.
> - **Changes 2–3** are the photo text editor (`PhotoTextEditor`): the step after picking
>   photos, where you add caption text and press Save.
> - The Figma frame's editor layers come through flattened (the MCP metadata only lists
>   HomeScreenContent), so measurements are read from the frame's 1x render.

## Request (verbatim)
1. "im having issues with emojis on the second line … its slightly cut off, not sure why and dont
   want to break another part of the app fixing just want to be able to see full emojis in the
   composer"
2. "we just added captions to images, update to match this style" (frame 3395:211411). Then, on
   device: "add the glass button styling to the color chips, same as the trash icon, same with the
   normal button (inactive state)", "try setting the color chips to … the background color we use
   for the active state of tabs on the profile screen", "set that same style on the vertical size
   selector. not the dragger but the triangle", "try centering the bold/normal".
3. "on the screen just after you upload an image, … move the Aa button to add text to the left of
   the save button so its easier to see and access visually consistent with other actions"

**Do not change:** not stated, so: the caption layers themselves (drag / pinch / rotate, fonts,
colour pairs, burn-in on Save), the size slider's knob, the header's back and Done controls.

## 1. Composer: emoji on line 2+ cut off at the top
**Cause:** the editor (`postComposer.ts composerTextArea`) is `bodyLg`, Balto 17 on a **22**
line. Apple Color Emoji at 17 stands taller than 22, and a multiline iOS `TextInput` clips every
line after the first to the line box, so the first line's emoji were whole and later lines' had
their tops cut off.

**Fix:** the same rule posts already use (`EmojiText` / `postBodyEmoji`). While a field's text
holds an emoji, swap to a roomy line.

| Slot (`typography.ts`) | Font | Line |
|---|---|---|
| `bodyLg` (unchanged) | Balto 17 | 22 |
| **`bodyLgEmoji`** (new) | Balto 17 | **24** (`lineHeight['24']`). The 16→22 / 18→25 twins put 17 at ~24 |

```ts
// postComposer.ts
composerTextAreaEmoji: { ...typography.bodyLgEmoji },

// PostComposer.tsx — main editor and every extra text block
style={[styles.composerTextArea, isReply && styles.composerTextAreaReply, hasEmoji(text) && styles.composerTextAreaEmoji]}
style={[styles.composerTextArea, styles.textBlockInput, hasEmoji(attachment.body ?? '') && styles.composerTextAreaEmoji]}
```
`hasEmoji` is the existing export from `components/global/EmojiText`. Side effect: lines space out
by 2 when the first emoji is typed and close back up when the last is deleted. That's expected.

## 2. Caption editor restyle (Figma 3395:211411 + device calls)

| Part | Before | After |
|---|---|---|
| Background toggle label | "A" | **"Aa"**, both states (lit = light fill + dark text, as before) |
| Colour swatch | 32 disc, **2pt ring in the pair's slab colour**, white selected ring, 18 text-colour dot | 32 disc, fill **`getGlassActiveFill`** (the profile tabs' active segment, `tabs.ts segmentedTabActive`) + **`GlassRim`**, 18 dot in the pair's text colour. **No slab ring, no selected ring**: the slab on the photo shows the live pick |
| Font chips (Bold / Normal) | left-aligned; inactive chip = glass fill only | **centred** across the bar (`toolbarRowCentered`); inactive chip gets the pill **`GlassRim`** (`pill radius={radius.max}`), like the "1 of 3" counter; active chip unchanged (light inversion) |
| Size slider: wedge (the triangle) | `overlayLight` | **`getGlassActiveFill`**, fill only (a GlassRim can't follow the wedge). The knob is unchanged |

Measured from the frame (1x): toggle and trash are 44 circles; swatches are 32 discs with an 18
dot, 6 apart (`toolChipRow` gap `xs`); 8 between toggle → first swatch → … → trash
(`toolbarRow` gap `sm`); Bold / Normal chips are 32 tall, 8 apart.

```ts
// photoTextEditor.ts
swatch: { width: 32, height: 32, borderRadius: radius.max, backgroundColor: getGlassActiveFill(theme), justifyContent: 'center', alignItems: 'center' },
// swatchActive — deleted
toolbarRowCentered: { justifyContent: 'center' },
```
```tsx
// PhotoTextEditor/index.tsx
<TouchableOpacity style={styles.swatch} …>       {/* no inline borderColor, no swatchActive */}
  <GlassRim />
  <View style={[styles.swatchInner, { backgroundColor: pair.text }]} />
</TouchableOpacity>
{active ? null : <GlassRim pill radius={theme.radius.max} />}   {/* inside the inactive font chip */}

// SizeSlider.tsx
<Path d={wedge} fill={getGlassActiveFill(theme)} />
```

## 3. Aa (add text) moves into the header
- At rest, the header's right side reads **[1 of N]** (multi-photo only) → **Aa** → **Save**.
  Aa is the same glass circle as the header's back button (`toolCircle` + `GlassRim`, 44),
  `hitSlop` from `getControlHitSlop`, disabled while saving.
- While a block is being edited, the header still shows only **Done** (no Aa: a tap on the photo
  starts the next block, as before).
- The bottom toolbar now **renders only while editing**. At rest it would be empty, and its
  `onStartShouldSetResponder` dead zone would swallow taps on the bottom of the photo.

## Code map (local `throwaway-composer-edits`; match these on your branch)
- `src/styles/components/typography.ts`: new `bodyLgEmoji`.
- `src/styles/components/postComposer.ts`: new `composerTextAreaEmoji`.
- `src/components/posts/composer/PostComposer.tsx`: `hasEmoji` import; emoji style on the editor and text blocks.
- `src/components/posts/composer/PhotoTextEditor/index.tsx`: Aa in the header; toolbar gated on editing; "Aa" toggle; swatch + chip rims; centred chip row.
- `src/components/posts/composer/PhotoTextEditor/SizeSlider.tsx`: wedge fill.
- `src/styles/components/photoTextEditor.ts`: `swatch` fill, `swatchActive` removed, new `toolbarRowCentered`.

## QA
- **Composer:** type several lines with emoji on lines 2 and 3, in the main field and in an
  added text block; nothing clipped. Type / delete the first emoji: the spacing steps 22 ↔ 24.
  A field without emoji is unchanged.
- **Caption editor, at rest:** one photo shows Aa then Save; several show "1 of N", Aa, Save.
  Aa drops a text block. No bottom bar; taps near the bottom of the photo still work.
- **Caption editor, editing:** Bold / Normal centred, inactive chip has the glass rim; toggle
  reads "Aa"; swatches show the active-tab fill + rim with the text-colour dot, and picking
  one recolours the slab; the trash circle is unchanged; the size slider's wedge is the active-tab
  fill and its knob still drags.
- **Save** still burns captions into the photos as before.

## Open questions
None.

## Prompt
```text
Implement the handoff nanza-vault/nanza-figma/handoffs/2026-11-09-composer-emoji-caption-editor.md
(read it in full). Figma file 7c3Zz4nDUOutQG1BsnRRAw, frame 3395:211411.

SCOPE: STYLES ONLY, plus moving the Aa control. No data or behaviour changes to captions.

1. COMPOSER EMOJI: new typography slot bodyLgEmoji (Balto 17 / 24); postComposer.ts
   composerTextAreaEmoji spreads it; PostComposer applies it to the main editor and every text
   block when hasEmoji(text) (components/global/EmojiText).
2. CAPTION EDITOR: background toggle label "Aa"; swatch = 32 disc, getGlassActiveFill + GlassRim,
   18 text-colour dot, no slab-colour ring, delete swatchActive; inactive font chip gets
   <GlassRim pill radius={radius.max} />; font chip row centred (toolbarRowCentered); SizeSlider
   wedge fill getGlassActiveFill (knob unchanged).
3. AA: move the add-text Aa circle from the bottom toolbar into the header, left of Save
   (after the "1 of N" counter); at rest only. Render the bottom toolbar only while a block is
   being edited.

House rules (CLAUDE.md): plan first with confidence scores; styles in
src/styles/components/*; font props only in typography.ts; no inline styles except runtime data
colours (the swatch dot); theme tokens only. Run /ui-review before finishing, then set this
handoff to built and add a Log line.
```

## Log
- 2026-11-09: handoff written; all three changes built and device-tested on `throwaway-composer-edits`
