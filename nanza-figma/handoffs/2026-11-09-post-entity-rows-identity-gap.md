---
tags: [nanza-figma, handoff, nanza-mobile, posts, entity, spacing]
status: open
date: 2026-11-09
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3392-137886&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3392:137886
nodes: [3392:137886, 3064:68432]
figma-2: https://www.figma.com/design/BFuH6ZKduwmAogACtc9AMe/Get-Nanza?node-id=203-8068&m=dev  # Get-Nanza file: post screen replies + thread line
frame: EntityLockupRow profile=seller-3 (inline post card) + HomeScreenContent (post rows) + Get-Nanza 203:8068 (post screen thread)
repo: nanza-mobile
scope: styles-only
builds-on: 2026-11-03-design-qa-posts-cards
---

# Posts — entity rows lose the card box (ruled list), tighter identity → text gap, post-screen reply alignment + thread line

Five changes to how posts render, all built and tested on device on a throwaway branch
(`throwaway-entity-card`). The values below are that build, so treat it as the reference
implementation. **Builds on [[2026-11-03-design-qa-posts-cards]]:** implement that one first.
This handoff supersedes its **item 5** (entity card → card 4 apart) and changes the context of
**item 2** (the inline card box now only appears on rail tiles and collection references).

> [!warning] App-wide — read first
> - **Change 3 is every post everywhere:** feed cards, replies and the post detail root.
> - **Change 1** is entity references inside a post's content. Gallery-rail entity tiles and
>   **collection** references keep the boxed card.
> - Change 2 is a real bug: the boxed card's 12 padding never applied. It affects the boxed
>   cards that remain.
> - **Changes 4–5 are the post screens:** the post pager (StoryPostPage, which both
>   PostPagerScreen and AffiliateStoriesScreen use) and PostDetailScreen.

## Request (verbatim)
1. "the spacing is off, it should look like this" (frame 3392:137886, with a device screenshot
   showing the card too tall)
2. "i want to remove the padding altogether and only separate them with a rule .. there should
   never be one above the first item and never one below the last item and no need for
   horizontal padding". Then, on device: "set to 20" (row top/bottom), "remove the top border
   but also remove the top padding on the first item", and "try setting it to 12" (above the
   first row).
3. "update the space between the username/avatar row and the post text … its really everywhere
   we have posts" (frame 3064:68432). Then, on device: "if its 9, try 8".
4. "update the post screen reply alignment and add that one rule between the post and the first
   reply in the list" (Get-Nanza 203:8068). The designer confirmed "rule" means the **vertical
   thread line**, not a horizontal divider.
5. "vertically center the text on the entity cards with the image so that visually the top of the
   text and the bottom of the market price button are equal distance from the top and bottom of
   the card"

**Do not change:** not stated, so: the entity row's text column (name / number / set line /
Market price pill, as built from 2026-11-03 item 2), the post footer, and the post's other
attachments.

## 1. Entity rows in a post: no box, ruled between rows

| Part | Before | After |
|---|---|---|
| Card chrome | glassBorder hairline, radius 20, gradient fill, padding 12 | **none**: no border, radius, fill or padding |
| Thumb → text | 16 | 16 (unchanged) |
| Thumb (`EntityHeroThumb`) | 137.9 wide on a 306 inner | **137.9 kept**, re-derived against the full 330 content column (`137.9 / 330`), so the text column gains the old padding |
| Set line fade | card gradient's end colour | page `background` (nothing else behind it now) |
| Above the first row | — | **12**, no rule |
| Between rows | 4 (boxed cards) | **20 · 1pt rule · 20** |
| Below the last row | — | **20**, never a rule |
| Rule | — | 1pt, `surface`, full content width: the post feed's own divider (`commentCardDivider`) |

A **single** entity reference follows the same rules: 12 above, 20 below, no rule.

**Unchanged (boxed, unruled):** entity tiles in a gallery rail, post tiles, collection references.

### How it's built
- `EntityLockupRow` gets a third chrome value: **`chrome: 'inline' | 'plain' | 'row'`**.
  `plain` has no container chrome and no gradient, uses `lockupContentPlain` (padding 0 both
  axes, gap 16) and the boxed thumb class (`chrome !== 'row'`). The meta fade falls through
  to `background`.
- `PostContentItems`, entity branch: `plainRow = !inRail && !tile` →
  `chrome={plainRow ? 'plain' : 'inline'}`, and `EntityHeroThumb frameStyle={plainRow ? styles.entityHeroThumbPlain : undefined}`.
- `PostContentItems`, block loop: `ruled = !tile && !railChild && isEntityCard(block[0])`.
  A block renders through the `entityStack` column when `block.length > 1 || ruled`, so a
  lone entity gets the stack's spacing too. A rule goes **before every row except the first**
  (`ruled && i > 0`).
