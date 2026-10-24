---
tags: [nanza-mobile, plan, design-system, entity-lockup]
status: IMPLEMENTED (2026-08-17) — pending on-device QA per surface. Fair-market pill extracted to the shared FairMarketPill component and adopted by the entity detail sheet + trade cards (2026-08-18, see "The fair-market badge"). Remaining: PostContentItems TILE variant still on posts.ts entityRow* for its carousel-slot geometry (its pill is now the shared component); Cart.tsx untouched (order line-item, not in scope); Sell-feed endpoint has no include contract, so trade Sell rows show set-only until the payload carries entityTags (Buy rows have full tags via fetchBids). posts.ts entityRow* classes stay while the tile + LinkPreviewCard (entityRowFill) consume them.
---

# Entity Lockup Standardization — Plan

One shared row component for the entity lockup — thumb + title / number / set+tags +
fair-market pill + price pills — so the treatment is styled in ONE place. The inline post
entity card (Figma 655-83703, just landed) is the reference look.

## The lockup standard (target)

- **Row 1** `itemTitle` — entity name (SemiBold 15/16, `text`). Renamed from `searchRowTitle`
  (2026-09-15) once the slot served every card/tile/row name, not just search rows.
- **Row 2** `entityRowMeta` — product number (Balto 14/16, `secondaryText`)
- **Row 3** `entityRowMeta` — set + ALL primary tags, dot-joined
  (e.g. "Promo · Rainbow Foil · London 2025 · Pro Tour") — every consumer shows all three
  rows; today several drop the tags row entirely
- **Bottom-left**: fair-market outline pill (border + `background` fill, `pricePillAmount`
  text + FAIRMARKET logo) — exactly the inline card's pill. Since 2026-08-18 the pill is its
  own component, **`FairMarketPill`** (`src/components/entity/FairMarketPill/`) — see
  "The fair-market badge" below
- **Bottom-right, same line**: other prices (list price, bid price, user price, ×qty) as
  pills mimicking the fair-market pill's geometry but **`text` (white) fill + `ink` label**
  (the bright-button recipe, buttons.ts:314), replacing today's right-column dark badges
- **Chrome**: the inline card's gradient fill (`cardGradientFrom/To`) everywhere;
  - `inline` (in a post): full 0.5 hairline border + radius 20 — unchanged
  - `row` (everywhere else): top + bottom 0.5 hairline only, no radius
- Thumb: proportional 5:7 card thumb (the entityRowImage recipe)

## Consumers (row-style lockups found)

| # | site | today | files |
|---|---|---|---|
| 1 | Inline post entity card | the reference look (just landed) | `PostContentItems.tsx` + `posts.ts` entityRow* |
| 2 | Search entity rows (+ composer card picker, collection search overlay) | own recipe in `searchEntityRow.ts`, tags line exists, no gradient/border | `SearchEntityRow/index.tsx` |
| 3 | Bid/listing/offer/cart cards | `EntityCardBase.tsx` + cards.ts `bidListCard*` / `entityCard*` — chip-style tappable tags, separate price row | `EntityCard/{EntityCardBase,List,Listing,Offer,Cart}.tsx` |
| 4 | Trade Buy/Sell/Offers rows (+ bulk sale, trade summary) | `TradeListItem` + cards.ts `tradeListItem*` — right-column dark price badges, NO tags row | `TradeListItem/index.tsx` |
| 5 | Post link card | `linkRow` in posts.ts — title+domain only | restyle to same chrome/text slots (no entity data) |

Out of scope (not row lockups): thumbs/grids (`ListingThumb`, `CollectionThumb`,
`CollectionGridCard`, share cards), `GroupSearchRow`, `TradeUserListRow`, shelf carousels.

## Approach

**New `EntityLockupRow` component** (`src/components/entity/EntityLockupRow/`) + new
classes in the `entity.ts` style bucket (migrating the `entityRow*` recipe out of
posts.ts). Consumers pass data + `chrome: 'inline' | 'row'` + optional `prices`,
`rightAccessory`/`overlay` slots (search's collection band + qty badges, trade's Sold tag),
and press handlers. Old per-consumer style classes retire as each consumer migrates.

## Known data gaps (tags row can't always render yet)

