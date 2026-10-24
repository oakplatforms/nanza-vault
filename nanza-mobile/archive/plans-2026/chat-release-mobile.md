# Chat Release — Handoff (current state)

_Updated 2026-08-02. **The vault is the source of truth** — read
`documentation/solution-designs/posts-chat.md` (mobile) and
`../nanza-api/documentation/solution-designs/posts.md` (API) before changing anything.
This file is the orientation map for a fresh agent._

## State: FEATURE-COMPLETE, uncommitted in this repo

Everything below is built, verified (`tsc` baseline **99** pre-existing errors — compare counts,
don't chase; eslint clean on all touched files), and iterated live with Skylar against mocks.
The API side is committed except small same-day deltas; mobile is entirely uncommitted.

## What exists (mobile)

| Piece | Where |
|---|---|
| Post cards (ink surface, identity row, pink secondary-only `#tags`, reply/heart/trash footer) | `components/posts/PostItem.tsx` |
| Content renderer — one renderer for cards AND composer previews | `components/posts/PostContentItems.tsx` |
| Reference tiles: listing/bid/lot hero tile (letterboxed art, feed-title name, white price pill, lot child-image pager); entity search-row card | same file + styles in `styles/components/posts.ts` |
| Open-graph LINK card (image left, title + domain; placeholder glyph fallback) | `components/posts/LinkPreviewCard.tsx` + `services/api/LinkPreview.ts` (`GET /link-preview`) |
| EMBED player (YouTube 16:9 / TikTok portrait WebView; platform DERIVED from the code — no enum; short links resolved via redirect-follow, cached; error-153 fixed via baseUrl iframe wrapper) | `components/posts/embed.ts`, `components/posts/EmbedPlayer.tsx` |
| Full-screen post modal (slide-up; dark-circle chevron dismiss; Post/Reply pill, pink when ready; fixed-height "Say something…" 17pt `bodyLg`; settle→double-rAF keyboard focus) | `components/posts/composer/PostComposer.tsx` |
| v1 attachment rule: ONE attachment (image/capture/link/embed/card/listing-incl-lot/bid); attach circles hide once attached or while a paste field is open; white X circle pinned top-right of the preview removes | composer + `composer/draft.ts` |
| Pickers: "Add cards" (My Collection/Nanza tabs) + shop (Listings/Bids 2-col thumb grid) — search-screen toggle tabs, bottom glass search pill + X | `composer/CardPickerLayer.tsx`, `composer/ElbPickerLayer.tsx` |
| Secondary-tag chicklets (outline chips, ambient association only, FE state until create — create sends BOTH supported + secondary ids) | composer |
| Chat entry: "Add post…" ActionPill (the homepage "See all" recipe) right of every Chat heading (`sectionHeadingRow`); listing/bid sections + tag screens (300pt capped inner-scroll thread) | `components/detail/DetailChatSection`, `screens/tags/*` |
| PostDetail screen (flat root + `bodyLg`, replies as minimal-footer cards, reply arrow → modal in reply mode w/ pinned real card + "Replying to @user…" placeholder, editor auto-anchored) | `screens/posts/PostDetailScreen` (route registered in AppNavigator) |
| Likes = SavedItem `'post'` target (outline heart, optimistic count) | `contexts/SavedItemsContext`, `services/api/SavedItem` |
| Deletes cascade the whole thread (confirm warns with reply count; no tombstones) | `PostThread`/`PostDetailScreen` + API |
| In-app browser sheet for links | `components/posts/InAppBrowserSheet.tsx` |

## Conventions a fresh agent MUST keep

- **Never run migrations or commits — Skylar does both.** No `git commit`, no `prisma migrate`.
- Types flow: `@oakplatforms/types` generated in `nanza-api/packages/types`; **published = 0.1.84**
  (EMBED, GROUP, childImages, LinkPreviewDto — installed here). Regenerate + bump there; Skylar
  publishes; a fresh install clears type errors on new enum literals.
- All styling through bucket files (`styles/components/*.ts`) + typography scale slots — the
  ESLint rule blocks inline font props; ui-review skill is the checklist. `theme.spacing.hit`=44.
- Schema files stay comment-light (Skylar's preference).
- No `PATCH` routes anywhere (CRA proxy) — updates are `PUT`.
- Composer previews and posted cards share `PostContentItems` — any tile change lands in both.
- Security invariants (vault posts.md § hardening): LINK http(s)-only; EMBED exact-hostname
  allowlist + constructed player URLs; post images raster-only (SVG rejected — sharp re-encode
  is the sanitizer); `/link-preview` SSRF guard (DNS-checked, manual redirects).

## Reserved / deferred (do NOT build without Skylar's go)

- **GROUP** content items (parent+children plumbing live in schema, API rejects; renderer has a
  future-safe carousel branch). Enabling = validation + composer change, no migration.
- Brand/homepage composer (affiliate-gated), share, VIDEO, real OG title fetch is done but X/
  Instagram embeds are not.

## Open items

1. Skylar: commit mobile, run migrations (`ContentItem.parentId`, enum values GROUP+EMBED),
   deploy API dev.
2. Icon assets pending: photo-library (`placeholder` stand-in), link (`share-2`), embed (`qr`).
   Swap = one-line each in `PostComposer`/`LinkPreviewCard`/renderer.
3. Reply-mode pinned card renders a minimal-footer PostItem — if mocks change again, that's
   `PostComposer`'s reply block only.
