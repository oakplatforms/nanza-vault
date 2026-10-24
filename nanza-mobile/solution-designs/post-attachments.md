---
tags:
  - nanza-mobile
  - plan
  - posts
status: implemented — pending device QA and the payload bump
figma: "33373-76173 (UserPost) / 34152-2931 (EntityCardRow)"
---

# Post attachments — the ENTITY reference card

> **Status: PLAN.** Not yet implemented. Fold the durable parts into a
> `posts` solution design once this lands, and delete this file.

## Why

A post can attach one reference. The ENTITY branch currently renders a thin
row — 44pt thumb, name, set name — that predates the design. Figma
`33373-76173` specifies a much richer card: a large 5:7 card thumb, a
three-line identity block (name / product number / tag line), and a
fair-market price pill, all on a gradient fill with a hairline rim.

Scope is **entity references only**. LISTING / BID / LOT keep their existing
full-bleed hero tile (name overlay + white price pill, plus the lot's child
image pager) — the Figma node doesn't cover them and replacing the pager
would be a behaviour change, not a restyle.

## The spec, measured

Figma `EntityCardRow` (`34152:2931`) is **370 × 140** inside a 386-wide
`UserPost` on the 402pt frame.

| Part | Figma | Notes |
|---|---|---|
| Container | row, `gap 16`, `pl 8 / pr 4 / py 8`, `radius 12`, `border 0.5` | gradient →right `#060606` → `#080808` ("Card Gradient 1") |
| Thumb `34152:2965` | `89 × 124`, radius `5.4` | 89/124 = 0.718 ≈ the app's canonical 5/7 card ratio |
| Right column `34152:2971` | `253 × 120`, `justify-between`, `pb 8` | |
| Name `34152:2980` | SemiBold 14/16, +0.105, `--overlaytext` | = `typography.bold` |
| Number `34152:2983` | Medium 12/14, +0.316, `--secondarytext` | matched `searchRowMeta` at the time; the lockup now runs `entityRowMeta` (Balto 14/16) and `searchRowMeta` was removed unused (2026-09-15) |
| Tag line `34152:2984` | same slot, 4pt below the number | |
| Price pill `34152:2992` | `117 × 24`, `px 8 / py 6`, `gap 6`, `radius 8`, `border 1`, bg `--background` | `$85.00` SemiBold 12/12 + FAIRMARKET logo 54×8 |

The post container itself already matches Figma exactly — `posts.ts`
`commentCard` is `paddingTop base + xxs` (24), `paddingHorizontal sm` (8),
`borderTopWidth 0.5` / `borderTopColor border`. Nothing to change there.

## Proportional sizing

Per [[design-system]] the thumb must not be a literal `89`. The row is
`w-full`, so the thumb is a **percentage of the row's inner width**
(370 − 8 pl − 4 pr = 358):

```
width: `${(89 / 358) * 100}%`   // 89 of the row's 358pt inner width
aspectRatio: 5 / 7               // height is a consequence of width
```

The Figma numerator stays visible, per the `thumbSizes.ts` convention.
`5 / 7` is preferred over a bespoke `89 / 124` (0.7177 vs 0.7143 — a third of
a point at this size) so the thumb matches every other card thumb in the app.

## The blocker: the payload is missing three of the four fields

`PostEntityReference` (`@oakplatforms/types` → `dist/shared.d.ts:80`) carries:

```ts
{ id, name, displayName, image, lowestAsk, highestBid, set: { id, displayName } }
```

Against what the design draws:

| Design element | Available? |
|---|---|
| Card image | ✅ `image` |
| "Kassai of the Golden Sand" | ✅ `displayName` |
| "HVY090" (product number) | ❌ **not on the reference** |
| "Heavy Hitters • Cold Foil" | ⚠️ `set.displayName` gives the set; the **tag half is missing** |
| "$85.00 FAIRMARKET" | ❌ only `lowestAsk` / `highestBid` — **fair market is `product.price`** elsewhere (`SearchEntityRow`) |

P0 #11 forbids inventing a local DTO to paper over this. So the row ships in
two passes:

