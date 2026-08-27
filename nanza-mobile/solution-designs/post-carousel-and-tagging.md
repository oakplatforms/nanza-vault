---
tags: [nanza-mobile, solution-design, plan, posts, tagging]
---

# Post Composer — Multi-Content Carousel & Tagging Screen — Plan

> **Status: PLAN, 2026-08-11 (Skylar, voice session).** Mobile half of
> [[../nanza-api/solution-designs/post-carousel-and-tagging|the api plan]] (model, caps, swap
> endpoint, group-tag visibility). Extends [[posts-chat|Posts Chat]]; supersedes its
> single-attachment v1 composer and revives tag selection (removed 2026-08-03) in a new form —
> a multi-select tag screen instead of the old single-select `TagPickerLayer` + chicklets.

## Third revision — the tag screen regroups (2026-08-14, voice; implemented, uncommitted)

- **Missing primaries fixed**: the flat fetch pulled the first 200 rows UNFILTERED and kept
  the primaries client-side — on a big taxonomy most primaries sit past the page, so the
  screen showed a fraction of what the home trending rail (server-side `isPrimary=true`)
  shows. The picker now sends `isPrimary=true` itself; the page holds nothing but showable
  values.
- **Grouped like the search filters** (Skylar: "we should have our labels… ours just shows
  them randomly"): one labeled section per tag (Color, Print, Rarity…) — the filter screen's
  `filterOpenSection`/`filterSectionTitle` dress over the same chip grid, minus the
  per-section Reset (the global Clear covers it).
- **Parents with children get their own section AND stay chips** (Skylar, second pass
  2026-08-12; revised 2026-08-17 — originally section-only, but the parent itself is a
  valid pick): a primary parent with children (e.g. Warrior) appears as a selectable chip
  in its tag's grid (Class) like any other primary, and ALSO renders as a section — its
  display name is the label, its children (ALL of them, via the include — they need no
  isPrimary of their own) are the selectable chips — slotted right after the parent's tag
  section. A primary child already shown under a parent section is deduped from the flat
  rows; one whose parent isn't primary falls back into its tag's grid.
- **Data rides `/brand-tags`, not the flat value list** (Skylar: "similar to how we do in
  search filter"): the picker fetches the search filter screen's own shape —
  `?brandId=&usePagination=false&valuesIsPrimary=true&include=tag&include=supportedTagValues.children.child`
  — so section order and labels mirror the filters, and the shape works against the deployed
  api for labels immediately. api additions (need a deploy, which is blocked locally —
  `.env.dev` is an empty stub, the deploy env lives in Skylar's shell): the brand-tags route
  now (1) preserves a caller's NESTED include on `supportedTagValues` instead of overwriting
  it (children.child is how parents get their child rows) and (2) takes `valuesIsPrimary=true`
  to narrow the nested values + their `_count` to primary supported tag values server-side
  (named to keep clear of BrandTag's own isPrimary). The picker keeps a client-side isPrimary
  filter as the graceful path until the deploy lands. Until then parent sections don't form
  (children aren't in the deployed payload) — labels and tag sections already work.
  **Superseded 2026-08-18 by the slim facets read** — see [[tag-taxonomy#Slim facets read +
  paged sections (2026-08-18)|Tag Taxonomy → Slim facets read]]: the picker now calls
  `/brand-tag-facets?brandId=&valuesLimit=6&valuesIsPrimary=true&includeChildren=true&valueIds=…`
  (display fields only, six values per tag + count, committed picks pinned in), and
  `TagPickerSection` pages the rest of a tag's primaries in on "Show more" and draws each
  loaded parent's children section itself.

## Second revision — per-type budgets & picker polish (2026-08-12 PM, voice + chat; implemented)

- **Per-circle caps** (mirrors the api's second revision): 1 library photo, 1 camera snap
  (IMAGE drafts carry `source` to tell them apart), 5 texts / links / listings+bids / cards,
  **1 collection** (briefly five, re-decided same afternoon). The cards budget is
  **EITHER/OR: one collection or up to five individual cards, never both** (server-enforced
  too) — the circle disables the moment either bucket is spent, the rows' Add button disables
  once any entity rides the post, and the card multi-select zeroes out once a collection does.
  `draftTypeCounts` in draft.ts is the single tally; `capMessageFor` names the exact
  overflowing budget. Pickers get per-bucket `maxSelectable`; replies keep the flat
  one-attachment total.
- **Composer gallery preview fixed**: the rail is drawn by the composer itself (the renderer's
  rail sat under the preview's `pointerEvents="none"` wrap, killing its scroll) — scrolls
  left/right, and **each child tile carries its own X** (remove any one at a time; an emptied
  gallery leaves the tree).
- **Composer polish** (same afternoon): promote-to-primary is **dormant by default** — the
  first attachment IS the primary; deleting the current primary is the one moment the promote
  tap arms (next item inherits the lead by position, tap another to take it instead; a
  promotion or an emptied tree disarms it). New T blocks **scroll into view** (they mounted
  under the keyboard). Deletable text blocks sit on a **whisper of a background fill**
  (`textBlockForm` — `ink` over the page background, the palette's near-invisible step;
  walked from outline → surface fill → darker `ink`, with `gutter` vertical padding for
  field-like height); the main editor stays bare. On the post detail the tree's text renders
  at **bodyDetail** (`PostContentItems detailText`) so every text block matches the primary
  text's scale.
- **Tag presentation SETTLED** (Skylar, 2026-08-12 PM, after many passes — outline, white
  fill, pink fill, 14pt labels, above-the-content placement all tried and walked back): one
  chicklet row wearing the **homepage "See all" pill recipe verbatim**
  (`cards.ts carouselActionPill/Text` — ink fill, overlayText label, roomier padding),
  sitting **below the content on cards AND the detail root** (shared `tagsRow` JSX; detail
  keeps extra air above the row, 8pt gap between pills). The **pink inline/trailing #hashtag
  UX is removed entirely** (`tagMention` lingers unused). Post text on both views is
  `postBody` (Medium 15/19 — a SemiBold pass walked back; the unused `postBodyBold` slot
  remains in the scale). The composer's chips stay in the pinned bottom cluster, unchanged.
  Cards also show the **first text run only**; the detail tree still renders all blocks, at
  a `gutter` stack gap.
- **Picker keyboard clearance — MEASURED, not avoided** (settled after several attempts):
  the pickers' search pill pads by the keyboard's measured height
  (`usePickerKeyboardPad` — Will* events on iOS, Did* on Android; pad = height while up,
  else the safe-area inset), and the pickers' KAV was REMOVED outright (once the pickers
  opened keyboard-closed the KAV started working and STACKED on the measured pad — the pill
  sailed half the page up). KeyboardLift also failed here (frozen Android window / iOS
  never-resizing windows). **Pickers still open KEYBOARD-CLOSED** (`openPickerLayer`
  dismisses first).
- **The composer toolbar LIVES AT THE SCREEN BOTTOM, always** (Skylar, 2026-08-12 PM — the
  final simplification): the tags line + attach circles no longer ride the keyboard at all —
  the keyboard simply covers them, the draft gets the full height, and dismissing (tap
  outside / drag, both now clean) brings them back. **No auto-focus on open** — the composer
  opens resting; the keyboard rises only when the user taps into the editor. The KAV now
  wraps only the scroll body + link field; the settled-focus effect, KeyboardLift ride, and
  keyboard-aware icon padding are all gone.
- ~~"Say more…" ghost replaces the T circle~~ — **WALKED BACK (Skylar, 2026-08-12 PM, voice:
  "the say more experience doesn't really work well")**. The ghost and all its logic are
  gone; the **T circle is back, leading the attach row** (root posts only, disables at the
  5-text cap). A tap adds a **resting** text block at the tree's foot — NO auto-focus, the
  keyboard rises only when the user taps into it. The block wears the **form dress again**
  (`textBlockForm`: the whisper `ink` fill + a "very very subtle" 1pt outline in
  `glassBorder` — deliberately a step quieter than the entity/link cards' `border`) with the
  standard **circle X** to remove; blur-self-removal is gone (empty blocks just don't
  serialize — `serializeDraft` already dropped them). Placeholder is **"Say more…"**, not the
  main editor's "Join the chat…". **Tapping into any text block scrolls it up into view above
  the keyboard** (`focusedInputRef` — the keyboard nudge now measures the FOCUSED input, not
  always the main editor; onFocus re-checks for keyboard-already-up focus hops). Everything
  else from this session's simplification stays (toolbar at screen bottom, no auto-focus on
  open, clean dismiss).
- **Composer-only spacing**: `previewList` marginTop 25 → `gutter` (Skylar: the first
  attachment sat too far off the text). Posted views unchanged.
- **Post reference cards stay fresh**: every mutation that already lazily invalidates
  `['Feed']` for listings/bids/lots/cart now also invalidates `['Posts']`, and
  `invalidateCollectionQueries` (the collection-change single home) adds `['Posts']` +
  `['Replies']` — a collection's card count in a post tracks membership changes on the next
  thread visit. The collection
  rows' button is the group row's Join recipe (`Badge` + `carouselActionPill` — the sm Button
  rendered its label invisibly there), labeled **"Add to post"**, muted when the slot is
  spent; rows got list gap; the selected entity row tint walked down to `ink`.
- **Cards picker re-reworked**: the **Collection / Nanza tabs are back** with the search pill.
  The Collection tab lands on your collections in the **Add-to-Collection row dress**
  (banner cover, name + count) with an **Add** button where that screen's stepper sits —
  attaches the whole library and closes; tapping the row opens the in-collection card list
  (multi-select). Nanza tab = the catalog behind the ≥2-char gate, same multi-select. One
  shared selection set across tabs, committed by Clear/Add.

## Implementation record — galleries release (nanza-mobile, 2026-08-12, uncommitted)

- **Pager removed** (`PostItem`): cards render text + the primary attachment only; the
  **detail root renders the whole ordered tree** via `PostContentItems` (text blocks, images,
  galleries) — skipping the joined text run there to avoid double-rendering, with the trailing
  `#tags` line kept after the tree. `galleryItem` width 230 → 200 (readers see the rail now).
- **Link card reverted** to the original bordered row — gradient wash gone, `radius.lg` /
  1pt border / 38% flush image restored; the functional `fill` (rail-height stretch) stays;
  the entity row keeps its gradient dress.
- **Composer**: caps count LEAVES (`draftLeafCount`); the **T circle** (literal glyph — no
  text icon exists, **leads the attach row**) appends editable RICH_TEXT blocks, free of the
  cap, clamped by the same grapheme limit; the preview is the ordered tree stacked (galleries
  draw their own rail); promote-to-primary swaps with the first ATTACHMENT (text blocks may
  precede it). The **"Add tag" chip** (white, no X) replaces the tag circle on the chips
  line: **inline-left while no tags are picked, then PINNED to the right edge** with the
  chips scrolling beneath it (absolute wrapper masks them; the row's trailing padding lets
  the last chip scroll clear); disabled at 10. Chips line steps `xs` (up from `xxs`) off the
  attach circles.
- **Shop picker** (`ElbPickerLayer`): multi-select across BOTH tabs keyed by referenceId,
  check-circle + dim selection dress, `maxSelectable` = the post's remaining slots, Clear/Add
  commit bar; Apply lands 1 pick as a plain reference, 2+ as ONE gallery draft.
  `GroupAddItemsSheet` (the group rail's reuse) adapted: batch-applies each idempotent add.
- **"My Library"** (`CardPickerLayer`): two stages — your collections as rows (primary first),
  then inside one: search + multi-select entity rows (`pickerRowSelected` tint) with Clear/Add,
  or **"Add collection"** (shown while nothing is picked) attaching the whole library as a
  COLLECTION draft. The My collection / Nanza toggle is gone. Banner/tab dress deferred.
- **Renderer**: `collection` reference branch — the entity row's gradient dress with the card
  count as meta, navigating to **`CollectionDetail`** (the real banner + 3-column screen; its
  `isOwn`/route-`accountId` logic already covers viewing someone else's). This retired
  `UserCollectionDetailScreen` entirely (2026-08-12 PM): the profile had already moved off the
  old row-layout screen, the post nav was its last reference, so the screen, its route, and
  its param type are deleted.
- **Types**: `PostReference` locally shadowed as `base | PostCollectionReference`, item `type`
  widened with `'COLLECTION'` — drop with 0.1.90. Service sends nested `children` on GROUP
  items.
- Verified: tsc/eslint clean on touched files (only the documented 0.1.87 `price` drift
  remains, incl. the picker preview shapes — clears when 0.1.90 installs).

## Revision — galleries, library picker, text blocks (2026-08-12, voice + chat)

Supersedes the pager/carousel direction below where they conflict. Mirrors the api revision
(see [[../nanza-api/solution-designs/post-carousel-and-tagging|api plan]] § Revision):

- **The full-width paged carousel (1/N counter) is REMOVED everywhere** — cards and detail
  render text + the **primary attachment only**; remaining flat attachments are not paged.
  Multi-item **galleries are GROUP content items** and render as the **small tile rail** (the
  composer-preview/edit-mode rail, sized down) for readers too.
- **Link card styling reverts** to the original bordered row (no entity gradient wash / fill
  slot); the entity reference row keeps its gradient look. Post + post-detail surfaces only.
- **Composer text blocks**: a new **T text circle** (last in the attach row) adds extra
  RICH_TEXT blocks, so a post reads text → primary → text…
- **Tag entry moves off the circle row** (chat, 2026-08-12): with T added, the tag circle is
  replaced by an **"Add tag" button on the chips line** — chip-shaped like the tag chips but
  **white instead of grey, no X**, floating right once chips exist. Opens the tag screen;
  disabled at the 10 cap.
- **Shop (ELB) picker → multi-select**: pick listings AND bids together (cap = the post's
  remaining leaf slots, 4 with a primary present), Apply creates ONE gallery (GROUP) item.
- **Cards picker → "My Library"**, modeled on the add-to-collection view: your collections
  first, click into one, then EITHER multi-select cards (→ entity gallery) OR "add the whole
  collection" (→ new COLLECTION content item — one collection max per post, no
  multi-collection select). Profile-style tabs / collection-banner treatments floated for the
  in-collection view — design felt out on-device.
- Create-only; post-edit flows deferred.

## Implementation record — nanza-mobile (2026-08-11, uncommitted)

Built same-day as the api half. Deltas and specifics:

- **Composer** (`PostComposer.tsx`): attachments are a true list — up to 5 on roots, 1 on
  replies (`MAX_ATTACHMENTS` / `REPLY_MAX_ATTACHMENTS`, matching the api). The attach circles
  now render in reply mode too and **disable at the cap** (`iconButtonDisabled`, existing
  class) instead of hiding; removing an item via its X re-enables. The link paste field works
  in reply mode as well.
- **Tags**: `selectedTags` state seeds from the ambient page value and grows via the picker;
  submit sends the flat list (`supportedTagValueIds`), roots only. A sixth **tag circle**
  (`tag` glyph) sits at the end of the attach row — root posts with a brand only — and
  disables at `MAX_TAGS = 10`. Chips row (`tagChipsRow` in postComposer.ts) reuses **search's
  applied-filter chip recipe verbatim** (`filters.ts filterChipOpen*`) — tap a chip to remove.
  The ambient value is a removable chip like any other (removing it just doesn't send it; the
  post's home still ties it to the page). **Placement (Skylar, 2026-08-11 voice): docked in the
  pinned bottom cluster** — inside the same `KeyboardLift` container as the attach circles,
  directly above them — not in the scroll body; the draft scrolls beneath and the picks stay in
  view. Resolves open decision 3.
- **TagPickerLayer**: rewritten single-select → multi-select, then **reworked same-session
  (Skylar): the brand-tag drill-down level is GONE**. The screen is a flat grid of the brand's
  supported tag values in the search filter screen's chip-grid dress
  (`filters.ts filterGrid/filterGridItem[Selected]`): one `?brandId=&isChild=all` fetch
  (limit 200), client keeps **parents + primary children only** and hides purely numeric
  labels (pitch 1 / cost 2 are filter facets, not what a post is about). Header shows `n/10`;
  selecting past the cap alerts ("You've already selected 10 tags") while deselection always
  works.
- **Serialization** (`draft.ts`): `serializeAttachment` is gone — `serializeDraft` takes the
  whole tree (text first, then attachments in added order = primary first) and collects
  `images[]` in IMAGE-item index order. `Post.ts` appends every file under the same `image`
  multipart field (the api pairs by order); `CreatePostPayload.image` → `images`.
- **Cards render the primary only** (`PostItem.tsx`): `primary = find(isPrimary) ?? first
  attachment` (legacy fallback matches the api's rule). The **detail root** adds a horizontal
  carousel of the remaining attachments (the renderer's `galleryScroll`/`galleryItem` rail).
  **Owner tap on a carousel item** → `alert.confirm` "Make primary" →
  `useSetPrimaryContentItem` (PUT `/post/:id/primary-content-item`), invalidating the thread
  live; non-owners keep each item's normal behavior (image, link sheet, reference nav). The
  owner's wrapper is `pointerEvents="none"` inside, so the prompt wins over item navigation —
  the "View / Make primary" option sheet stays an open UX call.
- **Types**: `PostContentItemDto.isPrimary` mirrored locally in `src/types/index.ts` until
  0.1.89 installs. Gotcha: `HydratedPostDto` is `PostDtoBase & {…}` and the base carries its
  own `contentItems` — an unannotated `post.contentItems` resolves elements to the base type
  and hides the hydrated extras; consumers must annotate (`const items: PostContentItemDto[]`).
- Verified: `tsc --noEmit` and eslint clean on touched files (the `PostEntityReference.price`
  errors are the documented pre-existing 0.1.87 type drift).

Same-day refinements (Skylar, 2026-08-11 voice, second pass):

- **Composer preview mirrors the detail shape**: the draft's first attachment leads full-width
  and slots 2–5 page the detail's own sibling rail (`posts.ts galleryScroll`/`galleryItem`),
  instead of stacking — the post is *felt* as primary-plus-carousel during creation, not just
  after. One `renderPreviewCard` (content + the white X chip) serves both slots.
- **Feed/thread cards grew a tag chicklet row** (`PostItem`): every association — parents
  included, since the picker offers parents only and a children-only row left most posts
  tagless — renders as the standard outlined tag pill (`Badge variant="tag"`), under the
  content and above the action icons. `postTagsRow` (posts.ts, previously unused) spaces it
  asymmetrically: snug to the content (the card's own 8pt gap), extra breath (14) before the
  icons. Cards only; the detail root keeps its inline child-value #tags.

Third pass (same voice session) — the tagging screen adopts the filter screen's commit model:

- **TagPickerLayer is apply-to-commit now**: toggles land in local `pendingTags` (seeded from
  the composer's committed set); the filter screen's Clear/Apply bar — same container, fade and
  button pairing — pushes the whole set back via `onApply`, and the dismiss discards. The back
  arrow became the standard **down-chevron dismiss** (icons.base), and the header count is
  always on (`0/10` included, pinned right by layerTitle's flex).
- **Values shown: `isPrimary` only, parent or child alike** — isPrimary is the curation flag,
  so the numeric-label heuristic is retired.
- **Clear/Apply hidden until actionable, in BOTH experiences** (tag picker and
  SearchFilterScreen): visible when something is pending *or* something already applied needs a
  Clear committed over it — the second clause keeps "clear pre-applied filters" applyable.
- **Docked chip height**: `tagChipCompact` (xxs vertical padding) layers over `filterChipOpen`
  — the search recipe's 12pt padding read as a button in the pinned cluster — and `tagChipsRow`
  pins `alignItems: 'center'` so the horizontal scroll can't stretch chips to the row.

Fourth pass (same voice session) — promote moves to creation; uniform carousel tiles:

- **Promote-to-primary lives in the COMPOSER now, not the detail.** On the published detail,
  every viewer — owner included — gets each carousel item's normal open behavior (a post is a
  public arrangement once it exists); `PostItem`'s owner prompt + `promptMakePrimary` are gone.
  In the composer's preview carousel, tapping a tile swaps it with the lead instantly
  (`promoteDraft` — the API's swap semantics, no confirm; a draft swap is one tap to undo), and
  the X still removes. This settles open decision 2. The `useSetPrimaryContentItem` mutation and
  the api swap endpoint stay — unused by mobile for now.
- **Uniform carousel tiles** (`PostContentItems` `tile` prop + posts.ts `galleryItemTile`):
  every rail slot shares the reference hero's aspect (386/251.86), so listing/bid/photo/card/
  link/video all cut the same height. Photos fill the slot (down from 3:2); the entity card
  stretches to it with a bigger (30%) centered thumb — intrinsic height used to cut it off;
  links/embeds center in it. Both consumers — the detail rail and the composer preview — pass
  `tile`.

Fifth pass (same voice session) — cards page in place:

- **Feed/thread cards became a full-width pager** (`PostItem`): a multi-item card pages ALL
  its content items horizontally in place — primary leads — so the whole post reads from the
  feed without opening the detail. Every slide renders in a **uniform fixed slot** (the hero
  aspect, `cardPagerSlide` + the renderer's `slide` mode): photos fill it, links and the
  product row take a side inset, everything **top-anchored** (centering was tried and walked
  back). A **"1/5" counter** (chosen over dots; label size, `overlayDark` scrim so it reads
  over artwork in both themes) rides a pill pinned to the pager WRAPPER's top-right — outside
  every slide's own design, so it holds still while slides pass under it. **Gesture rule**:
  the wrapper claims its own touch starts (`onStartShouldSetResponder`), so the whole pager
  area is swipe territory — dead space around a slide neither opens the post detail nor falls
  through; only the slides' own touchables (tiles, links) navigate. That resolved the first
  on-device round, where swipes only worked on the card content and stray taps opened the
  detail. Single-item cards and the detail root are unchanged (the root keeps its tile rail).
- **Composer promote regained its confirm** ("Make primary" `alert.confirm`) — the prompt is
  what tells you the tap means promote, not open; it lives ONLY in the post modal. Published
  posts everywhere use each item's normal UX.
- **Tile alignment**: link/embed tiles hang from the same top line as the entity card
  (`galleryLeafTop`, xs headroom; `contentItemsTile` is top-anchored now). Entity tile:
  content near the top (xs headroom), two-line name + product number (no set line at rail
  size), fair-market pill dropping below the identity stack on a gutter's spacing.
- **Card chicklets tuned**: hairline `border` color instead of secondary.700 (subtler on the
  feed), and an xxs step off the content (12 above, 14 below).

**Not done / open:** "Add tag" placement stays the end-of-row circle; child tags and Set
remain out.

## What changes, in one pass

| surface | today | planned |
|---|---|---|
| Composer attachments | 1 max; circles hide when attached | up to **5**; circles **disable** at 5, X on an item re-enables |
| Composer tags | none (ambient only) | new **tagging screen**, up to **10** parent values, chips with X-to-remove |
| Reply composer | plain text, attach circles hidden | **one attachment allowed**, circles shown; tags stay hidden |
| Post cards (feed/threads) | the single attachment | the **primary** item only |
| Post detail | items in order | primary + **carousel** (slots 2–5), tap → "make primary?" swap |
| Group screens | same composer | identical caps; tags allowed (silent — see api plan) |

## Composer: five attachments, primary-first

- `PostComposer`'s `attachments: DraftItem[]` already holds an array — the v1 cap is UI-level.
  Raise to 5; the attach circles (photo library / camera / ELB / link) **disable at the cap
  rather than hide** (supersedes the hide-on-attach behavior). Removing an item via its X
  re-enables them. Same pattern as the tag cap below — one consistent "full" treatment.
- **The first attachment added is the primary** — no picker at compose time; promotion happens
  later on the post detail. The composer preview renders items in order (primary leads).
- `draft.ts` serializes all attachments with sequential `index` (text blocks interleaving as
  today); the API stamps `isPrimary` on the first attachment server-side.
- Replies: attach circles become visible in reply mode, capped at one (the old root behavior).
  Tags remain absent in reply mode.

## The tagging screen

- **Entry point (v1): an "Add tag" circle at the end of the attach-circle row.** Placement is
  explicitly provisional — Skylar floated a narrower "Add tags" button above the circles with a
  chip carousel (search-style); park that until the flow is felt out. The circle disables at 10
  tags, re-enables when one is removed.
- **The screen itself**: SearchFilter-style layered full screen (the established composer-layer
  pattern — `TagPickerLayer` is the starting point but the selection model changes):
  - Lists the brand's **supported tag values** the way filter/search presents them (the view is
    big — accepted for now, revisit presentation later).
  - **Multi-select up to 10** — selecting toggles; at 10, a notice ("You've selected the maximum
    of 10 tags") and further selection blocked. The server cap is 10 flat across parents and
    children; since the screen offers parents only, the UI cap and server cap coincide.
  - **Parents only for now**: child tags are a UI deferral (the server accepts them under the
    same flat cap); no Set. Set isn't a tag and gets its own thinking later.
- **Chips ("chicklets") in the pinned bottom cluster** (decided 2026-08-11, voice): selected
  tags render as the search screen's removable chips — X to remove, freeing a slot — docked in
  the SAME container as the attach circles, directly above them, so the picks stay in view while
  the draft scrolls beneath. (Supersedes the earlier "between the preview and the attach row"
  in-scroll placement.)
- On a supported-tag-value page, the page's own value stays ambient and pre-fills as a chip
  (the api now allows adding more alongside it).

## Post cards render the primary only

Feed and thread cards (`PostItem` and friends) pick the `isPrimary` content item (fallback:
first attachment, for anything predating the migration) and render just that — the card stays
the same size regardless of how many items the post carries. The other items are discoverable
only by opening the post.

## Post detail: carousel + promote-to-primary

- Below the root post's primary item, remaining attachments (slots 2–5) render as a horizontal
  **carousel** — reuse the renderer's existing group-carousel branch (built future-safe for
  GROUP items; this is its first real consumer, generalized to "sibling attachments").
- **Tapping a carousel item (owner only)** shows a bottom prompt/modal: *"Make this the
  primary?"* Confirm → `PUT /post/:id/primary-content-item` → the item and the current
  primary **swap positions** (promoted item leads, old primary takes its slot). Optimistic swap
  locally; feed cards pick up the new primary via the standard query invalidation.
- Non-owners: tap keeps whatever the item's normal behavior is (image viewer, reference
  navigation, link sheet) — the prompt is an owner affordance. For owners, that same normal
  behavior needs to stay reachable (e.g. the prompt offers "View" / "Make primary" / cancel) —
  exact treatment felt out on-device.

## Group screens

The composer mounted from group detail behaves identically — 5 attachments, 10 tags. The
tags are real associations used for reference *from* the post; the api guarantees the post
never surfaces on tag pages (see the api plan's union filter). No group-specific UI.

## Touched code (mapped)

- `components/posts/composer/PostComposer.tsx` — cap, circle disable states, chips row, reply
  attach mode, "Add tag" circle.
- `components/posts/composer/TagPickerLayer.tsx` — single-select → the multi-select tagging
  screen (or a new `TagSelectLayer` beside it if the brand composer still wants single-select).
- `components/posts/composer/draft.ts` — multi-attachment serialization (already index-based).
- `components/posts/PostContentItems` — primary-only mode for cards; carousel branch for detail.
- `PostItem`, `PostDetailScreen` — primary pick, carousel, promote prompt, swap mutation.
- Types: `PostContentItemDto.isPrimary` via `@oakplatforms/types` (local mirror until publish,
  the established pattern).

## Open decisions (Skylar)

Shared list lives in the [[../nanza-api/solution-designs/post-carousel-and-tagging|api plan]]
(isPrimary vs index-only, reply attachment optional-vs-required, child-tag linkage, Set, swap
audience). Mobile-only:

1. "Add tag" placement — end-of-row circle (planned) vs. narrow button + chip carousel above.
2. ~~Owner tap on a carousel item~~ — **decided 2026-08-11**: promote happens at creation
   (instant swap in the composer preview); the published detail keeps normal item behavior.
3. ~~Chips row placement in the composer stack~~ — **decided 2026-08-11**: docked in the pinned
   bottom cluster with the attach circles (see the implementation record).
