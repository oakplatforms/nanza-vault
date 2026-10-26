---
tags: [nanza-api, nanza-mobile, solution-design, plan, pricing]
---

# TCG API — Market tab + scheduled price refresh (Plan)

> **Status: plan, 2026-10-24. Spike done; backend built 2026-10-24 and reworked the same day
> to store nothing of TCG API's (uncommitted): migration `20261024120000_product_external_priced_at`
> (Skylar runs), types `0.1.125` (to publish), `src/utils/tcgApi.ts`, `src/utils/tcgApiMatch.ts`,
> `src/services/market.ts`, `src/validation/market.ts`, `GET /entity/:id/market`,
> `migrations/orchestration/generate_tcgapi_prices_sql.py` + `tcgapi_common.py`. Key `tcgApiKey`
> is in `nanza-credentials-dev/prod` (Skylar). **Mobile built 2026-10-24 (uncommitted, pending
> types `0.1.127` publish + `types:update` + device QA):** `EntityTabs`, `EntityMarketTab`,
> `useFetchEntityMarket`, `styles/components/market.ts`, `react-native-gifted-charts@^1.4.81`;
> `MarketDto` lives in the types package's hand-maintained `shared.ts`.**
> Spans nanza-api (TCG API client + matcher, `GET /entity/:id/market` proxy, refresh script) and
> nanza-mobile (four-tab card screen + Market tab with chart). Flesh and Blood only. Fold into
> [[card-pricing|Card Pricing]] (scheduled refresh) and a mobile entity-screen design when it lands.

## The ask (Skylar, voice, 2026-10-24)

- **Card screen tabs.** On the entity / product screen, a row of four tabs under the display
  name and set name: **Chat · Market · Trades · Details**. Styled like the Trade screen's pill
  tabs, but inline under the title, not pinned to the top of the screen. Tapping a tab swaps the
  content, so chat and trades stop being stacked down one long page.
- **Market tab = on-demand fetch.** Opening Market calls TCG API (through our API) with a loading
  state and shows fair market price plus charts and price history.
- **Scheduled price refresh.** An orchestration shaped like the FAB card migrations: bulk calls
  to TCG API, then generated SQL that updates `Product.price`, run in DBeaver every few days.
- **Our number is the join key.** We keep our own card data; TCG API is a reference. Our
  `Product.number` + `Set` name are matched onto their listings whenever we need a price.
- **Token on the server.** The app calls a nanza-api endpoint, nanza-api calls TCG API and
  reshapes the payload into our own small schema, and the key never ships in the app.

## What TCG API gives us (docs read 2026-10-24)

- Base `https://api.tcgapi.dev/v1`, auth header `X-API-Key: tcg_live_…`. The data comes from
  TCGplayer (images on the TCGplayer CDN, `tcgplayer_id` on every card).
- **Flesh and Blood = game slug `flesh-and-blood-tcg`, id `64`**: 111 sets, 10,955 cards, priced
  daily. Checked live against the public `GET /games`.
- **FAB printings** (exact names for `?printing=`): Normal, Rainbow Foil, Cold Foil, 1st Edition
  Normal / Rainbow Foil / Cold Foil, Unlimited Edition Normal / Rainbow Foil.
- **Card fields that matter:** `id` (their key), `number` (collector number, a string),
  `tcgplayer_id`, `set_id`, `name`, `rarity`, `image_url`, `tcgplayer_url`.
- **Price fields:** `market_price`, `market_price_as_of`, `low_price`, `median_price`,
  `lowest_with_shipping`, `buylist_price`, `price_change_24h/7d/30d`, `no_market_price_since`.
- **History** `GET /cards/:id/history?range=month|quarter|year|all&printing=` returns rows of
  `date, printing, market_price, low_price, avg_sales_price, sales_volume`, oldest first. It's
  on every tier, but retention depends on plan: Free 7 days, Hobby 30, Starter 90, Pro full.

**Plans** (daily quota, resets at midnight UTC; `429 RATE_LIMIT_EXCEEDED` with `Retry-After`):

| Tier | $/mo | Req/day | What it unlocks for us |
|---|---|---|---|
| Free | 0 | 100 | Spike only |
| Hobby | 9.99 | 1K | 30-day history |
| Starter | 19.99 | 2.5K | 90-day history, bulk resolve (100/batch) |
| **Pro** | **49.99** | **10K** | **Full history, `GET /sets/:id/prices`, `GET /bulk/prices` (500 ids)** |
| Business | 99.99 | 50K | Bulk history, set export |

