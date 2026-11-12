---
tags: [nanza-figma, handoff, nanza-mobile, posts, cards, design-qa]
status: built
date: 2026-11-03
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3171-38851&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3171:38851
nodes: [3064:68432, 3171:38931, 3171:38861, 3126:145973, 3171:38851]
frame: PostItem profile=profile14 (+ HomeScreenContent, PostContentItems, posts.commentFooter, GroupBannerThumb profile=seller)
repo: nanza-mobile
---

# Design QA — play badge, inline entity cards, post footer, tile rims, multi-entity spacing

Five design-QA fixes in one handoff. Every frame is in the Nanza file and 402 wide where it is a
full screen.

## Request (verbatim)
**Changed:**
1. the play icon is too large in the circle carousel - 8px high with 1px border radius
   (node 3064-68432)
2. inline posts should match this sizing/spacing - the text size spacing etc should apply to all
   cards like this (in search etc). ignore the green price color for now (node 3171-38931)
3. group the share icon with the other icons for each post (node 3171-38861)
4. remove bottom border from groups and supportedtagvalue cards (nodes 3064-68432, 3126-145973)
5. look at this spacing between elements and match this when there are multiple entities inline
   in a post (node 3171-38851)

**Do not change:** not stated, so: anything else on these screens.

## Measurements

### 1. Circle carousel play badge (`HomeScreenContent` › `CardCarousel` › `CircleThumb` › `Group 2`)
| Part | Figma | Code today (`lg`, 88 outer) |
|---|---|---|
| Ring | 88 outer, 2.5 ring, 76 image | same |
| Badge fill | 24 × 15, radius ≈5.5, `highlight` | 29 × 20 outer incl. 2 `background` rim → 25 × 16 fill, radius 0.38 × h |
| Badge rim | 3 `background` outside stroke → 30 × 21 outer | 2 |
| **Play glyph** | **7.32 × 8, corner radius 1**, fill `#121212` (= `surface` = `ink`, neutral.900) | `play` icon at `playBadgeHeight × 11/20` = **11pt**, sharp corners |
| Glyph position | x 9 → 16.3 in the 24 fill (centre 12.65, 0.65 right of centre); vertically centred | left pad = glyph / 3 (centroid rule) |

Only the glyph was called out. As a fraction: 8 / 88 of `outerSize`.