- **Inline post card — OPEN, blocked on backend (verified 2026-08-18).**
  `PostEntityReference` (`@oakplatforms/types` **0.1.93, the latest published** — not a stale
  package) carries only `kind/id/name/displayName/image/price/set`. Posts are fetched with no
  `include` contract: the reference is a server-shaped projection, so there is no client-side
  param to add and no DTO bump that helps.
  **Required backend change:** expand the ENTITY reference projection with
  `product.number` and `entityTags.tag.brandTags` (the same shape `getDisplayTags` reads
  everywhere else), then publish a types bump.
  **Client is already wired** — `metaLine1` takes the number, `metaLine2` calls
  `getEntityTagLine`; both fill in with zero code change once the payload lands.
  The hardcoded `PLACEHOLDER_PRODUCT_NUMBER` ('HVY090' on every card) was **deleted**
  2026-08-18: printing one card's number on all of them asserted something false, the same
  call the fair-market value got in 2026-08-08. Decision (user): wait for the projection
  rather than hydrate per-card client-side (rejected: one request per entity card in a feed).
- **Trade rows**: `fetchTrade.ts` doesn't `include` entityTags → row 3 empty until the
  include param is extended (service-file change, allowed under P0 #8).
- Search rows already include `entityTags.tag.brandTags` — full data.

## The height floor is list-rows-only (2026-08-18)

`lockupRow` used to carry `minHeight` (104 + 40 = 144) for every consumer. That is a FIXED
floor against a PROPORTIONAL thumb (`89/358` of the row's inner width, height from 5:7), so
it only holds at full width: a full-width row's content lands right at 144 and the floor
never shows. Put the same card in anything narrower — a post gallery-rail slot — and the
thumb shrinks, the content comes in ~17pt under the floor, and `justifyContent: 'center'`
splits that slack onto the top and bottom. The card's insets stop matching its neighbours'.

The floor now lives in `lockupRowFloor`, worn only with `chrome="row"` (list rows, where a
uniform row height is the point). The inline card is unfloored: it is exactly as tall as its
thumb plus its insets at every width.

## One thumb box, and why the tappable rows were broken (2026-08-18)

Trade rows and search rows both render through `EntityLockupRow`, yet trade thumbs came out
at roughly a quarter of the size, and the row's 144pt floor (not its content) was holding the
height — so the text lines drifted apart under `space-between`. The component had TWO thumb
mounts: a plain `<Image>` for the non-tappable rows, and — where the thumb is its own press
target (trade: thumb → entity, row → edit) — the RNGH `TouchableOpacity` **itself** wearing
the box style.

The second one cannot work. RNGH's `GenericTouchable` renders
`<BaseButton style={containerStyle}><Animated.View style={style}>`, so a `style` of
`width: '24.86%'` lands on the INNER view, whose parent `BaseButton` has no width of its own
and is sized by its content. The percentage resolves against a box that is sized by the
percentage — Yoga collapses it. Reading the same `LOCKUP_THUMB_WIDTH` constant bought
nothing: the constant was never the variable, the mounting point was.

**The rule now: the sizing box is always a plain `View` and a direct child of
`lockupContent`, never a touchable.** Where the thumb is pressable, the touchable is a child
pinned to the box's four edges (`lockupThumbFill` on BOTH `style` and `containerStyle` — the
outer `BaseButton` is the node that takes the touch, and left unstyled it collapses to zero
and swallows the tap, which is why an earlier absolute-sibling attempt "lost" the press). It
parents the image so the press still dims it. `lockupImage` / `lockupImagePlaceholder`
retired — `lockupThumbBox` + `lockupThumbFill` are THE thumb, and the box carries the radius,
the clipping and the empty-state fill.

Generalising: a proportional width is a contract with the PARENT, so any wrapper that is
sized by its children (touchables, gesture wrappers, `alignItems`-shrunk boxes) silently
voids it. When a shared component looks right on one surface and small on another, compare
what each surface's node is mounted *inside*, not what constants it reads.

### …and then the same percentage still drew three different sizes

With one box, trade thumbs came out LARGER than search's. Same class, same constant — because
a percentage resolves against the parent's CONTENT box, and two things were varying it:

- **`lockupContentReserveBand` was shrinking the thumb.** Search rows reserve `icons.xl` (48)
  so no text runs under the collection plus-circle, and that padding sat on `lockupContent` —
  which is also the thumb's containing block. Content box 402−8−48 = 346, thumb 86pt, while
  an unbanded trade row got 402−8−4 = 390 and a 97pt thumb. The band is an ABSOLUTE overlay
  and only ever covers text, so the reserve moved to the body (`lockupBodyReserveBand`). The
  thumb no longer knows or cares whether a row has a band.
- **The fraction was the inline card's, applied to full-bleed rows.** `89/358` — and 358 is
  the POST card's inner width (370 at 16pt page margins, minus `lockupContent`'s 8/4). A
  list row's inner width is 390, so the same fraction drew 97 instead of 89.

