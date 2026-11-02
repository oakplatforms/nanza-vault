---
tags: [nanza-mobile, nanza-api, solution-design, plan, posts, tags, composer]
---

# Inline hashtags — tags typed into the post, X-style

**Status:** **plan, 2026-11-01 (Skylar, voice + two X compose screenshots), all questions
answered. API side built 2026-11-01 (uncommitted, tsc + lint clean): `src/utils/postMarkup.ts`,
`validatePostTextTokens` + stripped grapheme count in `validation/post.ts`, the create / update
checks in `routers/post.ts`, OG description strips the markup. Mobile built 2026-11-01 (uncommitted,
tsc + lint clean on the touched files): `utils/postMarkup.ts` + `PostMarkupText` inline runs with the
trailing fallback in `PostItem` / `PostContentItems`, `TagTextInput` (react-native-controlled-mentions
3.1.0, `#` trigger) in the editor and text blocks, `TagSuggestions` panel under the text, ids derived
from tokens, combined cap. Add tags chip line + `TagPickerLayer` / `TagPickerSection` retired the same
evening (Skylar, after the first device look). First device round: the list drew OVER the text —
fixed by giving the draft scroll its measured content height while the list is open; divider moved to
`surface`, a step more air above the list. Pending the next device round.**
Spans nanza-mobile (composer + post rendering) and
nanza-api (body markup validation). When it lands, fold the durable part — the body markup,
the inline render, the retired tag screen — into
[[post-carousel-and-tagging|Post Composer — Carousel & Tagging]] and
[[tag-taxonomy|Tag Taxonomy]], then delete this file.

## Why

Today tags are a separate step: an **Add tags** chip under the editor opens the full-screen
tag picker (`TagPickerLayer`, the search filter's grouped sections — Trending, then Class,
Rarity, …), picks become chips, and the posted card trails them after the text as pink
`#hashtags`. Skylar wants tagging to happen **where the text happens**, the way X does it:

> "When I type a hash it should bring up a list, and the tag itself should show in the middle
> of the post… inside the copy we don't need to show the hashtag, we just want that text in
> pink. If you click the pink part it takes you to the tag."

Three consequences, all Skylar's:

1. **Typing `#` opens a flat suggestion list right under the text** — "like our search
   experience for add tags", but a plain list, not the grouped Trending / Class / Rarity
   sections. Typing `#call` lists every tag matching *call* (Calling, …). Tap one and it lands
   in the copy.
2. **The tag sits inline in the copy, pink, without the `#` sign.** A tag typed mid-sentence
   stays mid-sentence; three tags typed at the end just show at the end. Tapping the pink run
   opens that tag's page (or set page), as the trailing hashtags do today.
3. **Add tags goes.** "If I wanted to add three tags at the end I don't need to go to a
   separate screen — hashtag, hashtag, hashtag." The chip line and the composer's
   `TagPickerLayer` screen are retired once inline tagging works. (The search filter keeps its
   own copies of the shared section code.)

The picker's height follows the text: on a one-line post the list opens high and tall; on two
paragraphs it opens lower and shorter, so it stays next to where the `#` is being typed.

## How it works

### The body carries the tags as markup

Tags stay **supported tag values and sets** — the join tables (`PostSupportedTagValue`,
`PostSet`) remain the source of truth for feeds, tag pages and the 10-association cap. What
changes is that the post's text records *where* each tag sits.

The body (and each `RICH_TEXT` content item) is plain text with one token shape:

```
#[Calling](clx…id)
```

`#[` + the value's display name at the time of typing + `](` + its id + `)`. Reasons for this
shape over the alternatives:

- **Offsets were rejected.** A tweet-style span list (`{start, end, id}` beside the text)
  drifts on every edit, and the composer counts graphemes while the server counts the same —
  but a span's UTF-16 offset would not. A token travels with the text it marks.
- **Id-only tokens (`{{tag:id}}`) were rejected.** A body with the name in it stays readable
  wherever the raw string shows up (admin, OG descriptions, logs); the renderer still resolves
  by id and draws the *current* display name from the join, so an admin rename never leaves a
  stale name on screen.
- **It is `react-native-controlled-mentions`' native markup** (`{trigger}[{name}]({id})`), the
  library that gives the composer the inline behaviour for free — see below.

Rules:

- Every token's id **must be among the post's `supportedTagValueIds` or `setIds`** on the same
  request. The composer derives those id lists from the tokens (plus a locked ambient tag —
  below), so they always agree; the server rejects a token whose id isn't in either list.
- The **500-grapheme cap counts the rendered text** — `Calling`, not `#[Calling](id)` — so
  markup never eats a user's characters. Server and composer count the same way.
- **Replies carry no tags today and still don't**: the trigger is off in the reply composer
  and the server rejects tokens on a reply.
- A literal `#[x](y)` typed by a user whose `y` is no known id is plain text — the server
  doesn't reject the body, it just isn't a tag. (Accepted edge; nobody types that.)

### Backwards compatibility: joins not in the body still trail the text

