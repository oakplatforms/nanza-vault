---
tags: [nanza-mobile, solution-design, profile, plan]
---

# Profile Sections — Plan

> **Status: plan, IMPLEMENTED (uncommitted) 2026-08-17.** Replace the profile tab pills
> with stacked, limit-trimmed sections + See-all screens, remove Reviews/References from
> the profile UX, and stage the load like Home. Fold into [[profile|Profile]] when it
> lands.
>
> **Later delta (2026-08-26):** Listings + Bids collapsed into ONE **Trades** rail
> (listings, lots and bids interleaved) over a new `GET /trade-feed`; See-all section
> `'trades'` replaces `'listings'` / `'bids'`. Written up in [[profile|Profile]] → "The
> Trades rail".
>
> **Deltas from the plan, decided live during the build (Skylar):**
> - **Friends row on the OWN profile** — the GroupDetail member-row recipe under the
>   hero: facepile of up to 4 AVATARED accepted connections + the full "N friends" count
>   ("No connections" at zero) on the left, **Add collection** as the white emphasis
>   pill on the right. The row runs the hero's 15pt inset and hugs the description
>   (paddingTop xs) more than Posts. Tap → the new **`FriendsScreen`** (route `Friends`,
>   root-stack vertical entrance): GroupPeopleScreen's `useTradeUserList` rows over
>   ACCEPTED connections with a Message action into `ConversationThread`.
> - **The white emphasis pill** — `ActionPill emphasis` (See-all geometry, `text` fill,
>   ink label; `carouselActionPillEmphasis` in cards.ts): the profile's Add collection
>   AND the group row's "Add new" (Join/Request-sent keep the group-button dress).
> - **Section headings at the hero inset** — profile section labels + See-all pills sit
>   at 15pt to align with username/description (`profileSectionHeadingRow`; CardGrid
>   grew a `headingStyle` prop and `profileSectionHeadingInset` tops up its 8pt section
>   padding); the cards stay on the 8pt grid gutter.
> - **Groups hero parity** — GroupDetail's social row moved ABOVE the description;
>   facepile avatars grew to 40pt on both hosts (`groupDetailAvatar`).
> - **Social logos dimmed to 70%** in the display rows (`socialLinkLogoDim`); the
>   editor keeps full strength.
> - **No divider anywhere** — the hero hairline was briefly revived (links-gated), then
>   removed from BOTH profiles AND GroupDetail; `profileHeroDivider` and
>   `groupDetailDividerNoLinks` styles are deleted.
> - **Hero: social row ABOVE the description** (net unchanged from the morning), with
>   new air below it (`profileHeroSocialRow`, gated on links so empty profiles spend no
>   margin).
> - **Section spacing** — `profileSectionsStack` gap adds ~12 on top of each section's
>   own 20pt break ("a tad more"), without touching the app-wide shelf rhythm.
> - **No Add-New pills on section headings** — explored (white pill beside See all,
>   posts + collections), then dropped; the shelf ShelfHeading/ActionPill/CardGrid API
>   is unchanged. Posts has no create entry (no standalone composer exists).
> - **Old-tab-era orphans deleted**: `ProfileScreen/tabs/*`, `ProfileScreen/Profile/`,
>   `UserProfileCollection`, `UserCollection`, `ReviewRow`, `fetchUserReviews`, and the
>   review style families in `profile.ts`/`layout.ts`.

## 1. Problem analysis — **Confidence: 0.95**

Both profile surfaces eagerly fire ~5 paginated queries at `limit=10` on mount
(`ProfileScreen/index.tsx:51-54`, `UserProfileScreen/index.tsx:83-162`) to drive a tab UI
(TabPillsRow / StickyTabs) where only one tab's data is visible at a time. Reviews
("References") is fetched even though the surface is being retired. Saved items fetch
`limit: 100` unpaginated (`useSavedFeedItems.ts`). The ask: one glanceable page — every
section visible at a small fixed size, heavy browsing moved behind See-all screens, and the
first paint gated only on the two cheapest sections (posts + collections), the same
payload-budget rule as [[app-load-performance|App-load performance]].

## 2. Proposed solution — **Confidence: 0.88**

