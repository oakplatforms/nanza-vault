---
tags: [nanza-mobile, solution-design, posts, chat]
---

# Posts Chat — "Join the chat" Composer, Post Cards, and the Content Renderer

> **Status:** Design only, 2026-08-01. Model and API contract in
> [[../nanza-api/solution-designs/posts|Posts]] § *Chat release*. Post-card **mocks are incoming**
> from Skylar and get folded in here when they land. No brand/homepage composer this release — the
> expanded composer view is deliberately its future shell.

## Overview

**Four screens get the chat** (decided): supported tag value detail, secondary tag value detail,
listing detail, and bid detail. There is no separate lot surface — **the listing detail screen
covers the Bulk case**: when it's showing a lot, the thread's home is the `BulkListing`
(`postableType: 'BULK'`), not the child listing. The current inline `PostThread` composer is
replaced by a progressive flow: a pill that says *Join the chat*, a snap-open composer on tap, and
(later) an expanded view that will become the brand composer.

**Feed cards lose their comment affordances** (decided): `FeedListingCard`, `FeedBidCard`, and
`FeedBulkCard` drop the comment count/button entirely — conversation lives on the detail screens,
not in the feed. This also retires the cards' count display, which neatly sidesteps the "counts
show 0 after the migration" concern; the API keeps emitting `commentCount`/`postCount` for old
builds regardless.

## The "Join the chat" input

- **Fixed to the bottom of the page** — the same treatment as CollectionDetail's search pill
  (`footerFloat`/`footerBar` + `CollectionSearchFooter` styling, the new-search glass pill). It is
  *only* a pill until tapped: no media row, no chicklets, nothing else visible.
- **On listing/bid/lot pages it sits ABOVE the action buttons** (Buy/Sell/Make offer). Those
  screens' white button bars keep the true bottom; the chat pill floats directly above them. Tag
  pages have no action bar, so the pill takes the bottom itself.
- Tapping opens the **composer modal** (below). Guests get the auth window instead, same as the
  current guest comment input.

### Guest taps open the auth window — never an error

This applies to **every** participation affordance, not just the composer: the heart, reply, and
the composer pill all open `AuthModal` when there's no account. A guest reaching for the heart is
expressing intent to participate — that's the moment to offer sign-up, not to tell them they
can't.

`GuestPostInput` had always done this (it's what a guest sees in place of the composer), but the
post card's heart and reply called `alert.error('Please sign in to …')` instead, so a logged-out
tap produced a failure alert rather than the sign-up sheet. Fixed 2026-08-08 in `PostItem` and
`PostDetailScreen`, both now using the standard `useAuthModal()` + `<AuthModal />` pair that
`GroupCircleCarousel` and `JoinChatFooter` already use.

The remaining `alert.error('Please sign in …')` calls in `PostThread` (submit, delete) are
unreachable defensive guards — a guest gets `GuestPostInput`, never the composer — so they stay
as guards rather than becoming modals.
- **Build it target-agnostic — the homepage likely mounts this exact component later.** The pill +
  composer take a `postableType`/`postableId` pair and per-surface options (chicklets source,
  action-bar offset) rather than knowing what screen they're on; when the brand feed arrives, the
  homepage passes `BRAND`/brandId into the same component instead of getting a sibling.

## The composer modal

**It snaps, it doesn't slide.** Same transition philosophy as the new search experience
(CollectionDetail's in-place flip: the overlay mounts on the *same screen* with no slide-up or
slide-in animation and no navigation — the input autofocuses so the keyboard rises immediately).
Tapping *Join the chat* snaps the composer into place over the page.

Stacked bottom-up:

1. **Text area** — autofocused on open. Text is a plain string in v1 (no formatting), and it's a
   **content item like any other**: a text block can sit above and below other items, multiple
   times, and is saved in the composer tree in order. A post with no attachments stays the simple
   `Post.body` fast path.
2. **Icon row under the input — four icons to start:**
   - **Image** — attach **one** image (v1 limit), resized for mobile (~500–600px).
   - **Link** — attach a URL.
   - **ELB** — one icon for all the reference types. Opens a picker using the **search screen's
     nav-tab component** (the Nanza / People / Groups switcher), relabeled **Cards / Listings /
     Bids** — cards being entities; lots surface within Listings. Pick a record and it lands in
     the tree as a reference item.
   - **Tags** — opens the tag picker (behavior below).
   The ELB and Tags surfaces work **like search's filter screen**: additional screens layered onto
   the overall composer (the SearchFilter pattern), not inline popovers.
3. **Chicklets row** (above the input) — secondary tag values offered as quick toggles once the
   post has at least one supported tag value associated (picked or ambient); tapping saves them to
   the post. Details in *Tagging* below.
4. **Composer view** (above the input and tags) — the ordered preview of the tree under
   construction: text blocks interleaved with the image, link, and reference items.
5. **Reply mode** — when composing a reply, a **summary card of the post being replied to** sits
   above the composer view, so the context is pinned while typing. (Replies are plain text, so the
   attach icons are hidden in this mode.)
6. **Expand** — grows to full screen. Same component contract as the future brand composer; this
   release it's just more room.

## Tagging in the composer (decided 2026-08-01)

The tag picker is **hierarchical**: it lists the brand's **brand tags** (class, print, …), and each
expands to its **supported tag values**. Selecting a supported tag value associates it with the
post (`supportedTagValueIds[]`). Once a post has supported tag values — picked or ambient — the
**chicklets row** offers their linked secondary tag values (*Boltyn*, *Kassai*); tapping saves
those to the post too (`secondaryTagValueIds[]`).

Per-surface behavior:

| composing from | Tags icon | auto-association | chicklets |
|---|---|---|---|
| **Listing / bid / lot detail** | **enabled** | none | from picked values |
| **Supported tag value detail** | visible but **disabled** | the page's value, automatically | that value's secondary values |
| **Secondary tag value detail** (e.g. the *Boltyn* chat) | visible but **disabled** | the page's value (it's the home) | **none** — the surface is itself a secondary tag; nothing further to refine |

