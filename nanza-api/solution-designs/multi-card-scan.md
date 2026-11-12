---
tags: [nanza-api, nanza-mobile, solution-design, plan, scan, camera, anthropic]
---

# Multi-card scan — one photo, up to 12 cards

**Status:** **plan + review, written 2026-11-05 (Skylar, voice). API + mobile built 2026-11-06,
both uncommitted (API on `dev`: typecheck + lint clean, no migration, no types bump; mobile:
typecheck + lint clean). Nothing deployed, nothing run on a device. Mobile needs `pod install`
and a native rebuild for the new crop library. Detection reworked 2026-11-08 from the contact's
second splitter — see "Detection reworked" below; typecheck + lint clean, synthetic smoke test
passes, not deployed.** A contact sent over a card splitter
(`server.ts`, on Skylar's Desktop). This doc reviews it, records what was measured, and
describes the design that was built. Two reviews were done in the same session (Opus first,
then Fable); where they differ is called out in "The two reviews compared".

Related: [[../../nanza-mobile/solution-designs/scan|Scan]] and
[[../../nanza-mobile/solution-designs/scan-quick-add|Scan Quick-Add]] (the single-card flow this
extends).

## The goal

Take one photo of a binder page or a few cards on a table, and get every card in it — instead of
one photo per card. The UX stays the flow we have: capture, see the matches for a card, add it,
move on. The only change is that "move on" goes to the next card in the photo, and back to the
camera after the last one. A single-card scan is the same flow with an array of one.

## Recommendation

Use the splitter, but only its detection half, and keep every request small:

1. **Phone** takes the photo and keeps the full-resolution file on the device.
2. **Phone** sends a small copy (longest side about 1,200 px, roughly 200 KB) to a new
   `POST /scan/split`.
3. **API** runs the splitter's detection and returns **boxes, not images**: one entry per card,
   in reading order, as fractions of the photo (`x`, `y`, `width`, `height`, `angle`). No Haiku
   call, nothing counted against the scan cap. Capped at 12.
4. **Phone** cuts each box out of the full-resolution photo and holds the crops in an array.
   If the API finds nothing, the array is the whole photo — today's behaviour.
5. **Phone** sends each crop to the **existing** `POST /scan/cards`, unchanged, three at a time.
6. **UI** shows card 1 as soon as its answer arrives. Add or Skip moves to the next. After the
   last card it returns to the camera.

Why this shape, rather than one request that does everything:

| Concern | One big request | This design |
| --- | --- | --- |
| API Gateway's 29-second limit | All 12 Haiku calls share one budget; one slow call fails the whole page | Each request makes one Haiku call (2–4 s, estimate); nothing comes near the limit |
| Lambda's 6 MB request / response limit | A sharp 12-card photo is 4–7 MB as base64 going up, and 12 crops are 2–6 MB coming back | Largest request is one card crop, well under 1 MB |
| One card fails | Whole page fails or needs partial-result handling | That card shows "couldn't read", the rest carry on |
| Scan cap counted per card | Needs new counting logic | Already true: each `/scan/cards` call counts one |
| Time to first card | After all 12 finish | 3–4 s; the rest fill in behind |
| Server memory | 600–950 MB measured (see below) | About 300–400 MB |

`POST /scan/cards` and its product matching do not change at all, which keeps the risk in two
new places: the split endpoint and the phone's crop-and-queue.

## What was measured (2026-11-05)

The splitter was run locally (Node 22, Apple silicon) on synthetic photos: real card images
from the 2026-10-13 drop composited onto 12-megapixel backgrounds.

| Test photo | Cards expected | Cards found | Time |
| --- | --- | --- | --- |
| Binder 3×3, dark page | 9 | 9 | 0.51 s |
| Binder 3×3, two empty pockets | 7 | 7 | 0.41 s |
| Binder 4×3 | 12 | 12 | 0.45 s |
| Binder 3×3, white page | 9 | 9 | 0.44 s |
| Six loose cards, tilted, plain table | 6 | 6 | 0.28 s |
| Six loose cards, tilted, noisy background | 6 | 6 | 0.33 s |
| One tilted card | 1 | 1 | 0.19 s |

- Loading OpenCV takes about 0.4 s locally. Expect 1–2 s on a cold Lambda (estimate, not
  measured).
- Tilted cards come back straightened and the set code is legible in the crops.
- **Memory is the real problem with the file as sent.** On the 12-megapixel 4×3 photo the
  process sat at 617 MB after one call and 952 MB after six. The dev API Lambda has 1,024 MB.
- The same photo shrunk to 900×1,200 px (219 KB) still found 12 of 12, in 0.16–0.24 s, with
  memory at 300–400 MB. This is what makes the small-copy design work: detection never needed
  the full photo.

**What this does not prove.** The test photos are clean. They have no glare, no pocket plastic,
no sleeves, no fingers, no camera perspective and no foils. Those are exactly the cases a
hand-tuned detector fails on. A run on about 20 real photos is the gate before building
(see Build order).

## Review of `server.ts`

What it does: finds card-shaped rectangles using brightness thresholds and edge detection,
keeps one per spot, fills gaps in the binder grid from the cards it did find, drops empty
pockets (no detail inside), straightens tilted cards, and returns one JPEG per card.

What is good: the approach fits the problem, it is fast, it handles tilt and empty pockets, and
it has a sensible fallback for a photo that is just one card.

What to fix when porting it into the API:

1. **Memory.** It copies the full photo into OpenCV once per card, all at once (12 copies of
   about 36 MB for a phone photo). Returning boxes instead of crops removes this entirely.
2. **Clean-up on error.** OpenCV memory is freed by hand at the end. If anything throws first,
   it is never freed, and a warm Lambda keeps the leak. Wrap the body in `try` / `finally`.
3. **No upper limit.** Cap the result at 12 cards.
4. **Grid fill on loose cards.** The gap-filling assumes a binder grid. Three cards scattered on
   a table can produce invented cells where their rows and columns cross. The only guard is
   "has detail inside", which a patterned playmat can pass. Only fill gaps when the found cards
   actually form a grid (at least two rows and two columns).
5. **Smallest rectangle wins.** This is how it picks the card over the pocket around it, but on
   a card with a strong inner frame it could pick the frame. Watch for it in the real-photo run.
6. **Upside-down cards stay upside-down.** It straightens tilt but does not know which end is
   up. Haiku usually reads upside-down text; confirm in the real-photo run.
7. **House style.** Semicolons, double quotes, block comments and `any` all need converting.
   It belongs in `src/services/cardSplit.ts`, with input checks in `src/validation/scan.ts` and
   the route in `src/routers/scan.ts`.
8. **Bundle size.** `@techstark/opencv-js` is one 13 MB file. Put the split endpoint in its own
   Lambda so the main API's cold start and memory are not taxed by it.

## Other approaches considered

- **Send the whole photo to Haiku and ask for every card.** No splitter needed, but Claude
  shrinks images to about 1,568 px on the long side. On a 12-card page each card is then around
  350 px wide and the six-character set code is unreadable. Splitting is needed.
- **Detect on the phone** (Apple Vision / ML Kit rectangle detection). Best end state: no split
  endpoint, live outlines in the viewfinder. It needs a native module and a native rebuild, and
  the React Native upgrade is already deferred. Revisit after this ships; the queue UI and
  per-card calls carry over unchanged.
- **Upload the photo to S3 and have the server crop and store thumbnails.** Works, but adds
  storage, clean-up and a second round trip for no gain over cropping on the phone.

## The two reviews compared

Both agree on: the backend change is small because `/scan/cards` already returns an array; a
split step is needed; the API must return boxes rather than base64 crops; the detector is
heuristic and needs real photos; cost is about a tenth of a cent per card on Haiku.

Where the second review changes the first:

- **Where the fan-out happens.** The first review had one request split the photo and make all
  12 Haiku calls in parallel inside the Lambda. That still has to get the full photo up to the
  Lambda (over the 6 MB limit for a sharp photo), and it ties 12 calls to one 29-second budget
  with no retries. Moving the fan-out to the phone removes both problems and gives the
  per-card cap count for free.
- **What is slow.** The first review worried about OpenCV's speed. Measured, it is half a second.
  Memory, not time, is the limit, and only when cropping on the server.

## Decisions

Skylar, 2026-11-05 / 06:

- The lifetime scan cap counts **per card found**, not per photo. Three cards in the photo is
  three scans.
- A crop that reads no card **does not count** against the cap.
- Maximum **12 cards** per photo (a 4×3 page).
- The phone is told how many scans are left before it starts on a photo.
- The split runs in **its own Lambda**.
- No separate real-photo test round first: deploy to dev and test on the phone.

## Built (API, 2026-11-06)

- **`POST /scan/split`** — new Lambda `scanSplit` (`lambdas/scanSplitHandler.ts`, 1,024 MB,
  15 s, sharp layer, behind the same authorizer; guest tokens get 401). Body
  `{ imageBase64 }`, at most 2 MB of base64 (the phone sends a copy about 1,200 px on its long
  side). Returns `{ cards: [{ x, y, width, height, angle }], maxCards: 12 }`. Each box is in
  fractions (0–1) of the photo, in reading order, with 4 % padding already added. `angle` is
  the card's tilt in degrees, clockwise. An empty array means "treat the photo as one card".
  It makes no vision call, reads no database and counts nothing against the cap.
- **`src/services/cardSplit.ts`** — the detection half of `server.ts`, ported to house style,
  with the review's fixes: boxes instead of crops, clean-up in `finally`, capped at 12, grid
  fill only when the cards really form a grid (two or more rows and columns, at least half the
  crossings found).
- **A bug found while porting, now fixed.** On a dark-bordered card against a dark page the
  detector boxed the card's **text panel** instead of the card. The count still read "9 of 9",
  so the first benchmark missed it; it showed up when the boxes were drawn on the photo. Two
  checks were added: a box sitting inside a much larger card-shaped rectangle is replaced by
  that rectangle, and with four or more cards a box under about half the typical size is
  dropped so the grid fill recovers the whole card. All seven test photos now box whole cards.
- **`POST /scan/cards`** — unchanged contract. It now counts a scan only when a card name came
  back, and a reply that isn't JSON (a crop with no card in it) is treated as "nothing read"
  rather than a 500.
- **`GET /scan/usage?accountId=`** — `{ used, cap, remaining }` for the lifetime scan cap.
- **New dependency:** `@techstark/opencv-js` (13 MB; bundled only into the split Lambda).

Measured on the bundled, minified handler (local, Apple silicon): 0.1–0.2 s per photo, memory
steady at about 320–350 MB over 27 calls. Not yet run on Lambda.

Local `npm run dev` does not serve `/scan/split` (it is not an Express route); test it on dev.

## Built (mobile, 2026-11-06)

Four files plus one dependency; no navigation changes. The header pill and the zoom button
reuse classes already in the scan style bucket; the arrows and the pager add five.

- **`services/api/Scan.ts`** — `splitPhoto` and `usage` calls with their hand-written types.
- **`screens/trade/data/scanPhoto.ts`** (new) — makes the 1,200 px detection copy, asks the API
  for boxes, and cuts each box from a full-size, orientation-baked copy of the photo, shrunk to
  850 px on its long side.
- **`ScanCameraScreen`** — the single `scanResult` became a **queue** of cards with an index.
  - **One card or none found:** the whole photo goes to `/scan/cards` exactly as before, and
    the result is a queue of one. Single-card scans behave as they did.
  - **Two or more found:** one crop per card, scanned three at a time. The first card shows as
    soon as it resolves; the others fill in behind it.
  - **Header:** a "2 of 9" pill appears for a multi-card photo, beside the existing camera
    (rescan) button, which abandons the rest of the photo.
  - **Back / forward arrows** sit on either side of the frame, and the frame is a pager of the
    crops: tapping an arrow slides to the previous or next card, and swiping does the same. A
    card can be revisited. There is no Skip (Skylar, 2026-11-06: arrows instead).
  - **Add** slides on to the next card; after the last one is added, the camera is live again.
    A card paged back to opens the add sheet on the count just saved.
  - **A card that can't be read** shows its crop with the reason where the match tiles would
    be; arrow past it.
  - **Fewer scans left than cards found:** an alert says so and only that many are scanned.
    None left: the normal limit sheet opens.
  - **Scan limit reached part-way:** the limit sheet opens once and the remaining cards are
    marked instead of sent.
  - **Zoom:** a 1× / 2× button beside the shutter, opening on 2× as before. At 1× the hint
    reads "Fit up to 12 cards in the frame, fronts facing up".
- **Fallbacks:** if `/scan/split` fails or is not deployed, or the crop library is missing from
  an older native build, the screen falls back to the single-card scan.
- **New dependency:** `@react-native-community/image-editor` 5.x (native; the app had nothing
  that could crop).

Known rough edges, to judge on a device:

- The "scans left" notice uses the error-style alert (there is no neutral one).
- A tilted card's crop is the upright box around it, so it shows with some background.
- Crops are straight cuts; the returned `angle` is not used yet.

## Detection reworked (API, 2026-11-08)

The contact sent a second version of the splitter (`card-split.ts` + `opencv.ts`, on Skylar's
Desktop), tuned on real photos this time: its thresholds carry measurements ("empty pockets and
blank sleeves up to 0.10, real cards from 0.13", "98 FaB Cube images"). Reviewed against our
`cardSplit.ts` and ported as a **hybrid**: its detection, our contract. Decision (Skylar, voice,
2026-11-08): go with the hybrid if it can be done safely.

**Taken into `src/services/cardSplit.ts`:**

- A **Lab colour-difference mask** (pixels unlike the photo's typical colour) as a third source
  of card outlines, for dark cards on a light table or floor.
- A **text-box test by orientation**: a card-shaped rectangle 15–45 % of another's area, inside
  it and lying *across* it, is that card's text or art box and is never picked as a card. This
  replaces the "promote a panel to its enclosing rectangle" check from the first port.
- **Grid gap filling**: rows and columns are rebuilt as bands with an inferred pitch and the gaps
  between found rows and columns are filled — so a whole low-contrast row *between* two found
  rows is recovered, not just crossings of found rows and columns. The grid test is stricter: at
  least three outlined cards sharing one angle, each in line with another. **Not taken:** the
  contact's extension of the grid to a 3-pocket side past the outermost found row. Skylar's
  first real test (2026-11-08, seven loose cards laid out like a grid on a wood table) got 9
  boxes for 7 cards: the extension boxed bare table and the cables beyond the cards. Wood grain
  passes the edge-detail test and brown passes the colourfulness test. A card in a row of its
  own is now left out (missed, if it was outlined at all) rather than guessed at.
- **Colourfulness check** on inferred grid cells (middle half of the cell): an empty pocket with
  a finger at its edge passes the edge-detail test but is grey in the middle.
- **Background-contrast check** on inferred grid cells: the share of the cell's interior that
  differs from the photo's typical colour must be at least 0.7× the median share of the outlined
  cards. Measured on the wood-table photos: cards 0.72–0.99, bare table 0.19–0.54, cables 0.53.
- Edge-detail threshold recalibrated: Canny 12/36 with 0.115, from 30/90 with 0.03.
- A card may **touch one photo edge** (slightly clipped), and the minimum size drops to 0.8 % of
  the photo.
- **Neighbours from lines**: when only one card stood out, the Hough-line fallback runs and its
  rectangles are kept if they match that card's size and angle.
- **Single-card early exit**: no outline found but the photo itself is card-shaped and detailed
  returns an empty list (the phone's "one card" path) instead of running the line fallback,
  which would box the art panel.
- **`missed`** in the response: card-shaped, detailed spots that weren't boxed and aren't part of
  a boxed card — most often the light text box of a card whose border blends into the
  background. `POST /scan/split` now returns `{ cards, missed, maxCards }`; `missed` uses the same
  fractions as a box, no padding. Mobile has the type (`ScanMissedSpot`, optional) but shows
  nothing yet.

**Left out, and why:**

- **Perspective-warp straightening.** It returns pixels cut from the full photo, which is the
  memory problem the boxes contract exists to avoid; the phone's crop library can't warp. Boxes
  and `angle` stay as they were.
- **`identifyCard` inside the split** (file `card-find.ts`, not supplied). It verifies a card
  rebuilt from a lone text box by sending both candidate sides to the model — a vision call in
  the split step, which must stay free of model calls and cap counting. Rebuilt text-box cards
  are therefore not returned as cards; their text box shows up in `missed` instead.
- **Card-back rejection** (`isCardBack`, file `card-back.ts`, not supplied). Not needed for
  correctness: a back sent to `/scan/cards` reads no card and doesn't count. Add it when the
  file turns up, to save the round trip.
- Clean-up stays in `finally` (the second version frees Mats at the end of the happy path only).

**Measured (synthetic, local, 2026-11-08).** Five generated 1,200 px photos: binder 3×3 (9/9),
binder with two empty pockets (7/7), six loose tilted cards (6/6), one card (empty list, as
designed), three dark cards on a light table (3/3). About 0.5 s per photo including JPEG
decode (the Lab pass and extra threshold passes make it slower than the 0.1–0.2 s of the first
port); RSS 365–380 MB over 15 calls.

**Measured (real, 2026-11-08).** Two phone photos from Skylar (recreations of the failing
test: 7 loose cards in a grid-like layout on a wood table, one with cables in frame, one with
a card under the hand's shadow). Before the grid fix: 9 boxes each (table, cables). After: 6 and
6 — the six fully visible cards, boxed correctly; the seventh (off-grid, in the hand's shadow)
is neither boxed nor reported as missed, since no outline of it was found. Loose cards are the
harder case; a real binder page is still to be tested on dev.

**Follow-ups:** show `missed` on the phone (outline on the frozen still, "retake this card");
ask the contact for `card-back.ts`; re-check the 4×3 page (grid completion stops at three per
side, so a 4-row page relies on all four rows being found).

## Build order

1. ~~Real-photo run~~ — replaced by testing on dev (Skylar, 2026-11-06).
2. **API** — built 2026-11-06, see above. Deploy to dev.
3. **Mobile** — built 2026-11-06, see above. `pod install` + native rebuild.
4. **Device test on dev:** binder pages with and without sleeves, foils, glare, loose cards on
   a playmat, a hand in frame. If binder pages work and loose cards do not, ship with
   on-screen guidance ("lay cards flat on a plain surface").
