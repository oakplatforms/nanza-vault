---
tags: [nanza-figma, handoff, pre-dev, nanza-mobile, tags, sets, groups, collections, profile, hero, gradient]
bucket: pre-dev
status: open
date: 2026-11-12
originally: 2026-11-09 (flat handoffs/ folder, before the buckets)
verified: 2026-11-12 against dev ac083896 (v0.4.3)
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3393-168141&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3393:168141
repo: nanza-mobile
scope: new shared component + render-tree changes on 6 screens + styles + 1 gradient token
---

# Detail-page fixed heroes (tag, set, group, collection, profile)

On every detail page with a banner, the **image is pinned** and the page scrolls up over it. A
full-height gradient carries the image into the page, so the hero and the content read as one
surface. The gradient is the separator between them. As you scroll, the image fades out
behind it.

Pages: tag (`SupportedTagValueScreen`), set, group, collection, your own profile and someone
else's profile. Each keeps its own banner height. Built and device-tested on the designer's
reference build.

> [!info] Pre-dev: why this one needs planning
> - **New shared component:** `components/detail/FixedBanner`. All six screens use it. Don't
>   build per-screen copies.
> - **Render-tree changes on six screens.** Each page's content after the hero is wrapped in an
>   opaque "sheet".
>   - `UserProfileScreen` moves `ProfileHero` **out of** its root view.
>   - `CollectionDetailScreen` adds a pinned layer behind its `FlatList`.
> - **Layering, not animation.** It uses `PageLayout`'s existing `fixedBackground` slot. There
>   are no scroll listeners. Don't copy ListingScreen/BidScreen's counter-translate approach.
> - **New gradient token:** `fadeHeroFixed` in both token files.
> - **No changes to:** data, queries, navigation, `PageLayout` itself.

## Request (verbatim, condensed)
1. "the gradient overlay is updated on the hero to extend to the top of the viewport and we need
   to fix it, similar to the listing/bid screens so that the image fixes but the gradient and
   content scrolls to cover it"
2. "all of those should get the same fixed hero style. they can keep their heights if different
   but the covering the entire height/fixed combo should be updated (groups, sets and
   collections …)", then "the profile should have been in".
3. The gradient brief (2026-11-12): "a fluid layout that feels like its one, with the gradient
   being the content separator … dark enough to blend w the page but light enough to show the
   image and as you scroll to create the effect of the image fading out".

**Do not change:** everything else on these pages. That includes the rails, the chat, the pinned
Join pill, the header buttons, titles, the grid layout, the collection search footer, the
profile avatar overlap and the tabs.

## Not in this handoff
- **Tag description font.** The 2026-11-09 version restyled `tagDetailDescription` to the
  profile-bio style. That change is **dropped**. Leave `typography.ts` alone.
- **Pull-down past the top (iOS).** iOS rubber-bands past the top of the page, so pulling down
  drags the content off the pinned image. That stays as it is. `bounces={false}` was considered
  but not taken, because it would also remove the bottom bounce.
- **`ShareTagValueCard`'s banner** is a static image with no scroll, so it's unchanged. It's
  now the only user of `collectionDetailBannerFade` (bottom 120), which also stays.

