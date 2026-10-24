---
tags: [nanza-mobile, plan, design-system, typography]
status: IMPLEMENTED (2026-08-17) — pending on-device QA; fold into design-system.md + post-attachments.md, then delete
---

# Figma 655-83703 Parity (PostDetail · Seller) — Plan

Bring the app 1:1 with Figma node `655-83703`: glass icon colours, post + entity-card
text, and the reply input. Fold into [[design-system]] and [[post-attachments]] when it lands.

## Source of truth caveat

Per the `figma-sync` skill: **code is canonical, Figma edits are proposals.** Two colour
tokens in this frame disagree with `dark.json`. They are treated here as proposals the user
has explicitly accepted ("match my figma design"), not as automatic truth.

Node `655-83703` binds the plugin-synced `theme.colors.*` variables and `Nanza/<slot>` text
styles, so it is directly diffable. Node `631-60563` (the earlier home frame) binds an
**older collection** — different `border`, `secondaryText`, `inactive`, and a post-body
tracking of 0.22 rather than 0.25. Where they disagree, 655 wins: it is the newer sync and
carries named styles rather than local overrides.

## The diff

### Colour tokens (`dark.json` / `light.json`)

| token | app now | Figma 655 | verdict |
|---|---|---|---|
| `glass` | `rgba(18,18,18,0.61)` | `#1212129c` | same value, hex-alpha notation — no change |
| `border` | `#2F333C` (neutral.800) | `#2f333c` | match |
| `text` | `#F0EBE6` (secondary.100) | `#f0ebe6` | match |
| `secondaryText` | `#BDC9D3` (neutral.200) | `#bdc9d3` | match |
| `background` | `#010101` (neutral.950) | `#010101` | match |
| **`textSecondary`** | `#FFFFFF` (secondary.50) | `#f8f6f4` | **re-point** |
| **`inactive`** | `#2F333C` (neutral.800) | `#576076` | **re-point** |

### Typography (`typography.ts`)

| slot | app now | Figma 655 named style | change |
|---|---|---|---|
| `postBody` | Balto 16/19, +0.22 | `Nanza/postBody` Balto 16/19, **+0.25** | tracking |
| `meta` | Roobert Regular 15/16, +0.11 | `Nanza/meta` **Balto Book** 15/16, +0.11 | family |
| `bodyInput` | Balto 14, no lh, +0.11 | `Nanza/bodyInput` Balto **16**/16, **+0.25** | size, tracking (NOT lh — see conflicts) |
| `title` | Roobert Bold 16/16, +0.11 | `Nanza/title` Roobert Bold 16/16, +0.11 | match |

Figma names `postBody`'s family "Balto **Trial**" while `meta`/`bodyInput` say "Balto
**Mobile**". The installed face is Balto Mobile (`BaltoMobile-Book`); the Trial reference is
a leftover in the Figma style, same visual face. No code impact — flagged for the designer.

### Glass icon colour

All three header icons (chevron-down 24, heart 15, share 16) render `#F8F6F4`, which this
frame binds to `theme.colors.textSecondary`. The app paints them `theme.colors.secondaryText`
(`#BDC9D3`) via `Header/index.tsx`. Once `textSecondary` is re-pointed to `#f8f6f4`, the fix
is a token swap at the three call sites — it does **not** become a per-icon colour fork
(see [[../../\_shared/INDEX|house rule]]: one colour rule for all darkCircle icons).

### Entity card row (`posts.ts`)

| element | app now | Figma 655 | change |
|---|---|---|---|
| `entityRow` radius | `radius['12']` = 12 | 20 | → `radius.lg` |
| `entityRowName` | `typography.bold` (SemiBold 14) + `overlayText` | Roobert SemiBold **15**/16, `text` | → `searchRowTitle` (exact match, already exists; renamed `itemTitle` 2026-09-15) |
| `entityRowMeta` | `searchRowMeta` (Roobert Medium 12/14, +0.32; slot removed 2026-09-15, unused) | **Balto Book 14**/16, +0.11, `secondaryText` | new/changed slot |
| `entityRowIdentity` gap | 4 flat | **6** outer, 4 inner | split the gap |
| price pill text | `pricePillAmount` (SemiBold 12/12, +0.11, `secondaryText`) | Roobert **Medium 13**/14, **−0.11**, `textSecondary` | changed slot |
| thumb / paddings / 16pt gap | 89/358 ≈ 24.86% | 86.67/349 ≈ 24.83% | already correct |

### Reply input (`postComposer.ts`)

`pillText` spreads `typography.body` (14/22). Figma binds `Nanza/bodyInput` (16/16, +0.25,
`secondaryText`). Change the spread to `bodyInput`; the colour is already right.

## Decisions (user, 2026-08-17)

1. **Glass pill stays 44pt**, with the existing `getControlHitSlop` 48pt touch box. Figma's 52
   is stale — it predates the 44pt control consolidation. Update Figma, not the code.
2. **`inactive` re-points app-wide** to `#576076`. 2 files edited, 29 files / 53 usages
   affected: 16 `placeholderTextColor` (currently near-invisible at `#2F333C` on `#010101` —
   this is a fix), 19 text/icon `color`, 11 `borderColor`, 2 `backgroundColor`.
3. **`postBody` tracking is 0.25** — node 655's named `Nanza/postBody` style supersedes the
   0.22 inherited from node 631. Everything else likewise takes Figma's value.

## Conflicts — do NOT silently apply

1. **Glass pill height.** Figma footer pill is **52pt**; `glass.ts` deliberately moved to the
   shared **44pt** control token (comment: "was 52 = hit + 8"). The Figma frame predates that
   consolidation. Changing it moves *every* glass control in the app.
2. **`bodyInput` lineHeight.** Figma says 16. The slot deliberately has **no** lineHeight —
   its comment records that a line box on a `TextInput` broke vertical centring differently
   per platform. Figma line-height ≠ RN `TextInput` lineHeight. Keep it absent.
3. **`inactive` re-point blast radius.** 53 usages across 20+ files, many of them muted
   control outlines (radio/toggle unchecked) where `#576076` is much lighter than `#2F333C`.
   `posts.ts:timestamp` currently works around exactly this by using `textMuted`; the
   re-point would make `inactive` correct again and let that workaround retire.

## Sequencing

1. Tokens (`dark.json`, `light.json`) — the two re-points.
2. Typography slots.
3. Consumers: `posts.ts`, `postComposer.ts`, `Header/index.tsx`.
4. Retire the `timestamp` → `textMuted` workaround if the re-point lands.
