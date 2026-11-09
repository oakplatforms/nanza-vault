---
tags: [nanza-figma, handoff, nanza-mobile, tags, sets, groups, collections, profile, hero, typography]
status: open
date: 2026-11-09
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3393-168141&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3393:168141
frame: SupportedTagValueScreen / default · Seller · edit me
repo: nanza-mobile
scope: styles-only
---

# Detail-page fixed heroes (tag, set, group, collection, profile) + bio-style tag description

Two changes, both built and tested on device on a throwaway branch (`throwaway-fixed-heros`).
The code below is that build, so treat it as the reference implementation.

1. **Fixed hero on every banner detail page:** tag, set, group, collection and profile (your
   own and someone else's). The image is pinned; the fade (now the full banner height) and the
   content scroll up over it. Each page keeps its own banner height.
2. **Tag description restyled to match the profile bio.** This is a shared style, so the set page
   and the tag share card follow.

> [!warning] App-wide — read first
> - **One shared component** (`components/detail/FixedBanner`) carries the hero recipe for all
>   six screens. Don't build per-screen copies.
> - **`tagDetailDescription` is shared**, so the set page and the tag share card change with it.
> - **Not changed:** `ShareTagValueCard`'s banner (a static image, no scroll). It is now the only
>   consumer of the old `collectionDetailBannerFade` (bottom 120), which stays as is.

## Request (verbatim)
**Changed:**
1. "the gradient overlay is updated on the hero to extend to the top of the viewport and we need
   to fix it, similar to the listing/bid screens so that the image fixes but the gradient and
   content scrolls to cover it"
2. "restyle the description right below the title [with] the same font styles as what we do for
   the bio on the profile screen"
3. "all of those should get the same fixed hero style. they can keep their heights if different
   but the covering the entire height/fixed combo should be updated (groups, sets and
   collections …)"
4. "the profile should have been in"

**Do not change:** not stated, so: everything else on these pages (rails, chat, pinned Join pill,
header buttons, titles, grid layout, search footer, profile avatar overlap, tabs).

## Measurements

### 1. Fixed hero (frame 3393:168141, 402 wide)
| Layer | Figma | Before | After |
|---|---|---|---|
| Banner | image cover, `surface` fill | scrolled with the page | **pinned** behind the scroll |
| Top scrim (`scrimTop`) | 0 → 120, 0.35 black → 0 at 40% | inside banner, scrolled | pinned with the image (keeps the floating back/share circles legible) |
| Bottom fade (`fadeHeroCollapsed`) | **the full banner height**. Stops 0 / 0.11 at 34.1% / 0.61 at 70% / solid at 100% | bottom 120 only | full banner height; **scrolls** with the content |
| Content below the banner | page background | transparent | opaque `background`, covering the pinned image |

The gradient curve is the existing `fadeHeroCollapsed` token, unchanged. Only how much of the
banner it covers changed.

Heights are unchanged per page:
| Page | Banner | `FixedBanner` size |
|---|---|---|
| Tag (`SupportedTagValueScreen`) | 290 on 402 (`aspectRatio`) | `tag` |
| Set (`SetScreen`) | 290 on 402 | `tag` |
| Group (`GroupDetailScreen`) | 230 | `standard` |
| Collection (`CollectionDetailScreen`) | 230 | `standard` |
| Profile (`ProfileScreen`, `UserProfileScreen` via `ProfileHero`) | 230 | `standard` |

### 2. Tag description under the jumbo title
| | Before | After |
|---|---|---|
| Slot | `postBodySm` (Balto 14.5/21, +0.25) | **`profileBio`** (Balto 15.5/21, +0.25): the profile hero bio |
| Color | `textSubtle` | **`onSurface`**, the same color as `profileHeroBio` |

Frame 3393:168141 still draws the old 14.5 / grey blurb. The designer's note supersedes the frame.

## How it's built

### The recipe: pinned layer + scrolling slot + opaque sheet
ListingScreen/BidScreen pin their hero by counter-translating it against an `Animated.Value`.
**Don't copy that here.** Three of these pages are `PageLayout` pages, and PageLayout already has
a **`fixedBackground`** slot: an absolute layer *behind* its transparent ScrollView (zIndex 0 vs 2),
which EntityScreen also uses. A pinned hero there needs no scroll listener and can't jitter.

1. **Pinned layer:** `FixedBannerImage` (image + top scrim at the banner's size).
2. **Scrolling slot:** `FixedBannerFade`, opening the scroll content. It's transparent and the
   same size as the banner, so the title lands where it did, and it carries the full-height fade.
3. **Opaque sheet:** everything after the slot sits on `detailBannerSheet` (`background` fill,
   `flexGrow: 1`, so short pages paint solid to the bottom).

No banner means no pinned layer and no slot; the page renders as before (`…HeaderBlockNoBanner`).
Loading states are unchanged.

### Shared component: `src/components/detail/FixedBanner/index.tsx` (new)
```tsx
type BannerSize = 'tag' | 'standard'   // 290-on-402 | 230

export const FixedBannerImage = ({ uri, size = 'standard' }) => (
  <View style={size === 'tag' ? cardStyles.tagDetailBanner : cardStyles.collectionDetailBanner}>
    <Image source={{ uri: getImage(uri, { width: IMG_WIDTH.BANNER }) }}
           style={cardStyles.collectionDetailBannerImage} />
    <LinearGradient {...getGradientProps(theme, 'scrimTop')}
                    style={cardStyles.collectionDetailBannerScrim} pointerEvents="none" />
  </View>
)

export const FixedBannerFade = ({ size = 'standard' }) => (
  <View style={size === 'tag' ? cardStyles.tagDetailBannerSpacer : cardStyles.collectionDetailBannerSpacer}
        pointerEvents="none">
    <LinearGradient {...getGradientProps(theme, 'fadeHeroCollapsed')} style={cardStyles.detailBannerFade} />
  </View>
)
```

### Styles (`cards.ts`)
```ts
// module scope — the 230 shared by the banner and its slot
const DETAIL_BANNER_HEIGHT = 230

collectionDetailBanner:       { …, height: DETAIL_BANNER_HEIGHT, … }   // was a raw 230
tagDetailBannerSpacer:        { width: '100%', aspectRatio: 402 / 290 },
collectionDetailBannerSpacer: { width: '100%', height: DETAIL_BANNER_HEIGHT },
detailBannerFade:             { position: 'absolute', top: 0, left: 0, right: 0, bottom: 0 },
detailBannerSheet:            { flexGrow: 1, backgroundColor: theme.colors.background },

// collection grid rows scroll over the pinned image too — the gap under each row must be opaque
collectionDetailGrid: { …, paddingBottom: theme.spacing.sm /* was marginBottom */,
                        backgroundColor: theme.colors.background, … }
```

### Per page
**Tag, set, group** (PageLayout):
```tsx
<PageLayout … fixedBackground={x.banner ? <FixedBannerImage uri={x.banner} size="tag|standard" /> : undefined}>
  {x.banner ? <FixedBannerFade size="tag|standard" /> : null}
  <View style={cardStyles.detailBannerSheet}>
    {/* header block → rail / member row / items → chat, unchanged */}
  </View>
  {/* group: GroupAddItemsSheet / ActionModal stay OUTSIDE the sheet, as before */}
</PageLayout>
```
Remove each screen's local `bannerFade` / `bannerScrim` and the now-unused `LinearGradient`,
`getGradientProps` and `getImage` imports. The group screen still uses `Image` and `getImage`
for avatars.

**Collection** (FlatList, no PageLayout):
- The pinned layer is the **first child** of `screenContainer`, wrapped in
  `layoutStyles.pageFixedBackgroundContainer` (absolute fill), with `pointerEvents="none"`. The
  floating `Header` (zIndex 10) stays above it, and the FlatList (a later sibling, transparent)
  scrolls over it. Render it **only when the grid is up**
  (`banner && !(isLoading && items.length === 0)`) so the loading state stays plain.
- `listHeader` = `FixedBannerFade` + the title/count block wrapped in `detailBannerSheet`.
- Grid rows are made opaque by the `collectionDetailGrid` change above. Its only consumer is this screen.

**Profile** (`ProfileHero`, shared by `ProfileScreen` + `UserProfileScreen`):
- Both screens pass `fixedBackground={profile.banner ? <FixedBannerImage uri={profile.banner} /> : undefined}`.
- `ProfileHero` swaps its banner block for `{banner ? <FixedBannerFade /> : null}` and keeps
  taking `banner` (it still drives the avatar overlap and the no-banner padding). Drop its
  `getCardStyles`, `getGradientProps`, `LinearGradient` and `Image` imports.
- **The avatar overlap stays.** `profileHeroAvatarOverlap` (−84) still pulls the avatar row up out
  of the identity block onto the fade, so the avatar sits over the pinned image as before.
- The identity block is opaque: `profileHeroBlock` gains `backgroundColor: theme.colors.background`.
  It **doesn't grow**. If it did, two growing fills on a short page would split the spare space
  and open a gap between the hero and the friends row.
- **Own profile (`ProfileScreen`):** the friends row + tabs are wrapped in the new
  `profileSheet` (`flexGrow: 1`, `background` fill) in `profile.ts`.
- **Someone else's profile (`UserProfileScreen`):** `profileStyles.root` already paints the
  background but used to wrap the hero too, which would hide the pinned image. `<ProfileHero>`
  moves **out of** `root`, directly above it. `root` now holds only the tabs and acts as their
  opaque sheet.

### Description: edit the shared default
```ts
tagDetailDescription: { ...scale.profileBio, color: theme.colors.onSurface },
```
Also update the stale comment on `groupDetailDescription`, which says the tag blurb uses
`postBodySm` / `textSubtle`.

## Code map (local `throwaway-fixed-heros`; match these on your branch)
- **New:** `src/components/detail/FixedBanner/index.tsx`
- `src/screens/tags/SupportedTagValueScreen/index.tsx`
- `src/screens/sets/SetScreen/index.tsx`
- `src/screens/groups/GroupDetailScreen/index.tsx`: most of its diff is re-indentation from the
  sheet wrapper. Real change ≈ 20 lines.
- `src/screens/collections/CollectionDetailScreen/index.tsx`
- `src/styles/components/cards.ts`: constant, 4 new classes, `collectionDetailBanner` height,
  `collectionDetailGrid`.
- `src/components/profile/ProfileHero/index.tsx`
- `src/screens/profile/ProfileScreen/index.tsx`
- `src/screens/profile/UserProfileScreen/index.tsx`
- `src/styles/components/profile.ts`: `profileHeroBlock` background, new `profileSheet`.
- `src/styles/components/typography.ts`: `tagDetailDescription` + the `groupDetailDescription` comment.
- `src/components/global/PageLayout/index.tsx`: **read only**; `fixedBackground` already exists.
- **`tagDetailDescription` consumers:** SupportedTagValueScreen, SetScreen, `ShareTagValueCard`.
- **Still on `collectionDetailBannerFade` (unchanged):** `ShareTagValueCard` only.

## QA
- **All six screens with a banner:** the image stays put; the fade and content slide up over it.
  Pulling down past the top leaves the image fixed while the content drags down over it.
- **All six without a banner:** same as before.
- **Short pages** (few posts / items): solid to the bottom, no image showing below the content.
- **Group:** member row, Buy & Sell rail and chat all cover the image; add-items sheet and action
  modal still open.
- **Collection:** gaps between grid rows stay solid over the image; card shadows look right on the
  row fill; loading state has no banner; the search overlay and footer still work.
- **Profile (own + someone else's):** the avatar still overlaps the banner bottom and sits over
  the pinned image; friends row (own) and tabs cover the image as they scroll; the avatar viewer
  (someone else's) and Edit profile / gear (own) still work; no gap between the hero and the
  friends row on a short page.
- **Description:** profileBio size in `onSurface` on the tag page, the set page and the tag share card.

## Open questions
None.

## Prompt
```text
Implement the handoff nanza-vault/nanza-figma/handoffs/2026-11-09-tag-page-fixed-hero.md (read
it in full). Figma file 7c3Zz4nDUOutQG1BsnRRAw, frame 3393:168141.

SCOPE: STYLES ONLY. No data, query, navigation or behaviour changes.

1. FIXED HERO on SupportedTagValueScreen, SetScreen, GroupDetailScreen, CollectionDetailScreen,
   ProfileScreen and UserProfileScreen (via ProfileHero), each keeping its banner height
   (tag/set 290-on-402, group/collection/profile 230). Build ONE shared
   component, components/detail/FixedBanner: FixedBannerImage (image + scrimTop, pinned) and
   FixedBannerFade (transparent banner-sized slot with fadeHeroCollapsed at FULL height; it
   scrolls). PageLayout pages: image via the existing fixedBackground prop (do not edit
   PageLayout); slot first in the scroll; everything after on an opaque detailBannerSheet
   (background, flexGrow 1). Collection: pinned layer as the first child of the screen container
   (pageFixedBackgroundContainer, only when the grid is up), slot + sheet-wrapped header block in
   listHeader, collectionDetailGrid gap margin -> padding + background fill. Profile: ProfileHero
   renders the slot, keeps the -84 avatar overlap, profileHeroBlock gets a (non-growing)
   background; own profile wraps friends + tabs in a new profileSheet; UserProfileScreen moves
   ProfileHero out of (above) profileStyles.root. No banner = no layer, no slot. Leave
   collectionDetailBannerFade and ShareTagValueCard's banner alone.
2. DESCRIPTION. Change the shared tagDetailDescription to ...scale.profileBio with
   theme.colors.onSurface (the profile hero bio). QA SetScreen and ShareTagValueCard too.

House rules (CLAUDE.md): plan first with confidence scores; this touches 11 files, so escalate the
file list in your plan. Styles in src/styles/components/*; no inline styles, raw sizes or
colours; typography slots only. Run /ui-review before finishing, then set this handoff to built
and add a Log line.
```

## Log
- 2026-11-09: handoff written for the tag page (fixed hero + bio-style description); built and device-tested on `throwaway-fixed-heros`
- 2026-11-09: scope widened to set, group and collection pages (same fixed hero, own heights) via the shared `FixedBanner` component; built and device-tested on `throwaway-fixed-heros`
- 2026-11-09: profile added (ProfileHero on ProfileScreen + UserProfileScreen) via the same `FixedBanner`; built and device-tested on `throwaway-fixed-heros`
