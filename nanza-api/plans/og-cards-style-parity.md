# OG Share Cards — Style Parity (product, lot, collection)

Bring the **product**, **lot (bulk)**, and **collection** OG share-image templates up to the
same visual standard already set by the **listing** and **bid** cards. Everything renders via
Satori → sharp in `src/services/og/`. The listing/bid cards are the canonical reference.

## Reference (already done): listing & bid
- Outer padding: **56** on all sides (one global wrapper).
- Title: Roobert **600**, white, `fontSize 62`, single line, pixel ellipsis (`whiteSpace:nowrap`
  + `overflow:hidden` + `textOverflow:ellipsis`).
- Subcode: Roobert **400**, **white**, `fontSize 42`.
- Hero image: **400 × 518**, `objectFit:contain`, `borderRadius:18`.
- nanza wordmark anchored bottom-left.

## 1. Product card (`templates/product.ts`)
Currently fights the listing styling: title is 700 (vs 600), subcode is muted gray (vs white),
hero is smaller (360×502), outer padding is 72/80 (vs 56).

Changes:
- Outer padding → **56** all sides (match listing) — one consistent global wrapper.
- Hero → **400 × 518**, `objectFit:contain`, radius 18 (match listing image style; bigger).
- Title → weight **600**, white, single-line **pixel ellipsis** (drop `clampTitle` char-clamp,
  use the listing's nowrap/overflow/textOverflow approach). Keep the larger product title size
  but bring weight in line — **fontSize 62, weight 600** to match listing exactly.
- Subcode (product number) → weight **400**, **white**, fontSize 42 (match listing subcode).
- Fair-Market block: keep on the right under the name; nudge the price typography so it reads
  consistent. Sits a comfortable gap below the name block.
- No nanza logo on product (intentional — confirmed).

## 2. Lot / bulk card (`templates/bulk.ts`)
The fan currently splays all 3 cards to the right with growing rotation. Title is 700 + char
clamp. Outer padding 72/80. No global image padding consistency.

Changes:
- Outer padding → **56** all sides (match listing).
- Title → weight **600**, white, single-line pixel ellipsis (match listing; drop `clampTitle`).
- Price pill stays (already shared `pricePill`).
- Fan: **front card centered**, the other two **peek out only at the edges** behind it — one
  tilted slightly left, one tilted slightly right (symmetric, subtle — just the edges show, not
  a wide splay). Make the cards **bigger** than now. No per-card padding — the global outer
  padding is the only padding.

## 3. Collection card (`templates/collection.ts`)
Full-bleed mosaic is good. One addition:
- Add a **deep but short drop-shadow gradient** along the **bottom** of the grid (transparent →
  dark), tall enough to seat the logo legibly but not dominating the mosaic.
- Place the **nanza wordmark** on top of that shadow, **bottom-left**, at the same inset the
  listing/bid footer logo uses.

## Verification
Local preview harness (`.og-preview/preview.ts`, satori → SVG → qlmanage PNG, placeholder
heroes) renders product/bulk/collection/listing side by side to eyeball parity. `npm run lint`
+ `tsc` clean. No new deps. `.og-preview/` is scratch (gitignored / removed before commit).
