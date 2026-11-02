---
tags: [nanza-api, plan, pricing, orchestration, tcg-api]
---

# TCG API — Daily price refresh orchestration (Plan)

> **Status: plan, 2026-10-27 (Skylar, voice).** A runbook for the daily loop, built on the
> `review` / `apply` script that [[tcg-api-pricing]] Phase 2 already delivered
> (`migrations/orchestration/generate_tcgapi_prices_sql.py`, uncommitted in nanza-api). Only two
> things change in code: the flag threshold drops to **30 %** (was 50 %) and the cadence becomes
> daily. The rest of this page is the contract — what goes in, what comes out, who does what —
> so the loop can run as a Claude orchestration the same way card pricing does
> (`.claude/agents/pricing/`). Fold into [[tcg-api-pricing]] once a week of runs has gone through.

## Update 2026-10-31 — one step, no review gate (Skylar, voice)

The two-step `review` → `apply` took too long day to day. The daily loop is now **one CSV in,
three files out**, from a single command:

```
python3 orchestration/generate_tcgapi_prices_sql.py run tcgapi-<date>.csv   # release = today
```

- `update/*.sql` — every `ok` + `new` row, plus the market fallback for skipped rows. Run it.
- `prices.csv` — exactly what the SQL sets (old price, new price, move, status).
- `flagged.csv` — the rows **left out** because the move tripped the flag rule (≥ 30 % and
  ≥ $1, or ≥ $25). Reference only; they keep their old price and are measured against the same
  baseline (and flagged again) tomorrow.

The key is set once: `TCGAPI_KEY=…` in `nanza-api/migrations/.env` (gitignored, never in the
vault). An exported `TCGAPI_KEY` still wins. `review` / `apply` stay for a run that should be
gated. Steps 2–4 below describe that gated path.

## The ask (Skylar, voice, 2026-10-27)

- We already read TCG API live for the Market tab. `Product.price` is ours, and it needs a system
  that refreshes it **every day** from the same source.
- **Set-level requests, not per-card.** One `GET /sets/:id/prices` per FAB set is about 111–120
  keyed requests for the whole catalogue, against Pro's 10K/day; per-card would be ~11K.
- **Match to our ids.** Each row is our `Product.id` + `Product.number`; the number is what finds
  the TCG price, the id is what the SQL updates.
- **CSV in, CSV out.** Skylar sends the input CSV (id, number, current price — the first run's
  current price is empty), the orchestration sends back the output CSV with the new price beside
  the old one.
- **Flag a day-to-day move of 30 % or more.** Flagged rows are for Skylar's review, never
  applied on their own. After the review, the reviewed CSV becomes the SQL that updates
  `Product.price` by id.

## The loop

```
DBeaver export ──► input.csv ──► review ──► review.csv ──► Skylar reviews ──► apply ──► update.sql ──► DBeaver
 (baseline)                    (~111 TCG                 (approve=yes on                 (approve=yes
                                set calls)                the flagged rows               rows only)
                                                          they accept)
```

1. **Baseline export** (Skylar, DBeaver → `migrations/import-csv/tcgapi-<date>.csv`). The
   product's *current* price is the baseline the move is measured against — the previous run's
   output, or empty for any card that has no price yet. Unpriced cards are **included every
   day** (not just on the first run): an empty price is a `new` row, pre-approved, so they pick
   up a price as soon as TCG has one.

   ```sql
   -- Every FAB product with a number, priced or not: a NULL price is a "new"
   -- row (pre-approved), so unpriced cards pick up a price on every run.
   -- LEFT JOIN on Set: the set name only breaks ties, so a card without one
   -- still belongs in the run. Brand name = the slug the other scripts use;
   -- check with SELECT "name" FROM "Brand" if this returns nothing.
   SELECT p."id" AS product_id, p."number", p."price",
          s."name" AS set_name, s."displayName" AS set_display_name
   FROM "Product" p
   JOIN "Entity" e ON e."id" = p."entityId"
   JOIN "Brand" b ON b."id" = e."brandId"
   LEFT JOIN "Set" s ON s."id" = e."setId"
   WHERE b."name" = 'flesh-and-blood' AND p."number" IS NOT NULL;
   ```

2. **`review`** (the orchestration, with `TCGAPI_KEY` in the environment — the key is never read
   from Secrets Manager or written anywhere):

   ```
   python3 orchestration/generate_tcgapi_prices_sql.py review tcgapi-<date> <date>
   ```

   - Pulls **every** FAB set's price listing once (`GET /sets/:id/prices`, ~111 calls) and
     matches each `Product.number` (`[U-]BASE[-RF|CF|GF|MV|EA|AA]`, `tcgapi_common.py`) across
     all of them. Our set names never have to equal theirs; ours only break a tie when two TCG
     sets carry the same number.
   - `new_price` = TCGplayer `market_price` × 0.90, rounded to cents (the pricing methodology).
   - Writes `output/tcgapi-prices/review.csv`, flagged rows first, biggest dollar move on top.

