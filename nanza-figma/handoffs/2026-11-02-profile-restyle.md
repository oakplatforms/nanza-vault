---
tags: [nanza-figma, handoff, nanza-mobile, profile, posts, trades, tabs]
status: built
date: 2026-11-02
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3082-121992&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3082:121992
nodes:
  - 3082:121992  # Binders tab
  - 3064:81234   # Posts tab
  - 3064:78154   # Trades tab
  - 3143:198149  # Refs tab
frame: Profile / default · Seller · edit me (+ Posts, Trades, Refs states)
repo: nanza-mobile
scope: styles-only
---

# Profile — restyle (all tabs) + app-wide post, trade-tile and tab-counter styles

> [!warning] App-wide + data — read first
> This is a **styles-only** handoff. The dev team has already wired the profile page (tabs, Refs data, create entries, name fields) on their branch. **Don't change data, behaviour, navigation or structure**, except for the two data flags below.
>
> Three changes in here are **app-wide, not profile-only**. Each one updates the shared component's *default* so every surface picks it up, with no profile-scoped variants:
> - **A. Post rows** (`PostItem` / `PostThread`): indented content, spacing, footer button alignment, colours.
> - **B. Trade tiles** (`ItemThumb` via TradeFeedThumb / ListingThumb / BidThumb): new card-number line, WTB/WTS pills, text and spacing.
> - **C. Tab counter badge** (`SegmentedTabs` count): one counter style everywhere. The older `countBadge` is an outlier to fold in.
>
> **Data the dev team may need to wire:**
> 1. **Trade tile card number** (`ARR001`) = `entity.product.number`. ItemThumb has no prop for it, and it isn't confirmed that the `/trades` feed or the listing/bid payloads include `product.number`. Wire the prop and payload (and the API if needed). Hide the line when it's null.
> 2. **Avatar play badge + ring** on the profile hero: reuse the `CircleThumb` ring + `playBadge` from the home affiliate carousel, and wire this profile's featured Short and seen state.
> 3. **Flag action in the post footer** (reply · heart · **flag** · share). If report/flag isn't in the full footer on your branch, wire it to the existing report flow.

> [!note] Local code may lag
> File paths and class names below come from local `dev`, which still shows the older stacked-sections profile. Apply each style to the matching component **on your branch**. Captured Figma layer names (`proposed.rangeTrack`, `cards.thumbCardFooterTile`, `posts.commentFooter`, …) are this repo's style classes.

## Principle: universal, no outliers
A style changed once applies everywhere. Edit shared defaults, list the consumers for QA, and where a second class or component draws the same pattern, **point it at the canonical style** rather than restyling both. Only the hero lockup, the Edit profile pill, the Add-new-binder tile and the ReviewCard are profile-scoped, and even those use shared tokens and typography slots only.

## Request (verbatim)
**Changed (designer notes):**
- Assumes display name is present, if no display name display the username as title – need to decide whether to use the display name/username lockup with or without backend field to prime user for when display name can be set in the future. If not, show username only and remove the '@username' displayed beneath
- Edit profile will become 'Add friend' and 'Message' once friends on public profiles
- 'Create new' for posts still WIP, currently showing text input to link to compose screen
- 'Create new' for trades still WIP, currently showing primary action button 'Trade'
- Missing binder/collection thumbs without banner/thumbs uploaded – card style will mirror the 'Add new binder' thumb with the text placement same as the Library/Binder thumb
- Binder/collection thumb items counter not styled
- Missing social icons, update to come

**Added in review:**
- The way posts are displayed in a feed (indented post content, spacing, button alignment, colour) applies across the app.
- The way trades (bid and listing cards) are displayed applies across the app: the card number, the WTB/WTS pills, and the rest of the text and spacing on the card.
- The tab bar's counter style applies universally.

**Decisions:** name lockup = **always show both** (title `getProfileName(profile)` plus an `@username` line, even when they match). Applies to **both** ProfileScreen and UserProfileScreen. The own-profile mocks lead, and the owner-only pieces are gated by the existing wiring.

**Do not change:** data, queries, navigation, tab order and behaviour, or anything not listed here.

## Measurements
Frames are 402 wide, page edge 15 (content 372), unless noted.

