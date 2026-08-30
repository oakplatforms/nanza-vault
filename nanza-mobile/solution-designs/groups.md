---
tags: [nanza-mobile, solution-design, groups]
---

# Groups — Solution Design

> **Status:** living. Started 2026-08-10 with the group-detail header alignment; same day,
> the detail body landed its shape (member facepile + Buy & Sell rail) and the list's
> refetch flash was fixed.

## What groups are on mobile

A group is a moderated space holding listings/bids/lots. The mobile surface:

| Screen | File | Role |
|---|---|---|
| List | `src/screens/groups/GroupsListScreen` | brand-filtered directory |
| **Detail** | `src/screens/groups/GroupDetailScreen` | banner header + facepile row + Buy & Sell rail |
| People | `src/screens/groups/GroupPeopleScreen` | read-only roster: connect / message members |
| Members | `src/screens/groups/GroupMembersScreen` | moderator roster management |
| Create/Edit | `src/screens/groups/CreateGroupScreen` | moderator CRUD |

Deep links land on the group share card (`ShareGroupCard`, code letter `G`) — see
[[sharing|Sharing]].

## Detail header: the shared banner recipe (2026-08-10)

The group detail header used to be a one-off — fixed 230pt banner with a flat black tint, a
hardcoded black bottom gradient, and a small (20pt/13pt) title + description **overlaid inside
the image**. It predated the detail-screen look and matched nothing else.

It now uses the **collection/tag detail treatment** (the same recipe as
`CollectionDetailScreen` and `SupportedTagValueScreen`):

- **Banner**: the `collectionDetailBanner*` style family — edge-to-edge 230pt image, a top
  scrim (`rgba(0,0,0,0.35)→0`, 120pt) so the floating dark-circle buttons stay legible, and a
  bottom fade to `theme.colors.background` (120pt) so the image blends into the page instead
  of ending on a hard edge.
- **Title lockup below the image, not on it**: jumbo `detailTitleHero` (32pt, 2-line clamp) +
  the description blurb in `ExpandableText` (collapsible, 4-line clamp — groups keep a feed
  below, so long blurbs clamp behind "more" unlike tag pages which render in full). The blurb
  moved off the tag pages' 14pt `tagDetailDescription` onto its own `groupDetailDescription`
  slot (2026-08-17): the profile hero's bio box (`profileBio`, Regular 16/19) with the same
  secondaryText step-back — group and profile are the two identity heroes, so their main
  copy reads at one size.
- **Group-specific header block** (`groupDetailHeaderBlock` / `…NoBanner` in `cards.ts`):
  same 16 title-to-blurb gap as the tag block, but at the app's **20pt page inset** — the
  member/join action row and feed cards below run at `base`, so groups do NOT adopt the tag
  pages' deliberate 15pt near-edge gutter.
- **No-banner case**: the old gray placeholder block is gone; a banner-less group starts at
  the title, top-padded to clear the floating buttons (same formula as collection/tag).
- `staticHeader`: back/edit/members/share are always-available circles with no title band, so
  the hide-on-scroll swing would only read as flicker (same call as the tag screen).

The banner recipe is now inlined in four places (collection, tag, group, `ShareTagValueCard`)
against shared style keys — if it grows a fifth consumer, promote it to a shared component in
`src/components/detail/`.

## The cold-open flash (fixed 2026-08-10)

Uncached opens flashed because the **loading frame carried different `PageLayout` props than
the loaded one** — the early return omitted `contentExtendsUnderHeader`, so the whole page
re-laid-out (content jumping up under the header) the instant the group query landed. Cached
opens skipped the loading frame entirely, which is why the flash was intermittent.

Rule worth keeping: **a screen's loading and loaded returns must render the same `PageLayout`
shell** (same `contentExtendsUnderHeader` / `staticHeader` / header props). The tag and
collection screens already follow it.

Data: two inline react-query queries — `['group', id]` (`?include=moderator&include=members`)
and `['group', id, 'items']` — both `STALE_TIME.LIVE` (5 min). No cache seeding from the
groups list yet; the item feed still pops in after the header on cold opens.

Rule worth keeping: **every join-flow mutation must invalidate `['group', id]`** — the
detail's Join / Request sent / Add new state reads `myMembership` off that query, and the
optimistic `localRequested` flag dies with the mount. The groups list and share landing
always did this; the detail's own Join button didn't until 2026-08-17, so a join looked
undone (Join again) on re-open until a full app reload.

