---
tags: [nanza-api, nanza-mobile, plan, profiles, posts, home, affiliates, shorts]
---

# Affiliate Rail — Featured Shorts & the Stories Pager

**Status:** **API built + deployed to dev 2026-10-18** (migrations
`20261019130000_featured_profiles_and_posts` + `20261019140000_post_display_name` run on dev;
types published through `0.1.120`). **Mobile built 2026-10-18, uncommitted, through a long device
QA loop the same day** (see *QA outcomes* below; `npx tsc` + eslint clean). Plan from a voice session the same day. Spans nanza-api
(one column, one validator branch, read exclusions, one new read endpoint, rail payload growth)
and nanza-mobile (own-circle sheet, a small *Add a Short* composer, a full-screen vertical
pager with progress bars, seen stamping per page). Builds directly on
[[affiliate-rail-latest-post-and-seen]] (rail, `PostSeen`, `GET /profiles/affiliates/latest-posts`)
and the EMBED content item from [[posts]] / [[../../nanza-mobile/solution-designs/posts-chat|posts-chat]].
When it lands, fold into [[posts]] (a *Featured* section beside Seen) and the mobile
[[../../nanza-mobile/solution-designs/whats-new|What's New]] design, and retire this file.

## The ask (Skylar, 2026-10-18)

1. **Affiliates can post a YouTube Short as a *featured* post** from the rail: tap your own
   circle → paste a Shorts link → it's live. Strict validation: it must be a YouTube **Shorts**
   URL, nothing else. "It's a post with one content item."
2. **Featured posts live only in the rail.** They are filtered out of the What's New feed and
   every other post list. They stay in the database forever (they're posts; the id is
   referenceable later), but after **24 hours** they stop surfacing anywhere.
3. **One live Short per affiliate** (Skylar, end of call — supersedes the mid-call "no cap"):
   while a Short under 24 hours old exists, the affiliate deletes it (hard delete, an **X** on
   it) before adding another. After 24 hours it stops showing and the slot is free again; the
   row stays in the database.
4. **The ring.** Pink when the viewer has something unseen from that affiliate — a live Short
   *or* their latest regular post (Skylar mid-call said "featured only", then reversed:
   "we actually do want the new posts to highlight"; the reversal is the rule). Seen clears
   it, as today. Any new activity, Short or regular, moves the affiliate to the front.
5. **One full-screen vertical pager for both kinds** ("it's in the same slider as the video
   player, it just looks different"). Tap a circle → full screen. A Short page is the video,
   full-bleed 9:16, **autoplaying as you swipe to it** (no play tap). A regular-post page is
   the post rendered flat on the dark background. **Swipe up** = next page. Swiping up past
   the last page **closes** back to Home — "at the end of the last highlight it should just be
   empty." A **down chevron** at the top also closes.
6. **Progress bar at the top**, Instagram-style: a thin bar that fills over the page's
   duration — the video's real duration (from the YouTube player) for a Short, a **5-second
   timer** for a regular post, after which the pager auto-advances. **Hold to pause**: a
   press-and-hold anywhere freezes the bar (and the video) so the viewer can read; release
   resumes.
7. Only Shorts from the **last 24 hours** appear in the pager. Older ones are kept for a
   future surface ("we might use them for something like the featured shorts page").
8. **Featured affiliates only in the rail** (Skylar, after reviewing the plan): a new
   `Profile.isFeatured` flag, admin-set. The rail and the pager include only affiliates with
   it on. An affiliate without it stays in the program (posts to What's New, affiliate link,
   vouchers — all unchanged) but has no circle. Shorts are accepted **only from featured
   affiliates** — "the post should only be from featured affiliates too" — since nobody could
   ever see one from anyone else.

## What exists today

- **Rail:** `GET /profiles/affiliates/latest-posts` (api `routers/profile.ts`) — every public
  AFFILIATE, their newest ACTIVE root BRAND post id/date, `viewerHasSeen`; ordered newest
  activity first. Mobile `AffiliateCircleCarousel` renders it; tap fetches the post and pushes
  `PostDetail`. The affiliate's **own circle is in the rail** (the endpoint doesn't exclude the
  viewer).
- **Seen:** `PostSeen` rows, `POST /post/:id/seen` (affiliate authors only via
  `POST_SEEN_AUTHOR_TYPES`), `useMarkPostSeen` stamps on `PostDetailScreen` mount and flips the
  cached rail row. `withPostSeen` exists in `utils/postSeen.ts` but is NOT wired into
  `hydratePosts`.
- **EMBED content item:** `validateContentItems` (api `validation/post.ts`) accepts a YouTube
  URL or `<iframe>` snippet in `body`, hostname-allowlisted. Mobile `embed.ts` mirrors it
  (`detectEmbed`) and `embedPlayer()` already extracts ids from `/shorts/<id>` URLs.
  `EmbedPlayer` is a `react-native-webview` loading a shell HTML page (`embedPlayerHtml`) with
  the YouTube iframe (`enablejsapi=1`), relaying `currentTime` up via `infoDelivery` messages;
  it can inject `pauseVideo` / `cueVideoById` commands.
- **YouTube has no separate "Shorts player."** A Short embeds through the same
  `youtube.com/embed/<id>` iframe as any video; the vertical presentation is just the frame's
  aspect. So the full-screen player is the existing shell in a 9:16 frame, not a new
  integration. The iframe API's `infoDelivery` also carries `duration` and `playerState`, which
  is everything the progress bar needs.
- **Composer:** `PostComposer` (five attachments, tags, body, drafts) is far more than a
  one-link post needs; the tab bar hosts it in a Modal for `BRAND`.
- **Post reads that list BRAND roots:** the brand home feed (`GET /posts` `scopeBrandFeed`),
  the author tab (`GET /posts?accountId`), the web profile (`GET /user/:username/page`
  count), and the rail's latest-post lookup. Tag/set pages union by tag rows (a featured post
  has none). The admin queue lists everything.

## Decisions (made while designing — Skylar to veto)

- **No new content item type.** A Short is an EMBED whose source passes a stricter check.
  Skylar: "it really could just be a link with validation… I'll let you make that decision."
  A `SHORT` type would duplicate the EMBED player and validator for one path rule.
- **`Post.isFeatured`** boolean, not a new `PostableType`: the home is still the brand; featured
  is a *placement* flag, like `isOffer` on `Listing`.
- **The 24-hour window is a read-time filter** (`createdAt >= now − 24h`), never a status
  change or cron. Nothing to expire; older rows are simply not selected.
- **The rail stays one light call**; the pager's hydrated posts come from a second call made
  on tap. The rail is in the first-paint gate and must not carry post bodies.
- **Shorts URLs only** — `youtube.com/shorts/<id>` (with `www.` / `m.`). `youtu.be/<id>` and
  `watch?v=` are rejected even though they can point at a Short, because the URL doesn't say
  so and the player would letterbox a landscape video full-screen.
- **A featured post has no body, no tags, no sets, no draft, no reply.** Exactly one content
  item. Cheap to relax later; expensive to take back.

## Design

### API

**1. Schema.** Two columns, one index, no enum changes:

```prisma
model Profile {
  …
  isFeatured Boolean @default(false)
}

model Post {
  …
  isFeatured Boolean @default(false)
  …
  @@index([isFeatured, createdAt])
}
```

Same word, two placements — `Profile.isFeatured` = *has a circle in the rail*,
`Post.isFeatured` = *is a Short in that circle* — Skylar's vocabulary for both ("is featured
action" on the profile, "featured" post type). Migration `20261019130000_featured_profiles_and_posts`
(Skylar runs it). Both land on the DTOs automatically (Prisma → JSON schema → types); bump
`packages/types` → `0.1.117`.

**1b. Admin toggle — `PUT /profile/:id/featured` `{ isFeatured }`**, `validateRole(…, 'admin')`,
mirroring `PUT /profile/:id/type` (same file, same shape). nanza-admin has no profile editor yet
(the `VERIFIED` type picker is still open from [[verified-profiles-and-affiliate-rail]]), so
for now Skylar flips it via the endpoint or DBeaver; when the admin picker lands it carries
this toggle beside the type.

**2. Create — `POST /post` with `isFeatured: true`** (multipart boolean, parsed like `draft`).
A new `validateFeaturedPost` in `validation/post.ts`, called from the create handler after the
existing checks, asserts all of:

- `postableType === 'BRAND'`, no `parentId`, not a draft;
- author's `profile.type === 'AFFILIATE'` **and `profile.isFeatured`** (re-checked on every
  mutating request, like the old brand gate — both are mutable admin flags);
