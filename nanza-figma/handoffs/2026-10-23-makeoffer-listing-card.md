---
tags: [nanza-figma, handoff, nanza-mobile, offers]
status: built
date: 2026-10-23
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=2836-103033
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 2836:103033
frame: MakeOffer / default · Seller · edit me
repo: nanza-mobile
---

# Make Offer — listing card redesign

## Request (verbatim)
**Changed:**
- listing card image styling
- listing card typography
- use "/" to separate multiple tags
- alignment of price and quantity
- spacing around remove from offer
- removed pink quantity indicator
- add quantity selector

**Do not change:** anything else on this screen

## Measurements
Edited subtree `Frame 3` (`2840:126585`), 362 wide at the 20pt inset of the 402 frame.

| Element | Spec |
|---|---|
| Thumb | 128.48×88 landscape (≈1.46), ≈0.355 of 362; blurred backdrop (scale 1.6, opacity 0.61, blur 16), top gradient `rgba(133,150,172,0.22)`→0 over 54.5%, photo contain |
| Thumb → text | 16 |
| Name | `Nanza/itemTitle` Roobert SemiBold 15/16, ls 0.11, `theme.colors.text`; 4 below thumb top |
| Meta lines | `Nanza/entityRowMeta` 14/16, `theme.colors.metaText`; line 1 is 6 below the name, line 2 is 2 below line 1. Figma names the font "Balto Mobile" ⚠️ |
| Tag line | `Promotion Card / GEM Pack 4` |
| Terms row | 52 tall, 16 below the thumb row; price box 106.5×52 left (≈0.294), amount centered; count + stepper right-aligned, centered vertically |
| Stepper | glass: 44pt circles, count in between with 16 each side; count label 20 from the stepper |
| Remove from offer | `Nanza/small` Roobert SemiBold 12/16, 16 below the terms row, left-aligned |

## Code map
- `src/components/offers/OfferLineCard/index.tsx` — the card
- `src/components/offers/OfferLineLockup/index.tsx` — thumb + pink badge; **also used by `OfferMessageDetailScreen`**
- `src/components/offers/OfferTermsRow/index.tsx` — price, count, stepper (`variant="glass"` exists); **also used by `ListingScreen`, `BidScreen`**
- `src/components/entity/EntityLockupRow`, `EntityMetaLine` — tag line; separator is app-wide (`TAG_LINE_SEPARATOR = ' · '` plus a drawn dot)
- `src/styles/components/offers.ts` — `line*` / `terms*` classes
- `src/components/cards/thumbSizes.ts` — needs a new width step

## Open questions
1. Show the stepper when only 1 is available? Today it's hidden at ≤1 and reads `Only 1 available`. Figma shows a stepper with `1 available`.
   **Answer (Skylar, 2026-10-23):** show it; the label reads "1 available". With more than one, it still counts down "N left".
2. Price box: fixed 106.5 on this card only, or keep the hugging box (a 2026-09-16 decision)?
   **Answer:** keep the hugging box.
3. Does the offer detail screen drop the pink badge too, or only the make-offer card?
   **Answer:** make-offer card only.
4. Gap below "Remove from offer" down to "Add more from this seller": change it, or leave it?
   **Answer:** left as is (covered by "do not change").
5. (Found while implementing.) Figma draws the line on the page with no surface card behind it. Should the card container go?
   **Answer:** yes, remove it; the line sits on the 20pt gutter.

