---
tags: [nanza-api, nanza-mobile, plan, posts, likes, replies]
---

# Reply Likes — the heart on reply cards

**Status:** **mobile built 2026-10-19, uncommitted, pending device QA (`tsc` + eslint clean).** No
API change (it already did everything the feature needs — verified below). Both open questions
closed on the recommendations: heart only, count from one.

> **Implementation record (2026-10-19):** `PostItem` — the heart + count moved into one shared
> `heartAction` node (same glyph pair, fill, count typography); the `'full'` footer renders it in
> its old slot, and `'minimal'` now renders a `commentFooter` row holding just that node at the
> left edge instead of no row. No style change; the reply's identity-row trash is untouched.
> Same-day follow-ups (Skylar, device QA): the **report flag joins the reply footer** after the
> heart (the API's report route already takes a reply id); the composer's pinned quote of the
> post being answered switched from `'minimal'` to `'none'` so no heart shows while writing a
> reply; and `PostDetailScreen` gained a `hasReplied` flag (set via a new composer `onPosted`
> callback, forwarded by `JoinChatFooter`) that enables the thread query after the first reply —
> the gate read the root's cached reply count, which never moved, so the first reply on a
> reply-less post left the screen empty until a reload. Fold into
[[../../nanza-mobile/solution-designs/posts-chat|Posts Chat]] (mobile) and the *Likes* section of
[[posts|Posts]] (api) when it lands, then delete this file.

## The ask

Replies should carry the same heart and like counter a root post does. Today a reply card renders
with `footerMode="minimal"`, which drops the whole footer row — the only affordance on a reply is
the owner's trash on its identity row — so a reply cannot be liked at all.

## What already exists (verified 2026-10-18)

The API treats a reply as a post in every way that matters here, so there is **no API change**:

- **`PostLike` is keyed on any post id.** `POST /post/:id/like` looks the post up, runs the
  viewability check against `post.parentId ?? post.id` (the root), and toggles the row. A reply
  id is a valid target today.
- **Thread reads hydrate likes.** `GET /posts?parentId=…` goes through the same page builder as
  the root feeds and ends in `hydratePosts` → `withPostLikes`, so every reply row already arrives
  with `likeCount` and `viewerHasLiked`. The detail screen's reply list is reading them and
  discarding them.
- **Mobile's cache writer covers replies.** `usePostLike` writes the toggle's answer into every
  cached copy of the post under the `Posts`, `Replies` and `UserProfilePosts` query keys plus
  `PostDetail`. `Replies` is the detail screen's reply pages, so an optimistic like on a reply
  settles the same way it does on a root.

## Design (nanza-mobile)

One change in `PostItem`: the `'minimal'` footer stops being *no footer* and becomes *the reply
footer*.

- **Reply footer = heart + count, nothing else.** No reply glyph (a reply on a reply retargets the
  root, and the composer pill below the thread already owns that), no share (reference codes are
  root-only), no report (stays on the identity row's flag as today). Same `footerAction` box,
  same `heart-2` / `heart-filled` pair, same `conditionColors.damaged` fill, same count typography —
  one heart across the app, as decided 2026-08-30.
- **Placement**: the heart sits at the footer's left edge under the reply's content, where the root
  card's heart sits relative to its reply glyph, so the thread's hearts line up down the page.
  The row keeps the root footer's height so a liked and an unliked reply are the same size.
- **Guests tap → auth modal**, exactly the root heart's branch (`handleLikePress` already does
  this; the reply just reaches it now).
- **Owner's trash stays on the identity row.** Nothing moves.
- **`footerMode` semantics**: `'full'` (root cards), `'minimal'` (replies: heart only), `'none'`
  (PostDetail root, whose actions live in the top nav). The default for `depth > 0` stays
  `'minimal'`, so every reply surface picks the heart up without a call-site change.

### Surfaces that change

- `PostDetailScreen` reply list (the one `footerMode="minimal"` call site) — gains the heart.
- Nothing else renders replies today (the home feed and profile posts are roots; Stories pages are
  roots). The web public profile page lists root posts only and is out of scope.

### Touched code

- `src/components/posts/PostItem.tsx` — render the heart block for `resolvedFooter === 'minimal'`;
  the `'full'` branch keeps reply + heart + report + share.
- `src/styles/components/posts.ts` — a `commentFooterMinimal` class only if the reply footer needs
  a different top inset from the root's; otherwise reuse `commentFooter`.

## Decisions (2026-10-19, built on the recommendations)

1. **Heart only.** The pill below the thread is the reply entry; a second reply affordance per row
   would read as nested threading we don't have.
2. **Count from one.** Glyph alone at zero, matching the root card.