## Reference build
- **Branch `throwaway-hue-change`** (designer's machine, uncommitted). This round is stacked on
  the [[2026-11-12-brand-rehue-logo-splash|brand re-hue]] handoff's changes, which aren't part
  of this one.
- **[[assets/2026-11-12-tag-page-fixed-hero/reference.patch|reference.patch]]** contains this
  round only, as a diff against dev `ac083896`: 12 files, +457 / −406. It **applies cleanly to
  dev**.
  - Most of the line count is `GroupDetailScreen` re-indenting under the sheet wrapper. Its real
    change is about 40 lines (`git diff -w`).
  - Use it to check your work, or apply it with `git apply` if your branch is close to dev. This
    document is the spec.

**Rebased onto current dev (2026-11-12).** Since the 2026-11-09 build, dev changed three things
this handoff touches:
- **Set page:** dev added a "What's inside" `CardCarousel` between the blurb and the chat. It
  goes **inside** the opaque sheet, so it covers the image like everything else.
- **Collection grid:** the binder-grid fix (gutter gaps, base inset, no `flexWrap`) is already
  on dev. This handoff's opaque-row-gap change sits on top of it (see Styles).
- **Collection `cardWidth`:** dev's pixel-floored version stays as it is.

## Measurements

### Layers (frame 3393:168141, 402 wide)
| Layer | Before | After |
|---|---|---|
| Banner image | scrolled with the page | **pinned** behind the scroll |
| Top scrim (`scrimTop`, 0 → 120) | inside the banner, scrolled | pinned with the image, so the floating back/share circles stay legible |
| Bottom fade | `fadeHeroCollapsed`, bottom 120 only | **`fadeHeroFixed`, the full banner height** (from the top of the viewport to the banner's bottom edge); **scrolls** with the content |
| Content below the banner | transparent | opaque `background`, covering the pinned image |

### Banner heights (unchanged per page)
| Page | Banner | `FixedBanner` size |
|---|---|---|
| Tag (`SupportedTagValueScreen`) | 290 on 402 (`aspectRatio`) | `tag` |
| Set (`SetScreen`) | 290 on 402 | `tag` |
| Group (`GroupDetailScreen`) | 230 | `standard` |
| Collection (`CollectionDetailScreen`) | 230 | `standard` |
| Profile (`ProfileScreen`, `UserProfileScreen` via `ProfileHero`) | 230 | `standard` |

### The gradient: new token `fadeHeroFixed`
Add it to `effects.gradients` in **both** `src/styles/tokens/dark.json` and `light.json`, next to
`fadeHeroCollapsed`. All stops are `background` at the given alpha:

| Position down the banner | 0 | 15% | 30% | 45% | 60% | 75% | 90% | 100% |
|---|---|---|---|---|---|---|---|---|
| Alpha | 0 | 0 | 0.08 | 0.28 | 0.55 | 0.80 | 0.97 | 1 |

```json
"fadeHeroFixed": { "stops": [{ "color": "background", "alpha": 0, "at": 0 }, { "color": "background", "alpha": 0, "at": 0.15 }, { "color": "background", "alpha": 0.08, "at": 0.3 }, { "color": "background", "alpha": 0.28, "at": 0.45 }, { "color": "background", "alpha": 0.55, "at": 0.6 }, { "color": "background", "alpha": 0.8, "at": 0.75 }, { "color": "background", "alpha": 0.97, "at": 0.9 }, { "color": "background", "at": 1 }] },
```

**Why this curve:**
- **Clear top 15%.** The image reads, and the top scrim already handles the header buttons.
- **Eased S-curve through the middle.**
- **Lands on solid with no slope change.** `fadeHeroCollapsed`, stretched to full height, was
  still climbing steeply when it hit the opaque sheet. That left a faint seam, and the band just
  above the title was too light.

**Why a new token:** `fadeHeroCollapsed` still serves the entity screen's short bottom cap, so
don't edit it.

## How it's built

### The recipe: pinned layer + scrolling slot + opaque sheet
1. **Pinned layer:** `FixedBannerImage`, the image + top scrim at the banner's size.
   - PageLayout pages pass it as `fixedBackground`. That's an absolute layer behind the
     transparent ScrollView, zIndex 0 vs 2, and EntityScreen already uses it.
2. **Scrolling slot:** `FixedBannerFade`, the first thing in the scroll content.
   - It's transparent and banner-sized, so the title lands where it did.
   - It carries the full-height `fadeHeroFixed`.
3. **Opaque sheet:** everything after the slot sits on `detailBannerSheet`, which has a
   `background` fill and `flexGrow: 1`. Short pages paint solid to the bottom.

No banner means no pinned layer and no slot; the page renders as before
(`…HeaderBlockNoBanner`). Loading states are unchanged.

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
    <LinearGradient {...getGradientProps(theme, 'fadeHeroFixed')} style={cardStyles.detailBannerFade} />
  </View>
)
```

### Styles
**`cards.ts`**
```ts
// module scope: the 230 shared by the banner and its slot
const DETAIL_BANNER_HEIGHT = 230

collectionDetailBanner:       { …, height: DETAIL_BANNER_HEIGHT, … }   // was a raw 230
tagDetailBannerSpacer:        { width: '100%', aspectRatio: 402 / 290 },
collectionDetailBannerSpacer: { width: '100%', height: DETAIL_BANNER_HEIGHT },
detailBannerFade:             { position: 'absolute', top: 0, left: 0, right: 0, bottom: 0 },
detailBannerSheet:            { flexGrow: 1, backgroundColor: theme.colors.background },

// on top of dev's binder-grid version: the rows scroll over the pinned image,
// so the gap under each row must be opaque. Margin becomes padding, plus a fill.
collectionDetailGrid: { …, paddingBottom: theme.spacing.gutter /* was marginBottom */,
                        backgroundColor: theme.colors.background, … }
```

**`profile.ts`**
- `profileHeroBlock` gains `backgroundColor: theme.colors.background`.
  - **It doesn't grow.** If it did, two growing fills on a short page would split the spare
    space and open a gap.
- New `profileSheet`: `{ flexGrow: 1, backgroundColor: theme.colors.background }`.

### Per page
**Tag, set, group** (PageLayout):
```tsx
<PageLayout … fixedBackground={x.banner ? <FixedBannerImage uri={x.banner} size="tag|standard" /> : undefined}>
  {x.banner ? <FixedBannerFade size="tag|standard" /> : null}
  <View style={cardStyles.detailBannerSheet}>
    {/* header block → rails / member row / items → chat, unchanged */}
  </View>
  {/* group: GroupAddItemsSheet / ActionModal stay OUTSIDE the sheet, as before */}
</PageLayout>
```
- Remove each screen's local `bannerFade` / `bannerScrim`, and any `LinearGradient`,
  `getGradientProps` and `getImage` imports that become unused. The group screen still uses
  `Image` and `getImage` for avatars.
- **Set:** the "What's inside" rail goes inside the sheet, between the title block and the chat.

**Collection** (FlatList, no PageLayout):
- **Pinned layer:**
  - It's the **first child** of `screenContainer`, wrapped in
    `layoutStyles.pageFixedBackgroundContainer` (absolute fill) with `pointerEvents="none"`.
  - The floating `Header` (zIndex 10) stays above it, and the transparent FlatList scrolls
    over it.
  - Render it **only when the grid is up** (`banner && !(isLoading && items.length === 0)`),
    so the loading state stays plain.
- **List header:** `listHeader` = `FixedBannerFade` + the title/count block wrapped in
  `detailBannerSheet`.
- **Grid rows:** made opaque by the `collectionDetailGrid` change. This screen is its only
  user.

**Profile** (`ProfileHero`, shared by `ProfileScreen` + `UserProfileScreen`):
- **Both screens:** pass
  `fixedBackground={profile.banner ? <FixedBannerImage uri={profile.banner} /> : undefined}`.
- **`ProfileHero`:**
  - Swap its banner block for `{banner ? <FixedBannerFade /> : null}`.
  - Keep the `banner` prop: it still drives the avatar overlap and the no-banner padding.
  - Drop its `getCardStyles`, `getGradientProps`, `LinearGradient` and `Image` imports.
- **The avatar overlap stays.** `profileHeroAvatarOverlap` (−84) still pulls the avatar up onto
  the fade, over the pinned image.
- **Own profile (`ProfileScreen`):** wrap the friends row and tabs in `profileSheet`.
- **Someone else's profile (`UserProfileScreen`):**
  - `profileStyles.root` paints the background, and it used to wrap the hero too, which would
    hide the pinned image.
  - Move `<ProfileHero>` **out of** `root`, directly above it. `root` now holds only the tabs
    and is their opaque sheet.

## Code map (dev @ `ac083896`)
- **New:** `src/components/detail/FixedBanner/index.tsx`
- `src/styles/tokens/dark.json`, `light.json`: new `fadeHeroFixed` gradient.
- `src/screens/tags/SupportedTagValueScreen/index.tsx`
- `src/screens/sets/SetScreen/index.tsx`: includes dev's "What's inside" rail inside the sheet.
- `src/screens/groups/GroupDetailScreen/index.tsx`: mostly re-indentation; about 40 real lines.
- `src/screens/collections/CollectionDetailScreen/index.tsx`
- `src/components/profile/ProfileHero/index.tsx`
- `src/screens/profile/ProfileScreen/index.tsx`
- `src/screens/profile/UserProfileScreen/index.tsx`
- `src/styles/components/cards.ts`: the constant, 4 new classes, the `collectionDetailBanner`
  height and `collectionDetailGrid`.
- `src/styles/components/profile.ts`: `profileHeroBlock` background and the new `profileSheet`.
- **Read only:** `src/components/global/PageLayout/index.tsx`. `fixedBackground` already
  exists.
- **Not touched:** `typography.ts` (the description change is dropped), `ShareTagValueCard`,
  `fadeHeroCollapsed`.

## QA
- **All six pages with a banner:**
  - **At rest:** the top of the image is clear. It dissolves into the page through the lower
    half, with **no visible line** where the solid page begins.
  - **While scrolling:** the image stays put, the content slides up over it, and the image
    fades out behind the gradient.
- **iOS pull-down past the top:** the image stays fixed while the content drags down over it.
  This is expected; see "Not in this handoff".
- **All six without a banner:** unchanged.
- **Short pages** (few posts or items): solid to the bottom, with no image showing below the
  content.
- **Set:** the "What's inside" rail covers the image as it scrolls up.
- **Group:**
  - The member row, the Buy & Sell rail and the chat all cover the image.
  - The add-items sheet and the action modal still open.
- **Collection:**
  - The gaps between grid rows stay solid over the image, and card shadows look right on the
    row fill.
  - The loading state has no banner.
  - The search overlay and footer still work.
- **Profile (own + someone else's):**
  - The avatar still overlaps the banner's bottom edge, over the pinned image.
  - The friends row (own) and the tabs cover the image as they scroll.
  - The avatar viewer (someone else's) and Edit profile / gear (own) still work.
  - No gap between the hero and the friends row on a short page.
- **Entity screen hero:** unchanged, still `fadeHeroCollapsed`.

## Open questions
None.

## Prompt
```text
Implement the pre-dev handoff
nanza-vault/nanza-figma/handoffs/pre-dev/2026-11-12-tag-page-fixed-hero.md (read it in full).
reference.patch in its assets folder is the designer's build of exactly this against dev
ac083896 and applies cleanly; use it to check your work. Figma 7c3Zz4nDUOutQG1BsnRRAw, frame
3393:168141.

1. TOKEN: add the fadeHeroFixed gradient (stops in the handoff) to dark.json AND light.json.
   Do not edit fadeHeroCollapsed.
2. COMPONENT: new components/detail/FixedBanner: FixedBannerImage (image + scrimTop, pinned) and
   FixedBannerFade (transparent banner-sized slot, full-height fadeHeroFixed; it scrolls).
3. STYLES: cards.ts DETAIL_BANNER_HEIGHT + tagDetailBannerSpacer, collectionDetailBannerSpacer,
   detailBannerFade, detailBannerSheet; collectionDetailGrid row gap margin -> padding +
   background (on top of dev's binder-grid version). profile.ts: profileHeroBlock background
   (non-growing), new profileSheet.
4. SCREENS (each keeps its banner height; tag/set 290-on-402, others 230):
   - Tag, set, group: image via PageLayout fixedBackground (do not edit PageLayout); slot first;
     everything after on detailBannerSheet. Set's "What's inside" rail goes inside the sheet.
   - Collection: pinned layer first in the screen container (pageFixedBackgroundContainer, only
     when the grid is up); slot + sheet-wrapped header block in listHeader.
   - Profile: ProfileHero renders the slot and keeps the -84 avatar overlap; ProfileScreen
     wraps friends + tabs in profileSheet; UserProfileScreen moves ProfileHero out of (above)
     profileStyles.root.
   No banner = no layer, no slot.
Do NOT touch typography.ts / tagDetailDescription, ShareTagValueCard, or scroll bounce.

House rules (CLAUDE.md): plan first with confidence scores; this touches 12 files, so escalate
the file list in your plan. No inline styles, raw sizes or colours. Run /ui-review before
finishing, then set this handoff to built and add a Log line.
```

## Log
- 2026-11-09: handoff written for the tag page (fixed hero + bio-style description), then
  widened to set, group, collection and profile via `FixedBanner`. Built and device-tested on
  `throwaway-fixed-heros`.
- 2026-11-12: moved to **pre-dev** and rebuilt on dev `ac083896` (`throwaway-hue-change`):
  - The description restyle is dropped.
  - Rebased onto dev's "What's inside" rail and the binder-grid fix.
  - iOS pull-down prevention was considered and left out.
- 2026-11-12: new `fadeHeroFixed` gradient (eased, lands on solid, clear top 15%) replaces the
  stretched `fadeHeroCollapsed` on the fixed hero. Device-tested and approved.
- 2026-11-12: **ready for dev.** `reference.patch` exported (applies cleanly to dev). ESLint
  and the full `tsc` run are clean.
