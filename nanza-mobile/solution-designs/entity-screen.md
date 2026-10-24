---
tags: [nanza-mobile, solution-design, plan, entity]
---

# Entity Screen — Modal → Full-Screen Slide-Up

> **Status: PLAN (in progress), 2026-08-10.** Converts the `EntityModal` bottom sheet into a
> full-screen detail screen using the app's standard slide-up pattern (supported-tag / post /
> listing / bid detail). Fold the durable outcome into a proper solution design when it lands.

## Why

The entity experience has outgrown the 80% bottom sheet: it already carries a hero image, title,
fair-market pricing, CTAs, description, tags, and a listings/bids feed grid. Every other detail
surface of that weight (SupportedTagValueScreen, PostDetail, ListingDetail, BidDetail) is a
full-screen slide-up with the floating chevron-down + share header. The entity surface should read
identically: shareable from the top, dismissed with a down arrow, and full-height.

## Where things stand today (mapped 2026-08-10)

- `EntityModal` (`src/components/modals/EntityModal/index.tsx`, ~830 lines) renders inside a
  `BottomSheet` at 80% height with a fixed background card image; the wrapper is
  `screens/entity/EntityScreen`, registered on the root stack as route **`Entity`** with
  `presentation: 'transparentModal'` and zero-duration transitions (AppNavigator.tsx:339-353) —
  the sheet does its own spring animation.
- **It is already a navigation route** — the feared "it's not in the navigation" problem doesn't
  exist. 15 call sites navigate to `Entity` with a full `EntityDto` snapshot (feed cards, entity
  cards, search rows, trade rows, cart, collections grid, post reference tiles, listing/bid info
  rows, scan edit).
- **Image tap today does NOT open a fullscreen viewer** — it springs the sheet's drag-zone spacer
  between 200 and 555pt to reveal more of the fixed background image. The shared
  `components/global/ImageViewer` is used by the feed cards and OrderItems, not by the modal.
- Share lives in the ellipsis **action sheet** (`ActionModal/Entity.tsx`), not a header icon,
  built from `product.referenceCode` (needs the detail fetch — list snapshots lack the code).
- The slide-up pattern is the **navigator default**: `AppNavigator`'s root `screenOptions` already
  define the vertical interpolator + vertical dismiss gesture; pattern screens simply omit the
  horizontal overrides and pass `{ headerShown: false, gestureEnabled: true }`.
- The header recipe is `PageLayout` `headerProps`: `floating: true`, `showBackButton` +
  `backIcon: 'chevron-down-2'`, `rightIcons: [{ icon: 'share-3', variant: 'darkCircle' }]`,
  share gated on `useCanShare()` (hidden for guests).

## Target design

**One route, restyled.** Keep the existing `Entity` route name and `{ entity }` param so all 15
call sites keep working untouched; swap the screen's presentation and body.

### Navigation (small, deliberate diff to the read-only zone)

- Re-register `Entity` like the other slide-up screens: drop `presentation: 'transparentModal'`
  and the zero-duration transition overrides; inherit the navigator's vertical interpolator;
  `{ headerShown: false, gestureEnabled: true }`.
- Add the same inline comment the other four screens carry (down-chevron rationale).
- Call sites that can land on an entity **from** an entity context (post reference tiles already
  do this via `push`) must use `push`, not `navigate` — the established trap: `navigate()` reuses
  the existing route and Back skips a level. Audit the 15 call sites; feed/search/trade/cart
  surfaces can stay `navigate`, anything reachable from within a detail stack becomes `push`.
- `initialTab` is a dead param (modal no longer reads it) — drop it from the param type and the 5
  call sites still passing it.

### Screen body (`screens/entity/EntityScreen` becomes the real screen)

Follow the ListingScreen/BidScreen composition — `PageLayout` with `contentExtendsUnderHeader`,
floating header, hero image with the shared banner bottom-gradient recipe, then the modal's
existing content blocks in order:

1. **Hero** — the card image, full-bleed under the floating header, bottom `LinearGradient` into
   the background (the groups/tag banner recipe). **Tap keeps the current expand/collapse
   experience** (decided 2026-08-10): the spring toggle that grows the image area to show the
   full card and collapses it back — ported from the sheet's drag-zone spacer to the screen's
   scroll header space. No fullscreen `ImageViewer`.
2. **Title row** — `DetailTitleBlock` only — **the ellipsis "more" button dies** (see header).
3. **Price row** — FairMarket price, Sell Now / Buy Now tiles with `scrollToFeedItem`, unchanged.
4. **CTA pills** — Place bid / Sell → `CreateBid` / `CreateListing`, unchanged.
5. **Description + `EntityTags`** with the ShowMore expand/collapse, unchanged.
6. **Feed grid** — the merged listings+bids 2-column grid, unchanged (keep the `feedReady`
   two-rAF deferral so the push animation stays smooth).

### Header (decided 2026-08-10: no ellipsis anywhere)

