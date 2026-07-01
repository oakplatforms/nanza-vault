# OG Shareable Template — Styling Improvements

Scope: `src/services/og/templates/` (nanza-api). Server-side OG image renderer (satori → sharp).
Note: this service uses hardcoded hex in a local `COLORS` map by existing convention — it is
NOT the RN app, so the mobile CLAUDE.md theme-token rules do not apply here.

## Requested changes (FINAL spec — confirmed by voice)

NOTE: WTS/WTB label was considered then **dropped** — do NOT add a status pill.

1. **Bigger hero image (right side)** — currently `360 × 502`. Make it bigger so it uses more of the space.
2. **Tighten + balance padding** — outer card padding is `72 (v) / 80 (h)`; inter-element `marginTop: 40`
   gaps. Reduce so the card uses more of the frame, but keep a nice EQUAL padding all around (not too much).
3. **nanza logo at the bottom** — footer wordmark on the **listing** and **bid** cards only (for now).
   Source: `nanza-mobile/assets/images/logo_full.svg` (viewBox 704×200, icon + "nanza" wordmark).
   Inline as a base64 SVG data URI (same pattern as `product.ts` FairMarket / `collection.ts` logo-icon).
   Placement: **underneath the price**, with generous spacing — not too close to the price, not too close
   to the bottom (roughly centered in the gap below the price).
4. **Font already swapped to Roobert** ✅ (done this session).

### Typography spec (exact, per voice)

| Element                         | Font weight        | Size  | Letter-spacing | Color |
|---------------------------------|--------------------|-------|----------------|-------|
| Title / header (entity name)    | Roobert SemiBold 600 | 60px | 0.11px       | white |
| Second line (product # / subCode) | Roobert Regular 400 | 40px | 0.11px      | white |
| Tags + condition pills          | Roobert SemiBold 600 | ~36px | —            | (pill defaults) |
| Price                           | Roobert SemiBold 600 | 40px | —             | (price pill) |

- Current values being changed: title `700 / 72px` → `600 / 60px + 0.11 ls`; subCode `600 / 39px` (muted)
  → `400 / 40px + 0.11 ls / WHITE`; pills `600 / 35px` → `600 / 36px`; price `700 / 52px` → `600 / 40px`.
- Price "$" stays Figtree (brand rule) but now one weight above 600 = **700** at the new 40px size.

## Affected files

- `src/services/og/templates/listing.ts` — hero size, paddings/margins, WTS pill, footer logo.
  This is the canonical home for shared primitives, so a new `statusPill()` helper + the inlined
  `NANZA_LOGO_SVG` data URI + a `footerLogo()` helper live here and are exported for `bid.ts`.
- `src/services/og/templates/bid.ts` — consumes the shared helpers: WTB pill + footer logo.

Product & collection templates are intentionally NOT touched yet (user: "for now, just listing + bid").

## Open questions to confirm before coding

- **Exact label text**: "WTS" / "WTB" (initialisms) vs "Want to Sell" / "Want to Buy" (full)?
- **WTS/WTB pill color**: a distinct accent (e.g. the pink `#FF6DFA` already used for FairMarket) so
  it stands apart from the dark variant pill and green condition pill — or match an existing pill style?
- **Hero target size**: bump `360×502` to roughly `420×586` (same 0.717 aspect), or larger/full-bleed-ish?
- **How much to tighten padding**: outer `72/80` → ~`56/64`, gaps `40` → ~`28`? (proposed defaults)

## Notes / constraints

- Adding a footer logo means the left column is no longer free-floating top content — the card body
  needs a `justifyContent: 'space-between'` column (content top, logo bottom) OR an absolutely
  positioned footer. Will use a bottom-anchored footer row spanning the card so the logo sits
  consistently regardless of how much pill/price content is above it.
- Satori rasterizes inline SVG data URIs fine (only WebP sources fail) — confirmed by existing
  FairMarket/logo-icon usage.
- Bigger hero may collide with reduced padding; will verify the `502→larger` height still fits inside
  `630 − (top+bottom padding) − footer logo height`.