**Layout (both screens).** Hero → stacked sections, each headed by the shared
`ShelfHeading` (title left, `ActionPill` "See all" right — the exact Home lockup):

1. **Posts** — 3 `PostItem` cards (limit 3)
2. **Collections** — 2×2 `CollectionTargetCard` grid (limit 4)
3. **Listings** — 2×2 sell-feed thumb grid (limit 4)
4. **Bids** — 2×2 `BidThumb` grid (limit 4)
5. **Saved** — 2×2 grid, **own profile only**, always last (limit 4)

Reviews/References: gone from both screens. Tabs, tab state, tab refs, and the
`onEndReached` tab delegation: gone — the profile page itself no longer infinite-scrolls.

**Visibility rules.**
- `UserProfileScreen`: empty sections **hide** (reuse the existing totals + the
  `hasCollectionContent` recipe verbatim).
- `ProfileScreen` (own): sections **never hide for emptiness** — empty states render
  instead. Access gating stays (Listings needs `hasSellerAccess`, Bids `hasCustomerAccess`)
  and affiliates keep their roster rules (no Saved; Listings/Bids by access).
- **See all** renders only when `total > visible count` (4, or 3 for posts) — a section
  whose content fits shows no pill.

**Data.** New *preview* queries per section — plain `useQuery`, `limit=4` (posts 3), keys
namespaced `['<Resource>', 'preview', accountId]` — replacing the screens' infinite hooks.
The existing infinite hooks (`fetchUserSellFeed` et al.) move to the See-all screens with
`PAGE_SIZE` 10→20.

**Staged load (the Home recipe).** Posts + collections previews fire on mount; the
listings/bids/saved previews pass `enabled: primaryReady && !!accountId`, where
`primaryReady` latches once posts & collections have settled (`isFetched` both) — the
`useHomeReady`/`deferredActive` pattern (`HomeScreen/index.tsx:100-107`), minus
`InteractionManager` unless jank shows.

**See-all screens.** One new **`ProfileSeeAllScreen`**, route
`ProfileSeeAll: { title: string; accountId: string; section: 'posts' | 'collections' | 'listings' | 'bids' | 'saved' }`,
registered once on the root stack beside `FeedGrid` (vertical entrance). It owns a
`useInfiniteQuery` at **`limit=20`** per section and renders: posts → `PostItem` list
(WhatsNewScreen shape), collections → `CollectionTargetCard` grid (tap → existing
detail screens: own-editable vs `UserCollectionDetail` by ownership), listings/bids/saved →
`CardGrid` of the same thumbs (FeedGridScreen shape, `PAGE_SIZE = 20`,
`getNextPageParam` from `page/total`, `onEndReached` threshold 0.5).

## 3. Alternatives considered

- **Option 1 (Recommended): one `ProfileSeeAllScreen`, one registration** —
  **Confidence: 0.85** — Single nav addition; all five sections share the paginated-screen
  chrome; profile domain stays self-contained; `FeedGrid` untouched.
- Option 2: extend `FeedGridScreen` with profile source kinds + a separate new posts screen
  + a separate collections screen — **Confidence: 0.6** — Rejected: posts/collections don't
  fit FeedGrid's card contract, so it still needs two *more* registrations (three nav
  touches) and scatters profile logic across three screens.
- Option 3: keep tabs, only shrink limits — **Confidence: 0.1** — Rejected: Skylar
  explicitly said the tabs go away completely.

## 4. Implementation steps

