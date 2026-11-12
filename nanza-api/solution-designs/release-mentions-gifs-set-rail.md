---
tags: [nanza-api, nanza-mobile, nanza-admin, plan]
status: plan — 2026-11-09 (Skylar, voice); all five parts built same day in api + mobile + admin, uncommitted, not deployed, no device QA
---

# Release batch: @-mentions, animated GIFs, set "What's inside" rail, trending sets

Four updates Skylar wants in this release. They're independent, so each part can be built and
QA'd on its own. Only parts 1 and 4 touch the schema, so they share one migration and one types
bump (`0.1.138` → `0.1.139`).

| # | Part | API | Mobile | Admin | Migration | Native rebuild |
|---|---|---|---|---|---|---|
| 1 | @-mentions in posts | yes | yes | — | `PostMention` | no |
| 2 | Animated GIFs | yes (+ resizer Lambda) | yes | — | no | **yes** (Android Fresco deps) |
| 3 | Set screen "What's inside" rail | no | yes | — | no | no |
| 4 | Trending sets | yes | yes | yes | `Set.isTrending` | no |

Suggested order: **3 → 4 → 1 → 2**. Part 3 is mobile only and the smallest. Part 4 is small
and shares the migration with 1. Part 2 needs a native rebuild and device QA on both platforms,
so it goes last.

---

## 1. @-mentions

**Want:** typing `@` in the composer opens the same drawer as `#`, listing **non-BASIC users**
(AFFILIATE + VERIFIED). Picking one inserts `@username` inline. Unlike tags, the **`@` stays
visible**, and the run renders in the same pink as a tag.

### How #tags work today (what we copy)
- `TagTextInput.tsx` wraps `react-native-controlled-mentions`. `triggersConfig` has one
  `tag` trigger (`#`). The library supports several keyed triggers, so `@` becomes a second
  `mention` entry with its own `getPlainString: d => '@' + d.name`.
- `PostComposer` watches `activeTrigger` and renders `TagSuggestions` in the drawer panel.
  `handleSuggestionSelect` checks the cap and calls `select({id, name})`.
- Text stores tokens as `#[Name](id)` (`postMarkup.ts`, the same regex on mobile and API). The
  ids also go up as `supportedTagValueIds` / `setIds` → join rows, and the **join rows are the
  source of truth** (`validatePostTextTokens` 400s on any token id not on the post).
- `PostMarkupText` resolves each token by id against the post's joins and paints it with
  `tagMention` (`theme.colors.action`).

### API
- **Schema:** `PostMention { postId, profileId, createdAt }`, a composite PK, with
  relations to `Post` and `Profile`, mirroring `PostSet`. `Post.mentions PostMention[]`.
- **Markup:** a new token `@[username](profileId)`, with `MENTION_TOKEN_PATTERN` in
  `src/utils/postMarkup.ts` next to the tag pattern (no collision, since tags start with `#`).
- **Create / update** (`POST /post`, `PUT /post/:id`): accept `mentionProfileIds: string[]`
  and write the `PostMention` rows. Validation (`validation/post.ts`):
  - every id is a Profile with `type in PUBLIC_AUTHOR_PROFILE_TYPES` (not BASIC), and not
    blocked by or blocking the author;
  - cap: **10 mentions**, counted separately from the 10 tags + sets;
  - extend `validatePostTextTokens` so `@` token ids must be in `mentionProfileIds`.
- **Read:** `include=mentions.profile` (slim select: id, username, displayName, avatar) so
  `PostMarkupText` resolves the current username even after a rename.
- **Picker search:** extend `GET /profiles` so `type` accepts a list (or add `type=PUBLIC`),
  i.e. `type: { in: ['AFFILIATE','VERIFIED'] }`. Make `q` match `username` **or**
  `displayName`, and add the block filter. Empty `q` returns featured / most-followed first, so
  the bare `@` drawer isn't blank.