## Detail body: facepile + Buy & Sell rail (2026-08-10)

The row under the description and the item feed were redesigned into the app's shared
shelf language:

- **Member facepile row** — up to four overlapping 35pt avatar circles (2026-08-15, down from five) (the avatar
  bucket's `circle_md` size; `groupDetailAvatar*` in `cards.ts`; background-colored ring
  keeps each circle legible) with the member count beside them at `bold` weight (was
  `meta`). The whole left side opens **GroupPeopleScreen**. The right slot is Join /
  Request sent for non-members and **"Add new"** for members (2026-08-15 — back in the
  row); the divider that used to close the row stays gone — it flows straight into the
  Buy & Sell rail. The slot's dress unified on `ActionPill` (2026-08-17): Join wears the
  same white `emphasis` pill as "Add new" and the profile's "Add collection", and Request
  sent is the pill's resting `disabled` form so the post-tap swap doesn't jump geometry.
  The old `groupDetailActionButton*` / `groupCardJoinButtonText*` recipes left `cards.ts`
  with it.
- **Buy & Sell rail** — the item feed's full-width `Feed*Card` stack became the homepage
  shelf experience: one `CardCarousel` (global) titled **"Buy & Sell"** mixing
  `ListingThumb` / `BidThumb` / `BulkThumb`, exactly the Query List shelf trio, so
  listings, lots and bids render identical tiles. The heading carries NO action pill
  (2026-08-15: "Add new" moved back up to the member row) — the rail reads like home's
  plain shelves; an empty group keeps the bare heading + empty-state line. Known trade-off:
  the thumbs don't carry `sourceGroupId`, so add-to-cart group attribution now only
  happens from the listing detail, and moderators lose the in-feed remove action
  (roster/items moderation still lives on the manage screens).
- **GroupPeopleScreen** (titled "Members") — the read-only people list behind the
  facepile: the group's ACTIVE members rendered with the search People-tab machinery
  (`useTradeUserList` + `TradeUserListRow`): avatar, username, connection-aware action
  (Add friend / Accept / Requested / Message → conversation). No moderation here —
  moderators keep their header icon into the managing GroupMembersScreen. Shares the
  members cache key with the manage screen.
- **Include discipline** — the detail query now asks for `members.account.profile` (the
  API's dot-path include) so the facepile has avatars, and GroupMembersScreen's group
  query was aligned to the same include: both write `['group', id]`, and a narrower fetch
  from one screen would strip the members the other reads.

- **Chat placeholder** — below the rail, the `DetailChatSection` lockup from the
  listing/tag details (the "Chat" heading + glass "Join the chat" pill), rendered
  visual-only in `GroupDetailScreen`: group chat has no posts infra yet, so the pill
  deliberately does nothing. Swap it for a real `DetailChatSection` mount once the API
  grows a GROUP postable target.
- **"Add new" is a sheet, not a navigation** (`GroupAddItemsSheet` in
  `src/components/groups/`) — the pill used to bounce to the Trade screen; it now
  slides the post composer's shop picker (`ElbPickerLayer`, reused verbatim:
  Listings/Bids toggle, your own items in the home grid, glass search + bottom X) up
  OVER the group as a modal. Tapping an item adds it to the group and keeps the sheet
  open for more; X dismisses back to the group. Backed by three new idempotent
  endpoints in nanza-api's group router — `POST /group/:id/listings|bids|bulks`
  (author-owned + ACTIVE/PUBLISHED item, moderator or ACTIVE member, upsert on the
  composite key) — until now items could only join a group at creation time via the
  targeting chips.

## Home shelves (2026-08-29): affiliates on top, Groups banners under Trending

The homepage header is now **affiliates → Trending → Groups → What's new**:

- **`AffiliateCircleCarousel`** (`components/profile/`) takes the circle rail's slot at the top:
  every public `AFFILIATE` profile's avatar + username from the new **`GET /profiles?type=`**
  endpoint (nanza-api `routers/profile.ts` — public, paginated, slim rows: id / accountId /
  type / username / avatar / description; not brand-filtered). For this release a tap simply
  opens `UserProfile`. Untitled, like the circle rail it replaced. `useHomeReady` mirrors its
  query (`['profiles','affiliates','home']`, limit 20) so the header still commits as one.