### Hero (profile-scoped, `ProfileHero`)
| Element | Spec |
|---|---|
| Banner | Unchanged (scrimTop + fadeHeroCollapsed). The hero block overlaps the banner by 84. |
| Avatar row | Row, `space-between`, inset 15. Avatar left, actions right. |
| Avatar | **132** outer, 4.5 ring `inputBorder`/`border`, an 8 `background` gap ring, image **114**. ≈ 132/362 = **0.365** of content width; check `thumbSizes.ts`. |
| Avatar badge | Centered under the avatar, overlapping its bottom (top 111). 38×26, radius 11.4, `inputBorder` fill, 4 `background` border, 10pt play glyph. **Carries over from the home affiliate carousel**: `CircleThumb`'s `ring` + `playBadge` ('brand' = unseen Short, pink; 'quiet' = seen, grey), as used in `AffiliateCircleCarousel`. Reuse that badge and ring logic (scaled to the 132 avatar; CircleThumb already sizes the badge as a fraction of the outer circle). The mock shows the 'quiet' grey state. **The dev team wires the data** (featured Short / seen state for this profile). Not part of the social links. |
| Social links row | **Below the bio, above the friends row** (moved from above the bio). 14 below the bio. Row of **40pt circles**, `glass` fill + GlassRim, radius max, **gap 12**. Logo 18 at the 11 inset, **85% opacity** (the existing `profile.socialLinkLogoDim`). 4 bottom padding, so 18 to the friends row. Restyle the existing `SocialLinksRow` / `socialLinkCircle`; don't build a new one. Renders nothing when the profile has no links, as today. |
| Edit profile | Text button: padding 10×12, radius 8, fill `glassBorder`. Text `pill` slot (SemiBold 13/14 +0.11), `textSecondary`. |
| Settings | Square icon button: padding 9, radius 8, fill `glassBorder`, 16pt gear. Gap 6 after Edit profile. |
| Title | `jumbo` (Bold 32/40), `overlayText`. |
| @username | 6 below the title. `tileLabel` (SemiBold 13/16 +0.11), `metaText`. Always shown. |
| Bio | 16 below the name block. `postBodySm` (Balto 14.5/21), `onSurface`. |
| Friends row | 14 below the hero (18 when the social row is present). The count is whatever the profile data sends (the mocks' "7" vs "21" is a typo). Facepile of 24pt avatars, 2.5 `background` border, −6 overlap. Gap 8 to the count, which is `thumbLabel` (Medium 12/14), `secondaryText`. |
| Hero → tabs | 23. |
| Header | Floating glass circles 44, `glass` fill + GlassRim. Left: chevron-down 24. Right: **share only** (16 glyph). Edit and gear moved into the hero. |

### C. Tab track + counter (app-wide, `SegmentedTabs` / `proposed.rangeTrack`)
| Element | Spec |
|---|---|
| Track | 372×43.2, `background` fill, 1.1 `glassBorder` border, radius max, padding 3. |
| Segment | Height 35, radius 17.5, equal flex. Label `pill` (SemiBold 13/14 +0.11). |
| Inactive | No fill, label `secondaryText`. |
| Active | Fill `glassBorder`, label `textSecondary`. |
| Counter | 5 after the label. Padding 0×6, bottom 2. Text `small` (SemiBold 12/16 +0.11). **Inactive:** `glassBorder` chip, `overlayText` text. **Active: light chip, dark text** (`onSurface` fill, `background` text; answered, Q5). Radius: Figma has 30 in two frames and 8 in one, so use **max** (pill). |
| Tabs → content | **24 on every tab** (answered, Q8). The mocks' 12 / 8 / 23 are artefacts of component padding; measure the *visible* gap and make it 24. |

Today `segmentedTabCount` is an `ink` circle with `micro` text, the same in both states. It becomes the spec above. **Outlier:** `tabs.ts` `countBadge` / `countText` / `hasCountBadge` (used by `components/global/Tabs/{Default,Text,Mixed}.tsx`) is a second counter look (`secondary.700`, `bold`). Point it at the same spec or share the class.

### Binders tab
| Element | Spec |
|---|---|
| Grid | 2 columns, gap 12, inset 15. Tile 180 wide (= (372 − 12) / 2), ≈228.7 tall, radius 16, 1 top hairline `glassBorder`. |
| Binder tile | **This frame wins** over [[2026-10-25-collection-card-styles]] (answered, Q1). Mosaic + ThumbShade (`surface` 0 → 0.84, full height). The eye-off "value hidden" and "–" "value missing" *states* from 10-25 still apply, in this pill's style. |
| Name | Bottom-left, 12 inset. **`bold` (SemiBold 14/16 +0.11)**, `textSecondary`, 1 line. |
| Value pill | 6 below the name, about 12 from the bottom. Fill **`background`**, padding 6×8, radius 8, gap 4. Amount Medium 12, `text`. "Market value" Medium 12/14 +0.11, `secondaryText`, tail ellipsis. The nearest slot is `thumbLabel` (Medium 12/14) for both. |
| "N items" counter | Top-left 12/12, 24 tall, `surface` fill, radius 6, 8 side padding, Regular 12 `secondaryText`. **The designer says it isn't final:** use this as a placeholder on the nearest 12pt slot and expect a follow-up. |
| Add new binder | Same tile box. Two shades: bottom `surface` 0 → 0.28@58% → 0.70@82% → 0.94; top `surface` 0.55 → 0.28@15% → 0@32%. Centered column, gap 8: a 17×16 plus glyph, then "Add new binder" in `pill` (SemiBold 13/14), `textSecondary`, centered. |
| Binder without thumbs | The **Add new binder** tile's look (shades, no mosaic) with the regular binder's text placement (count top-left, name + value bottom-left). |

### Posts tab
| Element | Spec |
|---|---|
| "Post a new update" | 372×44, radius 22, `glass` fill, 1 `glassBorder` top + bottom with the GlassRim. Placeholder at 20 left in `bodyInput` (Balto 16), `cardBackground`. The JoinChatFooter pill look. 24 below to the first post. Style it as shown (it's a WIP placeholder, Q4). |
| Post rows | See **A** below. |

### A. Post row (app-wide, `PostItem`)
Measured from PostItem `3143:169977`.
| Element | Spec |
|---|---|
| Row | Padding: top 8, bottom 4, sides 16. **Bottom** hairline 1 `surface`. Gap 8 between the content block and the footer. |
| Identity | Avatar **31**, gap 9, then name and time on one baseline (gap 3). Name `title` (Bold 16/16), `textSecondary`. Time `thumbLabel` (Medium 12/14), `secondaryText`. |
| Body | 6 below the identity. **Indented 40** (= 31 avatar + 9 gap, so it lines up with the name). **Every** PostItem gets it, including PostDetail and thread replies (answered, Q6). `postBody` (Balto 16/19), `text`. |
| Footer | **Indented 40** (same column), padding top 16, bottom 8, right 10. Gap 20 between actions. Reply 17pt · heart 16 + count (gap 8, `labelBold` SemiBold 13/16 −0.11, `metaText`) · **flag 16** · share 16. Icon colour `metaText`, or the existing footer icon token if it already matches. |

Side inset: the row uses 16, the rest of the page 15. Keep the shared PostItem default (the profile passes `inset="page"`). Don't add a profile-only inset.

### Trades tab
| Element | Spec |
|---|---|
| Trade button | Full width (372), padding 18, radius 28, fill `overlayText`. Label SemiBold 16/16 +0.11, `textAccent`. This is the primary action button; reuse the shared primary class if it matches. 12 below to the grid. Style it as shown (it's a WIP placeholder, Q4). |
| Grid | 2 columns, gap 12 (column and row), inset 15. |

### B. Trade tile (app-wide, `ItemThumb`)
Measured from ItemThumb `3143:197422`. **This supersedes [[2026-10-25-bid-listing-tiles]] where they conflict**; that handoff's backdrop, art and shade specs still hold.
| Element | Spec |
|---|---|
| Tile | 180 wide (column width), ≈187 tall. Radius 20, 1.33 top hairline `glassBorder`, `background` fill. |
| Backdrop / shades / art | As 10-25: image at opacity 0.61, top tint `rgba(133,150,172,0.22)` → 0, bottom shade 0 → 0.8@60%. Art at 12 from the top, 5:7, radius 6, centered. |
| Intent pill | Top-left 12 / 10. Padding 1×4, radius 6. Text SemiBold 11/16 −0.11. **WTB: `highlight` fill, dark-pink text. WTS: mint (`conditionColors.mint`) fill, dark-green text.** This **reverses the 10-25 swap** (answered, Q2: follow this frame). If your branch already applied the swap, undo it across every IntentTag variant (tile, row, compact). Text colours are unbound in Figma (`#510625`, `#052e1c`); use the existing `primary.950` / `tertiary.950`. |
| Footer | Padding 12, pinned to the bottom, width 158. Name block then 6 then price. |
| Name | `bold` (SemiBold 14/16 +0.11), `textSecondary`. Max 2 lines, tail ellipsis (as 10-25). |
| **Card number** | **New.** 2 below the name. Regular 11/14 +0.11, `overlayText` at 75% opacity. No exact slot: pick the nearest 11pt slot (`legal`/`micro`) and an `overlayText`-based colour token, and flag it in the plan. **Data: see the callout.** Hidden when null. |
| Price pill | Fill `backgroundSecondary`, padding 4 top / 5 bottom / 5 sides, radius 6. Text SemiBold 12/12, `background`. |
| Quantity | Not in this frame. Keep the 10-25 spec ("N in stock" / "N wanted", hidden at ≤1). |

### Refs tab (profile-scoped, new `ReviewCard`)
| Element | Spec |
|---|---|
| Card | Full width, padding 20, bottom hairline 1 `surface`, fill `background`. Gap 20 between the content and the reviewer row. 8 between cards. The tabs → first card gap is 24, like every tab. |
| Stars | 5 × 18 (90×18), filled colour **`Condition/Fair` (#ffe566)**. Use the existing `conditionColors` token. |
| Body | 16 below the stars. `postBody` (Balto 16/19), `text`. Keep line breaks. |
| Reviewer | Avatar 24, gap 9, name `bold` (SemiBold 14/16), `textSecondary`. Time `thumbLabel`, `secondaryText`, gap 3, same baseline. |

## Code map (local `dev`; match these on your branch)
- `src/components/profile/ProfileHero/index.tsx` (shared by ProfileScreen and UserProfileScreen): avatar, badge, actions, title + @username, bio. Styles in `src/styles/components/profile.ts`, title in `detail.ts` `detailTitleHero`.
- `src/screens/profile/ProfileScreen/index.tsx`: header right icons; friends row (`cards.ts` `groupDetailAvatar*`, typography `groupDetailMemberCount`).
- `src/components/global/SegmentedTabs/index.tsx` and `src/styles/components/tabs.ts` (`segmentedTab*`, `segmentedTabCount*`). **Consumers:** EntityScreen, TradeScreen, MyTradesScreen, CreateTradeScreen, MessagesScreen, GroupMembersScreen, the profile.
- **Outlier:** `tabs.ts` `countBadge` / `countText` / `hasCountBadge`, used by `components/global/Tabs/{Default,Text,Mixed}.tsx`.
- `src/components/posts/PostItem.tsx` and the `posts.*` style classes (`postIdentityRow`, `commentFooter`). **Consumers:** PostThread (home WhatsNewSection, WhatsNewScreen, DetailChatSection), PostDetailScreen, PostPager / StoryPostPage, MyShortScreen, ProfileSeeAllScreen, ProfileSections, SharePostCard, PostComposer.
- `src/components/cards/ItemThumb/index.tsx`, `src/components/global/IntentTag` with `src/styles/components/badges.ts` (`intentTag*`), and `cards.ts` (`thumbCard*`, `thumbIntentCorner`, `thumbPricePill`). **Consumers:** TradeFeedThumb, ListingThumb, BidThumb, BulkThumb, TradesGrid, EntityTradesTab, GroupDetailScreen, GroupSeeAllScreen, FeedGridScreen, ProfileSections, ProfileSeeAllScreen, OfferSellerListingsPicker, ElbPickerLayer. IntentTag's row and compact variants share the colours, so the colour rule is app-wide.
- `src/components/cards/CollectionThumb`: binder tile, Add-new and no-thumbs variants.
- `src/components/posts/composer/JoinChatFooter`: the "Post a new update" pill.
- `src/components/cards/thumbSizes.ts`: column widths; no new step expected (180 = the 2-up column).

## Open questions
Answered by the designer on 2026-11-02:
1. **Binder tile:** this capture differs from the 10-25 collection-card spec (name 14 vs 13, value pill).
   **Answer:** go with **this newer version** (name `bold` 14, `background` value pill). It supersedes 10-25 for the binder tile.
2. **WTB/WTS colours:** 10-25 swapped them; this frame shows WTB pink, WTS green.
   **Answer:** **follow this frame**: WTB pink (`highlight`), WTS green (mint). No swap, app-wide.
3. **Social links:** **Answer:** the social icons sit **below the bio, above the friends row** (frame 3082:121992, now updated). Spec in the hero table.
   **Avatar play badge (answered):** it's the play badge from the home card carousel (`CircleThumb` `playBadge` + ring in `AffiliateCircleCarousel`) carried over to the profile avatar. Build it by reusing that badge; the dev team gets it working (featured Short + seen state).
4. **"Post a new update" and "Trade" placeholders:** **Answer:** style them as shown.
5. **Active tab counter:** **Answer:** the active tab uses the **light chip** (dark text); inactive uses the dark chip.
6. **Post indent on replies:** **Answer:** yes, **every** PostItem, including PostDetail and thread replies.
7. **Card number when empty:** **Answer:** hide it (lots, sealed product, anything without a number).
8. **Tab bar → content gap:** **Answer:** **24 on every tab.** Where the mocks show 12, they're likely counting extra space inside a component; make the *visible* gap 24.
9. **Figma leftovers:** **Answer:** **don't build anything hidden** in Figma. The friends count ("7" vs "21") is a typo; show whatever the profile data sends.

## Prompt
```text
Restyle the profile page and three app-wide patterns from the handoff
nanza-vault/nanza-figma/handoffs/2026-11-02-profile-restyle.md (read it in full, including
the answers under Open questions). Figma file 7c3Zz4nDUOutQG1BsnRRAw, frames 3082:121992
(Binders), 3064:81234 (Posts), 3064:78154 (Trades), 3143:198149 (Refs).

SCOPE: STYLES ONLY. The page is already wired on your branch (tabs, Refs data, create
entries, display name). Do not change data, queries, navigation, tab order or behaviour.
Exceptions, wire only if missing: (1) the trade tile's card number = entity.product.number
(prop + payload/API), hidden when null; (2) the flag action in the post footer; (3) the avatar play badge + ring data (featured Short, seen state).
Do not port the Figma artifacts listed in Q9.

UNIVERSAL, NO OUTLIERS: edit shared defaults so every consumer updates. No profile-only
variants for posts, trade tiles or the tab counter. Where a second class draws the same
pattern, point it at the canonical style.

APP-WIDE
A. Post rows (PostItem, every feed): padding 8/4/16, bottom hairline surface; avatar 31,
   gap 9, name title/textSecondary, time thumbLabel/secondaryText; body and footer
   INDENTED 40 (aligned with the name); body postBody/text, 6 below the identity; footer
   pt16 pb8 pr10, gap 20: reply 17, heart 16 + count (gap 8, labelBold/metaText), flag 16,
   share 16. Every PostItem, replies included. QA every consumer in the handoff's code map.
B. Trade tiles (ItemThumb, supersedes 2026-10-25-bid-listing-tiles on conflict): 180 col,
   radius 20, top hairline glassBorder; intent pill top-left 12/10, pad 1x4, radius 6,
   SemiBold 11/16; WTB highlight (pink) fill, WTS mint (green) fill, NO swap; footer pad 12: name
   bold/textSecondary max 2 lines, NEW card number 2 below (11pt, overlayText ~75%),
   6 below that the price pill (backgroundSecondary, pad 4/5/5, radius 6, SemiBold 12,
   background). QA every consumer.
C. Tab counter (SegmentedTabs): gap 5 after the label, pad 0x6 (+2 bottom), radius max,
   small slot; inactive glassBorder chip + overlayText; active onSurface chip + background text.
   Fold tabs.ts countBadge/countText (Tabs/Default|Text|Mixed) into the same look.
   Track: 372x43, background fill, glassBorder 1.1, pad 3; segment h35, pill slot;
   inactive secondaryText; active glassBorder fill + textSecondary. Visible gap from the
   track to the tab content = 24 on every tab.

PROFILE
1. Hero: avatar 132 (ring 4.5 inputBorder, 8 background gap, image 114) left; Edit
   profile (pad 10x12, radius 8, glassBorder, pill/textSecondary) + gear (pad 9, radius
   8, 16 glyph), gap 6, right. Title jumbo/overlayText; @username 6 below,
   tileLabel/metaText, ALWAYS shown; bio postBodySm/onSurface 16 below; friends row 14
   below; SOCIAL LINKS ROW between bio and friends (40 glass circles, gap 12, 18 logo at
   85%, 14 below the bio, 18 to friends); friends row (24 avatars, -6 overlap,
   thumbLabel/secondaryText, count from data); 23 to the tabs. Header right = share circle
   only. Avatar play badge = reuse CircleThumb's ring + playBadge
   from AffiliateCircleCarousel (dev team wires featured-Short/seen data). Build nothing that's hidden in Figma.
2. Binders: 2-up grid gap 12, inset 15, tile 180, radius 16. Binder tile = THIS frame
   (supersedes 10-25): name bold 14/textSecondary, value pill background fill pad 6x8
   radius 8 Medium 12 (text + secondaryText "Market value"); items counter is a
   placeholder (not final). Add new binder tile and the no-thumbs binder share the shaded
   empty look (specs in the handoff).
3. Posts: "Post a new update" pill 372x44, radius 22, glass + glassBorder + GlassRim,
   bodyInput placeholder at 20; 24 to the first post. Rows = A.
4. Trades: full-width Trade button (pad 18, radius 28, overlayText fill, SemiBold 16
   textAccent), 12 to the grid. Tiles = B.
5. Refs: ReviewCard pad 20, bottom hairline surface, 8 between cards; stars 5x18
   (Condition/Fair token); body postBody/text 16 below; reviewer avatar 24, gap 9,
   bold/textSecondary + thumbLabel/secondaryText time.

House rules (CLAUDE.md): plan first with confidence scores. Styles go in
src/styles/components/* buckets; no inline styles, no raw sizes or colours, theme tokens
and typography slots only (compose tokens where none match, and say which). Size
proportionally from the 402 frame. This touches more than 5 files, so escalate the file
list in your plan. Run /ui-review before finishing, then set this handoff to built and
add a Log line.
```

## Log
- 2026-11-02 — handoff written (styles only; app-wide A/B/C flagged for the dev team)
- 2026-11-02 — designer answered Q1–9: binder tile from this frame, WTB pink / WTS green (no swap), social row below the bio (re-measured from the updated frame), tab gap 24, nothing hidden, friends count from data
- 2026-11-02 — avatar play badge answered: reuse the carousel's CircleThumb ring + playBadge; the dev team wires the data
- 2026-11-02 — built in nanza-mobile (uncommitted, pending device QA): hero on both profiles (132 avatar, affiliate-only play badge from the cached rail, Edit/Settings in hero, share-only header, @username, social row below bio), SegmentedTabs counter + track app-wide (15 inset; entity + create-trade capsules keep their pages' 20), PostItem rows app-wide (40 indent, bottom hairline, postBody back to 16/19), ItemThumb app-wide (card number, WTB highlight, price pill, 180/187), binder filled value pill + empty look, Refs card. Friends row on viewed profiles blocked: GET /connections is own-account only. tsc/eslint clean; /ui-review not run
- 2026-11-02 — device pass: bio 14.5 → 15.5 (profileBio slot, new size['15.5'] token), friends count thumbLabel 12 → labelLg 14, facepile 24 → 36 (overlap 8), @username tileLabel 13 → itemTitle 15, Edit profile pill 13 → bold 14, hero avatar 132 → 120 with a thinner, tighter ring (3.5 ring, 2 gap, 121/132 image; grey for everyone, pink/grey Short state for rail affiliates) and a grey placeholder disc when there is no photo. Handoff type reads ~1pt small; guideline added to nanza-figma INDEX
- 2026-11-02 — device pass: hero avatar ring line removed for now; the 121/132 photo sits on a black (`background`) backing disc; play badge unchanged
