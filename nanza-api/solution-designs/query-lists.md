---
tags: [nanza-api, solution-design, query-lists]
---

# Query Lists — Solution Design

> **Status:** Design only. No models or endpoints exist yet. First consumer is the homepage UI;
> Query Lists will be reused elsewhere over time.

## Overview

A **Query List** is a saved, admin-configured *dynamic* collection. Where a [[data-sync|`List`]]
holds a fixed set of entities via explicit `EntityList` join rows, a Query List holds **no rows** —
it holds the *rules* for a query, and its contents are resolved live at read time by running that
query against the marketplace.

The motivating use is the homepage: instead of hand-curating "listings under $100" or "bids that
match a Defense Reaction card," an admin builds a Query List once, and the API materializes the
matching listings/bids/bulk listings whenever the homepage asks for them.

Two independent dimensions define a Query List:

1. **Type(s) — what to pull.** A multi-select of sources: `LISTING`, `BID`, `BULK_LISTING`,
   `ENTITY` (and `SET` reserved for later). Pick one or many; the selected types are simply
   *unioned* — there is no AND/OR here, "listings and bids" means both.
2. **Criteria — how to filter.** An ordered set of individual criterion rows (price bounds, entity
   tag, set, group, talent, …), combined by a single **`AND` / `OR`** combinator that the admin
   chooses. `AND` = an item must match every criterion; `OR` = match any one.

This split is deliberate: the type dimension is a set-union with no logic knob; the combinator knob
lives **only** on the criteria. That mirrors how the admin thinks — "give me listings *and* bids,
where price > $10 *and* price < $100."

## Data model

Three new pieces: a `QueryList` root, a `QueryCriterion` child (one row per condition so admins add
and remove them individually — **not** a JSON blob), and two enums. Names, images, brand association,
and audit columns copy `List` so the admin surface can reuse existing patterns.

### `QueryListType` enum

The set of sources a Query List can draw from. `SET` is included now (per request) so the enum is
stable even though the resolver won't support it in v1.

```prisma
enum QueryListType {
  LISTING
  BID
  BULK_LISTING
  ENTITY
  SET
}
```

### `QueryCombinator` enum

How the criteria combine. Exactly one value per Query List.

```prisma
enum QueryCombinator {
  AND
  OR
}
```

### `QueryCriterionField` / `QueryCriterionOperator` enums

The vocabulary a single criterion is built from — the "column," the comparison, and (below) the
value. Kept as enums, not free strings, so the admin UI is a set of dropdowns and the resolver has a
closed set to translate.

```prisma
enum QueryCriterionField {
  PRODUCT_PRICE
  ENTITY_TAG
  SET
  GROUP
  TALENT
}

enum QueryCriterionOperator {
  EQUALS
  NOT_EQUALS
  GREATER_THAN
  GREATER_THAN_OR_EQUAL
  LESS_THAN
  LESS_THAN_OR_EQUAL
}
```

### `QueryList` model

```prisma
model QueryList {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // custom
  name          String
  displayName   String?
  banner        String?
  logo          String?
  thumbnail     String?
  description   String?
  isPrivate     Boolean? @default(false)
  isPrimary     Boolean? @default(false)
  index         Int?
  referenceCode String?  @unique

  // query definition
  types      QueryListType[]
  combinator QueryCombinator @default(AND)

  // relations
  brand            Brand?   @relation(fields: [brandId], references: [id], onDelete: SetNull)
  brandId          String?
  createdBy        Admin?   @relation(name: "CreatedBy", fields: [createdById], references: [id], onDelete: SetNull)
  createdById      String?
  lastModifiedBy   Admin?   @relation(name: "LastModifiedBy", fields: [lastModifiedById], references: [id], onDelete: SetNull)
  lastModifiedById String?

  criteria QueryCriterion[]

  @@index([brandId])
  @@index([isPrivate])
  @@index([isPrimary])
  @@index([referenceCode])
}
```

Notes:
- `types` is a Postgres enum array (`QueryListType[]`) — the multi-select. Empty array is invalid
  (validation rejects it); at least one source must be chosen.
- `isPrimary` copies `List.isPrimary` (indexed) — flags a Query List as a **homepage** row. The
  homepage is exactly the set of `isPrimary` Query Lists.
