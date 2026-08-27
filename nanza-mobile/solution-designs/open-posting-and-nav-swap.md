---
tags: [nanza-mobile, solution-design, plan, posts, navigation]
---

# Open Home Posting & the Plus/Profile Nav Swap — Plan

> **Status: PLAN, 2026-08-14 (Skylar, voice session).** Mobile half of
> [[../nanza-api/solution-designs/open-posting-and-friend-visibility|the api plan]] (gate
> removal + friend-scoped feed reads). Revises [[posts-chat|Posts Chat]]'s "creation lives
> only in the header plus circle" and the affiliate-only composer entry.

## The ask

1. **The nav swap (mobile only, not web):** the top-header plus circle (today: 44pt glass
   circle, affiliates only, opens the BRAND composer) and the bottom nav's "You" avatar
   circle **swap roles**:
   - **Top header**: the small circle becomes the **profile avatar** — navigates to the
     Account/profile experience.
   - **Bottom nav**: the "You" slot becomes a **pink plus button** — the plus glyph in the
     brand pink, sized as the avatar circle is today — and opens the post composer.
2. **Everyone can post to the homepage feed** — the composer entry is no longer
   affiliate-gated (the api drops the create gate; visibility is scoped server-side:
   affiliate posts reach everyone, a basic user's posts reach only themselves + accepted
   connections/friends; you always see your own posts in feed order).

## Today's wiring (mapped 2026-08-14)

- **Header plus**: `TopHeader/index.tsx:45-55` — `isAffiliate && selectedBrandId` gates a
  `JoinChatFooter placement="headerPlus"` (`JoinChatFooter.tsx:99-103`): 44pt
  `headerPlusCircle` (headers.ts:138-146, glass fill + GlassRim), plus glyph `icons.xs`.
  Press → auth modal if signed out, else the composer Modal (`postableType="BRAND"`,
  `postableId=selectedBrandId`).
- **`isAffiliate`**: SessionContext.tsx:55 — `profile.type === 'AFFILIATE'`.
- **Bottom nav**: custom `FloatingTabBar` — `MAIN_TABS` (Home/Groups/Trade/Activity) in the
  ink glass pill; **Account renders separately as the "You" circle**
  (FloatingTabBar/index.tsx:261-307): 66pt `floatingNavYou` glass circle, 56pt
  `floatingNavAvatar` image (dimmed 0.5 when unfocused), profile-3 icon fallback,
  signed-out shows "Log In" and opens the auth modal.
- **Feed**: `usePosts('BRAND', selectedBrandId)` → `GET /posts?...&limit=10`, key
  `['Posts','BRAND',id]`; homepage windows one page into three 3-post "What's new" bands;
  See-all → `WhatsNewScreen`. Reply hosts (`placement="none"`) exist on both surfaces —
  creation deliberately lives only in the plus entry.
- **Pink**: `theme.colors.action` (= primary.400 `#FF6DFA`); the pink circle recipe is
  posts.ts `sendIconButtonActive` (40pt). No pink plus exists yet. NOTE
  postComposer.ts:180-190 documents the decision to keep the composer's commit CTA
  off-pink — that reasoning is about the composer's internal button language, not nav.

## Implementation record (2026-08-14, uncommitted)

Built same-session. Specifics beyond the plan:

- **JoinChatFooter grew a `navPlus` placement** — the pink circle IS a JoinChatFooter, so
  the auth prompt, the composer Modal host, and the BRAND target all reuse the one
  component. `usePressScale` (TabItem's export) gives it the pill tabs' press "give".
  Icon ink follows the send button's active pair (`theme.colors.text` on the pink fill),
  glyph at `icons.md` — both felt out on-device.
- **`layout.ts floatingNavPlus`**: `floatingNavYou`'s box with
  `theme.colors.action || primary.400` fill, no GlassRim (opaque commit fill takes no
  rim — the Post pill's rule).
- **FloatingTabBar**: the whole You branch is gone (avatar, Log In state, `onLoginPress`
  prop and BottomTabNavigator's auth-modal wiring with it — JoinChatFooter owns the
  signed-out prompt now). Mounts the navPlus only when `selectedBrandId` exists.
- **TopHeader**: `profileEntry` replaces `brandPostEntry` — the 44pt `headerPlusCircle`
  glass with the avatar inset (`headerPlusAvatar`, hit−sm = 36pt, the You circle's
  56-in-66 proportion carried down), profile-3 glyph fallback, → `Account/ProfileHome`.
  Signed-in only; the signed-out header keeps Sign Up as the single auth entry.
- **Account route stays registered**; its tabPress listener keeps the stack-reset (and a
  belt-and-braces signed-out preventDefault).
- Verified: tsc/eslint clean on touched files (repo baseline errors unchanged).

## Second pass — Skylar's on-device round + the audience dropdown (2026-08-14 PM)

- **Header avatar bigger**: inset 36 → 40 (`hit − xxs`, near full-bleed, a whisker of
  glass rim left).
- **Signed-out keeps the ORIGINAL Log In circle** (profile glyph + label, auth modal via
  `onLoginPress` — that wiring is restored); the pink plus shows **signed-in only**.
  Resolves open decision 3.
- **Pink plus dressed in the nav's glass container**: outer = `floatingNavYou` verbatim
  (glass + GlassRim), inner pink disc (`floatingNavPlus`) inset an `xs` ring — walked up
  from `xxs` ("a tad more padding"). **No press animation** — the press-scale squeeze was
  tried and walked back same session; the disc holds its size.
- **Audience dropdown** (single-select — the multi-select audience model was dropped for
  "public or a group", and group composers show NO dropdown): homepage composer only,
  in the header beside the close chevron — a glass pill (`audiencePill`, Post-pill glass
  + label + select chevron) opening an ink menu (`audienceMenu`, the text blocks'
  whisper outline; current pick reads in the action pink). Options: **Public** (default)
  + your groups in the brand (`['groups','mine',…,'forTargeting']` — the listing form's
  targeting-chips query, filtered by brand; no groups → no dropdown). The pick simply
  **switches the created post's home**: `useCreatePost` accepts a per-post `target`
  override and invalidates the actual target's `['Posts']` key. Server-side the group
  path is an ordinary GROUP post (member-gated by `validateGroupPostPermission`).
- Verified: tsc/eslint clean on touched files.

## Third pass — pinned chat entry (2026-08-14 PM, voice; implemented)

- **"Join the chat" pins to the screen bottom on the surfaces WITHOUT a buy bar** — tag
  values, groups, entities; listing/bid detail keeps the inline heading→input rhythm
  (their bottoms carry the Buy CTAs), and PostDetail keeps its inline Reply entry (its
  own documented rationale). Mechanics: `DetailChatSection` grew
  `composerPlacement: 'inline' | 'pinned'` — pinned renders no inline pill, keeps the
  reply-tap modal host (`placement="none"`), and pads the thread's tail
  (`chatSectionPinnedClearance`); the SCREEN wraps PageLayout in
  `chatPinnedScreen` (flex:1) and mounts `JoinChatFooter placement="fixed"` as a
  sibling (an absolute pill inside the scrolled section can't pin). Group's pinned pill
  is members-only like the section.
- **The "Chat" label earns its place**: always on the inline surfaces (it anchors the
  input), but on pinned surfaces only once the page has ≥1 post — otherwise the label
  "is just sitting there by itself" (Skylar). `DetailChatSection` reads `total` off the
  same `['Posts']` query PostThread uses.
- **Dropdown padding walked up** (menu `xxs→sm` vertical, rows `md→base` horizontal).
- Verified: tsc/eslint clean (the tag screen's `inset="tag"` error is pre-existing
  baseline, untouched).

## Fourth pass — locked tag chips on tag-page composers (2026-08-14 PM, voice)

- **`ComposerTagContext.ambientSupportedTagValue` → `lockedTags[]`** (threads through
  JoinChatFooter and DetailChatSection): a tag page's own association(s) pre-fill the
  composer as **locked chips** — shown, SENT, and unremovable (no X; remove taps and the
  picker's toggle both no-op on them; the picker's Clear frees the free slots only, and
  Apply re-asserts the locked set). A **parent page locks itself; a child page locks its
  parent(s) plus itself** (the child previously passed nothing). The flat 10 cap counts
  them — Warrior + 9 free.
- **The tag screen finally passes `brandId`** (`?include=brandTag` on its detail query)
  — without it the composer's whole tags line never rendered on tag pages, which is why
  "Add tags" was missing there.
- Verified: tsc/eslint clean on touched files.

## Fifth pass — PostDetail pinned reply + profile Posts tabs (2026-08-14 PM, voice + chat)

- **PostDetail's reply entry is pinned now too** (supersedes its documented inline
  rationale — Skylar's call): fixed "Reply to post" pill below PageLayout, the "Reply"
  heading REMOVED entirely (replies or not), tail clearance spacer added.
- **Profile tabs, both screens** (UserProfileScreen + own ProfileScreen):
  - **Reviews → "References"** (label only; ids/routes unchanged).
  - **Empty tabs hide** — Listings, Bids, References render only when their count is
    non-zero (Collections and own-profile Saved always show). Tab first-pages fetch
    EAGERLY now (UserProfile's visited-tab lazy gating removed; own profile reads the
    same query keys its tab components use, so React Query dedupes). A hidden active tab
    falls back to Collections.
  - **New "Posts" tab** right after Collections: the account's root posts across every
    home, hidden at zero — `useUserPosts` → **`GET /posts?accountId=` (new api author
    mode)**: owner sees all their roots; other viewers get public-chat homes plus BRAND
    posts only when the author is an affiliate or an accepted connection; GROUP posts
    stay off other viewers' lists. Cards render as standard thread cards (minimal
    footer), own-profile version in `tabs/PostsTab.tsx`.
- Verified: tsc/eslint clean on touched files in both repos.

## Sixth pass — the signed-out nav (2026-08-22, Skylar)

Guests were being shown the full signed-in chrome: four tabs they can't meaningfully
land on, and a white **Sign Up** pill in the header. The two auth paths swap weight —
the loud one moves DOWN to the nav, the quiet one stays up top.

- **Top header, signed out**: the pill reads **"Log In"** in the standard glass
  (`topHeaderBalancePill` + `topHeaderBalanceLabel`) — the inverted `topHeaderSignUpPill`
  treatment is dropped, and the modal opens at `initialStep="Login"` rather than `Email`.
  It is still the only header entry (no avatar circle for guests).
- **Bottom nav, signed out: there is no nav.** `FloatingTabBar` returns early on
  `!isRegistered` with **`JoinChatFooter placement="fixed"`** — the same pinned chat
  entry the tag-value, entity, group and post-detail screens already mount: a thin glass
  input reading "Join the chat" over the `fadeActionBar` gradient scrim. No pill, no
  tabs, no circle, no jelly, no `RadialGlow`, no ink backdrop. Targets the selected
  brand (`BRAND`/`selectedBrandId`), and `JoinChatFooter` owns the auth prompt, so a
  guest tap opens the auth modal instead of the composer.
  - **Why the whole cluster is skipped**: the footer brings its own `EdgeFade`
    backdrop, so layering the nav's ink scrim under it would double-darken the band —
    the same reasoning as the cluster's one shared glow. It also supplies `insets.bottom`
    itself, so the tab-bar container needing no height is fine.
  - This replaces a bespoke `floatingNavJoinPill` built earlier the same day (a
    full-width CTA in the cluster, tried white-then-glass, label felt out across four
    typography slots). Reusing the app's existing pinned-composer treatment beat
    inventing a nav-only one; the bespoke style and its label are deleted.
- **Search sits wherever the bottom bar leaves room, and auth decides which**:
  `TopHeader.hideSearch` defaults to `hideSearch ?? isRegistered`. Signed IN the tab
  bar's circle owns it (the 2026-08-15 move down stands) and the header shows none;
  signed OUT the bottom band is the chat footer with no circle slot, so search returns
  to the header — inner side, Log In in the corner. Never both at once, and never
  neither. The explicit prop still overrides per screen. Browsing stays open to guests
  either way, so search is only ever relocated, never gated.
- **The guest header pill keeps the plain glass balance-pill treatment.** A solid
  non-glass fill was tried on 2026-08-22 and reverted immediately: it's `Log In`, the
  quieter of the two auth paths, so it should not out-shout the chat CTA. The inverted
  `topHeaderSignUpPill` recipe survives in headers.ts but is now unconsumed.
- **The home groups shelf goes inert for guests** (`GroupCircleCarousel`), reversing the
  2026-08-17 "guests skip the dim" call. Every thumb is dimmed and carries **no
  `onPress` at all** — `CircleThumb` sets `disabled={!onPress}`, so omitting it kills the
  press feedback too (a no-op handler would still look pressable). Previously every guest
  tap opened the same auth modal, which made the shelf twenty identical sign-up buttons;
  dimmed-and-dead says the same thing structurally — *these exist, they're not yours yet*
  — and the one join prompt now lives in the bottom bar. The carousel's `AuthModal` and
  `useAuthModal` wiring are deleted with it.
- **`circleThumbPending` lifted 0.2 → 0.45** (cards.ts). At 0.2 one pending thumb among
  bright ones read fine, but a guest's whole shelf at that opacity nearly disappeared.
  The guest state has to keep thumbnails legible to advertise the groups at all, while
  still reading disabled — so it sits well under the 0.61 the muted controls use. Shared
  with `TradeUserListRow`'s pending state, which gets the same lift.
- **The three guest promo screens are deleted.** `GroupsPlaceholderScreen`,
  `TradePlaceholderScreen` and `MessagesPlaceholderScreen` were full-screen "Sign up"
  promos rendered by an early return on the Groups / Trade / Inbox tab roots. With no
  tab bar signed out and no deep link to any of the three (checked `AppNavigator`'s
  `linkingConfig` — none registered), a guest cannot reach them, so all three were dead.
  - The **guards stay**, now returning `null`. Trade and Messages key off
    `viewerAccountId`, which is also briefly falsy pre-hydration, and everything below
    narrows on it (`TradeScreen` passes `viewerAccountId as string`; every Messages tab
    takes it as required) — removing the guard would have been a crash, not a cleanup.
  - Orphaned with them: `PromoVariant` / `PROMO_CONFIG` / `getPromoAndroidStyles` and
    eight `*Promo*` style entries in layout.ts, `promoTitle` / `promoSubtitle` in
    typography.ts, and ~7.1 MB of assets (`groups.png`, `messages.png`, `buy.png`,
    `placeholderGradient.jpg`). `logo_full.svg` is KEPT — `AuthModal` still uses it.
  - Note a neighbouring `savedPlaceholder*` / `savedPromo*` style family in layout.ts is
    also unreferenced, but was already dead before this change and is left alone.
- **Search's empty state is auth-aware too** (`RecentSearches`): recent searches are
  stored per account and the query is `enabled: Boolean(accountId)`, so a guest never
  fetches and always lands on the empty branch. "No recent searches yet" described a
  history that can't exist for them — it read as a broken feature rather than an absent
  one. Signed out now reads "Start typing to search cards, people and groups" (the
  screen's own Nanza/People/Groups tabs); the signed-in copy is untouched.
- The jelly/tab machinery still runs its hooks when signed out (the early return sits
  below them, as it must) but renders nothing.
- The post FAB was already `isRegistered`-gated, so guests see no composer entry.
- Verified: tsc/eslint clean on touched files. NOT verified on-device (no simulator run
  this session) — the pill's width math and label seating want an on-device look.

## Planned changes

### Bottom nav: the pink plus

- `FloatingTabBar`'s Account/"You" branch becomes the **post entry**: keep the
  `floatingNavYou` 66pt geometry and press-scale, swap the glass fill for
  **`theme.colors.action` pink** (fallback chain `action || primary.400`), drop the
  GlassRim (opaque fill takes no rim — the Post pill's own rule), center a **plus glyph**
  (white/`overlayText`-family ink, sized to read at the avatar's presence — felt out
  on-device).
- Press: signed-out → auth modal (existing `onLoginPress` path); signed-in → the SAME
  composer Modal the header plus hosts today (`postableType="BRAND"`,
  `postableId=selectedBrandId`). The tab bar needs a composer host — either mount a
  `JoinChatFooter placement="none"`-style Modal host in the tab bar, or lift the open
  state to the navigator. **No `isAffiliate` gate.**
- The plus rides the tab bar, so it appears on every tab and always posts to the selected
  brand's home feed (the composer contract is unchanged).
- The **Account tab route stays registered** (the header avatar navigates to it) — only
  its tab-bar representation changes.

### Top header: the avatar

- `TopHeader`'s plus slot becomes the **profile circle**: `headerPlusCircle` geometry
  (44pt glass) holding the user's avatar image (the FloatingTabBar avatar recipe scaled
  in), profile-3 icon fallback; signed-out → auth modal.
- Press → `navigation.navigate('Account', { screen: 'ProfileHome' })` (the tab's own
  target).
- No affiliate gate; shows whenever a user is signed in (and the fallback icon + auth
  modal when not).

### Feed / composer

- No query changes — the api filters server-side; `['Posts','BRAND',id]` responses simply
  arrive pre-scoped (affiliates + friends + self).
- Composer entry ungated; everything else about the composer is untouched.

## Open decisions (Skylar)

1. **Profile reachability off-home**: `TopHeader` renders on the home surface — with the
   You circle gone, is the header avatar the only profile entry everywhere, or does
   TopHeader (with avatar) appear on the other tabs too?
2. Plus glyph size/ink on the pink circle — felt out on-device.
3. Does the bottom plus show for signed-out users (as a login prompt like today's "Log
   In") or hide?
