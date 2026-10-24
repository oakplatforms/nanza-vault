---
tags: [nanza-mobile, solution-design, detail-screens]
---

# Detail Screens & Thumb Cards — Solution Design

## Overview

In-app detail pages for marketplace items — **`ListingScreen`** (route `ListingDetail`,
singles AND lots) and **`BidScreen`** (route `BidDetail`) — plus the **card-thumb system**
and the **shelf domain** that surface them everywhere. Before these existed the feed cards
were the terminal surface; now every thumb, feed ellipsis menu, and (eventually) deep link
lands on a full detail page.

## The screens

- **Composition** — a pinned hero under a scrolling content sheet: an `Animated.ScrollView`
  counter-translates the hero by the scroll offset (clamped against pull-down bounce), and
  the sheet's first child is an eased **gradient cap** (transparent → background) that
  travels with the scroll and binds image into content. Photo heroes letterbox (contain)
  over a blurred cover copy so nothing crops; card (bid) heroes float on the ink well under
  a top-down "gallery light" wash. Styles live in `src/styles/components/detail.ts`.
- **Lots** ride inside `ListingScreen` via the `bulkListingId` param: the hero becomes a
  swipeable `DetailHero` carousel with the windowed-dot `DetailPager` (max 9 dots, tappable,
  paging pauses while a slide is zoomed via `ZoomableImage` pinch/pan/double-tap).
  `BulkDetailScreen` remains the seller-facing item-management surface.
- **Content column** — `DetailTitleBlock` (jumbo lockup: title, name/number pair, tag
  pills, condition pill), `UserActionRow` ("Sold by"/"Wanted by" with message + connection
  actions via `useMessageUser`, which wraps the existing messages hooks/sheets),
  `DetailChatSection` (inline `CommentThread`), and the sticky `DetailActionBar` CTA bar
  (shared with the form language — see [[forms|Forms]]).
- **Header actions** — floating darkCircle chrome: owner-only Edit, save Heart
  (`SavedItemsContext`, optimistic), Share; one `RadialGlow` spans the group.
- **Entrance (Aug 2026)** — both screens enter **from the bottom**, matching
  `SupportedTagValueScreen` and `PostDetail`. They get it by *not* overriding: dropping
  `gestureDirection: 'horizontal'` + `CardStyleInterpolators.forHorizontalIOS` from their
  `Stack.Screen` options lets them inherit the navigator's default screenOptions
  (translateY from screen height + opacity fade + vertical dismiss). The header's
  `backIcon` is `chevron-down-2` to match — **the two must move together**, since a down
  chevron on a right-sliding screen points the way it does not exit. Note the dismiss
  gesture changed with it: a down-swipe now closes these screens, not a right-swipe.

## The 15pt page gutter — one left edge per page

The Chat heading sits at **15** (`detail.ts` `chatSectionInset`), and so did the tag
screens' title block (`cards.ts` `tagDetailHeaderBlock`) until October 2026 (next section).
The detail screens' sections sat at **20**, so the title, "Sold by"/"Wanted by" and
"Condition" hung 5pt right of "Chat" on the same page.

The fix is **opt-in**, not a retune. `detail.ts` `section` / `titleBlock` are shared far
beyond these screens — CreateListing, EditListing, CreateBid, EditBid, EditOffer,
`EntityModal`, `ScanItemEditScreen` and six share cards all mount
`DetailTitleBlock` / `UserActionRow` / `DetailConditionSection`, and every one of them is
composed around the 20. So `sectionPage` / `titleBlockPage` (15) were added alongside, and
the three components take an `inset?: 'page'` prop that defaults to today's behaviour.
Only the two detail screens pass it. The prop shape deliberately copies `PostThread`'s
existing `inset`, so there is one convention rather than two.

Moving Chat 15 → 20 instead was rejected: the tag screens were built around 15 at the time,
so it would have traded one misalignment for another. Chat's gutter is now **host-selectable**
instead — see the next section.

## Tag screens on the 20pt page inset (2026-10-13, Figma 2191:209052)

`SupportedTagValueScreen` (and `ShareTagValueCard`, which mirrors it) moved off the 15pt
near-edge gutter onto the app's 20pt page inset, so title, blurb, children rail and chat
share one left edge:

- **Gutter** — `cards.ts` `tagDetailHeaderBlock` / `…NoBanner` and the rail's
  `carouselHeadingPadTag` / `carouselContentTag` retuned 15 → `base`. The `tag` inset variant
  stays as the knob; only its value moved (its history: 20 → 8 → 12 → 15 → 20).
