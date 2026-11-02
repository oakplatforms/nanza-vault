---
tags: [nanza-mobile, plan, cards, entity, offers, collections]
status: plan — 2026-10-26, built same day (uncommitted, tsc + lint clean), pending device QA
date: 2026-10-26
---

# Card restyle — entity lockup, bid/listing tiles, collection cards, Make Offer

One plan for the four 2026-10-25 handoffs in `nanza-figma/handoffs/`:
[[../../nanza-figma/handoffs/2026-10-25-entity-card-styles|entity cards]] →
[[../../nanza-figma/handoffs/2026-10-25-bid-listing-tiles|bid/listing tiles]] →
[[../../nanza-figma/handoffs/2026-10-25-collection-card-styles|collection cards]] →
[[../../nanza-figma/handoffs/2026-10-25-makeoffer-screen-restyle|Make Offer]].
Skylar, 2026-10-26 (voice): "CSS and light layout changes, nothing around business logic."
`2026-10-24-entity-tabs-market` is already built; its tab bar already scrolls with the page
(`EntityScreen` 2026-10-25), so nothing to do there.

## Shared pieces (built once, worn by all four)

| Piece | Where | Confidence |
|---|---|---|
| **Landscape zoom thumb** on every entity lockup: `EntityHeroThumb` (posts) rendered by `EntityLockupRow` itself when it has an `image` and no custom `thumb`. Frame ≈1.535 (135.06×88), `neutral.950`→`background` fill, top hairline `border`, radius 10 (`radius['10']` added), backdrop ×1.6 at 0.61 offset −0.29w/−0.04h, top tint onWell 0.22→0 over 54.5%, 5:7 art at 83.4% height radius `min`. Sizes as fractions of the host's inner width (386 row / 362 page / 358 inline / 366 offer line). | `entity.ts` `lockupHero*`, `EntityLockupRow` | 0.85 |
| **Inset variant** `inset?: 'sm' \| 'page'` on `EntityLockupRow`: 8 (search, posts, offer line) vs 20 (trade rows). Thumb 135.06/386 vs 110.5/362. Row 20 top/bottom; `lockupRowFloor` re-derived from the new thumb height. | `entity.ts`, `TradeListItem` | 0.85 |
| **Backdrop blur only on listing photos** (`backdropBlurRadius` path, 1:1 with Figma = 3; was 16 on the offer line). Catalog art stays unblurred (iOS paints it blank). | `OfferLineLockup`, `ItemThumb` | 0.75 — alternative: keep 16 on the offer line; rejected, Figma says 3 everywhere |
| **Type slots** (typography.ts): `metaSm` Balto 12/14, `pillAmountSm` SemiBold 11/11, `pillLabelSm` Regular 9/10 (size `9` + lineHeight `10` tokens added), `tagPrice` Bold 11/11, `tileQuantity` Balto 11/11, `tileCount` SemiBold 10/11, `intentTag` Bold 10/11, `labelTight` Medium 13/14, `labelLgTight` Medium 14/14, `rowValue` Balto 13/22. | `typography.ts`, `dark.json` | 0.9 |
| **Market price pill** = `FairMarketPill` restyled (1 `glassBorder`, radius `xs`, padding 5/6/8, gap 4, baseline), `label` prop ("Market price" / "Market value"), `hidden` prop (eye-off 11×9 in place of the amount). One pill everywhere — DetailTitleBlock and ScanAddModal shrink with it. | `FairMarketPill`, `entity.ts` | 0.75 — alternative: scope the restyle to lockups only; rejected, one badge was the 2026-10-24 rule |
| **Price tag** (bid/listing/trade): `overlayText` fill, radius `xxs`, padding 4, `tagPrice` in `background`. Replaces the lockup price pills, the tile price pill and the Buy/Sell CTA pill. Quantity "N wanted / N in stock" `tileQuantity`, hidden ≤1. | `entity.ts` `lockupPricePill*`, `cards.ts` `thumbPricePill*` | 0.9 |
| **IntentTag colours swap app-wide**: WTB = `conditionColors.mint` + `tertiary.950`; WTS = `primary.400` + `primary.950`. Tile variant radius `xxs`, padding 3×4, `intentTag`. All variants. | `badges.ts`, `IntentTag` | 0.9 |
| **Grid spacing** 8 inset / **12 gap** on `CardGrid` (`cardGridRows` + the width math). Every grid host moves (feed, trades, profile collections, pickers). | `CardGrid`, `cards.ts` | 0.8 — alternative: a tile-only gap prop; rejected, Skylar: "use the Figma grid spacing" |

