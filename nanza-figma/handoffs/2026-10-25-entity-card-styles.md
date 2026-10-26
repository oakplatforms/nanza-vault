---
tags: [nanza-figma, handoff, nanza-mobile, entity, cards]
status: open
date: 2026-10-25
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3002-147837
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3002:147837
frame: Entity Cards
repo: nanza-mobile
---

# Entity cards (lockup rows) — restyle

## Request (verbatim)
**Changed:**
- The image thumbnail is updated with the zoom background
- The font sizes are updated
- The Market Price is styled
- Missing market price is styled
- Price and add to collection style is updated
- Bid and listing cards use the same style
- This should also be reflected in posts

**Do not change:** anything else (no "do not change" given, so this defaults to everything outside these cards)

## Measurements
The frame is a 402-wide sheet with four `EntityRow` instances stacked:
- `3002:148104`: entity with a market price and the add badge
- `3006:148159`: entity with no market price
- `3002:148131`: bid row ("2 wanted")
- `3006:148186`: listing row ("2 in stock")

| Element | Spec |
|---|---|
| Row (entity) | **8 side inset**, 20 top/bottom, bottom hairline `surface`. Thumb 135.06×88, 20 gap to the text column. Use the 8 inset in most places (search etc.). |
| Row (bid / listing) | **20 side inset** (the listing-card case, or any page with a 20pt gutter), 20 top/bottom. Thumb 110.5×72 (same ≈1.535 landscape aspect). Right column holds the price and quantity. |
| Thumb (zoom background) | `neutral.950` fill, top hairline ≈1 `secondary.900` (→ semantic `border`), radius 10. **Backdrop:** the same image, `cover`, scale 1.6, opacity 0.61, offset per Figma (`left −0.29w`, `top −0.04h`). **Blur only on listing rows** (user photos, `blur 3`); entity and bid rows (catalog art) are not blurred. Top tint `rgba(133,150,172,0.22)`→0 over 54.5%. **Card art:** 5:7, 83.4% of thumb height, centered, radius 4. This is the `EntityHeroThumb` recipe (posts / offer line). |
| Name | SemiBold 13/16 +0.11 `text`: the **`tileLabel`** slot (recolor to `text`). 1 line. |
| Number + set lines | Balto Book **12/14** +0.11 `metaText`. Gap 3 from the name; gap 2 between the lines. Set line keeps the `EntityMetaLine` scroll fade. (`entityRowMeta` is 14/16, so this is a smaller step: find or mint a slot.) |
| Market price pill (has price) | 8 below the meta group. Outline pill: 1 `glassBorder`, radius 8, padding 5 top / 6 bottom / 8 sides, gap 4, baseline-aligned. Amount SemiBold 11/11 `text`; label "Market price" Regular 9/10 `metaText`. Both are new slots. |
| Market price pill (missing) | Same pill, amount "–". |
| Add badge (search) | 24 circle, `overlayText` fill, 9pt plus glyph centered, right edge of the row, vertically centered. |
| Price (bid / listing) | Top of the right column. Fill `overlayText`, radius 5.33, padding 4. Bold 11 (line box 9.33) `background`, e.g. "$42.50". |
| Quantity (bid / listing) | 16 below the price, right-aligned. Balto Book 11/11 `secondaryText`, "N wanted" (bid) / "N in stock" (listing). **Hidden at quantity ≤ 1**, as today (`quantityLabel` in `ItemThumb`). |

## Code map
- `src/components/entity/EntityLockupRow/index.tsx` — the one entity lockup recipe. Consumers:
  - `SearchEntityRow`
  - `TradeListItem` (bid/listing rows)
  - `PostContentItems` (posts, `chrome="inline"`)
  - `OfferLineLockup`
  - `BulkSaleCard`
  - `EntityCardBase`

  Its header comment says "style it in entity.ts only".
- `src/styles/components/entity.ts` — `lockup*` classes, `THUMB_WIDTH_PT` (the single thumb-size knob), `LOCKUP_*_INNER_WIDTH`, `lockupRowFloor`.
- `src/components/posts/EntityHeroThumb` — the zoom-background thumb (blur prop `backdropBlurRadius` was added for offers on 2026-10-23).
- `src/components/entity/FairMarketPill` — today's fair-market pill. Becomes the Market price pill (see Q1).
- `src/components/search/SearchEntityRow` + `src/styles/components/searchEntityRow.ts` (`badgeAdd`) — add badge.
- `src/components/trade/TradeListItem` — builds `prices` (`×N`, `$`, or a `Buy`/`Sell` CTA pill).
- `src/components/cards/thumbSizes.ts` — check for a width step before adding one.

## Open questions
1. "Market price" is on every entity row, but fair market is hidden app-wide (`SHOW_FAIR_MARKET`). What's the number and when does it show?
   **Answer (Skylar, 2026-10-25):** Market price **replaces fair market** for now. Rely on the dev team's existing logic or flag for when it shows and when it doesn't.
2. Entity rows use an 8 inset with a 135×88 thumb; bid/listing rows use 20 with 110×72. Intentional?
   **Answer:** yes. 8 is the default (search etc.); 20 is for the listing-card case or any page with a 20pt gutter.
