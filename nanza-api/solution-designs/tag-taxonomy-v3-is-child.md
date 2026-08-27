---
tags: [nanza-api, plan, tags, taxonomy]
---

# Tag Taxonomy v3 — One Model, `isChild` + Self-Join

**Status:** **implemented across all four repos 2026-08-04** (uncommitted, pending Skylar's
review). nanza-api: schema + migration `20260804120000_tag_taxonomy_v3_is_child`, routers, posts,
share/OG, types regenerated (published as `@oakplatforms/types@0.1.87`). Clients (against 0.1.87):
nanza-admin (one detail view + child chips, parents read-only on child pages), nanza-mobile (one
screen, data-driven children rail + parent chips, composer/hashtags/share collapsed), nanza-web-app
(share card V-only). All three typecheck clean. Supersedes the "PrimaryTagValue rename" idea
discussed earlier today (abandoned) and the v2 two-model design in
[[tag-taxonomy-v2|Tag Taxonomy v2]]. When this lands, fold it into the solution design and update
`INDEX.md`.

## The idea

Collapse the two-model taxonomy into one. `SupportedTagValue` stays (it's in prod) and becomes the
*only* tag-value model. A "secondary" value is now just a supported tag value with
**`isChild = true`**, linked to one or more parents through a **self-referential m:n join**.
`SecondaryTagValue` and its joins are deleted outright.

```
BrandTag (Flesh and Blood × class)
├─ SupportedTagValue (Warrior,  isChild=false) ─┐
├─ SupportedTagValue (Guardian, isChild=false) ─┼─ self m:n ─ SupportedTagValue (Boltyn, isChild=true)
└─ SupportedTagValue (Ninja,    isChild=false) ─┘
```

Why this wins over v2: one CRUD surface, one image pipeline, one DTO, one share/OG path, posts tag
everything through the single `PostSupportedTagValue` join, and the admin chip flow creates rows in
the same table it already manages.

Naming: `isChild` (not `isSecondary`) — explicit about the parent/child direction, and
`isPrimary` is already taken on this model with a different meaning. Default `false`, so every
existing prod row is automatically a parent — no backfill needed for the flag itself.

## Schema changes

```prisma
model SupportedTagValue {
  // ... existing fields unchanged (isPrimary keeps its current meaning) ...
  isChild Boolean @default(false)

  parents  SupportedTagValueParent[] @relation(name: "Child")
  children SupportedTagValueParent[] @relation(name: "Parent")

  @@index([brandTagId, isChild])
}

// Self m:n: which parents a child belongs to. Both ends must share a brandTag,
// and parent/child roles are validated in handlers (not expressible here).
model SupportedTagValueParent {
  createdAt DateTime @default(now())

  parent   SupportedTagValue @relation(name: "Parent", fields: [parentId], references: [id], onDelete: Cascade)
  parentId String
  child    SupportedTagValue @relation(name: "Child", fields: [childId], references: [id], onDelete: Cascade)
  childId  String

  @@id([parentId, childId])
  @@index([childId])
}
```

**Dropped:** `SecondaryTagValue`, `SupportedTagValueSecondaryTagValue`, `PostSecondaryTagValue`.
**Kept:** `PostSupportedTagValue` (decided 2026-08-04) — posts tag parents *and* children through it.

### Schema decisions (settled 2026-08-04, voice)

1. **Join model name** — `SupportedTagValueParent` (a row = "one parent of a child").
2. **`@@unique([brandTagId, name])`** — **not now** (Skylar): prod may hold duplicate supported
   tag value names and de-duping is out of scope for this change. Good future add; revisit later
   with the dup check `SELECT "brandTagId", name, count(*) FROM "SupportedTagValue" GROUP BY 1,2 HAVING count(*) > 1`.
3. **No nested children** (Skylar): one level deep only. Links validate
   `parent.isChild === false && child.isChild === true`. Relaxing later is additive.

## Invariants (handler-enforced, carried over from v2)

1. **A child has ≥1 parent.** Creating a child requires non-empty `parentIds[]`, links created in
   the same transaction. Unlinking the last parent is rejected (delete the child instead,
   deliberately).
2. **Same `brandTagId` on both ends of every link.**
3. **Parent/child roles** — see open decision 3.
4. **Deleting a parent blocks (409), never sweeps** when it is the last parent of any child; the
   error names the affected children.
5. All writes stay admin-gated with `validateAdmin` + `lastModifiedById` stamping, **including
   link/unlink** (v2 decision, unchanged).

## nanza-api changes

- **`src/routers/secondaryTagValue.ts` — delete.** The `/secondary-tag-value(s)` routes shipped
  API-side only (2026-07-31); no released client calls them. Remove from `all_routes.ts`.
- **`src/routers/supportedTagValue.ts`:**
  - List endpoints (`GET /supported-tag-values`) default to **`isChild = false`** so released
    admin/mobile builds keep seeing exactly the rows they saw before. Add explicit filters:
    `?isChild=true|false|all` (or `includeChildren`), and `?parentId=` to list one parent's
    children via the join.
  - Chip flow, re-pointed at the one table:
    - `POST /supported-tag-value/:id/child` — find-or-create by `(brandTagId, name)` with
      `isChild: true` (brandTagId derived from the parent, never from the client), then upsert the
      link. Idempotent; returns `created: true|false`.
    - `DELETE /supported-tag-value/:id/child/:childId` — unlink only; 409 when it's the last parent.
  - `GET /supported-tag-value/:id` includes `parents`/`children` (ids + display fields) in the
    detail payload.
- **Legacy `/sub-tag-value(s)` shims** — keep the paths returning the paginated envelope shape
  (empty `data` is fine) so old mobile builds render without crashing. They currently read
  `SecondaryTagValue`; re-point to children or hardcode-empty — either satisfies the bar.
- **Posts** (`src/routers/post.ts`, `src/validation/post.ts`, `src/services/postReferences.ts`):
  drop `secondaryTagValueIds` everywhere; tagging is `supportedTagValueIds` only (children allowed).
- **Share / OG** (`src/services/referenceResolver.ts`, `src/services/og/*`,
  `lambdas/metaHandler.ts`, `lambdas/ogHandler.ts`): remove the secondary-tag-value branch; the
  supported-tag-value path now serves children too. Fold the secondary reference-code type into the
  supported one (all unreleased, so no compat needed).
- **Types** (`packages/types`): delete `SecondaryTagValueDto`; `SupportedTagValueDto` gains
  `isChild`, `parents?`, `children?`.
- **Images:** children use the existing `supported-tag-value/<id>.webp` S3 folder. The
  `secondary-tag-value/` folder is dead.

## Slim facets read (2026-08-18)

`GET /brand-tag-facets?brandId=&valuesLimit=6&valuesIsPrimary=&includeChildren=&valueIds=` —
the lean read behind mobile's search filter and tag-picker screens (design + rationale in
[[../../nanza-mobile/solution-designs/tag-taxonomy#Slim facets read + paged sections (2026-08-18)|nanza-mobile tag-taxonomy]]).
A Prisma `select` of display fields only (brand tag id/index, tag id/name/displayName, value
id/name/displayName/isPrimary/index, optional `children.child` with the same three fields),
`take: valuesLimit` per tag ordered index-nulls-last then displayName — the same order as
`/supported-tag-values`, so the client's "Show more" pages continue the sequence — and a
`_count` that mirrors the value filter. `valueIds` pins named values into their tag's list
past the cut (a pinned child pins its parents); pins must still pass the filters. The
generic `/brand-tags` include is unchanged (admin still uses it). `/supported-tag-values`
gained an optional `include=` (generateIncludes) for the same flow.

## DB migration

Nothing "secondary" is in prod use, so this is close to drop-and-recreate:

1. `ALTER TABLE "SupportedTagValue" ADD COLUMN "isChild" BOOLEAN NOT NULL DEFAULT false;` + index.
2. Create `SupportedTagValueParent`.
3. **Existing `SecondaryTagValue` rows (dev): wipe, don't migrate** (decided 2026-08-04 — only a
   few exist and Skylar will recreate them as children by hand). No id preservation, no S3 moves.
4. Drop `PostSecondaryTagValue`, `SupportedTagValueSecondaryTagValue`, `SecondaryTagValue`.

## Client surfaces

- **nanza-admin** — `src/app/Tags/SecondaryTagValueDetail.tsx` merges into
  `SupportedTagValueDetail.tsx` (a child's detail is the same screen); chip list on the parent
  detail calls the new `/child` endpoints; top-level tag lists rely on the API's `isChild=false`
  default; delete `src/services/api/SecondaryTagValue.ts`; types in `src/types/index.ts`.
- **nanza-mobile** — **one detail screen + one share component for both levels** (decided
  2026-08-04, voice): delete `SecondaryTagValueScreen`; `SupportedTagValueScreen` renders parents
  and children alike, nav param is just the id.
  - **Children rail is data-driven, not flag-driven:** render the rail when the loaded value has
    children (`children.length > 0`) — only parents can — so no `isChild` conditionals scattered
    through the UI. Rail fetch = `?parentId=` query.
  - **No parent chips on child pages** (reversed 2026-08-04 after seeing it live — the page reads
    cleaner without them; the detail payload still carries `parents` if a hop-up affordance is
    ever wanted). A child page is banner, title, description, conversation.
  - **Taxonomy navigation pushes, never navigates:** rail thumbs and post hashtags target the
    same `SupportedTagValueScreen` route name, so `navigate` would swap params in place and lose
    the parent from the back stack — `push` stacks a new card and back walks up the taxonomy.
  - `PostComposer`/`PostItem` drop secondary arrays; share components (`ShareTagValueCard`,
    `ShareDetailView`, `useShareActions`, `useFetchByReference`) collapse to one tag-value type;
    delete `src/services/api/SecondaryTagValue.ts`.
- **nanza-web-app** — `src/app/ShareDetail/index.tsx` + `TagDetailCard.tsx` drop the secondary
  branch.

## Rollout

Single release. The only must-not-break surfaces are (a) prod `/supported-tag-value(s)` behavior
for released builds — protected by the `isChild=false` list default — and (b) `/sub-tag-value(s)`
returning the envelope shape. Everything secondary is unreleased and can change freely. The
"narrow window" argument from v2 applies double here: ship before real taxonomy data accumulates.

After landing: rewrite `documentation/solution-designs/tag-taxonomy-v2.md` → v3 (or new file +
INDEX.md update), and update `posts.md` where it references `PostSecondaryTagValue`.
