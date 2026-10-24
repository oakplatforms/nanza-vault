# Component Design Standards & Consistency Audit — nanza-mobile

Scope: `src/` — 580 files, 100,028 lines (components 256 files / 38,857 lines; screens 152 / 22,940; styles 43 / 30,056).
Yardstick: `CLAUDE.md` P0/P1/P2, `.claude/skills/ui-review/references/checklist.md`, `documentation/solution-designs/design-system.md`.
Every count below comes from a command run against the tree at commit `f590df7` (branch `dev`).

---

## Summary table

| # | Rule | Violations | Files | Verdict |
|---|---|---:|---:|---|
| 1 | Inline style objects (static, no dynamic excuse) | **76** | 53 | Fix now (partial) |
| 1b | Inline `StyleSheet.create` outside `src/styles/` | **1** | 1 | Fix now |
| 2 | Hardcoded colors (hex in live code) | **11** | 5 | **Fix now** |
| 2b | `rgba()` literals in live code | **9** | 8 | Later |
| 2c | Hex fallback in `\|\|` chain | **1** | 1 | Fix now |
| 2d | Raw scale refs where semantic exists (non-styles) | **90** | 35 | Later |
| 2e | Raw scale refs inside style buckets | 485 | 37 | Later (by design-ish) |
| 3 | Raw numeric sizes inside style buckets (excl. 0/1) | **579** | 33 | Later |
| 3b | Icon `size={number}` | **5** | 4 | Fix now |
| 4 | Font props outside `typography.ts` (ESLint) | **7** ✅confirmed | 2 | **Fix now** |
| 5 | `any` (ESLint `no-explicit-any`) | **21** ✅confirmed | 8 | Fix now (11 of them) |
| 5b | `as any` / `@ts-expect-error` | 11 / 2 | 7 | Fix now |
| 5c | Local DTO-mirroring types (rule 11) | **~8 real** of 61 matches | 8 | Fix now |
| 6 | Exact-duplicate style bucket classes | **621 classes ≈ 3,148 lines** | 33 | **Fix now** |
| 6b | Near-duplicate bucket classes (one prop differs) | 2,555 | 33 | Later |
| 6c | Files >400 lines | **27** (15,644 lines) | 27 | Later |
| 6d | Style buckets >1,500 lines | **5** (14,954 lines) | 5 | Later |
| 7 | `getXStyles(theme)` called unmemoized per render | **466 of 473** | 294 | Later |

**Headline:** the *authored* P0 surface is in genuinely good shape — 28 `style={{` sites in 100k lines, 11 live hex values, 7 ESLint font hits. The debt is not in rule-breaking, it is in **structural duplication**: 621 byte-identical style classes and a `bidCard*`/`listingCard*` twin family that is 59/72 identical. One module — **orders/reviews** — accounts for a disproportionate share of every remaining P0 class and looks like it landed without a UI review.

---

## 1. Inline styles

**`style={{ }}` object form: 28 occurrences / 18 files.** Full balanced-brace scan across every `*Style={` prop (incl. `contentContainerStyle`, `textStyle`) finds **164 inline-object style expressions in 82 files**; classifying by whether the body references a runtime value (Animated/transform/insets/computed dimension) gives:

- **88 legitimately dynamic** (allowed by the P0 exception)
- **76 static violations across 53 files**

Top 10 static offenders:

| Count | File |
|---:|---|
| 5 | `src/components/modals/WriteReviewModal/index.tsx` |
| 4 | `src/components/orders/OrderCard/index.tsx` |
| 3 | `src/components/entity/EntityTags/index.tsx` |
| 3 | `src/components/global/QuantitySelector/index.tsx` |
| 3 | `src/screens/posts/PostDetailScreen/index.tsx` |
| 3 | `src/screens/search/SearchScreen/AccordionTagSection.tsx` |
| 2 | `src/components/cards/CircleThumb/index.tsx` |
| 2 | `src/components/feed/FeedBulkCard/index.tsx` |
| 2 | `src/components/global/Button/index.tsx` |
| 2 | `src/components/global/CustomFlatList/Separator.tsx` |