- `index` is the ordering/priority field (same field the rest of the system orders by). Consumers
  sort the `isPrimary` set by `index` to decide which rows show and in what order on the homepage.
- No `account` relation (unlike `List`): Query Lists are admin-authored marketplace configuration,
  not user-owned collections. No freemium cap applies.
- No stored count/value derived fields on the root — a Query List's contents are inherently dynamic,
  so any count is computed per-request by the resolver (see below) and never persisted.

### `QueryCriterion` model

One row per condition. The `value` is a single string column; the resolver interprets it against
`field` + `operator` (numeric for price, `tagName=tagValue` for tags, a set/group name or id
otherwise). Keeping value as one typed-by-field string avoids a wide table of nullable typed columns.

```prisma
model QueryCriterion {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // custom
  field    QueryCriterionField
  operator QueryCriterionOperator
  value    String
  index    Int?

  // relations
  queryList   QueryList @relation(fields: [queryListId], references: [id], onDelete: Cascade)
  queryListId String

  @@index([queryListId])
}
```

`onDelete: Cascade` means deleting a Query List removes its criteria automatically, exactly like
`EntityList` under `List`.

## How the examples map

Every example from the request expresses cleanly as `types` + a set of `QueryCriterion` rows joined
by one combinator:

| Example | types | combinator | criteria (field / operator / value) |
|---|---|---|---|
| Listings above $100 (incl. lots) | `LISTING`, `BULK_LISTING` | `AND` | `PRODUCT_PRICE > 100` |
| Listings/bids $10–$100 | `LISTING`, `BID`, `BULK_LISTING` | `AND` | `PRODUCT_PRICE > 10`, `PRODUCT_PRICE < 100` |
| Listings/bids $0.50–$10 | `LISTING`, `BID`, `BULK_LISTING` | `AND` | `PRODUCT_PRICE >= 0.50`, `PRODUCT_PRICE < 10` |
| Listings/bids below $0.50 | `LISTING`, `BID` | `AND` | `PRODUCT_PRICE < 0.50` |
| Playable by a particular hero | `LISTING`, `BID` | `AND` | `TALENT = <hero>` (matches that talent + `..generic..`) |
| Matches an entity tag | `LISTING`, `BID` | `AND` | `ENTITY_TAG = defense reaction` |
| Matches multiple entity tags | `LISTING`, `BID` | `AND` | `ENTITY_TAG = defense reaction`, `ENTITY_TAG = has action value` |
| Belongs to a group | `LISTING`, `BID` | `AND` | `GROUP = <group>` |
| Belongs to a set | `LISTING`, `BID` | `AND` | `SET = <setName>` |

The "between $10 and $100" case is the canonical demonstration of the design: two rows
(`> 10`, `< 100`) with `AND`. Switch the combinator to `OR` and the same two rows mean "under $100 or
over $10" — the admin controls the meaning with the one dropdown.

## Resolution (read path)

A resolver (planned `src/services/queryList.ts`) turns a `QueryList` into results. It runs the query
**once per selected type** and unions the results, because the criteria translate to a *different*
Prisma `where` on each source table — the fields don't live in the same place across `Listing`,
`Bid`, `BulkListing`, and `Entity`. This per-type translation is the crux of the design, so it is
spelled out against the real schema below.

### Per-type translation matrix

For each selected type, every `QueryCriterion` becomes one Prisma clause; the whole set is wrapped in
`{ AND: [...] }` or `{ OR: [...] }` per the combinator. The clause shape depends on both the field
**and** the type:

| Criterion field | `LISTING` (`Listing`) | `BID` (`Bid`) | `BULK_LISTING` (`BulkListing`) | `ENTITY` (`Entity`) |
|---|---|---|---|---|
| `PRODUCT_PRICE` | `price` (Decimal) | `price` (Decimal) | `totalPrice` (Decimal, derived) | `product.price` (Decimal) |
| `ENTITY_TAG` | `entity.entityTags.some({ tag: { name }, tagValue })` | same via `entity` | `listings.some({ entity: { entityTags.some(...) } })` | `entityTags.some({ tag: { name }, tagValue })` |
| `TALENT` | `entity.entityTags.some({ tag: { name: 'talent' }, tagValue: { in: [hero, '..generic..'] } })` | same via `entity` | via `listings.some({ entity: … })` | `entityTags.some({ tag: { name: 'talent' }, … })` |
| `SET` | `entity.set.name` (or `setId`) | same via `entity` | `listings.some({ entity: { set: { name } } })` | `set.name` |
| `GROUP` | `groups.some({ groupId })` (`GroupListing`) | `groups.some({ groupId })` (`GroupBid`) | `groups.some({ groupId })` (`BulkListingGroup`) | n/a — entities don't join groups |

