# Global Search Rework + Backend Recent Searches

Rework the nanza-mobile global **Search** screen to mirror the CollectionDetail search UX
(bottom glass pill + **one filter circle** to its right), move the current chicklet/filter
UI onto a dedicated **Filter** screen, and make the Search landing state show **recent searches**
(up to 10 per user, stored on the backend, clearable).

> **Clarified:** there is only **ONE** trailing button — the **filter icon inside the standard
> grey circle** (the same `ink`/grey circle used in the top nav and for back arrows), sized to
> match those circles, placed to the right of the search pill. There is no separate plain circle.
> The earlier "circle icon + filter icon" wording was a misread of the user's request.

Scope: **nanza-mobile** (UI) + **nanza-api** (recent-searches persistence) + `@oakplatforms/types`
(RecentSearch DTO). No admin changes.

---

## 1. Problem Analysis

**Today (global Search):**
- Input is a **top header search bar** (`SearchBar variant="large"` in `PageLayout` header
  centerContent) — `src/screens/search/SearchScreen/index.tsx` (input built lines ~449–468).
- The "before you type" state renders the inline `<Filter variant="open">` — the sets +
  entity-tag **chicklets** (Filter.tsx `renderSetSection` ~192–331, `OpenTagSection` per brand tag).
  Data from `useFetchBrandTags`, `useFetchBrandSets`, `useFetchSupportedTagValues`.
- No recent/saved/history search exists **anywhere** (verified across `src` — zero matches).

**Target (mirror CollectionDetail):**
- **Bottom glass pill** input (like `CollectionSearchFooter` + `collectionSearch.ts` glass styles),
  pinned above the keyboard via KeyboardAvoidingView.
- To the **right of the pill**: a **circle icon** (the existing `ink`-filled ~50px circle used
  elsewhere) and a **filter icon** circle.
- Tapping **filter** opens a **Filter screen** that holds the current chicklet UI (the whole
  `<Filter variant="open">` sets + entity-tag selection). Selecting there applies filters back
  to Search (same `filterSelecteds` mechanism).
- Search **landing state** (no query): **recent searches** list — the raw strings the user
  recently searched, newest first, with a **Clear** action; empty state "no recent searches yet".
- Tapping a recent search **runs it immediately** (confirmed by user).
- Persist recent searches on the **backend, per account, capped at 10** (confirmed by user).

**Confidence: 0.90** — Evidence: `SearchScreen/index.tsx` (top-bar input + inline Filter landing),
`components/collections/CollectionSearchFooter/index.tsx` + `styles/components/collectionSearch.ts`
(glass pill to mirror), `components/collections/CollectionSearchOverlay/index.tsx` (overlay scaffold),
`hooks/useStandardHeaderIcons.ts` + `Header` `variant:'darkCircle'` (the reusable circle icon),
nanza-api `prisma/schema.prisma` `Account`/`SavedItem` (child-table pattern), `src/routers/user.ts`
(home for the new routes), `src/validation/user.ts` `validateAccount`.

---

## 2. Proposed Solution

**Frontend** — restructure `SearchScreen` into three visual zones matching CollectionDetail:

1. **Body**
   - No query → **RecentSearches** panel (new component): list of tappable rows (each runs the
     search) + a "Clear" affordance; empty state when none. Filter chicklets are **no longer**
     on this screen — they move to the Filter screen.
   - Query ≥ 2 chars (or active filters) → results, unchanged (`useFetchSearch`, `EntityCardList` /
     `TradeUserListRow`, Tabs, reference-code `ShareDetailView`). Applied-filter chips row stays.
2. **Bottom bar** — a shared glass pill footer + **one** trailing filter circle:
   - `[  🔍  glass search input pill        ]  ( ⚙filter )`
   - The filter circle: the **filter icon inside the standard grey circle** (same style as
     `Header` `darkCircle` / overlay `backButton` — `ink`/grey fill, radius max, ~50px), sized to
     match the top-nav / back-arrow circles. Tapping it → navigates to the Filter screen. Use
     `filter-2` as the icon placeholder until the user sends the specific asset.
3. **Filter screen** (new route `SearchFilter`) — renders the existing `<Filter variant="open">`
   (sets + entity-tag chicklets). Confirm/back applies selected filters to Search via the existing
   `route.params.filters` → `filterSelecteds` merge effect (reuse `navigateToSearch`/`setParams`
   plumbing so no new state channel is invented).

**Backend** — new `RecentSearch` child model on `Account`, three routes on `userRouter`:
- `GET /user/recent-searches?accountId=…` → newest 10.
- `POST /user/recent-search` `{ accountId, query }` → upsert-and-bump (unique `[accountId, query]`),
  then trim to 10 newest.
- `DELETE /user/recent-searches/:accountId` → `deleteMany`.
All gated by `validateAccount(reqUser, accountId, 'authenticated')`.

**Recording a search**: when a search actually runs (query ≥ 2 chars, debounced), fire the POST
so the term lands in history. Fire-and-forget; invalidate the recent-searches query key on success.

