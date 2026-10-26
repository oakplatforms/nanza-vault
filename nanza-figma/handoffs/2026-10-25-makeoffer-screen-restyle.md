---
tags: [nanza-figma, handoff, nanza-mobile, offers]
status: open
date: 2026-10-25
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=2995-147203
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 2995:147203
frames:
  - 2836:103033 — single line, Submit inactive (grey)
  - 2995:147203 — two lines, Submit active, checkbox ticked
frame: MakeOffer / default · Seller · edit me
repo: nanza-mobile
---

# Make Offer — screen restyle

Follows [[2026-10-23-makeoffer-listing-card]] (same screen, built). Where the two conflict,
this handoff wins: the "Remove" label, the price-box width and the line thumb.

## Request (verbatim)
**Changed:**
- The bid/listing cards should reflect our updates in the entity card styles handoff
- The price and quantity styles update (dont change any logic)
- Remove from offer changes to just Remove
- The add more from seller button is updated
- The collapsible shipping options are all updated (try to standardize the options in each collapsed sections so that its uniform and matches the styles of the rest of the page as far as typography, sizing, spacing, color and avoid using pink for any selected radios etc.)
- Same in the checkbox for I agree, no pink just white fill)
- Submit offer is fixed and grayed out (this is the style moving forward for all inactive action buttons)
- Add spacing around 'seller has… to accept…'

**Do not change:** any logic

## Measurements
All values are at the 402 frame.

| Element | Spec |
|---|---|
| Line card | The entity-card lockup ([[2026-10-25-entity-card-styles]]), **8 side inset**, 20 top / 8 bottom. Thumb 122×80, gap 16, name `tileLabel` in `text`, meta Balto 12/14 `metaText` (gaps 3 and 2), set line with scroll fade. No pink badge. Line group: 24 bottom padding, bottom hairline `surface`. |
| Price box | Left, **same width as the thumb** (122), 42 tall. `surface` fill, 1 `border` stroke, radius 12, padding 12×20, amount centered. SemiBold 13 `textSecondary` ("$" + amount). |
| Remove + stepper | Right cluster, gap 12, vertically centered with the price box. **"Remove"** Regular 13/16 `text`. Stepper pill 93×40, `surface` fill, radius 20: minus at 0.35 opacity when at minimum, count SemiBold 13/14 `secondaryText` centered, plus 9pt. Behaviour is unchanged. |
| Add more from this seller | 24 padding above and below, 20 inset. Outline button, full width (362), about 57 tall: transparent fill, **1.7 `glassBorder` stroke**, radius 28, padding about 19×30. SemiBold 16/16 `secondaryText`. |
| Checkout rows | Shipping / Shipping method / **Shipping options** / Payment / Processing fee. Each row: padding 24×20, hairline `surface` (top on the first row, bottom on all). Label SemiBold 13/16 `text`. Value Balto 13/22 `metaText`, right-aligned, single line with tail ellipsis; gap 12 before the 12pt chevron box (7×12 glyph). |
| Expanded sections | Not drawn. **Standardize** every section: same padding/insets as the rows, type = row value style (Balto 13/22) for options plus `text` for the selected label, no new sizes, **no pink**. Selected radio = white (`text`) fill/dot, unselected = `secondaryText` outline. |
| Summary | `body` Balto 14/22: label `metaText`, amount `text`, rows 26 tall with gap 2. Total row `bold` SemiBold 14/16 `text`, 28 tall. |
| Agree checkbox | 20 above and below a full-width `surface` hairline, then the row at the 20 inset, gap 12. Box 16, radius 4. **Unchecked:** 1 `secondaryText` stroke. **Checked:** `textSecondary` (white) fill, 1 `text` stroke, dark check, **no pink**. Label Medium 13/14 `secondaryText`, link part in `text`. |
| "The seller has … to accept" | **30 above, 10 below** (padding 30 / 10 / 20 sides), centered, Medium 14/14 `inactive`. **Keep the app's copy** ("three days"). |
| Submit Offer | **Fixed** at the bottom over an `EdgeFade` (162 tall, `fadeActionBar` gradient). Button 362×52, 20 inset, radius 28, padding 18. **Inactive:** `surface` fill, label `border` colour. **Active:** `textSecondary` fill, label `background`. Label SemiBold 16/16. The inactive look becomes the **shared Button style for all inactive xxl action buttons**. |

## Code map
- `src/screens/offers/MakeOfferScreen/index.tsx`:
  - Submit (inline today, `inactive={!listingReady}`)
  - the checkbox (`styles.checkbox` / `checkboxChecked`)
  - the three-days copy (lines 346-347)
  - "Add more from this seller" (line 399)
- `src/components/offers/OfferLineCard`, `OfferLineLockup` — line card. `OfferLineLockup` is also used by `OfferMessageDetailScreen`, so keep the change opt-in. "Remove from offer" is at `OfferLineCard` line 73.
- `src/components/offers/OfferTermsRow` — price box + stepper (`variant="glass"`). Also used by `ListingScreen` / `BidScreen`, which must not change.
- `src/components/offers/OfferCheckoutRows` — the collapsible rows (`expanded` state; "Shipping services" becomes "Shipping options").
- `src/components/offers/OfferPaymentPanel` and `src/components/global/Form/RadioField` — radios (pink today).
- `src/components/offers/OfferSummary`, `OfferCheckoutSummary` — summary.
- `src/components/global/Button` + `src/styles/components/buttons.ts` (`disabled`, the `inactive` / `actionUnavailable` path) — shared inactive style.
- `src/components/global/EdgeFade` — fixed footer fade (JoinChatFooter / entity screen pattern).
- `src/styles/components/offers.ts` — `line*` / `terms*` / `navRow*` / `summary*` classes.

