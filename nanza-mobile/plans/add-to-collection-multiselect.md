# Design: "Add to Collection" — per-entity, multi-collection screen

## Goal

From the entity detail action menu (the ellipsis), replace the current
single-collection inline controls (quantity stepper + "Update collection" header,
and the "Remove from {name}" row) with an **Add to Collection** action that opens a
**full screen**. On that screen you set, for the one entity you came from, **how
many of it to put in each of your collections**, then commit everything in one
batch. **Share stays** in the action menu.

> This is a **screen**, not a bottom sheet (per latest direction). It effectively
> re-introduces the previously-deleted AddToCollections screen, intentionally.
> Navigation registration is required (see Escalation).

## Mental model

You're already inside an entity, so the entity is fixed. The screen is a list of
**your collections**; each row lets you choose a quantity of *this entity* for
*that collection*:

> "For my 6 collections: add 3 of this card to Marvel, 2 to Vintage, 1 to
> Wishlist, 0 to the rest." → one **batch** write.

## Visual spec

Header: standard `Header` with a back/close and title "Add to Collection" (show
the entity name as subtitle or a small card up top). Below: a `FlatList` of
collection rows. Sticky/footer **Update** button commits the batch.

Each **row is a card** modeled on two existing patterns:

- **Card chrome** like the group-list card (`cardStyles.groupCard` /
  `groupCardContent` / `groupCardThumbnail` / `groupCardTextContainer` in
  `GroupsListScreen`): a thumbnail/banner image on the left, name + meta stacked.
  - Collection image source: `collection.thumbnail || collection.banner ||`
    fallback (a small mosaic or placeholder). Name: `displayName || name`. Meta:
    `entityListCount` items.
- **+/- circle stepper** exactly like `TradeCardSelectionRow`
  (`tradeStyles.selectionQuantityStepper` / `selectionQuantityButton` /
  `selectionQuantityText`): two circular buttons with `minus`/`plus` icons and the
  number between them. Selected state highlights the card outline
  (`selectionCardRowSelected`) — i.e. **quantity > 0 → highlighted/white outline**,
  matching the "highlight makes it white" behavior the collect flow uses.

```
┌───────────────────────────────────────────────┐
│  [img]  Marvel                     ( – ) 3 ( + )│  ← qty>0: highlighted outline
│         128 items                               │
├───────────────────────────────────────────────┤
│  [img]  Vintage                    ( – ) 0 ( + )│  ← qty 0: normal outline
│         40 items                                │
├───────────────────────────────────────────────┤
│  [img]  Wishlist                   ( – ) 1 ( + )│
│         12 items                                │
└───────────────────────────────────────────────┘
            [          Update          ]
```

No checkbox needed — quantity is the single source of truth (0 = not in it,
highlighted when > 0), same as `TradeCardSelectionRow`.

## Conditional: single vs. multiple collections

The new screen only appears when it adds value:

- **Account has exactly one collection** → **keep today's UX unchanged**: the
  inline quantity stepper + "Update collection" header and the "Remove from
  {name}" row in the action sheet, acting on that one collection. Do **not** open
  the new screen.
- **Account has two or more collections** → drop the inline stepper/remove rows
  and show a single **"Add to Collection"** action that opens the new screen.

So `EntityActionModal` branches on `collections.length`:
- `<= 1`: render the existing `quantityHeader` + "Remove from {name}" (current
  behavior, current wiring).
- `>= 2`: render the "Add to Collection" navigation row instead.

Share is present in both cases. Count uses the COLLECTION-type lists for the
account (the same `collections` array passed in).

## Data & write path (backend already done)

- **Collections**: `currentUser.account.lists` filtered to `type === 'COLLECTION'`
  (or `getAllListsQueryConfig(accountId)` for freshness).
- **Initial quantities for this entity**:
  `extractEntityListData(entity.entityList)` → `{ existingQuantities, existingIds }`
  keyed by `listId` (`ListModal/data/fetchExistingEntityLists.ts`).
- **Commit**: `entityListService.batchUpdateByEntity({ entityId, accountId,
  items: changedRows.map(r => ({ listId, quantity })) })` →
  `POST /entity-list/batch-by-entity`. Send only rows whose quantity changed;
  `quantity: 0` removes. **One request, no per-collection looping.**