## 1. Entity lockup (`EntityLockupRow`)

- Name → `tileLabel` in `text`; meta lines → `metaSm` `metaText`, gaps 3 / 2 (`lockupIdentity` gap `min`+1, `lockupMetaGroup` gap `min`).
- Market price pill 8 (`sm`) under the meta group (`lockupBody` space-between keeps it there).
- **Right column** (new props `priceTag?: string`, `quantityLabel?: string`): price tag top-right, quantity 16 below. `TradeListItem` moves from `prices` (×N / $ / CTA pills) to `priceTag` + `quantityLabel`; `EntityCardBase`, `BulkSaleCard`, `SearchEntityRow` keep `prices` (now tag-styled).
- Search add badge: 24 circle `overlayText`, 9pt plus (`icons.xxs`).
- Posts: `PostContentItems` ENTITY card keeps the inline chrome; its `EntityHeroThumb` frame moves to the shared landscape size (135.06/358, 1.535) — `posts.ts entityHeroThumb*` re-derived, `entityHeroBody` kept.
- Offer line: `OfferLineLockup heroThumb` → thumb 122×80 (122/366, same 1.535 aspect? Figma 122/80 = 1.525; use the Figma pair), gap 16, name `tileLabel`, meta `metaSm`.
- `thumbSizes.ts`: no new step for lockups (they are percentages of the row); the tile step is below.

## 2. Bid / listing tiles (`ItemThumb`)

- **Square** tile, radius `lg` (20), 1.33 top hairline `glassBorder` (`borderTopWidth`), `background` fill. `aspect` prop accepted and ignored (no export removal).
- Backdrop: image `cover` in a 158%×102% box at −29%w / −1%h, opacity 0.61; blur 3 on listings with their own photo (`ListingThumb`/`BulkThumb` pass `backdropBlur` when `listing.image` is set). Art 5:7 at 53.3% of height, centred, top 11.2%, radius `xxs`. `ThumbShade` gets a `'deepBottom'` variant = `shadeBottomDeep` (0 → 0.8 @60% → 0.8) + the onWell top tint.
- Footer 12 inset / 11.5 bottom: name `tileLabel` `textSecondary` 2 lines tail, 6 gap, price tag + quantity (gap 8).
- `thumbSizes.ts`: `tile` step = 186/362, square; `ListingThumb`/`BidThumb`/`BulkThumb` default `size` → `'tile'` so carousels match grids. `card` step stays for any other consumer.
- `PostContentItems` LISTING/BID hero tile keeps its own recipe (not in scope); it only picks up the IntentTag colours and the price tag through the shared classes.

## 3. Collection card (`CollectionThumb`)

- Card ratio stays 189:240 (Figma 182×231 is the same ratio), radius `md`, 1 top hairline `glassBorder`.
- Count tag top-left 12/11.5: `surface` fill, radius `xxs`, padding 6×8, `tileCount` `secondaryText`.
- Footer 12 inset / 12 bottom: name `tileLabel` `textSecondary` 1 line, gap 6, then `FairMarketPill label="Market value" hidden={valueHidden}`; no data → "–". Replaces the value row + `MarketPriceLabel`.

## 4. Make Offer screen

