---
tags: [nanza-mobile, solution-design, forms]
---

# Forms — the stacked form language (July 2026)

One form language across every create/edit surface, established on Create
Collection and rolled out to Sell (listing create/edit), Buy (bid create/edit),
Group, Scan-item edit, Lot (BulkTab form variant), and Offer.

## The language

Fields stack full-width inside `inputStyles.formStack` (20pt `base` gutters,
18pt gaps), no divider rows, no label columns — placeholders act as labels and
optional help text sits under each field.

- **Text input** — `formTextInput`: 56pt outline pill (1pt `border`,
  `radius.max`), `pill` type slot, entered text `overlayText`, placeholder
  `inactive` (set via `placeholderTextColor` at the call site). Focus swaps the
  border to `overlayText` via `formTextInputFocused` (call sites track focus).
  Variants: `formTextInputMultiline` (auto-height, `radius.lg`, descriptions)
  and `formTextInputRow` + `formTextInputRowInput`/`formTextInputPrefix` (the
  "$" price row).
- **Help text** — `formHelpText` (`meta` slot, `inactive`, indented to the
  field's 20pt text inset). Wrap field + help in a plain View so the stack gap
  doesn't split them. `formHelpRow`/`formHelpRowText` hold composed help
  content — the fair-market price + logo under Price fields.
- **Pill toggles** — `formPillToggleRow` splits equal-width 50pt pills
  (`formPillToggle`); `formPillWrapRow` + hug-width `formPill` wrap multi-option
  rows. Resting pills: `background` fill, `border` outline, `inactive` label.
  Selected: `formPillToggleActive` (background fill, `inactive` border,
  `overlayText` label) — except **conditions**, whose selected pill fills with
  the condition's own color (`getConditionColor` /
  `theme.colors.conditionColors`) with a `background` label
  (`formPillTextOnColor`; dark fills like damaged keep the light label).
- **Quantity** — `Form/QuantityPills` (global): pills 1–4 plus a `plus-2` pill
  that reveals a numeric custom input; value flows as the string the forms'
  `useQuantityInput` already holds. Consumers: listing create/edit, bid
  create/edit, scan edit.
- **Pickers** — `formOutlineButton` ("Add photo/banner/thumbnail": 56pt outline
  pill, camera icon + `inactive`-toned label so empty reads as unfilled); picked
  media becomes a full-width rounded preview (`formPhotoPreview` 1:1,
  `formBannerPreview` 2:1, `formThumbnailPreview` 3:4) with the shared remove ×.
- **Shared chip components** — `ConditionChipsRow` (global) and
  `GroupTargetingChips` render the pill language for all five listing/bid/scan
  forms; the old horizontal-scroll + edge-fade (and its `modalBgColor` prop) is
  gone. GroupTargetingChips carries its own "Where to post" help text.

## The fixed action bar

Form screens render `DetailActionBar` (the Listing/Bid detail CTA bar) as a
sibling of their scroll container with `stickToKeyboard`:

- **Keyboard tracking** — `useKeyboardHeight` (hooks/): iOS animates with each
  keyboard event's own duration; Android returns 0 because the manifest's
  `adjustResize` already lifts fixed overlays. The bar translates up by
  `keyboardHeight − insets.bottom`, landing one gutter above the keys.
- **One-tap dismiss** — scroll containers use
  `keyboardShouldPersistTaps="handled"`; the bar's gradient band claims taps
  only while the keyboard is open and dismisses it (it otherwise swallowed the
  natural "tap above the keyboard" spot, forcing a second tap).
- **Gating** — `primary.inactive` ghosts the CTA to 4% opacity (applied to the
  slot wrapper with `pointerEvents: none`, NOT via Button `disabled`, which
  would stack a 50% dim and press-flash) until the form reports validity.
- **Save flow** — forms are forwardRef components exposing `submit()` and
  mirroring `onBusyChange`/`onCanSubmitChange`; screens wire them through
  `useFormBar` (hooks/), which also holds the post-save close for 700ms so the
  CTA's success checkmark reads. Screens keep a `didCloseRef`-guarded goBack
  because some forms fire both onSuccess and onClose.
- Scroll content adds `layoutStyles.formScreenScrollContent` bottom padding to
  clear the bar's solid band.

## Button loading — trailing arc + checkmark (app-wide)

The global Button no longer swaps its title for a spinner. `loading` keeps the
label at full opacity and renders the dotLottie arc
(`assets/icons/loading-2.lottie`, bundled via the `lottie` assetExt added to
metro.config.js) at the right edge — absolute-right on fullWidth buttons,
inline-after-label on hug-width ones (Panel `sm` Saves). New `success` prop
swaps the arc for `checkmark-3`. Both recolor to the variant's resolved label
color (`StyleSheet.flatten` of the composed text style; the lottie via
`colorFilters` on its single "Shape Layer 1"). Presses stay blocked while
loading/success; the 50% dim remains tied to the `disabled` prop only.
LoadingSpinner survives only for the Wallet payment-method overviews.

The Button **action variant is THE CTA**: `overlayText` fill, `surface` label,
`radius.xxl`, all five sizes — single-sourced in buttons.ts (DetailActionBar
carries no per-bar override). The old tertiary-green action fill and the
`buttonBuy`/`buttonSell` overrides are gone.

## Key decisions

- **Bar ownership is the screen's, not the form's** — forms stay pure field
  stacks; imperative handles (deliberately un-memoized so submit never closes
  over stale state) carry submit/delete/share up.
- **BulkTab and EditOffer keep in-flow submits** (restyled to the action
  variant) — each has multiple hosts (three bulk screens; two offer screens +
  EntityModal), so the host-owned bar conversion was deferred rather than
  touching modal hosts. They still inherit the trailing-arc loading.
- **`stacked` layouts deleted** — CreateListing/EditListing/CreateBid/EditBid
  each had a dead `layout="stacked"` render path (no call sites); the rewrite
  removed them along with `variant`/`surfaceBgColor` props.
- **Legacy DSL removed** — the `listingInlineBorderedRow` family, condition
  chiclets, camera buttons, fair-market row styles, `groupBanner/Thumbnail`
  previews, `buttonBuy`/`buttonSell`, and `createCollectionSubmitRow` were
  deleted as zero-consumer once the last form migrated. `listingInlineTitle*`
  (the Sell/Buy entity header) and `listingInlineItems*` (lot items row)
  survive as live styles.

## Not yet migrated (own follow-up plan)

Auth screens, ConfirmIdentity views, AddToCollection, and the Panel-based
account/wallet/shipping screens keep their current layouts (they get the new
button loading for free). The Panel-header Save pattern is likely to stay by
design — see `.plans/form-language-phase-next.md` in nanza-mobile for the
draft scope.
