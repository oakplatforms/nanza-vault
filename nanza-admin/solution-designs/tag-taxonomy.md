---
tags: [nanza-admin, solution-design, tags, taxonomy]
---

# Tag Taxonomy (admin surface) — Nested Value & Sub-Tag-Value Editing

> **Status:** Implemented 2026-07-30 — this describes the **shipped** admin surface. Canonical data
> model and API in [[../nanza-api/solution-designs/tag-taxonomy-v2|nanza-api tag-taxonomy-v2]].
>
> [!note] **Refactored to v2 on 2026-07-31.** The dialog keeps its shape, including the third view
> — `SubTagValueDetail` became `SecondaryTagValueDetail` and the chip list stays on the
> supported-tag-value detail. What changed is the *meaning* of the chips: adding one is
> **find-or-create-then-link** (an existing Boltyn under this brand tag gets linked to a second
> class rather than duplicated, via `POST /supported-tag-value/:id/secondary-tag-value`), and
> removing one **unlinks** (`DELETE …/secondary-tag-value/:id`) rather than deletes. The API
> rejects unlinking the last class, and that message is surfaced verbatim since it tells the
> operator to delete the value outright instead. Secondary values now arrive nested through the
> join (`secondaryTagValues[].secondaryTagValue`), unwrapped on load.

## Overview

Today the brand-tag dialog (`src/app/Tags/BrandTagDialog.tsx`) treats supported values as **dumb
chips**: type a value, press Enter, chips accumulate, save diffs creates/deletes. That was right
when a value was just a string. Now a supported tag value is a *page* — banner, thumbnail, description —
and owns a one-to-many list of **sub tag values**, each of which is also a page. The chip editor can't
carry that.

## Chosen UX: nested views **inside the dialog**, with back navigation

Rather than routing to a new page (which would drop the brand-tag editing context), the existing
dialog becomes a small **view stack**:

```
View 1: Brand Tag (today's dialog)
  └─ click a value chip → View 2: Supported Tag Value detail
        banner upload · thumbnail upload · displayName · description (RichTextEditor?)
        sub tag values list (chips)
        └─ click a sub tag value → View 3: Sub Tag Value detail
              banner upload · displayName · description
```

- A `viewStack` state (`[{ view: 'brandTag' } | { view: 'value', id } | { view: 'subTagValue', id }]`)
  drives which panel renders; a back chevron in the dialog header pops the stack. Closing the
  dialog resets it.
- **Values must be persisted before they can be opened** — a freshly-typed chip has no `id` until
  save. Simplest rule: creating a value saves it immediately (the dialog already creates values
  individually via `supportedTagValueService.create`), so every chip is clickable right away. This
  moves the dialog from "diff on save" toward "save as you go" for values; the brand-tag fields
  (tag, isPrimary, index) keep the explicit Save button.
- Detail views save via `PUT /supported-tag-value/:id` and the sub-tag-value CRUD routes;
  banner/thumbnail are multipart FormData like the existing brand/entity image uploads (service
  modules in `src/services/api/`). The image field is labelled **Thumbnail** in the UI.

This "nested modal with back navigation" is a new pattern for the admin app — reuse it deliberately
(it will likely apply to other catalog editors) rather than inventing per-feature variants.

## Gotchas

- **No `PATCH` in nanza-admin.** The dev server has no `REACT_APP_API_BASE_URL`; it reaches the API
  through CRA's `proxy` field in `package.json`. That proxy forwards `GET`/`POST`/`PUT`/`DELETE`
  but drops `PATCH`, which then falls through to the static file server and 404s against
  `localhost:3000`. Every admin-facing update route is therefore a `PUT` — the update routes for
  supported tag values, sub tag values, and profile type were all switched from `PATCH` to `PUT`
  for this reason.
- **`DialogDescription` renders a `<p>`** (`Headless.Description as={Text}`), so form content
  nested in it triggers `validateDOMNesting` warnings. Use `DialogBody` (a `<div>`, already padded
  `p-4`) for dialog form content instead.

## Data layer

- `src/services/api/SupportedTagValue.ts` gained `get`, `update` (FormData); new
  `SubTagValue.ts` service (list/get/create/update/delete).
- The detail views (`SupportedTagValueDetail.tsx`, `SubTagValueDetail.tsx`) load via direct
  service calls in the component (matching the dialog's existing idiom) rather than React Query
  hooks; the Tags page refetches brand tags on dialog close since values save as you go.
- `SubTagValueDto` and `SupportedTagValuesDto` come straight from `@oakplatforms/types` 0.1.74;
  the temporary hand-rolled DTO and `thumbnail` override were removed once that version installed.

## Out of scope here

- Attaching sub tag values to entities — deferred entirely; entities keep their existing
  `EntityTag` (string value) relation for now, so there is nothing to build in the Products editor
  yet.
- Mobile tag pages (banner/description/comments rendering) belong to nanza-mobile.
