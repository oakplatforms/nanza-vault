---
tags: [nanza-figma, handoff, pre-dev, nanza-mobile, posts, replies, thread]
bucket: pre-dev
status: open
date: 2026-11-12
originally: 2026-11-09 (change 4 of handoffs/2026-11-09-post-entity-rows-identity-gap, pulled out on its own)
verified: 2026-11-12 against dev ac083896 (v0.4.3)
figma: https://www.figma.com/design/BFuH6ZKduwmAogACtc9AMe/Get-Nanza?node-id=203-8068&m=dev
fileKey: BFuH6ZKduwmAogACtc9AMe
node: 203:8068
repo: nanza-mobile
scope: new PostItem inset variant + one new element on 2 screens + styles
---

# Single post page: replies line up with the post, plus a thread line

On the single post page, the replies now line up with the original post, and a short vertical
**thread line** runs from the post down to its first reply. It applies to both post screens:

- **The post pager** (`StoryPostPage`): every post opened from a feed. `PostPagerScreen` and
  `AffiliateStoriesScreen` both render it.
- **`PostDetailScreen`**: posts opened from a push notification or share link.

Built and device-tested on the designer's reference build.

> [!info] Pre-dev: why this one needs planning
> - **New `PostItem` inset variant:** `PostInset` gains `'flush'`. `PostItem` is shared by every
>   feed, shelf and chat section. Only the pager passes the new value, so check that no other
>   host changes.
> - **New element in the render tree:** the thread line is a `View` rendered conditionally on
>   both post screens.
> - **Inline styles removed:** both screens' reply lists drop an inline style object for
>   classes.
> - **No changes to:** data, queries, navigation, the reply composer, `PageLayout`.

## Request (verbatim, condensed)
1. "update the post screen reply alignment and add that one rule between the post and the first
   reply in the list" (Get-Nanza 203:8068). "Rule" means the **vertical thread line**, not a
   horizontal divider.
2. 2026-11-12: "remove the indent so the avatars line up with the original post".

## Not in this handoff
- The rest of the 11-09 posts handoff: entity rows without the card box, the boxed card's
  padding bug, the identity → text gap, and text centred on the entity thumb. None of these are
  on dev yet. They stay in `handoffs/2026-11-09-post-entity-rows-identity-gap.md`, waiting to be
  sorted.
- **Pager bottom clearance** (`detail.ts postPagerPinnedClearance`, 111): **already on dev.**
  Nothing to do.

## Reference build
- `assets/2026-11-12-post-screen-thread-line/reference.patch`: the designer's build of exactly
  this, against dev `ac083896`. **4 files, +67 / −12. It applies cleanly** (checked on a fresh
  dev worktree). Use it to check your work, not as a substitute for reading this file.
- `PostItem.tsx`: dev has added @-mentions to this file since the 11-09 build. The patch touches
  only the inset lines. Don't take the old 11-09 build's copy of the file.

## Measurements
Frame Get-Nanza 203:8068 is in the **other Figma file** and is partly older (a 35 avatar, an 18pt
name). Take only the reply alignment and the thread line from it. The designer overrode some
values on device; those overrides win.

### Alignment

| Screen | Root post | Replies before | Replies after |
|---|---|---|---|
| Pager (`StoryPostPage`) | page pads 8 (`stories.ts postPage`); root avatar at **8** | `inset="detail"` (20) on top → avatar at **28** | `inset="flush"` (0) → avatar at **8**, in line with the root |
| `PostDetailScreen` | root at 20 (`threadContainerDetail`) | `detail` 20 on a −20 bleed → **20** | **unchanged** (already lined up) |

On the pager, the reply's divider also runs flush, from the page's 8 to its right edge, not
inset 8 from each side as on a feed. The reply's avatar and divider start on the same line as
the root avatar.

### Thread line (both screens)

| Part | Value |
|---|---|
| Size | **1.5 × 72**. Width `min − 0.5` (no token for 1.5; the frame draws 2, the designer chose 1.5 on device). Height `xl + 32` |
| Colour | **`surface`**: the same as the divider between posts (`commentCardDivider`). The frame draws `glassBorder`; overridden on device |
| x | 60 in from the root avatar's left edge (`xl + base`) |
| Above | 7 under the post (`xs + 1`) |
| Below | the replies start straight away. `commentListAfterThreadLine` (paddingTop 0) replaces the old 20 margin + 20 list padding; the first reply's own 8 (`commentCardFlush`) lands on the frame's 7 |
| When | only while the post has replies, or they're loading. Never on a post with no replies |

## How it's built

### `src/styles/components/posts.ts`: five new classes (add to `CommentStyles` too)
```ts
postThreadLine: {
  width: Number(theme.spacing.min) - 0.5,
  height: Number(theme.spacing.xl) + Number(theme.spacing['32']),
  marginLeft: Number(theme.spacing.xl) + Number(theme.spacing.base),
  marginTop: Number(theme.spacing.xs) + 1,
  backgroundColor: theme.colors.surface,
},
commentListAfterThreadLine: { paddingTop: theme.spacing.none },
commentListFlush: { marginHorizontal: theme.spacing.none },          // pager list doesn't bleed
commentCardDividerFlush: { left: theme.spacing.none, right: theme.spacing.none },
commentCardInsetFlush: { paddingHorizontal: theme.spacing.none },   // reply sits on the page's 8
```
No existing class changes value.