Existing posts have join rows and no tokens. Rendering keeps today's trailing-hashtag
fallback for exactly those: **any join row whose id does not appear as a token in the body
renders after the text as a pink `#name` run**, as `PostHashtags` does now. No data migration;
old posts look as they do today, new posts have their tags where they were typed, and a post
that mixes both (an ambient tag from posting on a tag page + inline picks) renders both.

### The composer

`PostComposer`'s editor (the "Join the chat…" `TextInput`, and each extra text block in the
stacked preview) becomes a controlled mentions input:

- **Library:** `react-native-controlled-mentions` — a `TextInput` whose value is the markup
  string; it renders the display text with the pink runs as nested `Text` children, maps the
  caret across tokens, deletes a whole token on backspace, and fires a suggestions render-prop
  with the keyword after the trigger. Trigger `#`, markup `#[name](id)`. Verify it on RN 0.83
  / new architecture first (it is a thin controlled `TextInput`; nothing native). Fallback if
  it misbehaves: an in-house `useInlineTags` doing the same four jobs over `onSelectionChange`
  — budget a day extra for Android selection quirks.
- **The draft stays the markup string.** `draft.ts` persists it; `composeSnapshot` (the edit
  dirty check) compares it; `splitAssociations` reads the ids out of the tokens instead of
  `selectedTags`. Edit mode loads the post's body/text items as-is — they are already markup.
- **Pink inline run style:** the existing `posts.ts tagMention` (postBody size, `action`
  colour), applied through the library's part style so the composer and the posted card draw
  the tag the same.
- **Cap:** 10 distinct tag/set ids across all text blocks (the server's
  `MAX_POST_ASSOCIATIONS`). The eleventh distinct pick is refused with the existing cap
  notice; repeating a tag already in the copy is free (one join row).
