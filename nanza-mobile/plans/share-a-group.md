# Share a Group — cross-repo plan (nanza-api · nanza-mobile · nanza-web-app)

Add a shareable **Group** to the existing shareability system: a new `G` reference type, an OG
share-image + link-preview meta for groups, an in-app share screen/card, and a share entry point
(ellipsis) on the group detail screen. Mirrors how Listing/Bid/Bulk/Collection/Product already work.

## New reference type: `G` = Group
Reference codes are 6 chars: **5 digits (2–9) + 1 type letter at a random position** (uniqueness
enforced by a `@unique` column per model). Existing: S=Listing, B=Bid, C=List, P=Product,
K=BulKlisting. Adding **G=Group**.

**Example valid G codes** (for backfilling existing groups by hand — pick any format-valid code):
```
85872G   22338G   G55672   G62556   694G96
```

---

## ⚠️ Blocking prerequisites (user-owned)
1. **Prisma migration** — `Group` has **no `referenceCode` field**. Needs `referenceCode String? @unique`
   added + migration. **User runs migrations** (per standing guidance) — I'll author the schema change
   but not run `prisma migrate`. Existing groups get backfilled with codes (user, using the examples above).
2. **`@oakplatforms/types` bump** — `GroupDto` has no `referenceCode`/`memberCount`-surfaced fields for
   the card until the package regenerates from the new schema. No custom DTOs — wait for the bump.

Frontend card/screen work can be built against the DTO shape but is **blocked from shipping** until
the migration + types bump land. Backend resolver/og/meta can be built immediately.

---

## 1. nanza-api

### 1a. Reference code plumbing
- `src/utils/referenceCodeGenerator.ts` — add `'G'` to `VALID_TYPES`; update the comment.
- `src/validation/referenceCode.ts` — add `'G'` to `VALID_TYPES`.
- `src/services/referenceResolver.ts` — add `'Group'` to `ReferenceRecordType`; add a `G` case that
  `prisma.group.findUnique({ where: { referenceCode }, include: … })`. Includes: `moderator.profile`
  (for handle), `members` or rely on the runtime `memberCount`. Enrich `memberCount` the same way the
  group detail endpoint does.
- `src/routers/universal.ts` — add `G → Group` to both endpoint switches + the prefix comments.
- `src/routers/group.ts` — **POST /group**: generate a `G` referenceCode via
  `generateReferenceCodeWithRetry` on create (mirrors listing/bid/bulk create).

### 1b. OG image (group share card)
Style ≈ the **collection** card (full-bleed image + bottom shadow + nanza logo) but single-image and
with text, per the brief:
- Background = the group's **banner** image (NOT thumbnail), full-bleed.
- A **drop shadow** along the bottom (reuse the collection card's gradient band).
- **Group name** on the shadow (sized like the entity-card display name / the title scale we use on
  listings), **description** below it, and the **nanza logo** bottom-left (collection-card position).
- Files: `src/services/og/fields.ts` (`GroupCardData` + `mapGroupCard` + `OG_GROUP_INCLUDES`),
  `src/services/og/templates/group.ts` (new), `src/services/og/index.ts`
  (`OG_INCLUDES_BY_TYPE.Group`, `renderGroupOg`, dispatch). Hero fetch = the banner key.

### 1c. Link-preview meta
- `src/services/og/meta.ts` — `META_INCLUDES_BY_TYPE.Group` + a `Group` case in `buildOgMeta`:
  title = group displayName (+ maybe member count), description = group description, image =
  `/reference/{code}/og.png` (add `Group` to `TYPES_WITH_IMAGE`).

## 2. nanza-mobile

### 2a. Reference plumbing
- `src/utils/referenceCode.ts` — add `'G'` to `ReferenceType` + `VALID_TYPES`.
- `src/hooks/useFetchByReference.ts` — add `| { type: 'Group'; data: GroupDto }`.
- `src/services/api/Reference.ts` — `buildIncludes`: append group includes for `G`
  (`moderator.profile`, members/memberCount).

### 2b. Share card + screen
- `src/components/share/ShareGroupCard.tsx` (new) — banner image (ShareImageWell or a banner variant),
  group **name**, **description**, **member count** (ShareMetrics), and a big colored **Join** button
  (Button `variant="action"`, like the Buy/Make-offer buttons). Auth-gated join via a new
  `onJoinGroup` in `useShareActions` (request membership; auth wall first like onBuy).
- `src/components/share/ShareDetailView.tsx` — add `case 'Group' → <ShareGroupCard />`.
- `src/hooks/useShareActions.tsx` — add `onJoinGroup(group)` (auth wall → POST membership request,
  mirroring the group detail request mutation).

### 2c. Group detail ellipsis (share entry point)
- `src/screens/groups/GroupDetailScreen/index.tsx` — add an ellipsis/more button that opens
  `ActionModal` (icon + label + description rows, the existing pattern). Rows by role:
  - **Non-moderator:** `Leave group`, `Share group`.
  - **Moderator:** `Edit group`, `Share group`, `Manage members`.
  - `Share group` → `guardedShare(buildShareUrl(group.referenceCode))` (RN share sheet) — same as the
    other share actions. (`isModerator = group.moderatorId === accountId`, already computed.)

## 3. nanza-web-app (reference landing)
- `src/app/SmartRouter.tsx` — `isReferenceCode` only allows `S/B/C`; **broaden to S/B/C/P/K/G** (it's
  currently also missing P and K — pre-existing gap to fix while here).
- `src/app/ShareDetail/index.tsx` — add a `Group` case → `<GroupDetailCard />` (new): banner, name,
  description, member count, Join CTA (web equivalent of the mobile card).
- `src/app/UniversalDetail/index.tsx` — add `Group` case if the username/refcode route should resolve
  groups too.

---

## Open questions to confirm before building frontend cards
1. Join from the share card/web landing for a **private** group — request-to-join vs blocked? (Mobile
   already has request semantics; reuse those.)
2. Member count source on the share card — the runtime `memberCount` enrich, confirmed present on the
   resolved record.

## Verification
- API: local OG preview harness (satori → resvg PNG) for the group card, same as the style-parity pass.
- Mobile/web: build the card against the bumped GroupDto; manual deep-link test once codes are backfilled.
- tsc + lint clean per repo. Sequence: API resolver/og/meta → migration + types bump (user) → mobile
  card/screen/ellipsis → web landing.