**Recommendation: Pro.** One `GET /sets/:id/prices` call per set refreshes the whole FAB
catalogue in about 111 calls. Full history makes the year and all-time charts real, and 10K/day
is plenty for on-demand Market opens (two calls each, no cache).

## The mapping problem

- **No lookup by collector number.** `/search` matches names only, and `/cards/:id` needs their
  id. The path is to list each set's cards (`GET /sets/:id/cards`, 100 per page) and match on
  `number`.
- **Association only, no card data (Skylar, 2026-10-24).** We have our own card data. Images,
  names, rarity and attributes from TCG API are never stored or shown. Their card list is read
  only to find the match; the only things we keep are their card id + printing, and the prices.
- **Our number encodes the version.** `Product.number` = optional `U-` prefix + base collector
  number + optional finish suffix. Examples (made up, but the real shape): `MON123`,
  `MON123-RF`, `MON123-CF`, `MON123-MV`, `MON123-GF`, `U-MON123`, `U-MON123-RF`. Parse it into
  three parts:

  | Token | Meaning |
  |---|---|
  | `U-` prefix | Unlimited edition |
  | `MON123` | base collector number (matched against their `number`) |
  | `-RF` | Rainbow Foil |
  | `-CF` | Cold Foil |
  | `-GF` | Gold Foil |
  | `-MV` | Marvel |
  | `-EA` | Extended Art (separate product, name `(Extended Art)`) |
  | `-AA` | Alternate Art (separate product, name `(Alternate Art)`) |
  | no suffix | Normal (non-foil) |

  Anything else (`-TP`, `-A/-B/-C`, an unknown token) is reported as unmatched, never guessed.
- **Version → TCG API printing.** TCG API's FAB printing names fold edition into the printing:

  | Our number | Sets with editions (WTR, ARC, CRU, MON …) | Sets without editions |
  |---|---|---|
  | `MON123` | `1st Edition Normal` | `Normal` |
  | `MON123-RF` | `1st Edition Rainbow Foil` | `Rainbow Foil` |
  | `MON123-CF` | `1st Edition Cold Foil` | `Cold Foil` |
  | `U-MON123` | `Unlimited Edition Normal` | — |
  | `U-MON123-RF` | `Unlimited Edition Rainbow Foil` | — |

  The matcher looks at the printings a card actually has and picks the edition form when it
  exists, and the plain form otherwise.
- **Gold Foil and Marvel aren't printings.** Neither appears in TCG API's FAB printing list, so
  TCGplayer almost certainly lists them as separate products. That means the same collector
  number can appear more than once in a set listing, told apart by name, e.g. "… (Marvel)".
  For `-MV` / `-GF` the matcher picks the separate product whose name carries that treatment
  and uses its single printing. The spike confirms the exact naming.
- **So one of our products = (TCG card id, printing),** resolved on the fly and never stored.
  The base number finds the card(s) within the matched set, the suffix picks Marvel / Gold Foil
  products or the foil printing, and `U-` picks the Unlimited edition printing. If more than one
  candidate is left, it's reported as ambiguous (`skipped` in the refresh, `available: false` on
  the Market tab).
- **Sets match by exact name.** Our `Set.displayName` (then `name`) is matched case-insensitively
  to their set name. Watch the known splits from the FAB migrations: promos vs Prism,
  `Set.code` not unique.
- **Bulk prices may be too coarse for FAB.** `GET /bulk/prices` returns only `market_price` and
  `foil_market_price`, so it can't tell Rainbow from Cold Foil. Prefer `GET /sets/:id/prices`
  (per-printing, per set) or `GET /cards/:id/prices` for the refresh; confirm in the spike.

## Store nothing of theirs (Skylar, 2026-10-24, on review)

- **No TCG-specific columns and no cache table.** The first build added `Set.tcgApiSetId`,
  `Product.tcgApiCardId / tcgApiPrinting / tcgApiPricedAt` and a `MarketCache` table. All of them
  were removed on review.
- **Match at request time instead.** The same pure matcher (`src/utils/tcgApiMatch.ts`, mirrored
  in `migrations/orchestration/tcgapi_common.py`) maps our `Set.name` / `displayName` → their
  set (exact, promo pinned) and our `Product.number` → their (card, printing), every time it's
  needed. Nothing is persisted, so nothing has to be kept in sync on a new drop.