**Confidence: 0.82** — mirrors a shipped, proven pattern (CollectionDetail overlay + savedItem
router). Main uncertainty is the exact footer layout (pill + 2 circles) and the circle icon's
action, both design-ish and cheap to iterate.

**Why this over alternatives:** reuses the exact glass-pill + circle-icon + filter-merge
primitives already in the app; the backend follows the `SavedItem` child-table convention 1:1.

---

## 3. Alternatives Considered

**A. Filter as a separate stack screen (Recommended)** — **Confidence: 0.82**
New `SearchFilter` route rendering the moved `<Filter variant="open">`; applies back via params.
Best: matches the user's ask ("opens a filter screen", "move the current screen to the filter page"),
keeps Search body clean for recent searches, and the params→`filterSelecteds` merge already exists.

**B. Filter as an in-place overlay (like CollectionSearchOverlay)** — **Confidence: 0.55** — Rejected:
user explicitly said "opens a filter **screen**" and "move … to the filter **page**"; an overlay is
more state to babysit (searchMode-style toggle) and muddies the Search body. Keep the overlay pattern
only for the *look* of the pill, not the filter navigation.

**C. Recent searches in local storage (AsyncStorage), no backend** — **Confidence: 0.40** — Rejected:
user explicitly chose backend modeling so history syncs across devices. Local-only also can't be
cleared server-side or reused elsewhere.

**D. Store recent searches as a JSON `String[]` column on `Account`** — **Confidence: 0.50** — Rejected:
repo convention is a child table for every user-owned collection (`SavedItem`, `ScanItem`, `List`…);
JSON columns are reserved for unstructured config. A child table gives clean `orderBy createdAt`,
`@@unique([accountId, query])` upsert-and-bump, and `deleteMany` clear.

---

## 4. Implementation Steps

**Backend (nanza-api) — do first so the mobile hooks have a real endpoint**
1. Add `RecentSearch` model to `prisma/schema.prisma` (`id`, `createdAt`, `updatedAt`, `query`,
   `accountId` FK `onDelete: Cascade`, `@@unique([accountId, query])`, `@@index([accountId])`) and
   `recentSearches RecentSearch[]` on `Account`. Generate a migration. — **Confidence: 0.88** —
   Risks: migration on prod; `@@unique` casing (searches are case-sensitive strings — decide
   whether to normalize/trim before store).
2. Add three routes to `src/routers/user.ts` (GET list / POST upsert+trim / DELETE clear), each
   `validateAccount(…, 'authenticated')`, using `generatePrismaError` for errors. No `all_routes.ts`
   change (userRouter already mounted). — **Confidence: 0.86** — Risks: the trim-to-10 must run in a
   transaction with the upsert to avoid races.
