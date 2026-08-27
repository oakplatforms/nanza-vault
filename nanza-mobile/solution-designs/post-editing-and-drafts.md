---
tags: [nanza-mobile, solution-design, plan, posts]
---

# Post Editing & the Single Draft — Plan

> **Status: IMPLEMENTED same session (2026-08-14, uncommitted) after Skylar approved the
> recommendations.** Mobile half of
> [[../nanza-api/solution-designs/post-editing-and-drafts|the api plan]].
> Extends [[posts-chat|Posts Chat]] and the composer.

## Implementation record (2026-08-14, uncommitted; tsc/eslint clean on touched files)

- **Service** (`api/Post.ts`): `updateFull` (PUT, JSON or multipart — existing images by
  `media` key, new ones as files), `getDraft`, `publish`, `draft` flag on create;
  `CreatePostContentItem.media` added.
- **draft.ts**: `deserializePost` — the serializer's inverse (leading RICH_TEXT → main
  editor, rest → draft items; existing images by media key; restored images take the
  library/camera buckets by position, since `source` isn't persisted). `DraftItem`
  IMAGE now `image?` OR `media?`; serialize emits `{ media }` without a file.
- **PostComposer**: `editPost` prop — lazy-seeded tree + tags, audience dropdown and
  draft machinery disabled, pill reads "Save", submit = `updateFull` + invalidate the
  home key and `['UserProfilePosts']`. Draft flow (new roots only): `['PostDraft']`
  query restores into a PRISTINE composer only (draft home re-selects its group in the
  dropdown once groups load); the header chevron routes through `handleDismiss` — has
  content → silent save (same-home: PUT the draft row; else a fresh `draft: true`
  create, which replaces server-side), emptied → the draft row deletes. Publish: same-
  home drafts sync (PUT) then `publish`; changed audience falls through to a normal
  create and clears the stale draft. Android back (Modal onRequestClose) does NOT
  save — known gap, noted.
- **PostDetail**: owner's trash icon → the **edit pencil** (`edit-2`, the pencils'
  hair-over sizing — Skylar walked back the ellipsis/ActionModal same day: "we don't
  need an ellipsis"), opening a slide-up Modal hosting the composer in edit mode
  (brandId passed for BRAND homes only — on other homes the tag line hides but existing
  tags re-send unchanged). **Delete lives INSIDE the edit screen**: a glass trash
  circle beside Save (`onDeleteRequest` prop — the host owns the confirm + goBack),
  the only post-delete surface now.
- **Save is dirty-gated**: `composeSnapshot` (the serialized payload + tag ids)
  captured at open; the pill stays disabled until the live snapshot drifts from it.
- **Edit SNAPS open** (`animationType="none"` — Skylar: a slide reads as going to a
  second post; the instant swap reads as this post entering edit). New-post composers
  keep their slide.
- **Draft update race FIXED** (Skylar's repro: remove items, dismiss, reopen → old
  draft back): on remount the `['PostDraft']` query served the CACHED pre-edit copy
  while revalidating, and the restore ran from it. Now the dismiss-save writes its
  response straight into the cache (`setQueryData`; failures invalidate instead) and
  the restore effect waits out `isFetching` before touching state.

## Design — editing (the listing/bid pattern)

- **Entry**: on the owner's own post DETAIL, the header's trash icon becomes an
  **ellipsis** (darkCircle) opening the listing detail's ActionModal recipe:
  **"Edit post"** and destructive red **"Delete post"** (delete keeps its existing
  confirm + reply-count copy). **Every other delete affordance retires** — audit
  PostItem's footer/owner affordances so delete lives only behind the ellipsis.
- **The edit screen IS the composer**: `PostComposer` gains an `editPost` mode —
  prefilled from the post (text runs → editor + text blocks, attachments → draft items
  with their references, images as remote URIs, tags → chips; audience/home NOT
  editable — the post lives where it lives). Submit → `PUT /post/:id` (replace-the-set;
  the api plan closes the gallery/image gaps). The Post pill reads **"Save"**.
- Cache: on success invalidate the post's home `['Posts']` key + `['Replies']` and the
  profile Posts tab key.

## Design — the single draft

- **Server draft is the source of truth** (api plan: a `DRAFT`-status post row, one per
  account), **plus a local autosave layer**: the composer debounces the draft tree to
  AsyncStorage on every change (cheap, survives a crash mid-typing), and syncs to the
  server draft on dismiss ("save draft?" prompt or silent — felt out on-device).
- **Restore**: opening the composer for a NEW root post with a draft present restores it
  (local copy if fresher, else the server row — images come back as uploaded media, the
  draft's home/audience/tags ride along). Publishing clears both layers; an explicit
  discard does too.
- Replies never draft (v1).

## Open decisions (Skylar)

Shared list in the [[../nanza-api/solution-designs/post-editing-and-drafts|api plan]].
Mobile-only: dismiss behavior (silent auto-save vs "Keep draft?" prompt), and whether
the composer surfaces a visible "Draft" chip/banner when restored.