- **no live Short already** — `post.count({ accountId, isFeatured: true, status: 'ACTIVE',
  createdAt: { gte: since24h } }) === 0`, else 409 "You already have a Short up — delete it
  first" (the mobile screen never offers the input while one is live, so this is the
  double-submit / stale-client guard);
- no `body`, no `supportedTagValueIds`, no `setIds`;
- exactly one content item, `type: 'EMBED'`, whose source (url or `<iframe src>`) has a
  YouTube host **and** path `^/shorts/[\w-]{6,}` — a `isYouTubeShortSource(source)` helper in
  the same file, reused by the EMBED leaf validator so the error reads "Featured posts take a
  YouTube Shorts link". Mobile mirrors it in `embed.ts` (`detectShort`).

**`PUT /post/:id`** rejects any change to `isFeatured` (400 — placement is set at create) and,
for a featured post, re-runs `validateFeaturedPost` on the new payload so an edit can't smuggle
in a body or a second item. `DELETE` unchanged. Lifetime post counter + voucher hook fire as
for any post (a Short is a post).

**3. Read exclusions — `isFeatured: false`** added to every list of BRAND roots that is not
the rail:

- `GET /posts` brand home feed (`scopeBrandFeed` arm) and `homeWhere` for `BRAND`;
- `GET /posts?accountId` author tab (both owner and stranger branches);
- `GET /user/:username/page` post count (web profile);
- the rail's latest-post lookup in `GET /profiles/affiliates/latest-posts` (the *regular*
  latest post must skip Shorts — they get their own fields).