3. Add `RecentSearch` to `@oakplatforms/types` (or reuse the API's response shape) so mobile is typed.
   — **Confidence: 0.75** — Risks: types-package publish/version bump cadence; may inline a local DTO
   short-term if a publish is heavy (mirror how CollectionDetail hooks type their data).

**Frontend (nanza-mobile)**
4. Add API service module `src/services/api/recentSearch.ts` (list/create/clear via `fetchData`) and
   data hooks under `src/screens/search/data/` (`fetchRecentSearches.ts` `useFetchRecentSearches`,
   `useAddRecentSearch`, `useClearRecentSearches`). — **Confidence: 0.85** — Risks: query-key
   invalidation timing.
5. Build the **bottom bar**: a shared glass pill (reuse/generalize `CollectionSearchFooter` styling
   from `collectionSearch.ts`, or extract a small shared `GlassSearchPill`) + **one** filter circle
   to its right (grey/`ink` standard circle, `filter-2` icon). — **Confidence: 0.75** — Risks:
   `CollectionSearchFooter` currently lives under `components/collections/` and is collection-flavored;
   decide reuse-in-place vs extract-shared (DRY check below). Footer must sit above keyboard
   (KeyboardAvoidingView like the overlay).
6. Move the input **from the header to the bottom bar**; remove the header `SearchBar`
   (`variant="large"`) usage, keep the results-controls `SearchBar mode="controlsOnly"` (applied
   chips) as-is. — **Confidence: 0.70** — Risks: `SearchBarHandle.focus()` ref plumbing + autoFocus
   behavior; header layout ripple in `PageLayout`.
7. Replace the landing `<Filter variant="open">` block in `SearchScreen` with the **RecentSearches**
   panel (new `src/screens/search/components/RecentSearches/`). Tap row → set query + run. Clear →
   mutation + optimistic empty. — **Confidence: 0.80** — Risks: empty-state copy; loading state.
8. Add **`SearchFilter` screen + route** (register in `AppNavigator`, add param type in
   `navigation.ts`) rendering the moved `<Filter variant="open">`; filter circle navigates to it;
   applying filters routes back into Search's existing `route.params.filters` merge. — **Confidence:
   0.72** — Risks: navigation.ts/AppNavigator are treated as **read-only** per INDEX — this is a
   deliberate, minimal exception (one new route) and must be called out; passing selected filters
   back cleanly (reuse `navigateToSearch`'s setParams-by-key trick).
9. **Record on search**: when a debounced query ≥ 2 chars fires `useFetchSearch`, call
   `useAddRecentSearch`. — **Confidence: 0.78** — Risks: don't spam the endpoint on every keystroke
   (debounce/only-on-committed-search); skip for guests (no account).
**Verify** (no automated tests — [[no-automated-tests]]): run the app, exercise search → results,
open Filter → apply → back, land on recent searches, tap one (runs), clear, empty state; guest path.

---

## 5. Files Modified

**nanza-api**
- `prisma/schema.prisma` — **edit** (+~12 lines: new model + relation)
- `prisma/migrations/<new>/migration.sql` — **NEW** (generated)
- `src/routers/user.ts` — **edit** (+~70 lines: 3 handlers)

**@oakplatforms/types** (or inline DTO in mobile if publish is heavy)
- schema/type for `RecentSearch` — **edit/NEW** (~5 lines)

**nanza-mobile**
- `src/services/api/recentSearch.ts` — **NEW** (~40)
- `src/screens/search/data/fetchRecentSearches.ts` — **NEW** (~60: 3 hooks)
- `src/screens/search/components/RecentSearches/index.tsx` — **NEW** (~90)
- `src/components/search/GlassSearchPill/index.tsx` (or reuse CollectionSearchFooter) — **NEW/edit** (~60)
- `src/screens/search/SearchScreen/index.tsx` — **edit** (move input to footer, swap landing panel,
  add filter/circle buttons; ~-40/+80)
- `src/screens/search/SearchFilterScreen/index.tsx` — **NEW** (~70; renders moved `<Filter>`)
- `src/screens/search/components/Filter.tsx` — **edit** (extract/reuse `variant="open"` for the new screen)
- `src/navigation/AppNavigator` — **edit** (register `SearchFilter`; read-only exception)
- `src/types/navigation.ts` — **edit** (`SearchFilter` param type)
- `src/styles/components/*` — **edit** (footer-bar + circle-row styles; likely a new `search.ts` bucket
  or extend `collectionSearch.ts`)

---

## 6. DRY Check Results

- **Searched for**: existing bottom glass pill, reusable circle-icon button, the chicklet filter
  panel, recent/saved-search persistence, capped-list helper.
- **Found existing**:
  - Glass pill → `CollectionSearchFooter` + `getCollectionSearchStyles` (`footer/footerBar/footerFloat/
    footerInput`, glassFill/glassBorder). **Reuse the visual language**; decide extract-shared vs
    reuse-in-place.
  - Circle icon → no standalone component; the pattern is `Header` `HeaderRightIcon variant:'darkCircle'`
    / overlay `backButton` (`ink`, radius max, ~50px) via `useStandardHeaderIcons`. **Reuse the style**,
    likely a tiny local `CircleIconButton` to avoid duplicating the JSX twice.
  - Filter chicklets → inline `<Filter variant="open">` + `OpenTagSection` + `useFetchBrandTags/Sets/
    SupportedTagValues`. **Reuse wholesale** on the new Filter screen (extract `variant="open"` render
    if it's currently entangled with SearchScreen).
  - Filter apply-back → `navigateToSearch` / `route.params.filters` merge effect. **Reuse.**
  - Recent searches → **nothing exists**; new. Backend follows `SavedItem` child-table pattern 1:1.
- **Decision**: Reuse glass-pill styles, circle-icon style, the whole `<Filter variant="open">` panel,
  and the filters-merge navigation. Create new only: `RecentSearch` model + 3 routes + mobile recent-
  searches service/hooks/panel + the `SearchFilter` screen. Extract a shared `GlassSearchPill` +
  `CircleIconButton` **only if** reuse-in-place forces duplication. — **Confidence: 0.80**

---

## 7. Escalation Check

- **Escalation needed?** Partial — flag two items, don't block:
  1. **Prisma migration on prod** (new table) — low-risk additive migration, but it's a schema/infra
     change; confirm deploy path before running against prod.
  2. **`navigation.ts` / `AppNavigator` are marked read-only** in the vault INDEX. Adding one
     `SearchFilter` route is a deliberate, minimal exception — surface it explicitly rather than
     treating navigation as freely editable.
- Otherwise low-risk: UI restructure reuses shipped patterns; backend mirrors `SavedItem`; no auth,
  payment, or data-model-wide changes. No automated tests to touch ([[no-automated-tests]]).

## 8. Overall Confidence Score

**0.82** — The pattern-reuse is strong and both explored repos confirm clean insertion points.
The circle-icon ambiguity is **resolved** (it's just the filter, in the standard grey circle).
Remaining uncertainties: (a) exact **footer layout** (pill + one filter circle) and header→footer
input move is design-iterative (0.75); (b) `@oakplatforms/types` publish cadence for the
`RecentSearch` DTO (0.75); (c) the one **read-only navigation** exception (deliberate, called out).
None are blockers.

## Open Questions
1. **New filter icon** — user will send a specific `filter` asset; use `filter-2` (if it exists) as
   placeholder meanwhile.
2. **Recent-search normalization** — trim/lowercase before store+dedupe, or keep exact strings?
3. **@oakplatforms/types** — publish a new `RecentSearch` type, or inline a local DTO in mobile
   short-term?
