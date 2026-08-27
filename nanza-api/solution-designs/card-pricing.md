# Card Pricing

**Status:** built 2026-08-20 · orchestration live in `nanza-api/.claude/agents/pricing/`

The price-recommendation orchestration: Skylar drops a CSV of cards (or a zip of CSVs), Claude
Code fans out pricing agents that research current market value on the live web, and each card
comes back with **one recommended price** — no ranges. Runs entirely inside Claude Code sessions
(no Anthropic API key, no per-token billing); the vault holds the methodology the agents read at
runtime.

## How it works

- **Input** — either a single CSV of **≤ 50 cards**, or a `.zip` whose CSVs each hold ≤ 50 cards.
  The 50-card cap per CSV is deliberate: it keeps each pricing agent's context small enough that
  no card gets skipped or hallucinated (the failure mode of oversized batches).
- **Orchestrator** (`pricing-orchestrator`) — validates the input, unzips if needed, splits any
  oversized CSV, then spawns one `card-pricer` agent **per CSV, in parallel** (they're
  independent). Merges the per-CSV outputs into a combined file plus a run summary.
- **Model tiering (added 2026-08-20)** — the fan-out workers are cheap, the judgment is not:
  `card-pricer` pins `model: haiku` (bulk lookup + arithmetic per 50-row shard),
  `pricing-orchestrator` pins `model: sonnet` (coordination), and the interactive session
  driving the run supplies the top-level brain. After the fan-out, the orchestrator collects
  every flagged row (low confidence, finish conflicts, unresolved) and re-runs just that tail
  through one `card-pricer` spawned with a `model: sonnet` override — so the expensive model
  only ever sees the hard few percent. (The migration agents don't pin models; this pipeline
  is the first to tier explicitly.)
- **No model review stage above the sonnet escalation (decided 2026-08-20)** — a reviewer can't
  verify a price without redoing the search, and cheap rows aren't worth re-verifying. The
  migration flow's manual SQL review is the human backstop.
- **Pricer** (`card-pricer`) — reads the **Pricing methodology** section below, prices its ≤ 50
  cards using live web search, and writes `<name>-priced.csv` preserving every input column and
  appending `price_usd`, `confidence`, `priced_at`, `note`.
- **Output** — `nanza-api/pricing/runs/<YYYY-MM-DD>-<input-name>/` containing the per-CSV priced
  files, `all-priced.csv` (zip runs), and `run-summary.md`. Run folders are local working
  artifacts like `migrations/output-sql/` — commit only if worth keeping.

### Two input modes: new cards vs. existing-price verification

- **New cards** (the original mode) — `product_price`/`price_usd` starts blank; the pricer fills
  it. Output drops straight into the entities-and-products migration.
- **Existing-price verification (added 2026-08-21)** — input is an export of cards that
  **already have a DB price** (e.g. `SELECT p.number, p.price, e."displayName", s."displayName"
  FROM "Product" p JOIN "Entity" e ON e."id"=p."entityId" LEFT JOIN "Set" s ON s."id"=e."setId"
  WHERE p.price > 100`). The goal is to **independently re-derive a price and compare it against
  what's already stored** — catching cases where the DB says $3,000 but the real market is closer
  to $1,200. Rules:
  - **Never overwrite the existing price column.** Add a **new column** (`new_price`) for the
    freshly-researched number; the original `price` stays untouched so the two can be diffed.
  - **Price it exactly as if researching from scratch** — no anchoring on the existing DB value,
    no "well it's already priced around there so this seems right." The whole point is an
    independent second opinion; seeing the old price first and then converging on it defeats it.
  - **Suffix → print column.** These exports carry a bare `product_number` (e.g. `FAB001-GF`),
    not a separate `print` column — derive one from the suffix before pricing: `-RF` → Rainbow
    Foil, `-MV` → Marvel, `-CF` → Cold Foil, `-GF` → Gold Foil, no suffix → Regular. Watch for
    non-finish suffixes that don't belong in that set (e.g. `-TP` item-type markers, `-A`/`-B`/`-C`
    pitch-color variants) — label those explicitly rather than forcing them into one of the four.
  - **Output = `product_number, print, price (existing), new_price (researched), entity_display_name,
    set_display_name, confidence, note`.** A large delta between `price` and `new_price` is the
    signal worth surfacing — the run summary should call out the biggest gaps first.

