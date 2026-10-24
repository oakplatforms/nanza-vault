---
tags: [nanza-web-app, solution-design, landing, responsive, mobile]
---

# Compact mode — Solution Design

What the public site does below **1024px**: phones, phones on their side, and
tablets in portrait. Audited and largely written on 2026-09-22.

## Overview

There is **one breakpoint**, `lg` / 1024px, and three things change on it
together:

| | below 1024 | 1024 and up |
| --- | --- | --- |
| Section copy | full width | 5 of 12 columns, left or right |
| The phone | `StaticPhone`, a dimmed backdrop | the 3D scene |
| The nav | hamburger + overlay | the full bar |

They move together because they are the same fact. The column exists to keep
copy clear of the phone beside it; where there is no phone beside it, there is
no column. `Navigation` had already drawn its line at `lg`, so the page now
changes mode on one line rather than three.

## How it got there

Measured at 390px before the change: `conversation` and `toolkit` rendered
their copy in a **122px column** — `Section`'s `col-start-1 col-end-6` had no
responsive variant, so the desktop 5/12 applied on a phone, putting a 36px
headline about three words to the line. `md` was the first fix and was still
wrong: at 768 the column comes to 253px, the same squeeze with a bigger
headline in it. Hence `lg`, and hence moving `useImmersive` to match — the band
from 768 to 1023 had been paying to download three.js in order to make the page
harder to read.

## The opening, without a phone to hand to

The spiral does **not** drain below 1024 (2026-09-22). The drain is not an
effect, it is a handoff: the cards collapse onto the exact silhouette the 3D
phone grows out of — a 0.069vh core, chosen as half the phone's width at the
dolly it starts from. Where that phone never arrives, the collapse has nothing
to hand to and reads as the spiral falling down a hole, on a screen where the
static phone is a different size in a different place.