- Line card: 8 inset, 20 top / 8 bottom, 24 bottom padding on the group, bottom hairline `surface`; lockup as §1 offer line. No pink badge (already gone on `heroThumb`).
- `OfferTermsRow` **new `variant="line"`** (opt-in; glass variant and ListingScreen/BidScreen untouched): price box = thumb width (122/366 of the row), 42 tall, `surface` fill, 1 `border`, radius `12`, `tileLabel` `textSecondary` centred, input inside; right cluster = **"Remove"** (`subtitle` 13/16 `text`, new `onRemove` prop) + stepper pill 93×40 `surface` radius `lg`, count `pill` `secondaryText`, minus at 0.35 opacity at the minimum, plus `icons.xxs`. The "N left" count text is not in the Figma row → dropped for this variant only (logic unchanged; the stepper still clamps to `available`). `OfferLineCard` drops its own "Remove from offer" link.
- "Add more from this seller": `Button variant="outline" size="xxl"` with offers.ts `addMoreButton` / `addMoreButtonText` overrides (1.7 `glassBorder`, radius `28`, `secondaryText`), 24 above/below, 20 inset.
- Checkout rows on the page (no surface card): padding 24×20, hairline `surface` (top on first, bottom on all), label `tileLabel` `text`, value `rowValue` `metaText` right + tail ellipsis, 12 gap, 12pt chevron. "Shipping services" → **"Shipping options"**.
- Expanded sections **standardised**: every panel = 20 inset, 24 bottom, options as `RadioField` rows (`rowValue` label, selected label `text`); the Shipping method tabs and the Shipping options pills become radio rows (multi-select stays a toggle on the options row — logic unchanged). **`RadioField` restyle app-wide**: 20 circle, unselected 1 `secondaryText` ring, selected `text` ring + `text` dot (no pink). Only other consumer: ReportPostScreen.
- Summary: `body` rows 26 tall gap 2, label `metaText`, amount `text`; total `bold` 28 tall.
- Agree row: 20 above/below a `surface` hairline, 16 box radius `min`, unchecked 1 `secondaryText`, checked `textSecondary` fill + `text` stroke + `background` check; label `labelTight` `secondaryText`, link `text`.
- "The seller has three days…": 30 / 10 / 20, centred `labelLgTight` `inactive`. Copy kept.
- Submit/Send Offer **fixed footer**: `EdgeFade` (160 = `xxl`×2, `fadeActionBar`) + the xxl action Button at the 20 inset; scroll content bottom padding clears it. **Shared inactive style**: `buttonXxlActionDisabled` → `surface` fill, label `border` colour (OfferMessageDetailScreen's disabled Buy/Sell Now follows). Active pill unchanged.

## Escalation (CLAUDE.md)

Over 5 files, shared Button + RadioField + CardGrid changes — approved by Skylar by voice on
2026-10-26 before building. Files: entity.ts, EntityLockupRow, EntityHeroThumb (reuse only),
FairMarketPill, TradeListItem, SearchEntityRow (+searchEntityRow.ts), OfferLineLockup,
posts.ts, typography.ts, dark.json/light.json, cards.ts, ItemThumb (+ThumbShade), ListingThumb,
BidThumb, BulkThumb, thumbSizes.ts, CardGrid, IntentTag + badges.ts, CollectionThumb,
offers.ts, OfferLineCard, OfferTermsRow, OfferCheckoutRows, OfferPaymentPanel,
OfferFeePayerPanel, RadioField + inputs.ts, buttons.ts, MakeOfferScreen.

## Log
- 2026-10-26 — plan written from the four handoffs (voice session).
- 2026-10-26 — built in full (32 files); the four handoffs set to built. Deviations from the mocks: Submit stays the 56 xxl (Figma 52); the offer line's thumb takes the shared 1.535 aspect (Figma 122/80 = 1.525); the make-offer line row drops the "N left" count text (not in the mock; the stepper still clamps).
- 2026-10-26 (device QA, voice) — every card slot one point over Figma (names on `bold` 14/16, meta 13/15, tag/quantity 12, pill 12 + 10, intent 11); card art 90% of the row thumb and 62% of the tile; backdrops anchored on the ART band (`getZoomBackdropStyle(frameAspect)`, art centre 0.35, the posts.ts rule) so the card's text box stays out of the window; listing photos keep a centred backdrop (`getZoomBackdropPhotoStyle`). Open: whether listing tiles should show the photo alone with no card backdrop (Skylar to check with his partner).
- 2026-10-26 (device QA 2, typed) — WTB tile art 62% → 58%; the white price tag pads 5 above / 3 below the digits (Roobert seats high; container fix, no nudge); **listing photos fill the tile and the row thumb plain** (`ItemThumb photo`, `EntityLockupRow photo`) — no zoom backdrop, no floating card, blur retired; catalogue art keeps the hero recipe. Answers the open listing question.
- 2026-10-26 (device QA 3, voice) — IntentTag colour swap reverted: WTS mint / WTB pink as before, new dress kept.
- 2026-10-26 (device QA 4, typed) — listing photos `contain` (zoomed out) on tiles and row thumbs; Trade screen: My trades is a glass pill (`tradeFooterGlass` + GlassRim, label "My trades"), the footer sits on `EdgeFade shadeBottomDeep`, the bottom nav backdrop is the same `shadeBottomDeep` token (FloatingTabBar), and the All · WTS · WTB capsule gets 12 under it (`tradeTabsSection`, Trade only). Container width left as is (shared container).
- 2026-10-26 (device QA 5, typed) — listing photo zoom-out (`contain`) reverted to cover on tiles and row thumbs, "for now".
- 2026-10-26 (device QA 6, typed) — TradeListItem back on the default 8 inset / 135×88 thumb so My trades rows align with the search rows (image, title, right column); the `inset="page"` variant stays available but unused.
- 2026-10-26 (device QA 7, typed) — trade search (CardPickerLayer → SearchEntityRow) shows an existing trade as the My trades right column (`priceTag` + `quantityLabel`) instead of the ×N / $ pills; list-row thumb at 95% of the Figma tile (`THUMB_SCALE`, ≈128×84), offer line and posts unchanged.
- 2026-10-26 (device QA 8, typed) — WTS tiles take the listing detail hero's recipe: the photo blurred (24) edge to edge behind, the photo itself `contain`ed in a frame (8% top / 6% sides / 30% bottom) over it, shade on top. Row thumbs (offer line) still cover.
- 2026-10-26 (device QA 9, typed) — WTS photo frame back to the full tile height (`contain` over the blurred backdrop, no inset): only the blur's edges show beside a portrait shot.
- 2026-10-26 (device QA 10, typed) — full-height photo frame dropped; back to the inset frame, a tad bigger (6% top / 4% sides / 26% bottom).
- 2026-10-26 (device QA 11, typed) — freshness: every listing / bid write that invalidates `['Trades']` now also invalidates `['collectionNanzaSearch']` and `['collectionDetail']`, the trade search picker's two queries, so an edited price / quantity shows on the Nanza and My Library rows on return (lazy, refetch-on-focus).
- 2026-10-26 (device QA 12, typed) — WTS listing photo: full bleed, no frame, no blurred backdrop (final). The hero-style frame classes are removed.
- 2026-10-26 (device QA 13, voice) — QuantityModal (update quantity) wears the ScanAddModal card (image, name, number • print • set via the shared `entityCardSubtitle`, Market price pill) over the glass stepper; the scan result strip's bare price row is the outlined FairMarketPill.
- 2026-10-26 (typed) — UI copy sweep: no "bid" / "listing" in user-facing strings; "item" (or WTS / WTB items where the kind matters) across the seller gate, own-item sheets, action menu (Share / Edit / View / Remove item), cap sheets, delete dialogs, group leave, vacation note, post composer cap, ELB picker tabs (WTS / WTB) + placeholders, OfferListingRow ("Your item"), not-found screens, Badge "Your price", offer copy ("your price"), camera permission, lot dialogs. Identifiers, query keys, route names and legal documents untouched.
- 2026-10-26 (typed) — copy sweep 2: entity Trades tab body "Buy or sell this card on Nanza." → "Add a WTS or WTB for this card."; the create / edit form alerts say "post a WTS / WTB", "update your WTS / WTB", "update your item".
- 2026-10-26 (typed) — affiliate rail Short badge: glyph is the new `assets/icons/play.svg` (iconMap `play`) instead of `play-solid`; disc, size and colour states unchanged.
- 2026-10-26 (typed) — Short badge shape: YouTube-style rounded rectangle (18×26 on the 88 circle, radius 30% of height) instead of the disc.
- 2026-10-26 (typed) — Short badge corners: radius 38% of height with `borderCurve: 'continuous'` (the YouTube red shape); 26×18 proportions kept.
- 2026-10-26 (typed) — Short badge 20×29 (from 18×26); the triangle is centred on its centroid (badge paddingLeft = glyph / 3), not its box.
- 2026-10-26 (typed) — Trades tab body: "Trade this card with other collectors." (the WTS / WTB line was rejected too).
- 2026-10-26 (typed) — Trades tab body settled: "Start a trade for this card."
- 2026-10-26 (device QA, typed) — Make Offer: Submit back IN the flow directly under the expiry note (`submitRow`); the fixed footer + EdgeFade from the mock are out ("too far down"). Scroll padding back to xxl.
- 2026-10-26 (typed) — Make Offer: "Add more from this seller" renders only when the seller has another in-stock ACTIVE listing not already on the offer (`useSellerOfferableListings` + `offerableListings`, shared with the picker on one query key).
- 2026-10-26 (typed) — offers: tapping a line's thumb opens the shared ImageViewer (`OfferLineLockup zoomOnPress`) on Make Offer and the offer detail; the detail's thumb no longer pushes the Entity screen.
- 2026-10-26 (typed) — inbox WTS offer (OfferListingCard): the seller's photo opens in the ImageViewer on tap (`contain`).
- 2026-10-26 (typed) — fee payer wording after send: "You will pay" / "They will pay" (checkout row read-only + the offer message's fee sentence); "I’ll pay" / "You’ll pay" stay for composing.
- 2026-10-26 (typed) — the offer message's fee sentence now appears on every non-cash offer (buyer-pays included), not only when the seller covers it.
- 2026-10-26 (typed) — fee wording split: the offer MESSAGE keeps the sender's voice ("I’ll / You’ll / They’ll pay the processing fee.", on every non-cash offer); the Processing fee ROW on a sent offer reads "You will pay" / "They will pay".
- 2026-10-26 (typed) — offer detail nav title by state (Accepted / Declined / Canceled / Expired offer; "Offer" while pending); offer summary row "Estimated tax" → "Sales tax" (the only occurrence across mobile, email, web, api).
- 2026-10-26 (typed) — checkout row panels: 8 top padding when expanded (Make Offer + offer detail share `panelContent`).
- 2026-10-26 (voice) — WTS photo tiles: the deep bottom shade (solid 0.8 over the bottom 40%) crushed the full-bleed photo, which sits under the shade where catalogue art floats above it. `ThumbShade` 'tile' takes a `photo` flag (ItemThumb passes it): photo tiles get the plain `shadeBottom` colours as a ramp from 45% to overlay 0.7 at the very bottom; catalogue tiles keep `shadeBottomDeep`. Same session (nanza-api): `GET /trades` spreads by owner so one seller's listings no longer fill the grid.
- 2026-10-26 (device QA, typed) — photo-tile shade a tad stronger at the bottom: ramp from 42% to the 0.7 overlay at 88%, held to the edge (`PHOTO_FADE_FULL`). Catalogue tiles unchanged.
- 2026-10-26 (device QA, typed) — WTS photo-tile shade, bottom only: 42% clear → overlay 0.7 at 84% → overlayDeep 0.8 at the edge (the deep token's terminal). Start unchanged.
- 2026-10-26 (device QA, typed) — "N in stock" / "N wanted" (`typography.tileQuantity`) one shade lighter: metaText (neutral 300) → secondaryText (neutral 200). The row lockups share the token and move with it.
- 2026-10-26 (voice) — "Add from this seller" picker: the chevron + title sometimes opened above the safe area and jumped as items loaded. Cause: the global `Header` padded its top with the native `SafeAreaView`, which re-measures inside a RN `<Modal>` as it presents. `Header` now pads by the provider's `insets.top` (same geometry, `headerHeight` already carried it; the hook value is stable), and the picker's Modal takes `statusBarTranslucent` like `InAppBrowserSheet` so Android spans the full height too. Header change is global; every screen keeps the same height and padding math.
