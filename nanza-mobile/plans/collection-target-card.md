# Collection Target Card — Design

Redesign the rows in `CollectScanTargetScreen` (the "Choose Collections" step of the
collection-scan flow) so each collection renders as a rich card that mirrors the **OG
Meta collection share graphic** (`nanza-api/src/services/og/templates/collection.ts`):
a full-bleed mosaic of the collection's card images, with a connected info bar below.

## Reference graphic

The OG collection card is a full-bleed 6-column mosaic of portrait card images with a
dark bottom drop-shadow gradient. We adopt that mosaic look as the card's top image
area. **No nanza logo** (per voice — the OG card has the wordmark; we drop it here).

## Card anatomy (per collection)

```
┌───────────────────────────────────────────┐
│ ┌───────────────────────────────────────┐ │
│ │ [Private]                   225 items │ │  ← overlay row on the mosaic
│ │                                       │ │
│ │   ███ ███ ███ ███ ███ ███             │ │  ← mosaic of entity images
│ │   ███ ███ ███ ███ ███ ███             │ │
│ │   ░░░░░░░ dark gradient ░░░░░░░░░░     │ │
│ └───────────────────────────────────────┘ │
│  Collection Name                          │  ← connected info bar below
│  $1,234.56  fair market                   │
└───────────────────────────────────────────┘
```

### Top — mosaic image area
- Full-bleed grid of `collection.entityList[].entity.image` (via `getImage`), styled like
  the OG card: thin dark gridlines, portrait cells, dark bottom gradient so overlays stay
  legible. Reuse the share-card mosaic approach; cap visible cells (e.g. 6–12).
- Fallback when a collection has no items: flat dark fill (`neutral['800']`).

### Top-left overlay — privacy pill
- **`isPrivate === true`** → outlined pill chip reading **"Private"**.
- **Public (`isPrivate` falsy)** → render **nothing** (per voice).
- Only Private is supported today, so in practice it always shows Private — but the
  public case is handled.

### Top-right overlay — item count
- `${entityListCount} items` (singular "item" when count === 1).

### Bottom info bar — connected card section
- **Collection name**: `displayName || name || 'Collection'`.
- **Fair market value**: `entityListValue` rendered with `PriceText` (so the "$" uses
  Figtree per P0 rule #6), followed by the muted label **"fair market"** to the right.
  - This is the SAME value shown as "Portfolio Value" on the profile
    (`collectionList?.entityListValue`), just per-collection. No recalculation needed —
    the value is already on each `ListDto`.

## Data — all on `ListDto` (`@oakplatforms/types`), already fetched

`getAllListsQueryConfig` already includes `entityList.entity`, so every field is present:

| Need | Field |
|---|---|
| Mosaic images | `entityList[].entity.image` |
| Item count | `entityListCount` |
| Fair market value | `entityListValue` |
| Privacy | `isPrivate` |
| Name | `displayName` / `name` |

No API change, no new DTO. (P0 #9, #12 satisfied.)

## Files to touch

1. **`src/components/share/CollectionTargetCard.tsx`** (NEW) — the card component.
   Props: `{ collection: ListDto; isSelected: boolean; onPress: () => void }`.
   Renders mosaic + overlays + info bar. Reuses `getImage`, `PriceText`.
   - DRY note: structurally similar to `ShareGroupCard.tsx`; that one is a single banner,
     this one is a mosaic, so a shared component isn't a clean fit — separate component.
2. **`src/styles/components/share.ts`** — add `collectionTargetCard*` style classes
   (frame, mosaic wrapper, mosaic cell, gradient, overlay row, privacy pill, count text,
   info bar, name, value row, fair-market label, selected state). All theme tokens; no
   inline styles; theme-based color fallbacks only. (P0 #1–5.)
3. **`src/screens/trade/CollectScanTargetScreen/index.tsx`** — swap the plain
   `TouchableOpacity` text rows in `renderItem` for `<CollectionTargetCard ...>`.
   Selection still toggles via `toggleList`; selected state shown on the card (ring/border).
   Footer "Next" button unchanged.

No navigation changes. No shared-infra changes.

## Confidence

- **Reuse OG mosaic look for the picker card** — 0.9. Clear directive from voice; data and
  the `getImage` mosaic pattern already exist.
- **Render value via `PriceText` using `entityListValue`** — 0.92. Exactly the profile's
  portfolio-value field; PriceText satisfies the dollar-sign P0 rule.
- **Separate `CollectionTargetCard` vs. extending `ShareGroupCard`** — 0.8. Group card is a
  single banner; collection card is a multi-image mosaic with a different info bar.
  Rejected alt: generalize `ShareGroupCard` into one card (Confidence 0.5) — the two
  layouts diverge enough that a shared abstraction would be conditional-heavy.

## Mosaic sizing — FIXED CELL SIZE, not fixed columns (confirmed via voice)

The card is **always 100% width** (full bleed in the picker, any screen size). We keep the
**same chicklet (cell) size** the OG card uses and let the **number of columns float** to
fill the width — so on a wide card you get ~12 across instead of 6/8. Approach:

- Fixed portrait cell size: pick a constant cell width (~`OG cell` feel, e.g. ~52–56px)
  and `CARD_ASPECT = 1.4` for height. Cell stays the same regardless of screen width.
- `columns = floor(cardWidth / (cellWidth + gap))` — computed at render from the measured
  card width (`onLayout`), so it adapts to device width. This is a legitimate dynamic
  runtime value, so the grid container width math can be inline per the P0 exception;
  static cell/gap styling stays in the stylesheet.
- Rows: enough portrait rows to fill a fixed-height band; gradient pinned to the bottom.
- Thin dark gridlines between cells (gap shows the dark frame through), matching OG.

## Status — BUILT (2026-06-21)

- NEW `src/components/share/CollectionTargetCard.tsx` — mosaic + overlays + info bar.
- `src/styles/components/share.ts` — added `collection*` style classes.
- `src/screens/trade/CollectScanTargetScreen/index.tsx` — rows now render the card
  (wrapped in `rowWrapper` for spacing); dropped the old text-row + Icon import.
- Fixed cell size: CELL_WIDTH 52 / CELL_HEIGHT 73 / GAP 3, columns float with measured
  card width via onLayout; MAX_ROWS 3; mosaic band height 168.
- Old `collectionTargetRow*` styles left in scanItems.ts (orphaned, harmless).
- tsc: zero errors in the 3 changed files (131 pre-existing baseline errors elsewhere).
- Not committed (per user's no-commit rule). Not yet run on device.
