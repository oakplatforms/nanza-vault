---
tags: [nanza-web-app, solution-design, landing]
---

# Landing opening — Solution Design

## Overview

The homepage opens on **centred copy** (headline, sub-copy, "Join the chat") standing in the
eye of an infinite **card spiral** that runs full bleed around it. Scrolling clears the copy
and drains the spiral inward, and the **3D phone grows out of the same point** as the last
cards are swallowed — the spiral turns into the phone, with nothing in between — then
sweeps right into the conversation section ("Find the people who get it").

Everything hangs off one point — the middle of the stage. The copy, the spiral's centre and
the phone's entrance are all that same point on screen, which is what lets each one hand
off to the next.

Replaced (2026-09-17) the earlier opening: centred lockup → contained video growing to full
bleed → the mark collapsing over the footage → a "Where gaming / communities live" wordmark
act → phone. That version is archived in code, not routed:
`src/sections/IntroHeroOld.tsx` with its phone anchors in `src/scene/sectionsOld.ts`. The
video file (`public/collecting-is-an-art-rc1.mp4`) and the components only the old opening
used (`LogoMask`, `SeamHeadline`, `VideoControls`, and now `GradientMark`) are still in the
repo for it.

## How it works

**One sticky stage, scrubbed by scroll.** `src/sections/IntroHero.tsx` is a 260vh section
with a 100vh sticky stage. Its ScrollTrigger maps the 160vh scrub distance to progress
0..1, and every beat is a sub-range of that progress — so the section height is the single
speed control for the whole opening.

| Beat | Progress | What happens |
|---|---|---|
| Copy out | 0 – 0.20 | Copy fades and lifts; CTA stops taking clicks once gone |
| Tighten | 0.02 – 0.55 | Clear zone closes from the copy's shape to the phone's core |
| Drain | 0.05 – 0.72 | Spiral contracts into the centre (positional — stop scrolling and survivors loop forever) |

The phone's own arrival is not on this list: it is driven by the shared scroll timeline off
the two markers below, and it deliberately begins *inside* the drain.

**The clear zone is the copy's own box.** `CardVortex`'s dissolve mask is an ellipse sized
from the copy's measured bounding box plus padding, so it follows the headline however it
wraps at a given width. Over `TIGHTEN` one interpolation closes radius, flattening and
falloff together into a small circle — the hole never changes size and shape on different
schedules. On short windows a cap squashes the ellipse so a band of cards always survives
top and bottom; it still clears the copy itself.

**The spiral closes on the phone's own silhouette.** `CORE_RADIUS` (0.069 of viewport
height) is not a taste value: it is half the phone's width at the dolly it starts from
(0.714 world units at `z = -5` through the 30° camera). The cards therefore vanish into
exactly the shape the phone grows out of.

**The phone starts centred and sweeps right.** It starts at dolly `z = -5`, opacity 0, at
`x = 0` — the centre of the viewport at any dolly — and ends at `x = 0.55, z = 0`. Its
`xDelay` of 0.34 holds the sideways travel back for the first third of the segment, so it
grows in place while cards are still draining into that point and only then sweeps out.

**Handoff markers.** Two zero-height markers inside the intro section drive the phone
timeline: `intro-end` (23.5%) and `intro-phone-in` (70%). Note these are percentages of the
*section* while the beats above are fractions of the *scrub*; progress p sits at p × 61.5%
of the section at this height. `intro-end` lands at p 0.38, well inside the drain: the
phone is already growing as the last cards arrive, which is the whole handoff.

**The rest of the page overlaps the intro's tail.** The intro carries a
`-mb-[60vh]`, which pulls everything after it up over the dead stretch at its
foot. That stretch is structural, not a choice of height: the stage is one
viewport tall and sticks to the section's bottom, so there is always exactly
100vh of scroll left once it releases. Nothing is drawn in it — the spiral is
drained by 115vh and the phone has parked by 182vh — while the conversation
copy, centred in its own 100vh section, only reached the middle of the screen at
260vh. Overlapped, "Find the people who get it" rises into the blank stage
*while* the phone is still growing and settles around 200vh, just after it
parks. The overlap is safe because everything left in the stage by then is
transparent and `pointer-events-none` (the drained vortex, the faded lockup) and
the conversation section paints over it as the later sibling at the same `z-10`.
It also brings the topic cycle forward: `cycle` crosses its threshold at the
midpoint of the `intro-phone-in` → `conversation` span, so the phone starts
flicking through topics as the chips arrive rather than 40vh later.

## Why

- **The overlap is the point.** An earlier cut had the spiral empty into a blank screen that
  the phone then faded into, which read as two events. `DRAIN` deliberately runs past the
  phone's start so the two share a stretch of scroll.
- **The same applies at the far seam.** The phone parking and the conversation copy
  arriving were two events separated by ~80vh of blank scrolling; pulling the page up over
  the intro's tail makes them one.
- **The clear zone is measured, not written down.** A hard-coded radius would only suit one
  headline at one width; the copy's box is the thing cards actually have to stay off.
- **Spiral fade-in on load** (1.2s) so cards never pop in before their artwork has decoded.
- **Ticker skips work once fully drained**, but only after drawing the drained frame —
  otherwise cards freeze mid-drain.
- The **Backdrop** is unchanged: it recedes over the first ~1.1 viewports (as the spiral
  drains) and returns around the sticky release, as the phone lands.

### Tried and removed

A logo beat sat between the spiral and the phone for one revision (2026-09-20): the mark
faded in and rose into the space the copy left, then slid left while the wordmark was
uncovered beside it, assembling the header's lockup, which then handed off to the phone. It
was cut in favour of the spiral merging directly into the phone. Worth knowing if it ever
comes back:

- The mark and letters were sized and spaced from the **header artwork's own ratios**
  (wordmark width 3.041 and gap 0.399, in units of the mark's height, measured off
  `public/nanza-logo-v2.svg`), and placed so the assembled pair's centre was the stage's
  centre. That reproduced the header's lockup at any viewport.
- The letters were **uncovered in place** with a `clip-path` inset, painted *before* the
  mark so the mark covered them — which is what made them look like they came out from
  behind it.
- Its weak point, and the reason it is worth rethinking rather than restoring verbatim: the
  phone grew from the middle of the assembled lockup, so it emerged *between* the mark and
  the letters rather than straight out of the mark.

## Known gaps

- **Mobile has not had a design pass.** Below 768px there is no 3D phone (`useImmersive`
  falls back to `StaticPhone`, fixed in the middle of the screen) and that fallback overlaps
  page copy, as it did before this work. The hero itself centres and the spiral orbits it,
  but the fallback phone sits behind the whole page.
