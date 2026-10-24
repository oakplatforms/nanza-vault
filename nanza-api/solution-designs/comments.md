---
tags: [nanza-api, solution-design, comments]
---

# Comments — Polymorphic Threads, Extended to Tag Pages

> [!warning] **Historical.** `Comment` was replaced by `Post` in the nanza-api backend on
> 2026-07-31 — the model, `CommentSubTagValue`, and `CommentableType` are gone from the schema, and
> `src/routers/comment.ts` is now only legacy shims. See [[posts|Posts]] for the live design. This
> document is kept as the record of what the comment system was, since prod mobile builds still
> speak its API and the shims exist to serve them.

## How the existing system works

One `Comment` model serves every commentable surface via a polymorphic target — no join table per
surface:

- **Target:** `commentableType` (enum `CommentableType`: `LISTING | BID | ENTITY | BULK`) +
  `commentableId` (plain string, indexed together). No DB-level FK to the target — integrity is
  enforced at write time by `validateCommentableExists` (`src/validation/comment.ts`), which
  switches on the type, checks the row exists, and (for listing/bid/bulk) that it's still active.
- **Threading:** self-referential `parentId` → `replies` (`"CommentThread"` relation). One level of
  UI nesting today, but the model supports arbitrary depth.
- **Author:** `accountId` (cascade delete). **Moderation:** `status` (`ACTIVE` default) for soft
  removal. **Rate limit:** `validateCommentRateLimit` caps an account at 50 comments/day.
- **Routes** (`src/routers/comment.ts`): `GET /comments?commentableType=&commentableId=&parentId=`
  (paginated), `POST /comment`, plus edit/delete guarded to the author.

## Extension: one conversation per supported tag value, filterable by sub tag value

The supported tag value page gets **one conversation** — "Warrior
is my favorite class" lives on the *Warrior* page. Sub tag values are **not a separate comment
surface**: they act like hashtags *within* that conversation. A comment on *Warrior* can carry
associations to **one or many** of Warrior's sub tag values (e.g. *Boltyn* and *Olympia*), and:

- **Unfiltered** — the *Warrior* page shows *all* comments, associated or not.
- **Filtered by sub tag value** — selecting *Boltyn* (on the Warrior page, or landing on the
  Boltyn page) shows only Warrior comments associated with *Boltyn*.

### Schema

One new enum member and a **many-to-many join** — a comment can be tagged with several sub tag
values, like multiple hashtags on a post. Unlike the polymorphic target, this side can be real FKs
because it's always a `SubTagValue`:

```prisma
enum CommentableType {
  LISTING
  BID
  ENTITY
  BULK
  SUPPORTED_TAG_VALUE
}

model CommentSubTagValue {
  createdAt DateTime @default(now())

  comment       Comment     @relation(fields: [commentId], references: [id], onDelete: Cascade)
  commentId     String
  subTagValue   SubTagValue @relation(fields: [subTagValueId], references: [id], onDelete: Cascade)
  subTagValueId String

  @@id([commentId, subTagValueId])
  @@index([subTagValueId])
}
```

`Comment` gains `subTagValues CommentSubTagValue[]` (only ever populated when
`commentableType == SUPPORTED_TAG_VALUE`). Deleting a sub tag value cascades **only the join
rows** — the comments themselves survive and fall back into the unfiltered pool, which is the
whole point of keeping the association out of the `Comment` row.

### Behavior

- **`POST /comment`** accepts optional `subTagValueIds: string[]`, rejected unless
  `commentableType === 'SUPPORTED_TAG_VALUE'`. Validation: every id must belong to the supported
  tag value being commented on (`subTagValue.supportedTagValueId === commentableId`).
- **`GET /comments`** accepts optional `subTagValueId` filter alongside the existing
  `commentableType`/`commentableId` params. Omitted → full conversation; present → only comments
  with a join row for that sub tag value (`subTagValues: { some: { subTagValueId } }`). Filtering
  applies to top-level comments; replies are fetched per-thread via `parentId` as today.
- **Replies inherit** the root comment's associations server-side (join rows copied from the
  root), so whole threads stay coherent inside a filtered view.
- `validateCommentableExists` gains one case: `SUPPORTED_TAG_VALUE` →
  `prisma.supportedTagValue.findUnique({ where: { id } })` — existence only; catalog rows have no
  `status`, so there is no "inactive" guard.

Everything else — threading, pagination, rate limiting, author-only edit/delete, `status`
moderation — is inherited unchanged.

## Decisions & gotchas

- **No `SUB_TAG_VALUE` commentable type.** Earlier drafts gave sub tag values their own comment
  pool; the chosen model is a single conversation per supported tag value with the sub tag value as
  a filter — one pool, no unioning across surfaces, and the sub tag value page is just a pre-filtered
  view of its parent's conversation.
- **Orphaned comments on target deletion:** because the polymorphic target has no FK, deleting a
  listing/bid today leaves its comments as dead rows (queries just never hit them). Same for
  supported tag values — sweep in the delete handler
  (`deleteMany({ commentableType, commentableId })`); tag-page deletions are rare, admin-driven
  events, so inline cleanup is fine.
- **Rate limit is global per account** (50/day across all surfaces), not per-surface; tag pages
  share the pool intentionally.
- **Reads are public, writes authenticated** — tag pages are public catalog surfaces like entities;
  same posture as `ENTITY` comments.
