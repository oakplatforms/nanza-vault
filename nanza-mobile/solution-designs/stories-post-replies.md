---
tags: [nanza-mobile, solution-design, plan, posts, affiliate]
---

# Stories pager — replies on post pages, light scroll vs hard swipe (Plan)

> **Status: plan, 2026-10-23 — building same day, mobile only (no API change).** Fold into
> `../nanza-api/solution-designs/affiliate-featured-shorts.md` (the pager's home) when it lands.

## The ask (Skylar, typed, 2026-10-23)

- The top-circle pager's **post pages show the post's replies and let the viewer add one**
  (today a tap on the card leaves for `PostDetail`, where replies live).
- A page longer than the screen must be **scrollable**: a *light* drag scrolls the thread (and
  the post's own rails scroll left/right as they already do). A **hard swipe up keeps the
  current page-turn animation**.

## Design

**`StoryPostPage`** (new, `src/screens/posts/AffiliateStoriesScreen/StoryPostPage.tsx`) replaces
the post branch of `StoryPageView`:

- **Thread** = the `PostDetail` recipe: root `PostItem` (`detailRoot`, no footer, live taps as
  today) + `usePostReplies` rows with the `minimal` footer (heart + count, owner delete via
  `useDeletePost` + the same confirm) + *View more replies* + the pinned-pill clearance. The
  thread query is gated on `_count.replies > 0 || hasReplied`, as on the detail screen.
- **Reply entry** = `JoinChatFooter placement="fixed" label="Reply to post"
  defaultReplyTo={post}` pinned to the page's bottom (the page is a full-height view, so the
  absolute footer lands on the screen edge). Its composer is a Modal, so the keyboard never
  touches the pager. `onPosted` flips `hasReplied` so a first reply switches the thread on.
- **Paging is sideways (2026-10-23).** Every vertical cut fought the thread's own scroll:
  release-velocity and bounce reads, then a gesture-handler `Pan` simultaneous with the scroll,
  which never attached on the pager's first page. Skylar's call after device QA was "swipe left
  and right instead … up and down is for reading the article". Both pagers (`AffiliateStories`,
  `PostPager`) are now `horizontal` paging `FlatList`s, so left means next. `StoryPostPage` is a
  plain vertical `ScrollView` with no page-turn logic. Pull-to-close is gone. The chevron
  leaves, and swiping past the last page still lands on the terminal page, which closes.
  - **Nested sliders:** a post's galleries and card carousels are native horizontal scroll
    views. The innermost one under the finger takes the drag, so touching a gallery slides it,
    and touching anywhere else on the page turns the page. At a gallery's last image, a swipe
    does not hand off to the next page, which matches Instagram and X. Accepted.
  - **Slider lock (same day, "the carousel inside a post isn't sensitive enough"):** the
    pager often took the drag before the gallery moved. `contexts/PagerLockContext` lets a
    slider lock the pager. `PostContentItems`' gallery rail and multi-image hero slider lock on
    `onTouchStart` and unlock on touch end, cancel, or `onScrollEndDrag`. Both pagers hold the
    lock as state on `scrollEnabled` and also unlock on any lift as a safety net. Outside a
    pager the context is a no-op.
  - A page leaving the viewport resets its thread to the top.
- Short (video) pages are unchanged by this plan; see the open question.

## Phase 2 — the post pager over What's New (Skylar, typed, 2026-10-23; built same day)

"If I click a post on the homepage, could it use this same UX?" Yes: a new `PostPager` route
(`src/screens/posts/PostPagerScreen`) is the Stories pager's post-page experience over a feed.
Tapping a root card on the homepage What's New bands or the What's New screen opens it on that
post; every page is a `StoryPostPage` (post, replies, pinned reply pill, light scroll vs hard
fling); swipe on through the feed in order; pull down on the first page closes; the terminal
page past the last post closes (only once `hasNextPage` is false — nearing the end pages the
feed on, two pages ahead, so the surface behind has those posts too). Seen is stamped per page.

