---
tags: [nanza-mobile, solution-design, trade]
---

# Trade tab screen — Solution Design

## Overview

The bottom-nav **Trade** tab (`src/screens/trade/TradeScreen`) is the seller/buyer's
working surface for their own market activity: a pill row, one search input, and the list
under it. Since 2026-08-26 it has exactly **two pills — `Bids` and `Listings`** (renamed from
"Buy" / "Sell" so the pill names the thing you're looking at, not the verb):

| Pill | Idle list | Search result rows |
|---|---|---|
| **Bids** | the viewer's ACTIVE bids (`fetchBids`) as `TradeListItem variant="buy"` | entities → place / edit a bid, with the "you already bid ×N @ $X" pills |
| **Listings** | the viewer's `/sell-feed` (ACTIVE singles + PUBLISHED lots; `BulkSaleCard` for lots) | entities → create / edit a listing; camera shortcut in the input for sellers; "Create lot" CTA pinned at the bottom |

`initialTab` (route param) accepts only `'Bids' | 'Listings'` — anything else is ignored so a
stale deep link can never land on a tab with no pill. The lot-creation flows navigate back with
`initialTab: 'Listings'`.

## What was removed (2026-08-26)

- **The Offers pill** — the seller-only `SellerOffers` surface (and the "Get ready to sell"
  onboarding for non-sellers) no longer lives here. Offers are still handled from the inbox
  (`OfferMessageDetail`) and the order/offer screens; `SellerOffers` itself stays for those.
- **The hidden Trade pill and its subsystem.** Trading (proposals, counters, accept/reject,
  user search to propose) was behind `TRADE_TAB_ENABLED = false` and is not a supported
  feature. Deleted outright: `screens/trade/data/{fetchTrades,fetchTrade,mutations}.ts`,
  `screens/trade/tradeDraftStore.ts`, `services/api/Trade.ts`, and the components
  `TradeProposalRow`, `TradeCardSelector`, `TradeSummary`, plus ~90 orphaned classes in
  `styles/components/trade.ts` (proposal/selector/overview/button families) and the
  `tradeEmptyContainer` / `tradeEmptyText` classes. The screen no longer invalidates
  `['trades']` / `['connections']` on focus.
- **Kept on purpose:** `useTradeUserList` / `TradeUserListRow` (the people-row recipe used
  by Friends, Group People and Search), `TradeListItem`, `BulkSaleCard`,
  `TradeCardSelectionRow` (used by the scan flows and `CollectTab`), `tradeSearchSignal`, and
  the `TradeDto` type. `TradeScreen/CollectTab.tsx` is not imported anywhere and is a
  candidate for the same treatment.

## Key decisions & rationale

- **Delete, don't hide.** The Trade tab had been "kept for a flip" since the trading feature
  was shelved; nothing else consumed its data hooks or service, so keeping it only cost every
  reader a mental branch.
- **Nouns for pills.** "Bids" / "Listings" match the profile's Trades rail vocabulary and the
  `TradeListItem` variants stay `'buy' | 'sell'` internally — only the labels changed.

## Related

- [[../INDEX|nanza-mobile]]
- [[profile|Profile]] — the Trades rail that shows the same bids/listings/lots publicly
- [[bulk|Bulk / Lots]] — the Create-lot flow the Listings pill launches