- Styles:
```ts
// entity.ts
lockupContentPlain: { paddingHorizontal: theme.spacing.none, paddingVertical: theme.spacing.none, gap: Number(theme.spacing['16']) },

// posts.ts
const ENTITY_HERO_PLAIN_INNER_WIDTH = 330   // 402 − 16 − 16 − 40 post indent
entityHeroThumbPlain: { width: `${(ENTITY_HERO_THUMB_WIDTH_PT / ENTITY_HERO_PLAIN_INNER_WIDTH) * 100}%` },
entityStack:     { gap: 0, paddingTop: theme.spacing.gutter /* 12 */, paddingBottom: theme.spacing.base /* 20 */ },
entityStackRule: { height: 1, backgroundColor: theme.colors.surface, marginVertical: theme.spacing.base /* 20 */ },
```
The rows carry no vertical padding of their own. The stack owns all the space, which is what
lets the first row take 12 on top while the others sit 20 off the rules.

## 2. Bug: the boxed card's 12 padding never applied
`lockupContentInline` set `padding: 12` over `lockupContent`'s `paddingHorizontal: 8` /
`paddingVertical: 20`. **RN ranks the per-axis props above the `padding` shorthand regardless
of style-array order**, so the boxed card drew 8 / 20. On device that made the card about 16pt
taller than Figma 3392:137886 (a 117.67 card), with the thumb not centred top to bottom.

Fix: spell it per axis.
```ts
lockupContentInline: { paddingHorizontal: theme.spacing.gutter, paddingVertical: theme.spacing.gutter, gap: Number(theme.spacing['16']) },
```
This still matters for the boxed cards that remain (rail tiles, collection references). The
other two `padding:` shorthands in `entity.ts` don't override per-axis values, so they're fine.

## 3. Identity row → post text: match the visible gap
Figma 3064:68432 specs 6 between the identity row (31 avatar) and the post text, and the code
already used 6. On device it still read loose, because **RN iOS puts a line box's spare leading
above the glyphs where Figma centres them**. Measured from screenshots, avatar bottom → first
glyph:

| | Figma | Device before | Device after |
|---|---|---|---|
| Feed card (16/19 body) | ~8.9 | ~10.7 | **~8** |
| Post detail root (18/24 body) | — | ~20.7 (6 + 10 top-up) | **~8** |

The designer chose a visible **~8** everywhere ("if its 9, try 8").

| Class (`posts.ts`) | Before | After |
|---|---|---|
| `commentCard` gap (feed cards) | 6 (`xs`) | **3** (`xxs − 1`, no token) |
| `commentItemReply` gap (replies) | 6 | **3** (`xxs − 1`) |
| `detailRoot` gap (post detail) | 6 | **1** (`min − 1`, no token). The 18/24 line carries ~2.5 more leading above its first glyph |
| `bodyDetailStandalone` marginTop | 10 | **0** |
| `contentItemsDetailTree` marginTop | 10 | **0** |

These gaps are each column's only gap: the identity row → the indented content column. Keep
the "visible-gap" comment above `detailRoot` so nobody resets the values to Figma's literal 6.

Side effects to QA:
- A text-less card's attachment sits 11 under the avatar: 3 + `contentItemsCard` 8 (was 14).
- A detail post whose first block is an **image** sits 1 under the avatar. Posts normally lead
  with text; flag it if one doesn't look right.

## 4. Post screen: replies line up with the post + a thread line
Frame Get-Nanza 203:8068 is in the **other Figma file** and is partly older (a 35 avatar,
18pt name). Take only the reply alignment and the thread line from it.

**Alignment (post pager).** A feed post opens `PostPagerScreen` → `StoryPostPage`. That page
pads 8 (`stories.ts postPage`), and the root post (detailRoot, no inset) sits on it: avatar 8,
text 48. Replies carried `inset="detail"` (20) on top, which put them at **28 / 68**. They now
take a new **`'flush'` inset**, matching the frame: the reply's **divider runs flush** from the
page's 8, in line with the root avatar, and the **reply indents 6** inside it (avatar **14**,
text **54**), so the avatar never sits left of the rule. The reply list stops cancelling a bleed
it never had.
- `PostItem`: `PostInset = 'shelf' | 'page' | 'detail' | 'flush'` → `commentCardInsetFlush`
  (padding 6), and the divider takes `commentCardDividerFlush` (left / right 0, instead of
  `commentCardDivider`'s 8) when `inset === 'flush'`.
- `StoryPostPage` replies: `inset="flush"`; the list style is now
  `[commentListDetail, commentListAfterThreadLine, commentListFlush]`. This replaces the inline
  `{ marginTop: base, marginHorizontal: 0 }`.
- **Bottom clearance (pager):** the reply list's tail spacer is a new
  `detail.ts postPagerPinnedClearance` = **111** (`75 + 32 + xxs`). It was the shared
  `chatSectionPinnedClearance` (75), which left the last reply's footer under the "Reply to
  post" pill. At full scroll the last reply's footer icons rest **59** above the pill (footer
  14 + list gap 14 + 111 + list paddingBottom 6 below the icons). PostDetailScreen keeps its
  `postDetailPinnedClearance` (107) plus PageLayout's bottom spacer.