- **`GroupBannerCarousel`** (`components/groups/`) sits under Trending, titled **"Groups"**:
  the same brand-filtered, membership-annotated list the circles read (same
  `['groups','home',brand,account]` key, so the prefetch is unchanged), memberships first, as
  **`GroupBannerThumb`** tiles — the group's banner (thumbnail fallback) as a 16:9 cutout at the
  new **`banner` thumb step (80% of the content width)**, deliberately wider than Trending's
  240/362 `tag` tile so it reads as a different shelf; name bottom-left, and bottom-right the
  group screen's own pills — **Join** in the white emphasis `ActionPill` dress the detail header
  wears, **Joined** in the groups list's resting `groupThumbJoinPillJoined` form (not separately
  tappable; the whole tile is the tap). Taps open the group detail (registered users only —
  guests' tiles are inert, as the circles were). `GroupCircleCarousel` is no longer mounted on
  Home but stays in the tree for any other host.

## Home shelf circles: open-first (2026-08-17)

`GroupCircleCarousel` (the untitled home shelf) no longer joins on tap. Tapping an
unjoined circle used to fire the membership request in place — which read as nothing
happening (public groups silently joined) or a surprise "Request sent" (private ones).
Now **every circle opens GroupDetailScreen** and joining is always the explicit Join
pill there, matching the search rows and the groups list. Visual states collapsed to
two: in the group = pink highlight ring at full strength; not in the group = the
disabled 20%-opacity dim (a pending request reads as still-not-in, same dim). Guests
keep full-strength thumbs (they're in nothing — dimming the whole shelf read as "all
disabled") and any tap opens the auth window. The shelf's own join mutation and
requested-state bookkeeping went away with the in-place join.

## Slide-up navigation (2026-08-10)

The whole groups chain now presents **vertically over the tab bar** — the same
open-from-bottom motion as the listing/bid details — instead of the old left/right push
under it. Mechanics (see architecture.md § Navigation):

- The root stack's default interpolator is already the slide-up; the group screens just
  stopped overriding it. Their nested Groups-tab copies were **deleted** — the tab stack
  keeps only `GroupsListScreen`, so every drill-down (detail → create/edit → members →
  people → profile) bubbles to the root registrations and stacks vertically, covering
  the bar. The bar only shows on the tab mains.
- Back affordances switched to the down chevron (`chevron-down-2`), matching the motion;
  swipe-down dismisses.
- The inbox's "view group" system message used to seed the nested Groups stack as
  [List, Detail]; it now switches the Groups tab underneath and pushes the root detail
  on top, so dismissing still lands on the group list.
- Same treatment landed on `UserProfile` (root copy — home posts/facepile paths) and the
  homepage "See all" screens (`FeedGrid`, `WhatsNew`). Trade/Inbox chains keep nested
  horizontal copies until they migrate (planned; conversations next).

## The list's refetch flash (fixed 2026-08-10)

Two causes, same symptom ("the list flashes / double-loads"):

- `isFetching && !isFetchingNextPage` swapped the whole list for a full-screen spinner on
  **every background refetch** — including the navigator's stale refetch-on-return from a
  detail screen. Rule: background refetches never replace loaded content; only
  `isLoading` (no data for the current key) gates the loading frame.
- The groups query ran before the brands effect picked `selectedBrandId`, firing an
  unfiltered request that was immediately superseded by the per-brand one. The query is
  now `enabled: !!selectedBrandId`, with the loading frame covering the one render before
  a brand is chosen.

## Open work

- Possible cache seeding (`setQueryData` from `GroupsListScreen`) if the items pop-in
  reads badly on cold opens.
- **Group chat infra — the next job (2026-08-10).** The posts model needs a GROUP
  postable target (api-side) so the detail's placeholder chat section can become a real
  `DetailChatSection` mount; further posts/content surfaces follow from there.

## Related

- [[sharing|Sharing]] — group share cards + reference codes (`G`).
- [[share-card-restyle|Share Card Restyle]] — the same "match the detail screens" push for
  the share surfaces; `ShareGroupCard`'s repaint is scoped there.
- [[design-system|Design System]] — tokens and typography slots the header uses.
