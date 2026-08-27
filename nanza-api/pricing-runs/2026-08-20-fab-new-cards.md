---
tags: [pricing-run, postmortem]
run-id: 2026-08-20-fab-new-cards-2026-08-17
date: 2026-08-20
input: migrations/import-csv/fab-new-cards-2026-08-17.csv
rows: 334
outcome: discarded
---

# 2026-08-20 · fab-new-cards (first run — output discarded)

First end-to-end test of the [[../solution-designs/card-pricing|card-pricing orchestration]].
**The prices were not usable and the merged output was renamed
`all-priced.SUSPECT-DO-NOT-IMPORT.csv`.** Kept because every rule in the current methodology
came out of this run. Superseded by [[2026-08-20-fab-rerun]].

## What ran

334 rows / 41 columns → 7 shards of ≤50, seven haiku workers in parallel, then a Sonnet
escalation over 55 flagged rows. ~150 third-party requests total — more than we want as a
standing pattern.

## What went wrong

**1. One worker never searched at all.** Shard 5 priced all 50 rows from model memory at a flat
$0.10/$0.20 and labeled them `medium` with empty notes. Caught by checking the agent transcript
for WebSearch calls (0, versus 8–16 for the other six). Its own final report read as confident
and thorough — the prose was no signal.

**2. Rarity heuristics stood in for research — the big one.** 306 of 334 rows (92%) had notes
citing no price and no source. Blocks of rows carried `"recent set, rare non-foil" → $3.25`,
which is a guess wearing a number. Measured against sourced figures:

| Card | Run 1 | Sourced | Error |
|---|---|---|---|
| AOL017 non-foil | $1.35 | ~$0.35 | 4× high |
| MPW044 non-foil | $3.25 | ~$0.39 | 8× high |
| FAB486-RF Pro Tour promo | $9.00 | ~$50 | 5× **low** |

Errors ran in **both directions**, so no correction factor could salvage the output.

**3. The escalation tier worked — and proved the cheap pass was wrong.** Re-pricing 55 rows on
Sonnet moved the Gold Foil FAB418-GF from $76 → $160 (explicitly refusing to substitute the
Cold Foil price), Marvel treatments from $0.20 → $42/$55, and rainbow foil commons from $0.20 →
~$2.00. Roughly 10× on the bulk rows.

## What it changed

- **Rule 5** — a named set of reputable sources (TCGplayer, Cardmarket, eBay sold, Star City
  Games, Dragon Shield, FabFoundry, FaB trackers), none privileged, corroborate across two,
  cite sources and figures in `note`.
- **Rule 5b** — never derive a price from rarity alone; use a named observed comparable at
  `low`, or leave it `unresolved`.
- **Rule 5c** — two-sided sanity check: recent-set non-foil commons/rares usually under $1;
  tournament/prize promos routinely $50+.
- **Rule 7** — third-party politeness: ~20 searches per shard, prefer set-level pages, no
  hammering one host, modest concurrency.
- **Rule 9** — evidence floor: at least one search per shard, no exceptions; bulk prices
  recorded as `low` + `set-level bulk estimate`.

## Carried forward

`set_code` is unreliable (MPW035 says `ROS` but is the MPW printing) — trust `raw_card_number`.
Mastery Pack Warrior released 2026-08-07, so a large `low`-confidence share is honest, not a
defect; those rows deserve re-pricing once the market matures.