- **Pass 1 (done)** — full geometry, gradient, typography and pill chrome.
  The number and the price are fed **named placeholder constants** at the top
  of `PostContentItems.tsx`, marked "DELETE AS A BLOCK", so the card can be
  reviewed on device rather than shipping with two blank rows.
- **Pass 2 (blocked)** — server expands `PostEntityReference` with
  `product.number`, the primary tag values, and `product.price`;
  `@oakplatforms/types` is bumped; the placeholder block is deleted and the
  three reads swap in. No layout work, because the row already accepts them.

`lowestAsk` is deliberately **not** used as the price stand-in: it is the
lowest ask, a different number from fair market (`product.price` everywhere
else — see `SearchEntityRow`). Printing it under the FAIRMARKET wordmark would
assert something untrue.

## New tokens

- **`radius['12']`** — the scale ran `min 4, xxs 6, xs 8, sm 11, md 16`, and
  `sm` was 1pt off. Added as a numeric escape hatch alongside the existing
  `'24'`/`'28'`/`'29'`/`'36'`.
- **`system.cardGradientFrom` / `To`** (+ theme aliases, both JSONs) —
  `#060606` → `#080808` sits between `background` (`#010101`) and `ink`
  (`#121212`), so no existing pair matched. Figma names the style "Card
  Gradient 1": design-system vocabulary, not a one-off. Light theme gets a
  near-white analogue.
