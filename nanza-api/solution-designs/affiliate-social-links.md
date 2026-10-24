---
tags: [nanza-api, solution-design, plan, affiliates, profile, groups]
---

# Affiliates — Profile Banners & Social Links

> **Status: IMPLEMENTED across all three repos (uncommitted), 2026-08-15 — pending Skylar's QA.**
> API shipped first (types 0.1.91→0.1.93 published by Skylar), then:
> - **nanza-admin**: `/social-link-types` catalog page (Conditions shell + the supported-tag-value
>   single-multipart save — logo rides create/update, no "save first" gate; Retire wording +
>   Active/Retired filter; nav/sidebar/create-menu wired).
> - **nanza-mobile**: `SocialLinksEditor` (one URL field per catalog service — one-per-type is the
>   form's shape) in EditProfile (affiliate-gated, own Update action) and the group form (rides
>   the save as a second sequential PUT); `SocialLinksRow` renders on UserProfileScreen (under
>   bio), the own-profile header, and GroupDetail (under description); affiliate banner picker in
>   EditProfile (avatar flow at 1200×600) and the banner renders on UserProfileScreen with the
>   group/collection hero recipe.
> Fold into a full Affiliates design after QA.

## The feature

Affiliates are a profile type (`Profile.type = 'AFFILIATE'`) getting a richer public page:

1. **Profile banner** — already existed end to end in the API (`Profile.banner` +
   `PUT /profile/upload-image/:id?field=banner`). **Every profile gets it** (Skylar,
   2026-08-15 — not affiliate-gated; only social links are).
2. **Social links** — an admin-curated catalog of services (logo + name), attached as **one URL
   per service** to a profile or to a group. Group links are managed by the moderator.
   Profile links were affiliate-only at release; **opened to every profile type 2026-08-17**
   (`validateAffiliateProfileOwner` → `validateProfileOwner`, the type check dropped; the
   mobile edit/render gates removed the same day) — nothing here is affiliate-exclusive
   anymore.

## Model (shipped)

- **`SocialLinkType`** — the catalog row: `name` (unique) + `displayName` + `logo` (uploaded,
  S3 key) + `index` (admin ordering) + `status`. Soft-deleted so retiring a service never rips
  existing links off pages mid-render.
- **`SocialLink`** — `url` + `socialLinkTypeId` + exactly one of `profileId` / `groupId`
  (nullable FK pair, not a polymorphic string pair — both owners are first-class relations).
  One-per-type is structural: `@@unique([profileId, socialLinkTypeId])` and the group twin.
- Migration is Skylar's (schema has no migration file from this work). Types package regenerated
  and bumped to 0.1.91.

## Endpoints (shipped)

- `GET /social-link-types` — public (pickers + render); `?status=` for admin views, ACTIVE
  default. `POST/PUT/DELETE /social-link-type(/:id)` — admin-gated (`validateRole`), multipart
  logo on the brand-logo pattern, DELETE is a status flip.
- `PUT /profile/:profileId/social-links` and `PUT /group/:groupId/social-links` — **replace-set**
  PUTs: the edit screens save the whole list, so the swap is the operation (add/edit/remove all
  fall out of one transactional deleteMany + createMany). Validation: `validateProfileOwner`
  for profiles (owner-only; any type since 2026-08-17), `validateGroupModerator` for groups,
  http(s) URLs, no duplicate types, types ACTIVE.
- **Reads need nothing new**: profile and group GETs already take
  `?include=socialLinks.socialLinkType` through `generateIncludes`.

## Next slices

- **nanza-admin**: catalog CRUD screen (name, logo upload, index, retire) — the query-lists
  admin pattern.
- **nanza-mobile**: banner upload in EditProfile (affiliates), social-links editors in
  EditProfile (affiliate-gated) and the group edit view (moderator-gated), and the render row
  (logo chips) on UserProfileScreen + GroupDetailScreen.
- ~~Decide the render treatment for a DELETED type's surviving links~~ **Decided (2026-08-15):
  soft delete under a plain "Delete" label.** A deleted type leaves the pickers and can't take
  NEW links, but existing links keep working indefinitely — including across their owner's
  future saves, via per-owner grandfathering in `validateSocialLinksPayload` (without it, the
  replace-set save would reject anyone holding a deleted-type link). One consequence: the mobile
  editor lists ACTIVE types only, so a grandfathered link isn't visible there to remove — an
  acceptable admin-side concern for now.
