---
tags: [nanza-mobile, plan, detail-screens]
status: IMPLEMENTED (bid 2026-09-11, listing 2026-09-15) — pending on-device QA; fold into detail-screens.md, then delete
---

# BidScreen glass sheet (Figma 1213:213817) — Plan

Bring `BidScreen` 1:1 with Figma `7c3Zz4nDUOutQG1BsnRRAw` node `1213:213817`
("DetailScreen-Bid / default · Seller · edit me"). Fold into [[detail-screens]] when it lands.

## Decisions taken with the user (2026-09-11)

- **Scope: bid only, opt-in.** Every new look sits behind a prop only `BidScreen` passes.
  `ListingScreen` (whose block is line-for-line the same), `EntityScreen`, the share cards and
  the offer screen do not move. Listing adopts later by passing the same props.
- **Message button stays** (not in the mock): a 44pt glass circle after the WTB tag,
  layout-neutral so the row keeps the mock's 24pt rhythm.
- **Follow the mock on:** title colour → `textSecondary`; WTB chip → `highlight` pink with
  4/2 padding and 4pt corners; CTA bar → 12pt side inset, 52pt pill (**app-wide** — the bar is
  shared by 16 screens).
- **Keep the app's value on:** count copy ("N left" / "Only N wanted", Skylar 2026-08-29 — the
  mock's "2 available" is placeholder copy for a bid).

## Where the mock is stale (code wins, no change)

- **Heart in the header** — saving was retired 2026-08-29 and `SavedItemsContext` no longer
  exists. Header stays Edit (owner) + Share.
- **`entityRowMeta` 13pt** — the app reverted to 14 on 2026-09-05 ("13 read too small").
- **`sectionTitle` 18pt** (Chat heading) — the app is 20/24 since 661:107557.
- **FairMarketPill's 1pt padding/gap deltas** — it is THE badge; its dress lives in `entity.ts`.
- ~~**CTA label face** (mock SemiBold, app `title` Bold)~~ — reversed 2026-09-15: every xxl button
  label is SemiBold now (see "Unavailable CTA + SemiBold xxl labels" below).

## Figma → code map

