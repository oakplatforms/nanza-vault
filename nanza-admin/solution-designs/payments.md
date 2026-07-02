---
tags: [nanza-admin, solution-design, payments, orders]
---

# Payments Visibility (Admin surface) — Solution Design

## Overview

The admin's only job around the delivery-mode × payment-type work is **visibility**: letting an
operator see, at a glance, whether an order ships or is in person, and whether it was paid by card or
cash. The model, fee/tax rules, and enums are defined in
[[../../nanza-api/solution-designs/payments|nanza-api — Payments]]; the mobile checkout UI is in
[[../../nanza-mobile/solution-designs/payments|nanza-mobile — Payments & checkout]]. This note is the
admin slice.

## How it works

An order carries `deliveryType` (`SHIP` | `IN_PERSON`) and `paymentType` (`CARD` | `CASH`) on the DTO
(from `@oakplatforms/types`). In-person orders have **no shipment and no shipping method**, and cash
orders have **no payment intent**.

- **No breakage.** The Orders views were already null-safe for the no-shipment / no-shipping-method
  case — `getActiveShipment` returns null and the Shipment / Shipping-Method sections are guarded
  behind `{activeShipment && …}` / `{order.shippingMethod && …}`, so an in-person or cash order renders
  cleanly (it just omits those sections). No defensive changes were needed there.
- **Orders list** (`app/Orders/index.tsx`): a new **Delivery** column shows a `Ship` / `In Person`
  badge plus a `Cash` badge when applicable (`getDeliveryBadge`), so cash / in-person orders are
  spottable without opening them.
- **Order detail modal** (`app/Orders/OrderDetailModal.tsx`): **Delivery** and **Payment** badge
  fields were added to the Order Details section.

## Key decisions & rationale

- **Visibility only, no operator actions.** In-person orders complete on the *seller's* accept in the
  mobile app; admin doesn't need to drive that flow, so the surface here is read-only badges.
- **Lean on existing null-safety.** Because the Orders views already guarded shipment/shipping-method
  access, supporting in-person/cash was additive (two badge fields + one column), not a refactor.

## Related

- [[../../nanza-api/solution-designs/payments|nanza-api — Payments]] — the authoritative model, fee/tax rules, and accept/collection-sync flow.
- [[../../nanza-mobile/solution-designs/payments|nanza-mobile — Payments & checkout]] — the buyer-facing checkout UI.
- [[../INDEX|nanza-admin]]
