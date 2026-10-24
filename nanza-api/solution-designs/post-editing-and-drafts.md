---
tags: [nanza-api, solution-design, plan, posts]
---

# Post Editing & the Single Draft — Plan

> **Status: IMPLEMENTED same session (2026-08-14, uncommitted) after Skylar approved all
> four recommendations by voice.** Api half of
> [[../nanza-mobile/solution-designs/post-editing-and-drafts|the mobile plan]].
> Extends [[posts|Posts]] and the carousel release's PUT semantics.

## Implementation record (2026-08-14, uncommitted; tsc+eslint clean)

- **Schema**: `DRAFT` added to `Status` (shared enum, additive). **Migration is Skylar's
  to run**: `ALTER TYPE "Status" ADD VALUE 'DRAFT';`
- **PUT full parity**: multipart (`uploadConfig.array`), galleries replace-the-set with
  children created INSIDE the transaction, and images both carried (by this post's
  `media` key) and added (media-less items pair with files; uploads land under generated
  keys BEFORE the transaction — a failed write sweeps the fresh keys, a successful one
  sweeps the dropped ones).
- **Drafts**: `POST /post` takes `draft: true` (status DRAFT, rate-limit exempt,
  replaces the previous draft after the new save commits — row deleted + images swept);
  `GET /post/draft` (registered before `/post/:id`; owner's row or null);
  `PUT /post/:id/publish` (owner+DRAFT only, re-runs the group gate, flips ACTIVE with
  a fresh `createdAt`). Draft RE-saves ride the ordinary full-parity PUT.
- **Reads exclude DRAFT**: `status: 'ACTIVE'` on every home-feed read and both author-
  mode arms (the owner's Posts tab excludes their draft too — it lives in the
  composer); `isRootPostViewable` returns owner-only for DRAFT roots (covers detail +
  thread reads); the share mint's existing ACTIVE check already refuses drafts.
- Known edge (documented): a draft with IMAGES whose audience is changed before
  publishing falls through to a normal create, which can't reference the draft row's
  media keys — that path errors. Fix later by letting create claim the requester's own
  draft keys.

## The ask

1. **Edit posts** — the listing/bid detail experience for posts: your own post offers an
   edit flow; **delete stops appearing "in a lot of places" and folds into it** (the
   ellipsis → red "Delete this post" pattern).
2. **A single persistent draft** — ONE draft per account (not multiple): add three content
   items, kill or reload the app, come back — the draft is still there, content items
   included. "Probably some additional modeling"; must survive long-term ("reference
   them long term"), so a big post is never lost to a refresh.

## What exists today (api)

- `PUT /post/:id` already replaces the set: body, contentItems (replies: one attachment),
  tags; re-stamps the primary so an edit never silently undoes a promotion. **Gaps to
  audit**: it REJECTS galleries ("Post edits cannot modify galleries yet" — accepting one
  would drop its children), and whether multipart image files ride PUT at all (create
  pairs files with IMAGE items; edit's image story is unverified).
- `enum Status { ACTIVE, DELETED, INACTIVE }` — **no DRAFT**, and the post list reads do
  NOT filter on status today (audit where DELETED posts are kept out — likely hard
  delete; confirm before anything soft lands).

## Design — editing

- Mobile owns most of it (composer in edit mode → PUT). Api work is closing the PUT gaps:
  1. **Gallery edits**: replace-the-set must include GROUP nodes' children (same
     create-side createMany-after posture, with the delete-and-recreate transaction PUT
     already uses for flat items).
  2. **Images on PUT**: accept multipart exactly as create does — files pair with the NEW
     IMAGE items in index order; existing IMAGE items referenced by their media key keep
     their upload (no re-upload on unrelated edits). Rollback sweeps only newly-uploaded
     keys.
- Delete stays as-is api-side; consolidation is a mobile-only change.

## Design — the single draft

**Recommended: a real DRAFT post row** (Skylar's "additional modeling" instinct) — the
alternative (client-only AsyncStorage) is cheaper but loses local images to OS cleanup
and can't be referenced long-term. Local autosave still complements it (see mobile plan).

- **Migration (Skylar runs, additive):** `ALTER TYPE "Status" ADD VALUE 'DRAFT';`
  (`Status` is shared by other models; the value is additive and unused elsewhere.)
- **One per account**: enforced in the write path (`findFirst status DRAFT for account`
  → the save UPDATES it), not by schema constraint (Status is shared, a partial unique
  index on posts alone is an option later).
- **Save** = `POST /post` with `draft: true` → the row lands `status: DRAFT` with its
  home, audience, tags, and content items exactly as a real create (images upload now —
  that's what makes the draft durable). Rate limit exempt for draft RE-saves (an update,
  not a new post). Subsequent saves ride `PUT /post/:id` on the draft row.
- **Publish** = `PUT /post/:id/publish` (or a flag on PUT): re-runs the create-time
  gates against current state (membership, tags, caps), flips `ACTIVE`, and **resets
  `createdAt`** so the post enters feeds at publish time, not first-save time.
- **Reads must exclude DRAFT**: every feed arm (brand, group, tag union, author mode),
  detail reads for non-owners, share codes. This is the same audit the missing status
  filter needs anyway — one pass adds `status: 'ACTIVE'` (owner's own draft fetch is the
  one exception: `GET /post/draft` or author mode with an owner-only `status=DRAFT`).

## Decisions (Skylar, 2026-08-14, voice — all four resolved)

1. Draft storage: **server DRAFT row** ✓.
2. Draft scope: **one global draft**, restored whenever the composer opens for a new
   root post; its saved home/audience/tags ride along ✓.
3. Edit: **full parity** — galleries and images on PUT ship with this ✓.
4. Dead references in a draft: **save anyway, tombstone on restore** ✓.

Mobile dismiss behavior defaults to silent auto-save (no prompt) — walkable on-device.

**Migration (Skylar runs, additive):**

```sql
ALTER TYPE "Status" ADD VALUE 'DRAFT';
```