| Mock element | Code |
|---|---|
| Info button | removed; the title row becomes the entity link (`openEntity`, existing) |
| `DetailInfoRow` w/ "Near Mint" glass pill, in-flow, 48pt | `DetailConditionSection variant="glass"` — heading dropped, glass pills (fill + `<GlassRim pill>`), 32pt tall, 15/8 inset; the 48pt slot is kept even with no conditions so the title never rises into the fade |
| Title + chevron (gap 6, pt 4) | `DetailTitleBlock onTitlePress` → touchable row, jumbo in `textSecondary`, `chevron-right` at `icons.sm` in `metaText` (renders ~9×16, 2pt weight = the mock's stroke), 4pt name→meta gap |
| FairMarketPill 16 under the meta lines | `DetailTitleBlock fairMarketPrice` (`entity.product.price`, formatted; `null` → dash, `undefined` → hidden) |
| User row: 24 avatar, 8 gap, SemiBold 15 name, WTB chip | `UserActionRow variant="glass"` + `IntentTag variant="compact"` |
| 1px `surface` rules | `detail.ts sheetRule` |
| Price box (inputBackground, inputBorder, r12, 52 tall) | `OfferTermsRow variant="glass"` — editable, "$" + input in a new `priceInput` slot (SemiBold 16, no lineHeight) |
| Count + two 44pt glass circles, value `title` | same variant — glass-circle stepper rendered in OfferTermsRow; icons `textSecondary` / disabled `inactive` (colour, not opacity) |
| Section gaps 24 / 20 | `sheetBodyTight` (24) around [top block, rule, chat]; `detailLockupStack` (20) around [title, user row, rule, terms] |
| Hero: 4× blurred card copy at 61% behind the card | `DetailHero cardBackdrop` — flex-centred `Image` at 250% width, card aspect, opacity 0.61, small `blurRadius` (tune on device) |
| CTA 378 wide, 52 tall | `actionBarGradient` inset → `gutter`; bar-only `minHeight` override → 52; `getActionBarGeometry.cta` follows so screen clearance stays exact |

## Files

`BidScreen/index.tsx` · `DetailTitleBlock` · `DetailConditionSection` · `UserActionRow` ·
`DetailHero` · `DetailActionBar` · `OfferTermsRow` · `IntentTag` · `detail.ts` · `offers.ts` ·
`badges.ts` · `typography.ts` (new `priceInput` slot). 12 edits, no new files, ~420 lines
(mostly style classes).

## Risks to QA

- TextInput hug width (min-width fallback if a platform won't auto-size).
- Two-line titles: the chevron centres on the block rather than trailing the last line.
- Bid payload may omit `entity.product.price` → pill shows the dash.
- Backdrop blur radius needs a device look.
- The CTA change is visible on all 16 bar screens.

## Listing adoption (2026-09-15)

`ListingScreen` (singles and lots) now passes the same opt-in props, so the two detail sheets
read identically. **Kept listing-specific:** the photo hero (no `cardBackdrop`), the `color` tag in
the meta line, the `useShareActions` CTA / lot buy logic, and the lot pager.

Decisions taken with the user:

- **Lot pager stays where it floated** — `DetailInfoRow` now renders for lots only, pager-only
  (no `onInfoPress`). Its line sits ~16pt above the in-flow glass condition row, so no collision.
- **Lot cover keeps the linked title colour** — the cover has no entity (no chevron, no link), but a
  new opt-in `DetailTitleBlock linkedTone` gives it the linked lockup's `textSecondary` + 4pt meta
  gap, so the title doesn't flip to `overlayText` while swiping cover ↔ child. The cover hides the
  fair-market pill (`undefined`) and keeps the empty 48pt condition slot.
- **WTS compact chip stays mint** — only WTB switched to `highlight` pink in the compact variant.
- Also picked up BidScreen's iOS `onContentSizeChange` composer re-park.

Wrap-up cleanup candidates (verify zero consumers first): `detail.ts` `offerSection` / `offerRule` /
`offerTermsSection`, `UserActionRow`'s `compact` branch + `userRowCompact` / `userAvatarCompact` /
`usernameCompact`, and `DetailInfoRow`'s info button + `sheetInfoButton`.

Files: `ListingScreen/index.tsx` · `DetailTitleBlock` (`linkedTone`) · `OfferTermsRow` (comment).

## Message button → header (2026-09-15)

The envelope left the glass user row on both screens and now sits in the floating header as a
`darkCircle` (default size) between owner-only Edit and Share — the `UserProfileScreen` header
pattern. The screens drive it with `useProfileConnect` (same gating as before: hidden for guests,
self, blocked; chat when connected, pending notice until accepted, friend request otherwise).
`UserActionRow variant="glass"` renders no envelope and skips its connections fetch; the row's
44pt `userRowGlass` box is unchanged, so no layout shift. Share cards (default variant) keep theirs.

Wrap-up cleanup candidate: `detail.ts` `userMessageGlass` (zero consumers).

## Unavailable CTA + SemiBold xxl labels (2026-09-15, Figma 1557:130138)

**Unavailable look.** The xxl `action` pill (the app's CTA) has one look when it can't be pressed —
`disabled` *or* gated `inactive`: `ink` fill + `textDisabled` label at full opacity (the mock's
"Submit Offer" in the Make an Offer sheet, which is `inactive={!listingReady}`). It lives in `Button`
itself (`buttonXxlActionDisabled` / `…Text`, laid over the caller's `style` / `textStyle`), so every
surface gets it: `DetailActionBar` (all 17 screens — Sold, out-of-range terms, the 12 gated forms),
MakeOffer, CollectTab, ScanItems, share cards, OfferMessageDetail, BulkTab. It replaces the old 50%
dim (`disabled`) and 4% ghost (`inactive`); other variants/sizes keep those. **Busy is not
unavailable**: `loading` / `success` keep the active pill under the arc / check (decision: busy was
dimming because screens pass `disabled: isWorking`). `DetailActionBar` now forwards `inactive` to
`Button` instead of ghosting its slot.

**SemiBold labels** (user, from the designs: every xxl button is SemiBold). New typography slot
`buttonLabel` — SemiBold 16/16 +0.11, `title`'s metrics one face down — used by `buttonXxlText`,
`detail.ts ctaDarkText` and `authScreens.ts socialButtonText`. Not moved: the iconned xxl *rows*
(Account menu, Wallet card rows) on `label` Medium 13 — they were never Bold CTA labels.

Wrap-up cleanup candidates: `detail.ts ctaInactive` (zero consumers), `share.ts ctaSecondaryTextV2`
(already zero consumers).