### `src/components/posts/PostItem.tsx`
- `export type PostInset = 'shelf' | 'page' | 'detail' | 'flush'`. Add `'flush'` to the inset
  comment table above it.
- `insetStyle`: `inset === 'flush'` → `styles.commentCardInsetFlush`.
- Divider: `[styles.commentCardDivider, inset === 'flush' && styles.commentCardDividerFlush]`.

Every other inset renders exactly as before.

### `src/screens/posts/AffiliateStoriesScreen/StoryPostPage.tsx` (pager)
- Before the reply list:
  `{replies.length > 0 || isLoadingReplies ? <View style={postStyles.postThreadLine} /> : null}`
- Reply list style becomes
  `[postStyles.commentListDetail, postStyles.commentListAfterThreadLine, postStyles.commentListFlush]`,
  replacing the inline `{ marginTop: base, marginHorizontal: 0 }`.
- Replies: `inset="detail"` → `inset="flush"`.

### `src/screens/posts/PostDetailScreen/index.tsx`
- The same conditional `postThreadLine` before the reply list.
- Reply list style becomes `[styles.commentListDetail, styles.commentListAfterThreadLine]`,
  replacing the inline `{ marginTop: base }`. Its replies keep `inset="detail"`.

## Code map (dev @ `ac083896`)
- `src/styles/components/posts.ts`: `commentListDetail`, `commentCardDivider`,
  `commentCardInsetDetail` (the new classes sit beside these).
- `src/components/posts/PostItem.tsx`: `PostInset`, `insetStyle`, the `showDivider` block.
- `src/screens/posts/AffiliateStoriesScreen/StoryPostPage.tsx`: the replies block after the root
  `PostItem`.
- `src/screens/posts/PostDetailScreen/index.tsx`: the replies block after the root post.

## QA
- **Pager, post with replies** (open from the home feed): the reply avatars, text and dividers
  line up with the root post's avatar and text on the left. The thread line sits 60 in, 7 under
  the post, and the first reply starts just below it.
- **PostDetailScreen** (push or share link): the same line. Reply alignment is unchanged.
- **Post with no replies:** no line on either screen.
- **Replies loading** (cold open): the line shows with the spinner under it.
- **Last reply on the pager:** its footer still clears the "Reply to post" pill (dev's existing
  clearance).
- **Regression:** feeds, home shelves, tag/listing/bid chat sections and WhatsNew look unchanged.
  None of them pass `'flush'`.
- Light and dark themes: the line is `surface` in both.

## Open questions
None.

## Prompt
```text
Implement the pre-dev handoff
nanza-vault/nanza-figma/handoffs/pre-dev/2026-11-12-post-screen-thread-line.md (read it in full).
reference.patch in its assets folder is the designer's build of exactly this against dev
ac083896 and applies cleanly; use it to check your work. Figma Get-Nanza
BFuH6ZKduwmAogACtc9AMe, frame 203:8068 (take only the alignment and thread line from it; the
handoff's device values win).

1. STYLES (posts.ts + CommentStyles): postThreadLine (1.5 x 72, surface, marginLeft xl + base,
   marginTop xs + 1), commentListAfterThreadLine (paddingTop 0), commentListFlush
   (marginHorizontal 0), commentCardDividerFlush (left/right 0), commentCardInsetFlush
   (paddingHorizontal 0). Don't change existing classes.
2. PostItem: PostInset gains 'flush' -> commentCardInsetFlush; the divider adds
   commentCardDividerFlush when inset === 'flush'. Keep dev's @-mention code as is.
3. StoryPostPage: thread line before the reply list, only when replies.length > 0 ||
   isLoadingReplies; list style [commentListDetail, commentListAfterThreadLine,
   commentListFlush] (drop the inline object); replies inset="flush".
4. PostDetailScreen: the same conditional thread line; list style [commentListDetail,
   commentListAfterThreadLine] (drop the inline object); replies stay inset="detail".
Do NOT touch the entity-row / identity-gap changes from the 11-09 posts handoff, or the pager
clearance (already on dev).

House rules (CLAUDE.md): plan first with confidence scores. No inline styles, raw sizes or
colours. Run /ui-review before finishing, then set this handoff to built and add a Log line.
```

## Log
- 2026-11-09: written as change 4 of the posts handoff (`2026-11-09-post-entity-rows-identity-gap`).
  Built and device-tested on `throwaway-entity-card`. Thread line set to `surface` at 1.5 on
  device; pager replies re-indented to a 6pt inset inside a flush divider.
- 2026-11-12: pulled out on its own as **pre-dev** and rebuilt on dev `ac083896`
  (`throwaway-hue-change`). The pager clearance is already on dev, so it's dropped from this
  handoff.
- 2026-11-12: the 6pt reply indent is removed on device (`commentCardInsetFlush` 6 → 0), so the
  reply avatars line up with the original post.
- 2026-11-12: **ready for dev.** `reference.patch` exported (4 files, +67 / −12; applies cleanly
  to dev). ESLint and the full `tsc` run are clean.