- Left: `chevron-down-2` dark circle → `goBack()`.
- Right cluster, dark circles like PostDetail's trash/heart/share row:
  - **Plus circle** (add to collection) — registered users. **Always** opens the existing
    multi-collection selection screen, even when the user has exactly one collection. The
    single-collection inline plus/minus stepper and the whole `EntityActionModal` bottom sheet
    are removed. Guests get `AuthModal`.
  - **Share circle** (`share-3`) — shown when `useCanShare()` && `referenceCode` from the detail
    fetch; `guardedShare({ message: buildShareUrl(code) })` — the ListingScreen recipe. The
    action-sheet share row goes away with the sheet.

### What carries over unchanged

- `fetchEntityDetail` (INSTANT staleTime), user bid/listing lookups, price-resolution precedence
  (live query → detail → route snapshot, incl. the flattened post-reference `price` fallback).
- `AuthModal` for guest participation taps.
- `PRODUCT_VIEW` analytics on screen open.
- The hero tap expand/collapse spring interaction (ported, not deleted).

### What gets deleted

- The `BottomSheet` wrapper, `dismissPanGesture`/scroll-top arming,
  `registerExternalModal('entity-modal')` cascade bookkeeping, and the orphaned
  `entityModalTabsWrapper`/`entityModalStickyTabs` styles.
- **The entire `EntityActionModal` entity action sheet** and the ellipsis entry point: share and
  add-to-collection move to header circles, and the single-collection stepper/dropdown is
  replaced by always routing to the multi-collection screen. The ModalStack `ConfirmSheet` for
  remove-from-collection moves with the flow it belongs to (the collection screen), or is removed
  if unreachable from the new surface.
- Long-term: `components/modals/EntityModal` moves to `screens/entity/` as the screen body; the
  "modal" name retires.

## Risks / watchpoints

- **Highest-traffic surface in the app** — 15 entry points; regression surface is wide. Ship
  behind a quick manual pass of each entry family (feed, search, trade, cart, collections, posts,
  listing/bid info rows, scan).
- The root-stack vertical dismiss gesture has its own response-distance behavior (from-top
  gesture) — verify dismiss feel matches the other slide-up screens, don't try to recreate the
  sheet's scroll-top drag arming.
- Entity-on-entity depth: a post tile inside the feed grid → PostDetail → entity tile → Entity
  again. The `push` discipline makes this stack correctly; check memory/height of stacked heroes.
- Android: the style-bucket memoization in the modal existed because the entrance spring
  stuttered — keep the memo when porting the body.

## Implementation record (2026-08-10) — built, uncommitted

- `screens/entity/EntityScreen` is now the real full-screen detail (the `EntityModal` file is
  deleted). PageLayout gained an optional `scrollRef` prop so the screen can drive
  scroll-to-feed-item; the hero is PageLayout's `fixedBackground` with the tap-toggled spacer.
  Heights tuned on-device with Skylar (2026-08-10): **225 collapsed → 555 expanded** — the
  straight port of the sheet's geometry (370/725) showed too much card collapsed and pushed
  the body off-screen expanded.
- `Entity` re-registered as a standard slide-up (inherits the navigator's default vertical
  interpolator; `transparentModal` + zero-duration overrides deleted, and the now-orphaned
  `stackNavigatorCardStyleTransparent` with them).
- Header: `chevron-down-2` back; right circles = **`cards-2`** (the composer's overlapping-
  cards glyph — reads as "collections", decided over a plus; always → `AddToCollection`,
  guests get `AuthModal`) + `share-3` (`useCanShare` && detail-fetched `referenceCode`, iOS
  `{url}` / Android `{message}` like the tag screen).
- **All 15 call sites now `push`, not `navigate`** — Entity can sit above listing/bid/trade
  screens that can open another Entity, and `navigate()` would pop back to the earlier one.
  `initialTab` dropped from the param type and the 5 senders. The stale "search overlay's own
  navigator" comment in `SearchEntityRow` corrected (overlay hosts pass `onPress`; the ambient
  branch always resolves to the root stack).
- Feed defer switched from the two-rAF trick to `InteractionManager.runAfterInteractions`
  (waits out the push transition). Description rendering now uses the shared
  `EntityDescriptionHtml` instead of a private copy of the HTML renderer.
- Orphaned styles removed from `modals.ts` (ellipsis button, sticky tabs, market-price rows,
  drag zone, content card…); `entityScreenImageContainer` added (the 550 hero box, no top
  radius). `EntityActionModal` itself survives — `EntityCard/List`'s long-press sheet still
  uses it; only the entity screen dropped it.

## Decisions (Skylar, 2026-08-10)

1. Hero keeps the **current tap expand/collapse** interaction — no fullscreen `ImageViewer`.
2. **No ellipsis at all**: share and add-to-collection are header dark circles; the entity
   action sheet is removed. Plus **always** opens the existing multi-collection screen, even
   with one collection (the inline plus/minus stepper and dropdown go away).
3. Route name stays `Entity`.
