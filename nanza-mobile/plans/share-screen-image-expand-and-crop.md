# Share screens: tap-to-expand + cropped image UX

Two changes to the dedicated share cards (the deep-link share experience), both centered on
the shared `ShareImageWell` building block in `src/components/share/ShareCardParts.tsx`.

## Goals

1. **Tap-to-expand images** — every share card's image gets the same full-screen, swipeable,
   pinch-to-zoom viewer the feed cards already use (`src/components/global/ImageViewer`).
   Single-image cards open one image; carousel cards (lots, collections) open a swipeable
   viewer aligned to the carousel order, starting on the tapped slide.

2. **Cropped (cover) images for Listings & Lots** — today every share card renders the image
   at `width: '62%'` with `resizeMode="contain"`, so the card art floats tiny inside the well.
   Listings and Lots should render full-bleed `cover` (a cropped capture) to match
   `FeedListingCard` and `FeedBulkCard`. Bids, Collections, and catalog Products keep the
   current `contain` fit.

   Resolution note: the spoken request said "not lots", but the typed clarification said
   **"Listings and Lots need the cropped UX similar to FeedListingCard and FeedBulkCard"** —
   and those are exactly the two feed cards that use `cover`. Going with the typed answer.

## Affected files (5)

| File | Change |
|---|---|
| `src/components/share/ShareCardParts.tsx` | Extend `ShareImageWell`: optional `onPress` (tap-to-expand) + optional `fill` (cover, full-width). |
| `src/styles/components/share.ts` | Add `imageWellImageFill` (full-width, `aspectRatio`) style for the cover variant. |
| `src/components/share/ShareListingCard.tsx` | Wire `ImageViewer` (single) + pass `fill`. |
| `src/components/share/ShareBulkCard.tsx` | Wire `ImageViewer` (carousel sources + initialIndex) + pass `fill` on every slide. |
| `src/components/share/ShareBidCard.tsx` | Wire `ImageViewer` (single). No `fill`. |
| `src/components/share/ShareCollectionCard.tsx` | Wire `ImageViewer` (carousel sources + initialIndex). No `fill`. |
| `src/components/share/ShareProductCard.tsx` | Wire `ImageViewer` (single). No `fill`. |

(7 files total. Exceeds the 5-file P1 guideline — escalation noted below.)

## Design

### `ShareImageWell` (the hub)
Add two optional props, keeping the default behavior identical so nothing else regresses:

```tsx
interface ShareImageWellProps {
  image?: string | null
  overlayConditionName?: string | string[]
  onPress?: () => void          // when set, the image becomes tap-to-expand
  fill?: boolean                // when true, render cover + full-width (Listings/Lots)
}
```

- `fill ? resizeMode="cover" : resizeMode="contain"`
- `fill ? styles.imageWellImageFill : styles.imageWellImage`
- when `onPress` is set, wrap the `<Image>` in a `TouchableOpacity activeOpacity={0.9}`;
  the condition-pill overlay stays a sibling so taps on the image still register.

### Style: `imageWellImageFill`
Full-bleed cover variant — fills the well width, fixed portrait-ish aspect so the crop is
consistent (matches the feed cards' framed-portrait crop):

```ts
imageWellImageFill: {
  width: '100%',
  aspectRatio: <card-portrait ratio, e.g. 3 / 4>,
  borderRadius: theme.radius.xs,
}
```
The `imageWell` container's `paddingHorizontal: lg` is reduced/removed for the fill variant so
the image reaches the card edges like the feed cards (via a `fill` container style or by
zeroing horizontal padding when `fill`).

### Tap-to-expand wiring per card
- **Single-image (Listing, Bid, Product):** `const [viewerOpen, setViewerOpen] = useState(false)`,
  pass `onPress={() => setViewerOpen(true)}` to `ShareImageWell`, render
  `<ImageViewer visible={viewerOpen} source={image ? { uri: getImage(image) } : null} onClose={...} />`.
- **Carousel (Bulk, Collection):** mirror `FeedBulkCard` — `const [viewerIndex, setViewerIndex] = useState<number | null>(null)`,
  build `viewerSources` (a `useMemo` over the slides/entries, in carousel order), each slide's
  `ShareImageWell` gets `onPress={() => setViewerIndex(slideIndex)}`, render
  `<ImageViewer visible={viewerIndex !== null} sources={viewerSources} initialIndex={viewerIndex ?? 0} onClose={() => setViewerIndex(null)} />`.
  - `renderSlide`/`renderEntry` receive the index (`ShareCarousel`'s `renderItem` already passes `(item, index)`).

## Confidence

- **Centralize on `ShareImageWell` with `onPress`/`fill` props** — Confidence **0.9**.
  Additive, no behavior change for cards that don't opt in. Rejected alt: edit each card's
  image JSX independently — Confidence 0.6, duplicates the TouchableOpacity+viewer pattern 5×,
  violates DRY/P2.20.
- **`cover` + `aspectRatio` full-width for the fill variant** — Confidence **0.85**. Matches the
  feed cards' crop. Minor trade-off: exact aspect ratio needs to match the feed card's frame —
  will read the feed card's image container style to copy the ratio rather than guess. Rejected
  alt: fixed pixel height like the current `spacing[230]` — Confidence 0.65, doesn't adapt to
  card width.
- **Carousel viewer uses `sources`+`initialIndex` (FeedBulkCard pattern)** — Confidence **0.95**.
  Proven, already in the codebase; `ImageViewer` supports it natively.

## Escalation: touches 7 files (P1.15 guideline is ≤5)

The count is driven by the request spanning all share cards; each change is small and additive,
and 1 of the 7 is a style file. No navigation, no API, no shared context/util changes. The only
shared-infrastructure-adjacent file is `ShareCardParts.tsx` (a share-local building block, not
`src/contexts`/`src/styles/utils`). Proceeding once the plan is approved.
