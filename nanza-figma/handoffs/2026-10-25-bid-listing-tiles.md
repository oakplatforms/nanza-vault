---
tags: [nanza-figma, handoff, nanza-mobile, cards, listings, bids]
status: open
date: 2026-10-25
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3007-148846
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3007:148846
frame: Collection Cards (bid / listing tiles)
repo: nanza-mobile
---

# Bid & listing tiles (ItemThumb) — restyle

## Request (verbatim)
**Changed:**
- Card names should wrap to no more than two lines before a trailing ellipses is added and its cut off
- The background images should be sized and positioned to match Figma and the blur should be matched 1:1, same with the image size/crop
- The WTB and WTS tag styles are updated
- The title, price and quantity is updated
- Make sure the gradient overlay matches and the border style)

**Do not change:** anything else (not given, so this defaults to everything outside the tile)

## Measurements
The frame name says "Collection Cards" but it holds bid and listing tiles in a 2-column grid (`3007:149132`):
- `3007:149090`: bid
- `3007:149091`: listing, blurred photo
- `3007:149153`: bid with a 2-line name

| Element | Spec |
|---|---|
| Grid | **8 side inset, 12 column gap**, 12 row gap (tiles at x 0 / 196, y 0 / 195). The same tile in grids **and carousels**. |
| Tile | **Square** 186×186 (≈ (386 − 12) / 2). Fill `neutral.950` (→ semantic background/well), **1.33 top hairline `glassBorder`**, radius 20.32. |
| Backdrop | The tile's image, `cover`, 294×190 (scale ≈1.58 of the tile width), left −54.2, vertically centered (−1.1), opacity 0.61. **Blur 3 on listings only** (user photos); bids (catalog art) are not blurred. |
| Bottom shade | Linear top→bottom: `rgba(0,0,0,0)` → 0.8 at 60% → 0.8 (Figma style `Nanza/gradient/shadeBottomDeep`). |
| Top tint | `rgba(133,150,172,0.22)` → 0, about 54.5% of the height (sits about 10 above the top). |
| Card art | 5:7, 70.76×99.08 (≈53.3% of tile height), horizontally centered, top 20.86, radius 5.33, `cover`. |
| Intent tag | Top-left at 12 / 10.67. Radius 5.33, padding 3×4, Bold 10 (line box 9.33). **WTB = `conditionColors.mint` fill, `tertiary.950` text. WTS = `primary.400` fill, dark pink text** (unbound `#580b55`; nearest is the primary scale's 950, so confirm the token in the plan). This is **swapped vs today** (today WTS = mint, WTB = pink). |
| Footer | Bottom 11.5, 12 side inset (162 wide). Name SemiBold 13/16 +0.11 `textSecondary` (`tileLabel`), **max 2 lines, tail ellipsis**. 6 below: price tag + quantity row, gap 8. |
| Price tag | `text` fill, radius 5.33, padding 4, Bold 11 (line box 9.33) `background`. |
| Quantity | Balto Book 11/11 `metaText`, "N wanted" / "N in stock". Hidden at ≤ 1, as today. |

## Code map
- `src/components/cards/ItemThumb/index.tsx` — the tile (`TITLE_MAX_LINES = 2` already; `aspect` 'photo' 189:240 / 'card' 143:200 → becomes square).
- `src/components/cards/ItemThumb/ThumbShade.tsx` — gradient layers.
- `src/components/cards/ListingThumb`, `BidThumb` — DTO → ItemThumb mappers.
- `src/components/global/IntentTag` + `src/styles/components/badges.ts` (`intentTagBuy` / `intentTagSell`, tile / row / compact variants) — colour swap applies to **every** variant (decided below).
- `src/components/cards/thumbSizes.ts` — tile widths for grids and carousels.
- `src/styles/components/cards.ts` — `thumbCard*`, `thumbPricePill*`, `thumbQuantityText`, `thumbIntentCorner`, `cardGridRows`.
- Grids and carousels that render it:
  - `FeedGridScreen`
  - `TradeMarketGrid`
  - `EntityScreen` (trades)
  - `GroupDetail` / `GroupSeeAll`
  - `ProfileSections` / `ProfileSeeAll`
  - `OfferSellerListingsPicker`
  - `CardCarousel` consumers
- `src/components/posts/EntityHeroThumb` — same backdrop + art recipe. Reuse its math rather than re-deriving it.

## Open questions
1. WTB/WTS colours are swapped vs the app. Swap everywhere, including the detail screens' row tag?
   **Answer (Skylar, 2026-10-25):** swap everywhere to match Figma.
2. Square tile in grids only, or carousels too?
   **Answer:** the Figma tile size in **grids and carousels**.
3. Grid spacing: change it to 8 inset / 12 gap?
   **Answer:** yes, use the Figma grid spacing.

## Prompt
```text
Implement the bid/listing tile (ItemThumb) restyle from Figma. Plan first, then build.

Figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3007-148846 (grid
3007:149132; tiles 3007:149090 bid, 3007:149091 listing, 3007:149153 bid with a 2-line name). Read
them with get_metadata / get_design_context. Read this handoff file first
(nanza-vault/nanza-figma/handoffs/2026-10-25-bid-listing-tiles.md). Its Measurements and
answered Open questions are the spec. Your branch may be ahead of the mapped snapshot. Wire to
what exists there, and STOP and ask if something is missing.

SCOPE: the ItemThumb tile (ListingThumb / BidThumb), its grid spacing, and IntentTag colours.
Change nothing else on the host screens. Hidden layers (extra IntentTags, hidden entityRowMeta
lines, Header/TabBar mocks) are artifacts.

1. Card names should wrap to no more than two lines before a trailing ellipsis is added and
   it's cut off. numberOfLines 2 + ellipsizeMode tail on the name (tileLabel 13/16,
   textSecondary). Verify the 2-line case (3007:149153) doesn't push the price row or overlap
   the art.
2. The background images should be sized and positioned to match Figma and the blur should
   be matched 1:1, same with the image size/crop. Square tile. Backdrop cover at ≈1.58×
   tile width, left −0.29w, vertically centred, opacity 0.61. Blur 3 on LISTINGS only (reuse
   EntityHeroThumb's backdropBlurRadius path; iOS paints blurRadius blank on catalog scans,
   which is why bids stay unblurred). Art 5:7 at ≈53.3% of tile height, centred, top
   ≈11.2%, radius 5.33. Express everything as fractions of the tile, not points.
3. The WTB and WTS tag styles are updated. Radius 5.33, padding 3×4, Bold 10. COLOURS SWAP
   app-wide: WTB = conditionColors.mint fill + tertiary.950 text, WTS = primary.400 fill +
   the dark-pink text (pick the token, e.g. primary.950, and confirm it in the plan). All
   IntentTag variants (tile/row/compact). Update the IntentTag header comment.
4. The title, price and quantity is updated. Footer 12 inset, 11.5 from the bottom, name →
   6 → row (price tag `text` fill, radius 5.33, padding 4, Bold 11 `background`; gap 8;
   quantity Balto 11/11 metaText, hidden at ≤1).
5. Make sure the gradient overlay matches and the border style. Bottom shade 0 → 0.8 @60% →
   0.8 (shadeBottomDeep gradient token) plus the top tint 0.22 → 0. Border = 1.33 TOP hairline
   glassBorder only, radius 20.32.
6. Grid + carousel sizing (Skylar): square tile everywhere (grids AND carousels), grid
   8 side inset / 12 gap. Add the width step(s) to thumbSizes.ts as fractions of 402.

Coupled files:
- src/components/cards/ItemThumb (+ ThumbShade), ListingThumb, BidThumb.
- src/components/cards/thumbSizes.ts: tile widths; carousels and grids read from here.
- src/styles/components/cards.ts: thumbCard*, cardGridRows (other thumbs share some of these,
  so add tile-specific classes rather than editing shared ones).
- src/components/global/IntentTag + badges.ts: colour swap.
- Every grid/carousel host listed in the handoff's Code map: check that the square tile fits
  each one.

Traps / decisions (raise them in the plan, don't guess):
- `aspect` ('photo' | 'card') becomes square. Remove or replace the prop and check every
  caller passing it.
- Lots (BulkThumb) also render through ItemThumb with WTS. Confirm they take the same tile.
- Over 5 files: plan for the CLAUDE.md escalation.

House rules (CLAUDE.md): plan first with confidence scores; proportional sizing; no inline
styles, no raw sizes, theme tokens and typography slots only. Run /ui-review before
finishing. When done, set this handoff's status to built and add a Log line.
```

## Log
- 2026-10-25 — handoff written; Q1–3 answered by Skylar
