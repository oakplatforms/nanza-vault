---
tags: [nanza-mobile, solution-design, profile]
---

# Profile — Solution Design

## Overview

There are two profile surfaces: **`ProfileScreen`** (your own profile, editable) and
**`UserProfileScreen`** (viewing someone else, read-only). Both present the same three tabs —
**Listings**, **Bids**, and **Collections** — over a paginated feed. The design goal is
**read-only parity**: a visited profile should look and feel like your own, minus any editing
affordances, so browsing another user is a first-class experience rather than a stripped-down
one.

## How it works

### Listings & Bids tabs — count + share header

On your own profile, each of the Listings and Bids tabs renders a **`ProfileSectionHeader`**: a
count label ("X Listings" / "X Bids") plus a **share-2** action. The user profile renders the
**same header** above each list, with:

- **Counts** computed from the already-fetched page totals (`sellFeedData` / `bidsData`) — the
  same expression the own-profile tabs use, no extra fetch.
- **Share** targeting the **viewed user's** `profile.referenceCode` via
  `buildShareUrl(code, 'sell' | 'bid')` → `guardedShare`. The share is guarded when the code is
  absent (the user-profile DTO must expose `profile.referenceCode`, same field the own profile
  reads from `currentUser.account.profile.referenceCode`). See [[sharing|Sharing]] for the
  profile-code (`U`) scheme.

### Collections tab — same card grid, read-only detail

Both profiles show the same **`CollectionTargetCard` grid** (the mosaic collection card). On the
own profile, tapping a card opens the editable `CollectionDetailScreen`; on a user profile it
opens a **read-only `UserCollectionDetailScreen`**:

- A header, a **debounced search input** (250ms), and a read-only `EntityCardList`
  (`gestures={false}`).
- Search is **server-side**, reusing the propose-trade collection-search endpoint
  (`GET /list/collection/{accountId}?search=…&page=…`) — the same endpoint `UserCollection`
  already calls, just with `&search=` added, so it handles collections with more than 20 items.
- **No** plus-circle/add, no edit, no save, no delete, and no edit-menu actions. **Share** is
  allowed (these are public collections).

The read-only detail is a **thin dedicated screen**, deliberately not built by threading a
`readOnly` flag through the large, stateful editable `CollectTab` — that would risk the working
own-profile flow. See [[collections|Collections]] for the full collection design.

### The profile hero (2026-08-15)

Both profile screens open on the **listing/bid/collection detail template**, via one shared
component (`components/profile/ProfileHero`): floating dark-circle header controls, an
edge-to-edge **banner** under them (`Profile.banner`; top scrim for control legibility, bottom
fade into the page background — the collection-detail classes verbatim), then **avatar → the
username in the jumbo `detailTitleHero` slot the collection name occupies → bio → social-links
row**. A profile without a banner skips straight to the identity block, padded down past the
floating controls (the `tagDetailHeaderBlockNoBanner` clearance recipe). This applies to
regular profiles and affiliates alike.

