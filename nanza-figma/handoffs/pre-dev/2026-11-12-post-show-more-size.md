---
tags: [nanza-figma, handoff, pre-dev, nanza-mobile, posts, typography]
bucket: pre-dev
status: open
date: 2026-11-12
verified: 2026-11-12 against dev ac083896 (v0.4.3)
repo: nanza-mobile
scope: one style class (type-scale slot swap)
---

# Posts: "Show more" / "Show less" at 15

The **Show more / Show less** link under a post's content drops from 16 to **15**. It uses the
existing `meta` slot. Nothing else changes: the colour, position, tap target and behaviour all
stay as they are. Device-tested on the designer's reference build.

> [!info] Pre-dev
> Filed pre-dev at the designer's call, so it ships with this sprint's other post-screen work
> ([[pre-dev/2026-11-12-post-screen-thread-line]]). The change itself is a single type-slot swap.

## Request (verbatim, condensed)
1. "the show more show less on posts, can we set that to our body sm font size" (the tag page
   description, `postBodySm`). Tried on device, then:
2. "try setting it to 15 if we have a style for 15". The size-15 font token is
   `fonts.size.input`; the existing Balto slot at that size is `meta`.

## Not in this handoff
- **The link's colour.** It stays `action` on dev. The brand re-hue handoff
  ([[pre-dev/2026-11-12-brand-rehue-logo-splash]]) moves it to the new `link` token. The two
  touch different lines of the same class, so they apply in either order.
- **Footer icon colour** (reply, heart, flag, share): tried a shade darker on device and
  reverted. They stay `metaText`.

## The change

| | Before (dev) | After |
|---|---|---|
| Slot | `postBody`: Balto 16/19, +0.25 | **`meta`**: Balto 15/16, +0.11 |
| Colour | `action` | unchanged |

The label is a single line, so `meta`'s tighter 16 line box doesn't clip it. The tap area keeps
`SHOW_MORE_HIT_SLOP` (PostItem) around the word, so it's still easy to hit.

### `src/styles/components/posts.ts` → `showMoreText`
```ts
showMoreText: {
  ...typography.meta,          // was ...typography.postBody
  color: theme.colors.action,  // unchanged here (re-hue handoff makes it `link || action`)
},
```
Also update the comment above it: the link no longer shares the hashtags' text run, only their
colour.

`showMoreText` in `posts.ts` is used only by `PostItem`'s Show more / Show less toggle. Don't
confuse it with `buttons.ts showMoreText` (the shelf "Show more" button), which is **not**
changed.

## Reference build
`assets/2026-11-12-post-show-more-size/reference.patch`: 1 file, +5 / −5 against dev `ac083896`
(the class plus its comments). It applies cleanly.

## QA
- A long post in a feed, on a home shelf and on the post page: "Show more" sits at 15, a step
  under the 16 post text, still in the link colour.
- Tap it: it expands, and "Show less" uses the same size.
- Shelf "Show more" buttons (`ShowMoreButton`) are unchanged.

## Prompt
```text
Implement the pre-dev handoff
nanza-vault/nanza-figma/handoffs/pre-dev/2026-11-12-post-show-more-size.md.
In src/styles/components/posts.ts, showMoreText: swap ...typography.postBody for
...typography.meta and keep the colour line as it is on your branch. Update its comment.
Don't touch buttons.ts showMoreText or the footer icon colours. reference.patch in the assets
folder is the designer's build against dev ac083896.
House rules (CLAUDE.md): typography slots only, no font props outside typography.ts. Set this
handoff to built and add a Log line when done.
```

## Log
- 2026-11-12: tried `postBodySm` (14.5/21) on device, then settled on `meta` (15/16). A darker
  footer-icon shade was tried and reverted. Handoff written as **pre-dev**, ready for dev, with
  `reference.patch` (applies cleanly to dev). ESLint is clean.