- **Notification (decided 2026-11-09: yes, push):** add `post.mentioned` to
  `NotificationKind`, and `outbox.queue` each *newly added* mention in create/update (an edit
  doesn't re-push existing mentions), copying the reply push at `post.ts:1074-1090`
  (`flush()` already skips the actor and blocked accounts). Target `{type:'post', id}`;
  `PushBridge` already opens posts.

### Mobile
- `TagTextInput`: a second trigger `mention: { trigger: '@', … }`. `toMentionsValue` /
  `fromMentionsValue` learn the `@` form.
- New `MentionSuggestions` drawer (same row shell as `TagSuggestions`: avatar circle,
  displayName, `@username`, verified/affiliate badge), backed by `useSearchMentionProfiles`
  → `GET /profiles?type=PUBLIC&q=`.
- `PostComposer`: track mention ids separately (`collectMentionTokenIds`) and send
  `mentionProfileIds`. Mentions are **allowed with no brand** (tagging is brand-gated today;
  mentions shouldn't be).
- `PostMarkupText`: render `@username` in `tagMention` pink, keep the `@`, and on tap go to
  `ProfileScreen`. A mention whose join row is gone renders as plain text.
- Backspace removes the whole token, the same as `widenToWholeTags`.

---

## 2. Animated GIFs

**Want:** a GIF picked for a post plays in the feed. Right now frames are lost in **four
places**, plus Android can't animate at all:

1. **Mobile resize** – `PostComposer.tsx:837` runs every asset through
   `resizeImage(... 'JPEG')`, which flattens it to one frame. The caption editor
   (`PhotoTextEditor`, `captureRef` jpg) flattens it too.
2. **API mime gate** – `post.ts:924` / `:1288` and `uploadImage.ts:34` reject `image/gif`.
3. **Upload Sharp** – `uploadImage.ts:70-86` calls `sharp(buf)` without `{ animated: true }`,
   so it keeps page 0 → `.webp()`.
4. **CDN resizer** – `lambdas/imageResizeHandler.ts:98-101` does the same; feed tiles always
   ask for a width.
5. **Android render** – core RN `Image` needs Fresco's `animated-gif` / `animated-webp` /
   `webpsupport` in `android/app/build.gradle`. None are there.

### Fix
- **Mobile picker:** if `asset.type === 'image/gif'`, skip `resizeImage` and upload the
  original (client-side cap, e.g. 15 MB, with a friendly error). Hide or disable the caption
  editor for GIFs (burning text in would need a GIF encoder, so it's out of scope).
- **API:** allow `image/gif` on post create/edit. In `uploadImage`, open animated sources
  with `sharp(buf, { animated: true })` so `.resize().webp()` keeps every frame (output:
  **animated WebP**, smaller than GIF). Raise the multer limit for posts (5 MB → 15 MB).
  Also fix the existing bug where S3 `ContentType` is the *upload's* mime, not the output's
  (`image/webp`).
- **Resizer Lambda:** the same `{ animated: true }` when the original has `pages > 1`.
- **Android:** add Fresco `animated-gif`, `animated-webp`, `webpsupport` (versions matched to
  RN 0.83's Fresco), then a native rebuild.
- **iOS:** core `Image` plays animated GIF. Animated **WebP** on iOS needs a device check
  (open question 2). If it doesn't play, store GIF sources as GIF instead (`.gif({...})` with
  `animated: true`).
- Other `uploadImage` callers (brand, set, profile, listing, …) are unaffected. They still
  send JPEG/PNG and `animated: true` is a no-op on single-frame images.

---

## 3. Set screen: "What's inside" rail

**Want:** on `SetScreen`, between the description and the chat, a **What's inside** rail of
the set's cards. The first 10 are the small image tiles with the backdrop (the listing/bid
tile look). The header arrow opens a **two-column paginated grid** of every card in the set.

- **Data:** `GET /entities?setId=&limit=10&page=0&include=product` (exists, cheap, indexed on
  `setId`, and already returns `lowestAsk`). No API change.
- **Rail:** `CardCarousel` (`title="What's inside"`, `onSeeAllPress`) with `ItemThumb` tiles:
  `imageId={entity.image}`, `price={entity.lowestAsk}`, `cardNumber={entity.product?.number}`,
  tap → `EntityScreen`. Hidden when the set has no cards. It sits where the retired "See all
  cards" button was (`SetScreen` comment :35-37).
- **See all:** there's no all-cards grid screen today (set badges go to Search rows). Reuse
  **`FeedGrid`** and add a source `{ kind: 'set'; setId }` that pages `GET /entities?setId=`
  (20 per page, `CardGrid`, same `ItemThumb`). Title = set displayName.
- **Order:** the endpoint orders by listing count, then bid count. A set catalogue probably
  wants **card number order** instead (open question 3). If so, add `sort=number` to
  `GET /entities`.

---

## 4. Trending sets

**Want:** sets get `isTrending` like tag values, and the bare `#` drawer's Trending list shows
trending **tags and sets** together.

- **Schema:** `Set.isTrending Boolean @default(false)`, `@@index([brandId, isTrending])`.
- **API:** `PUT /set/:id` accepts `isTrending` (admin-guarded, like
  `PUT /supported-tag-value/:id`). `GET /sets` takes an `isTrending=true` filter.
- **Admin:** an `isTrending` toggle in the Sets detail editor, next to `isVisible`.
- **Mobile:** `TagSuggestions` trending branch also calls
  `GET /sets?brandId=&isTrending=true&isVisible=true` and merges the sets in as `kind: 'set'`
  rows (meta "Set"). Order: tags first, then sets, or interleaved by name (open question 4).
  Search already returns sets, so nothing changes there.

---

## 5. Group moderator push on new posts (added 2026-11-09)

**Want:** when someone posts in a group, that group's **moderators** get a push.

- New kind `group.post.created` in `NotificationKind`. In `POST /post` when
  `postableType === 'GROUP'` and the post is published (not a draft, not a reply), queue one
  notification per `GroupMember` with `role = MODERATOR`, `status = ACTIVE`. `flush()` drops
  the author (a moderator posting doesn't push themselves) and blocked accounts.
- Copy: "<name> posted in <group>" + `pushPreview(body)`. Target `{type:'post', id}`.
- Draft → publish (`PUT` publish path) fires the push at that point instead.
- No schema change and no mobile change beyond confirming `PushBridge` opens the post.

## Packages
No new npm packages. Part 2 adds three **Android Gradle** (Fresco) dependencies in
`android/app/build.gradle`. They're native, not npm, so they need a native rebuild.

## Rollout
1. Migration (`PostMention` + `Set.isTrending`), which Skylar runs. Then `prisma generate`,
   types `0.1.139`, publish.
2. API deploy to dev (parts 1, 2, 4), including the resizer Lambda.
3. Mobile `types:update`, then build parts 3 → 4 → 1 → 2. Native rebuild for part 2.
4. Admin trending-set toggle.
5. Device QA on both platforms: GIF plays in feed + lightbox, mention tap → profile, rail →
   grid paging.

## Open questions
1. ~~Mention notifications~~ — decided: yes, push.
2. ~~iOS animated WebP~~ — decided: animated WebP (lighter) **only if it plays on both
   platforms**. Playback is the requirement, so if a device check fails, store GIF as GIF.
3. ~~Set grid order~~ — decided: card number (ABC001, ABC002, …) for both the rail and the
   grid. Add `sort=number` to `GET /entities` (order by `product.number`, then id).
4. ~~Trending mix~~ — decided: one list, trending tags and sets mixed (sorted by name, set
   rows keep the "Set" meta).
5. ~~Mentions in replies~~ — decided: **yes**. `validatePostTextTokens` lets `@` tokens
   through on replies, and the composer enables the `@` trigger even when `isReply`. **Tags
   stay off in replies**: tags put posts on tag/set pages, where replies would show up as
   context-free fragments.

All open questions answered 2026-11-09. Ready to build in order 3 → 4 → 1 → 5 → 2.

## Build log (2026-11-09)

All five parts built the same day, uncommitted. API tsc + lint clean, mobile tsc clean
(lint: only old warnings), admin tsc red until types `0.1.139` is installed.

**Pending, in order:** run migrations `20261109120000_set_is_trending` +
`20261109130000_post_mentions` → publish types `0.1.139` (version bumped, built) →
deploy API dev (incl. `imageResizeHandler`) → `types:update` in mobile + admin → Android
native rebuild (Fresco Gradle deps) → device QA on both platforms.

What landed (and where it differs from the plan above):
- **Set rail.** `EntityThumb` (entity → `ItemThumb`, no intent chiclet, price = lowest
  ask), `CardCarousel` "What's inside" on `SetScreen` with `divider` (same seam as the tag
  page's children rail), and `FeedGrid` source `{ kind: 'set', setId }`. `GET /entities` got
  `sort=number` (`product.number` asc, then id).
- **Trending sets.** `Set.isTrending` + index, `GET /sets?isTrending=`, `POST`/`PUT /set`
  accept it. The admin Sets editor has a Trending checkbox. Mobile `useFetchTrendingSets`
  merges into the `#` Trending list, sorted by name. **The existing trending tags are now
  name-sorted too** (they were in facet order).
- **Mentions.** API: `PostMention`, `@[username](profileId)` in `utils/postMarkup.ts`
  (`parsePostMarkup` now returns `mention` parts; `stripPostMarkup` prints `@name`),
  `validatePostMentions` (non-BASIC, has an account, no block, max 10), and
  `mentionProfileIds` on create / edit (replace-the-set) / draft. `postInclude.mentions`.
  `GET /profiles?type=PUBLIC` matches `q` on username **or** displayName and drops blocked
  accounts. A featured Short refuses mentions. Mobile: both triggers are always configured
  in `TagTextInput` (`#` is ignored where tagging is off), plus a new `MentionSuggestions`.
  Its rows are **text only, like the `#` list** (`@username` over the display name), with
  no avatar. Mentions go into the same `inlineTagNames` map as `kind: 'mention'`, render
  pink with `@`, tap → `UserProfile`, and are never trailing chips.
- **Pushes** (`services/postNotifications.ts`). `post.mentioned` goes to newly mentioned
  people on create / publish / edit (edits push only the added ones), gated by
  `isRootPostViewable`. `group.post.created` goes to active MODERATORs on a root group
  post's create or publish. Reply push previews now strip markup (they used to show raw
  `#[Name](id)`).
- **GIFs.** `uploadImage` checks `pages` and opens animated sources with
  `animated: true` (no `rotate()`), writing animated WebP. `gif` format supported for the
  fallback. The resizer Lambda does the same and keeps `.gif` keys as GIF. Post routes
  accept `image/gif`. The S3 `ContentType` is now the output type. Mobile uploads GIFs
  untouched; the caption editor hides Aa and tap-to-type on GIF pages. Android adds Fresco
  `animated-gif` / `animated-webp` / `webpsupport` 3.6.0 (RN 0.83's Fresco).
  **Changed from the plan:** the size cap is **4 MB per GIF on the phone**, not 15 MB on
  multer. API Gateway → Lambda caps the request at ~6 MB, so multer stays at 5 MB.
  Several big GIFs in one post can still exceed the payload.
  **Verified locally** with sharp 0.33.5 on a 10-frame GIF: upload WebP 10 frames (70 KB
  vs 146 KB as GIF), CDN resize 10 frames. The Lambda layer's sharp version is unchecked.
  If it's old, animated output may not work; check on dev.