- Disabled, not hidden — the icon keeps its place in the row on every surface so the composer
  reads identically everywhere; it's just inert where the page already decides the tag.
- On a tag page the auto-association is what makes the chicklets meaningful immediately: open the
  composer on *Warrior* and its secondary values are already offered, no picking required.
- **Association rule (revised 2026-08-01): one supported tag value max on every chat surface.**
  The tag picker on listing/bid is **single-select** — picking a value replaces the previous pick.
  Secondary chicklets come only from that one value's linked secondaries (server-enforced via the
  taxonomy join), **up to 25 per post**. No supported value → no chicklets. The old 5-value
  multi-select belongs to the future brand composer only, where posts fan out across brand tags.

## Post cards (from mocks, 2026-08-01)

Dark elevated card, rounded corners, in-thread on listing/tag detail screens. Anatomy top to
bottom, per the mocks:

1. **Header row** — circular avatar, username (bold), relative timestamp (`3w`, `2d`, muted).
2. **Body text** — with **tags rendered as inline colored mentions**, not a trailing hashtag row
   (see below). E.g. *"**Hala** photos will continue until morale improves or we get more
   **Mastery Pack Warrior** cards."* — both accent-colored (the magenta/purple link tone) and
   tappable mid-sentence, no `#` glyph.
3. **Content items**, in order. The mocks show the two headline renders:
   - **IMAGE** — full-width within the card, rounded.
   - **LISTING reference** — the listing's own photo with the **entity name overlaid** and a
     **white price pill** (`$85.00`) bottom-left — the existing `thumbPricePill` treatment, so the
     reference card reuses that styling rather than inventing one.
4. **Footer row** — reply (curved-arrow icon + count), heart (+ count), and **share top-right of
   the row**. Share is drawn in the mocks but stays **non-functional this release** (needs the
   referenceCode lift) — render it disabled or hidden behind a flag.

Interactions: tapping the card/reply affordance opens the post detail (post on top, replies under —
one level); heart is the like/SavedItem gesture (`viewerHasLiked` fill, optimistic toggle).

### Inline tag mentions (supersedes the trailing-hashtag idea)

The mocks settle the tag presentation: associated tags appear **inside the body text as colored,
tappable mentions** — *Hala*, *Kassai*, *Mastery Pack Warrior* read as part of the sentence. The
slim payload is still exactly sufficient: the renderer scans the body for each associated tag's
`displayName` (case-insensitive), colorizes matches in the accent tone, and wires the tap to the
route picked by `kind` with `id` as the param.

