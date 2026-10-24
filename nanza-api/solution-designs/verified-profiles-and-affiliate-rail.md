---
tags: [nanza-api, nanza-mobile, plan, profiles, posts, home]
---

# Verified Profiles + Affiliate Rail Restore

**Status:** **built 2026-10-08 in api + mobile, uncommitted, pending QA.** Still to do: Skylar
runs migration `20261008120000_profile_type_verified` (dev), the pipeline publishes
`@oakplatforms/types@0.1.106` and mobile takes it (until then `useCanSharePost` compares as
strings on purpose), check nanza-admin's type picker offers VERIFIED, settle the ToS wording
(open question 4). Spans nanza-api (schema + read rules + types) and nanza-mobile (home rail
restore, share gate). When it lands, fold the durable parts into [[profile-types]],
[[open-posting-and-friend-visibility]] and the mobile [[../../nanza-mobile/solution-designs/whats-new|What's New]]
/ [[../../nanza-mobile/solution-designs/app-load-performance|app-load-performance]] designs, then
delete this file.

## The ask (Skylar, 2026-10-08)

1. **Bring the affiliate circles back** to the top of the mobile home page for the next release.
   They were removed 2026-09-28.
2. **A third profile type, `VERIFIED`.** A real, known community member who is *not* an affiliate.
   They never appear in the affiliate rail, but their posts reach everyone on the What's New feed,
   including signed-out visitors, who today see affiliate posts only.

## What exists today

**Profile type.** `ProfileType` is `BASIC | AFFILIATE` (`prisma/schema.prisma`), admin-assigned
only via `PUT /profile/:id/type`, which validates against the Prisma enum, so a new value works
there with no handler change. The types package has no standalone enum; the value is inline on
`Profile.type` in `@oakplatforms/types` (currently `0.1.105`).

**Where `AFFILIATE` means "public author".** Since open posting (2026-08-14) anyone can post to the
BRAND home; the affiliate distinction lives at read time. Six API spots hard-code
`type === 'AFFILIATE'` to mean *this author's brand posts are public*:

| Spot | What it does |
|---|---|
| `src/validation/post.ts` `brandVisibilityArms` | the feed filter arm guests get, and the public arm for members |
| `src/validation/post.ts` `isRootPostViewable` | detail / thread / share-code gate for one root post |
| `src/routers/post.ts` author mode (`?accountId=`) | strangers see the author's BRAND posts only if affiliate or friend |
| `src/routers/post.ts` share-code mint | only affiliate brand posts can be shared by link |
| `src/services/referenceResolver.ts` | public resolve of a shared brand post |
| `src/routers/user.ts` `GET /user/:username/page` | web profile counts BRAND posts only for affiliates |

Mobile mirrors the share rule once: `src/hooks/useCanSharePost.ts` (`profile.type === 'AFFILIATE'`).
`SessionContext.isAffiliate` exists but nothing consumes it. There is no profile-type badge or
label anywhere in the app.

**The affiliate rail.** The API half is intact: `GET /profiles?type=AFFILIATE&page=0&limit=20`
(`src/routers/profile.ts`, public, trimmed to id / username / avatar / description, ordered by
username). The mobile component `AffiliateCircleCarousel` was **deleted outright** in commit
`de3023ff` ("fix homepage issues with affiliate carousel", 2026-09-28), along with its entry in the
`useHomeReady` first-paint gate. It is recoverable verbatim from `de3023ff^`. Before that commit the
page opened: affiliate circles → Trending → Groups → What's new. Now: Trending → What's new 1 →
Groups → What's new 2 → Query Lists.

## Design

### One concept: "public author" types

Add a single constant and route every hard-coded check through it, so the next type (or a
demotion rule) is one edit:

```ts
// src/constants/profileTypes.ts
export const PUBLIC_AUTHOR_PROFILE_TYPES = ['AFFILIATE', 'VERIFIED'] as const satisfies readonly ProfileType[]
export const isPublicAuthorType = (type?: ProfileType | null) => !!type && PUBLIC_AUTHOR_PROFILE_TYPES.includes(type)
```

- `brandVisibilityArms`: `{ account: { profile: { type: { in: PUBLIC_AUTHOR_PROFILE_TYPES } } } }`.
  This alone delivers the feed ask: guests and members both get VERIFIED posts on What's New,
  on tag pages (`taggedTagValuePostsFilter`) and in item chats (`referencedPostsFilter`), since
  they share the arms.
- `isRootPostViewable`, author mode, share mint, `referenceResolver`, `user.ts` page counts:
  swap `=== 'AFFILIATE'` for `isPublicAuthorType(...)`. VERIFIED posts are then also openable
  by link and counted on the web profile, which is the consistent reading of "public author".
- Live state, as today: demote VERIFIED → BASIC and the posts retreat on the next read.

### What stays affiliate-only

- **The rail.** The component requests `?type=AFFILIATE` explicitly, so VERIFIED never shows.
  `GET /profiles` keeps `AFFILIATE` as its default but should validate `type` against the enum
  (today an unknown value falls through to Prisma and errors as a 500).
- Nothing else. Social links, banners and the composer are already open to every type.

### Schema + migration

- `enum ProfileType { BASIC AFFILIATE VERIFIED }` (no inline comments, enums block).
- Migration `20261008120000_profile_type_verified`:
  `ALTER TYPE "ProfileType" ADD VALUE 'VERIFIED';` — additive, no backfill, Skylar runs it.
- `npx prisma generate`, `npm run generate:types && npm run build:types`, bump
  `packages/types` → `0.1.106`.

### Admin

`PUT /profile/:id/type` needs no change. **nanza-admin is not in this session's workspace**: if
its profile-type control is a hard-coded two-option picker it needs `VERIFIED` added; if it reads
the enum from types it only needs the package bump. To confirm.

### Mobile

1. **Restore the rail.** `git show de3023ff^:src/components/profile/AffiliateCircleCarousel/index.tsx`
   back into place, unchanged. Render it first in `HomeScreen`'s `listHeader`, above
   `HomeTagCarousel`, hidden while switching brands as before. Update the four "circles are gone"
   comments (HomeScreen ×2, useHomeReady, HomeTagCarousel) and `whats-new.md` /
   `app-load-performance.md` placement notes.
2. **First-paint gate: put it back in `useHomeReady`** as the third request (the 2026-09-11 →
   09-28 state). Rationale: the gate exists so the above-the-fold rows commit together; a rail
   that pops in after commit would push Trending and What's New down under the reader. Cost is
   one small public read (20 trimmed rows) on cold start. (The 09-28 removal was because there
   were no affiliates yet, not performance — see open question 1 — so the original gate design
   stands.)
3. **Share gate.** `useCanSharePost`: `type === 'AFFILIATE'` → `['AFFILIATE','VERIFIED'].includes(type)`,
   mirroring the API. (`ProfileDto['type']` widens automatically once the app takes `0.1.106`.)
4. **No badge.** Not asked for; VERIFIED looks like BASIC in the UI. A checkmark on the profile
   hero is a natural follow-up if wanted.

### Copy / docs touched after landing

- Mobile `TermsOfServiceBody` line ~102 states "signed-out visitors see only affiliate content".
  Needs "affiliate and verified member content" — legal copy, Skylar to confirm wording.
- Vault: [[profile-types]] (enum + semantics table), [[open-posting-and-friend-visibility]]
  (arms), [[posts]] (stale "AFFILIATE-only BRAND creation" lines), mobile `whats-new.md` and
  `app-load-performance.md` (rail back, gate back to three), plus both `INDEX.md` summaries.
- Stale `@openapi` text on `POST /post` ("brand posts require an AFFILIATE profile") — fix while
  there.

## Open questions for Skylar

1. ~~Why was the rail removed on 2026-09-28?~~ **Answered (Skylar, 2026-10-08): there were no
   affiliates yet, so the row rendered empty. Not a bug, not cold-start time.** Restore it as it
   was, back in the `useHomeReady` gate (Mobile §2, main variant). Record this in the restored
   component's comment so the next "empty rail" isn't mistaken for a bug.
2. ~~Link sharing + web profile for VERIFIED posts~~ **Answered (Skylar, 2026-10-08): VERIFIED
   is fully public — everything an affiliate can do, sharing included. The only difference is the
   rail.** The constant-based design above is exactly that.
3. **nanza-admin** — does its type picker need the new option? (Repo not open here.) Still open.
4. **ToS wording** for the new type.

## Order of work

1. ~~nanza-api: constant → six call sites → `GET /profiles` type validation → schema + migration
   file → prisma generate → types regen + bump → typecheck.~~ Done 2026-10-08
   (`src/constants/profileTypes.ts`; stale `@openapi` text on `POST /post` and the three
   `enum: [BASIC, AFFILIATE]` listings fixed too).
2. ~~nanza-mobile: restore component + gate + `listHeader` → `useCanSharePost` → comments → tsc/lint.~~
   Done 2026-10-08 (component restored verbatim plus a "why it was gone" note; gate back to
   three requests).
3. Skylar: run migration (dev), flip a test profile to VERIFIED via admin, QA guest + member feeds.
4. Fold into solution designs, delete this plan.
