---
tags: [nanza-figma, handoff, nanza-mobile, collections, cards]
status: open
date: 2026-10-25
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3007-148439
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3007:148439
frame: Collection Cards
repo: nanza-mobile
---

# Collection cards — restyle + market value

## Request (verbatim)
**Changed:**
- Moved item count to below title
- Reused market price for collection value and changed 'price' to 'value'
- Added eye hidden icon for when collection value is hidden
- Restyled item count, collection name and value (same as entity card just different text)

**Do not change:** anything else (not given, so this defaults to everything outside the card)

## Measurements
Two `CollectionThumb` cards, each 182×231: `3007:148762` (value shown) and `3007:148814` (value hidden).

| Element | Spec |
|---|---|
| Card | 182×231, radius 16, 1 top hairline `glassBorder`. Existing image or mosaic plus `ThumbShade` layers, unchanged. |
| Item count | **Stays top-left** (the "below title" note was an error, see Q1). Tag at 12 / 11.5: `surface` fill, radius 5.33, padding 6×8. SemiBold 10 (line box 9.33) `secondaryText`, e.g. "2 items". |
| Name | Bottom-left, 12 inset, 12 from the bottom. SemiBold 13/16 +0.11 `textSecondary` (the `tileLabel` slot), 1 line. |
| Value pill | 6 below the name. **The entity card's Market price pill** (handoff [[2026-10-25-entity-card-styles]]): 1 `glassBorder`, radius 8, padding 5/6/8, gap 4. Amount SemiBold 11/11 `text`; label **"Market value"** Regular 9/10 `metaText`. |
| Value hidden | Same pill. The amount is replaced by an `eye-off` glyph 11×9 (`metaText`); the label still reads "Market value". |
| Value missing | Same pill, amount "–". |

## Code map
- `src/components/cards/CollectionThumb/index.tsx` — the card. Today the count pill sits top-left (`collectionThumbCountWrap`) and the footer shows the name plus `FairMarketLogo` value behind `SHOW_FAIR_MARKET`.
- `src/styles/components/cards.ts` — `collectionThumb*` classes, `thumbCardTitle` (shared by other thumbs, so don't restyle it in place).
- The Market price pill from the entity-card handoff: reuse that component, with the label passed in.
- `src/components/global/Icon/iconMap.ts` — `eye-off` already exists.
- Collection **hide value**: the setting/field may exist only on the dev branch (it isn't in the snapshot this was mapped from). Wire to it there.
- Consumers: `ProfileSections`, `ProfileSeeAllScreen`.

## Open questions
1. The note says the count moved below the title, but Figma keeps "2 items" top-left. Which is right?
   **Answer (Skylar, 2026-10-25):** the note was wrong. The count stays **top-left**.
2. No value data at all (not hidden): show "–"?
   **Answer:** yes, the dash.

## Prompt
```text
Implement the collection card (CollectionThumb) restyle from Figma. Plan first, then build.
Build AFTER (or alongside) the entity-card handoff: this card reuses its Market price pill.

Figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3007-148439
("Collection Cards"). Cards: 3007:148762 (value shown), 3007:148814 (value hidden). Read
this handoff file first (nanza-vault/nanza-figma/handoffs/2026-10-25-collection-card-styles.md).
Its Measurements and answered Open questions are the spec.

NOTE: your branch may be ahead of the snapshot this was mapped from. The collection
"hide value" setting exists on the dev side. Wire to it. If it isn't there, STOP and ask.

SCOPE: CollectionThumb only. Image, mosaic and ThumbShade layers are unchanged. Hidden layers
(collectionThumbFooter, the hidden entityRowMeta lines, Header/TabBar/KeyboardLift mocks) are
artifacts. Ignore them.

1. Moved item count to below title. CORRECTION: the count stays TOP-LEFT (Skylar). Restyle it
   only: surface-filled tag, radius 5.33, padding 6×8, SemiBold 10 secondaryText, at 12 / 11.5.
2. Reused market price for collection value and changed 'price' to 'value'. Under the name
   (gap 6) render the entity card's Market price pill with the label "Market value" and
   the collection value. It replaces the FairMarketLogo row. Show or hide it with the same
   market-price flag the entity card uses (dev team's logic).
3. Added eye hidden icon for when collection value is hidden. When the collection's value is
   hidden, the pill shows the eye-off icon (11×9, metaText) in place of the amount. Missing
   value (no data) → "–".
4. Restyled item count, collection name and value (same as entity card just different
   text). Name = tileLabel SemiBold 13/16 textSecondary, 12 inset from the left and bottom.

Coupled files:
- src/components/cards/CollectionThumb: footer and count.
- src/styles/components/cards.ts: collectionThumb* only. Don't touch the shared
  thumbCardTitle (it backs other thumbs). Add a collection-specific class if needed.
- The Market price pill component from the entity-card handoff: give it a label prop
  ("Market price" / "Market value") and a hidden state, not a fork.

House rules (CLAUDE.md): plan first with confidence scores; proportional sizing; no inline
styles, no raw sizes, theme tokens and typography slots only. Run /ui-review before
finishing. When done, set this handoff's status to built and add a Log line.
```

## Log
- 2026-10-25 — handoff written; Q1–2 answered by Skylar