- **No posts travel in params.** `PostPager` takes `{ startPostId, postableType, postableId,
  pageSize? }` and mounts `usePosts` with them — the same key the opening surface rendered
  from (home bands pass `HOME_POSTS_PAGE_SIZE`; the What's New screen the default), so opening
  costs no request and the pager sees the live cache.
- **Entry:** `PostItem` gains `onOpenDetail?: (post) => void`, an override for the card tap that
  runs after the same guards (player touch claim, modal stack) and replaces the `PostDetail`
  push; `PostThread` forwards it as `onOpenPost`. Only the two What's New hosts set it — group,
  tag, set and profile threads still open `PostDetail`. Inside the pager the root card keeps its
  own `PostDetail` tap.
- One glass down-chevron per page, `gestureEnabled: false` on the route like `AffiliateStories`.

## Phase 3: every chat area (Skylar, typed, 2026-10-23; built same day)

The ask: "whenever I open a post detail I want all the other posts for that area. If I'm on a
tag with 3 other posts, I should be able to slide through those."

- **One default, no per-screen wiring.** `PostThread`'s root-card tap now pushes `PostPager` over
  its own feed: `{ startPostId, postableType, postableId, pageSize }`, the thread's own read
  key. Hosts can still override with `onOpenPost`, but none does now. The What's New hosts
  dropped their overrides because the default does the same thing.
- **Coverage:** every `DetailChatSection` host, meaning products (Entity), listings, bids,
  groups, tag values and sets, plus the homepage What's New bands and the What's New screen.
  Not covered: profile post tabs (`ProfileSections` and `ProfileSeeAll` render `PostItem`
  directly, not through `PostThread`), which still open `PostDetail`.
- **Post's own home.** `StoryPostPage` now reads replies, the reply target and delete from the
  post's own `postableType` and `postableId`, falling back to BRAND for rail posts, not a
  hard-coded BRAND. A group post reaches its group's thread. A brand post reached through a
  tag or set union feed replies on the brand, which matches `PostItem`'s detail push. The pager
  itself still pages the feed it was opened from, so a tag page swipes through that tag's
  posts.

## Open question

- **Shorts too?** A Short page is the full-bleed video with the author panel; replies there would
  need a comment bubble in the panel opening a sheet with the thread + input. Not built until
  Skylar says so — post pages only for now.

## QA

Long post: light drag scrolls, galleries still swipe sideways, hard fling up turns the page with
the usual animation, hard fling down at the top goes back / closes on the first page. Short post:
paging unchanged. Reply: pill → composer → posted reply appears under the root; heart on a reply;
owner delete. Guest: pill shows the sign-in path, no crash.

## Phase 4: close at both ends, profiles (Skylar, typed, 2026-10-23; built same day)

- **Swiping back past the first page closes**, the mirror of swiping on past the last. Both pagers
  put an empty `start` page at index 0, next to the existing `end` page. The list opens at index
  1 (the tapped post, or the tapped affiliate), and reaching either empty page closes. That close
  is a **fade**, not the stack's slide-down (`navigation/closeWithFade.ts`: sets
  `forFadeFromCenter` plus a 180 ms timing on this screen, then `goBack` on the next frame). A
  vertical drop right after a sideways swipe read as two motions. The chevron keeps the
  slide-down. In the
  Stories pager, the "nothing to show" guard is now `pages.length <= 2`.
- **Profiles.** `PostPager` takes either feed source:
  `{ startPostId, postableType, postableId, pageSize? }` or `{ startPostId, accountId }`. The
  account source reads `useUserPosts`, the profile See-all's full 20-per-page list. That means
  one request when opened from the profile's 3-post preview, and a cache hit when opened from See
  all. Both hooks mount, but only the one the route names is enabled. The starting index is
  frozen the first time posts are in hand, because the account source may still be loading when
  the screen opens. `ProfileSections` (own and other profiles) and `ProfileSeeAllScreen` pass
  `onOpenDetail` to push it. With this, every post list in the app opens the pager.

## Phase 5: fixed controls (Skylar, typed, 2026-10-23; built same day)

"Can the down arrow not move with each scroll … the nav buttons stay fixed … share too, it's weird
to have actions slide." The controls moved out of each page into **one `PagerChrome`**
(`components/posts/PagerChrome.tsx`), rendered once per pager over the list and fed the
**active** page: the chevron on the left, and on the right the pencil (viewer's own Short only)
and share (when `useCanSharePost` allows, PostDetail's rule). Pages slide under it.

- **The Short's progress bar** lives in the same layer. The Stories screen owns one
  `Animated.Value` and zeroes it on every page change. The active Short's player writes it. The
  bar's row is always laid out on that pager and only its opacity follows "is a Short", so the
  buttons sit at the same height on video and post pages.
- **`getPagerChromeHeight(theme, topInset, withProgress)`** is the one number both pagers pad
  their post pages with, so the post always starts just under the buttons.