- **One generic column: `Product.externalPricedAt DateTime?`.** It's stamped whenever `price` is
  set from an outside source (today: the TCG API refresh). It drives "updated <date>" and is null
  for hand-priced products. The name stays source-agnostic on purpose.
- **Cost of not storing the match:** a Market open is two keyed calls (`/sets/:id/prices` for
  the match + current price, `/cards/:id/history?range=all`) plus the public set list, memoized
  per warm Lambda for 6 h. That's fine on Pro's 10K/day.

## Spike results (2026-10-24, Skylar's Pro key, 7 calls)

- **Our numbers match theirs exactly.** `number` = `MON003`, the same format as our base
  number. Double-faced cards come back as `MON002 // MON001` (32 of them in MON) and need a
  split on `//` (or `/`) before matching.
- **Editions are printings inside one set.** There's no separate "Monarch Unlimited" set. MON
  printings seen: `1st Edition Normal`, `1st Edition Rainbow Foil`, `1st Edition Cold Foil`,
  `Unlimited Edition Normal`, `Unlimited Edition Rainbow Foil` (plus plain `Normal` on sealed).
  This confirms the `U-` / suffix → printing table above.
- **`GET /sets/:id/prices` is the one endpoint we need for both mapping and refresh.** One call
  returned all 1,185 MON rows: one row per (card, printing), with `card_id`, `name`, `number`,
  `printing` and all price fields. It used **1 request** of the 10,000/day quota. So mapping
  the whole FAB catalogue and each full refresh are each about 111 calls (one per set).
  `GET /sets/:id/cards` is less useful: it returns one row per card with a single printing, so
  it's not used.
- **Same number, different product = an art treatment in the name.** In MON, three numbers have
  two products, e.g. `Herald of Erudition` vs `Herald of Erudition (Extended Art)`, both
  `MON004`. Also seen: `(Alternate Art)`. Pitch is in the name too (`(Red)`), but pitch
  variants have their own numbers, so it doesn't matter for matching. There's no Marvel in MON.
  Marvel / Gold Foil naming (`(Marvel)`?) still needs checking on a newer set; the first full
  refresh review will show it as a `skipped` reason.
- **Sealed products** sit in the same set with `number: null`. Skip them.
- **History** (`/cards/:id/history?range=month`): 31 daily rows with `market_price`,
  `low_price`, `avg_sales_price`, `sales_volume`. On no-sale days avg and volume are `0`, not
  null.
- **Promos** live in one set, "Flesh and Blood: Promo Cards", which lines up with our FAB Promo
  Set.
- **Art suffixes (Skylar, 2026-10-24):** our numbers also use `-EA` (Extended Art) and `-AA`
  (Alternate Art). These map to the separate product whose name ends `(Extended Art)` /
  `(Alternate Art)`, the same way `-MV` / `-GF` are matched by name.
- **Why not the bulk endpoints (2026-10-24).**
  - `GET /bulk/prices` returns one row per card with only `market_price` +
    `foil_market_price`. It can't tell 1st Edition from Unlimited, or Rainbow from Cold Foil,
    so most of our numbers would get the wrong price.
  - `POST /bulk/resolve/tcgplayer` needs TCGplayer ids, which we don't have.
  - `GET /sets/:id/prices` gives every printing with its `number`, at 1 request per set. It's
    both more precise and cheap enough (about 111 calls for all of FAB).
- **Set matching is exact, not fuzzy.** Our promo set (`Set.name = 'promo'`) is pinned in code
  to their "Flesh and Blood: Promo Cards". Any other set with "promo" in its name never matches;
  its products show up as `skipped: set not matched` in the refresh review.

## Plan

### Phase 0 — Spike (done 2026-10-24, see results above) (Skylar's key, about 10 calls, Free tier is enough)

Run with `TCGAPI_KEY` exported in the shell. Nothing is stored.

1. `GET /sets?game=flesh-and-blood-tcg&per_page=100` → compare set names against our FAB `Set`
   rows.
2. `GET /sets/<MON id>/cards?per_page=100` → confirm the `number` format (`MON123`?), whether
   rows carry prices, and how Marvel and Gold Foil products are named. Also check whether 1st
   Edition and Unlimited are printings of one card or (for MON / WTR / ARC / CRU) separate
   TCGplayer sets ("Monarch" vs "Monarch Unlimited"). If they're separate sets, `U-` picks the
   Unlimited set instead of a printing.