1. Preview hooks (`limit 4`/`3`, `preview` keys) in `screens/profile/.../data`; bump
   existing infinite hooks to `PAGE_SIZE 20`; add `limit` param to `fetchLists`;
   paginated saved-items fetch (verify `savedItemService.list` accepts `page`/`limit` —
   if the API can't page, See-all falls back to the current `limit=100` single fetch).
   — **Confidence: 0.85** — Risks: saved-items API pagination support unverified.
2. Shared `ProfileSections` component (`components/profile/ProfileSections/`) rendering
   the five sections from props (`isOwnProfile`, per-section data/totals/handlers), using
   `ShelfHeading` + `CardGrid`; section spacing via existing `theme.spacing` tokens in the
   `profile` style bucket. — **Confidence: 0.8** — Risks: own-vs-user divergence
   (affiliate rules, empty-state vs hide) must live in the screens, not the component.
3. `ProfileSeeAllScreen` + route type + **one `AppNavigator` registration** (escalated
   below). — **Confidence: 0.8** — Risks: nav file is P0-protected; needs explicit
   approval.
4. Rewire `ProfileScreen`: drop TabPillsRow/tab components/refs/`onEndReached`, render
   `ProfileSections`, delete the `tabs/` folder. — **Confidence: 0.85** — Risks: losing
   the Saved surface's quantity affordances — port `SavedTab`'s card behavior into the
   saved section/See-all.
5. Rewire `UserProfileScreen`: drop StickyTabs + inline tab renderers, reuse empty-hide
   totals for section visibility; `defaultTab` route param becomes inert (accepted,
   ignored) so existing `navigate('UserProfile', { defaultTab })` callers stay valid.
   — **Confidence: 0.8** — Risks: callers expecting to land on a specific tab now land on
   the page top.
6. Remove reviews from profile UX: `fetchUserReviews` hook usage, `ReviewsTab`,
   `renderReviewsTab`, `ReviewRow`, `reviewList*`/`reviewCard*` styles. **Keep**
   `Review.ts` service (order flow) and `WriteReviewModal`. — **Confidence: 0.9** —
   Risks: minimal; order-flow review writing untouched.
7. Staged-load latch + lint pass + on-device sanity of both profiles, all five See-alls,
   affiliate + guest views. — **Confidence: 0.85**

## 5. Files modified (≈18)

- **NEW** `src/components/profile/ProfileSections/index.tsx` (~250)
- **NEW** `src/screens/profile/ProfileSeeAllScreen/index.tsx` (~220) + `data/` hooks (~80)
- **NEW/edit** preview hooks in `src/screens/profile/UserProfileScreen/data/` (4 files, ~30 each)
- Edit `ProfileScreen/index.tsx` (~-80/+60), `UserProfileScreen/index.tsx` (~-250/+80)
- Edit `types/navigation.ts` (+6), `AppNavigator.tsx` (+8, **escalated**)
- Edit `fetchLists.ts` (+limit param), `useSavedFeedItems.ts` (+preview/paged variants)
- Edit `src/styles/components/profile.ts` (+section classes, −review classes),
  `layout.ts` (−`reviewList*`)
- **DELETE** `ProfileScreen/tabs/` (6 files), `components/profile/ReviewRow/`

## 6. DRY check — **Confidence: 0.9**

- **Searched for**: see-all header lockups, paginated grid screens, posts list screens,
  section carousels, staged-load gates.
- **Found existing**: `ShelfHeading` + `ActionPill` (the Home see-all lockup),
  `CardGrid`, `FeedGridScreen`/`WhatsNewScreen` (paginated-screen shapes to copy, not
  extend), `PostItem`, `CollectionTargetCard`, `UserProfileCollection`, the
  `useHomeReady`/`deferredActive` gating recipe, `EmptyState`, `LoadingIndicator`.
- **Decision**: reuse all of the above; create only `ProfileSections` +
  `ProfileSeeAllScreen` because no existing screen renders mixed profile content types.

## 7. Escalation check — **YES** (two triggers: nav file edit; >5 files)

**Proposed change**: one added registration in `AppNavigator.tsx` (`ProfileSeeAll`,
vertical entrance beside `FeedGrid`) + one route-param type addition; ~18 files total.
**What could break**: nav registration order/deep links (addition only, no reorder);
profile browsing regressions. **Why best**: the feature requires a destination screen;
one registration is the minimum footprint (Option 1 vs Option 2's three touches,
Confidence 0.85 vs 0.6). **Mitigation**: additive-only nav diff, no param changes to
existing routes, on-device pass over deep links `profile` / `user/:profileId`.

## 8. Overall confidence — **0.85**

Key uncertainties: saved-items API pagination support; whether any `defaultTab` caller
truly needs section-scroll (deferred); exact skeleton/empty-state visuals during the
staged load (will match Home's shelf placeholders).