- **Chat follows its host** — `DetailChatSection` takes `inset?: 'page' | 'base'` (default
  `'page'` = 15, unchanged for listing/bid/group/entity). `'base'` puts the heading on
  `detail.ts` `chatSectionInsetBase` (20) and the thread rows on `PostThread`'s existing
  `'detail'` gutter (`posts.ts` `commentCardInsetDetail`, 20) — no new thread class.
- **Pinned pill follows too** — `JoinChatFooter` takes `inset?: 'sm' | 'base'` (`'fixed'`
  only; default `'sm'` = the 8pt floating-input gutter the Search footer and guest tab bar
  share). `'base'` lays `postComposer.ts` `footerRowInsetBase` (20) over the search footer's
  row. Only the tag screen passes it; group, entity and PostDetail keep the 8.
- **Section rules** — the home feed's hairline (`cards.ts` `shelfDivider`: 1pt `surface`
  top border, 16 below, 24 margin after) now separates blurb → rail and rail → Chat.
  The rail takes it through `TagCarousel`'s existing `divider`; `DetailChatSection` gained
  a matching opt-in `divider`, drawn only while its heading shows so an empty pinned thread
  never ends in a stray line. The header block's 40 below the blurb became 24 (the feed-row
  seam) and the rail's `tagDetailCarouselSection` margin wrapper was retired — the rule owns
  that spacing now.
- **Children circles a quarter larger** — `thumbSizes.ts` `circleSm` is `(66 * 1.25) / 362`
  (82.5 on the frame, image 67.5 via `CircleThumb`'s unchanged 54/66 ratio). The step feeds
  only the tag rails' glass circle, so nothing else moved.
- **Pill label reads the thread** — the screen calls `usePosts` (the same query
  `DetailChatSection` reads, so one request) and passes `label="Start the chat"` when the
  count is a confirmed zero; while loading, and once anyone has posted, the pill keeps its
  default "Join the chat", so a populated tag never flashes "Start".
- **Thread tail** — the screen passes `disableBottomPadding` to `PageLayout`. Its default
  spacer (safe area + 66) and the chat section's own `chatSectionPinnedClearance` (75) both
  exist to clear bottom chrome and were stacking, leaving ~135pt under the last post; the
  clearance alone lands the last row a feed seam above the pill; `tagDetailPostsSection`
  then adds 32 (21 + 32 bottom margin) so the last row's icons sit in the clear part of the
  pill's 162pt backdrop fade rather than under it. Group, entity and PostDetail still stack
  both paddings — retune them the same way if asked.
- **Hero** — `cards.ts` `tagDetailBanner`: the shared `collectionDetailBanner*` recipe
  (image, `scrimTop`, `fadeHeroCollapsed`) at the Figma master's taller proportion,
  `aspectRatio: 402 / 290` rather than a fixed height. The group/collection/profile heroes
  keep the shared 230. The title clamps at 3 lines (the master frame is sized for three
  38pt lines).
- **Blurb** — `typography.ts` `tagDetailDescription` composes the new `postBodySm` scale slot
  (Balto 14.5/21, +0.25 — postBody one size down; Figma 2318:60108 draws it at 14/20 and the
  app takes it a half size and a point of leading up by decision, on new `size.14.5` and
  `lineHeight.21` tokens) in `textSubtle`: Figma draws it in a raw #909090 that is not a palette token,
  and `textSubtle` (neutral.400) is the closest by luminance. Down from `postBody` 16/19 in
  `secondaryText`.

Not adopted from the mock: its "Read the article" CTA (no link field on
`SupportedTagValueDto`), the news-post identity row, the titled "Related" card-tile rail (the
untitled circle rail stays), and the mock's 240×160 r28 TagThumb (the Oct 11 220×165 r11
tune stands).

The info button (`detail.ts` `sheetInfoButton`) also moved onto the shared glass recipe —
`getGlassFill` plus a `<GlassRim />` child, replacing a bespoke 12%-white frost with an 8%
flat border that predated it. Geometry is unchanged at 32pt; see [[design-system]]
"Glass chrome" for why a circle takes a gradient rim rather than a border.

## Data (read-only, existing endpoints)

`useFetchListing` / `useFetchBid` (screen-local `data/` hooks) wrap the pre-existing
`listingService.get` / `bidService.get`; `useFetchByReference` (existing) is the
deep-link/recovery path resolving S/B/K codes; `useBulkListing` (existing) powers lots;
`useViewerActiveListing` reuses `useMakeOfferStart`'s exact
`['UserProductListing', entityId, accountId]` key so BidScreen's "Sell 1 for $X" hits a warm
cache. **No new endpoints, no mutations.**

## Entry points

- **Thumbs `push`** their detail route (`navigation.push`) so browsing 20 items deep stacks
  and back walks the chain; feed/menu entries `navigate`.