### CSV contract

The canonical input is the **migration new-cards CSV** (same file the card-data migration
consumes, e.g. `migrations/fab-new-cards-<date>.csv`). Columns the pricer uses:

- **`raw_card_number`** (required) — the printing-specific identifier, finish suffix included:
  `AOL003` vs `AOL003-RF`, `FAB368-GF`, `MPW155-MV`. This is the primary lookup key; each row is
  one distinct printing.
- **`print`** — the finish: `Rainbow Foil`, `Cold Foil`, `Gold Foil`, `Marvel`, blank = regular
  non-foil. **Biggest price driver** — same base code spans pennies to five figures. Must agree
  with the `raw_card_number` suffix; on conflict, trust `raw_card_number` and flag it.
- **`entity_display_name`** — sanity check for the lookup; mismatch = flag, don't silently reprice.
- **`rarity`**, **`edition`**, **`event_type`/`event`** — context (Promo/Legendary/Marvel, prize
  provenance) that steers search depth.
- **`set_code`/`set_name`** — context only; these are sometimes inconsistent in the source data
  (trust `raw_card_number`).
- **`product_price`** — the **output column**: the pricer fills it with the single recommended
  price. Condition is always NM (new marketplace inventory), so no condition column is expected;
  a `condition` column is honored if one ever appears (NM ×1.00, LP ×0.95, MP ×0.85, HP ×0.70).

**Output = same schema, `product_price` filled** — a copy of the input CSV (never edited in
place) that drops straight into the migration's entities-and-products stage. Confidence and flags
live in the companion `pricing-report.csv` (`raw_card_number`, `price_usd`, `confidence`, `note`)
and `run-summary.md`, so the migration CSV schema stays untouched.

A minimal ad-hoc CSV (just a `code`/`Card Code` column, optional `finish`, `name`) is also
accepted — output then appends `price_usd`/`confidence`/`priced_at`/`note` columns instead.

## Pricing methodology (source of truth)

> Agents read this section at runtime — edit it here (in Obsidian or directly) and the next run
> obeys it. No agent changes needed. *(Seeded 2026-08-20 from Skylar's Gemini research session;
> replace/merge freely when the original methodology markdown lands.)*

1. **One price, USD.** Never a range. Determine the current fair-market **Near Mint** midpoint,
   then apply the discount rule below.
2. **The 10% rule.** `price_usd = fair NM market midpoint × 0.90`. The output is deliberately a
   buy-side / competitive-listing price, 10% under market.
3. **Condition multipliers** (only when a `condition` column is present): NM ×1.00, LP ×0.95,
   MP ×0.85, HP ×0.70 — applied *after* the 10% rule.
4. **Rounding, then a small cosmetic jitter (updated 2026-08-20):** < $1 → nearest $0.05;
   < $20 → nearest $0.25; < $100 → nearest $1; < $1,000 → nearest $5; ≥ $1,000 → nearest $25.
   After rounding, nudge the **last digit only** by ±1–4 cents so output prices don't read as
   machine-flat (e.g. $0.10 → $0.12, $0.15 → $0.14, $3.25 → $3.23) — mimicking how real listings
   land on odd cents rather than clean round numbers. This is cosmetic only: it happens strictly
   *after* rounding, moves the price by at most a few cents, and **never changes which range the
   number came from** — it is not a second chance to adjust the price. Skip the jitter on
   already-precise sourced numbers (e.g. a sold price of exactly $17.99) — only apply it to
   prices that resulted from rounding/estimation, where the "flatness" is an artifact of the
   pipeline, not the market.