Untouched: `GET /post/:id` (the pager reads full posts by id; a shared link still opens),
replies, likes, seen, the admin queue, tag/set unions (no tag rows on a featured post).

**4. Rail payload grows** (`GET /profiles/affiliates/latest-posts`). The profile query gains
`isFeatured: true` (featured affiliates only — the one behavioural change for existing
affiliates: no flag, no circle). Plus one more query — the newest live Short per affiliate:

```ts
prisma.post.findMany({
  where: { accountId: { in }, postableType: 'BRAND', parentId: null, status: 'ACTIVE',
           isFeatured: true, createdAt: { gte: since24h } },
  orderBy: { createdAt: 'desc' }, distinct: ['accountId'],
  select: { id, accountId, createdAt },
})
```

Row shape adds `featuredPostId`, `featuredPostAt`, `viewerHasSeenFeatured` (seen rows fetched
for both id sets in the one `getViewerSeenPostIds` call). **Order key** becomes
`max(latestPostAt, featuredPostAt)` newest first, then no-activity affiliates by username.
Seen still isn't a sort key.

**5. New read — `GET /profiles/affiliates/recent-posts`** — the pager's one call (named for the
payload, not the UI, per the `/rail` → `/latest-posts` precedent). Public (guests allowed).
Returns, **in rail order**, every affiliate with activity and their page items:

```json
{ "data": [
  { "profileId": "…", "accountId": "…", "username": "…", "avatar": "…",
    "posts": [ /* HydratedPostDto, isFeatured true first (newest first), then the latest regular post */ ] }
] }
```