- `PostDetailScreen` replies already lined up at 20 (root in `threadContainerDetail` 20,
  replies `detail` 20 on a −20 bleed), so that screen gets only the line.

**Thread line** (both post screens), from the root post toward its first reply:

| Part | Value |
|---|---|
| Size | **1.5** × 72 (`min − 0.5`, no token; `xl + 32`). The designer's device call; the frame draws 2 |
| Colour | **`surface`**, the same as the divider between posts (`commentCardDivider`). The frame draws `glassBorder`; overridden on device |
| x | 60 in from the root avatar's left edge (`xl + base`). Literal from the frame |
| Above | 7 under the post (`xs + 1`) |
| Below | the replies start straight away: `commentListAfterThreadLine` (paddingTop 0) replaces the old 20 + 20 lead-in, and the first reply's flush 8 (`commentCardFlush`) lands on the frame's 7 |
| When | only while there are replies, or they're loading; never dangles on a reply-less post |

```ts
postThreadLine: { width: min - 0.5, height: xl + 32, marginLeft: xl + base, marginTop: xs + 1, backgroundColor: theme.colors.surface },
commentListAfterThreadLine: { paddingTop: theme.spacing.none },
commentListFlush: { marginHorizontal: theme.spacing.none },
commentCardInsetFlush: { paddingHorizontal: theme.spacing.xs },          // 6
commentCardDividerFlush: { left: theme.spacing.none, right: theme.spacing.none },
```

## 5. Entity rows: text centred on the thumb (plain rows only)
Measured on device: the thumb is 89.3 tall and the visible text block (name glyph top →
pill bottom) is 77.3. Top-pinned, that sat **2.3 / 9.7** from the thumb's top and bottom edges.
Now it's **~6 / ~6**.
- `lockupBodyCentered: { justifyContent: 'center', paddingBottom: theme.spacing.min }`, applied
  by `EntityLockupRow` when `chrome === 'plain'`. Centring the box alone leaves the name's ~2pt
  of line leading above its glyphs (the pill has none below it), so the bottom gives back 2 to
  even the **visible** gaps.
- **Plain rows only.** Boxed rail tiles and search / trade / offer rows stay top-pinned (centring
  was tried there and dropped, 2026-10-11).
- A row without a price pill centres its name and lines on the thumb as well.

## Code map (local `throwaway-entity-card`; match these on your branch)
- `src/components/entity/EntityLockupRow/index.tsx`: the `plain` chrome; `lockupBodyCentered` on plain rows.
- `src/components/posts/PostContentItems.tsx`: entity branch (`plainRow`) + block loop (`ruled`, rules between rows).
- `src/styles/components/entity.ts`: `lockupContentInline` per-axis fix, new `lockupContentPlain`, new `lockupBodyCentered`.
- `src/styles/components/posts.ts`: `entityStack`, new `entityStackRule`, new `entityHeroThumbPlain` + `ENTITY_HERO_PLAIN_INNER_WIDTH`, `commentCard` / `commentItemReply` / `detailRoot` gaps, `bodyDetailStandalone`, `contentItemsDetailTree`; new `commentCardInsetFlush`, `postThreadLine`, `commentListAfterThreadLine`, `commentListFlush`.
- `src/components/posts/PostItem.tsx`: `'flush'` inset.
- `src/screens/posts/AffiliateStoriesScreen/StoryPostPage.tsx`: flush replies, thread line, list classes, `postPagerPinnedClearance`.
- `src/styles/components/detail.ts`: new `postPagerPinnedClearance`.
- `src/screens/posts/PostDetailScreen/index.tsx`: thread line, `commentListAfterThreadLine` (replaces the inline marginTop).
- **`EntityLockupRow` chrome consumers:** PostContentItems (`plain` / `inline`), SearchEntityRow,
  EntityCardBase, TradeListItem, BulkSaleCard, OfferLineLockup (`row`, unchanged).
- **PostItem consumers (change 3):** PostThread (home WhatsNewSection, WhatsNewScreen,
  DetailChatSection), PostDetailScreen, PostPager / StoryPostPage, MyShortScreen,
  ProfileSeeAllScreen, ProfileSections, SharePostCard, PostComposer.