5. **Reputable sources — consult several, privilege none (revised 2026-08-20).** The price must
   come from **observed marketplace numbers for that exact printing**, gathered this run. Aim to
   corroborate across **at least two** independent sources and take the midpoint of what they
   show; a single source is acceptable when it's all that exists, but then say so in `note`.

   **Non-USD sources are usable, but only with an explicit, cited conversion (relaxed
   2026-08-21).** Cardmarket (€), Canadian/UK/AUD retailers, hasuya.jp (¥) price a different
   market with different supply, so **prefer a USD source when one exists** — but when USD data
   is thin or absent, a foreign listing is real evidence, not nothing. To use one:
   1. **State the FX rate you used and where it came from** in `note` — e.g. "amusengames.ca
      CAD $0.50 → ×0.73 (CAD/USD) → ~$0.37 USD."
   2. **Mark it `confidence: low`** — a converted price carries both thin-supply skew and FX
      risk stacked together, never `medium` or `high` regardless of how clean the source looks.
   3. **Never silently convert.** The `note` must show the original currency, the original
      figure, the rate, and the converted result — a bare USD number with no conversion trail
      is indistinguishable from a fabricated one and gets treated as such.
   What's still banned: **presenting a foreign figure as if it were already USD** (the original
   sin here — a Cardmarket/cardsrealm/CAD number reported as a clean US price with no
   conversion shown) and **treating one foreign source as if it settles the price** — it's one
   data point among the sources, not an override.

   The working set (all reputable USD sources, none authoritative on its own):
   - **TCGplayer** — market price + low/mid listings. The primary US reference.
   - **eBay sold listings (US)** — the strongest signal when present, since it's completed sales.
   - **Star City Games, Dragon Shield card manager, and comparable US retailers** — good
     corroboration; note that out-of-stock listings drift above true market.
   - **FabFoundry** — FaB-specific marketplace/price data; good single-card corroboration and
     often carries printings the generalist sites list thinly.
   - **Specialist FaB trackers** (fabstocks.net, FABREC, the LSS gold-foil print registry) —
     best for prize/promo cards. Print counts justify premiums but are **context, not prices**.
   - **High-end vendors** (Tier 1 Games, hasuya.jp, …) when open-market data is genuinely thin.
   - **Google Shopping results (via web search)** — searching the card name + printing often
     surfaces Shopping-tab price-comparison snippets alongside organic hits, a useful extra
     angle when the usual retailers are JS-blocked. Filter to USD listings only, same as any
     other source; a Shopping snippet showing a non-USD price doesn't count.

   **How to use the synthesized answer (2026-08-20, per Skylar):** the search tool's own prose
   summary (its synthesized "US$X.XX" figure) is a usable reference, at two tiers:
   1. **You already have real listing data** — use it as one more number to corroborate or
      sanity-check against, exactly like any other source in the mix.
   2. **You have nothing else at all** — the synthesized figure can stand in *if* it reads as
      mid-to-high confidence: a specific number tied to a named retailer/count-in-stock (not a
      vague range), and you've traced it to a real page.
   In both cases, **first click through to the actual listing the summary is citing and confirm
   its currency yourself — never take the summary's own "$"/"US$" label at face value.** A
   search for MPW025-RF returned "US$85.90" in prose; the real listing was invasioncnc.ca, a
   Canadian retailer priced in CAD. The summary's number can still be useful once you know the
   real currency (skip it if non-USD, same as any other source) — the rule is "verify then use,"
   not "never use."

   Weight *recency and whether it's a real transaction*, not the brand of the site. A sold price
   beats a listing; a listing beats an estimate; an estimate beats a guess. Outliers (one
   out-of-stock listing at 3× everyone else) get discarded, not averaged in.

   **Cite the sources you actually used in `note`** — e.g. `TCGplayer $0.35 / SCG $0.99 → mid
   $0.55`. A note that names no source is not evidence.
