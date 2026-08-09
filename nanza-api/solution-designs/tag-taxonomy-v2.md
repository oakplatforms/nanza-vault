---
tags: [nanza-api, solution-design, tags, taxonomy]
---

# Tag Taxonomy v2 — Secondary Tag Values (Siblings, Not Children)

> **Status:** **Backend implemented 2026-07-31** (nanza-api only — admin and mobile still call the
> old endpoints). Supersedes the data model in v1. Landed in the same pass as [[posts|Posts]], which
> owns conversation behaviour on tag pages. **The migration has not been generated or run** — see
> [[posts|Posts]] § *What shipped* for the remaining steps.

> [!note] **Rename decided:** v1's `SubTagValue` becomes **`SecondaryTagValue`**. "Sub" implied a
> child of one supported tag value, which is exactly what v2 undoes; "secondary" keeps the pairing
> with `SupportedTagValue` (which stays as-is — it's in the prod DB and not worth a rename
> migration). The rename is nearly free right now: the `SubTagValue` table shipped today and is
> empty everywhere, and v2 rebuilds it regardless.

## What v1 shipped (2026-07-30, currently deployed)

The v1 design doc has been removed — it was superseded within a day and kept inviting people to
build the wrong model. This is the summary of what actually runs in dev/prod today, which the v2
work has to migrate away from.

**Promoting tag values to destinations.** Before v1, a tag was only metadata on a card:
`Tag → BrandTag → SupportedTagValue`, with cards carrying `EntityTag` rows whose `tagValue` is a
plain string. v1 made a supported tag value a *place you can visit*:

- `SupportedTagValue` already had `banner` / `description` columns that nothing read; v1 surfaced
  them and **renamed `logo` → `thumbnail`** (it's a carousel/filter square, not a brand mark).
- Added `GET /supported-tag-value/:id` and an admin-only multipart `PUT` for those fields, plus
  `GET /supported-tag-values?tagName=&brandId=` so clients can address a tag by its stable slug in
  a single request instead of fetching every brand tag to resolve an id.
- Added `SubTagValue` — the model v2 replaces — as a **one-to-many child of a single**
  `SupportedTagValue`, with the same page fields and full CRUD at `/sub-tag-value(s)`.
- Images: one S3 folder per model, keyed by row id (`supported-tag-value/<id>.webp`,
  `<id>-thumbnail.webp`), via `uploadImage`'s 4th `imageId` argument.
- Comments: `CommentableType` gained `SUPPORTED_TAG_VALUE`, and a `CommentSubTagValue` join let a
  comment be tagged with sub tag values as **hashtag filters** over the parent's single
  conversation. [[posts|Posts]] replaces this wholesale.
- Deliberately **not** built: any direct card ↔ sub-tag-value association (`EntitySubTagValue`).
  Cards still reach the taxonomy through the string-based `EntityTag` path, which v2 doesn't change.

Client surfaces from v1 remain accurate and are documented separately:
[[../nanza-admin/solution-designs/tag-taxonomy|admin]] (nested brand-tag dialog) and
[[../nanza-mobile/solution-designs/tag-taxonomy|mobile]] (`TagCarousel` + two detail screens).

## Why v1 is wrong

v1 makes the second level the **child of exactly one** supported tag value:

```
BrandTag (class)
└─ SupportedTagValue (Warrior)
   └─ SubTagValue (Boltyn)      ← belongs to Warrior and nothing else
```

That holds only while every hero belongs to one class. It doesn't survive contact with the real
card games: a hero like *Boltyn* can legitimately be a Warrior **and** a Guardian. Under v1 the only
way to express that is to duplicate Boltyn once per class — separate rows, ids, banners,
conversations — and the duplicates drift.

The mistake was treating "hero" as a *sub-category of class* when it is really its own axis that
*relates to* class. It is still conceptually "under" the classes — a secondary tag value is always
reached through supported tag values, never free-floating — hence *secondary*, not *sub*.

## The v2 model

Secondary tag values hang off `BrandTag`, as siblings of supported tag values, with a
**many-to-many** link between the two levels:

```
BrandTag (Flesh and Blood × class)
├─ SupportedTagValue (Warrior) ─┐
├─ SupportedTagValue (Guardian)─┼─ m:n ─ SecondaryTagValue (Boltyn)
└─ SupportedTagValue (Ninja)   ─┘        SecondaryTagValue (Olympia)
```

The `BrandTag` still bounds everything — Boltyn cannot leak outside Flesh and Blood. The scope is
intact; only the arity changed.

### Schema

```prisma
model SecondaryTagValue {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  name        String
  displayName String?
  banner      String?
  thumbnail   String?
  description String?

  // Parent — the brand tag, not one supported tag value
  brandTag   BrandTag @relation(fields: [brandTagId], references: [id], onDelete: Cascade)
  brandTagId String

  createdBy        Admin?  @relation(name: "CreatedBy", fields: [createdById], references: [id], onDelete: SetNull)
  createdById      String?
  lastModifiedBy   Admin?  @relation(name: "LastModifiedBy", fields: [lastModifiedById], references: [id], onDelete: SetNull)
  lastModifiedById String?

  supportedTagValues SupportedTagValueSecondaryTagValue[]
  posts              PostSecondaryTagValue[]              // see posts.md

  @@unique([brandTagId, name])
  @@index([brandTagId])
}

// The m:n join between the two levels.
model SupportedTagValueSecondaryTagValue {
  createdAt DateTime @default(now())

  supportedTagValue   SupportedTagValue @relation(fields: [supportedTagValueId], references: [id], onDelete: Cascade)
  supportedTagValueId String
  secondaryTagValue   SecondaryTagValue @relation(fields: [secondaryTagValueId], references: [id], onDelete: Cascade)
  secondaryTagValueId String

  @@id([supportedTagValueId, secondaryTagValueId])
  @@index([secondaryTagValueId])
}
```

`@@unique([brandTagId, name])` — uniqueness per brand tag, which is what "one Boltyn per game"
actually means.

### Admin-only writes (verified, carry over unchanged)

Every write on both `SupportedTagValue` and the second level is admin-gated today, and must stay
that way in v2 — including the **new link/unlink routes**, which are writes even though they touch
no row directly.

Two guards exist and they are not equivalent:

- **`validateAdmin(reqUser, adminId, 'admin')`** — used on create/update. Checks the JWT
  `principalId` matches the named admin's `user.authId`, that `reqUser.role === 'admin'`, *and*
  that `user.isAdmin` is true. It ties the action to a specific admin record, which is what
  populates `createdById` / `lastModifiedById`.
- **`validateRole(reqUser, 'admin')`** — used on delete. Checks only the JWT claim; it does no DB
  lookup and takes no admin id. Correct where there's no audit column to write, but it is the
  weaker of the two: it trusts the token's role claim alone.

**For v2:** link/unlink routes should use `validateAdmin` and record `lastModifiedById` on the
affected secondary tag value, so taxonomy edits stay auditable — an unlink can orphan or reshape a
page and shouldn't be the one write with no attribution.

### Invariants (all handler-enforced, in `src/validation/`)

The database cannot express either of these, so they live in validation guards covered by every
write path:

1. **At least one class.** A secondary tag value must always link to ≥1 supported tag value.
   `POST /secondary-tag-value` requires a non-empty `supportedTagValueIds[]`, created in the same
   transaction as the row. The unlink route rejects removing the last remaining link (the operator
   deletes the secondary tag value instead, deliberately).
2. **Same brand tag on both ends of a link.** The join can physically connect values from
   different brand tags; the link/create handlers must validate
   `secondaryTagValue.brandTagId === supportedTagValue.brandTagId`.

### Deleting a supported tag value: block, don't sweep

**Decided:** deleting a supported tag value that is the **last class** of any secondary tag value is
**rejected**, with the affected secondary tag values named in the error. A silent cascade-sweep was
considered and rejected — under [[posts|Posts]], a secondary tag value owns a banner, description,
and its own conversation; destroying Boltyn's page as a side effect of deleting the Warrior class
is unacceptable. The operator relinks or deletes the named values first, deliberately.

## Backward compatibility

**Old nanza-mobile builds must not break — but the feature need not keep working in them.**
`GET /app-version` only *prompts* an update, so shipped builds run indefinitely and keep calling
`/sub-tag-value(s)`. Since those tables are **empty**, there's nothing to preserve; the bar is that
old builds render without crashing.

- Keep `GET /sub-tag-values?supportedTagValueId=` and `GET /sub-tag-value/:id` as paths. Simplest
  is to have them read `SecondaryTagValue` under the old field names — `supportedTagValueId` maps
  onto the join, which is exactly what the old client means by it, so this is nearly free and old
  builds show a correct (single-class) subset. Returning an empty paginated envelope is also
  acceptable.
- **Either way, return the envelope shape** (`{ data: [], total, page }`), never a 404 or a bare
  object — the client flat-maps `data` and throws otherwise.
- New `/secondary-tag-value(s)` routes serve the full m:n model.
- Shims are removed later, on evidence of old-build traffic dying off — see the sunsetting note in
  [[posts|Posts]].

There's a **narrow window** here worth using: while these tables are empty, compatibility costs
almost nothing. That argues for shipping v2 *soon*, before real secondary tag values exist.

## Migration path from v1

v1's tables are deployed (dev + prod) but **empty** — they shipped the same day this redesign
landed and no sub tag values were created. **Verify with `SELECT count(*) FROM "SubTagValue"`**
(and the same for `CommentSubTagValue`); assuming zero rows, the migration is a clean
drop-and-recreate, no backfill:

- Drop `SubTagValue` and `CommentSubTagValue`.
- Create `SecondaryTagValue` and `SupportedTagValueSecondaryTagValue`.
- (The post-side join `PostSecondaryTagValue` arrives with the Posts stage, not here.)

If rows *do* exist, fall back to the add-backfill-drop sequence from the previous revision of this
doc (git history), plus a duplicate-name check across classes before applying
`@@unique([brandTagId, name])`.

## Surface impact

- **nanza-api** — `secondaryTagValue` router replaces `subTagValue`: routes at
  `/secondary-tag-value(s)`; `POST` takes `supportedTagValueIds[]` (non-empty); `PUT` for
  fields/images; list filters by `brandTagId` or by `supportedTagValueId` (via the join). S3 folder
  becomes `secondary-tag-value/`. Plus:
  - **`POST /supported-tag-value/:id/secondary-tag-value`** — the admin chip flow in one call:
    given `{ name, displayName }`, look up an existing secondary tag value by
    `(brandTagId, name)` — derived from the supported tag value's own `brandTagId`, never trusted
    from the client — and either link the existing row or create it and link. Idempotent: linking
    something already linked is a no-op success, not a duplicate-key error. Returns the secondary
    tag value plus whether it was created, so the UI can say "linked existing Boltyn".
  - **`DELETE /supported-tag-value/:id/secondary-tag-value/:secondaryTagValueId`** — unlink only.
    Rejects if it would remove the last class (the ≥1 invariant), naming the value so the operator
    knows to delete it outright instead.

  Both are writes, so both use `validateAdmin` and stamp `lastModifiedById`.
- **nanza-admin** — the supported-tag-value detail **keeps its secondary-value chip list**; what
  changes is what adding a chip means. Typing a name there is now **find-or-create, then link**:
  if a secondary tag value with that name already exists under the brand tag, the action just
  creates the join (Boltyn is now also a Guardian); if not, it creates the row *and* the join in
  one call. Removing a chip **unlinks** — it does not delete the secondary tag value, unless that
  was its last class, which the API rejects (see above) so the operator deletes it deliberately.
  This needs one endpoint doing find-or-create-and-link server-side rather than the client
  round-tripping a lookup; the API section covers it. A brand-tag-level list of all secondary tag
  values (with a multi-select of their classes) is the natural companion view for bulk editing,
  but the chip flow is the primary path.
- **nanza-mobile** — `SubTagValueScreen` → `SecondaryTagValueScreen`; `TagCarousel`'s second-level
  rail resolves through the join (a supported tag value page still shows "its" secondary values —
  decided: the rail stays). Comment wiring on these pages is defined by [[posts|Posts]].
- **types** — `SecondaryTagValueDto` replaces `SubTagValueDto` in `@oakplatforms/types`.

## What shipped (2026-07-31)

`src/routers/secondaryTagValue.ts` replaces `subTagValue.ts` and carries the whole surface:

- Full CRUD at `/secondary-tag-value(s)`; `POST` requires a non-empty `supportedTagValueIds[]` and
  rejects ids spanning more than one brand tag. Images land under `secondary-tag-value/<id>.webp`
  and `<id>-thumbnail.webp`.
- **`POST /supported-tag-value/:id/secondary-tag-value`** — the find-or-create-and-link chip flow
  in one call. `brandTagId` is derived from the supported tag value, never taken from the client,
  which is what stops a hero from one game being linked into another's classes. The join is
  `upsert`ed, so re-adding an existing chip is a no-op rather than a duplicate-key error. Returns
  `created: true|false` so the UI can say "linked existing Boltyn".
- **`DELETE /supported-tag-value/:id/secondary-tag-value/:secondaryTagValueId`** — unlink only,
  `409` naming the value when it's the last class.
- The supported-tag-value delete **blocks with `409`**, naming every value that would be orphaned,
  instead of sweeping them.
- Legacy `/sub-tag-value(s)` shims serve `SecondaryTagValue` under the old field names —
  `GET /sub-tag-value/:id` flattens the join back to a single `supportedTagValueId` so old clients
  parse it.

All writes use `validateAdmin` (which checks the JWT `principalId` against the admin's
`user.authId`, the role claim, *and* `isAdmin`) and stamp `lastModifiedById` — including link and
unlink, which reshape pages and shouldn't be the one taxonomy write with no attribution.

## Decisions log (2026-07-30)

- Name: **`SecondaryTagValue`** (Skylar; "sub" wrong post-v2, `SupportedTagValue` stays to avoid a
  prod migration).
- Orphan handling: **block** STV delete with named values, no silent sweep.
- The supported-tag-value page keeps its secondary-value rail.
- Admin keeps adding secondary values from the **supported-tag-value modal**, with
  **find-or-create-then-link** semantics; removing a chip unlinks rather than deletes (Skylar).
- Every write is admin-gated, **including link/unlink** — audited via `validateAdmin` +
  `lastModifiedById`, not the weaker `validateRole`.
- **Old mobile builds must not break, but need not keep the feature** (Skylar): `/sub-tag-value(s)`
  paths stay as shims (ideally reading `SecondaryTagValue` under old names — nearly free), always
  returning the paginated envelope shape. Ship v2 soon, while the tables are still empty.
- A post/comment may be tagged with **multiple** secondary tag values (join allows it; UI exposes it).
- Ships **before** the Comment→Post rename.
