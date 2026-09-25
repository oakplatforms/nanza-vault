# Collection Detail — Full-Screen Restyle

**Goal:** Replace the current `CollectionDetailScreen` (which reuses `CollectTab`'s
search-and-batch-save flow) with a full-screen, entity-modal-styled detail view: a banner
hero, jumbo title + item count, and a 3-column grid of cards. Each card shows name, product
number, and fair-market value, with a tap/hold quantity badge that writes to the server.

This is essentially a from-scratch screen. `CollectTab` is left untouched (still used by the
Trade "add to collection" flow).

---

## Visual layout (top → bottom)

1. **Banner hero** — the collection's `banner` image, full-width, `resizeMode: cover`, same
   treatment as the entity modal's hero (`entityModalImageContainer` — absolute, ~fixed
   height, rounded top corners) plus the same gradient cap fading into the background.
   - **No banner:** render a placeholder surface — a grayer undertone
     (`backgroundSecondary`/`surface`) **with a drop shadow**. No such placeholder exists
     today (`groupDetailBannerPlaceholder` etc. have no shadow), so add a new
     `collectionDetailBannerPlaceholder` style with `shadow*`/`elevation`.
2. **Floating `Header`** — `floating showBackButton backIcon="chevron-left-3"` (NOT
   `floatingInFlow`), overlaying an **edge-to-edge banner that runs to the very top** under the
   buttons — same treatment as the **group detail** screen. **No nav title, no solid header
   background.** Right side carries two dark-circle buttons in the **listing/bid-detail** style:
   **edit pencil** (`edit-2`, → CreateCollection) and **share** (`share-3`, gated on
   `referenceCode`). Back arrow on the left. (Corrected after first pass, per Skylar.)
3. **Headline** — big collection title (`displayName || name`), hero-sized.
4. **Item count** — under the title, e.g. "N items" (from `entityListCount`, or the query
   `total`). No search bar for now.
5. **3-column grid** of cards.

## The card (each grid cell)

- **Quantity badge** — top of the card: a circled number (1, 2, 3…) = quantity owned.
  - **Tap** → optimistic +1 in the circle, show a spinner inside the circle while the
    debounced save is in flight.
  - **Debounced save** — taps accumulate; ~500ms after the last tap, persist the final
    quantity (one request, not one-per-tap). Disable nothing; just debounce.
  - **Long-press (hold)** → open the existing `QuantityModal` (bottom sheet) seeded with the
    current quantity; its `QuantitySelector` lets them set any number up or down. On "Update"
    it saves and the badge reflects the new value.
- **Body** — product **name** and **product number**. (Tags dropped per latest instruction.)
- **Fair-market value** (bottom): reuse `listCardFairValueSection` → `listCardFairValueLogo`
  (`<FairMarketLogo>`, shrunk from 65×7 to fit a narrow card) over `listCardFairValuePriceRow`
  (`<PriceText>` + `.cents`), price from `entity.product.price` via the `splitPrice` helper.
  Use `listCardPriceDollarsSm` for the smaller price text. Logo on the right, price on the
  left with the "$", "fair market" label small underneath — per the voice spec.

---

## Data

- **Metadata:** `useQuery(getListByIdQueryConfig(listId, accountId))` → `ListDto`
  (`banner`, `displayName`/`name`, `entityListCount`, `entityListValue`). Already imported in
  the current screen.
- **Items:** `useInfiniteQuery` over `listService.getEntities(listId, params)`
  (`GET /list/:listId/entities`, page 20, `include=product&include=brand&include=set`),
  mirroring `UserCollectionDetailScreen` / `CollectTab`. Each page item is
  `{ id, entity: EntityDto, quantity }` — `id` is the **entity-list row id** needed for saves,
  `entity.product.{number,price}`, `entity.image`, `entity.displayName`.
  queryKey e.g. `['collectionDetail', listId, accountId]`, `staleTime: STALE_TIME['LIVE']`.

## Quantity persistence (per card)

Mirror the entity-modal single-row pattern (`entityListService`):
- If a row id is known and `quantity > 0` → `update(rowId, { quantity, accountId })`
- `quantity <= 0` → `delete(rowId, accountId)`
- No row yet and `quantity > 0` → `create({ listId, entityId, quantity, accountId })`
  (only relevant if the card can go from 0 — normally these all start ≥1)

Then `invalidateCollectionQueries(queryClient, entityId)` (covers `collectTab`, `list`,
`userCollection`, feed, search…). **Add `['collectionDetail']` to that util's invalidation
list** so other surfaces refetch after an edit here.

**Debounce mechanics:** keep a per-card local quantity (optimistic), a per-card
`isSaving` flag (drives the in-circle spinner), and a per-card debounce timer. Each tap:
bump local qty, set a 500ms timer (resetting any prior one). On fire: set `isSaving`, run the
save with the latest local qty, clear `isSaving`. Long-press's modal save goes through the
same save fn (bypassing the debounce — immediate).

---

## Files

- **Rewrite:** `src/screens/collections/CollectionDetailScreen/index.tsx` — new full-screen
  layout; drop the `CollectTab` render.
- **New component:** `src/components/collections/CollectionGridCard/index.tsx` — one grid
  cell: quantity badge (tap/hold), name, number, fair-market block. Owns its optimistic qty +
  debounce + spinner, calls an `onSaveQuantity(entityId, rowId, quantity)` prop.
- **New styles:** add to `src/styles/components/cards.ts` (grid card + badge) and
  `src/styles/components/modals.ts` or `cards.ts` (banner hero + shadowed empty placeholder).
  Reuse existing `listCardFairValue*` styles where possible. No inline styles / hardcoded
  colors (per REFERENCE rules).
- **Edit:** `src/utils/invalidateCollectionQueries.ts` — add `['collectionDetail']`.
- Route param `CollectionDetail: { listId }` already exists — no navigation change.

## Reuse (don't rebuild)

- `QuantityModal` + `QuantitySelector` for long-press. `min/max` via `minQuantity`/`maxQuantity`.
- `FairMarketLogo`, `PriceText`, `splitPrice` (lift `splitPrice` out of EntityModal into a
  shared util, or copy the ~8 lines — it's currently local to EntityModal).
- `Header` (floating), `LoadingIndicator`, `EmptyState`, `getImage`.
- Banner hero styles (`entityModalImageContainer`/`Image`/gradient) as the template.

---

## Decided

- **Tapping the card body** (name/image, anywhere but the badge) → **opens the `EntityModal`**
  for that card. (Confirmed by Skylar.)

## Notes / risks

- Grid card is narrow (~1/3 width minus gutters) — the fair-market logo + price + "Buy/Sell"
  columns from the modal won't all fit; keep it to logo + single price + "fair market" label.
- Debounced save + optimistic badge must reconcile with query invalidation (invalidation will
  refetch and could momentarily fight the optimistic value) — hold the optimistic value until
  the refetch lands, like EntityModal's `localQuantityOverride` clears on `serverTargetQuantity`
  change.
- Confirm whether quantity has a max (QuantitySelector is unbounded up by default).