3. At quantity 1, show "1 wanted"? And what happens to the Trade tab's no-price "Buy"/"Sell" pill?
   **Answer:** hide at quantity 1, as today. The CTA pill moves to the new price-pill style.
4. Only listing backdrops are blurred. Intentional?
   **Answer:** yes, listings only.
5. Posts: keep the inline card's outline and gradient, and change only the thumb, type and pill?
   **Answer:** yes, only those changes.

## Prompt
```text
Implement the entity card (EntityLockupRow) restyle from Figma. Plan first, then build.

Figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3002-147837
("Entity Cards", 402 wide). Rows: 3002:148104 (entity + market price + add badge),
3006:148159 (entity, no price), 3002:148131 (bid), 3006:148186 (listing). Read them with
get_metadata / get_design_context to confirm the numbers. Read this handoff file first
(nanza-vault/nanza-figma/handoffs/2026-10-25-entity-card-styles.md). Its Measurements table and
answered Open questions are the spec.

NOTE: your branch may be ahead of the snapshot this handoff was mapped from (the market-price
data/flag in particular). Wire to what exists on your branch. If something the spec needs
isn't there, STOP and ask.

SCOPE: the entity lockup on every surface it renders (search, trade rows, posts, offer
lines, bulk). Change nothing else on those screens. Hidden layers in the frame (Header,
FloatingTabBar, KeyboardLift, the "My Trades / Saved" mock, Button) are capture artifacts.
Don't port them.

1. The image thumbnail is updated with the zoom background. Landscape thumb (135.06×88 at
   the 8 inset, 110.5×72 at the 20 inset, aspect ≈1.535) using the EntityHeroThumb recipe:
   blurred-scale backdrop (×1.6, 0.61 opacity), top tint, centered 5:7 art at 83.4% height,
   radius 4, box radius 10, top hairline. Backdrop blur ONLY for listing photos (reuse the
   existing backdropBlurRadius path; catalog art stays unblurred). Size proportionally via
   THUMB_WIDTH_PT / thumbSizes (135.06/386 and 110.5/362 of the inner width). Don't hardcode.
2. The font sizes are updated. Name = tileLabel (SemiBold 13/16) in `text`. Number + set =
   Balto 12/14 metaText, gaps 3 and 2. Find or mint the 12/14 meta slot in typography.ts.
3. The Market Price is styled. FairMarketPill becomes the "Market price" outline pill (1
   glassBorder, radius 8, padding 5/6/8, gap 4, amount SemiBold 11/11 text + "Market price"
   Regular 9/10 metaText), 8 below the meta. Market price REPLACES fair market for now. Show
   or hide it with the dev team's existing logic/flag (SHOW_FAIR_MARKET or its successor).
4. Missing market price is styled. Same pill with "–" (FairMarketPill's existing dash rule).
5. Price and add to collection style is updated. Bid/listing price = overlayText-filled
   tag (radius 5.33, padding 4, Bold 11, `background` text). Quantity "N wanted / N in stock"
   Balto 11/11 secondaryText, 16 below, hidden at ≤1. The Trade tab's no-price Buy/Sell CTA
   pill takes the same tag style. Search's add badge = 24 circle, overlayText fill, 9pt plus.
6. Bid and listing cards use the same style. TradeListItem rows render the right column
   (price tag over quantity) instead of the ×N + price pills. Row inset 20 (gutter pages),
   entity rows inset 8.
7. This should also be reflected in posts. PostContentItems' inline entity card keeps its
   outline + gradient chrome. Only the thumb, type and pill change.

Coupled files:
- src/styles/components/entity.ts: all lockup sizing; THUMB_WIDTH_PT moves every thumb.
- src/components/entity/EntityLockupRow: new right-column layout (opt-in for trade rows),
  inset variant (8 vs 20), thumb slot.
- src/components/posts/EntityHeroThumb: reuse; don't fork the recipe.
- src/components/entity/FairMarketPill: restyle and relabel. Also used by EntityScreen and
  DetailTitleBlock: check those still look right or scope the change.
- src/components/trade/TradeListItem, src/components/search/SearchEntityRow (+ searchEntityRow.ts),
  src/components/offers/OfferLineLockup (its 2026-10-23 opt-in heroThumb props), BulkSaleCard.
- src/styles/components/typography.ts: new slots (12/14 meta, 11/11 amount, 9/10 label,
  Bold 11 tag, Balto 11/11 quantity). Check existing slots first.

Traps / decisions (raise them in the plan, don't guess):
- lockupRowFloor and the inline vs row inner widths are derived from THUMB_WIDTH_PT; a
  landscape thumb changes the floor math.
- Figma colours on the thumb are scale refs (neutral.950, secondary.900). Use the semantic
  tokens (background/well, border) per house rules.
- The offer line card already ships a landscape EntityHeroThumb (2026-10-23). Converge on
  one path, don't add a second.
- Over 5 files: plan for the CLAUDE.md escalation.

House rules (CLAUDE.md): plan first with confidence scores per item; proportional sizing
(fractions of the 402 frame / 362 or 386 inner width); no inline styles, no raw sizes, theme
tokens and typography slots only. Run /ui-review before finishing. When done, set this
handoff's status to built and add a Log line.
```

## Log
- 2026-10-25 — handoff written; Q1–5 answered by Skylar