The post-detail mock settles the fallback: **both presentations exist**. The detail root shows a
trailing `#kassai` hashtag after the body (accent-colored, no card), while the compact cards
colorize the name inline mid-sentence. Rule: **inline mention coloring when the tag's name appears
in the body; trailing `#tag` hashtags otherwise — and the detail root always shows the trailing
hashtags**, so an association is never invisible. Same slim payload drives all of it.

## Post detail screen (from mock, 2026-08-01)

Opened by tapping a post card. Per the mock:

- **Floating dark-circle back (top-left) and share (top-right)** — the standard detail-screen
  floating header treatment (share non-functional this release, as everywhere).
- **The root post renders flat on the background** — no card surface: avatar/name/time header, the
  body at a **larger type size** than in-thread cards, trailing accent `#tag` hashtags, then its
  reply + heart counts.
- **Replies render as cards below it**, visually identical to in-thread post cards (avatar header,
  inline-colored mentions, reply/heart/share footer). One level only.
- The replies' footers show a reply affordance even though replies can't have replies — tapping it
  composes **a reply to the root**, keeping the thread flat (the reply summary card in the
  composer pins what you tapped for context).
- The fixed **join-the-chat pill is present here too**, in reply-to-this-post mode — the detail
  screen is the reply surface.

## The content renderer

One component walks a post's `contentItems` **in index order** and emits the matching block —
publishing-tool style, so a rich-text block can sit above an embedded listing:

| type | renders |
|---|---|
| `RICH_TEXT` | plain text block (v1 — no formatting); blocks interleave freely with other items |
| `IMAGE` | single image (`getImage` sizing; uploads capped ~500–600px) |
| `LINK` | a **Nanza link row** — Nanza link icon + linkable treatment inside the post, not an inline anchor. Tapping opens an **Instagram-style in-app browser modal**: a sheet with a web preview that never leaves the app. |
| `LISTING` / `BID` / `ENTITY` | the record's own card, drawn from the server-expanded `reference` object (feed-card data: entity image, price, condition). LISTING covers listings **and lots** — `reference.kind` picks `FeedListingCard` vs `FeedBulkCard`. `reference: null` → tombstone ("no longer available"). |

The renderer is shared between the thread cards, the post detail, and the composer preview — one
tree-builder, three hosts. Reference cards reuse the existing feed card components rather than
inventing new tiles.

## What this replaces / touches

- `PostThread`'s inline `PostInput` moves out of the scroll into the fixed pill + modal flow;
  the thread itself stays (cards, pagination, replies).
- Listing/bid detail screens: footer layout change (pill above the action bar); listing detail
  targets `BULK` when showing a lot.
- Tag detail screens: pill replaces the inline composer section.
- `FeedListingCard` / `FeedBidCard` / `FeedBulkCard`: comment count + affordance removed (and the
  `commentCount` prop threading through `GroupDetailScreen`, saved items, etc. goes with it).
- New: composer modal, chicklet row, content renderer, the Nanza link row + in-app browser sheet,
  the ELB picker (search nav-tabs relabeled Cards / Listings / Bids), and the Tags picker screen
  (SearchFilter-style layered screens). Picker scope — own listings/bids vs. anyone's — TBD with
  the mocks.

## Deferred

- **Entity surfaces get no chat UX yet** (decided): the entity modal and entity screens do NOT get
  the pill/composer this release, even though `ENTITY` is a valid post home in the API (comments
  used to live there) — don't add it by inference. Embedding a *card in a post* (the ENTITY
  reference item via the Cards tab) is unaffected; it's the entity's own thread that waits.
- **Brand/homepage composer** (affiliates) — the expanded view's real purpose, next release(s).
- **Share** on posts and tag pages — needs reference codes; separate lift.
- **VIDEO** content items — still reserved/rejected in the API.

## Implementation record (2026-08-01) — deltas decided while building

Built in nanza-mobile (uncommitted). Where this section disagrees with the spec above, this
section wins — these were Skylar's calls during the build.

- **The composer is a shared full-screen post modal** (`components/posts/composer/PostComposer`),
  presented with an RN `Modal` sliding up from the bottom — not a bottom sheet, not a navigation
  route. Mock layout: floating chevron-down glass circle top-left dismisses; a **"Post" pill
  top-right is the only submit** (no send icon, no expand button).
