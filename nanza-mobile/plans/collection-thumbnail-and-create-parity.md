# Plan: Thumbnail + Banner on Collection & Group, both in the inline-row form style

> **Direction reversed from v1.** Earlier this plan re-skinned Create Collection to look
> like Create Group. The user now prefers the **Create Collection inline-row style** (same
> as Create/Edit Bid) and wants **Create Group rebuilt to match Collection**. Both forms get
> a **Thumbnail** row and a **Banner** row (label + "Add thumbnail" / "Add banner").

## Goal
- **Create Collection**: keep its current inline bordered-row layout. Replace the single
  "Photo" row (which uploaded to `logo`) with **two** image rows — **Thumbnail** and **Banner** —
  wired to the `thumbnail` and `banner` fields. Drop `logo` from the UI.
- **Create Group**: rebuild in the same inline bordered-row style as Collection. Rows:
  **Title**, **Thumbnail**, **Banner**, **Description** (taller text-area-style row).
  No visibility selector (groups are always private → keep hardcoded `privacy: 'PRIVATE'`).

Backend `thumbnail` on List is already DONE & verified (see "Backend status").

---

## Backend status (already done — Part A of v1)
- `List.thumbnail String?` added to `prisma/schema.prisma`; list upload/delete routes + OpenAPI
  enums accept `'thumbnail'`; `prisma generate` + `generate:types` run; API tsc/lint clean.
- `Group` already has `banner String?` and `thumbnail String` (required) in schema; group
  create/update routes already accept `banner`+`thumbnail` as multipart in the request body
  (`uploadConfig.fields([{banner},{thumbnail}])`). **No further backend work for Group.**
- **Blocker (unchanged):** nanza-mobile uses PUBLISHED `@oakplatforms/types@0.1.52`; the new
  `List.thumbnail` field only exists in the API repo's regenerated types until the user
  publishes a new version and bumps it in mobile. **Frontend `thumbnail` typing is blocked
  on that publish.** Group is NOT blocked (its types already have thumbnail/banner).

---

## Part B1 — List service: widen `field` unions
`src/services/api/List.ts`: `uploadImage`/`deleteImage` `field?: 'banner' | 'logo' | 'thumbnail'`.
Keep `'logo'` in the union (backend still supports it; we just stop using it in the UI).
- **Confidence: 0.97** — additive.

> **Pre-existing bug to verify:** `deleteImage` calls `method: 'PUT'` against `/list/delete-image/:id`,
> but the backend route is `DELETE`. The current logo delete may be silently failing. When wiring
> the new rows, verify the method matches the route (likely change service to `method: 'DELETE'`).
> Confirm with the running app before changing — flag to user. **Confidence: 0.6 → verify first.**

## Part B2 — Create Collection: Photo row → Thumbnail + Banner rows
`src/components/lists/CreateCollection/index.tsx`:
- Replace `selectedImage`/`imageRemoved` (logo) state with two pairs:
  `thumbnailUri`/`thumbnailRemoved` and `bannerUri`/`bannerRemoved`.
- Init from `list.thumbnail` and `list.banner` in the `useEffect` (drop `list.logo`).
- Replace the single Photo row JSX with **two** inline rows, each identical in structure to
  the current Photo row but:
  - Thumbnail row: label "Thumbnail", button text "Add thumbnail", resize 500×500,
    `uploadImage(id, fd, 'thumbnail')` / `deleteImage(id, 'thumbnail', accountId)`.
  - Banner row: label "Banner", button text "Add banner", resize 1200×600,
    `uploadImage(id, fd, 'banner')` / `deleteImage(id, 'banner', accountId)`.
- Generalize `uploadLogoForList`/`deleteLogoForList` → `uploadListImage(field, uri)` /
  `deleteListImage(field)`. Create flow uploads each provided image after `create()`; edit flow
  mirrors current logic per field (delete-then-upload on change, delete on removal).
- Preserve the `CreateCollectionHandle` ref (deleteNow/requestShare/canDelete/canShare) and
  share/delete logic **exactly**. Keep Title + Visibility rows as-is.
- **Confidence: 0.85** — contained change; main risk is the per-field edit logic + the
  delete-method bug above.