## Prompt
```text
Implement the Make Offer listing card redesign from Figma. Plan first, then build.

Figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=2836-103033 (frame "MakeOffer / default · Seller · edit me", 402×874, the standard reference frame). The redesigned card is "Frame 3" (2840:126585), 362 wide at the 20pt screen inset. Read it with get_metadata / get_design_context to confirm the numbers below. Check the "Open questions" section of this handoff file (nanza-vault/nanza-figma/handoffs/2026-10-23-makeoffer-listing-card.md) for answers before planning.

SCOPE: only the listing card in the offer stack. Change nothing else on this screen. "Add more from this seller", the Shipping/Payment rows, OfferSummary, the terms checkbox and the CTA stay as they are. The Figma frame shows red row labels and a grey summary panel; those are artifacts of the edit copy, not part of the redesign. Do not port them.

1. Listing card image styling. The thumb is now a 128.48×88 landscape tile (aspect ≈1.46) instead of the order-card portrait thumb. It has a blurred backdrop of the same photo (scale 1.6, opacity 0.61, blur 16), a top gradient (rgba(133,150,172,0.22) → 0 over the top 54.5%), and the photo object-contain on top. Before building anything new, check whether EntityHeroThumb / PostContentItems / DetailHero already have this blurred-backdrop recipe, and reuse it if so. The width is 128.48/362 ≈ 0.355 of the content width, and thumbSizes.ts has no step for it. Propose a new step rather than a hard-coded number, with a confidence score.
2. Listing card typography. Name: Nanza/itemTitle (Roobert SemiBold 15/16, ls 0.11, theme.colors.text). Number + tag line: Nanza/entityRowMeta (14/16, theme.colors.metaText). Figma reports entityRowMeta in "Balto Mobile" Book, but Roobert is the only font. Use the existing typography slot, flag the mismatch, and don't add a font. Layout: 16 from thumb to text; name starts 4 below the thumb top; line 1 is 6 below the name, line 2 is 2 below line 1.
3. Use "/" to separate multiple tags. Line 2 reads "Promotion Card / GEM Pack 4". EntityMetaLine draws a dot between segments and TAG_LINE_SEPARATOR is ' · ', and both are app-wide. Add an opt-in separator on EntityMetaLine / EntityLockupRow (metaLine2) and pass "/" from this card only. Do not change the global separator.
4. Alignment of price and quantity. The terms row is 52 tall and starts 16 below the thumb row. The price box is on the left at 106.5×52 with the amount centered. Count + stepper form a right-aligned cluster flush to the content edge, all vertically centered on the row. The count is 20 from the stepper.
5. Spacing around "Remove from offer". 16 below the terms row, left-aligned, Nanza/small (Roobert SemiBold 12/16). The gap below it isn't measurable (the redesign is overlaid on the scroll content), so ask before changing the gap to "Add more from this seller".
6. Remove the pink quantity indicator (orderCardQuantityBadge in OfferLineLockup).
7. Add a quantity selector: the glass stepper (two 44pt GlassRim circles, minus/plus, count between them with 16 each side). This already exists as OfferTermsRow variant="glass". Reuse it.

Coupled files:
- src/components/offers/OfferLineCard/index.tsx: the card; passes variant, separator, and a no-badge flag.
- src/components/offers/OfferLineLockup/index.tsx: thumb + badge. Shared with OfferMessageDetailScreen, so the badge removal and new thumb are opt-in props for this card unless confirmed otherwise.
- src/components/offers/OfferTermsRow/index.tsx: glass variant, row alignment, count label. Also used by ListingScreen and BidScreen, which must not change.
- src/components/entity/EntityLockupRow, EntityMetaLine: separator prop.
- src/styles/components/offers.ts: line*/terms* styles.
- src/components/cards/thumbSizes.ts: new width step.

Traps / decisions (raise them in the plan, don't guess):
- OfferTermsRow hides the stepper when available <= 1 and labels it "Only 1 available" ("N left" when the stepper shows). Figma shows a stepper with "1 available" at quantity 1. Confirm the visibility rule and the wording.
- The price box hugs its amount today (2026-09-16 decision, matches the detail screens). Figma shows a fixed 106.5 width (≈0.294 of content). Confirm fixed-on-this-card vs. keep the hug.
- The header comments in OfferLineLockup / OfferLineCard describe "the pink badge". Update them to match what ships.

House rules (CLAUDE.md): plan first with confidence scores per item; proportional sizing (fractions of the 402 reference / 362 content width); no inline styles, no raw sizes, theme tokens only. Run /ui-review before finishing. When done, set this handoff's status to built and add a Log line.
```

## Log
- 2026-10-23 — handoff written (first one; seeded the nanza-figma folder)
- 2026-10-23 — built in nanza-mobile (uncommitted; tsc clean, ui-review PASS). Reused the post card's `EntityHeroThumb` (the blurred-backdrop recipe Figma's layers came from) through new frame/backdrop/art style props instead of adding a thumbSizes step; new opt-in props `OfferLineLockup.heroThumb`, `OfferTermsRow.stepperAtOne`, `EntityMetaLine.separator`, `EntityLockupRow.metaSeparator` / `rowFloor`. Name→meta gap kept at the lockup's device-tuned 4 (Figma draws 6). Pending device QA.
- 2026-10-23 — QA: backdrop wasn't blurred (the BLUR-bucket upscale is only ~3× at this tile size). Listing photos now get RN blurRadius 16 (Figma's blur-[16px], same as the post's listing hero) through a new `EntityHeroThumb.backdropBlurRadius` prop; bid lines (entity catalogue scans) keep the upscale, because blurRadius paints blank on iOS for those.
- 2026-10-23 — same request re-sent without "do not change"; that defaults to "anything else on this screen", so the prompt is unchanged
- 2026-10-25 — partly superseded by [[2026-10-25-makeoffer-screen-restyle]] ("Remove" label, price box = thumb width, entity-card line thumb)