3. `GET /cards/<id>/prices` on a card with several printings → confirm the printing names.
4. `GET /cards/<id>/history?range=month` → confirm the history shape.
5. Check `rate_limit.daily_remaining` after each call to learn the real credit cost.

The outcome decides the column shape below and which price endpoint the refresh uses.

### Phase 1 — Schema + where the code lives (nanza-api)

- **Schema:** only `Product.externalPricedAt DateTime?` (migration
  `20261024120000_product_external_priced_at`, Skylar runs it), types bump.
- **No mapping step.** The refresh and the Market tab both match at runtime (see "Store nothing
  of theirs").
- **Where it lives (same split as the card migrations and card pricing):**
  - **Code in nanza-api, committed:** `migrations/orchestration/generate_tcgapi_prices_sql.py`
    + `tcgapi_common.py` (Python, beside `generate_pricing_sql.py`); `src/utils/tcgApi.ts`,
    `src/utils/tcgApiMatch.ts`, `src/services/market.ts` for the API.
  - **Run output, gitignored:** `migrations/output-sql/<release>/tcgapi-prices/` (`review.csv`,
    batched SQL, a `cache/` of that run's raw TCG responses so a rerun of the same release is
    free). Input CSVs go in `migrations/import-csv/`.
  - **Vault:** this design plus the methodology (number grammar, printing table, flag and skip
    rules) only. No code.

### Phase 2 — Scheduled price refresh (nanza-api, operator-run)

Two steps with a human gate, so a bad fetch can't turn a $1.40 card into $75.40 overnight
(Skylar, 2026-10-24):

1. **Baseline export.** Right before each run, Skylar exports every FAB product with its number,
   set name and **current `Product.price`**, which is the last run's price (query at the top of
   the script).