## Open questions
1. Figma says "72 hours", the app says "three days". Change it?
   **Answer (Skylar, 2026-10-25):** keep the app's text.
2. Figma says "Shipping options", the app says "Shipping services". Rename it?
   **Answer:** use **Shipping options**.
3. The second line card in `2995:147203` shows a lone dimmed-minus circle instead of "Remove".
   **Answer:** a Figma error. Every line uses the "Remove" text, as in `2836:103033`.
4. Gray inactive button: shared Button or offers only?
   **Answer:** update the **shared Button** style.
5. Price box width: still hugging its amount (2026-10-23 decision)?
   **Answer:** no. It's **the same width as the image thumb**.

## Prompt
```text
Implement the Make Offer screen restyle from Figma. Plan first, then build. Don't change any
logic. Build after the entity-card handoff (2026-10-25-entity-card-styles), whose lockup the
line cards reuse.

Figma (file 7c3Zz4nDUOutQG1BsnRRAw):
- 2836:103033: one line, Submit INACTIVE (grey), checkbox unticked
- 2995:147203: two lines, Submit ACTIVE, checkbox ticked
Read them with get_metadata / get_design_context. Read this handoff file first
(nanza-vault/nanza-figma/handoffs/2026-10-25-makeoffer-screen-restyle.md). Its Measurements
and answered Open questions are the spec. It supersedes the 2026-10-23 handoff on "Remove",
price-box width and the line thumb. Your branch may be ahead of the mapped snapshot. Wire to
what exists there, and STOP and ask if something is missing.

SCOPE: the Make Offer screen's styling plus the shared inactive Button style. Logic, totals and
copy (other than "Remove" and "Shipping options") are unchanged. Artifacts to ignore: the
second line's lone minus circle (Figma error, use "Remove"), layer names "bold · Shipping"
on the Remove label and "subtitle · I already have an account" on the checkbox/notice text,
the "72 hours" copy (keep "three days"), hidden termsPriceBox / RowContent / IntentTag layers.

1. The bid/listing cards should reflect our updates in the entity card styles handoff. Line
   card = the new entity lockup at the 8 inset (thumb 122×80, gap 16, tileLabel name, Balto
   12/14 meta). No pink badge. Opt-in on OfferLineLockup (OfferMessageDetailScreen shares it).
2. The price and quantity styles update (don't change any logic). Price box = thumb width
   (122), 42 tall, surface fill, 1 border stroke, radius 12, centred SemiBold 13
   textSecondary. Stepper 93×40 surface pill, radius 20, count SemiBold 13 secondaryText,
   minus at 0.35 opacity at minimum. Scope it so ListingScreen / BidScreen (OfferTermsRow)
   don't change.
3. Remove from offer changes to just Remove. "Remove", Regular 13/16 `text`, 12 left of the
   stepper, same handler.
4. The add more from seller button is updated. Full-width outline: transparent, 1.7
   glassBorder, radius 28, about 57 tall, SemiBold 16 secondaryText. 24 above and below.
5. The collapsible shipping options are all updated. Rows: padding 24×20, surface hairlines,
   label SemiBold 13/16 text, value Balto 13/22 metaText right-aligned with ellipsis, 12 gap,
   12pt chevron. Rename "Shipping services" → "Shipping options". STANDARDIZE every expanded
   section (address, method, options, payment, processing fee) to one recipe: the same insets,
   the same option type (Balto 13/22; selected label `text`), the same spacing, NO PINK. Selected
   radio = white (`text`) fill, unselected = secondaryText outline. Fix it in RadioField or
   an opt-in variant (it's shared, so check other forms in the plan).
6. Same in the checkbox for I agree, no pink, just white fill. Checked = textSecondary fill
   + text stroke + dark check; unchecked = secondaryText stroke. 16 box, radius 4.
7. Submit offer is fixed and grayed out. Move Submit to a fixed footer over EdgeFade (162,
   fadeActionBar), 362×52 at the 20 inset, radius 28. Inactive = surface fill + `border`
   label. Active = textSecondary fill + background label. Make the inactive look the SHARED
   Button style for all inactive xxl action buttons (buttons.ts actionUnavailable /
   disabled path), and list the other screens that change in the plan. Pad the scroll
   content so the last row clears the footer.
8. Add spacing around 'seller has… to accept…'. 30 above, 10 below, 20 sides, centred
   Medium 14 inactive. Keep the app's copy.

Coupled files: MakeOfferScreen, OfferLineCard, OfferLineLockup, OfferTermsRow, OfferCheckoutRows,
OfferPaymentPanel, Form/RadioField, Button + buttons.ts, offers.ts. Over 5 files and a shared
Button change: plan for the CLAUDE.md escalation.

House rules (CLAUDE.md): plan first with confidence scores; proportional sizing (122 =
thumb width, a fraction of 386); no inline styles, no raw sizes, theme tokens and typography
slots only. Run /ui-review before finishing. When done, set this handoff's status to built,
add a Log line, and add a "superseded by" note to the 2026-10-23 handoff's Log.
```

## Log
- 2026-10-25 — handoff written; Q1–5 answered by Skylar
