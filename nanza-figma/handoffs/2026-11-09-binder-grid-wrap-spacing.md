---
tags: [nanza-figma, handoff, nanza-mobile, collections, binder, grid]
status: open
date: 2026-11-09
repo: nanza-mobile
scope: styles-only
builds-on: 2026-11-09-tag-page-fixed-hero
---

# Binder grid — third card no longer wraps, spacing aligned with the title

A bug fix plus a spacing pass on the collection / binder detail grid (`CollectionDetailScreen`),
built and tested on device on a throwaway branch (`throwaway-binder-items-wrapping`). No Figma
frame: the designer worked from device screenshots. **Builds on
[[2026-11-09-tag-page-fixed-hero]]**, which turned `collectionDetailGrid`'s row gap from a margin
into opaque padding (the rows scroll over the pinned banner). That still holds; only the values
change here.

## Request (verbatim)
1. "this next one seems to only be an issue for me but i can see it everytime i open my
   binder/collection with more than 3 items. the wrapping is broken"
2. "ok its fixed but can we improve the spacing, more between cards (not much) and aligned with
   the title"

## 1. Bug: the third card wraps onto its own line
**Symptom:** a 3-column grid rendered 2 + 1 per row (rows of uneven card sizes), on a 402pt-wide
device.

**Cause:** the card width was the exact share of the row,
`(window − 8·2 − 8·2) / 3` = **123.33** on a 402 window. Three cards plus the gaps and padding
summed to **exactly** the window width, so any rounding over tipped the row. The FlatList row
style (`columnWrapperStyle` = `cards.ts collectionDetailGrid`) carried **`flexWrap: 'wrap'`**,
which dropped the overflowing third card onto a new line. Other window widths happen to land a
hair under, which is why it looked device-specific.

**Fix (both parts):**
- `collectionDetailGrid`: **remove `flexWrap: 'wrap'`**. `numColumns` already puts exactly
  `COLUMNS` cards in each row, so wrapping can only break it.
- `CollectionDetailScreen` `cardWidth`: **floor to a whole device pixel**, so a row always has a
  hair of slack:
```ts
const cardWidth = useMemo(() => {
  const exact = (windowWidth - theme.spacing.base * 2 - theme.spacing.gutter * (COLUMNS - 1)) / COLUMNS
  const scale = PixelRatio.get()
  return Math.floor(exact * scale) / scale
}, [windowWidth, theme.spacing.base, theme.spacing.gutter])
```

## 2. Spacing

| | Before | After |
|---|---|---|
| Grid side inset | 8 (`sm`) | **20** (`base`), in line with the title block (`collectionDetailHeaderBlock` 20) |
| Between cards, across | 8 (`columnGap` `sm`) | **12** (`gutter`) |
| Between rows | 8 (`paddingBottom` `sm`) | **12** (`gutter`) |
| Card width on a 402 window | ~123.3 | ~113.3, floored to the pixel |

```ts
// cards.ts
collectionDetailGrid: {
  flexDirection: 'row',                    // no flexWrap
  columnGap: theme.spacing.gutter,         // 12
  paddingBottom: theme.spacing.gutter,     // 12 — opaque row gap (fixed-hero recipe)
  backgroundColor: theme.colors.background,
  alignItems: 'flex-start',
  paddingHorizontal: theme.spacing.base,   // 20
},
```
`cardWidth` must read the same tokens as the row (`base` sides, `gutter` gaps). If one changes
without the other, the row overflows again.

## Code map (local `throwaway-binder-items-wrapping`; match these on your branch)
- `src/styles/components/cards.ts`: `collectionDetailGrid` (only consumer: CollectionDetailScreen).
- `src/screens/collections/CollectionDetailScreen/index.tsx`: `cardWidth` (tokens + pixel floor), `PixelRatio` import.

## QA
- Open a binder / collection with more than 3 items on a 402pt-wide device (iPhone 16 / 17 Pro):
  every row holds 3 cards, all the same size.
- Check one other width (e.g. a 393 or 430 device): still 3 per row.
- The grid's left edge lines up with the "Binder" title; 12 between cards across and down.
- Scrolling over the pinned banner: the gaps between rows stay solid (the fixed-hero recipe).
- Quantity badges, long-press and the search footer are unchanged.

## Open questions
None.

## Prompt
```text
Implement the handoff nanza-vault/nanza-figma/handoffs/2026-11-09-binder-grid-wrap-spacing.md
(read it in full). It builds on 2026-11-09-tag-page-fixed-hero (collectionDetailGrid's opaque
row padding).

SCOPE: STYLES + the grid's width math only. No data or behaviour changes.

1. BUG: remove flexWrap from cards.ts collectionDetailGrid; floor CollectionDetailScreen's
   cardWidth to a whole device pixel (PixelRatio) so a row never overflows.
2. SPACING: collectionDetailGrid paddingHorizontal base (20, in line with the title), columnGap
   and paddingBottom gutter (12). cardWidth uses the same tokens: (window - base*2 -
   gutter*(COLUMNS-1)) / COLUMNS, floored.

House rules (CLAUDE.md): plan first with confidence scores; styles in
src/styles/components/*; theme tokens only. Run /ui-review before finishing, then set this
handoff to built and add a Log line.
```

## Log
- 2026-11-09: handoff written; wrap fix + spacing built and device-tested on `throwaway-binder-items-wrapping`