## Part B3 — Create Group: rebuild in inline-row style to match Collection
`src/components/groups/CreateGroup/index.tsx`:
- **Keep the data flow unchanged**: still builds `FormData` with `displayName`, `description`,
  `privacy: 'PRIVATE'`, `thumbnail` blob, `banner` blob, (create-only) `brandId`, and calls
  `groupService.create/update`. Group images stay in the request body (backend uses multer
  `fields`). Only the JSX/layout changes.
- Replace the current stacked media-field layout with inline bordered rows like Collection:
  1. **Title** row — first row, `listingInlineBorderedRowNoTopBorder`, left-aligned
     `bidInlineFieldInputLeft`, placeholder "Enter title". Maps to `displayName` (same as
     Collection's title→displayName). Re-label state `name`→title semantics (no functional change).
  2. **Thumbnail** row — same image-row pattern as Collection (preview + remove, or
     "Add thumbnail" button). Required on create (keep existing validation).
  3. **Banner** row — same pattern, "Add banner". Optional.
  4. **Description** row — inline bordered row using a **taller text-area input**
     (`multiline`, grows, placeholder "Enter description"). See B4 for the style.
  - No visibility row.
- Keep `KeyboardAvoidingView` + `ScrollView` wrappers.
- Submit button stays (Create group / Save changes); keep `canSubmit` logic (name + thumbnail
  on create).
- **Confidence: 0.82** — largest change. The group create/edit screen wrapper
  (`CreateGroupScreen`) is untouched. No navigation changes.

## Part B4 — Styles
- Reuse existing inline classes from `src/styles/components/inputs.ts`:
  `listingInlineBorderedRow`, `listingInlineBorderedRowNoTopBorder`,
  `listingInlineBorderedRowTopAligned`, `listingInlineRowLabelColumn`, `listingInlineRowLabel`,
  `listingInlineCameraButton`, `listingInlineCameraButtonText`, `listingInlinePhotoPreview`,
  `listingInlinePhotoPreviewImage`, `listingInlinePhotoRemoveButton`, `bidInlineFieldInputLeft`.
- **New class needed**: a left-anchored multiline input for Description. Existing
  `listInlineFieldInputMultiline` is right-aligned; add
  `bidInlineFieldInputLeftMultiline` = `bidInlineFieldInputLeft` + `{ minHeight: 60,
  textAlignVertical: 'top' }` (all theme tokens, in the inputs bucket). The Description row
  uses `listingInlineBorderedRowTopAligned` so the label sits at the top.
- Banner image preview is wide (1200×600 aspect). The current `listingInlinePhotoPreview` is a
  fixed 195-square. Decide during impl: reuse the square preview for both, OR add a
  `listingInlineBannerPreview`/`listingInlineBannerPreviewImage` (wider, 16:9-ish) for the banner
  row. Leaning toward a banner-specific preview class so the banner reads as a banner.
  **Confidence: 0.8** — additive style classes, no rule violation either way.
- No inline styles, no hardcoded colors/sizes (P0 #1–#5).

---

## P0 Compliance
- [ ] All styles in buckets; no inline objects.
- [ ] Theme tokens only; no hex/named colors/sizes.
- [ ] No custom DTOs — `List.thumbnail` from regenerated `@oakplatforms/types` (blocked on publish);
      Group thumbnail/banner already in types.
- [ ] No navigation changes; screen wrappers untouched.
- [ ] API edits only in `src/services/api/List.ts` (Group service unchanged).
- [ ] tsc + lint clean before done. (List `thumbnail` typing blocked until types publish.)
- [ ] No DB migration run by agent.

## File touch count (frontend)
1. `src/services/api/List.ts`
2. `src/components/lists/CreateCollection/index.tsx`
3. `src/components/groups/CreateGroup/index.tsx`
4. `src/styles/components/inputs.ts` (new multiline + maybe banner-preview classes)
= 4 files, within the surgical limit.

## Sequencing / blockers
- **Group (B3)** can proceed now — not blocked on the types publish.
- **Collection (B1/B2)** needs `List.thumbnail` in the mobile types → blocked until the user
  publishes & bumps `@oakplatforms/types`.
- User is handling the publish and will signal when ready.

## Confirmed decisions
- Both Collection AND Group get Thumbnail + Banner rows (labels + "Add thumbnail"/"Add banner").
- Collection drops the `logo` UI; its two rows map to `thumbnail` + `banner`.
- Group rebuilt to the Collection inline-row style: Title, Thumbnail, Banner, taller Description;
  no visibility (always private).
