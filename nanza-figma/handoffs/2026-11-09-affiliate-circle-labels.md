---
tags: [nanza-figma, handoff, nanza-mobile, home, circle-thumb, typography]
status: open
date: 2026-11-09
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3064-68432&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3064:68432
frame: HomeScreenContent (the affiliate circle rail at the top)
repo: nanza-mobile
scope: styles-only
---

# Home affiliate rail — circle caption type + spacing

The captions under the home screen's affiliate circles (`AffiliateCircleCarousel` →
`CircleThumb`, the `lg` ink-ring variant), built and tested on device on a throwaway branch
(`throwaway-circle-thumb-fonts`). The values below are that build, so treat it as the reference
implementation.

> [!warning] Scope
> - **Only the `lg` caption changes**, i.e. the home affiliate rail. The `sm` glass circles (tag
>   pages' TagCarousel rails) keep `thumbLabel` 12/14 in `secondaryText`, per the tag-page frame
>   3393:168141.
> - **The ring colours are unchanged and out of scope.** Pink (`brand`, unseen Short) vs grey
>   (`quiet`, seen) is per-person data, not styling, so a mock showing pink rings next to a
>   device showing grey ones is a data difference.

## Request (verbatim)
"we need to update the typography and spacing for the circle thumbs, see this example"
(frame 3064:68432, plus a device screenshot and a mockup).

**Do not change:** the ring colours and states, circle size, the play badge, rail spacing.

## Measurements

| Part | Figma 3064:68432 | Device before | After |
|---|---|---|---|
| Rail | 8 side inset, 12 between circles, 12 below | same | unchanged |
| Circle | 88, 2.5 ring, 76 image | same | unchanged |
| Caption type | Roobert Medium **12 / 12**, +0.11, **`text`** | Medium 12 / 14, `secondaryText` | **12 / 12, `text`** |
| Circle → caption box | **8** | 8 + **6** (`badgeRoom` marginTop) = 14 | **8** |
| Circle bottom → caption glyph (measured) | ~10 (mockup) | ~17 | ~10 |

The play badge overhangs the circle's bottom by ~2.5 (it's 15 tall at y 75.5 in the 88 box), so a
caption 8 below clears it. The extra room `badgeRoom` added (2026-10-23) is no longer needed.

## How it's built
```ts
// typography.ts — the lg caption slot
thumbLabelLg: {
  fontFamily: theme.fonts.family.primaryMedium,
  fontSize: theme.fonts.size.xs,              // 12
  lineHeight: theme.fonts.lineHeight['12'],   // was lineHeight.xs (14)
  letterSpacing: 0.11,
  color: theme.colors.text,                   // was secondaryText
},
```
- `CircleThumb`: remove the `badgeRoom` prop and its `marginTop: spacing.xs` on the caption, so
  the caption sits on `circleThumb`'s `gap: sm` (8) for every variant. This also drops an inline
  style.
- `AffiliateCircleCarousel`: stop passing `badgeRoom` (its only caller).
- `cards.ts circleThumbLabelLg`: unchanged (still `...typography.thumbLabelLg` + centred); only
  its comment changes. `thumbLabelLg` / `circleThumbLabelLg` have no other consumers.

## Code map (local `throwaway-circle-thumb-fonts`; match these on your branch)
- `src/styles/components/typography.ts`: `thumbLabelLg`.
- `src/components/cards/CircleThumb/index.tsx`: `badgeRoom` prop and caption margin removed.
- `src/components/profile/AffiliateCircleCarousel/index.tsx`: `badgeRoom` removed from the call.
- `src/styles/components/cards.ts`: `circleThumbLabelLg` comment.

## QA
- Home affiliate rail: captions cream (`text`), 12/12, sitting ~10 under the circle (glyph);
  a circle with a play badge still clears its caption; long names still cap at 2 lines.
- Ring colours unchanged: pink for an unseen Short, grey once seen.
- Tag-page circle rails (TagCarousel, `sm`): unchanged, 12/14 grey, 8 gap.
- Profile hero avatar (`hero`, no caption): unchanged.

## Open questions
None.

## Prompt
```text
Implement the handoff nanza-vault/nanza-figma/handoffs/2026-11-09-affiliate-circle-labels.md
(read it in full). Figma file 7c3Zz4nDUOutQG1BsnRRAw, frame 3064:68432.

SCOPE: STYLES ONLY, home affiliate rail captions. Do not touch ring colours or states.

1. typography.ts thumbLabelLg: Medium 12 / lineHeight['12'], +0.11, color text.
2. CircleThumb: remove the badgeRoom prop and its caption marginTop (the caption sits on
   circleThumb's 8 gap); AffiliateCircleCarousel stops passing badgeRoom.
Leave thumbLabel / circleThumbLabel (the sm glass circles) as they are.

House rules (CLAUDE.md): plan first with confidence scores; font props only in typography.ts;
theme tokens only. Run /ui-review before finishing, then set this handoff to built and add a
Log line.
```

## Log
- 2026-11-09: handoff written; built and device-tested on `throwaway-circle-thumb-fonts`