Query: the same profile set (`AFFILIATE`, public, **`isFeatured`**) + block subtraction as the rail; the live Short
(`isFeatured, createdAt >= since24h`, at most one per author by the create rule) plus the newest
non-featured root per author (the `distinct` query the rail already runs); `postInclude` + `hydratePosts`; and
**`viewerHasSeen` attached via the existing `withPostSeen`** — this is the first surface that
renders it, so it's wired here only (not globally in `hydratePosts`). ~10 affiliates × a few
rows, one page, no pagination.

**6. Seen** — unchanged: `POST /post/:id/seen` per page; Shorts are affiliate-authored so the
author-type gate already admits them.

### Mobile

**1. Rail (`AffiliateCircleCarousel`).** `AffiliateRailItem` gains the three fields. Ring rule:

```ts
const hasUnseen = !!viewerAccountId && (
  (!!item.featuredPostId && !item.viewerHasSeenFeatured) ||
  (!!item.latestPostId && !item.viewerHasSeen))
```

Tap on **another** affiliate → `navigation.push('AffiliateStories', { startProfileId: item.id })`
(no post prefetch any more — the pager fetches). No activity at all → profile, as today.
Tap on **your own** circle (signed-in affiliate, `item.accountId === viewerAccountId` — only
featured affiliates have one, so no extra gate is needed) → `push('MyShort')` (below). Guests tap into the pager too (public content), quiet rings, no
seen stamping — "they won't see any highlights or do anything beyond just view it".

**2. My Short — `MyShortScreen`** (new route `MyShort`, full-screen, dark, same chrome as the
pager; Skylar, end of call — replaces the earlier *Add a Short / View* sheet idea). Two states
off the viewer's own row in the recent-posts data (or one `GET /posts?accountId&featured`-free
read: the row's `featuredPostId` → `postService.get`):

- **Live Short exists:** the Short full-bleed in the fullscreen `EmbedPlayer` (no autoplay
  here — it's yours), an **X** on it → confirm → `postService.delete(id)` (hard delete, the
  existing `DELETE /post/:id`) → the screen flips to the empty state; rail + recent-posts keys
  invalidated.
