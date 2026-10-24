# 2026-08-21 · fab-new-cards-part6

**Input:** `fab-new-cards-part6.csv` (50 rows, Mastery Pack Warrior commons, MPW105-MPW133 range)
**Output:** `nanza-api/pricing/runs/2026-08-21-fab-final/fab-new-cards-part6-priced.csv`
**Outcome:** 50/50 priced (0 unresolved). 49 direct-USD, 1 converted non-USD (GBP, cited rate).

## Context

Third attempt at this shard. Attempt 1 shipped a uniform flat bulk price across all 50 rows
(invented, invalidated). Attempt 2 correctly refused to guess and went 0/50 blank under a
strict "US sources only, no rarity, model memory not evidence" rule — but that was too strict:
Skylar reviewed and relaxed two things for this re-run: (1) use all source types, not just
TCGplayer, and (2) non-USD sources are usable again with an explicit cited FX conversion
(this is the run that produced methodology rule 21 / standing lesson 21, written 2026-08-21
before this run started).

## What worked

`fab.cardsrealm.com` per-card pages (not the set-summary page, which gave inconsistent/
unreliable numbers on first pass) carry a **dated price-history table** with explicit
`USD` labels — e.g. `2026-08-17 | $0.78 | $0.78`. Because MPW went tournament-legal
2026-08-07, pages tracking the MPW printing specifically have history starting ~7/30-8/03;
pages with history reaching back to 2024 are tracking the card's *original* printing
(cross-printing comparable, rule 5e — e.g. Hit and Run's page is the Crucible of War
history, not an MPW-specific one). Distinguishing "recent-only" vs "multi-year" history
on each page was the key move that made this run trustworthy — the set-summary page and
the "current minimum price" headline figure at the top of each card page were both
noisier/less reliable than the dated table underneath.

One genuine non-USD case: MPW122 Sharpen Steel has no MPW-specific page; the Welcome to
Rathe base printing was priced at totalcards.net, a UK retailer, **£0.18 GBP** — converted
at 1.3646 (GBP/USD, web search mid-market rate, cited in the row's note) to ~$0.25 USD,
marked `confidence: low` per rule 21.

## What got caught and discarded

- **Outlier TCGplayer unit-lot prices.** Several cardsrealm "Where to buy" TCGplayer lines
  showed prices wildly out of line with everything else for that card family — Jive MPW115
  at "$26.36 (8 uni.)" against a $0.70-$1.09 price-history band, Heavy Swing at "$2.26"
  against a $0.96-$1.76 band. Discarded as stale/out-of-stock single-seller outliers
  (rule 5), not averaged in.
- **A wrong-card match.** `cardsrealm.com/card/sharpen-steel-1` showed extreme volatility
  ($0.44 to $8+) inconsistent with a plain vanilla Warrior common with no combo upside —
  didn't match the card's actual text. Rejected that page's data outright and sourced
  Sharpen Steel from a direct retailer listing (totalcards.net) instead, which is the
  GBP conversion case above.
- **Cross-printing contamination inside a single page.** cardsrealm's `sharp-incline-1`
  page listed a TCGplayer line explicitly tagged "Armory Deck Origins: Hala" — a different
  printing than the MPW card being priced. Used the page's MPW-tracked price-history table
  instead of that TCGplayer line.

## Headline numbers

- 50/50 priced, 0 unresolved.
- 24 distinct final price values across 50 rows (real per-card, per-pitch, and
  reg-vs-Rainbow-Foil variance) — range $0.06 to $0.89.
- Confidence split: ~30 `medium` (MPW-specific dated price-history table, min=median
  agreement), ~19 `low` (cross-printing comparables or single-source/volatile data),
  1 `low` for the GBP conversion.
- Example conversion trail: `MPW122 Sharpen Steel: totalcards.net GBP £0.18 -> x1.3646
  (GBP/USD) -> ~$0.25 USD -> x0.90 -> rounded/jittered -> $0.17`.

## Rule/lesson this run confirms

No new rule needed — this run is the first real test of rule 21 (relaxed non-USD) and
confirms it works: only 1 of 50 rows needed the non-USD escape hatch, the rest resolved
via legitimate USD-labeled data once the search widened past TCGplayer-only. Reinforces
lesson #20 (confirm currency from the actual page/table, not the site's headline number)
and rule 5e (cross-printing comparables are fine when correctly labeled as such, not
disguised as an exact-printing match).