5b. **Never derive a price from rarity alone — and no exact-printing match means blank, not a
   comparable (tightened 2026-08-20).** "Rare, recent set, so ~$3" is a *guess wearing a
   number*, and it is the failure mode that produced 4–8× errors on the first test run. Rarity,
   set age, and foil tier tell you **how hard to look** and how to sanity-check — they never
   produce the price.

   **If real search finds no listing for that exact printing (exact `raw_card_number`, exact
   finish), leave `price_usd` empty and `confidence: unresolved`. Full stop — do not substitute
   a "closest comparable" instead.** A same-card-different-treatment price (Cold Foil standing
   in for Gold Foil), a same-treatment-different-printing price (base-set Rainbow Foil standing
   in for a reprint's Rainbow Foil), or a same-tier-different-card price (another Legendary's
   price standing in for this one) are all comparables, and every one of them turned out to be a
   disguised guess in practice — see [[../pricing-runs/INDEX|Pricing Runs]] lesson #17. Skylar
   would rather see a blank cell to fill in by hand than a plausible-looking number with no real
   backing.

   **The bulk-estimate exception is narrower than it sounds (tightened 2026-08-21).** It fires
   only when the product literally has no per-card singles market *ever* — cards that ship
   exclusively inside sealed precon product (rule 5's example: Armory Deck / demo-deck
   exclusives). **A newly-released set whose singles market simply hasn't been indexed by USD
   sites yet is NOT that case** — it's the ordinary thin/no-data outcome, and the ordinary
   outcome is blank, same as any other unresolved card. Two shards of the same 2-week-old set
   hit an identical wall (TCGplayer/SCG JS-blocked, only banned non-USD sources had numbers) —
   one correctly left 44 rows blank, the other used a uniform two-price bulk estimate across all
   50. The bulk estimate was wrong to ship: a common and its Rainbow Foil sibling landing on the
   exact same flat number with zero card-to-card differentiation **is** the "skipped research"
   signature (standing lesson #2), even when produced honestly and labeled `low`. When in doubt
   whether a shard qualifies for the exception, it doesn't — leave the rows blank.
5d. **Two phases: find the range with judgment, then stop touching the number (2026-08-20).**
   Reasoning is exactly what's wanted while you're *locating* the right price — deciding which
   sources are trustworthy, which listings are stale outliers, which printing a comparable
   actually belongs to. That judgment is the job, and it's what made MPW011-RF's four candidates
   ($17.99, $71.21, $91.09, $13.13) resolve down to two credible ones. **Once you've done that
   work and settled on a range you believe, the recommendation is: the price implied by your
   reliable sources — two of them, or four, however many you trust — minus 10%.** Nothing gets
   added back in after that. No narrative nudge for "fresh reprint so demand is up", "this will
   spike", "seems low for a Legendary" — that's reopening a decision that's already made, and it
   invents money on top of evidence in a way that's harder to catch than a bare guess because it
   arrives wrapped in real research.

   *The case that produced this rule:* MPW011-RF Braveforge Bracers. The source-vetting was
   correct — SCG $17.99 and cardsrealm $13.13 kept, TotalCards $71.21 and Troll and Toad $91.09
   discarded as stale/out-of-stock. That work located the range: roughly $13–$18. The
   recommendation from that range, minus 10%, is ~$14. The worker then added $20 on top anyway,
   shipping $35 "to reflect this is a fresh MPW reprint" — discarding its own correct work.

   If the evidence genuinely doesn't fit the card (e.g. every source is for a *different*
   printing), the honest moves are: use the closest observed comparable **at its observed value**
   and mark `low` with the caveat in `note`, or leave it `unresolved`. Never split the difference
   between evidence and intuition.

5e. **Reprints: price the printing you have, from that printing's data.** A base-set figure
   (WTR116-RF) is a **cross-printing comparable** for the reprint (MPW011-RF), not a match — use
   it at its observed value, mark `low`, and name both printings in `note`. Reprints usually
   trend *at or below* the original once new supply lands; never mark one up on reprint
   speculation.

5c. **Sanity-check against the base card.** Most FaB non-foil commons/rares from recent sets sit
   **under $1**; a non-foil, non-promo card priced above ~$2 needs an observed number behind it.
   Conversely, tournament/prize promos (Pro Tour, Nationals, GEN CON, Gold Foil) routinely run
   **$50+** — under-pricing those is as wrong as over-pricing bulk. Both directions of error are
   real; the first test run made both.
6. **Finish discipline.** Never conflate treatments: a *golden/gold prize foil* is not the pack
   *cold foil*, which is not the *rainbow foil*, which is not the base printing. If search
   results mix treatments, price only the requested one.
