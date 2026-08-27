---
tags: [nanza-api, solution-design, plan, posts, connections]
---

# Open Home Posting & Friend-Scoped Visibility — Plan

> **Status: PLAN, 2026-08-14 (Skylar, voice session).** Api half of
> [[../nanza-mobile/solution-designs/open-posting-and-nav-swap|the mobile plan]] (nav swap
> + ungated composer entry). Revises [[posts|Posts]]' "BRAND posts require
> `profile.type === 'AFFILIATE'`" — the first profile-types gate — into a read-side
> visibility rule.

## The ask

1. **Everyone can post to the homepage (BRAND) feed** — the affiliate create-gate goes
   away. Same Post model, same composer, same endpoint.
2. **Visibility is scoped at read time**: an **affiliate's** brand posts reach everyone
   (signed-in or guest); a **basic user's** brand posts reach only **themselves and their
   accepted connections (friends)**. You always see your own posts, in normal feed order,
   interleaved with the affiliate/friend posts.

## Today's wiring (mapped 2026-08-14)

- **Feed**: `GET /posts?postableType=BRAND&postableId=<id>` (post.ts:194-259) — plain home
  where `{postableType, postableId, parentId: null}`, `createdAt desc`, offset-paginated
  (paginatePrisma, parallel count → `total`), viewer resolved from JWT only (optional;
  guests read too). No read gate on BRAND (GROUP 403s non-members).
- **Gate**: `validateBrandPostPermission` (validation/post.ts:177-189) throws for
  non-AFFILIATE on BRAND; enforced at create (post.ts:447), edit (:645), and
  primary-swap (:849). Replies inherit the parent's home, so they hit the same gate.
- **Friends**: `Connection` (schema:1664) — symmetric mutual-consent request,
  `status: PENDING|ACCEPTED|BLOCKED`, `@@unique([initiatorId, recipientId])`, indexes on
  both sides. The pairwise primitive is `findConnectionBetween` /
  `assertConnectionAcceptedBetween`; **no batch "friend ids for X" helper exists yet**.
- **Post has no audience column** — the home IS the audience (BRAND = world, GROUP =
  members-only, enforced imperatively). `Profile.type BASIC|AFFILIATE` (admin-set only).

## Implementation record (2026-08-14, uncommitted)

Built same-session, exactly per the design below. Specifics:

- `services/connection.ts`: `findAcceptedConnectionAccountIds(accountId)` — one findMany
  (`status ACCEPTED`, bidirectional OR), mapped to the other party's id.
- `validation/post.ts`: `validateBrandPostPermission` DELETED (three router call sites
  removed — create, edit, primary-swap; `validateGroupPostPermission` stays at all three).
  New `brandVisibilityArms` (private) + `brandPostVisibilityFilter` (exported);
  `taggedTagValuePostsFilter` now takes `(supportedTagValueId, viewerAccountId,
  friendAccountIds)` and ANDs `OR: [not-BRAND, ...visibility arms]` onto its shape.