The three schema realities this matrix pins down:

1. **Price lives on different columns.** `Listing.price` and `Bid.price` are stored decimals;
   `BulkListing` has **no per-unit price** — it exposes `totalPrice` (a derived field), so a bulk
   listing is compared by its total. `ENTITY` compares `product.price`.
2. **BulkListing has no direct entity.** It's a bag of child `listings`, so entity-scoped criteria
   (`ENTITY_TAG`, `TALENT`, `SET`) reach the entity through `listings.some({ entity: … })` — "any
   line in the bulk matches." This is the one type where entity criteria go two relations deep.
3. **Tags go through `tag.name`.** An `EntityTag` has `tagValue` plus a `tag` relation carrying
   `name`, so both entity-tag and talent criteria filter on `{ tag: { name }, tagValue }`. `TALENT`
   is just the `talent`-named tag, and it matches the hero **plus** `'..generic..'` so generic cards
   surface for every hero.

`SET` as a *source type* is reserved and skipped by the v1 resolver (logged); `SET` as a *criterion
field* works today (it filters entities/listings by their set).

### Then

- **Enrich + paginate** each type's results like the existing list read paths; the combined count is
  computed per request, never stored.
- **Legal criteria only.** Not every field/operator/type combination is valid — numeric operators
  (`>`, `<`, …) only make sense for `PRODUCT_PRICE`; `GROUP`/`SET`/tags take `EQUALS`/`NOT_EQUALS`;
  `GROUP` is invalid for the `ENTITY` type. Validation (planned `src/validation/queryList.ts`)
  enforces this matrix **on write**, so the resolver only ever sees criteria it can translate.

## Admin surface (nanza-admin)

The admin exposes Query List authoring under a **Homepage** section — because the homepage *is* the
set of `isPrimary` Query Lists, ordered by `index`. Operators build the homepage by creating primary
Query Lists and ordering them. The builder reuses the List editor's shell (name, images, brand) plus
two query-specific controls: a **Types** multi-select over `QueryListType`, and a **Criteria** row
list (each row = field/operator/value + add/remove) with one `AND` / `OR` combinator dropdown above.

The full admin-surface design lives in the **nanza-admin** vault at
[[../../nanza-admin/solution-designs/query-lists|nanza-admin — Query Lists / Homepage]]; this file is
the canonical cross-repo model.

## Key decisions & rationale

- **Rows, not JSON.** Criteria are individual `QueryCriterion` rows so the admin adds/removes them
  one at a time and so each is independently validatable and indexable — explicitly chosen over a
  JSON blob on the root.
- **Two dimensions, one combinator.** Type is a set-union multi-select with no logic knob; the
  `AND`/`OR` combinator applies **only** to the criteria. This matches how operators reason about the
  query and keeps a single, unambiguous place for the boolean.
- **`SET` in the enum now, unsupported in v1.** Added to `QueryListType` up front (per request) so
  the enum is stable and migrations don't churn; the resolver simply skips it until built.
- **Copy `List`, don't extend it.** Query Lists share `List`'s presentational columns and audit
  pattern, but a `List` is a user-owned bag of fixed entities and a `QueryList` is admin config with
  zero stored membership. Overloading `List` with a nullable rules blob would muddy both; a parallel
  model keeps each concept clean while letting the admin UI reuse the shell.
- **Nothing persisted about contents.** Because results are live, no count/value is stored on the
  root — mirroring how `List`'s `entityListCount`/`entityListValue` are treated as recompute-only,
  taken to its logical end.

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
- [[data-sync|Data sync]] — the fixed-membership `List` / `EntityList` model this contrasts with
- [[payments|Payments]] — price/delivery model behind the price criteria