- **`typography.pricePillAmount`** — SemiBold 12/**12**, +0.11, secondary.
  `small` is the same face and size on 16 leading, which made the pill 28pt
  tall against the spec's 24. Font props may only live in `typography.ts`, so
  the tighter leading had to become a slot rather than a local override.

## Implementation notes

**The row is three nodes — rim, fill, content — and each one is load-bearing.**

Putting the border and the gradient on a single node **double-edges the card**:
Android paints the rounded background drawable and the outline as separate
passes, so the fill's edge shows as a second hairline just inside the border.
The rim therefore lives on a plain outer `View` (radius + `borderWidth: 0.5` +
`overflow: hidden`) and the gradient is an inset-0 child.

That outer node carries **no padding**, which is the second half of the trick:
Yoga — unlike CSS — offsets absolutely-positioned children by their parent's
padding, so a fill inside a padded parent stops short and leaves an unpainted
band at the edges (the same artifact, arrived at from the other direction).
The padding lives on a third `entityRowContent` node instead.

**Top/bottom anchoring** is `alignSelf: 'stretch'` + `justifyContent:
'space-between'` on the body column. Without the stretch, the row's
`alignItems: 'center'` centres the column and both blocks drift as the line
count changes. The column insets 10pt (`sm + min`) inside the thumb's top and
bottom edges, per the node.

## The multi-entity rail — same card, narrower (2026-08-18)

Attaching MORE than one entity puts each child in a horizontal rail. It used to
render a different card: the slot was a flat **200pt** (`posts.ts` `galleryItem`,
narrowed from 230 on 2026-08-12 to keep the rail "the supporting cast"), which is
about half a card. At that width the name truncated to one line, the fair-market
pill overflowed and clipped at the slot's edge, and the set line had to be
suppressed to fit at all (`renderLeaf(…, rail = true)`).

The slot is now **container width − a fixed peek** (`theme.spacing.xl`, 40), so a
rail child is the SAME lockup as a lone attachment, only narrow enough that the
next card shows. The rule lives in one place — `components/posts/galleryRail.ts`
(`getRailSlotWidth` + `estimateRailSlotWidth`) — because two different hosts draw
a rail:

| Rail | Host | Measured by |
|---|---|---|
| Draft | `PostComposer.tsx` (`RailScrollView`, platform-split — see below) | its own `onLayout` |
| Posted | `PostContentItems.tsx` GROUP branch (GH `ScrollView`) | its own `onLayout` |

**Why a point-peek and not a fraction**: the peek's job is the constant affordance
"there's another card that way", which should read the same on every device. The
slot stays proportional because the container it subtracts from is. This is also
why the rule is NOT a step in `thumbSizes.ts` — that table is fractions of the
CONTENT width for thumbs, and these two hosts have different insets (composer body
vs the posted card's 8pt `commentCard`), so one window-derived fraction would
mis-size one of them.

**The draft rail's scroller is platform-split** (`RailScrollView`), because the two gesture
systems fail in opposite directions in this one spot:

- **iOS → gesture-handler's `ScrollView`**, the app's rule for horizontal rails. RN's own
  scroller sits under the JS responder that the `DismissTapWrap` ancestor claims on
  touch-down, and the rail took no horizontal drag at all. (The earlier note here claimed iOS
  was safe because its recognizers run in parallel — that held for the responder *fight*, not
  for a Touchable ANCESTOR wrapping the whole draft tree.)
- **Android → RN's `ScrollView`.** This rail nests in an RN vertical scroll inside a native
  Modal, where the GH pan never activated even under a `GestureHandlerRootView`;
  native-in-native nesting hands the horizontal/vertical split to Android itself. Android
  never had the responder problem either — `DismissTapWrap` renders bare there for that
  reason.

The posted GROUP rail has no such split: it is GH's everywhere, like every other rail.

**Why measured, not derived**: `onLayout` on each rail is exact and host-agnostic
(the precedent is `tileWidth`, the lot pager's measured slide width in the same
file). Both seed from `useWindowDimensions` on the first frame so nothing ever
lays out at zero width.

Consequences: `galleryItem` is layout-only now (`flexShrink: 0`; the width arrives
at runtime, the sanctioned dynamic-value exception). The `rail` flag no longer
changes the ENTITY card — it survives only for tile mode's own tag line — and the
2026-08-12 "drop the set line in the rail" call is reverted, since it was a
workaround for the 200pt slot and died with it. Still to verify on device: the
posted half stays unexercised until the server emits GROUP items, so QA runs
through the composer rail.

## The LISTING/BID hero footer is the rail tile's footer (2026-08-18)

The hero card for a listing or bid attached to a post (name + price pill bottom-left over the
artwork) had its own overlay classes in posts.ts — `referenceHeroOverlay` /
`referenceHeroPriceRow` / `referenceHeroName` — which were a copy of what a carousel tile
already draws (`ItemThumb` → cards.ts `thumbCardFooter` / `thumbCardTitle` /
`thumbPricePillHitbox`). It even borrowed `thumbPricePill`/`thumbPricePillText` from cards.ts
for the pill itself, so only the frame around it was duplicated.

The copy had drifted, in the way copies do. Both butt the title straight against a
fixed-height, centre-justified slot — the slot, not a margin, is the breathing room — but the
tile's slot is `spacing.hit` (44) and the hero's was `icons.xl` (48). Pill height is 22
(`pricePillAmount`'s 14pt line + 4/4 padding), so the gap is `(slot − 22) / 2`: **11pt in a
carousel, 13pt on a post**. The same listing sat 2pt looser depending on where you met it.
`icons.xl` was also a category slip — an icon-size token standing in for a touch height, with
a comment calling 48 "the standard touch height" when the token for that is 44.

The hero now renders the tile's own classes and the three posts.ts copies are retired
(zero-consumer checked). Bottom inset is unchanged either way: the tile pairs its 11pt slot
with a 2pt `paddingBottom` compensation, so both surfaces still seat the pill 13pt off the
card's bottom edge — only the name→price distance moved.

**The rule:** name-over-price-over-artwork is ONE lockup — `thumbCardFooter` in cards.ts.
Post heroes, rail tiles, and grid tiles all render it; nothing re-implements it.

## Files

| File | Change |
|---|---|
| `styles/tokens/dark.json` / `light.json` | `radius['12']`, `system.cardGradientFrom/To` + aliases |
| `styles/utils/types.ts` | the two colour aliases on `ThemeTokens` |
| `styles/components/typography.ts` | `pricePillAmount` slot |
| `styles/components/posts.ts` | `entityRow*` rewritten; identity / meta / pill classes added |
| `components/posts/PostContentItems.tsx` | entity branch JSX + the placeholder block; GROUP rail measures itself and sizes its slots |
| `components/posts/galleryRail.ts` | **NEW** — the rail slot rule (container − peek) shared by both rails |
| `components/posts/composer/PostComposer.tsx` | draft rail measures itself and sizes its slots |

No new component: `entityRow*` had exactly one consumer, so extraction would
have been speculative. Both the posted card and the composer preview route
through this one renderer, so the single change covers both surfaces.