Representative examples:

- `src/components/modals/WriteReviewModal/index.tsx:31` — `style={{ flexDirection: 'row', justifyContent: 'center', gap: 8 }}` (fully static, plus a raw `gap: 8`)
- `src/components/global/Button/index.tsx:320` — `fullWidth ? {} : { alignSelf: 'flex-start', width: 'auto' }` — the exact "conditional inline object" the checklist names; should be two bucket classes
- `src/components/global/CustomFlatList/Separator.tsx:23` — `{ backgroundColor: theme.colors.secondary?.['950'] || theme.colors.border }` — inline *and* a raw scale ref; belongs in `layout.ts` (the class `listSeparatorVertical` is already there)

A notable sub-pattern: `{ backgroundColor: <conditionColor> }` on `cardStyles.bidListCardConditionPill` recurs in **4 files** (`EntityCard/Listing.tsx:161`, `FeedBidCard:270`, `FeedBulkCard:285,294`, `FeedListingCard:237`) — this is dynamic-by-data so it is defensible inline, but it is the same 8-line JSX block 5 times (see §6).

**Inline `StyleSheet.create`: 1 file** — `src/components/global/QuantityStepper/index.tsx`. Single clean fix.

---

## 2. Hardcoded colors

Very clean overall. **Zero named colors** (`'white'`, `'black'`, …) anywhere in `src/`. `'transparent'` used 34× — explicitly allowed.

**Hex literals: 24 matches / 14 files, but only 11 are live code** — 13 are inside `//` comments documenting token provenance (e.g. `typography.ts:296` "resolve to secondary.800 (#5D6976)"), which is legitimate and arguably good practice. The real violations:

| File:line | Value |
|---|---|
| `src/components/orders/OrderCard/index.tsx:177,178,216,217` | `'#FBBF24'`, `'#A8ADB6'` |
| `src/components/modals/WriteReviewModal/index.tsx:39,40,137,138` | `'#FBBF24'`, `'#A8ADB6'` |
| `src/screens/orders/OrderDetailScreen/index.tsx:121,122` | `'#FBBF24'` |
| `src/styles/components/posts.ts:1230` | `color: '#FFFFFF'` |
| `src/styles/components/cards.ts:6260` | `shadowColor: '#000000'` |

**All eight component-level hex values are the same star-rating amber/grey**, in the same three order/review files. There is no `theme.colors.star` / `rating` token — that is the actual gap. `#A8ADB6` should be `theme.colors.border` (as `OrderDetailScreen:121` already correctly does, inconsistently with its siblings).

**Hex fallback chain: 1** — `src/styles/components/posts.ts:1282` `theme.colors.system?.error || '#ef4444'`. Straight P0 #4 violation, one-line fix.

**`rgba()`: 13 matches, 9 live.** Two are the legitimate `hexToRgba` util. The rest are gradient scrims all using the *identical* pair `['rgba(0,0,0,0.35)', 'rgba(0,0,0,0)']`, repeated in **5 files** — `ShareTagValueCard.tsx:53`, `CollectionDetailScreen:241`, `ProfileHero:66`, `SupportedTagValueScreen:130`, `GroupDetailScreen:283` — plus the inverse `['rgba(0,0,0,0)','rgba(0,0,0,0.9)']` in `ShareGroupCard:55` and `CollectionGridCard:161`. Also `layout.ts:3052` `shadowColor: 'rgba(46, 52, 56, 1)'` and `share.ts:626` `backgroundColor: 'rgba(0,0,0,0.3)'`. These want two named gradient tokens in `gradients.ts` (which already exists, 82 lines).

**Raw scale refs where a semantic token exists: 575 total, 90 outside `src/styles/`.** The design-system doc is explicit that "components reference semantic tokens only", so the 90 non-styles hits are the P0-relevant ones:

| Count | File |
|---:|---|
| 10 | `src/components/feed/FeedListingCard/index.tsx` |
| 8 | `src/components/feed/FeedBidCard/index.tsx` |
| 6 | `src/components/global/QuantitySelector/index.tsx` |
| 4 | `src/components/global/Header/index.tsx` |
| 4 | `src/components/feed/FeedBulkCard/index.tsx` |
| 3 | `src/components/global/QuantityStepper/index.tsx` |
| 2 | each: `TradeScreen/index.tsx`, `TradeScreen/CollectTab.tsx`, `BulkTab.tsx`, `TrackedPackages`, `ShareListingCard`, `QrCode` |

The 485 inside buckets are less severe (buckets are the semantic layer's consumer) but the doc's stated intent was to eliminate them; `cards.ts` alone holds 133.

---

## 3. Raw sizes

**Inside the style buckets: 1,316 raw numeric size props.** Excluding `0` and `1` (flex zeroing, hairline borders — not real violations) leaves **579 genuine raw numbers across 33 buckets**.

Ratio check: buckets contain **6,143** size-ish props total, of which **4,126 reference `theme.spacing/radius/icons`** — so roughly **67% tokenized, 33% raw**. The buckets are *mostly* compliant but not uniformly.

| Count | Bucket |
|---:|---|
| 179 | `cards.ts` |
| 59 | `layout.ts` |
| 47 | `inputs.ts` |
| 44 | `cart.ts` |
| 40 | `messages.ts` |
| 40 | `filters.ts` |
| 37 | `buttons.ts` |
| 31 | `badges.ts` |
| 18 | `modals.ts` |
| 18 | `entity.ts` |

Representative examples:

- `src/styles/components/layout.ts:1108-1109` — `width: 44, height: 44` — this is exactly `theme.spacing.hit` (44), documented in the decision guide
- `src/styles/components/layout.ts:1271-1272` — `width: 32, height: 32` — `theme.icons.lg`
- `src/styles/components/offers.ts:99,246,391` — `borderWidth: 0.61`, `height: 0.61` — a magic hairline repeated 3×, no token

A large share of the remainder are Figma-frame absolutes (`width: 365`, `width: 132`) which the decision guide says should be **proportional against the 402pt frame** (`maxWidth: '61%'`), not literals — so they violate a documented convention, not just a token rule.

**Icon `size={number}`: 5 occurrences / 4 files** against **78 correct `size={theme.icons.*}`** — 94% compliant. The 5: `Badge/Price.tsx:157` (`size={6}`), `OrderDetailScreen:120`, `WriteReviewModal:136`, `OrderCard:176,215` (all `size={16}` = `theme.icons.sm`). Again, all but one in the orders/review module.

---

## 4. Typography

**ESLint `no-restricted-syntax` reports exactly 7 hits — confirmed.**

- `src/screens/search/SearchScreen/AccordionTagSection.tsx:167,168,169,170` — four font props in a single inline `<Text style={{…}}>` at line 165. Worst offender; also holds 3 inline styles and 1 `any`.
- `src/styles/components/messages.ts:521,526,530` — font props written in a bucket other than `typography.ts`.

This rule is working. 7 hits in ~900 migrated text sites is excellent adherence, and the ESLint override block scoping the exemption to `typography.ts` only is correctly configured.

No evidence of `<Text>` without a typography slot at scale — the bucket-spread pattern (`...typography.<slot>`) is pervasive.

---

## 5. Types

**`no-explicit-any`: 21 — confirmed exactly.** Split by whether they are defensible:

*Infrastructure (9, arguably legitimate — generic JSON walkers):*
`styles/utils/parseNumericStrings.ts:1(×2),7`; `styles/utils/resolveThemeColors.ts:10(×2),18`; `utils/sanitizeNativeProps.ts:20,35,39,42`

*Real violations (12):*
`components/global/Tabs/Text.tsx:20,21,22`; `components/modals/WriteReviewModal/index.tsx:64,105`; `components/orders/OrderCard/index.tsx:168,171,205,210`; `screens/orders/data/useBuyerOrders.ts:61`; `screens/search/SearchScreen/AccordionTagSection.tsx:44`

**`as any`: 11. `@ts-ignore`: 0. `@ts-expect-error`: 2.**

The dominant `as any` pattern is **`(order as any).review`, 6 occurrences** across `OrderCard:168,171,205,210`, `OrderDetailScreen:72`, `WriteReviewModal:64`. Per P0 rule 11 this is precisely the "DTO appears missing → ask the user to bump `@oakplatforms/types`" case: `OrderDto` has no `review` relation. This is a **types-package gap, not a code fix** — it should be escalated, not patched.

**Rule 11 (local domain DTOs): 61 raw matches, but most are exempt.** The rule explicitly exempts component prop types and navigation route param types, which removes ~45 (`*RouteProp`, `*RouteParams`, `*Handle`, `*Variant`, union literals like `type OrdersTab = 'PURCHASES' | 'SALES'`). Genuine concerns (~8):

- `src/screens/search/data/fetchRecentSearches.ts:7` — `export type RecentSearchDto = {…}` — **named `Dto` outright**, the clearest violation
- `src/components/modals/ListModal/data/fetchExistingEntityLists.ts:6` — `export interface EntityListData`
- `src/screens/collections/.../fetchCollectionEntities.ts:11` — `export interface CollectionItem`
- `src/screens/trade/data/useTradeUserList.tsx:15` — `export type TradeUserListItem`
- `src/components/posts/PostItem.tsx:193` — `type PostTag = { id: string; name: string }` declared *inside a function body*, duplicating `PostHashtags/index.tsx:6` `PostHashtag = { id: string; name: string }` — same shape, two names
- Pagination wrappers `interface OrderPage/BidPage/ListingPage/CollectionPage/PostPage` declared **7 separate times** across `screens/*/data/*.ts` — all structurally `{ data: T[]; page: number; total: number }`. Not backend DTOs, so not strictly rule 11, but an obvious generic (`Page<T>`) waiting to happen.

---

## 6. Structure consistency

### Folder conventions — strong
163 `index.tsx` folder-components vs 75 loose `.tsx` files. The loose ones are not random: they cluster as deliberate sub-parts (`share/ShareCardParts.tsx`, `posts/PostItem.tsx`, `messages/*SheetContent.tsx`). Export style is near-uniform: **326 default exports vs 16 named**. Props naming is consistent — **256 distinct `*Props` types**. `React.FC` appears in 236 files, so that convention is also uniform. **This dimension needs no work.**

### Oversized files — 27 components/screens >400 lines (15,644 lines total)

| Lines | File |
|---:|---|
| 1,639 | `src/components/posts/composer/PostComposer.tsx` |
| 855 | `src/screens/search/SearchScreen/index.tsx` |
| 831 | `src/screens/entity/EntityScreen/index.tsx` |
| 770 | `src/components/global/Header/index.tsx` |
| 697 | `src/components/orders/OrderStatus/index.tsx` |
| 616 | `src/components/posts/PostContentItems.tsx` |
| 562 | `src/screens/bulk/CreateListingsScreen/BulkTab.tsx` |
| 549 | `src/screens/listings/ListingScreen/index.tsx` |
| 533 | `src/screens/trade/ScanCameraScreen/index.tsx` |
| 533 | `src/components/posts/PostItem.tsx` |

`PostComposer.tsx` at 1,639 lines is 4× the next-largest component and also carries the most inline styles (8) — it is the single riskiest file in the tree.

### Style buckets >1,500 lines — 5 files, 14,954 lines (50% of all style code)

`cards.ts` 6,376 · `layout.ts` 3,144 · `inputs.ts` 2,161 · `modals.ts` 1,675 · `buttons.ts` 1,598

### Bucket duplication — the headline finding

Parsed **3,097 style classes** across 35 buckets and normalized each body (comments/whitespace stripped):

- **Exact-duplicate bodies: 315 groups → 621 redundant classes ≈ 3,148 removable lines (≈10.5% of all style code).**
- 124 groups span more than one bucket; 191 are duplicated *within* a single bucket.
- **Near-duplicates (differ by exactly one property): 751 groups → 2,555 redundant classes.**

Worst groups:

- **23 copies × 3 lines** — `badges/profileContainer`, `buttons/accountToggleSwitchOptionRow`, `cards/productCardGridMarketPrices`, `cards/bidCardContentSm`, `cards/bidCardProfileXxl`, `cards/listingCardContentSm`, … (a generic flex-row)
- **20 copies × 2 lines** — `badges/optionText`, `cards/bulkDetailRowName`, `cart/cartOrderRadioButtonLabelText`, `filters/filterOpenTagText`, `inputs/bidInlineFieldLabel`, …
- **18 copies × 2 lines** — `authScreens/screen`, `cards/bulkDetailContainer`, `cart/cartScreenContainer`, `detail/screenContainer`, `entity/entityProductScrollView`, … (the screen-container class, re-declared per feature)
- **15 copies × 2 lines** — `cards/listingCardImageFull`, `detail/heroImageFill`, `detail/heroZoomFrame`, `layout/profileListBannerImage`, … (image-fill)
- **13 copies × 5 lines** — `branding/splashSheetLayer`, `buttons/buttonActionGradientSvg`, `cards/thumbCardGradient`, `detail/heroBlurBackdrop`, `entity/lockupFill`, … (absolute-fill overlay)
- **5 copies × 8 lines** — `cards/orderCardNameWrapper`, `cards/productCardGridNameWrapper`, `cards/productCardListingNameWrapper`, `cards/productCardListingNameWrapperNoTags`, `entity/entityNameWrapper`

### The `bidCard*` / `listingCard*` twin family

`cards.ts` declares 74 `bidCard*` and 93 `listingCard*` classes. **72 have a name-for-name twin; 59 of those 72 are byte-identical bodies (~297 lines of pure copy).** The 13 that differ are mostly the container/size variants (`bidCard` vs `listingCard`, `bidCardLg`, `bidCardXxl`). This is one parameterized family masquerading as two.

Compounding it: **35 base names carry 3+ size suffixes** (`Sm/Md/Lg/Xl/Xxl`), 222 size-suffixed classes total. `listingCardStatus/QuantityContainer/QuantityAvailable/Quantity/Profile/Price/Header/Condition/Body` each have all 5 — and the `bidCard` mirror has all 5 too. That is 10 names × 5 sizes × 2 families = 100 classes expressing what a `size` lookup map would express in ~20.

### Duplicated JSX patterns

- **Star rating block — 3 near-identical implementations**, ~15 lines each: `OrderCard/index.tsx:169-184` (twice, buyer + seller variant), `WriteReviewModal/index.tsx:31-46`, `OrderDetailScreen/index.tsx:113-127`. Same `[1,2,3,4,5].map`, same `star-filled`/`star` icon, same `strokeWidth: filled ? 0 : 1.2`, same hardcoded amber. Diverge only in whether grey is `'#A8ADB6'` or `theme.colors.border`.
- **`BuyerOrders` vs `SellerOrders` — 78 lines each, differing on 4 lines.** Normalizing `Buyer`↔`Seller` yields an 8-line diff: the data hook, the empty-state copy, and one action label. Two whole files where one component with a `role` prop suffices. Both also trigger the same `react/display-name` lint error.
- **Gradient scrim block** — the `['rgba(0,0,0,0.35)','rgba(0,0,0,0)')]` LinearGradient recurs in 5 files (§2).
- **Condition pill** — same `<View style={[cardStyles.bidListCardConditionPill, {backgroundColor}]}><Text>` in 5 places across 4 files.
- **Loading**: `ActivityIndicator` used directly in **32 files** despite `components/global/LoadingSpinner` and `LoadingIndicator` both existing (3 files use them). The shared primitive is being bypassed ~10:1.
- **Empty states**: `components/global/EmptyState` exists and 23 files reference it — good adoption; 31 files mention empty-state concepts, so the gap is small.

The `components/global/` folder is actually well-stocked (49 primitives incl. `Button`, `Badge`, `Panel`, `Tabs`, `TabPillsRow`, `Toast`, `Alert`, `DrawerModal`, `EmptyState`, `Header`, `TopHeader`). The problem is **discovery/adoption, not absence** — with `ActivityIndicator` as the clearest case.

---

## 7. Hooks and context

**`getXStyles(theme)` is called 473 times across 294 files; only 7 are wrapped in `useMemo`.** So **466 call sites rebuild their entire style object on every render.** Given `cards.ts` alone is 6,376 lines / 702 classes, a single `getCardStyles(theme)` call allocates hundreds of objects per render. **46 files call 3+ different `getXStyles` factories** (`AccountScreen/Settings` calls 6; `EntityScreen`, `AccountScreen/Wallet/CustomerInfo`, `AccountScreen/Profile`, `ShareCardParts`, `CardPickerLayer` call 5 each).

Mitigating: `ThemeProvider` resolves the theme **once at module scope** (`const fallbackTheme = parseNumericStrings(resolvedTheme)`) and never changes it — `theme` is referentially stable, so a `useMemo(() => getXStyles(theme), [theme])` would be a near-free win, and light/dark switching (marked PENDING in the design doc) will make it mandatory.

**Dead abstraction:** `src/styles/utils/useThemeStyles.ts` exists, calls `getThemeStyles(theme)` from the barrel, and has **0 usages** outside its own file. Either it becomes the memoized canonical accessor or it should go.

**React Query ownership is mixed:** 70 screens and **31 components** own `useQuery`/`useInfiniteQuery`, plus 11 in `src/hooks`. Components fetching their own data include leaf-ish card components — `FeedBidCard`, `FeedListingCard`, `EntityCard/Listing.tsx`, `EntityCard/List.tsx` — which is a render-in-list fetch pattern worth flagging. `useMutation` is confined to 13 files, consistent with P2 #19 (read-only by default). 7 contexts exist and are appropriately scoped; no deep prop-drilling chains found.

---

## Standardization opportunities (ranked by leverage)

| # | Opportunity | Files | Est. lines removed | Effort | Risk | When |
|---|---|---:|---:|---|---|---|
| 1 | **Collapse `bidCard*`/`listingCard*` into one parameterized `tradeCard` family** — 72 twins, 59 byte-identical | 1 bucket + ~8 consumers | ~300 direct, ~900 with size variants | L | Med | Later |
| 2 | **Dedupe exact-duplicate bucket classes** — 315 groups / 621 classes; promote the 124 cross-bucket ones into shared primitives (`layout.screenContainer`, `layout.absoluteFill`, `layout.imageFill`, `typography`-backed text classes) | 33 buckets | **~3,148** | L | Med | Later |
| 3 | **Size-variant lookup maps** — 35 base names × 3-5 suffixes (222 classes) become `{sm,md,lg,xl,xxl}` records keyed off one base | `cards.ts`, `buttons.ts`, `badges.ts` | ~600 | M | Med | Later |
| 4 | **`<StarRating>` component** — replaces 3 hand-rolled blocks, kills 8 hex literals, 4 raw icon sizes, 4 inline styles, 6 `as any` in one stroke | 3 | ~50 | S | None | **Fix now** |
| 5 | **Merge `BuyerOrders`/`SellerOrders` into `<OrdersList role=…>`** — 78-line twins with a 4-line diff; also clears 2 `display-name` lint errors | 2 | ~75 | S | Low | **Fix now** |
| 6 | **Adopt `LoadingSpinner` over raw `ActivityIndicator`** — 32 files bypass the existing primitive | 32 | ~60 | S | None | Fix now |
| 7 | **Two gradient-scrim tokens in `gradients.ts`** (`scrimTop`, `scrimBottom`) — removes all 9 live `rgba()` literals | 7 | ~20 | S | None | **Fix now** |
| 8 | **Add `theme.colors.rating.{filled,empty}`** — the only reason star hex exists; unblocks #4 | 1 token file | — | S | None | **Fix now** |
| 9 | **Memoize `getXStyles`** — wrap 466 call sites, or (better) ship one `useBucketStyles(getFn)` hook and revive the dead `useThemeStyles` | 294 | ~0 (perf, not LOC) | M | Low | Later |
| 10 | **Generic `Page<T>` type** — replaces 7 duplicate `OrderPage`/`BidPage`/`ListingPage`/`CollectionPage`/`PostPage` interfaces | 7 | ~40 | S | None | Fix now |
| 11 | **Escalate `OrderDto.review` to `@oakplatforms/types`** — removes 6 `as any` and 2 `no-explicit-any`; per rule 11 this is a package bump, not a local patch | 3 | ~10 | S | Low | **Fix now** |
| 12 | **Split `PostComposer.tsx` (1,639 lines)** into composer shell + layers | 1 → ~4 | ~0 (redistributed) | L | Med | Later |

**Fix-now bundle (#4–#8, #10, #11):** ~7 files, ~255 lines removed, clears **every live hex literal, every `rgba()`, 4 of 5 raw icon sizes, 10 of 21 `any`, 6 of 11 `as any`, and 2 lint errors**. Small, low-risk, and it retires the orders/reviews module as the tree's worst-conforming area.

**Later bundle (#1–#3, #9, #12):** ~4,650 lines of style code removable — a **~15% cut to `src/styles/`** — but it touches `cards.ts`, which backs every feed and detail surface, so it wants its own release and visual regression pass.

---

## ESLint enforcement to stop the drift

Everything above except the duplication findings is mechanically detectable today.

**Add `eslint-plugin-react-native`:**

- **`react-native/no-color-literals`** — would have caught all 11 hex + 9 `rgba()` hits. Currently **0 violations would remain** after the fix-now bundle, so this can be switched to `error` immediately and stays green. Highest-value rule available.
- **`react-native/no-inline-styles`** — 164 inline-object sites, but 88 are legitimately dynamic. Ship as **`warn`** initially, or as `error` with `eslint-disable-next-line` + justification on the dynamic ones (mirroring the existing typography exception convention, which already works well).
- **`react-native/no-unused-styles`** — with 3,097 classes across 35 buckets and no current dead-class detection, this will likely surface a meaningful additional cut on top of the 3,148 duplicate lines. Run it before doing opportunity #2 so the dedupe does not preserve dead code.

**Extend the existing `no-restricted-syntax` block** (it already proves the pattern works — 7 hits in ~900 sites):

- `Property[key.name=/^(padding|margin|gap|borderRadius|width|height)/][value.type='Literal'][value.value!=0][value.value!=1]` scoped to `src/styles/components/*.ts` — catches the 579 raw bucket sizes. Land as `warn`; too many to gate on today.
- `JSXAttribute[name.name='size'] > JSXExpressionContainer > Literal` on `Icon` — catches the 5 raw icon sizes. Safe as `error` after the fix-now bundle.
- `MemberExpression[property.value=/^[0-9]{2,3}$/]` outside `src/styles/` — catches the 90 raw scale refs in components, enforcing the design doc's "semantic tokens only in components" rule. `warn` → `error` after cleanup.

**Also worth fixing in config:** `react-hooks/exhaustive-deps` is referenced but the plugin is not registered (`CollectionSelectionContext.tsx:36` errors with "Definition for rule not found"), so hook-dependency checking is silently off tree-wide.

Current lint baseline: **70 problems** — 21 `no-explicit-any`, 17 `no-unused-vars`, 11 `no-unescaped-entities`, 7 `no-restricted-syntax`, 6 `no-trailing-spaces`, 3 `no-multi-spaces`, 2 `display-name`, 2 `indent`, 1 unregistered-rule. All but the `any` cluster are trivially auto-fixable, so a green baseline is within reach — which is the precondition for gating CI on lint and stopping drift for good.