2. **`review`.** Match each product's set (exact name, promo pinned) and number onto the TCG
   listings, fetching `GET /sets/:id/prices` per matched set (about 111 calls; per-card would be
   about 10K). Then compute market × 0.90, compare against the baseline, and write `review.csv`:
   - `ok`: normal move, pre-approved.
   - `new`: no baseline price, pre-approved.
   - `flagged`: big move, approve left blank. Flagged when **≥ 50 % and ≥ $2**, or **≥ $25**
     whatever the percent. The $2 floor keeps penny-card noise out. Tunable with `--pct`,
     `--min-delta`, `--max-delta`.
   - `skipped`: set or card not matched (with the matcher's reason), no market price, TCGplayer
     stopped pricing it, or a market price older than 14 days. Never applied.
   Flagged rows are listed first, biggest dollar move on top.
3. **Confirm.** Skylar sets `approve=yes` on the flagged rows they accept, or types a
   different `new_price`.
4. **`apply`.** Generates the batched `UPDATE "Product" SET price, externalPricedAt` SQL from
   `review.csv`, for `approve=yes` rows only. Rows that aren't approved keep their old price
   and get flagged again next run against the same baseline.

Offline check (2026-10-24, MON spike data): $1.40 → $28.01 and $0.10 → $3.78 were flagged and
held back, normal moves and first prices went through, and an approved flagged row plus a
hand-edited price both made it into the SQL.

Later, once the numbers are trusted, this can become a scheduled Lambda that writes directly
and holds only the flagged rows.

### Phase 3 — Market proxy endpoint (nanza-api)

- `src/utils/tcgApi.ts`: a thin client with base URL, `X-API-Key` from `getSecrets()`
  (`tcgApiKey`), an 8 s timeout, and errors as `TcgApiError`. It has three calls: the FAB set list
  (public, memoized 6 h per warm Lambda), `/sets/:id/prices`, and `/cards/:id/history?range=all`.
- `src/utils/tcgApiMatch.ts`: the pure set + number matcher (mirrors the Python one; both give
  identical results on the MON spike data).
- `src/services/market.ts`: entity → `Product.number` + `Set` name → matched set →
  `/sets/:id/prices` (match + current price + 24h/7d/30d change) → that card's full history,
  filtered to the printing and sliced to the range by date (week 7 d, month 30, quarter 91,
  year 365, all). No DB writes, no cache.
- **What the mock needs (Skylar, 2026-10-24; styling comes from Figma over the MCP):** daily
  sales volume, average sales price, peak, low, at-release price and 24h / 7d / 30d change, over
  ranges 1W · 1M · 3M. All of it comes from TCG API; peak / low / average / volume are computed
  by our service.
- `MarketDto`: `{ available, range, printing, marketPrice, lowPrice, medianPrice, change24h,
  change7d, change30d, asOf, stats: { peak: {price, date}, low: {price, date},
  atRelease: {price, date}, avgSalesPrice: {price, from, to}, totalVolume, salesThrough },
  history: [{ date, marketPrice, lowPrice, avgSalesPrice, salesVolume }] }`.
  - **Peak / low** are taken over the selected range.
  - **avgSalesPrice** is volume-weighted over the last 7 days that have sales data (the mock's
    "Oct 7–13").
  - **salesThrough** is the last day with sales data (the mock's "through Oct 13"). Sales lag
    prices by 1–3 days.
  - **atRelease** is the first priced day of the full history (the history call is always
    `range=all`, so it costs nothing extra).
- **Data first:** a debug render of the raw `MarketDto` JSON in the Market tab ships behind
  the spinner before the chart, so the payload can be checked on device against the Figma mock.
- `GET /entity/:id/market?range=week|month|quarter|year|all` in `entity.ts`, signed-in only
  (guest tokens rejected). A card that doesn't match returns `{ available: false }` (200); a TCG
  failure returns 502 "Market data is not available right now." Both render the quiet live-slot
  message on mobile.

### Phase 4 — Card screen tabs + Market tab (nanza-mobile)

**Mocks (Skylar, 2026-10-24; Figma frames to come over the MCP):**

- **Tab row:** directly under the display name / number / set lines. One rounded outlined
  container with four equal-width tabs, **Chat · Market · Trades · Details**. The active tab is a
  filled darker pill and the others are plain text. It isn't pinned to the top of the screen; it
  scrolls with the page under the hero card.
- **Chat tab** (default): the post list (author, age, body, reply count + heart) with the
  "Join the chat" footer pinned at the bottom, i.e. today's `DetailChatSection` +
  `JoinChatFooter`.
- **Market tab, top to bottom:**
  1. "Market price" label, then the price in large type ($12.33). Per decision 5, a subtle
     "Pricing from TCG API · updated <date>" sits under it; it isn't in the mock but was agreed.
  2. Three change chips in a row: arrow + % + period (`24H`, `7D`, `30D`). Down = red ▼ with
     red text, up = green ▲. Each is a rounded chip with a tinted fill.
  3. A price line chart, green line with a fading green gradient fill and no axes. Dots +
     labels at **Low** and **Peak**, and a dot at the latest point.
  4. "Daily volume" label left, "through <salesThrough>" right. A thin grey bar chart, one bar
     per day; the tallest bar is highlighted (lighter) with its count above it ("59"). Weekly
     x-axis date labels (Sep 19, Sep 26, Oct 3 …).
  5. Range pills **1W · 1M · 3M**, with 1M active (filled pill). There's no "since release"
     pill in the mock, so the API's `all` / `year` go unused for now.
  6. A 2×2 stats grid: **At release** $7.47 / Sep 19 · **Avg sale price** $9.44 / Oct 7 – 13 ·
     **Low** $2.13 / Sep 27 · **Peak** $14.77 / Oct 16. Label small and grey, value large,
     date small under it.
- The spinner sits under the top block (1) while the request loads. Errors and unmapped cards
  replace sections 2–6 with a quiet message.


- `EntityScreen`: a tab row directly under the display name + set chip, reusing the Trade
  screen's `tradePillTab*` styles. Order: **Chat · Market · Trades · Details**.
  - **Details**: today's image-adjacent block (prices, tags, description, buttons).
  - **Chat**: `DetailChatSection` + `JoinChatFooter`.
  - **Trades**: the listings/bids feed (keeps its deferred fetch).
  - **Market**: new.
- **Instant range switching (2026-10-24):** the Market tab makes one request (`range=quarter`)
  and cuts 1W / 1M / 3M from that history on the device; Low and Peak are recomputed per pill,
  and the other stats come from the response. Switching pills makes no request and shows no
  spinner, which fixed a jump where every pill change reloaded the whole live section.
- **As built (2026-10-24):** the tab reads **Trade** (not Trades), and so do its shelf heading
  and See-all title. Place bid / Sell sit at the **top of the Trade tab**, above the feed (Skylar:
  parked there while the layout evolves; briefly above the tab row first). The
  market price row stays in the identity group behind `SHOW_FAIR_MARKET` (on since 2026-10-24,
  labelled "Market price"). A price-row tap
  switches to Trades and scrolls to the item. `JoinChatFooter` and `DetailChatSection` render
  only on Chat. The line goes `success` green when the range ends at or above where it started,
  otherwise `error` red. Volume axis labels are every 7 days on 1M, every 2 on 1W, and every 21
  on 3M.
- **Default tab Chat.** The Market tab renders only when signed in.
- **Market tab:** the top layer reads `product.price` + `product.externalPricedAt` from the entity
  detail ("Market price", big number, "Pricing from TCG API · updated <date>"). Below it,
  `useFetchEntityMarket(entityId, range)` (react-query, `enabled` only while the tab is active)
  shows `LoadingIndicator` while fetching, then 24h / 7d / 30d change, range pills (1W · 1M ·
  3M · Since release), a price line chart, a daily-volume bar chart, and peak / low / avg sale
  stats. On error or `available: false`, a quiet message replaces the chart slot.
- **Chart library: `react-native-gifted-charts` (recommended).** TCG API has no chart guidance
  (their docs and guides only cover the data). Gifted Charts needs only `react-native-svg` and
  `react-native-linear-gradient`, and both are already installed, so there's no new native
  module and no pod/gradle churn. It does line (with gradient area fill and a press-and-drag
  pointer/tooltip) and bar charts, which covers price + volume. Alternatives considered:
  - **victory-native (XL):** good, but it pulls in `@shopify/react-native-skia`, a heavy new
    native dependency, just for one tab.
  - **react-native-wagmi-charts:** made for price lines, but it's tied to Reanimated
    internals, and we're on Reanimated 4 / RN 0.83.
  - **Hand-rolled `react-native-svg`:** no dependency, but we'd own the scales, the pointer
    and the tooltip. Fallback if Gifted Charts doesn't match the Figma.
  Confirm the installed version builds on RN 0.83 before committing to it.

## Token onboarding

- **Scripts / spike:** `export TCGAPI_KEY=tcg_live_…` in the shell; scripts read
  `process.env.TCGAPI_KEY`.
- **API (dev/prod):** the `tcgApiKey` field in the `nanza-credentials-<stage>` secret (added by
  Skylar 2026-10-24), read through `getSecrets()` at runtime, not at build time. Claude never reads or writes the secret.
- **Local API:** whatever `npm run dev` already uses for secrets. Confirm before Phase 3.

## Build notes (2026-10-24)

- **Cloudflare blocks Python's default User-Agent** (403, `error code: 1010`). The scripts send
  `User-Agent: nanza-tcgapi-scripts/1.0`. Node `fetch` on the Lambda is not blocked.
- History is fetched without `?printing=` and filtered by our printing name in the service,
  because the docs only promise `Normal|Foil` for that filter.
- **Reworked on review (2026-10-24):** the TCG-specific columns, `MarketCache` and the separate
  mapping script (`generate_tcgapi_mapping_sql.py`) were removed. Matching now runs at request /
  run time.
- Guests: the endpoint rejects both no-auth and `role: 'guest'` tokens (guest tokens do set
  `req.user`).
- `GET /entity/:brandSlug/:entityName` was **retired 2026-10-24**: it would have shadowed
  `/entity/:id/market`, and its only caller (nanza-web-app `EntityDetail` via `SmartRouter`) sits
  behind a commented-out route. Mobile and admin only use `/entity/:id`.
- Offline check of the matcher on the spike's MON data: `MON003`, `U-MON003-RF`, `MON003-CF`,
  `MON004-EA` and `MON155-AA` all map correctly. `MON003-RF` (no such printing) and
  double-faced `MON002` (two products) come back unmatched as intended. The TypeScript and Python
  matchers return identical results.
- `MarketDto` is not in `@oakplatforms/types` (that package is Prisma-generated only), so
  mobile declares it locally, mirroring `src/services/market.ts`.

## Figma handoff review (2026-10-24, later)

The design handoff `nanza-figma/handoffs/2026-10-24-entity-tabs-market.md` was reviewed against
the build; it now carries a **Dev status** section that overrides its older text. Decisions made
in that review:

- **Top price on the Market tab = TCG `marketPrice`** with "Prices from TCG API · updated
  `fetchedAt`" — supersedes decision 5's stored-price top block, so the number at rest, the chart
  and the scrub readout all agree. Our ×0.90 `Product.price` stays the Fair Market pill;
  `externalPricedAt` is kept for ops but unused on this tab.
- **Stat grid is all-time, not range-bound:** `stats.low/peak` are over the full history, and
  `stats.atRelease` is the first priced day on/after `Product.releaseDate` (null without a date,
  or if history starts after it). Chart Low/Peak markers stay range-bound, computed on device.
  `MarketDto` gained `fetchedAt` and the diagnostic `reason`.
- **Sticky tab bar: yes** (the handoff's Q3). Not built yet; the handoff prompt does it.
- **Matcher fixes from live traffic:** (1) a suffix-less number whose only product is a
  treatment product (The Hunted's Marvels, `HNT262 "… (Marvel)"`) now matches it; (2) a token /
  back face on several double-faced products (`MON155`: "Chane // Galaxxi Black" vs "Galaxxi
  Black // Soul Shackle") prefers the product where our number is the front face. TS and Python
  matchers verified identical on the MON data.
- **Trade requests are lazy:** the public listings/bids feed and the user's own bid/listing
  (CTA labels) fetch only once the Trade tab is open (`tradeReady`); they were all firing on
  first open.

## Resolving a card without our set names (2026-10-24, later still)

Live traffic showed `AHA027: set not matched (name 'hala-bladesaint-of-the-vow', displayName
'Hala, Bladesaint of the Vow')` — our set is the hero, theirs is "Armory Deck Origins: Hala". Our
set names will never reliably equal theirs, so neither path depends on them any more:

- **Market tab (per card): name search first, set listing as fallback.** TCG has no lookup by
  collector number, but `GET /search?q=<card name>&game=flesh-and-blood-tcg` returns every card
  with that name — all pitches, treatments and reprints — each with its `number`. The pipeline is
  `pickCard` (number + treatment suffix + set-name tie-break) → `pickPrinting` on
  `GET /cards/:id/prices` (`-RF` / `-CF` / `U-` → the printing) → `pickAmong` (front-face, then
  most-listed) when several cards had the printing. Up to four candidate cards get priced
  (`MAX_CARD_CANDIDATES`). The search term is the display name before any `(` or `//`. If the
  search finds nothing, the old set-name path runs. Cost: 1 search + 1–4 prices + 1 history.
- **Daily run (per set): every FAB set, one global index.** `generate_tcgapi_prices_sql.py`
  pulls all ~111 listings regardless of our sets and matches each number across them; our set
  name only breaks a tie when two TCG sets carry the same number (a Blitz deck reprint of a
  Monarch card). "set not matched" no longer exists as a skip reason.
- **Order matters:** printing availability is checked *before* the front-face tie-break
  (`MON002` sits on the front of two products; only one has 1st Edition Normal). The TS
  set-listing path, the TS search path and the Python matcher were verified identical on the
  MON data.

## Two requests per Market open (2026-10-24, night)

Skylar: keep the Market tab's TCG traffic to two calls or fewer, and the headline price must be
**our `Product.price`** (the daily refresh), because collections and listings use that same
number — the Market tab can't show a different one.

- **`GET /cards/:id/prices` is gone from the per-card path.** The history already has a price
  for every day, so the current price is the latest priced day and `change24h / 7d / 30d` are
  that day against the price 1 / 7 / 30 days earlier (one decimal, null without that day — the
  same definition TCG uses). The printing is picked from the printings present in the history.
  Per open: **search + history = 2 calls** (plus one history per extra candidate card when a
  number is shared, capped at 4; the set fallback is 3). Both calls are memoized per warm
  Lambda for an hour, so a repeat open of the same card costs nothing.
- **Two is the floor** without storing their card id, which we don't.
- **Mobile headline = `Product.price` + "Pricing from TCG API · updated `externalPricedAt`"**
  (back to decision 5; the fetch-time line from the handoff is dropped). The big price no
  longer follows the finger — the scrub bubbles carry the day's values. `MarketDto.marketPrice`
  stays as TCG's latest, for reference; `medianPrice` is gone.

## Mobile UI pass from the Figma handoff (2026-10-24, evening)

The handoff prompt was run (status → built). What shipped in nanza-mobile, uncommitted:

- **Chart layer is hand-rolled `react-native-svg`** (`src/components/entity/market/`): a
  Catmull-Rom curve with a gradient fill to a dashed baseline, Low / Peak markers within the
  range (labels with a `background` text halo), a pulsing latest-price dot (Reanimated loop, off
  under Reduce Motion), daily volume bars (dim / avg-window / scrubbed tiers, max-bar count), five
  axis dates. `react-native-gifted-charts` was removed: its pointer is per-chart and its x/y
  mapping isn't exposed, and the synced scrub needed one shared geometry anyway.
- **Hold-to-scrub both charts** (`useMarketScrub`): one RNGH `Pan().activateAfterLongPress(220)`
  over the chart block; hairline x on the UI thread, day index to JS only when it changes; a
  glass bubble per chart (price + date / volume + date); the big price follows the finger and
  snaps back on release. No haptics.
- **Sticky tab bar:** a pinned copy rendered beside `PageLayout` at `safeTop + 54`, shown when
  `scrollY + pinTop ≥ heroHeight + flowPull + capHeight + barY` (a `useAnimatedReaction`); the
  in-flow bar goes transparent while pinned; each pane has a viewport-tall floor and a tab tap
  while pinned scrolls to the pin point. PageLayout untouched.
- **Tokens / type / icons:** `positive` / `negative` aliases of `conditionColors.mint` /
  `.damaged` (both themes; 16 % tints via `hexToRgba`), `smallRegular` (Roobert Regular 12/16),
  `add-to-collection` header icon, `triangle-up` / `triangle-down` chip glyphs.
- **Tabs & panes:** Details (always expanded, primary rows above attributes, Show more gone) ·
  Market · Trade (grid, Figma empty copy + the Place bid / Sell pair) · Chat ("Start the chat"
  at 0 posts). The sheet body's block gap is gone; each pane carries its Figma gap.
- **Chips:** null / 0 → gray "0%" with no triangle.

## Decisions (Skylar, 2026-10-24)

1. **Plan tier:** we already have **Pro**. Use `GET /sets/:id/prices` for the refresh and full
   history on the Market chart.
2. **Price rule:** `Product.price` = TCGplayer `market_price` × 0.90, the same as the
   card-pricing methodology.
3. **Tab order and default:** **Details · Market · Trades · Chat**, opening on **Details** (reordered twice on 2026-10-24; was Chat-first; the label went Trades → Trade → Trades the same day). Pane loading = the small system spinner (`PaneSpinner`), not the Lottie logo.
4. **Guests:** signed-out users don't see the Market tab at all (hidden for now). The endpoint
   also requires auth.
5. **Market tab layout, two layers:**
   - **Top, always instant (no TCG call):** "Market price" label, our stored `Product.price` in
     big numbers, and a subtle line under it: "Pricing from TCG API · updated Oct 22". The date
     is the last refresh that wrote this product's price.
   - **Under it, the live request:** a spinner while `GET /entity/:id/market` loads, then the
     range pills + chart + change figures. If the request fails, or the card isn't mapped, show
     a plain message in that slot (e.g. "Price history isn't available right now"). The top
     layer stays. So unmatched cards keep the tab, with the message in the live slot.
6. **Gold Foil / Marvel:** Skylar sent the TCGplayer details. The matcher treats them as separate
   products told apart by a name parenthetical. The first real refresh review will confirm the
   naming (unmatched rows show the reason).
7. **Store nothing of theirs** (on review): no TCG ids, printing or cache in our DB; one generic
   `Product.externalPricedAt`.

### What the decisions add to the build

- **`Product.externalPricedAt DateTime?`**, stamped by the refresh SQL alongside `price`. It feeds
  "updated Oct 22" and stays null (the line is hidden) for prices that didn't come from an
  external source.
- The entity detail response (`?include=product`) carries `product.price` +
  `product.externalPricedAt`, so the top layer renders with no extra request.
- `MarketDto.available: false` (unmatched) and any error both render the live-slot message.
  Neither one is a screen-level error.

## Order of work

Phase 0 spike (done) → backend (built, reworked) → Skylar tests the API, runs the migration,
publishes types `0.1.125` → Phase 2 refresh runs → Phase 4 mobile. The mobile tab restructure
(Chat / Trades / Details) doesn't depend on TCG API.