- **No live Short:** a single centred URL input ("Paste a YouTube Shorts link"), `detectShort`
  on submit, then `postService.create` with
  `{ postableType: 'BRAND', postableId: brandId, accountId, isFeatured: true,
  contentItems: [{ type: 'EMBED', index: 0, body: url }] }`. On success the player takes the
  input's place (same screen, live state), the rail key is invalidated with
  `refetchType: 'active'` (as the composer does) so the circle jumps to the front, pink. A
  rejected URL (bad host, not a `/shorts/` path, or the server's 409) shows inline.

Not `PostComposer`: a one-link post needs none of its five attachments, tags, body or drafts.

**3. The pager — `AffiliateStoriesScreen`** (new route `AffiliateStories: { startProfileId }`,
full-screen modal presentation, dark background). Data: `GET /profiles/affiliates/recent-posts`
(query key `['profiles','affiliates','recent-posts', viewerKey]`, `STALE_TIME.LIVE`). Page list
is built client-side:

- start at the tapped affiliate; **all** of their items (so re-opening a seen circle still
  shows something);
- then every later affiliate in rail order, **unseen items only**;
- then one terminal empty page — reaching it closes the screen (`navigation.goBack()`), which
  is what "swipe up past the last one closes" means in `FlatList` terms.

A vertical `FlatList` with `pagingEnabled`, `snapToInterval = screen height`, one item per
screen, `onViewableItemsChanged` → `activeIndex`. Chrome over every page: a top row with the
**progress bar** (one segment per page of the *current affiliate*, Instagram-style), the
author's avatar + username, and a **chevron-down** close on the left.

- **Short page:** `EmbedPlayer` in a new `variant="fullscreen"` — 9:16 frame filling the
  screen, `mediaPlaybackRequiresUserAction={false}`, `autoplay=1` on the player URL, and the
  shell relay extended to forward `duration` and `playerState` beside `currentTime`. Only the
  active page mounts a WebView (inactive pages render the thumbnail, so swiping never has two
  videos running). Progress = `currentTime / duration`. `playerState === 0` (ended) → advance.
- **Post page:** `PostItem` (`variant="detail"`-like flat treatment, `enableDetailNav` off,
  minimal footer with heart + reply counts) centred on the page. A 5-second timer
  (`reanimated` progress) → advance.
- **Hold to pause:** a long-press responder over the whole page freezes the timer / injects
  `pauseVideo`; release resumes (`playVideo`). Horizontal swipes do nothing in v1.
- **Seen:** when a page becomes active and the viewer is signed in, stamp it — the body of
  `useMarkPostSeen` extracted into a `markPostSeen(post, accountId, queryClient)` helper that
  both the detail screen and the pager call; its rail-cache flip now sets
  `viewerHasSeen` *or* `viewerHasSeenFeatured` by matching either id.
- **Tap on a post page** → `push('PostDetail')` for replies (the Stories pager is a reader,
  not a thread). Tap on a Short page does nothing beyond the player's own controls.

**4. Types.** `isFeatured` on `HydratedPostDto` via `@oakplatforms/types ^0.1.117`;
`AffiliateRailItem` + a local `AffiliateRecentPosts` row type in `services/api/Profile.ts`.

### Not in scope

A page for Shorts older than 24 hours; horizontal (per-affiliate) navigation; reactions on
Shorts; Shorts in What's New; analytics beyond `post_viewed` with `is_featured`; the web app
(the public profile page simply doesn't list them).

## Open questions for Skylar — all answered on the call (2026-10-18)

1. ~~Caption on a Short?~~ **No** — "just the one embed item, that's it."
2. ~~Auto-advance when a Short ends?~~ **Yes** — the bar runs the full video length, then it
   scrolls up to the next item, same as the 5-second rule for posts.
3. ~~Sharing a featured post.~~ **Leave the mint alone for now**; a shared link opens the
   normal `PostDetail`. "We can figure out the best way of doing that one later."
4. ~~Own circle → sheet vs composer.~~ **Neither** — a `MyShort` screen: the live Short with an
   X to delete, or the input when there is none (see Mobile §2). One live Short at a time.
5. ~~`youtu.be` / `watch?v=` rejected.~~ **Fine.**
6. ~~Guests open the pager.~~ **Yes**, view only — no rings, no tracking.

## QA outcomes (device, 2026-10-18) — what the YouTube embed allows

Skylar's pass through the pager settled a pile of player decisions; the durable ones:

- **YouTube's chrome can't be reached from a shell page** — the player is a cross-origin iframe
  there; `controls=0` removes the bar but nothing removes the start chrome (a centred play/pause
  button drawn on every playback start) or the title row. `showinfo` / `modestbranding` are dead.
- **So the pager loads the embed page DIRECTLY** (2026-10-18, "direct mode" in `EmbedPlayer` /
  `directEmbedScript`): the WebView's top document is youtube.com's embed page (the referrer the
  player demands rides as a request header), so an injected script runs on YouTube's own origin
  and can `display:none` the player's chrome. On device (2026-10-18) the Shorts embed draws ALL of
  it — the centred play/pause, the channel avatar + name row, the action menu, the fullscreen
  controls, the scrim — inside one `#player-controls` host, none of it under the classic `ytp-`
  names; the script hides that host (the `ytp-` names stay as a fallback for the classic player).
  The same script owns the relay (`movie_player` exposes the
  iframe API's methods) and sends play while the player still reports unstarted/cued half a second
  after load — iOS refuses autoplay with sound on some loads, and with the bezel hidden a command
  start is free, so the url autoplays WITH sound and the mute-then-unmute (which sometimes left a
  Short silent) is gone. Deliberately brittle: it leans on
  YouTube's class names and will need a touch-up when they change (Skylar accepted that). The
  card players keep the shell page. A full-page **hold layer** takes every touch (hold = pause,
  180 ms delay so a swipe never pauses; tap on a post page = `PostDetail`). A **slate** covers the
  player until its first "playing" report (6 s safety reveal).
- **Position streams only after a state change** YouTube noticed, so the shell page polls: it
  re-sends the `listening` handshake every 250 ms and keeps its own clock (last reported time +
  elapsed while state = playing), posting `{ t, d, s }`; the player caches the last duration. It
  also nudges `playVideo` while unstarted/cued, but only from 1.5 s after load — a command
  flashes the bezel, so the url's own autoplay gets first go.
- **Short page layout:** the video is a true 9:16 frame at 90 % of the page height (`SHORT_FRAME_HEIGHT_RATIO`, 2026-10-23 — full height clipped too much off the sides), centred both ways — a cover
  fit that clips a sliver off each side rather than letterboxing. Our pink tracker and the
  chevron overlay the top; four stacked gradients (`scrimBottom` ×3 at 36 / 28 / 18 % +
  `fadeHeroCollapsed` at 28 %) anchor to the frame's bottom so the video melts into the page.
  Over them the **author panel**, a full-width column 8 pt above the home indicator: 34 pt avatar
  + `title`-slot username + the card's relative timestamp (lightened to `textSecondary`), the
  Short's **`displayName`** ("Description" in the UI, `postBody` face) under them, then bottom-left
  a red **Shorts** mark (silhouette + white play glyph, placeholder SVG until Skylar's lands) with
  the word *Shorts* that opens the video in the in-app browser sheet. Holding a Short pauses it
  and lets YouTube's own pause bezel show. Post pages carry the chrome's 8 pt side gutter. Pull-down on the first page closes.
- **One page per affiliate:** the live Short, else the latest post — never both. The ring follows
  the same fallback. The sequence is the tapped affiliate first, then **every** other affiliate in
  rail order (newest first) with the tapped one filtered out — from circle 3: 3, 1, 2, 4, 5 … —
  seen or not. (Skipping seen pages, or only running forward from the start, read as the pager
  closing at random.) Seen only drives the rings.
- **Own circle:** with a live Short → the pager at yourself, a pencil in the chrome (own Short
  pages only) opening **My Short**; without one, latest post or not → straight to My Short to add
  one. The pager's pages follow the data (no freeze), so a Short deleted or posted from My Short
  under it swaps that page for the latest post in place.
- **My Short is a form** in the Create Collection language: no header title, outline
  *Description* (optional, `Post.displayName`) and *YouTube Shorts link* fields with help text,
  placeholders as labels, and the post composer's top-right pill — *Post* with nothing up, *Save*
  once a live Short's description changes (PUT with just `displayName`; the API skips the body
  requirement for featured posts) — plus a 16:9 card-size preview with the form bucket's X over
  its corner → confirm → delete (description and Short go together).
- **`Post.displayName`** (API, same day): optional, ≤120 graphemes, accepted on create and
  replaced-when-sent on edit; migration `20261019140000_post_display_name`.

## Order of work

1. ~~nanza-api~~ **Done 2026-10-18:** `Profile.isFeatured` + `Post.isFeatured` + index +
   migration; `constants/featuredPosts.ts` (`FEATURED_POST_WINDOW_MS`, `featuredWindowStart`);
   `PUT /profile/:id/featured` (admin); `validateFeaturedPost` + `isYouTubeShortSource` (the
   EMBED host check was lifted into a shared `parseEmbedSource`); create (`isFeatured` multipart
   flag) + edit (toggle rejected, featured rows re-validated) wiring; read exclusions on the brand
   feed, both author-tab branches, the web page count and the rail's regular lookup; rail:
   featured-only profiles via a shared `findRailProfiles`, `featuredPostId` /
   `featuredPostAt` / `viewerHasSeenFeatured`, order by newest activity of either kind;
   `GET /profiles/affiliates/recent-posts` (first consumer of `withPostSeen`, hydrated via
   `postInclude` + `hydratePosts`, affiliates with nothing omitted); OpenAPI JSDoc; types
   `0.1.117` generated + built (publish on the dev pipeline).
2. ~~nanza-mobile~~ **Done 2026-10-18:** `embed.ts` `detectShort` (shorts path only),
   `embedPlayer(code, { autoplay })`, the shell relay now posts `{ t, d, s }` (position,
   duration, playerState) parsed by `parseEmbedPlayerRelay`; `EmbedPlayer` gains
   `variant="fullscreen"`, `autoplay`, `paused` (injects pause/play), `onProgress`, `onEnded`
   (ended fires once per playback); `styles/components/stories.ts` (new bucket) +
   `posts.ts` `embedPlayerFullscreen`; `useMarkPostSeen.ts` exports `markPostSeen(post,
   accountId, queryClient)` (flips `viewerHasSeen` *or* `viewerHasSeenFeatured` on the rail
   row) and the hook wraps it; `AffiliateCircleCarousel`: `railItemHasUnseen`, own circle →
   `MyShort`, any activity → `AffiliateStories`, else profile; `affiliateRecentPostsQueryOptions`
   shared by both screens; `AffiliateStoriesScreen` (vertical `FlatList` with `pagingEnabled`,
   pages frozen on first data — tapped affiliate's all, later affiliates' unseen, terminal empty
   page closes; per-page chrome: segmented bar + avatar/username + chevron-down close; only the
   active Short mounts a WebView, inactive Shorts draw the avatar slate; 5 s `Animated.timing`
   for posts, player relay for Shorts, hold overlay pauses both, short tap on a post page pushes
   `PostDetail`; seen stamped per active page); `MyShortScreen` (`PageLayout` floating header,
   trash `rightIcon` when live → confirm → `useDeletePost('BRAND', brandId)`, 9:16 frame player,
   else hint + `CollectionSearchFooter` input → `postService.create({ isFeatured: true,
   contentItems: [EMBED] })`, both rail keys invalidated with `refetchType: 'active'`); routes
   `AffiliateStories` (`gestureEnabled: false` — the pager owns the vertical swipe) + `MyShort` in
   `AppNavigator` (a P0 nav change, covered by this approved plan).

   **QA watch-list (device):** iOS autoplay *with sound* inside WKWebView
   (`mediaPlaybackRequiresUserAction={false}` + `autoplay=1` + an injected `playVideo`; if iOS
   still mutes, the fallback is `mute=1` + an unmute tap); hold-to-pause over the WebView (the
   transparent hold layer sits above the player, so YouTube's own controls are unreachable on a
   Short page by design); the vertical swipe reaching the `FlatList` through the WebView; the
   5-second post timer resuming from where it paused.
3. **Skylar:** run the migration (dev), flag the test affiliates featured, post a Short, check
   the ring / pager / 24-hour drop / feed exclusion as guest and member, and that an unflagged
   affiliate has no circle.
4. Fold into `posts.md` + mobile `whats-new.md`, delete this plan.

## Follow-up — the Short play badge (Skylar, voice, 2026-10-23; mobile built same day)

A circle whose pager page is a live Short now carries a small **brand-pink play disc at the
ring's six o'clock** (white `play-solid` glyph, page-colour rim, straddling the ring's edge so
it sits between the circle and the name) — "similar to the YouTube icon, in our pink circle".
It says *what* the circle opens and takes the ring's colour for *whether* it's been seen: pink
while the Short is unseen, the ring's quiet border grey once seen (Skylar, same day: "make the
play icon grey like the border"). `CircleThumb` gains `playBadge: 'brand' | 'quiet'` (`lg`
variant only; 26 on the 88 circle, half hanging below the ring — lowered from a third on
Skylar's "down a smidge"); the rail derives one state for ring and disc and passes
`playBadge={item.featuredPostId ? state : undefined}`. The rail's caption now reads the profile's display name
when set (see [[profile-display-name]]). Pending device QA.