3. **Review** (Skylar). Open `review.csv`, look at the `flagged` rows, set `approve=yes` on the
   ones to accept, or type a different `new_price`. Leave the rest blank.

4. **`apply`** (the orchestration):

   ```
   python3 orchestration/generate_tcgapi_prices_sql.py apply <date>
   ```

   Writes `output-sql/<date>/tcgapi-prices/*.sql` — batched
   `UPDATE "Product" SET "price" = …, "externalPricedAt" = NOW() WHERE "id" = …` for
   `approve=yes` rows only. Rows left blank keep their old price and get flagged again tomorrow
   against the same baseline.

   **Market-price fallback (Skylar, 2026-10-27).** A `skipped` row that still carries a TCG
   `market_price` (mostly prices older than 14 days) is written with that market price **as-is,
   not × 0.90**, instead of staying unpriced. Skipped rows with no market price at all stay
   unpriced. Fallback statements are commented `[market fallback: <reason>]` and come last, in
   the final batch. First run: 577 of 13,585 updates.

5. **Run the SQL** (Skylar, DBeaver). `externalPricedAt` is what the Market tab's
   "Pricing from TCG API · updated <date>" line reads.

## CSV contracts

**Input** (the baseline export; extra columns are carried through untouched):

| column | meaning |
|---|---|
| `product_id` | our `Product.id` — the key the SQL updates |
| `number` | our `Product.number` — what finds the TCG card + printing |
| `price` | current `Product.price`; empty on the first run |
| `set_name`, `set_display_name` | tie-break only |

**Output** (`review.csv`, one row per input row, nothing dropped):

| column | meaning |
|---|---|
| `product_id`, `number`, `price` | as sent (`price` = the old price) |
| `new_price` | market × 0.90, or empty when skipped |
| `delta`, `pct` | `new_price − price`, and the percent of the old price |
| `market_price_as_of` | TCG's price date (staleness check, 14 days) |
| `status` | `ok` · `new` · `flagged` · `skipped` |
| `reason` | why it was flagged or skipped (the matcher's words: ambiguous, no printing, no market price, stale …) |
| `approve` | `yes` pre-filled on `ok` and `new`; **blank on `flagged`** — Skylar's column |

## Flag rule (2026-10-27)

A row is **flagged** when either holds:

- `|pct| ≥ 30 %` **and** `|delta| ≥ $1` ($1 since 2026-10-31, was $2; only matters under $10, since 30 % of $10+ is ≥ $3) — the dollar floor keeps $0.10 → $0.14 noise out of the
  review (30 % of a dime is not a decision).
- `|delta| ≥ $25`, whatever the percent.

`--pct`, `--min-delta`, `--max-delta` override per run. The first run has no baseline, so every
priced row is `new` and pre-approved; the 30 % rule starts biting on day two.

## Request budget

| call | count / day |
|---|---|
| `GET /sets?game=flesh-and-blood-tcg` (public set list) | 1 |
| `GET /sets/:id/prices` | ~111 (one per FAB set, grows with each drop) |
| **total** | **~112** of Pro's 10,000 |

Leaves ~9.9K/day for Market-tab opens (two calls each). A `429` carries `Retry-After`; the
script stops and the run is re-tried later the same day — nothing partial is written.

## Who runs what

Today, **operator-run, Claude-assisted**, the card-pricing shape: Skylar drops the export into
`import-csv/`, asks for the refresh ("run today's TCG refresh on tcgapi-2026-10-28.csv"), the
orchestration runs `review` and hands back `review.csv` with the flagged count and the top
movers in the reply; Skylar edits `approve`, asks for `apply`, runs the SQL. A
`tcg-price-refresh` skill / agent in `nanza-api/.claude/` can wrap exactly those two commands
once the loop has run by hand a few times (next step, not this plan).

**Later: a scheduled Lambda** (the `cancelOrderHandler` / `reviewOrderHandler` pattern,
`serverless.yml` `schedule`), once the numbers are trusted for a week: it runs `review` itself,
writes the `ok` / `new` rows directly, and parks only the `flagged` rows for Skylar — the same
rule, no CSV hand-off. Needs the key in `nanza-credentials-<stage>` and the matcher's TS twin
(`src/utils/tcgApiMatch.ts`, already verified identical to the Python one on MON data).

## Decisions wanted (Skylar)

1. **First-run baseline.** Run 1 writes every matched FAB price as `new` (pre-approved) — about
   the whole catalogue in one SQL. OK, or review that first batch too?
2. **Skipped rows.** Today they stay at their old price silently; should the output also
   carry a `skipped` summary (count + reasons) so a broken set name shows up on day one?
3. **Time of day.** Prices are "priced daily" on their side; a morning run (after their refresh)
   is the natural slot. Confirm when they refresh before fixing the hour.

## Order of work

30 % default in the script (done 2026-10-27) → first hand run on a fresh export → a week of daily
runs → `tcg-price-refresh` agent → scheduled Lambda.
