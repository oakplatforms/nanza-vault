---
tags: [nanza-mobile, solution-design, sharing, share-card-restyle]
---

# Share Card Restyle — Solution Design

> **Status:** proposed. This is a **styling-alignment** design, not an architecture change. The
> deep-link flow, the reference-code plumbing, and the `ShareScreen → ShareDetailView →
> Share*Card` switch all stay exactly as they are (see [[sharing|Sharing]]). What changes is the
> **look and feel** of the share cards so a shared/deep-linked object reads like the new
> in-app detail screens (`ListingScreen` / `BidScreen`) shipped 2026-07-14.

## Goal

When someone opens a Nanza share link (`https://nanza.app/<code>`), they land on `ShareScreen`,
which renders a per-type share card (`ShareListingCard`, `ShareBidCard`, …). Those cards are the
**old look** — bordered light cards with a header row, a small price button, and "For sale" /
"Wanted" badges. Meanwhile, tapping the same object from inside the app (a home thumb, a feed
card) opens the **new look**: `ListingScreen` / `BidScreen`, built from `src/components/detail/*`
— a full-bleed pinned hero on near-black, a jumbo title, a "Sold by / Wanted by" seller row, and
a sticky "Buy 1 for $X" CTA.

**We want the share cards to feel like the detail screens.** Same palette, same typography, same
CTA copy, same seller row — so a shared link and an in-app tap present the same object the same
way. We are **not** re-pointing deep links at `ListingScreen`/`BidScreen` (that was considered and
rejected — those screens assume the full app shell and nav stack; the share landing is
deliberately a chrome-less root screen with just the wordmark). We keep the share cards and
restyle them.

## Scope: the shared surfaces

Everything below lives in **nanza-mobile only**. The affected surfaces:

| Surface | File | Renders |
|---|---|---|
| Deep-link landing | `src/screens/share/ShareScreen.tsx` | wordmark + `ShareDetailView` |
| Type switch | `src/components/share/ShareDetailView.tsx` | picks the card by reference type |
| **Listing card** | `src/components/share/ShareListingCard.tsx` | single listing |
| **Bid card** | `src/components/share/ShareBidCard.tsx` | single bid |
| **Lot card** | `src/components/share/ShareBulkCard.tsx` | bulk listing (carousel) |
| **Collection card** | `src/components/share/ShareCollectionCard.tsx` | list (read-only) |
| **Product card** | `src/components/share/ShareProductCard.tsx` | catalog entity (read-only) |
| **Group card** | `src/components/share/ShareGroupCard.tsx` | group (Join) |
| **Profile view** | `src/components/share/ShareProfileView.tsx` | user's listings/bids toggle |
| Shared parts | `src/components/share/ShareCardParts.tsx` | the DRY layer to grow |
| Styles | `src/styles/components/share.ts` | `getShareStyles(theme)` |

The `ShareDetailView` also renders **inline inside Search** (headerless) when a reference code is
typed — so any restyle must keep working without the `ShareScreen` wordmark chrome around it.

## The two looks, side by side

### Old (current share cards)
- **Bordered card**: `ShareCardFrame` — radius `xl`, `secondary.950` fill, `secondary.925`
  border, overflow hidden. Cards stack in a scroll view under the wordmark.
- **Header row** (`ShareHeader`): avatar + username + a right-aligned status **badge**
  — `badgeKind="sale"` → literal **"For sale"**; `badgeKind="wanted"` → literal **"Wanted"**.
- **Image well** (`ShareImageWell`): `fill` (cover) for listings/lots, `contain` for bids /
  collections / products; condition pills overlaid bottom-left.
- **Details row**: title (2-line) + subtitle on the left, `PriceText` on the right.
- **CTA**: a full-width `Button variant="action" size="xxl"`, but the label is a **bare verb** —
  `ShareListingCard` = literal **"Buy now"**; `ShareBidCard` = literal **"Make offer"**;
  `ShareGroupCard` = **"Join"** / **"Requested"**. Only `ShareBulkCard` composes a priced label
  (**"Buy 5 for $120"**) on its cover slide. Collection/Product have **no CTA**.

### New (detail screens — the target)
Backing stylesheet `src/styles/components/detail.ts` (`getDetailStyles`), dark theme
(`tokens/dark.json`). Resolved palette:

| Role | Token | Hex |
|---|---|---|
| Screen + sheet fill | `background` (neutral.950) | `#010101` |
| Hero well / dark CTA / icon circles | `ink` (neutral.900) | `#121212` |
| Tag-pill fill | `surface` (secondary.950) | `#2E3438` |
| Primary title / dot / dark-CTA label | `text` (neutral.50) | `#FFFFFF` |
| Tag-pill text, non-hero title | `textSecondary` (neutral.100) | `#F5F5F5` |
| Section heading + **primary CTA pill fill** | `overlayText` (neutral.100) | `#F5F5F5` |
| Card-number subtitle / envelope icon | `secondaryText` (secondary.300) | `#C7D5DA` |
| Condition-pill border | `inactive` (secondary.800) | `#5D6976` |
| Saved-heart tint | `action` (primary.400) | `#FF6DFA` |