## QA
- **Post with 2+ entity references:** no box; 12 above the first row, 20 · rule · 20 between
  rows, 20 under the last row, no rule above the first or below the last.
- **Post with one entity reference:** no box, no rule, 12 above, 20 below.
- **Gallery rail containing an entity, and collection references:** still the boxed card,
  now with a true 12 padding all round.
- **Set line on a plain row:** still scrolls sideways; its fade blends into the page.
- **Composer attachment preview of an entity:** renders as a plain row.
- **Entity row text centring:** the name's top and the pill's bottom sit ~6 from the thumb's top
  and bottom edges; a row without a pill centres too.
- **Post pager (open a post from the feed / profile / affiliate stories):** replies' avatars
  sit 6 inside their dividers (avatar 14, text 54); each divider starts in line with the root
  avatar (8), never right of a reply avatar.
- **Pager bottom:** scroll to the end of a post with replies; the last reply's footer icons
  rest ~59 above the "Reply to post" pill.
- **Thread line** on the pager and PostDetail (push / share link): 1.5 × 72 in `surface` (matches the post divider), 60 in from the
  root avatar, 7 under the post, first reply ~8 below it; absent when a post has no replies.
- **Identity → text, ~8 visible:** feed cards, replies, the post detail root (text-only and
  with attachments), profile Posts tab.

## Open questions
None.

## Prompt
```text
Implement the handoff nanza-vault/nanza-figma/handoffs/2026-11-09-post-entity-rows-identity-gap.md
(read it in full). It builds on 2026-11-03-design-qa-posts-cards: implement that first, and this
one supersedes its item 5. Figma file 7c3Zz4nDUOutQG1BsnRRAw, frames 3392:137886 and 3064:68432.

SCOPE: STYLES ONLY. No data, navigation or behaviour changes.

1. ENTITY ROWS IN A POST (+ item 5): add chrome 'plain' to EntityLockupRow (no border/radius/gradient,
   lockupContentPlain = padding 0, gap 16; boxed thumb class). PostContentItems: entity refs
   outside rails/tiles render 'plain' with EntityHeroThumb frameStyle entityHeroThumbPlain
   (137.9 / 330). Every top-level entity block (a lone one too) renders in entityStack:
   paddingTop 12, paddingBottom 20, gap 0; a 1pt surface rule (entityStackRule, marginVertical
   20) BETWEEN rows only. Rail tiles and collection references keep the boxed 'inline' card.
   Plain rows centre the text column on the thumb: lockupBodyCentered (justifyContent center,
   paddingBottom 2).
2. BUG: lockupContentInline must set paddingHorizontal + paddingVertical (12), not the
   `padding` shorthand, which loses to lockupContent's per-axis values.
3. IDENTITY -> TEXT, visible ~8 everywhere: commentCard gap 3 (xxs-1), commentItemReply gap 3,
   detailRoot gap 1 (min-1), bodyDetailStandalone marginTop 0, contentItemsDetailTree
   marginTop 0. Keep the visible-gap comment.
4. POST SCREENS: new PostInset 'flush' (padding 6; its divider runs flush, left/right 0, via
   commentCardDividerFlush); StoryPostPage replies use it so the rule starts under the root
   avatar on the page's 8 and the reply indents 6 inside it, with the list on commentListAfterThreadLine +
   commentListFlush (no inline styles). Thread line postThreadLine (1.5 x 72, surface — the
   post divider's colour — marginLeft 60, marginTop 7) between the root and the replies on StoryPostPage AND
   PostDetailScreen, rendered only when replies exist or are loading. The pager's tail
   spacer becomes postPagerPinnedClearance (75 + 32 + xxs = 111), so the last reply's footer
   rests ~59 above the reply pill.

House rules (CLAUDE.md): plan first with confidence scores; styles in
src/styles/components/*; no inline styles, raw sizes or colours; compose tokens where none
match (and say so). Run /ui-review before finishing, then set this handoff to built and add a
Log line.
```

## Log
- 2026-11-09: handoff written; changes 1–3 built and device-tested on `throwaway-entity-card`
- 2026-11-09: added change 4 (post screen reply alignment + vertical thread line, Get-Nanza 203:8068) and change 5 (plain-row text centred on the thumb); built and device-tested on `throwaway-entity-card`
- 2026-11-09: thread line restyled on device: `surface` (the post divider's colour) and 2.21 wide, was `glassBorder` at 2
- 2026-11-09: pager replies re-indented on device: divider flush on the page's 8 (under the root avatar), reply 6 inside it (was flush 0 with the divider 8 in, so avatars poked left of the rule)
- 2026-11-09: thread line width set to 1.5 on device (was 2.21)
- 2026-11-09: pager bottom clearance 75 → 111 so the last reply's footer clears the reply pill by ~59