6b. **How to actually search a printing — split the code from the treatment (2026-08-20).** The
   `-GF` / `-CF` / `-RF` / `-MV` suffixes are **our internal notation, not market identifiers**.
   Retailers do not list "FAB368-GF"; searching the suffixed string finds nothing and pushes a
   worker toward guessing. Instead:
   - **Search the base number + the card name + the treatment as words**: `FAB368 Prized Galea
     gold foil`, `MPW155 Dorinthea marvel`, `MON107 Valiant Dynamo cold foil`.
   - Suffix → search term: `-GF` = "gold foil" (also "golden"), `-CF` = "cold foil", `-RF` =
     "rainbow foil", `-MV` = "marvel", **blank = regular / non-foil / unfoiled**.
   - Then **confirm from the listing itself** that the treatment matches before using the price.
     A hit on the bare base number is usually the *regular* printing — that is a cross-treatment
     comparable, not a match, and rule 6 still applies.
   The suffix tells you *which* treatment to look for; it is never part of the query string.
7. **Search budget — and be a good citizen of third-party sites.** Commons and bulk from the
   same set may share one set-level lookup; rare/promo/prize cards get individual searches (aim
   ≤ 3 per hard card, ~20 per 50-row shard). Never output a price with **zero** supporting
   evidence from this run — model memory alone is `confidence: low`.
   **Politeness rules (2026-08-20):** TCGplayer, Cardmarket, eBay and the specialist trackers
   are someone else's infrastructure — prefer one broad set-level or price-guide page over many
   per-card hits, never hammer the same host in a tight loop, don't scrape listing pages in
   bulk, and keep concurrency modest (the ≤50-row shards with a handful of shards in flight is
   the intended ceiling — don't fan out dozens of workers all searching the same domains). If a
   price needs more requests than that to pin down, mark it `low` and move on rather than
   grinding the source.
8. **Confidence + honesty.** `high` = recent sold data found; `medium` = live listings only;
   `low` = thin/stale/no data — put why in `note` and still give the single best number. A card
   code that can't be resolved at all gets an empty `price_usd` and `note: unresolved`, never a
   guess.
9. **Evidence floor — minimum one search per shard, always (added 2026-08-20 after the first
   test run).** A shard of all-commons is the dangerous case: it's tempting to assign one
   blanket bulk price from memory. **Every shard must run at least one web search**, even if
   all 50 rows share a set-level bulk price, and a shard-wide bulk price must be recorded as
   `confidence: low` with `note: set-level bulk estimate` on every row it covers — never a bare
   `medium` with an empty note. Uniform prices across a whole shard with no note is the
   signature of skipped research, and the orchestrator treats it as a flag.

## After every run: write a run note in the vault

Each run ends by writing a note into [[../pricing-runs/INDEX|`nanza-api/pricing-runs/`]] —
`<date>-<input-name>.md`, newest-first row added to that INDEX — so the runs are visible and
linkable in Obsidian. The bulk CSVs stay in `nanza-api/pricing/runs/<run-id>/` (local working
artifacts); only the *learning* goes in the vault.

A run note carries: what ran (rows, shards, models, request count), the headline numbers,
**what the pipeline got wrong and how it was caught**, and which methodology rule changed as a
result. When a run exposes a new failure mode, add the rule here *and* add a one-line entry to
the INDEX's "Standing lessons" — that list is the accumulated memory every future run inherits.

This is the feedback loop that makes the methodology improve instead of repeating mistakes: run
→ postmortem → rule → next run starts smarter.

## Why this shape

- **Claude Code, not the API** — at current volumes (dozens to low thousands of cards per run) the
  subscription session is free-at-the-margin and the vault doubles as the config surface. If
  volume grows toward ~100k cards/week, the documented escalation path is the Claude API:
  Batch API (50% token discount) + server-side web search ($10/1k searches) + structured outputs,
  with a tiered design that only web-searches low-confidence cards.
- **Same conventions as the migration pipelines** — per-run folders, fan-out to small
  single-purpose agents, generate-don't-mutate (the pricer writes files; nothing touches the DB).
  Pattern summarized in [[../../_shared/capabilities/orchestrations|Orchestration patterns]].
- **Distinct from Sources & Insights** — [[sources-and-insights|Sources & Insights]] / oak-cortex
  is the *in-product* price/insight engine over Nanza's own data. This is an *operator-side*
  research tool over the open web; its outputs may later feed price migrations, but they share no
  code or runtime.