- **The tree IS the editor.** No separate textarea: text blocks are borderless inline inputs
  interleaved with attachments, so you type → add a photo → type → add a listing → type, and the
  post carries that exact order (`composer/draft.ts` serializes; text-only trees collapse to the
  `body` fast path).
- **Entry points differ by surface:** listing/bid/lot detail mount an **inline "Add a post…"
  pill at the chat section's foot** (in the scroll, `DetailChatSection` owns it and the reply
  state); tag pages pin a **fixed "Join the chat" pill** to the screen bottom
  (`JoinChatFooter placement="fixed" | "inline"` — same component, target-agnostic; the future
  brand feed mounts it with BRAND).
- **Attach circles ride the keyboard:** photo library / camera / ELB / tags, glass circles
  (`theme.spacing.hit`). The **tags circle is hidden (not disabled)** on tag pages — supersedes
  the earlier disabled-not-hidden decision. **Link attach is out of the v1 icon row** (the mock
  shows four circles); the LINK renderer + in-app browser sheet exist, so re-adding is one button.
- **ELB picker** is full-screen with the main search screen's toggle pill (Nanza/People/Groups
  styles from `searchFooter.ts`) relabeled **Cards / Listings / Bids**. Scope (decided): Cards
  search the catalog (`/search`, brand-scoped); **Listings and Bids list the viewer's OWN active
  records** — the motivating flow is "I'm selling this", and listings/bids have no server-side
  text search today.
- **Chicklets are pure FE state until create.** On a supported-tag-value page the page's own
  value's primary secondaries show immediately; nothing persists until POST, which sends **both**
  `supportedTagValueIds` (the ambient page value or the picked one) and `secondaryTagValueIds`
  explicitly.
- **Likes:** heart on post cards goes through the shared `SavedItemsContext` with a new `'post'`
  target (`SavedItem.postId`); count renders `likeCount` ±1 optimistic against `viewerHasLiked`.
  The unfiltered `/saved-items` excludes liked posts (old-build safety) — `type=post` opts in;
  whether the saved-items screen shows them is an open UX call.
- **Replies:** reply taps route out of the thread (`PostThread hideComposer/onRequestReply`) and
  open the modal in reply mode (pinned summary card, attach circles hidden, plain text).
  **Reply-on-a-reply retargets to the root client-side** as well as server-side. Inline editing
  is suppressed in external-composer threads (the modal has no edit mode yet); delete remains.
- **Renderer** (`PostContentItems`): ordered blocks — text, image (3:2, `getImage`), LINK as a
  Nanza link row opening the **in-app browser page sheet** (`InAppBrowserSheet`, WebView), and
  reference tiles drawn from the server-expanded lean `reference` (name overlay + white
  `thumbPricePill`; `null` reference → "No longer available" tombstone). Tag presentation:
  inline accent mentions when the tag name appears in the body, trailing `#tag` otherwise, both
  tappable through to the tag screens.
- **Feed cards** (`FeedListingCard/FeedBidCard/FeedBulkCard`) lost comment counts and
  affordances entirely; the optimistic comment-count cache bumps in `usePosts` went with them.
- **Runtime types** (`PostReference`, `HydratedPostDto`, `PostContentItemDto`) live in mobile's
  `src/types/index.ts` until `@oakplatforms/types` ships them in `shared.ts` (added there
  locally, missed the 0.1.80 publish — swap on the next types release).

**Not built yet:** the dedicated post-detail screen (root flat + larger type, replies as cards,
trailing hashtags always) — reading/replying currently happens in-thread; post cards don't
navigate anywhere on tap yet. Also still to come: dedicated icon glyphs (library/link), the
homepage brand composer, share, VIDEO.

### Final iterated state (2026-08-01, end of day) — supersedes conflicting bullets above

The implementation record above captures the first build; a long live-iteration session with
Skylar then reshaped the surface. Current truth:

- **Entry everywhere = "Add post"** — the standard ink `ActionPill` (the homepage "See all"
  recipe) on the right of every **Chat** heading row (`sectionHeadingRow` in `detail.ts`), on
  listing/bid sections and both tag screens alike. No fixed bottom pill, no input-look pills.
  Empty threads show a placeholder line ("No conversation here yet — be the first to post.").
