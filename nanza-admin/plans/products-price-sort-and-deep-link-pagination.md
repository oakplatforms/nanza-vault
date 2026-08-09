# Products: price sort + deep-linkable pagination

**Status:** Planned, on hold (2026-08-03). Was started and reverted — bigger project took priority. Everything below was verified against the code, so implementation is mechanical.

## Goal

1. In admin **Products**, sort entities by `Product.price` (expensive cards first, plus low→high).
2. Deep-linkable pagination/filters: open `/products?page=3&sort=price_desc&brandId=...` and land directly on that page with those filters — useful when working through a filtered list.

## Key facts (already verified)

- Sorting must be **server-side**: the Products list is paginated (10/page) via `GET /entities`, which currently has **no sort param** — it hard-codes `orderBy` listings-count → bids-count → id (`nanza-api/src/routers/entity.ts`, `GET /entities` handler ~line 132).
- The exact price-sort pattern already exists in `nanza-api/src/routers/list.ts` (~line 613, from the collection sort work): `sort=price_asc|price_desc` → Prisma `{ product: { price: { sort, nulls: 'last' } } }`.
- The API already accepts `page` as a query param, so deep-link pagination is **admin-only** work.
- `PaginationControls` displays 1-based ("Page {currentPage + 1}"); internal state and API are 0-based. URL param should be 1-based to match what the user sees.

## API change (nanza-api, entity.ts `GET /entities`)

1. Add `sort` to the destructured `req.query`.
2. Replace the hard-coded `orderBy` with:

```ts
const orderByClause: Prisma.EntityOrderByWithRelationInput[] =
  sort === 'price_asc'
    ? [{ product: { price: { sort: 'asc', nulls: 'last' } } }, { id: 'asc' }]
    : sort === 'price_desc'
      ? [{ product: { price: { sort: 'desc', nulls: 'last' } } }, { id: 'asc' }]
      : [ /* existing default: listings _count desc, bids _count desc, id asc — keep the id-tiebreak comment */ ]
```

No sort param → behavior unchanged for mobile/other consumers.

## Admin changes (nanza-admin, Products feature)

### `src/app/Products/data/fetchProductEntities.ts`
- Add optional `sort?: string` param; append `sort` to the query string when set; add it to the `queryKey` (`['productEntities', page, brandId, sort]`).

### `src/app/Products/index.tsx`
- **URL as source of truth** (replaces `useState` for page/brand): derive from the existing `useSearchParams`:
  - `page` — 1-based in URL; `currentPage = max(parseInt(page ?? '1') - 1, 0)`
  - `brandId` → `filterBrandId`
  - `sort` → `sortOrder` (`'' | 'price_asc' | 'price_desc'`)
- Setter helper writes/deletes params via `setSearchParams(prev => …, { replace: true })`; omit params at their defaults (page 1, no sort). Changing brand or sort clears `page`.
- Rewrite `handleNextPage`/`handlePrevPage` — they currently use functional `setCurrentPage(prev => …)`, which the URL-derived setter can't support; use `currentPage ± 1`.
- The existing `create=true` searchParams effect and the auto-select-first-brand effect keep working; auto-select now writes `brandId` into the URL (good — links become shareable).
- **Sort dropdown** (Tailwind `Select`, next to the brand select): Default / Price: high → low (`price_desc`) / Price: low → high (`price_asc`).
- **Search integration:** `performSearch` builds its own query params — append `sort` there too, and add `sortOrder` to the deps of the effect that re-runs search when `filterBrandId` changes.
- **Price column:** table has no price column today; add one (`productEntity.product?.price`) so the sort is visible. Note price arrives as a serialized Prisma Decimal (string).

## Verify

No tests (standing preference). `npm run build` in admin, `npx tsc --noEmit` in api; then manually: sort dropdown flips order across pages, and pasting a `/products?page=3&sort=price_desc&brandId=…` URL restores the exact view.