- `routers/post.ts` GET /posts: friend ids fetched once when the read can surface BRAND
  posts (`BRAND && !parentId`, or any SUPPORTED_TAG_VALUE read — the union arm runs on
  thread reads too); the brand-feed filter applies to ROOT reads only (a visible root
  brings its whole thread). `paginatePrisma`'s parallel count shares the where, so
  `total` (mobile's band-overflow/See-all driver) stays consistent for free.
- Verified: `tsc --noEmit` clean, eslint clean on touched files.

## Second pass — audiences & hard privacy (2026-08-14 PM, voice + chat; implemented)

- **Audience = the home. NO new modeling.** A multi-audience design (Post.isPublic +
  GroupPost, the listing pattern) was built and **walked back the same hour** (Skylar:
  "we shouldn't need as much complexity — one pick, public or a group"): the composer's
  audience dropdown simply switches the post's `postableType/postableId` — Public →
  BRAND home, a group → an ordinary GROUP-homed post. Schema untouched, **no migration**.
- **Hard privacy CONFIRMED and implemented** (`isRootPostViewable` in validation/post.ts —
  the one gate, called with the ROOT id):
  - `GET /post/:id` — subsumes the old GROUP member 404; BRAND posts by basic authors
    404 for signed-out/non-friend viewers.
  - **Thread reads** — `GET /posts` with `parentId` now scopes by parent alone (the
    parent pins the thread) and vets the parent through the same gate first, so private
    threads' replies can't be probed through another surface's params.
  - **Share codes** — minting refuses basic-author BRAND posts ("visible to friends only
    cannot be shared by link"); the public T-code resolver double-checks and resolves
    such codes to null (covers pre-rule mints and affiliate demotions).
- Verified: tsc + eslint clean.

## Design

**No schema change.** Visibility derives from live state — the author's *current*
`profile.type` and the viewer's *current* accepted connections — the same posture as the
mutable-permission re-checks and the private-collection tombstones: demote an affiliate or
unfriend someone and their old posts retreat on the next read.

### Create/edit: drop the gate

`validateBrandPostPermission` stops throwing for BASIC profiles (delete the check or the
whole helper — audit its three call sites together). Root posts, replies, edits, and the
primary swap all open up; ownership checks are untouched.

### Read: one visibility builder, in SQL

New builder in `validation/post.ts` (beside `taggedTagValuePostsFilter`, the same "one
home for the rule" posture) — e.g. `brandPostVisibilityFilter(viewerAccountId,
friendAccountIds)`:

```ts
OR: [
  { account: { profile: { type: 'AFFILIATE' } } },   // affiliates reach everyone
  ...(viewer ? [{ accountId: { in: [viewer, ...friendAccountIds] } }] : []),
]
```

- **Friend ids batch-fetched once per request** (new `connection` service helper —
  `findAcceptedConnectionAccountIds(accountId)`: one `findMany` on
  `status ACCEPTED, OR:[{initiatorId},{recipientId}]`, mapped to the other party). The
  per-pair `findConnectionBetween` is the wrong shape for a feed.
- Applied in the WHERE, not post-fetch — offset pages and the parallel `count` (mobile's
  `total` drives band overflow/See-all) stay correct for free.
- Guests get the affiliate arm only. Signed-in viewers get affiliates ∪ friends ∪ self,
  ordered by `createdAt desc` as today.
- BRAND only — every other home keeps its current read rules.
- Root-level filter; replies render inside their thread (a visible root brings its whole
  thread, whoever replied — replies are conversation, not feed placement).

### Tag pages must not leak

A basic user's brand post can carry tags, and the tag-page union's *tagged-here* arm would
surface it to strangers. `taggedTagValuePostsFilter` (already the single home for the
GROUP no-leak rule) grows the same visibility scoping for BRAND-home posts by
non-affiliates — the arm becomes viewer-aware, taking the same
`(viewer, friendAccountIds)` inputs.

### Detail reads & share codes (recommend: hard privacy)

`GET /post/:id` has no BRAND gate, and share codes serve posts publicly. Recommended: a
basic author's brand post 404s for signed-out/non-friend viewers (the GROUP posture), and
share-code resolution refuses them the way it refuses group posts. Alternative (soft
privacy): filter feeds only and leave direct links open — cheaper, but "only friends can
see it" would be false the moment a link is shared. **Decision Skylar's.**

## Rollout

1. Connection service: `findAcceptedConnectionAccountIds`.
2. Gate removal (three call sites) + `brandPostVisibilityFilter` on `GET /posts`.
3. Tag-union scoping via `taggedTagValuePostsFilter`.
4. Detail/share-code posture per Skylar's call.
5. Mobile nav swap ([[../nanza-mobile/solution-designs/open-posting-and-nav-swap|plan]]) —
   no mobile query changes; responses arrive pre-scoped.
6. Fold durable outcome into [[posts|Posts]] (visibility table) and
   [[profile-types|Profile Types]] (the create-gate section retires); retire this plan.

## Open decisions (Skylar)

1. ~~Detail-read + share-code posture~~ — **resolved 2026-08-14: hard privacy** ("when I
   post to the public, they should be locked down to only my friends"). Implemented.
2. Should a BLOCKED connection also hide the *affiliate* posts of the blocked party from
   the blocker (today the affiliate arm ignores connections entirely)?
3. Performance: the affiliate arm joins `account.profile` per row — fine at current scale;
   if the feed grows, consider denormalizing `authorIsAffiliate` onto Post at write time.
