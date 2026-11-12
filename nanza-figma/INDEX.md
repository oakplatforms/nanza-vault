---
tags: [nanza-figma, index]
---

# nanza-figma

Figma → code handoffs. A designer edits a frame in Figma. The `figma-handoff` skill in
nanza-mobile reads that frame over the Figma MCP and writes **one implementation prompt per
frame** into `handoffs/`. A coding session in nanza-mobile then implements from that file.

These prompts live here, not in the app repo, so the repo stays code-only and every handoff
stays visible in Obsidian.

- **How a handoff is made:** `nanza-mobile/.claude/skills/figma-handoff/SKILL.md`. Run it with
  `/figma-handoff`, or paste "give me a prompt for: <link> / i changed: … / do not change: …".
- **How the app side uses one:** [[../nanza-mobile/solution-designs/figma-handoff|Figma handoff (mobile)]].
- **The code → Figma direction** (token sync, screen capture) is the `figma-sync` skill and the
  `nanza-figma-sync` repo.

## Lifecycle

`open` → `in-progress` (a session is implementing) → `built` (code written, uncommitted) →
`done` (QA'd and committed). Change the `status:` frontmatter and add a line to the file's Log
at each step. Answers to a handoff's open questions go **into the handoff file**, so the
implementing session sees them.

## Buckets: pre-dev vs post-dev

Every handoff goes in exactly one bucket. The bucket tells the dev team how to treat it.

| Bucket | Folder | What it may contain | How dev treats it |
|---|---|---|---|
| **Pre-dev** | `handoffs/pre-dev/` | Code changes deep enough to need implementation judgement: moving or restructuring components, new or changed tokens and shared types, native files, animation logic, component behaviour. Still no new product functionality. | Start of each sprint. Plan it, estimate it, and raise questions before building. |
| **Post-dev** (design QA) | `handoffs/post-dev/` | **Pure styling only:** colour, spacing, radius, type-scale slot, border values on existing classes. Nothing moves, nothing changes how it works. | Apply as-is during design QA. If a post-dev item turns out to need more than a value change, send it back to be re-filed as pre-dev. |

If a handoff mixes the two kinds of work, it goes in **pre-dev**. Set `bucket:` in the
frontmatter to match the folder.

## Handoffs

### Pre-dev

| Date | Handoff | Scope | Status |
|------|---------|-------|--------|
| 2026-11-12 | [[handoffs/pre-dev/2026-11-12-brand-rehue-logo-splash\|Brand re-hue — persian-rose palette, logos, grey links, plus-button gradient, splash fixes]] | tokens (primary, action, highlight, new link + type), logos, FloatingTabBar, JoinChatFooter plus discs, LogoBurst, Android splash / notification tint, iOS launch logo | open — ready for dev |
| 2026-11-12 | [[handoffs/pre-dev/2026-11-12-tag-page-fixed-hero\|Detail-page fixed heroes — pinned banner, full-height fadeHeroFixed gradient (tag, set, group, collection, profile)]] | new FixedBanner; SupportedTagValueScreen, SetScreen, GroupDetailScreen, CollectionDetailScreen, ProfileHero (ProfileScreen, UserProfileScreen); cards.ts, profile.ts, gradient token | open — ready for dev |
| 2026-11-12 | [[handoffs/pre-dev/2026-11-12-post-screen-thread-line\|Single post page — replies line up with the post, vertical thread line]] | PostItem (new `'flush'` inset), StoryPostPage (pager), PostDetailScreen; posts.ts | open — ready for dev |

### Post-dev (design QA)

| Date | Handoff | Scope | Status |
|------|---------|-------|--------|

### Not yet sorted

These predate the buckets. Each one moves into a bucket once it has been re-verified against dev.

| Date | Handoff | Screen | Status |
|------|---------|--------|--------|
| 2026-10-23 | [[handoffs/2026-10-23-makeoffer-listing-card\|Make Offer — listing card]] | MakeOffer | built |
| 2026-10-24 | [[handoffs/2026-10-24-entity-tabs-market\|Entity — tabs + Market charts]] | Entity | open |
| 2026-10-25 | [[handoffs/2026-10-25-entity-card-styles\|Entity cards — restyle]] | EntityLockupRow (search, trade, posts, offers) | open |
| 2026-10-25 | [[handoffs/2026-10-25-collection-card-styles\|Collection cards — restyle + market value]] | CollectionThumb | open |
| 2026-10-25 | [[handoffs/2026-10-25-bid-listing-tiles\|Bid & listing tiles — restyle]] | ItemThumb (grids, carousels) | open |
| 2026-10-25 | [[handoffs/2026-10-25-makeoffer-screen-restyle\|Make Offer — screen restyle]] | MakeOffer | open |
| 2026-11-02 | [[handoffs/2026-11-02-profile-restyle\|Profile — restyle (all tabs) + app-wide posts, trade tiles, tab counter]] | ProfileScreen / UserProfileScreen + PostItem, ItemThumb, SegmentedTabs | open |
| 2026-11-03 | [[handoffs/2026-11-03-messaging-restyle\|Messaging — inbox, thread, reply/emoji/block sheets + app-wide tab counter, sheets]] | MessagesScreen / ConversationThread + SegmentedTabs, BottomSheet, ConfirmSheetContent | open |
| 2026-11-03 | [[handoffs/2026-11-03-design-qa-posts-cards\|Design QA: play badge, inline entity cards, post footer share, tile rims, multi-entity spacing]] | CircleThumb, EntityLockupRow (posts + search/trade), PostItem, TagThumb / GroupBannerThumb | open |
| 2026-11-09 | [[handoffs/2026-11-09-post-entity-rows-identity-gap\|Posts — entity rows without the card box (ruled list), identity → text gap, post-screen reply alignment + thread line]] | PostContentItems / EntityLockupRow (plain chrome), PostItem (cards, replies, detail root), builds on 2026-11-03 design QA. Change 4 (post-screen alignment + thread line) moved to [[handoffs/pre-dev/2026-11-12-post-screen-thread-line\|pre-dev]] | open |
| 2026-11-09 | [[handoffs/2026-11-09-composer-emoji-caption-editor\|Composer — emoji clipping, caption editor restyle, Aa in the header]] | PostComposer (text fields), PhotoTextEditor (+ SizeSlider) | open |
| 2026-11-09 | [[handoffs/2026-11-09-binder-grid-wrap-spacing\|Binder grid — third card wrap fix + spacing aligned with the title]] | CollectionDetailScreen (collectionDetailGrid, cardWidth) — builds on 2026-11-09 fixed hero | open |
| 2026-11-09 | [[handoffs/2026-11-09-affiliate-circle-labels\|Home affiliate rail — circle caption type + spacing]] | CircleThumb (lg) / AffiliateCircleCarousel | open |