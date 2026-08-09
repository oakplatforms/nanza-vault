---
tags: [nanza-mobile, solution-design, plan, posts, whats-new]
---

# What's New — Affiliate Brand Posts on the Homepage

> **Status: PLAN, 2026-08-04.** Quick design from Skylar's voice walkthrough; the **What's New
> section mock has landed** (folded in below), header plus-circle mock still incoming. This is the "brand/homepage composer
> (affiliates)" release that [[posts-chat|Posts Chat]] deferred. Model and gating already exist in
> [[../nanza-api/solution-designs/posts|nanza-api Posts]] — `BRAND` home, affiliate-only creation.

## Overview

Affiliates get a **plus circle in the Nanza Mobile top header** that opens the existing post
composer in brand mode. The posts they create are **brand-level posts** (`postableType: 'BRAND'`,
`postableId: brandId`) and surface on the homepage in a new **"What's New" section between
Trending and Top Picks** — three post cards plus a **See all** that leads to a dedicated What's
New screen.

## The header plus circle

- Lives in `TopHeader` (`components/global/TopHeader`), **next to search and the Balance pill —
  to the right of Balance**.
- **Same size as the search icon circle**, but a **pink (primary) circle background** with a plus
  icon inside — visually the primary action in the header.
- **Affiliate accounts only.** Gate: `profile.type === 'AFFILIATE'` (the same rule the API
  enforces on `BRAND` post creation). SessionContext has no profile-type awareness today — add an
  `isAffiliate` flag alongside `hasSellerAccess` (the session's user fetch already includes
  `account.profile`, so the data is there).
- Tap → opens the **existing `PostComposer` modal** (`components/posts/composer/PostComposer`),
  the same modal used everywhere else (listings, bids, tag pages) — no new composer. It was built
  target-agnostic for exactly this: pass `BRAND`/brandId instead of a listing/tag home.
  - Note: [[posts-chat|Posts Chat]] kept `JoinChatFooter`'s `inline`/`fixed` glass-pill
    placements "for the future brand feed" — **the entry is now this header plus circle instead**;
    those placements stay unused.

## Composer in brand mode (v1 scope)

- Same composer, same single-attachment v1 rules (image / link→embed / card / listing / bid).
- **No tags in v1.** The tags attach circle is hidden in brand mode (same mechanism as tag
  pages). The API allows up to 5 supported + 25 secondary values on brand posts — that fan-out
  UX (down to supported-tag types) is explicitly **later**; don't build any of it now.

## The "What's New" homepage section (mock landed 2026-08-04)

- **Placement: between Trending (`HomeTagCarousel`) and "Top picks for you"** in
  `HomeScreen`'s list header. Current order: groups shelf → Trending → *(What's New here)* →
  Top Picks → query-list shelves. Heading reads **"What's new"** (sentence case, standard
  section heading), with **"See all" far right of the heading row** (the standard `ActionPill`
  heading recipe, same as Top Picks).
- **Content:** the brand's posts — `posts WHERE postableType='BRAND' AND postableId=:brandId`
  (the homepage brand feed the API design already describes). Newest first.
- **Cards (per the mock):** the **same post-card design** the chat threads use — `ink`-surface
  card, avatar + bold username + muted relative time (`2d`), body text with **inline pink tag
  mentions** (*Kassai*), footer row with reply arrow + count and outline heart + count on the
  left and a **share icon far right** (drawn in the mock; share remains behind the
  reference-code lift, so render it per the posts-chat share rule). Tap-through to `PostDetail`
  as usual.
- **Reference attachments render as the row tile** (mock, card 2): record image left, bold name
  + muted set/code right (*Runechant of Greed / IAR145*), **white price pill on the far right**
  (`$650.00`) — the entity search-row shape with the `thumbPricePill`, not the dark hero square.
- **Three posts shown** in the preview; the section is a fixed preview like the home feed — no
  in-place pagination.

## The What's New screen

- "See all" opens a dedicated screen listing **all** the brand's posts with pagination — same
  preview→"See all"→infinite-scroll pattern as the homepage feed (`FeedGrid`) and query-list
  shelves. Standard post cards, load-more paging.
- Standard detail navigation rules apply (`push`, never `navigate` — see the posts-chat gotcha).

## What already exists (why this is mostly FE work)

From [[../nanza-api/solution-designs/posts|nanza-api Posts]]:

- `BRAND` is a real `postableType` pointing at the Brand row — the homepage feed is just a
  home-scoped post query; no new model.
- **Creation is already affiliate-gated server-side** (`profile.type === 'AFFILIATE'`, the first
  consumer of Profile Types). Everyone reads the feed; only affiliates write.
- Content items are already allowed on brand posts.

Mobile work: the header plus circle + affiliate flag, mounting the composer with a BRAND target,
the What's New section + screen, and the feed query hook (`screens/home/data/` `useFetchX`
naming).

## Implementation record (2026-08-04) — built, uncommitted

Brand posts only this release (and secondary tag values are gone repo-wide — no effect here,
brand posts carry no tags). What landed:

- **Session**: `isAffiliate` on `SessionContext` (`account.profile.type === 'AFFILIATE'`; the
  user fetch already included `account.profile`).
- **Entry**: `JoinChatFooter` gained `placement="headerPlus"` — the primary-pink plus circle
  (`headerPlusCircle` in `headers.ts`: the search circle's recipe at the 44pt hit size — a step
  smaller than the 50pt search circle, per Skylar — primary-500 fill).
  `TopHeader` mounts it after the Balance pill for affiliates with a selected brand, passing
  `BRAND`/`selectedBrandId`; the header icon group's 8pt gap spaces it. Same composer, no tag
  context → tagless brand posts.
- **Section**: `components/home/WhatsNewSection` — the shelf recipe (`carouselSection*` +
  `ShelfHeading` at the sm inset) titled "What's new", mounted in `HomeScreen`'s list header
  after `HomeTagCarousel`. Renders nothing when the brand has no posts. `PostThread` gained
  `previewCount` (cap at 3, suppress load-more); "See all" shows when `total > 3` and navigates
  to `WhatsNew`. Replies go through an entry-less `JoinChatFooter` host, same as PostDetail.
  New `shelfBodyPadSm` class in `cards.ts` pads the bare thread at the grid gutter.
- **Screen**: `screens/posts/WhatsNewScreen` + root-stack route `WhatsNew` (no params — reads
  `selectedBrandId` from the session), FeedGrid's push-over-tabs pattern and floating title
  header. Pages the same `['Posts','BRAND',brandId]` query on scroll via PageLayout
  `onEndReached`; `PostThread` gained `hideLoadMore` so the tap-to-load button stays hidden.
- Also this session: `HOME_PREVIEW_COUNT` 6 → 4 (the related homepage tweak below).

## Open until the remaining mocks land

- Exact plus-circle styling vs. the mock (pink token, icon weight) — header mock still incoming.
- The What's New screen's header treatment.

## Related homepage tweak (2026-08-04)

- The home feed preview (`HOME_PREVIEW_COUNT` in `HomeScreen`) drops from a grid of 6 to a
  **grid of 4** for now, making room for What's New on the first screenful.