Figma draws an **89pt thumb in both frames**; only the card's outer inset differs. So the
size is now one number in points, resolved per chrome against that chrome's real inner width:

```
THUMB_WIDTH_PT = 89
row    → 89/390   (full-bleed list row)
inline → 89/358   (post card at 16pt page margins)
```

`lockupThumbBox` / `lockupThumbBoxInline` differ in nothing else, and share the one source
number — they cannot drift the way two hardcoded fractions did. It stays a percentage so a
card in a narrower slot (post gallery rail) still scales with its card.

Net: search 86 → 89 (row height unchanged, still on the 144 floor), trade 97 → 89 (row 152 →
144), inline post card unchanged. All three identical at the 402 frame.

**The rule to carry forward:** padding on a lockup's own container is never free — it is the
thumb's containing block. Reserve space on the BODY.

Still NOT on the lockup: `TradeCardSelectionRow` (the ± stepper rows in `TradeCardSelector`,
`CollectTab`, `CollectScanItemsScreen`) hand-rolls its own row over `trade.ts`
`selectionCard*`, with a hardcoded 64×90 thumb. Migrating it means running the stepper
through the lockup's `overlay` + `reserveRightBand` slots, the way search's plus-circle band
does — deferred, it changes behaviour on three screens.

## The fair-market badge (2026-08-18)

The pill was a recipe copied per surface (the lockup's `lockupFairPill`, the post rail
tile's identical `entityRowPricePill`, and a wordmark-over-price stack on the entity detail
sheet, plus wordmark-then-price rows on the two trade cards). It is now ONE component —
`FairMarketPill` — over `entity.ts` `fairMarketPill` / `fairMarketPillText` (renamed from
`lockupFair*`, since the badge outlived the lockup). It owns the en-dash rule for unpriced
subjects, the 65:7 wordmark at 7pt, and the pill geometry; consumers pass a pre-formatted
amount (or `null`) and, where they need it, a layout-only style (margin/alignment) from
their own bucket.

Consumers: `EntityLockupRow` (so every row lockup inherits it), the `PostContentItems` rail
tile, the **entity detail sheet** (`EntityScreen`'s price row — previously the odd one out,
with the wordmark stacked above the amount), `BulkSaleCard`, `TradeCardSelectionRow`. On the
sheet the identity block and the fair-market row are wrapped in one group
(`entityModalIdentityGroup`, gap 20 = the card's own name→price distance): the sheet body's
54pt block gap read as a section break between the tags and the price, when they are one
lockup.
`TradeCardSelectionRow` also stopped printing `$0.00` for unpriced entities — absent is not
free, so it passes `null` and gets the dash.

**Deliberately NOT on this badge** — these quote fair market in a different idiom and were
left alone (user decision, 2026-08-18):
- Form help rows — `CreateBid`, `EditBid`, `CreateListing`, `EditListing`, `ScanItemEdit`:
  already consistent with each other via `inputs.ts` `formHelpRow` (amount + 80×9 wordmark,
  no rim). The pill was drawn for cards, not form footers.
- Collection value overlays — `CollectionThumb`, `CollectionGridCard`, `ShareCollectionCard`,
  `ScanCameraScreen`: a collection's TOTAL value over imagery, not an entity's price.
- `Badge variant="fairValue"` (`global/Badge`) is dead — zero call sites, and its
  "Fair Market™" text labels predate the wordmark asset. Retire on the next Badge pass.

Retired with it: `posts.ts` `entityRowPricePill`/`entityRowPriceText`, `cards.ts`
`listCardFairValueLogo` / `tradeListItemFairMarketRow` / `tradeListItemFairMarketLeft` /
`tradeListItemPrice`, `trade.ts` `selectionCardPriceRow` / `selectionCardPrice` — all
zero-consumer checked.

## Decisions (user, 2026-08-17)

1. Row 2 = product number alone; row 3 = set + all primary tags, dot-joined
2. Tag row truncates at 1 line (ellipsis); full tags live on the detail screen
3. Tags are plain text everywhere — bid/listing cards intentionally LOSE tap-to-filter
4. Extend `fetchTrade.ts` include params so trade rows carry entityTags

## Sequencing

1. Styles: `entity.ts` lockup classes (from posts.ts recipe) + white price pill
2. Component: `EntityLockupRow`
3. Migrate: PostContentItems → SearchEntityRow → TradeListItem → EntityCardBase+variants
4. Link card restyle
5. Retire dead classes (posts.ts entityRow*, searchEntityRow.ts, cards.ts tradeListItem*/
   bidListCard* leftovers) — zero-consumer checked per the wrap-up ritual