The old **Deals / Collected / Portfolio stats row is gone** from the user profile, and the
own-profile screen regained an identity block the same day it had been stripped — both screens
now share the hero. Social links load on the dedicated `profileSocialLinks` query
(`view=session` strips derived relations from the session user, so they can't ride it), and
the **social row sits ABOVE the bio** (2026-08-17 — was under it). Banner-overlap avatar:
46pt on regular profiles, 64pt (well onto the image) on affiliates. The hero-to-content
hairline divider is **gone from both profile screens** (2026-08-17, after two intermediate
passes — affiliate-only, then links-only, then always-on, then removed; the
`profileHeroDivider` style survives for GroupDetail).

**Own-profile header (2026-08-15):** the floating circles are edit pencil → share (`U` code,
registered-only, hidden until minted) → settings cog — no avatar up there; the live tab bar
beneath is the way out. The main top header everywhere else keeps the avatar circle when signed
in; signed OUT that same slot is a **Log In circle** beside the Sign Up pill (both feed one
AuthModal at different steps), and the tab bar's circle slot is **Search for everyone** — the
old signed-out Log In circle there is retired along with its auth-modal wiring.

### Tabs & the Posts tab (2026-08-15; reordered 2026-08-17)

Both profiles carry a **Posts** tab of the account's root posts. **Posts leads and
Collections follows, for everyone** (2026-08-17 — the profile is a posts surface first).
**Affiliates** (revised 2026-08-17 from the posts-only surface — they CAN trade) get Posts
(always — it's their fallback surface) → Collections → Listings/Bids; never Saved or
References.

**Empty-hide is the PUBLIC profile's rule only** (2026-08-17): on UserProfileScreen,
Posts/Listings/Bids/References hide when empty and **Collections hides when the owner has
only the default primary collection and it's empty** (no public second collection,
`entityListCount` 0 — the screen runs UserProfileCollection's `['lists','COLLECTION']`
query verbatim so React Query dedupes). A posts-only profile therefore shows no pills row
at all; the fallback surface is the roster's first tab. On your OWN profile the owner
always sees their entry points: Listings/Bids show whenever access allows (the sell-feed
and bids first-page hooks remain as prefetch), Collections and Saved always show, and only
Posts/References stay count-gated.

Post cards on profiles wear the **full footer** (reply / heart / share) — the same dress as the
homepage — and **no card anywhere carries a delete or edit icon**. Tapping a card (or its reply
glyph, which has no composer to open here) pushes the post's `PostDetail`, whose top nav owns
edit and delete. The trash that used to sit in the full footer after the heart is gone from
`PostItem` entirely; replies keep their identity-row trash, since the detail reply list is their
only surface.

### Edit profile — one Save-changes bar (2026-08-15)

EditProfile adopted the group form's save UX: the per-section "Update" heading actions are
gone; a fixed **Save changes** DetailActionBar (stick-to-keyboard, `useFormBar` checkmark →
dismiss) lights up when any section is dirty and saves every dirty section (profile fields +
social links; each save path reports success so a failure holds the screen open). Media
uploads (avatar/banner) still happen on pick and the privacy toggle keeps its own confirm —
they were never Update-gated. The scrolled content reserves the bar's height via
`useActionBarPadding` — the same clearance fix applied to CreateGroupScreen, whose form was
hiding its last fields under the bar.

### Social-links editor — no in-flow dropdown (2026-08-16)

Social links are **every profile's now, not just affiliates'** (2026-08-17): the EditProfile
section, the hero render, and the API's replace-set PUT all dropped their affiliate gates
(see the vault's nanza-api `affiliate-social-links` design for the endpoint side).

`SocialLinksEditor` (shared by EditProfile and the group form) dropped its
unclaimed-row state: rows are now always claimed (logo circle + url field + X-that-deletes),
and **"Add new" opens the provider action sheet directly**, appending the picked row — the
"Select provider" dropdown row read as an extra step. Tapping a row's logo reopens the sheet
to switch that row's provider; an empty set is just the Add-new pill. One-per-type still
holds because the sheet only offers unclaimed services, and "Add new" retires when every
provider has a row.

### Keyboard insets on Android (2026-08-16)

PageLayout's `automaticallyAdjustKeyboardInsets` is an iOS-only RN prop, and `adjustResize`
no longer shrinks the window under enforced edge-to-edge (targetSdk 35) — so with the
keyboard up, forms could scroll but their tail stayed under the keys. PageLayout now pairs
the prop with an **Android measured-keyboard-height content pad** (the composer pickers'
`usePickerKeyboardPad` treatment): any screen passing `automaticallyAdjustKeyboardInsets`
gets both platforms handled. EditProfile switched to that prop, replacing its inner
`KeyboardAvoidingView` — which sat *inside* PageLayout's ScrollView and never resized the
viewport; CreateGroupScreen already passed it.

### Header spacing

The count-header's bottom spacing was bumped to **18px** via a new **`theme.spacing.smd = 18`**
token (there was no 18 between `md`=14 and `lg`=24), keeping it a named token rather than a
hardcoded size, and repointing `collectionCountHeaderTight.marginBottom` at it.

## Key decisions & rationale

- **Reuse, don't fork.** Parity is achieved by reusing existing pieces — `ProfileSectionHeader`,
  `CollectionTargetCard`, `EntityCardList`, `buildShareUrl`/`guardedShare` — with only two thin
  new files (the user-profile collection grid variant and the read-only detail screen).
- **Read-only detail as its own screen.** Cleaner separation and zero risk to the editable Collect
  flow; reuses a proven server-side search endpoint instead of inventing client-side filtering.
- **Server-side collection search.** Handles collections larger than one page (>20 items),
  debounced and paginated — the same behavior the trade flow already relies on.
- **A named `smd` token, not an inline 18.** Honors the no-hardcoded-sizes rule; the token is
  additive to `dark.json` and changes no existing value.
- **Guard the share when the code is missing.** The user-profile fetch must surface
  `profile.referenceCode`; if it's absent for a given profile, the share action is hidden rather
  than shipping a broken link.

### The Trades rail (2026-08-26)

The separate **Listings** and **Bids** sections are gone from both profiles. In their place
sits ONE horizontal rail titled **Trades** (plain "Trades" on the own profile too — Skylar
declined "My Trades") that mixes the account's ACTIVE listings, PUBLISHED lots and ACTIVE bids
newest-first — the group detail's Trades rail on a person, rendered by the same
`ListingThumb` / `BulkThumb` / `BidThumb` trio. The section order is now Posts → Collections →
Trades → Saved Items (own only).

- **Data:** `GET /trade-feed?accountId=&page=&limit=` (nanza-api, next to `/sell-feed`), a
  discriminated union `{ kind: 'listing' | 'bulk' | 'bid', … }` typed as `TradeFeedItem` in
  `types/index.ts`. Hooks live in `UserProfileScreen/data/fetchUserTradeFeed.ts`:
  `useUserTradeFeedPreview` (6 items, the rail) and `useUserTradeFeed` (20/page, the See-all
  grid via `ProfileSeeAll` section `'trades'`). The query-key prefix is `['UserTradeFeed',
  accountId]`; every listing/lot/bid create/edit/delete site invalidates it alongside
  `UserSellFeed` / `AccountBids`, so the rail refreshes without a manual pull.
- **Visibility is the server's call:** the owner sees everything; anyone else gets the same
  public-or-shared-group rules `/listings` and `/bids` apply, so private/group trades never
  leak onto a public profile (the old sell-feed preview had no viewer filter).
- **Gates:** the own profile shows Trades when the viewer has seller OR customer access (the
  two former per-section gates merged); the public profile always tries and hides it when
  empty (the per-section empty-hide rule). The vacation-mode empty copy carried over.
- **Quantity pill on thumbs:** `ItemThumb` grew a `quantity` prop; above 1 it renders a
  **×N** pill beside the price in the same white-pill dress as the trade/search rows'
  quantity badge (`thumbPillRow`, the lockup's pill-cluster gap). Listings pass
  `listing.quantity`, bids `bid.quantity`, lots their available total — so every rail and
  grid that uses the thumb trio shows it, not just the profile.
- **WTB / WTS intent tag in the pill row (2026-08-29, shared `IntentTag`):** every carousel /
  grid tile says what its owner wants as the FIRST pill of the footer's pill row, ahead of the
  white price pill; the ×qty count sits at the far RIGHT end of that same row (`marginLeft:
  'auto'`) as a black `ink` badge with off-white `neutral.100` digits (`thumbQuantityBadge*`,
  2026-08-29 — it had been a second white pill beside the price): `intent="buy"` (bids → **WTB**, bid pink `action` / primary 400)
  or `"sell"` (listings and lots → **WTS**, condition-scale mint `conditionColors.mint`), ink
  label on both. `ItemThumb` takes `intent`; `ListingThumb` / `BulkThumb` pass `sell`,
  `BidThumb` passes `buy`. No owner identity on the tile: an avatar + username strip was tried
  the same day (top of the tile, then above the title, then under the pills) and scrapped
  along with its API plumbing — query-list results and the entity rails' `ProductListings` /
  `ProductBids` are back to `entity` only, with no `account.profile` widening.
  The chiclet itself lives in `src/components/global/IntentTag` with its styles in
  `badges.ts` (`intentTag*`): the `'tile'` variant wears the thumb price pill's exact dress
  (`pricePillAmount` face, xs sides, xxs vertical, xs corners, the same `min` descender nudge) so
  the row reads as one run of pills; the `'row'` variant (sm sides, xxs corners) is the squarer
  chiclet the listing / bid detail's compact `UserActionRow` shows beside the username (it used
  to own its own `userIntentTag*` styles in `detail.ts` — those moved here so the two surfaces
  can't drift). An earlier one-off corner tag (`thumbIntentTag*`, plus a temporary white "WTT"
  demo variant) had been removed; this is the durable replacement. The pill row's slot is `xs`
  shy of the min hit target (`thumbPillSlot` / `thumbPillSlack` in `cards.ts`) — tightened the
  same day — and the footer padding subtracts the bottom slack so the pills land on the gutter.
- The old preview hooks (`useUserSellFeedPreview`, `useUserBidsPreview`) were deleted; the
  infinite `useUserSellFeed` / `useUserBids` stay for the Trade screen and the post composer's
  ELB picker.

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
- [[collections|Collections]] — the collection grid + read-only detail in depth
- [[sharing|Sharing]] — the profile (`U`) reference code and share cards

## Saved items retired (2026-08-29)

The profile's **Saved Items** section and its See-all mode are gone, along with every heart on
listing, bid and lot cards and detail screens (`Feed*Card`, `ListingScreen`, `BidScreen`). `SavedItemsContext` and `services/api/SavedItem.ts` are deleted too (2026-08-30): the heart on a
post is `hooks/useLike.ts` — local optimistic state seeded from the post's `likeCount` /
`viewerHasLiked`, one `postService.toggleLike` (`POST /post/:id/like`) — so nothing about likes
is fetched at app load any more. `screens/savedItems/`, `FeedItem.isSaved` and the `savedItems`
styles/query keys are deleted. Favorites replaces
saving in a later release. API side: [[../../nanza-api/solution-designs/saved-items|Saved items →
post likes only]].