- **Feed ellipsis menu** (`ActionModal/Feed`): "View listing / View bid / View lot" rows.
- Routes are registered at the root stack (`AppNavigator`) with horizontal-iOS transitions;
  params in `types/navigation.ts` (`listingId`/`bidId`/`bulkListingId` + `referenceCode`).

## The thumb system (`components/cards/`)

Presentational `ItemThumb` (image, `ThumbShade` radial-vignette + bottom-fade, 2-line
title, price pill in a 44pt hit slot) + DTO wrappers `ListingThumb` / `BulkThumb` /
`BidThumb` (trading-card 143:200 aspect). `CollectionThumb` (banner or `CollectionMosaic`
bleed-grid cover, count pill, fair-market row) and `GroupThumb` (full-width cover, join
pill) complete the family. Width is ALWAYS decided by the parent via `thumbSizes`
(`sm`/`smd`/`md`/`lg`, fractions of the 402pt reference frame) or an explicit `width` —
thumbs never size themselves. No thumb fetches or mutates; all behavior is injected or a
route push.

**The cutout tile recipe (2026-10-13, Figma 2142:194698).** `TagThumb` is the reference:
the `tag` step is 240/362 at 3:2 (240×160 on the frame, reversing the Oct 11 trim to 220),
radius 28, a `<GlassRim pill radius={28} tone="image" />` rim-light on the top and bottom edges
(mounted AFTER the absolute image and shade, unlike the glass circles' first-child order;
the `image` tone — named for the surface the rim sits on, an image thumbnail rather than a
flat glass fill — scales the shared curve to 0.45, a ~13% peak against the mock's 11%
hairline, since the chrome's full 30% read as a frame around the artwork), the shared
`ThumbShade` `fade` variant now starting at 32.7% (was 46% — the post reference hero moved
with it by decision), and the name on `itemTitle`/`overlayText`, one line, 16 from the bottom
and right and 20 from the left (the mock's 16 plus a 4pt on-device nudge, composed as
`16 + xxs`). Both group cards wear the same recipe "except width": `GroupBannerThumb`
keeps the `banner` step and `GroupThumb` stays full-width, each with height derived to land
at 160 on the frame (`aspectRatio: (362 * 0.8) / 160` and `362 / 160`); their Join pill and
the list card's member-count line are content, not recipe, and stayed. Rail spacing was
already on the mock's numbers (12 heading gap, 8 shelf inset, 12 item gap, 20 section break).

## The shelf domain (`components/shelf/`)

Containers for card content — **cards/ is content, shelf/ is containers**:

- **`ShelfHeading`** — the shared title-left / action-pill-right lockup (44pt action row,
  ink tag-pill recipe). Consumed by CardGrid, CardCarousel, and GroupsListScreen.
- **`CardGrid`** — titled expandable wrap-row grid (columns → computed card width, Yoga-3
  `alignItems:'flex-start'` fix); `initialCount` gates the `ShowMoreButton` footer; call
  sites that paginate externally pass `initialCount = data.length`.
- **`CardCarousel`** — titled horizontal RNGH FlatList shelf (coexists with the stack's
  back-swipe on Android).
- **`types.ts`** — `ShelfProps<T>`, the family contract every variant extends. Deliberate
  decision: NO slot-based `Shelf` container yet — ShowMore is stateful content-coupled
  logic, and only two variants exist. At variant #3, promote the chrome into a `Shelf`
  component rendered *inside* the variants (zero call-site churn since the contract holds).

**Home (transitional):** HomeScreen renders a 2-column mixed-thumb CardGrid above the card
feed from the **same** feed query and pagination — a cosmetic mirror intended to replace
the card list outright once validated.

## Deliberately not shipped (July 2026 partial release)

The branch's "More from this seller/buyer" CardCarousel shelves (and their
`useSellerItems`/`useBuyerBids` hooks), the home shelf sections + `ShelfScreen` +
`useFetchFeedItems`, and `CircleThumb`/`GroupCircleCarousel`. These live on
`fe-updates-7-13-26` for a later release.

**EntityModal (hybrid):** the modal adopted the branch's detail-language chrome —
gradient-cap sheet flow over the background fill (no more surface card + handle pill),
`DetailTitleBlock heroTitle` lockup, `action`-variant CTA pills with plain-string
labels (dark hugging Sell via `detail.ts ctaDark`), and the detail tag-pill skin for
attribute tags — but deliberately kept its merged chronological listings/bids feed
(`FeedListingCard`/`FeedBidCard`, exact-item scroll from the Buy Now / Sell Now prices,
inline offer sheet) instead of the branch's Buy now / Sell now thumb rails. The
`entityModalFeedRails` class is staged in `modals.ts` for if/when the rails land.