- **Ambient tag:** posting from a tag or set page keeps that association locked on the join
  (today's `lockedTagIds`) without putting a token in the copy — it trails via the fallback,
  as it does now.
- **`#` typed with no pick** stays plain text: tags are supported values only, there is no
  free hashtag, and nothing commits on space — a tag is always a tap on the list (Skylar,
  2026-11-01).

### The suggestion panel

- **Where:** directly under the text block being typed in — anchored to the text's *content*
  height (the lines typed so far, via `onContentSizeChange`), not the editor box's fixed
  height — and filling down to the keyboard. That is the "based on the size of the text" rule:
  one line of text leaves a tall list high on the screen; two paragraphs leave a short one
  near the bottom. Attachments and the pinned cluster are covered while it is open; closing it
  (pick, or delete back past the `#`) restores them.
- **What:** a flat list, X-style (Skylar's screenshots, 2026-11-01): one row per hit, two
  lines — the **display name bold** on the first line, and on a subtle second line **the
  brand tag's label** (Hero, Class, Event, Rarity… — where X writes "Trending"); a set's second
  line says **Set**; a **child value's second line is its parent's display name** instead of
  the brand tag (Skylar, 2026-11-01 — e.g. *Boltyn* under *Warrior*, not *Class*). Text only,
  no art, no section headers, no chips. A thin rule separates the list from the text above
  it, and the list scrolls under the keyboard.
- **Data:** sets AND tags, "similar to Add tags" (Skylar) — the search filter's own reads,
  already paired in `TagPickerLayer`: `useSearchSupportedTagValues(brandId, keyword)`
  (`GET /supported-tag-values?brandId=&search=`, parents and children; the hook's groups carry
  each value's brand tag, which is the second line) and `useSearchBrandSets`
  (`GET /sets?brandId=&search=`, display name, short name and code) — flattened into one
  list, tags first then sets, each group alphabetical. Bare `#` (empty keyword) shows the
  brand's **Trending** values (`isTrending`), the way X leads with trending on a bare `#`.
- **Pick:** tap inserts `#[name](id)` plus a trailing space and closes the panel; the
  keyboard stays up.
- **Guests / BASIC authors:** no change to who may post; the trigger follows whoever can
  compose a root post.

### Rendering

`PostItem` / `PostContentItems` already render the text through `EmojiText` with the trailing
`PostHashtags` run as children. The text parse moves into a small shared util
(`src/utils/postMarkup.ts`: `parsePostText(body) → parts`, `stripPostMarkup(body) → plain`):

- The body splits into plain runs and tag runs; a tag run draws its join row's **current**
  display name in `tagMention` (no `#`), with the existing press → tag page / set page
  handling (`handleTagPress`, push not navigate on a tag page). A token whose id has no join
  row (deleted tag) draws its stored name as plain text.
- The trailing fallback run (joins without a token) follows, as today.
- Cards, the detail root, Show-more trees, the Stories pager, the share capture: all go through
  the same two components, so one change covers them. The accessibility label reads the plain
  text.
- Anything else that reads a post body raw strips the markup with `stripPostMarkup`: the
  web public pages / OG descriptions (nanza-web-app, if they show post text), push
  notification previews, admin lists. Inventory these during the build; each is a one-line
  call.

### API

`src/validation/post.ts`:

- `validatePostBody` and the `RICH_TEXT` branch parse tokens, count graphemes on the stripped
  text, and collect token ids. After `supportedTagValueIds` / `setIds` are parsed, every token
  id must be in one of them — else 400 `Post text references a tag that is not on the post`.
- Replies: a token in a reply body is a 400 (replies carry no tags).
- Featured posts (no body, no tags) are untouched.

No schema change, no migration, no types bump. The markup is a body convention; the DTOs are
as they are. `src/utils/postMarkup.ts` in the api is the same parser as the mobile one (small
enough to duplicate; not worth a package).

### Retiring Add tags

After inline tagging QAs clean: delete the `tagChipsLine` block and its styles from
`PostComposer` / `postComposer.ts`, the `'tags'` layer, `TagPickerLayer` and the composer's
use of `TagPickerSection` (the search filter keeps `TagPickerSection` if it still imports it —
check; otherwise it goes too). Vault: the "tagging screen" section of
[[post-carousel-and-tagging]] becomes history.

## Touched code (mapped)

**nanza-mobile**

- `package.json` — `react-native-controlled-mentions`.
- `src/utils/postMarkup.ts` — new: parse / strip.
- `src/components/posts/composer/PostComposer.tsx` — editor + text blocks on the mentions
  input; suggestion panel; ids derived from tokens; cap; reply guard; Add tags line removed
  (phase 2).
- `src/components/posts/composer/TagSuggestions.tsx` — new: the flat list panel.
- `src/components/posts/composer/draft.ts` — markup text (no shape change, noted).
- `src/components/posts/PostItem.tsx`, `PostContentItems.tsx`, `PostHashtags/index.tsx` —
  inline runs + trailing fallback.
- `src/styles/components/postComposer.ts` — panel styles; `posts.ts tagMention` reused.

**nanza-api**

- `src/validation/post.ts` — token validation + stripped grapheme count.
- `src/utils/postMarkup.ts` — new.

**nanza-web-app / nanza-email / nanza-admin** — `stripPostMarkup` wherever a post body prints.

## Phases

1. **Markup + render (api + mobile, ~1 day):** parser, server validation, inline runs with the
   trailing fallback. Old posts unchanged; the current Add tags flow still works (its picks
   trail, as now).
2. **Composer (mobile, ~2 days):** mentions input, suggestion panel, ids from tokens, cap,
   edit + draft. Device QA on both platforms — caret, backspace over a token, Android
   selection, the panel height under the keyboard on an SE and a Pro Max.
3. **Retire Add tags (mobile, ½ day):** delete the chip line and the composer tag screen.

## Open questions (Skylar)

All answered 2026-11-01 (Skylar) — ready to build:

1. ~~Auto-commit on space~~ — no; a tag is always a tap on the list.
2. ~~Bare `#` list~~ — Trending values.
3. ~~Sets in the list~~ — sets and tags, like Add tags.
4. ~~Children~~ — shown in the list; their second line is the parent's display name, not the
   brand tag.
5. ~~Row art~~ — text only, name over the second line.

## Log

- **2026-11-01 (night)** — First device round (Skylar): the list sat over the typed line. Cause: a
  ScrollView has no intrinsic height, so `flexBasis: 'auto'` collapsed it and the panel took the
  frame. Fix: `onContentSizeChange` feeds the scroll an explicit `height` while a `#` is open
  (still `flexShrink: 1` under the panel's three-row minimum). Divider `border` read white →
  `surface` (the comment card's rule); panel marginTop sm → md. Same round: Add tags retired —
  the chip line, the `'tags'` layer + its Modal, `TagPickerLayer`, `TagPickerSection`, the chip
  styles; `ComposerTag` now lives in `PostComposer`. `selectedTags` stays as the untyped
  associations (locked ambient tag, pre-inline joins on an edit) and only ever shrinks.
- **2026-11-01 (evening)** — Mobile built. Notes from the build: the library's native markup is
  `{#}[Name](id)` (trigger in braces), converted to ours at the editor boundary
  (`toMentionsValue` / `fromMentionsValue`); the editor shows a picked tag as its bare name in
  pink (`getPlainString`), matching the card; keyword allows two spaces for multi-word sets;
  the panel is the scroll's next sibling with the scroll hugging its content
  (`layerScrollHugged`) + a three-row minimum; the text nudge measures against the panel's top
  while it is open; keyboard-hide closes the list; an over-cap change to token-bearing text is
  refused rather than cut. Inline tags in a sentence draw the join's full `displayName`, not the
  chip's short name.
- **2026-11-01 (later)** — API side built: parser util, token validation on create and update
  (texts the post ends up with vs the ids it ends up with), cap counts the stripped text, OG
  description strips markup. No schema, migration or types change.
- **2026-11-01** — Plan written from Skylar's voice brief. Two X compose screenshots (bare
  `#` → trending list; `#` after a long body → shorter list) set the row shape: name bold,
  subtle second line = the brand tag's label, sets included. Same day, the three open
  questions closed: tap to commit (never on space), Trending on a bare `#`, children carry
  their parent's name on the second line.
