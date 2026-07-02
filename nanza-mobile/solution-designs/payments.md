---
tags: [nanza-mobile, solution-design, payments, checkout]
---

# Payments & Checkout — Solution Design

## Overview

How the mobile cart lets a buyer choose **how they get an order** and **how they pay for it**, and
how the summary reflects that. An order has two orthogonal axes — a **delivery mode**
(Untracked / Tracked shipping, or **In Person**) and a **payment type** (**Card** or **Cash**) — both
persisted on `Order` and surfaced to the app through the generated `@oakplatforms/types` DTO
(`deliveryType`, `paymentType`). The backend fee/tax rules and the cross-repo model live in
[[../../nanza-api/solution-designs/payments|nanza-api — Payments]]; this note is the **UI slice**.

The single rule that drives the UI: **only in-person Cash is free of Stripe** — so only cash zeroes
tax and the small-order fee. In-person **Card** is a normal Stripe order minus shipping (still taxed,
still pays the $1 minimum). Everywhere the gating predicate is `isCash`, not `isInPerson`.

## How it works

### Delivery selector — one 3-way control

`ReviewCartScreen/OrderShipping/SelectedShippingMethodAction` renders a **3-way segmented control**:
**Untracked / Tracked / In Person**. It reuses the existing `ToggleSwitch` (generalized from a fixed
2-tuple to `readonly ToggleOption[]`; the active indicator positions by index so any count works).

- Untracked/Tracked pick a `shippingMethod` (via `orderService.update`) and set `deliveryType: SHIP`
  + reset `paymentType: CARD` (so a stale CASH from a prior in-person session can't violate the
  cash-only-for-in-person backend rule).
- **In Person** uses a sentinel option id; selecting it calls a sibling handler that sets
  `deliveryType: IN_PERSON` + defaults `paymentType: CASH`, and the API detaches any shipping method.
  While in person, the quantity-driven ship-tier auto-switch effects are disabled.
- Under the toggle sits a two-line subtle **helper** describing the active option, always ending
  "Seller must accept order.":
  - Untracked — "Ships in a plain envelope with a stamp, no tracking, so a lost package can't be verified."
  - Tracked — "Comes with a tracking number that confirms delivery. Most secure way to order."
  - In Person — "For people who know each other or are meeting up."

### Payment lives in the global "Pay With", not per-order

`CartActions` (one section under the whole cart, next to the order summary) owns payment and address:

- **Pay With** shows the saved card always; a **Cash** radio is the **second** option and appears only
  when the cart has ≥1 in-person order. Selecting Cash/Card writes `paymentType` to **every**
  in-person order in the cart (`ReviewCartScreen` passes down `inPersonOrderIds`); switching away from
  in-person falls the selection back to the card.
- **Deliver To** is hidden entirely for an all-in-person cart (no address needed). An all-in-person
  **cash** cart requires neither a card nor an address, so those checkout gates are relaxed via
  `requiresPayment` / `requiresShipping` props derived in `ReviewCartScreen`.
- Pay With / Deliver To were converted from dropdowns to **auto-selected radios** (`RadioField`):
  a filled pink inner dot inside a thin gray ring — no checkmark.

### Summary math — identical rows, zeroed, so nothing jumps

The zeroing rule is applied in every price surface — `CartOrderSummary` (per order in the cart),
`OrderSummary` (post-order detail), the `CartOrder` collapsed-total preview, and the `ReviewCartScreen`
cart-total/fees preview:

- **In person:** `shipping = 0`, no handling/options; the summary still renders the **same rows**
  (Tax, Shipping Rate, etc.) at `$0` so toggling In Person never shifts the layout. Row order is
  **Tax, then Small Order Fee** directly beneath it.
- **Cash** additionally zeroes tax and the small-order fee; **in-person card** keeps the real tax and
  the $1 small-order fee (it goes through Stripe).
- The "incl. … tax and shipping" lines (both the per-order collapsed preview and the cart-top total)
  fall back to a **"includes no extra fees"** placeholder for in-person, preserving the exact two-line
  spacing instead of disappearing.

### No-jump principle: disable, don't unmount

The visible "jump"/fade when toggling In Person came from elements **unmounting**. The fix is to keep
them mounted and *disable* them: the shipping-options section and the $1-minimum warning stay rendered
but dimmed for in-person, and the toggle's transient loading state blocks re-taps **without** dimming
(the disabled-opacity CSS that flashed on every tap was removed; selection is optimistic).

### Order status & confirmation for in-person

`OrderStatus` renders a dedicated **two-step** in-person timeline: *Awaiting Seller* → *Order Complete*,
where the second step (and its connector line) only appear once the order is `COMPLETED` (i.e. the
seller accepted). Ship-To is hidden on the order detail; the confirmation screen copy is in-person
specific ("awaiting the seller to confirm your in-person handoff"), and cash copy notes no payment was
processed through Nanza. Seller Accept/Decline works without a tracking status (in-person has none).

## Key decisions & rationale

- **One global Pay With, cash as the second option.** Keeping payment in the existing bottom section
  (rather than a per-order selector) matches the old card/address flow; Cash simply joins it,
  enabled only when relevant, and writes through to every in-person order.
- **Same rows, zeroed — never unmount.** The whole class of "jump"/fade bugs traced to conditionally
  rendered rows/sections. Rendering identical (zeroed) rows and disabling-not-removing the extras is
  the durable fix; placeholders keep even the fee subtitle's spacing stable.
- **`isCash`, not `isInPerson`, gates money.** In-person card is a normal taxed Stripe order minus
  shipping; only cash — which never reaches Stripe — is fee/tax-free. This keeps the mobile math in
  lockstep with the backend (which enforces the same predicate authoritatively).

## Related

- [[../../nanza-api/solution-designs/payments|nanza-api — Payments]] — the authoritative fee/tax rules, the enum model, and the accept/collection-sync flow.
- [[../INDEX|nanza-mobile]]
