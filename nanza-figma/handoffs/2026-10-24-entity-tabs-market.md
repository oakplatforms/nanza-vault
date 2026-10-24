---
tags: [nanza-figma, handoff, nanza-mobile, entity, market, charts]
status: open
date: 2026-10-24
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=2751-65639
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 2751:65639
frames:
  - 2751:65639 — Market tab ("Entity-Tabs-Market / default · Seller · edit me")
  - 2758:88229 — Chat tab
  - 2758:88920 — Trades tab, empty state
frame: Entity-Tabs-Market / default · Seller · edit me
repo: nanza-mobile
---

# Entity screen — tabs + Market (price charts)

## Request (verbatim)
**Changed:**
- I added tabs to the entity screen
- I added charts and data displays to the market tab on the entity screen
- I moved the chat (posts feed) to the first tab and made the 'join the chat / start the chat' (should be conditional based on if posts already exist or not) conditional in that it shows up fixed at the bottom like it does now on the chat tab only — [2758-88229](https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=2758-88229)
- I moved trades into the trades tab and display bid and listing cards in a grid for now like a shelf screen (more design to come but not ready for this). If there are no results it should show the content on this screen — [2758-88920](https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=2758-88920) — and link the trade button to the create listing screen for now
- I moved the card descriptions and tags all to the details screen (styles to come but they can display the way they do now)
- All the tab movement is important but the key thing here is this new pricing tab with the charts. It's based on assumptions about how we'll implement charts but that will ultimately be decided by our dev team. This should all work with react native gifted charts and the api endpoints from tcg api — https://tcgapi.dev/api/prices/ . I added oak-cortex to this workspace because it already fetches the prices we'd be displaying here so it's an example. Our dev team will ultimately lead the integration into nanza mobile though so this should work with however they get the data onto the screen
- My goal here is to allow the dev team to basically print plain text on the screen from the api and then run this handoff and have these screens be built for them with 1:1 pixel perfect designs and functionality that mirrors our dev/architectural/semantic decisions used elsewhere in Nanza-mobile and in the vault
- The chart should show 1M selected by default but both charts (line and bar) update when you toggle 1W (one week) or 3M (three months). If the price change is negative over a time period the line chart should show red (we have a 'damaged' red in our design system that should work)
- There are some new font styles that will need to be minted or updated for this new screen
- We need indicators for the price today/latest price (ideally this would animate like a pulse animation, not too fast but so you know it's living/moving) and then a low and peak indicator. That low and peak text will need an outline or shadow in case it ever overlaps with the line/gradient in the chart
- When you tap and hold the charts a vertical line with a popover should display with the data from that date for each respective chart. Use our styles to create a bubble like style for that data that contrasts will with the chart and fits our design system. If we need to revise we'll share another mcp but you can implement from what you have now. The data should display the price and date or volume and date
- The spacing between the card name and number/set may be updated, check to confirm
- I updated the add to collection icon in the header to use a new icon

**Do not change:** anything else on this screen

## Prerequisite
The dev team gets the market data onto the entity screen first. Plain text is enough. This
handoff then builds the UI on that data and its types. It does not decide how the data is
fetched (see Open questions D1–D5).

## Measurements
All frames are 402 wide, with content 362 wide at the 20pt inset. Y offsets are measured inside `modals.entityModalSheetBody`.

**Shared (every tab)**

| Element | Spec |
|---|---|
| Title block | Unchanged, checked: name `jumbo` 32/38; number 8 below (`spacing.sm`); set line 2 below the number (`spacing.min`). Matches `detail.titleTextGroup` / `detailLockupMetaGroup`. |
| Tab bar | 362×44, **32 below the title block**. Track: `background` fill, 1.1 `glassBorder` stroke, full pill radius, 3 padding. Four equal segments (flex 1, about 35 tall, radius about 17.5): **Chat · Market · Trades · Details**. Label `pill` 13/14; active `textSecondary` on a `glassBorder` fill (same value as `glassActive`, which is the better semantic name); inactive `secondaryText`. **Sticky** under the floating header. |
| Header right | The `cards-2` circle is replaced by a new icon: a plus in a square over a second square, 20×20 in the 44 glass circle (vector in instance `2798:123555`). Needs an SVG export and an `iconMap.ts` entry. The Chat and Trades frames still show the old icon, but the new one applies everywhere. |

**Market tab** (node `2751:65639`)

| Element | Spec |
|---|---|
| Price block | 26 below the tab bar. Label "Market price": **new Regular 12/16 +0.11 slot**, `secondaryText`. Price below it (gap 6, padding 8 above and below the pair): `large` 24/28, `textSecondary`. Then `legal` 11/13 `textOnSurface`, "Prices from TCG API · updated {fetch time}", directly underneath. |
| Change chips | 22 below the block. Three equal chips (flex 1, gap 8, about 115 each), 30 tall, full pill radius. Triangle 9×8 at x12, y11. Percent in `pill` 13/14 at x27. Period (`24H` / `7D` / `30D`) in `small` 12/16 `secondaryText`, right-aligned with 12 padding. **Up:** `positiveSubtle` fill, `positive` text, triangle up. **Down:** `negativeSubtle` fill, `negative` text, triangle down. **0% (or no data):** `border` fill, `secondaryText` text, no triangle, shown as "0%". |
| Line chart | 20 below the chips. 362×147 smooth line, top of the line about 39 down. Fill under the line is the line color fading to 0 at the baseline. Dashed baseline at the bottom. **Green (`positive`) when the change across the selected range is ≥ 0; red (`negative`) when < 0.** |
| Latest dot | 12 diameter at the last point, line color, centered on the right edge (it overhangs by 6). **Pulse:** a ring grows from the dot (scale about 1 → 2.2) and fades out (opacity about 0.5 → 0) over about 2s, looping; off under Reduce Motion. |
| Low / Peak markers | Dots 8.8 diameter, `text` color, at the min and max **within the selected range**. Label `small` 12/16 `secondaryText`, about 18 to the right of the dot and just above it ("Low" bottom at the dot center, "Peak" over the dot). **Text outline/shadow in `background`** so it stays legible over the line or fill. |
| Volume header | 18 below the line chart. "Daily volume" left (new Regular 12/16 slot, `secondaryText`); "through {last sales date}" right (same slot, `inactive`). 7 below it come the bars. |
| Volume bars | 362×about 52. One bar per day; 1M = 30 bars, pitch 12.07, bar 7.46, gap about 4.6. Baseline hairline at the bottom. **Dim bars by default; brighter inside the Avg-sale-price window; brighter still under the finger while scrubbing.** The highest-volume bar in range gets its count (`small` 12/16 `inactive`) about 3 right of the bar top. Dim and bright bar colors aren't bound to tokens in Figma: propose `border` (dim) and `secondaryText` (bright), with confidence scores. |
| X axis | 7 below the bars. 5 evenly spaced dates, `thumbLabel` 12/14 `inactive`. |
| Range toggle | 16 below the chart block. Centered `1W · 1M · 3M` with **1M the default**. Active pill: `border` fill, radius 8, padding 8×6, `pill` 13/14 `secondaryText`. Inactive: no fill, `inactive` text, padding 18×9. Hit areas ≥ 44. **Both charts follow the toggle.** |
| Stat grid | 24 below the toggle. 2×2 grid, column gap 8, row gap 12, right column 12 left padding. Each cell: padding 8 above and below, gap 6. Label is the new Regular 12/16 slot in `secondaryText`; value `itemTitle` 15/16 `text`; date `micro` 10/14 **`secondaryText`** (Figma binds `cardBackground`, which is wrong; see Q15), 4 below the value. **Not tied to the range toggle.** |

**Chat tab** (node `2758:88229`)

| Element | Spec |
|---|---|
| Thread | `DetailChatSection` / posts list, 27 below the tab bar. Thread layout is unchanged. |
| Chat entry | `JoinChatFooter placement="fixed"`, **only while Chat is the active tab**. 362×44 glass pill at the 20 inset, `EdgeFade` (162) behind it, `bodyInput` label. Label reads **"Start the chat"** when the entity has 0 posts and "Join the chat" otherwise (default while the count loads). This is the existing `SetScreen` / `SupportedTagValueScreen` `chatEntryLabel` pattern. |

**Trades tab** (node `2758:88920`)

| Element | Spec |
|---|---|
| Empty copy | 56 below the tab bar, 60 side padding inside 362 (text 242 wide), centered. "Be the first to make an offer" in `title` 16/16 `text`; 8 below it, "Buy or sell this card on Nanza and your offer will show up here." in `body` 14/22 `secondaryText`. Layer names carry older copy; **use the text content**. |
| CTAs (interim) | Figma's single outline "Trade" button (164×54, 1.1 `glassBorder`, radius 28, 24 below the copy) will eventually merge buy and sell. **For now: the existing Place bid → `CreateBid` / Sell → `CreateListing` pair**, on the Trades tab only: under the empty copy, and above the grid when there are results. |
| With results | Listings and bids in the 2-column grid (`ListingThumb` / `BidThumb`, the existing expanded-feed grid / FeedGrid treatment). No carousel, no "Trades" shelf heading. |

**Details tab**
- Description (`EntityDescriptionHtml`), the set row, the primary tag rows and the attribute tags, styled as they are today.
- **All shown in full:** no `ShowMoreButton`, no collapsible or measurer.

## Data mapping (from oak-cortex's TCG API client, for reference only)
oak-cortex (`scripts/tcgapi/lib/client.ts`, tables in `supabase/migrations/083_tcgapi.sql`) shows the
fields this UI needs. The dev team decides how they reach mobile.

| UI | Source |
|---|---|
| Market price | `market_price` |
| "updated …" | the time **we fetched** the data (`fetched_at` in oak-cortex), formatted like Figma ("Oct 19, 03:47 UTC") |
| 24H / 7D / 30D chips | `price_change_24h` / `_7d` / `_30d` (15.5 = +15.5%; null means not enough data, shown as 0%) |
| Line chart | daily history `market_price` by `date`. **1W** = last 7 daily points (the API has no week range), **1M** = last 30, **3M** = last 90 |
| Volume bars | daily history `sales_volume`; "through" = `sales_as_of` / last date with volume |
| At release | `Product.releaseDate` (`entity.product.releaseDate`, already in `@oakplatforms/types`) → the first history point on or after that date; the date label is the release date. No release date, or history that starts after it → "–" |
| Avg sale price | `avg_sales_price` over the last 7 days of sales; the label is that window ("Oct 7 – 13") |
| Low / Peak (stat grid) | min / max over **all available history**, with dates. Not range-bound. (The chart's Low/Peak markers *are* range-bound.) |

Coverage: about 10% of FAB entities don't map to the TCG API, and the API only keeps history for cards worth $1 or more.

## Code map
- `src/screens/entity/EntityScreen/index.tsx` — the screen. Today it renders everything stacked: title → CTA row → description/tags with Show more → Trades carousel/grid → `DetailChatSection`, plus the fixed `JoinChatFooter`. The header's `rightIcons` includes `cards-2`.
- `src/components/detail/DetailChatSection`, `src/components/posts/composer/JoinChatFooter.tsx` (`label` prop) — chat; see `SetScreen` lines 65-68 for the post-count label.
- `src/components/global/Tabs`, `TabPillsRow`, `src/styles/components/tabs.ts` — check these for a reusable segmented track before building a new one.
- `src/components/global/CardCarousel`, `ListingThumb`, `BidThumb`, `cards.cardGridRows` — Trades grid (the current `feedExpanded` branch).
- `src/styles/components/modals.ts` — `entityModal*` classes (TradeButtonsRow, InlineDetails…).
- `src/styles/components/typography.ts` — new Regular 12/16 +0.11 slot.
- `src/styles/tokens/{dark,light}.json` — new `positive` / `negative` (+ subtle) aliases.
- `src/components/global/Icon/iconMap.ts` + `assets/icons/` — new header icon, triangle up/down.
- `react-native-gifted-charts` is **not installed**. `react-native-svg`, `react-native-linear-gradient` and `react-native-reanimated` (pulse, scrub) already are.

## Open questions
1. Default tab on open?
   **Answer (Skylar, 2026-10-24):** Chat (first tab).
2. Place bid / Sell pills are hidden in every frame. Are they gone?
   **Answer:** they'll merge into one "Trade" button, but the buy/sell trade screen route doesn't exist yet. For now, keep the two existing buttons (Place bid → CreateBid, Sell → CreateListing) **on the Trades tab only**: in the empty state in place of "Trade", and above the grid when there are results.
3. Sticky tab bar?
   **Answer:** yes, sticky.
4. Details tab: keep "Show more"?
   **Answer:** no, show everything in full.
5. Do the 24H / 7D / 30D chips follow the range toggle? What shows when there's no change?
   **Answer:** fixed, independent of the toggle ("recent momentum", the Robinhood/StockX/TCGplayer convention). A 3M chip or a range-change readout can come later. 0% (or no data) shows "0%" on a `border`-gray pill with `secondaryText`, no triangle.
6. Up/down colors: new tokens, or `conditionColors`?
   **Answer:** add semantic aliases **`positive` / `negative`** (plus `positiveSubtle` / `negativeSubtle` 16% tints for the chip fills) that **point at `conditionColors.mint` / `.damaged`**, with no new hex values. They mean "direction of change", so any delta can reuse them. They stay separate from `success` / `error` (action outcomes, about 35 style uses, untouched) and keep charts decoupled from the card-condition palette. Needs light-theme values too.
7. Line color rule?
   **Answer:** green when the selected range's change is ≥ 0 (flat stays green), red when < 0. The fill matches the line.
8. Stat grid: range-bound? Where does "at release" come from?
   **Answer:** separate from the range. "At release" = price data matched to the product's release date (`Product.releaseDate` exists; it may also live on the set, but should be on the entity). No date or no matching data → "–". Low and Peak in the grid are all-time over the available history.
9. Volume bar highlighting?
   **Answer:** both: bars in the Avg-sale-price window are brighter at rest, the scrubbed bar brightens while scrubbing, and the highest-volume bar in range carries its count.
10. Pulse on the latest dot (ring scale + fade, about 2s loop, off under Reduce Motion)?
    **Answer:** yes.
11. Scrub behavior?
    **Answer:** tap and hold on either chart scrubs **both in sync**, with a vertical hairline (`secondaryText`) at the same date on both. Each chart gets its own bubble: price + date on the line chart, volume + date on the bars. Bubble: glass surface (`glass` fill, `glassBorder` stroke, rounded) using existing type slots, contrasting with the chart. **The big Market price follows the finger** and snaps back to latest on release. **Haptic tick per day only if the dev team confirms a haptics library** (none is installed; see D5).
12. Empty states?
    **Answer:** two tiers. (a) Price but no history (under $1): price, chips and stats as usual; the charts and toggle are replaced by one muted line, "Not enough sales to chart yet"; At release / Low / Peak show "–". (b) No market data at all (unmapped): the Market tab shows the Trades-style empty block ("No market data yet" / "We don't have pricing for this card yet."), no button. Tabs never hide.
13. Timestamp?
    **Answer:** the time the data was fetched.
14. Fixed footer on other tabs?
    **Answer:** no. Only Chat pins its entry for now; a pinned Trade button may come later.
15. The stat dates are bound to `cardBackground`.
    **Answer:** don't introduce a different color. `cardBackground` = `secondaryText` = `neutral.200` in dark, but in light `cardBackground` is `neutral.50` (near white), which would make the dates unreadable. Use **`secondaryText`**: identical in dark, correct in light, no new token.

**For the dev team** (Skylar: "leave those for the dev team")
- **D1. Data path.** Mobile should not call the TCG API directly (that would expose the key). Options: nanza-api proxies it, or reads oak-cortex's `tcgapi_*` tables. The DTO must come from `@oakplatforms/types` (CLAUDE.md P0 #11: no local domain types).
- **D2. Add `react-native-gifted-charts`?** Its peers (`react-native-svg`, `react-native-linear-gradient`) are already installed. Check that it supports synced scrub across two charts, pinned labels and a custom dot (pulse). Otherwise build on `react-native-svg` directly.
- **D3. Query freshness.** The data updates nightly, which suggests `STALE_TIME.LIVE` rather than `INSTANT`. Decide the query key.
- **D4. Entity → TCG API mapping.** One entity per printing (oak-cortex's `tcgapi_card_map`). Which side resolves it?
- **D5. Haptics.** No library is installed. Is adding one (e.g. `react-native-haptic-feedback`) OK? If not, ship the scrub without haptics.

## Prompt
```text
Implement the entity screen tabs + Market tab from Figma. Plan first, then build.

PREREQUISITE: the dev team has already put the market data on the entity screen (plain text).
Find that hook/query and its DTO and build on them. Don't add fetching, and don't define local
domain types (P0 #11). If the data or its type isn't there, STOP and ask.

Read this handoff first: nanza-vault/nanza-figma/handoffs/2026-10-24-entity-tabs-market.md.
Its Measurements, Data mapping and Open questions (all answered, plus D1–D5 for the dev team)
are the spec. Confirm numbers with get_metadata / get_design_context on:
- Market tab  2751:65639 (price block 2754:66306, chart card 2776:89658, tab bar 2754:66352)
- Chat tab    2758:88229 (footer 2783:122346)
- Trades tab  2758:88920 (empty copy 2768:89643, button 2889:127015)
- New header icon: instance 2798:123555
File 7c3Zz4nDUOutQG1BsnRRAw. Layer names are stale in places ("subtitle · Market price" on every
stat label, "pill · Month" on every range label, "Daily sales"). Trust the text content. Ignore
hidden layers (proposed.detailTabs underline variant, proposed.printingPill, "fixed buy / sell").

SCOPE: entity screen only. The title block, hero, header behavior, chat thread layout and the
Trades thumbs are unchanged (title spacing was checked and matches the code).

1. Tabs. Segmented bar (Chat · Market · Trades · Details), 32 below the title block, sticky
   under the floating header, Chat active on open. DRY-check Tabs / TabPillsRow / tabs.ts first.
2. Chat tab: DetailChatSection as today. JoinChatFooter fixed ONLY while Chat is active.
   Label "Start the chat" at 0 posts (SetScreen chatEntryLabel pattern), else default.
3. Trades tab: 2-column ListingThumb/BidThumb grid (today's expanded-feed grid, no carousel,
   no heading). Empty → Figma copy block. Interim CTAs: the existing Place bid → CreateBid and
   Sell → CreateListing pair, Trades tab only (under the empty copy / above the grid). They
   replace Figma's single "Trade" button until the buy/sell trade screen exists.
4. Details tab: description + set row + primary tag rows + attribute tags, styled as today,
   always fully expanded. Delete the ShowMore / collapsible / measurer path.
5. Market tab, top: price block, 3 fixed change chips (24H/7D/30D, not range-bound),
   attribution line with the FETCH time. Exact numbers in the Measurements table.
6. Market tab, charts: line chart + daily volume bars driven by a 1W/1M/3M toggle (1M
   default). Line + fill color by the selected range's change (≥0 positive, <0 negative).
   Latest dot pulses (reanimated, about 2s, off under Reduce Motion). Low/Peak markers within the
   range, labels with a `background` outline/shadow. Bars: dim by default, brighter in the
   avg-sale window, brightest under the scrub; max bar labeled. Tap-and-hold scrubs BOTH charts
   in sync: shared hairline, a glass bubble per chart (price+date / volume+date), and the big
   price follows the finger, snapping back on release. Haptics only if D5 is approved.
   Chart library per D2. If gifted-charts can't do synced scrub + custom dots cleanly,
   say so in the plan with confidence scores rather than forcing it.
7. Market tab, stat grid: At release / Avg sale price / Low / Peak per the Data mapping
   table, NOT range-bound, "–" when missing.
8. Empty states: tier (a) no history → charts + toggle become "Not enough sales to chart
   yet", At release/Low/Peak "–"; tier (b) no data → Trades-style empty block.
9. Header: replace the `cards-2` circle icon with the new icon (export SVG from 2798:123555,
   add to iconMap). Same onPress (AddToCollection / AuthModal).

Tokens / type:
- Add `positive` / `negative` / `positiveSubtle` / `negativeSubtle` as aliases of
  conditionColors.mint / .damaged (16% tints for Subtle) in dark.json AND light.json. No new
  hex. Leave success/error alone.
- Add ONE typography slot: Roobert Regular 12/16, +0.11, default secondaryText (stat labels,
  "Market price", "Daily volume"). Everything else maps to existing slots: large, legal,
  pill, small, micro, thumbLabel, itemTitle, title, body, bodyInput.
- Stat dates use secondaryText (Figma's cardBackground binding is wrong in light mode).
- Triangle up/down glyphs (9×8) need SVG exports too.

Traps:
- Adding color names may need ThemeTokens in src/styles/utils/types.ts, and possibly
  resolveThemeColors for alias-with-alpha. src/styles/utils/* is escalation-only (P0 #7).
  Escalate with options.
- Over 5 files touched, so plan for the CLAUDE.md escalation. Suggest a component split (e.g.
  src/components/entity/market/*) and a style bucket in the plan; don't put chart styles in
  modals.ts.
- Sticky tabs: the tab bar sits inside entityModalSheetFlow under a fixedBackground hero and a
  floating header. Pin offset = header height. Switching tabs while pinned must keep the bar
  pinned (don't jump the scroll to the top of the hero).
- Keep the feedReady / InteractionManager deferral for chat + trades fetches. Charts can mount
  lazily on first Market visit.
- scrollToFeedItem / Sell Now–Buy Now (SHOW_FAIR_MARKET, off) now targets the Trades tab.
  Switch tab first, then scroll, or flag it if not worth keeping.
- Fixed footer + sticky bar + scrub gesture versus the vertical dismiss gesture and the
  ScrollView: the scrub must claim the touch only after a long-press so scrolling still works.
- Figma's active range pill is smaller (33×26) than the inactive hit areas. Keep ≥44 hit
  targets.
- Update modals.ts comments and the entity-screen solution design to match what ships.

House rules (CLAUDE.md): plan first with confidence scores per item; proportional sizing
(fractions of the 402 frame / 362 content width, e.g. chart heights); no inline styles, no raw
sizes, theme tokens + typography slots only; query freshness rules for any new keys. Run
/ui-review before finishing. When done, set this handoff's status to built and add a Log line.
```

## Log
- 2026-10-24 — handoff written; questions 1–15 answered by Skylar; D1–D5 left for the dev team