So on this path the field keeps looping and the whole spiral fades out instead,
over `VORTEX_OUT` (0.18–0.55 of the hero's scrub), starting after the copy has
cleared so the two read as one movement away from the hero. `drain` stays 0, the
clear eye stays the shape of the copy, and the ticker stops once opacity reaches
zero so a phone is not spending battery on an invisible field.

The switch is `useImmersive()`, not a width query — reduced-motion and no-WebGL
readers do not get the phone either, and the spiral should not drain into
something none of them will see.

## The phone below 1024

`StaticPhone` is what renders, and until 2026-09-22 it threw the screenshots
away: it read `SCREENS` for its labels and painted a gradient, so every phone
visitor — the only people who ever see this path, plus reduced-motion and
no-WebGL — got grey where the app should be. It now renders `spec.src` over
that gradient, and a capture that 404s falls back to exactly what the component
used to be (see `screenSpecs.ts`; `02-profile.jpg` and `03-payout.jpg` are
still uncaptured).

It is a **fixed backdrop** and the copy scrolls over it, so below `lg` the
capture carries a `bg-page/65` scrim. Measured after: the ground under the body
copy sits at L≈1 (about 12:1 against `ink-muted`) while the phone area still
reads at mean L≈14–26 with sd≈21–29 — dim enough to read over, present enough
to be the app.

It also arrives and leaves with the 3D phone, by reading the same `opacity` off
`SECTIONS`. It used to be permanently on, which put a phone behind the hero and
behind the closing statement. It **rises** as it fades in — 28px under a 700ms
ease — because the 3D phone grows forward out of the spiral's centre and this
path cannot; appearing at full size on an opacity ramp alone reads as a cut,
and a short lift gives it a direction to have come from.

**It is not a still of the 3D scene.** Trying to be one is how it ended up
showing nothing at all. It tells the same story with what a phone can afford,
which means it keeps the two set pieces:

- **The topic cycle**, through "A space for every part of the game". The 3D
  phone flicks these on a 0.55s timer; here the reader's thumb is the clock — a
  ScrollTrigger over the section maps its travel onto `TOPICS`, so the set
  cannot run out before they arrive or keep flicking after they have gone, and
  scrolling back up runs it backwards. Captures are preloaded on first entering
  the section and only then, and swapped in only once decoded, or the cycle
  reads as images loading rather than as a phone flicking.
- **The offer card**, on "Buying and selling made easy". In 3D it lifts off the
  glass through perspective; here it lifts the flat way, up and in, sitting
  **over** the scrim rather than under it — dimmed, it would read as part of the
  screenshot instead of as something landing on top of it. Low on the glass, to
  stay clear of the copy crossing the middle.

`TOPICS` moved to `scene/topicSpecs.ts` to make this possible: plain data with
no `three` import, the same split `screens.ts` / `screenSpecs.ts` already uses,
so the non-WebGL path can read a list of filenames without the 3D library
coming with it.

## The 0% commission section is not pinned below 1024

The pin is a desktop gesture: it spends 150vh of scroll to hold one sentence
still while the closing CTA arrives and shoves it away. On a phone that is half
a minute of thumbing past a line already read, the shove lands off the bottom of
a screen that is mostly copy, and the reader cannot tell why the page stopped.
Below `lg` it is an ordinary section — one screen, centred, scrolls past.

## Touch details

**Hit areas, not icon sizes.** The hamburger's mark is 21px and its target was
21px; it is now 43px, by padding and a negative margin that cancels it, so the
bar's layout and the icon are untouched. Same trick on the logo (24 → 44), the
overlay's store icons (24 → 48) and the footer's legal links (16 → 32).

**The menu pins the page.** `body { overflow: hidden }` while the overlay is
open, restoring the previous value rather than clearing it, plus
`overscroll-contain` on the overlay itself — the body lock alone is unreliable
for touch on iOS. The overlay is `overflow-y-auto` because on a phone held
sideways the menu is 472px tall in 390px of screen, and it is still the only
menu on offer there.

## Asset weight

Fixed on 2026-09-22: the capture set came to **4.9MB** and now comes to
**1.4MB** over the whole mobile journey.

- **Two captures were off-spec.** `topic-attackreaction.jpg` and
  `topic-worldtour.jpg` were 1318x2868 at 1.4MB and 1.7MB, where
  `scene/topicSpecs.ts` documents 828x1800. Resized to spec, they are 160KB and
  179KB.
- **All captures re-encoded** at quality 80 with **4:4:4 chroma**
  (`subsampling=0`). These are UI screenshots: chroma subsampling is exactly
  what smears coloured text and 1px rules, and full chroma at q80 beats q90 at
  4:2:0 for this material. Measured 39–43 dB PSNR against the originals.
- **The offer card is WebP**, not PNG. It has soft transparent corners so it
  cannot be a JPEG, and the alpha was costing 264KB against 57KB for the same
  picture at 44 dB. Both paths load it as an image — `THREE.TextureLoader` is an
  `<img>` underneath — so nothing else had to change.

The **card artwork** the spiral draws from (`src/assets/cards`, 39 WebP files,
2.1MB, ~56KB each) is already efficient: re-encoding at q78 saves 6%. The lever
there is how many of them load, not how they are encoded — `CardVortex` takes a
`count`, which is 40, and a phone pulls the whole pool for a spiral that has
faded out by the end of the first screen.

## How to audit it again

Chrome over CDP with `Emulation.setDeviceMetricsOverride`, measuring rather than
eyeballing: `document.documentElement.scrollWidth` against `innerWidth` for
sideways scroll, `getBoundingClientRect` on every `h1/h2/p/li` for text set in a
thin measure, and on every `a/button` for targets under 44px. That is what found
the 122px column, and it survives a reviewer who cannot see the screen.

## Known gaps

**At exactly 1024** the copy column is 359px against a `text-6xl` headline —
tight, and unchanged from before this work. It opens up quickly above that (533
at 1440).

**Share and cart surfaces are not covered here.** This is the marketing site;
`app/ShareDetail` and friends keep the light theme and their own layout.