- **Composer modal**: full-screen slide-up; standard floating dark-circle chevron-down dismiss;
  **"Post"/"Reply" pill top-right, primary-pink when submittable**; fixed-height
  "Say something…" editor (`bodyLg` = Medium 17, new typography slot); reply mode pins an
  identity-row summary (avatar, name, text, artwork) and the placeholder reads
  "Replying to <user>…". **Outline chicklets** (no label) + the four glass attach circles ride
  the keyboard. One text item + max one attachment (v1 cap; tree architecture intact beneath).
  Keyboard raises once via Modal `onShow` → settled → double-rAF focus (the search fix).
- **Post cards**: `ink`-surface column cards (one shade over the page), 40pt avatar identity
  row, text with trailing pink `#tags` (13pt `label`), attachment, footer = reply arrow(+count),
  outline heart(+count), owner's trash far right. **Tapping the text opens PostDetail.**
- **`PostDetail` screen** (route registered): the mock's layout — root post FLAT on the page
  (no card, `bodyLg` text, full footer; reply arrow opens the modal in reply mode), replies
  below as standard cards with the **minimal footer** (no reply/heart on replies — owner delete
  only; same rule for replies nested in threads). Root delete pops the screen.
- **Tag screens**: Chat heading row + capped 300pt inner-scroll thread (own 10-a-page load-more)
  + Add post in the heading. Profile nav from post identities enabled everywhere.