- **No card frame.** Full-bleed pinned hero (402:500) on `#010101`. A gradient cap
  (`#010101` 0 → 0.7 → solid, 230pt tall, pulled up `-200`) fades the content in over the hero —
  the fade *is* the binding, no border/shadow/rounded surface.
- **Hero variants**: `photo` (listing) letterboxes the image `contain` over a blurred `cover`
  copy of itself; `card` (bid) centers card art at 61% width, aspect 244:340, radius `md=16`, under
  a top "gallery light" wash.
- **Title block** (`DetailTitleBlock` with `heroTitle`): **jumbo 32px SemiBold `#FFFFFF`** title →
  product-number line (14px `#C7D5DA`) → tag pills → condition pills. Tag pill = `surface`
  `#2E3438` fill, radius `xl=30`, 13px SemiBold `#F5F5F5`. Condition pill = outline shell
  (`#010101` fill, `#5D6976` border) with a **per-condition colored label** (mint `#99FF95`, etc.).
- **Seller row** (`UserActionRow`): a bold **18px `#F5F5F5` section heading** whose literal text is
  passed by the caller — **"Sold by"** (listing) / **"Wanted by"** (bid) — then 50px avatar +
  16px username, and a right-side circular envelope icon-button (`ink` circle, `#C7D5DA` icon) when
  the counterparty is messageable.
- **Condition section** (`DetailConditionSection`): "Condition" heading + the same outline pills.
- **Chat** (`DetailChatSection`): "Chat" heading wrapping the existing `CommentThread`.
- **Sticky CTA bar** (`DetailActionBar`): a fading `#010101` backdrop and a bottom row.
  - **Primary light pill**: `Button variant="action" size="xxl"`, `overlayText` `#F5F5F5` fill,
    radius `xxl=50`, label color `surface` `#2E3438`. Label is **priced**:
    listing = **"Buy 1 for $X"** ("Buy now" if no price); lot cover = **"Buy N for $TOTAL"**;
    lot child = **"Buy N for $X"** / **"Sold"**; bid = **"Sell 1 for $X"** ("Sell now").
  - **Secondary dark pill** (bid only): `ink` `#121212`, auto-width, label **"Offer"**.

## The plan

The restyle is best done by **growing the DRY layer** (`ShareCardParts.tsx`) to mirror the detail
building blocks, then re-composing each card on top of it. Today `ShareCardParts` only abstracts
frame / header / image well / metrics — the **title lockup and CTA are inlined in every card**, so
without new shared parts we'd be editing seven cards by hand. Promote them.

### 1. Repaint the shell (`share.ts` + `ShareCardFrame`)
Drop the bordered light card in favor of the detail look:
- Background → `#010101` (`background`), matching the detail screen. `ShareScreen` already fills
  with `layoutStyles.pageContainer`; align that token so the card and screen are one seamless dark
  field instead of a card-on-background.
- Remove the `cardFrame` border + radius (or reduce to a seamless full-bleed column). Keep
  `overflow` handling for the image well only.
- Because the same body renders inside **Search**, keep the frame's outer look driven by a prop so
  the Search embed can stay in a contained card if we want, while the deep-link landing goes
  full-bleed. (Decide during build — default both to the new dark look.)

### 2. New shared parts to add to `ShareCardParts.tsx`
Mirror the detail components so copy and styling live in one place:
- **`ShareTitleBlock`** — jumbo 32px title, product-number subline, tag pills, condition pills.
  Reuse the detail typography roles (`typography.jumbo`, `typography.subtitle`, `typography.pill`)
  and the outline condition-pill recipe (per-condition colored label via `getConditionColor`).
  This replaces the inlined `detailsBlock`/`title`/`subtitle` in each card.
- **`ShareSellerRow`** — the `UserActionRow` analog: caller passes the heading label
  (**"Sold by"** / **"Wanted by"**), plus avatar + username; optional envelope button reusing
  `useMessageUser`. Replaces `ShareHeader`'s role on the single-item cards. (Keep `ShareHeader` for
  now only where a compact avatar row is still wanted — e.g. profile/collection.)
- **`ShareCTA`** — a thin wrapper over the global `Button variant="action" size="xxl"` styled as
  the **light primary pill** (`overlayText` fill, `surface` label, radius `xxl`), plus an optional
  **dark secondary pill** (`ink`) for the bid "Offer". Centralizes the CTA look and, critically,
  the **priced label copy** so every card speaks the detail-screen language.

### 3. Align CTA copy to the detail screens
This is the most visible change and should match `DetailActionBar` verbatim:

| Card | Old label | New label (match detail) |
|---|---|---|
| `ShareListingCard` | "Buy now" | **"Buy 1 for $X"** (→ "Buy now" if no price) |
| `ShareBidCard` | "Make offer" | primary **"Sell 1 for $X"** (→ "Sell now") + secondary **"Offer"** |
| `ShareBulkCard` cover | "Buy 5 for $120" | keep — already priced; restyle pill only |
| `ShareBulkCard` child | "Buy now" / "Sold" | **"Buy N for $X"** / **"Sold"** |
| `ShareGroupCard` | "Join" / "Requested" | keep copy; restyle to the light pill |
| `ShareCollectionCard` | none | none (read-only) |
| `ShareProductCard` | none | none (read-only) |

> **Decision (confirmed):** match the detail screen exactly. The bid card leads with
> **"Sell 1 for $X"** (the viewer selling *to* the bidder) and demotes **"Offer"** to the secondary
> pill. The old **"Make offer"** lead is dropped. Whatever the current `BidScreen` / `DetailActionBar`
> uses is what the share card should use — no "make offer" experiment.

### 4. Per-card notes
- **Listing / Bid**: the big win — swap header→`ShareSellerRow`, details→`ShareTitleBlock`,
  button→`ShareCTA`. Hero: listing keeps `fill` (already cover); bid could adopt the `card`
  gallery-light treatment if we want full parity, but that's optional (`ShareImageWell` stays fine).
- **Lot (`ShareBulkCard`)**: already carousel + metrics + priced CTA; mostly a repaint. Restyle
  the slide title/CTA via the new parts; keep `ShareCarousel` + `ShareMetrics`.
- **Collection / Product**: read-only, no CTA. Repaint to the dark field + jumbo title + tag pills;
  keep the FairMarket value block. These diverge most from the detail screens (no seller/CTA) —
  restyle for palette + typography consistency, not structural parity.
- **Group**: already a custom full-bleed banner card — closest to the new look already. Restyle the
  **Join** button to the light primary pill; align text tokens.
- **Profile view**: embeds real `Feed*Card`s under a Bids/Listings toggle. Out of the strict scope
  (its look is governed by the feed cards, not the share cards) — align the header + toggle
  palette, but leave the embedded feed cards for a separate pass unless we want them restyled too.

## Key decisions & rationale

- **Restyle the cards, don't re-point the links.** The new `ListingScreen`/`BidScreen` assume the
  app shell (floating header, tab nav, scroll-pinned hero). The share landing is a deliberately
  chrome-less root screen (just the wordmark, taps back to Home) reused headerless inside Search.
  Cheaper and safer to bring the detail *look* to the cards than to bring the app *shell* to the
  deep link.
- **Grow `ShareCardParts`, then recompose.** Title + CTA are inlined across seven cards today.
  Promoting `ShareTitleBlock` / `ShareSellerRow` / `ShareCTA` means the copy and palette live once,
  matching how `DetailTitleBlock` / `UserActionRow` / `DetailActionBar` centralize the detail look.
- **Match CTA copy to `DetailActionBar` verbatim.** Priced, action-first labels ("Buy 1 for $X")
  are the single biggest tell that a screen is "the new look"; aligning the strings is high-impact
  and low-cost.
- **Reuse tokens, never hardcode.** All values map to existing theme tokens (`background`, `ink`,
  `surface`, `overlayText`, `secondaryText`, `inactive`, `action`) and typography roles
  (`jumbo`, `subtitle`, `pill`, `sectionTitle`). No new hex literals — per the repo's no-hardcoded-
  color rule ([[../REFERENCE|Frontend Reference]]).
- **Keep Search parity.** `ShareDetailView` renders inside Search headerless; the restyle must not
  assume the `ShareScreen` wordmark chrome.

## Open questions

1. **Bid hero** — restyle `ShareImageWell` to the `card` gallery-light treatment for full parity,
   or leave the current well?
2. **Search embed** — should the in-Search render go full-bleed dark too, or stay a contained card?
3. **Profile view** — in scope for this pass, or leave the embedded feed cards until a feed
   restyle?

_Resolved:_ **Bid primary verb** → match the new detail screen: lead with **"Sell 1 for $X"**,
secondary **"Offer"**; drop "Make offer".

## Related

- [[sharing|Sharing]] — the reference-code plumbing and `ShareScreen`/`ShareDetailView`/`Share*Card`
  architecture this design restyles (unchanged).
- [[../REFERENCE|Frontend Reference]] — theme tokens, typography roles, no-hardcoded-color rule.
- [[bulk|Bulk / Lots]] — the Lot (`K`) share card and carousel.
- [[collections|Collections]] — the collection share card + FairMarket value block.
- [[profile|Profile]] — the profile share view and its embedded feed cards.