### 2. Inline entity card (`PostContentItems` › `EntityLockupRow`, 330 wide)
Card width 330 = 402 − 16 − 16 − 40 (the post's 40 content indent, already `postIndent`).

| Part | Figma | Code today |
|---|---|---|
| Card chrome | 1 `glassBorder` all round, radius **20** (`radius.lg`), padding **12** all sides, overflow clip | 1 `border`, radius `md` (16), padding 8 / 8 |
| Thumb → text gap | **16** | 20 (`spacing.base`) |
| Thumb (`EntityHeroThumb`) | 137.9 × 90 (aspect 1.532 ≈ `LOCKUP_HERO_ASPECT`), radius 10.27, top hairline `glassBorder`. 137.9 / 306 inner = **45.1%** | inline thumb from `posts.ts`; derived from `THUMB_WIDTH_PT / 354` |
| Text column | top-aligned (`items-start`), name block then price line with a **12** gap | `justifyContent: 'space-between'` |
| Name | `Nanza/bold` SemiBold 14/16 +0.11, `text` | `typography.bold`, matches |
| Name → meta group | **6** | `spacing.min + 1` = 3 |
| Number → set line | **3** | `spacing.min` = 2 |
| Number + set lines | Balto Book **13/14**, `metaText` | `metaSm` = Balto 13/**15** |
| Set line fade | `entity.metaScrollFade`, 20 wide | 20 (`spacing.base`), matches |
| Market pill | 1 `glassBorder`, radius 8, padding **5 / 6** (v / h), gap **5**, centre-aligned. Amount SemiBold 12/12 (green `price`, **ignore the colour**). "Market price" `pillSm` SemiBold 10.5 `secondaryText` | padding 5 top / 6 bottom / 8 sides, gap 4, baseline, `pillAmountSm` |

### 3. Post footer (`posts.commentFooter`, 370 wide)
- One left cluster, **gap 20** between actions: reply 17 · heart 16 + count (gap 8, `labelBold`,
  `metaText`) · flag 16 · **share 16**. Share sits 20 after flag. Nothing pushed to the right edge.
- Indent 40, right pad 10. Icons `metaText`.

### 4. Tag + group image tiles (`TagThumb` "Assassin/Brute", `GroupBannerThumb` "Lore Junkies")
- Figma: radius 28, fill `surface`, **top hairline only** (1 `glassBorder` = 11% white), no
  bottom edge.
- Code: no CSS border. The edge is `<GlassRim pill radius={28} tone="image" />`. Its gradient is
  bright at **both** top and bottom (stops 0.78 → 0.10, 1.0 → 0.26, × 0.45 for `image`). The
  bottom stops draw the "bottom border".

### 5. Multiple entities in one post (`PostItem` profile=profile14)
Measured on absolute frames (y within the post):

| Gap | Figma | Code today (trace to confirm) |
|---|---|---|
| Entity card → entity card (**the only gap in scope**) | **4** (203 → 207) | `contentItems` gap 16 |
| Body text bottom → first entity card (out of scope) | 4 (83 → 87) | `postContent` gap 8 + `contentItemsCard` marginTop 8 (+ `contentItems` marginTop 2?) |
| Last card → footer glyph top | **4 + 13 = 17** (323 → footer 327, py 13) | `postContent` gap + `commentFooter` paddingTop 16 |
| Footer bottom pad | 13 (in this frame); text-only posts in the home frame use pt 16 / pb 8 | (control − icon.sm) / 2 = 14 |
| Avatar → name | 9, avatar 31; name `title` 16/16 `textSecondary`, time `thumbLabel` 12/14 `secondaryText`, gap 3, bottom-aligned | unchanged; already built per profile handoff |
| Identity → body | 6 | unchanged |

## Code map
- `src/components/cards/CircleThumb/index.tsx`: play badge math (`playGlyphSize`, `playGlyphCentroidPad`, lines ~155–166) and the `<Icon name="play">`. **Consumers:** AffiliateCircleCarousel (`lg` + `playBadge`), ProfileHero (`hero`), TagCarousel / TagThumb (`sm`, no badge). The `'play'` icon (`assets/icons/play.svg`, sharp 7.32 × 8) has **one consumer**: this component (`lg` and `hero`).
- `src/styles/components/cards.ts` › `circleThumbPlayBadge*`: badge fill / rim (unchanged unless Q3 says otherwise).
- `src/components/entity/EntityLockupRow/index.tsx` + `src/styles/components/entity.ts` (`lockupChromeInline`, `lockupContent*`, `lockupBody`, `lockupIdentity`, `lockupMetaGroup`, `lockupMeta`, `lockupPriceLine`, `fairMarketPill*`). **Consumers:** PostContentItems (`chrome="inline"`), SearchEntityRow, EntityCardBase, TradeListItem, BulkSaleCard, OfferLineLockup (all `chrome="row"`).
- `src/components/entity/FairMarketPill/index.tsx`. **Consumers:** EntityLockupRow, CollectionThumb, QuantityModal, ScanAddModal, DetailTitleBlock, MarketPriceLabel.
- `src/components/posts/PostContentItems.tsx` + `src/components/posts/EntityHeroThumb.tsx` + `posts.ts` (`entityHeroBody`, `contentItems`, `contentItemsCard`, `contentItemsNarrowGap`, `contentItemsDetailTree`): the inline entity card and the stack of references.
- `src/styles/components/typography.ts` › `metaSm` (Balto 13/15). **Also used by** `messages.ts:495` and `inputs.ts:1699`.
- `src/components/posts/PostItem.tsx` (~line 562) + `posts.ts` › `footerShare` (`marginLeft: 'auto'`). `footerShare` has one consumer. **PostItem consumers:** PostThread (home WhatsNewSection, WhatsNewScreen, DetailChatSection), PostDetailScreen, PostPager / StoryPostPage, MyShortScreen, ProfileSeeAllScreen, ProfileSections, SharePostCard, PostComposer.
- `src/components/global/GlassRim/index.tsx`: rim gradient. `tone="image"` has **three** consumers: TagThumb, GroupBannerThumb, GroupThumb (GroupsListScreen grid). `GlassRim` overall has ~30 consumers (header circles, bars, sheets, modals); they must not change.
- `src/components/cards/TagThumb/index.tsx` (TagCarousel → SupportedTagValue rails), `src/components/cards/GroupBannerThumb/index.tsx` (GroupBannerCarousel).

## Open questions
1. **Item 2 scope: "all cards like this (in search etc)".** Proposal: the **type and inner
   gaps** become the shared lockup defaults for every `EntityLockupRow`, so search, trade and
   offers get them too. That covers name → meta 6, number → set 3, meta 13/14, meta → pill 12
   with the body top-aligned, and the market pill's padding, gap and type. The **inline chrome**
   stays inline-only, because row chrome is a bordered list row rather than a card: padding 12,
   gap 16, radius 20, `glassBorder`, thumb 45%. Is that the split you want?
   **Answer (2026-11-03):** yes, follow the proposal. Type + inner gaps + market pill are shared
   lockup defaults app-wide; the outline/chrome stays posts-only.
2. **Meta 13/14 slot.** No Balto 13/14 slot exists. `metaSm` is 13/15 and is also used by the
   messages and inputs styles. Retune `metaSm` to 13/14 app-wide, or mint a new lockup meta slot?
   **Answer:** change `metaSm` to 13/14 app-wide (messages + inputs follow; list them for QA).
3. **Item 1: badge vs glyph.** The note says only the glyph (8 high, radius 1). The Figma badge
   also differs slightly: a 24 × 15 fill with a 3 rim, against code's 25 × 16 fill with a 2 rim.
   Glyph only, or match the badge too? Also: the rounded glyph needs a new asset. Edit `play.svg`
   in place, which also changes the profile hero badge (its only other user)? Or add a new
   `play-rounded` icon for `lg` only?
   **Answer:** edit `play.svg` in place (1pt rounded corners) so the glyph is always consistent,
   carousel and profile hero alike. Badge itself: glyph-only change, badge size/rim unchanged.
4. **Item 4: which group cards?** The Figma covers `GroupBannerThumb`. `GroupThumb` (the
   GroupsListScreen grid) uses the same `tone="image"` rim. Proposal: make `tone="image"`
   top-only, which fixes all three image tiles at once and leaves the other ~30 GlassRim uses
   alone. Or scope it to TagThumb + GroupBannerThumb only?
   **Answer:** add a **new, separate GlassRim style** for full-bleed image cards: top rim only.
   Buttons, bars, circles and sheets keep the existing top + bottom rim. Every full-bleed image
   card uses the new style: **collection cards** (`CollectionThumb`, which today draws a CSS
   `borderTopWidth: 1` glassBorder in `cards.ts collectionThumbCard` and should move onto the new
   rim), **SupportedTagValue cards** (`TagThumb`) and **group cards** (`GroupBannerThumb`,
   `GroupThumb`).
5. **Item 5: footer pad.** In this frame, the gap from the last entity to the footer is 4 + 13 =
   17, with 13 below. Text-only posts use 16 / 8 (the profile handoff). Keep one footer spec of
   16 / 8 everywhere, which puts entity posts within 1pt of the frame? Or treat 13 / 13 as
   deliberate when a post ends in an entity?
   **Answer:** item 5 is about the spacing **between entity cards only** (card → card 4). Leave
   body → card and card → footer spacing and the footer padding as they are.

## Prompt
```text
Design QA — five fixes from Figma (file 7c3Zz4nDUOutQG1BsnRRAw). Handoff with full
measurements and answered questions: nanza-vault/nanza-figma/handoffs/2026-11-03-design-qa-posts-cards.md
— read its Open questions section first and follow any answers written there.

SCOPE: exactly the five changes below. Change nothing else on these screens. Ignore the green
`price` colour on the market pill (keep the current amount colour). Do not port Figma artifacts:
the hex `#bdc9d3` on the post timestamp is `secondaryText`, and `#121212` on the play glyph is
`ink`/`surface`.

1. "the play icon is too large in the circle carousel - 8px high with 1px border radius"
   (node 3064:68432, CircleThumb 'lg' + playBadge, home AffiliateCircleCarousel)
   - Glyph 8 tall (7.32 x 8) at the 88 outer ring → size = outerSize * 8/88, not
     playBadgeHeight * 11/20 (11pt today). Corner radius 1 on the triangle.
   - Figma seats the glyph 0.65 right of the fill's centre; keep the existing centroid-padding
     rule if it lands within ~0.5pt, otherwise retune the ratio.
   - 'hero' keeps its own HERO_BADGE_GLYPH_RATIO. Badge fill/rim unchanged unless Q3 says so.
   - Rounded glyph (Q3 answered): edit assets/icons/play.svg in place — same 7.32 x 8 triangle,
     1pt rounded corners (scaled in the 8x8 viewBox). Only consumer is CircleThumb, so the
     profile hero badge picks it up too; that is intended.

2. "inline posts should match this sizing/spacing - the text size spacing etc should apply to
   all cards like this (in search etc)" (node 3171:38931, EntityLockupRow)
   Shared lockup defaults (all EntityLockupRow consumers — Q1 answered: yes):
   - body column top-aligned; name block → price line gap 12 (replace space-between)
   - name: typography.bold (14/16) — already correct
   - name → meta group 6; number → set line 3
   - number + set lines Balto 13/14 metaText — retune typography.metaSm to 13/14 app-wide
     (Q2 answered); messages.ts and inputs.ts follow, list them for QA
   - FairMarketPill: padding 5 vertical / 6 horizontal, gap 5, centre-aligned, radius 8,
     1 glassBorder; amount SemiBold 12/12, label pillSm 10.5 secondaryText. FairMarketPill has 6
     other consumers — list them for QA.
   Inline chrome only (chrome="inline", the post card, 330 wide = 402 − 32 − 40 indent):
   - padding 12 all sides, thumb → text gap 16, radius 20 (radius.lg), border glassBorder
   - EntityHeroThumb 137.9 x 90 → 137.9/306 = 45.1% of the card's inner width, aspect stays
     LOCKUP_HERO_ASPECT; radius 10 token, top hairline glassBorder.
   Keep row chrome's insets, thumb width and floors as they are.

3. "group the share icon with the other icons for each post" (node 3171:38861)
   - Drop the right-push on share (posts.ts footerShare marginLeft 'auto'); share joins the left
     cluster after flag at the row's 20 gap: reply · heart+count · flag · share.
   - When flag is hidden (own post / guest) share simply follows the heart. Remove footerShare if
     it is left with no properties (zero-consumer check first).

4. "remove bottom border from groups and supportedtagvalue cards" (nodes 3064:68432, 3126:145973)
   - The edge is GlassRim tone="image" (TagThumb, GroupBannerThumb, GroupThumb). Its gradient
     lights top AND bottom; Figma keeps only the top hairline (11% white).
   - Q4 answered: add a NEW GlassRim style for full-bleed image cards — top rim only (top stops
     kept, bottom stops zero). Additive: the existing default/'image' behaviour for buttons, bars,
     circles and sheets (~30 consumers) keeps top + bottom. Name it for the surface (e.g. an
     'imageCard' tone or an `edge="top"` option — pick in the plan).
   - Every full-bleed image card uses it: TagThumb (SupportedTagValue), GroupBannerThumb,
     GroupThumb, and CollectionThumb. CollectionThumb has no GlassRim today — its edge is a CSS
     borderTopWidth 1 glassBorder on cards.ts collectionThumbCard; move it onto the new rim
     (radius = its radius.md) and drop the CSS top border so the collection card matches the
     others.

5. "look at this spacing between elements and match this when there are multiple entities inline
   in a post" (node 3171:38851)
   - Entity card → entity card: 4 (Figma 203 → 207). That is the ONLY gap that changes (Q5
     answered). Body → first card, last card → footer, and the footer padding stay as they are.
   - Today the stacked references are spaced by posts.ts contentItems gap 16. Trace whether
     that gap also spaces non-entity blocks (images, galleries, link/embed leaves, text) — if it
     does, scope the 4 to consecutive entity cards only and say how in the plan.

COUPLED FILES: CircleThumb + cards.ts + (play.svg | iconMap); EntityLockupRow + entity.ts +
FairMarketPill + typography.ts; PostContentItems + EntityHeroThumb + posts.ts; PostItem;
GlassRim + TagThumb / GroupBannerThumb / GroupThumb / CollectionThumb + cards.ts. That is more
than 5 files — escalate per CLAUDE.md and list consumers for QA (search, trade, offer line,
collection thumbs, detail title, scan/quantity modals, messages + inputs (metaSm), groups list,
tag rails, home affiliate rail, profile hero).

CONFIRM WITH THE DEV TEAM BEFORE IMPLEMENTING: the answers in the handoff are the design
decisions, not implementation sign-off. In the plan, present items 1–5 one at a time — the
proposed implementation, the files touched, the consumers affected, confidence score — and get
the dev team's confirmation on EACH item before writing its code. Do not batch-approve.
Items that need explicit dev confirmation in particular:
- 1: editing play.svg in place (changes the profile hero badge too).
- 2: moving the lockup type/gaps and FairMarketPill to app-wide defaults (search, trade,
  offers, collection thumb, detail title, modals); retuning metaSm app-wide (messages, inputs).
- 3: removing footerShare's right-push (every PostItem surface).
- 4: the new GlassRim style — its API shape (tone vs edge option), and moving CollectionThumb
  off its CSS top border onto it.
- 5: how the card → card 4 is scoped so other content blocks keep their 16.

TRAPS / DECISIONS:
- Q1–Q5 are answered in the handoff (design intent) — still confirm each with the dev team.
- The 2026-10-25 entity-card handoff set name 13/16 → meta 12/14, gaps 3/2, pill 8 below; this
  frame supersedes it (14/16, 13/14, 6/3, 12).
- The sharp `play` glyph was a deliberate swap on 2026-10-26 (Skylar) away from the rounded
  play-solid — the new 1pt radius is a small round, not a return to play-solid.

HOUSE RULES: plan first with confidence scores (min 3, alternatives under 0.8); proportional
sizing from the 402 frame (fractions, not raw points); no inline style objects (runtime
dimensions only); no raw sizes — theme spacing/radius/icon tokens, composed where none match;
theme.colors.* only; text via typography slots only; no navigation or contexts changes; lint
src + tsc clean; run /ui-review before finishing.
```

## Log
- 2026-11-03 — handoff written (five frames read over Dev Mode MCP curl bridge; GlassRim traced as the tile bottom edge)
- 2026-11-03 — Q1–Q5 answered: lockup type/gaps app-wide with posts-only outline; metaSm → 13/14 app-wide; play.svg edited in place; new top-only GlassRim style for all full-bleed image cards (tag, group, collection); item 5 = card → card gap only. Prompt updated.
- 2026-11-03 — answers are design intent only; prompt now requires dev-team confirmation per item before implementation.
- 2026-11-03 — built in nanza-mobile (uncommitted, pending device QA). 1: play.svg rounded in place, `lg` glyph = outerSize × 8/88 with a glyph/6 centroid pad (hero keeps /3 and its ratio). 2: shared lockup body top-aligned, name→meta 6, number→set 3, price line 12 under; `metaSm` 13/14 app-wide (meta scroll slot follows); FairMarketPill pad 5/6, gap 5, centred, label `pillSm` secondaryText via new `fairMarketPillLabel`; inline chrome radius lg + glassBorder, pad 12, gap 16; inline hero thumb 137.9 on a 306 inner width with a glassBorder top hairline; posts' centred `entityHeroBody` override removed. 3: `footerShare` deleted, share joins the left cluster. 4: GlassRim `edge="top"` (bottom stops zeroed) on TagThumb, GroupBannerThumb, GroupThumb, CollectionThumb (CSS top border dropped). 5: consecutive entity cards grouped into an `entityStack` column at 4; all other block gaps unchanged.