- `JoinChatFooter` placements: `button` (standard), `inline`/`fixed` (legacy glass pills, kept
  for the future brand feed), `none` (entry-less modal host — PostDetail's reply arrow).
- API delta: reference expansion now emits `price` as a real number (Decimal string bug);
  client formats via `formatPriceAsDollar` either way.

### Single-item v1 + groups deferred (2026-08-02)

- One attachment per post: image (library/camera), link (paste-a-link input), card, listing
  (incl. lots), or bid. Attaching hides every attach circle; removing restores them. LINK
  renders as the Nanza link row → in-app browser sheet.
- GROUP (renamed from gallery) is schema-reserved and API-rejected this release; the renderer's
  group-carousel branch exists but nothing emits it.
- Deleting a post deletes its replies (confirm copy warns with the count); no more tombstones.

### Reference tile designs (2026-08-02 mocks)

- **Listing / bid / lot** — the detail-hero read as a tile: dark rounded square,
  the record's photo letterboxed center, name (heading) + white `thumbPricePill`
  overlaid bottom-left. Lots page their child images (lean `childImages` keys on
  the bulk reference, capped 6, from the expansion).
- **Link** — Open Graph card: bordered row, OG image left (link-glyph placeholder
  when none), bold title + muted domain right. Backed by a new
  `GET /link-preview` API endpoint (server-side OG scrape, 5s timeout, SSRF
  guard on private hosts; failures return domain-only data so the card renders
  its placeholder instead of erroring). Client caches previews for a day.
- **Entity** — the simple search-row card: card image left, name + set right.
- Composer previews share the renderer, so every tile looks identical before and
  after posting. Reply mode pins the real post card titled "Replying to @user".
- Types 0.1.84 (unpublished): `PostBulkReference.childImages`, `LinkPreviewDto` —
  mobile uses local mirrors until installed.

### Entity references carry a FLATTENED price (2026-08-09)

An ENTITY reference is a **lean, flattened shape, not an `EntityDto`**. The API's expansion
(`postReferences.ts`) selects `product: { select: { price: true, number: true } }`, then
destructures `product` **off** and emits `price` and `productNumber` at the top level — price
null, never 0, so a card can tell "free" from "unpriced"; `productNumber` null when there's no
number, and the line is simply omitted. The reference genuinely has no `product` object.

`productNumber` joined the projection 2026-08-19 (the card's identity line is name + number +
set; only the set had been on the wire, so the composer and the posted card both showed the set
alone). `CardPickerLayer` carries `entity.product.number` into the local preview the same way
it carries `price`; `PostContentItems` passes it as the lockup's `metaLine1` (full row) and as
the first sub-line in tile/rail mode. Until `@oakplatforms/types` ships the field, `PostReference`
in `src/types/index.ts` intersects it onto the entity variant — drop on install.

That flattening is the source of a recurring class of bug: **anything reading
`entity.product.price` off a post reference gets `undefined` and renders a dash.** Three
places had to be taught the difference, and any new consumer will too:

- **The post's own entity row** reads `reference.price` (correct by construction). An entity
  with no product price now shows an **en dash in the pill** rather than hiding the pill —
  keeping the row's layout stable between priced and unpriced cards. Dash glyph matches the
  other unpriced surfaces (`EntityCard/List`, `CollectionGridCard`).
- **The entity modal** (`EntityModal`) re-fetches full detail, and its fair-market line now
  follows the same fallback discipline as `lowestAsk`/`highestBid`: prefer
  `entityDetail?.product?.price`, fall back to the route-param snapshot. It previously read
  the raw prop only, so **every post-opened card showed a dash** even though the post row
  beside it showed the price. Fixing it in the modal (not the call site) covers the other
  entry points that forward a nested entity whose parent query may not have included
  `product`.
- **The navigation call site** re-nests `product: { price: reference.price }` so the modal
  paints the price on the first frame instead of dashing until its fetch lands. The modal
  prefers the fetched value, so a stale snapshot corrects itself.

Note the composer preview builds its own local reference and must carry `price` too, or a
card shows a dash in the composer and a price once posted.

> **Type drift:** `@oakplatforms/types` (0.1.87) still declares `lowestAsk`/`highestBid` on
> `PostEntityReference` and has **no `price`**, so these call sites fail `tsc` until the
> package ships `price?: number | null`. Runtime is correct on both ends; only the
> declaration is stale.

### Embeds (2026-08-02)

- New content item type **EMBED**: the raw code/share-url lives in `body` — **no platform enum,
  no extra column**. Platform is DERIVED from the code on both ends (server validation + client
  player), so supporting a new platform is a code change, never a migration.
- **v1 is YouTube ONLY** (narrowed 2026-08-02). Inline players are WebViews, so an arbitrary
  host in the frame would run its own scripts inside the app — the allowlist is the boundary,
  and it stays exact-match on youtube hosts at both ends. Adding a platform means adding a
  constructed player URL for it, not loosening the list.
- **Everything else is a LINK, not an error.** TikTok, X, and the rest fall through to our own
  preview card (`/link-preview` OG data) opening the in-app browser sheet.
- **ONE paste field, type DERIVED (2026-08-02).** There is no embed circle and no "link vs
  embed" choice — the composer has a single link field, and the url decides: YouTube → EMBED
  (inline player), anything else → LINK (preview card). EMBED-vs-LINK is a backend distinction
  and stays there; surfacing it as two buttons made the user classify their own url for no
  gain. A pasted `<iframe>` snippet still resolves via its `src`.
- **Player clicks: native controls stay native, navigations escape (2026-08-04).** Everything
  that stays inside the player — play, pause, scrub, YouTube's own expand/fullscreen — is
  untouched. Any click that would LOAD another page into the small frame (logo → home, title →
  watch page) is intercepted (`onShouldStartLoadWithRequest` allowlist + `onOpenWindow` for
  `target="_blank"`) and opened in the in-app browser sheet on that URL, pausing the inline
  video first. Tap-counting ("second tap opens the sheet") was tried and rejected — it broke
  pause.
- **Playback position survives the browser sheet (2026-08-04).** The sheet's youtube.com page is
  heavy enough that iOS reclaims the covered player's web process; the WebView comes back
  reloaded. The shell page streams `currentTime` up to RN (iframe-API `listening` handshake →
  `infoDelivery` → `postMessage`); RN keeps the last position in a ref and, on any
  `onLoadEnd` with a remembered spot, re-cues the video there (`cueVideoById` + one delayed
  retry). `onContentProcessDidTerminate` reloads explicitly so the card never sits blank. The
  player is memoized (`React.memo` + stable `onOpenLink` + memoized `source`) so the sheet's
  open/close state never re-renders it either.
- **No secondary-tag chicklets (2026-08-03).** The composer no longer offers secondary tag
  selection. Associations are ambient ONLY: a supported-tag-value page sends its own value as
  `supportedTagValueIds`; a secondary-tag-value page sends nothing, because the post's home
  (`postableType`/`postableId`) already ties it to that tag. The backend still accepts
  `secondaryTagValueIds` untouched — the next release writes them from the hashtag/mention UX,
  which is why the API side was deliberately NOT narrowed.
  - Gotcha for whoever builds that: `validatePostTags` REJECTS secondary ids sent without a
    supported id ("Secondary tag values require a supported tag value association"). Writing a
    self-association from a secondary page needs that rule relaxed first.
- **Post text caps at 560 graphemes** — twice a tweet, enforced on both ends and counted the
  same way (`Intl.Segmenter`, so an emoji is one character). Client truncates on paste rather
  than rejecting. The count must stay grapheme-based: `TextInput`'s `maxLength` counts UTF-16
  units and would split an emoji, disagreeing with the server. Note the API has TWO ceilings —
  `body` (560) and `RICH_TEXT` content items (5000) — and the composer switches paths the
  moment you attach something, so the client cap is what actually keeps them consistent.
- **Reference tiles and post cards `push`, never `navigate` (2026-08-03).** A post can point at
  the same KIND of screen you're already on (a listing post read on a listing detail).
  `navigate()` reuses the existing route of that name, silently replacing the screen you came
  from so Back skips to the tab root. Same trap for PostDetail opened from a PostDetail.
- **`CardGrid` column math (2026-08-03).** The grid pads itself (`cardGridSection`,
  `spacing.sm`) AND sizes columns off the full window width. A host that also pads its content
  must declare that via `horizontalPadding`, which the grid ADDS to its own — otherwise the
  columns are sized for space that isn't there and wrap to one. The width is also `Math.floor`ed:
  the unrounded math summed to *exactly* the available width on a 393pt screen, so any rounding
  up wrapped the row. If a 2-column grid shows 1 column, this is the first thing to check.
- **Keyboard: scroll ON DEMAND, never up front (2026-08-03).** The composer does NOT shift when
  the keyboard opens, and there is no `keyboardVerticalOffset` — the view opens showing the
  replied-to card from the top. It scrolls only when the text you're typing would actually go
  under the keyboard: `keyboardDidShow` records the keyboard's `screenY`, and on every keystroke
  (and content-size change) the editor is `measureInWindow`'d — if its bottom + a 12pt gap
  crosses that line, the scroll nudges by exactly the overlap. Everything above slides up as a
  consequence, which is what keeps the typed line visible.
  - Two traps: measure AFTER layout (`requestAnimationFrame`), or the editor still reports its
    old height and the overlap reads as zero; and `scrollTo` needs an absolute `y`, so the
    current offset must be tracked via `onScroll` rather than assumed to be 0.
  - Rejected alternatives: a `keyboardDidShow` one-shot `scrollToEnd` (fires once, never as you
    type) and `keyboardVerticalOffset` (shifts the whole body up front, lifting the attach row
    off the keyboard — the visible symptom was the attach icons jumping up the screen).
- **Composer layout: ONE scroll, nothing capped (2026-08-03).** The editor grows with its text
  (`minHeight`, not a fixed height — a fixed one parked the attachment a full block below a
  one-line post), and the attachment sits a fixed 25pt under the LAST line. Content items
  render at FULL height with no inner scroll: the modal's own ScrollView is the only one, and a
  long post scrolls underneath the attach row. That row and the link pill are siblings OUTSIDE
  the ScrollView, which only stays pinned because the scroll body uses `flex:1` — with
  `flexGrow:1` a long post grows the body and shoves the row off the bottom. Same
  `flexGrow`-vs-`flex` trap as the pickers.
- **In-app browser: never hand off to another app.** `x.com` answers mobile requests with a
  `twitter://` app-deep-link redirect the WebView can't load, which kills the page ("redirect
  to a URL with a scheme that is not HTTPS"). The sheet rewrites `x.com` → `twitter.com` and
  sends a desktop UA so the open-in-app interstitial is never served, plus blocks non-http(s)
  navigations outright. Reach for the UA/host rewrite before the navigation block — blocking
  alone leaves a dead page.
- Player: 16:9 full-width WebView, unparsable codes degrade to the link card. Two gotchas the
  mocks won't show you — the post card's card-wide TouchableOpacity swallows taps on the player
  unless it claims the touch responder (a WebView isn't a nested touchable and doesn't win the
  tap on its own), and the WebView's own scroll view bounces vertically inside the card unless
  scrolling is disabled on BOTH the WebView and the HTML page.
- Types 0.1.85 (EMBED enum value only) pending publish.