- **After commit**: run EntityModal's existing `invalidateCollectionQueries`
  (lists, feed, entities, search, savedItems, entity detail), then
  `navigation.goBack()`.

## Components & changes

### 1. NEW screen `AddToCollectionScreen`
`src/screens/collections/AddToCollectionScreen/index.tsx`
- Route param: `{ entity: EntityDto }` (or `{ entityId }` and refetch — prefer
  passing the entity to avoid a round-trip, mirroring how `Entity`/`CollectTab`
  pass objects).
- Loads collections + initial quantities; holds a working `Record<listId, qty>`
  in state; renders `Header` + `FlatList` of `CollectionAddRow` + footer `Button`.
- "Update" diffs working vs. initial, calls `batchUpdateByEntity`, invalidates,
  goes back. Guests → existing auth-required guard.

### 2. NEW row `CollectionAddRow`
`src/components/collections/CollectionAddRow/index.tsx`
- Props: `{ collection: ListDto, quantity: number, onIncrement, onDecrement,
  maxQuantity? }`.
- Built from the group-card chrome + the `TradeCardSelectionRow` stepper. Styles
  go in a bucket (reuse `cards.ts` group-card classes + `trade.ts` stepper
  classes, or add `collectionAddRow*` to `cards.ts`). No inline styles.

### 3. EDIT `EntityActionModal` (`ActionModal/Entity.tsx`)
- **Branch on collection count** (new prop `collectionCount` or pass `collections`):
  - `<= 1`: keep the existing `quantityHeader` (single stepper + "Update
    collection") and the "Remove from {name}" row — **unchanged**.
  - `>= 2`: hide those; show an **"Add to Collection"** action row (icon e.g.
    `plus`/`folder-plus`) whose `onPress` navigates to `AddToCollection` with the
    entity.
- Keep **Share** in both branches.
- Keep the existing single-collection props (`collectionQuantity`,
  `onQuantityUpdate`, `onRemoveFromCollection`) — they're still used in the
  single-collection branch. Add the new nav trigger alongside.

### 4. EDIT `EntityModal` (`modals/EntityModal/index.tsx`)
- Keep all current single-collection wiring (it powers the `<= 1` branch).
- Pass the collection count (or `collections`) to `EntityActionModal` so it can
  branch, plus an `onAddToCollection` that navigates to the new screen with the
  entity.
- The new screen reuses the same `invalidateCollectionQueries` keys (extract the
  helper to a shared util, or duplicate the key list).

### 5. EDIT navigation (REQUIRED)
- Register `AddToCollection` in `AppNavigator.tsx` (next to `CollectionDetail` /
  `CollectionPicker`).
- Add `AddToCollection: { entity: EntityDto }` (or `{ entityId: string }`) to
  `RootStackParamList` in `src/types/navigation.ts`.

### 6. Other `EntityActionModal` call sites
- Grep all usages (global search "primary collection" mode included). Each must
  navigate to the new screen instead of passing single-collection handlers.
  Confirm search should use the same screen.

## Escalation (per CLAUDE.md P0 #7 — navigation)

This **requires navigation changes** (new screen + param type), which P0 normally
forbids. Memory note "Nav changes permitted when cleaner" says the user accepts
this when it's the right call; a dedicated screen is clearly cleaner than cramming
a scrollable multi-collection editor into a bottom sheet. **Confirm before
implementing.**

## Open questions

1. **Search menu**: same Add-to-Collection screen there too, or entity-modal only?
2. **Max quantity** per collection — cap or unbounded?
3. **Create-new-collection** affordance on this screen (a "+ New collection" row)?
   In scope?
4. **Collection image**: use `thumbnail`/`banner`; if neither, a mini-mosaic of the
   collection's cards (like `CollectionTargetCard`) or a plain placeholder?
5. **Entity display** on the screen — small card/header showing which entity
   you're adding?

## Effort

Medium, mostly additive: 1 new screen, 1 new row component, edits to
`EntityActionModal` + `EntityModal`, nav registration, and updating other call
sites. **No backend work** (batch endpoint exists). No changes to shared
infra beyond the nav registration.
