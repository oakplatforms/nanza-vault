# Move Collect experience → Profile Collections tab

## Goal
Move the per-collection "Collect" workflow (search-to-add, plus/minus steppers, running
total, Save, scan) out of the Trade tab and onto the user's OWN profile Collections tab.
Tapping a collection card opens the Collect experience scoped to that collection. Remove
the Collect tab from Trade.

## Decisions locked with user
- NEW screen `CollectionDetail({ listId })` renders the existing `CollectTab` scoped to one collection.
- REMOVE `'Collect'` from `TradeScreen` `TAB_ORDER` now.
- Profile Collections tab: count header ("N Collections" + small "+" → CreateCollection),
  then a list of `CollectionTargetCard`s → tap opens `CollectionDetail`. No edit pencil / share / link this pass.
- **Paginate** the collections list (useInfiniteQuery, the lists endpoint supports page/limit).
- **Pre-target scan** to the opened collection (thread `listId` through ScanCamera → CollectScanTarget → CollectScanItems).

## Approach
1. `CollectTab` — add additive `lockedCollection?: ListDto` prop.
   - When set: `activeCollectionId = lockedCollection.id`, force `listService.getEntities(listId)`
     (not the by-account primary endpoint), hide the multi-collection switcher, ignore CollectionSelectionContext.
   - Scan icon passes `listId={lockedCollection.id}` into `ScanCamera`.
   - When absent: behavior identical to today (Trade no longer renders it anyway).
2. NEW `src/screens/collections/CollectionDetailScreen/index.tsx`
   - Resolve `ListDto` by `listId` (paginated COLLECTION lists / getListByIdQueryConfig fallback).
   - Render `Header` + `<CollectTab lockedCollection={list} bottomPadding=... />`. Loading / not-found states.
3. Navigation
   - `src/types/navigation.ts`: add `CollectionDetail: { listId: string }`.
   - `src/types/navigation.ts`: extend `ScanCamera` params with `listId?: string`; extend `CollectScanTarget` with `listId?: string`.
   - `src/navigation/AppNavigator.tsx`: register `CollectionDetail`.
4. `CollectScanTargetScreen` — when `route.params.listId` present, skip the picker and
   navigate straight to `CollectScanItems({ listIds: [listId] })`.
5. `ScanCameraScreen` — carry `listId` from its own params into `CollectScanTarget`.
6. Profile `CollectionTab` redesign
   - Replace carousel + "N collected" row + `List` grid.
   - Count header: left "N Collections" (reuse `listResultsHeader` + `collectionPickerMeta`),
     right "+" button (Icon name "add", like ProfileListThumbnails:86-101) → `navigate('CreateCollection')`.
   - `FlatList` of `CollectionTargetCard` (isSelected=false) → `onPress` → `navigate('CollectionDetail', { listId })`.
   - Paginate via `useInfiniteQuery` (type COLLECTION). Keep `CollectionTabHandle.handleEndReached`
     wired to fetch the next page so ProfileScreen's scroll-driven pagination keeps working.
7. Remove Collect from `TradeScreen` — `TAB_ORDER`, `activeTab === 'Collect'` render block,
   Collect search-placeholder branch, and now-dead `CollectTab` / `collectionList` wiring. Clean unused imports.
8. New style additions in `src/styles/components/profile.ts` (count header + card list spacing).
9. Typecheck + lint changed files.

## Files
- NEW `src/screens/collections/CollectionDetailScreen/index.tsx` (~50)
- EDIT `src/screens/trade/TradeScreen/CollectTab.tsx` (~20: lockedCollection gating + scan listId)
- EDIT `src/screens/profile/ProfileScreen/tabs/CollectionTab.tsx` (~90: rewrite render + paginate, keep ref API)
- EDIT `src/screens/trade/TradeScreen/index.tsx` (~20 removed)
- EDIT `src/screens/trade/CollectScanTargetScreen/index.tsx` (~10: skip picker when listId)
- EDIT `src/screens/trade/ScanCameraScreen/index.tsx` (~3: carry listId)
- EDIT `src/navigation/AppNavigator.tsx` (~10)
- EDIT `src/types/navigation.ts` (~3)
- EDIT `src/styles/components/profile.ts` (~20)

9 files (1 new).

## P0 compliance
- All styles → buckets; no inline objects. No hardcoded colors/sizes; theme tokens only.
- No `any`; no custom DTOs (ListDto/EntityDto from src/types). Price "$" via PriceText (already used in CollectTab).
- Navigation changes pre-approved by user.

## Deferred / known
- Scan now pre-targets the opened collection (was going to be deferred; user opted in).

## Risks
- Keep `CollectionTabHandle.handleEndReached` working against the new paginated FlatList (ProfileScreen calls it).
- Clean removal of dead Collect wiring in TradeScreen (lint).
- `CollectTab` switcher-hide + endpoint routing must be exact so a non-primary collection loads via getEntities.

## Overall confidence: 0.84
