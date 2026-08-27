# Pricing Runs

Run log for the card-pricing orchestration ([[../solution-designs/card-pricing|Card Pricing]]).
One note per run — newest first. Each note is written by the orchestrator at the end of a run;
the bulk CSVs stay in `nanza-api/pricing/runs/<run-id>/` (local working artifacts, not the vault).

**Why this exists:** the methodology only improves if each run's failures come back to it. A run
note records what was priced, what the pipeline got wrong, and what rule changed as a result —
so the next run starts from everything the previous ones learned, and Obsidian shows the trail.

## Runs

| Run | Input | Rows | Outcome |
|-----|-------|------|---------|
| [[2026-08-20-fab-new-cards\|2026-08-20 · fab-new-cards]] | fab-new-cards-2026-08-17.csv | 334 | ❌ **Output discarded** — rarity-heuristic pricing, 4–8× errors both directions. Produced rules 5/5b/5c/9. |
| [[2026-08-20-fab-rerun\|2026-08-20 · fab-rerun]] | same, corrected rules | 334 | 🔄 in progress |
| [[2026-08-21-fab-new-cards-part6\|2026-08-21 · fab-new-cards-part6]] | fab-new-cards-part6.csv | 50 | ✅ 50/50 priced (49 direct-USD, 1 GBP-converted). First real test of rule 21 (relaxed non-USD) — confirms it works. |

## Standing lessons (carried into every run)

Rules born from real failures — the detail lives in each run note, the rule lives in the
[[../solution-designs/card-pricing|methodology]]:

1. **Verify workers actually searched.** A confident summary is not evidence; check the output
   for cited sources (and the transcript for search calls if in doubt). *(2026-08-20 run 1: one
   worker priced 50 rows from memory and labeled them `medium`.)*
2. **Uniform prices + empty notes = skipped research.** Treat as a flag on sight.
3. **Rarity is not a price.** "Rare + recent set → $3.25" caused the largest errors.
4. **Errors run both ways.** Bulk gets over-priced, promos get under-priced — never assume a
   single correction factor.
5. **`set_code` is unreliable; `raw_card_number` is the key.**
6. **A new set has no market.** Thin data is a fact to report at `low` confidence, not a gap to
   fill with precision.
7. **Third-party sites are someone else's infrastructure.** ~20 searches per shard, prefer
   set-level pages, modest concurrency.
8. **Marvel / prize printings carry hero-tier prices.** Re-run shard 1 found Vynnset CON007-MV
   at $225 and Raydn/Flail at ~$80 where run 1 said $9 — always price a Marvel or prize promo
   individually, never off a set baseline.
9. **Discard cross-treatment "matches" loudly.** A $9.95 "Flail of Agony Promo" is a different
   printing than the $89 Marvel — the right move is to reject it and say so in the note.
10. **Precon-only singles have no secondary market.** Armory/demo-deck cards (AOL, DDD) ship
    inside sealed product, so a set-level bulk estimate at `low` is the honest answer.
11. **Blocked sources are normal.** TCGplayer/Cardmarket/fabstocks often return JS shells or
    403 — move on to another source, don't retry-grind (rule 7).
12. **Never search the suffixed code.** `-GF`/`-CF`/`-RF`/`-MV` are *our* notation — no retailer
    lists "FAB368-GF". Search **base number + name + treatment as words** ("FAB368 Prized Galea
    gold foil"), then confirm the treatment from the listing. Searching the suffix returns
    nothing and pushes workers toward guessing — a likely contributor to the thin-data
    `unresolved`/`low` rows on the expensive printings.
13. **US / USD sources only.** No Cardmarket (€), no GBP/CAD/AUD retailers, no hasuya.jp (¥), and
    **never convert** a foreign figure into a US price. Re-run shard 4 sourced all 50 rows from a
    GBP page at a 1.27 FX rate — plausible-looking output built on the wrong market. Non-USD data
    is a `low` comparable at best, or `unresolved`.
14. **cardsrealm is a GBP/Cardmarket-style page** — it looks like a convenient per-card set guide
    and workers reach for it when TCGplayer is blocked. It is now out of bounds under #13.
15. **Judgment finds the range; then stop.** Vetting sources, discarding stale outliers,
    resolving which printing a comparable belongs to — that's good work, keep doing it. Once a
    range is settled, the recommendation is that range's reliable-source price minus 10%,
    nothing added back. MPW011-RF: correct vetting narrowed it to ~$13-18 (SCG $17.99 +
    cardsrealm $13.13 kept, two stale outliers discarded) — right answer ~$14, but the worker
    shipped **$35** "to reflect a fresh reprint", discarding its own correct work.
16. **A reprint is a different printing.** Base-set data (WTR116-RF) is a cross-printing
    comparable for the reprint (MPW011-RF) — use it at its observed value, mark `low`, name both.
    Reprints trend at or below the original; never mark up on reprint speculation.
22. **A uniform price isn't automatically the skipped-research bug — check whether it's
    corroborated.** Shard 5's re-run found 22 Rainbow Foil commons converging on the same
    $1.50-1.55 CAD floor across two Canadian retailers and multiple cards — real per-card
    searches, a shown conversion trail, correctly `low`. That's different from shard 6's
    original bug (one invented number applied to 50 rows with no per-card lookup at all). The
    tell: did the worker actually search each card and the market genuinely came back flat, or
    did it skip straight to an estimate? Check the notes for individual citations, not just
    whether the final numbers repeat.
21. **Non-USD is usable again, with a shown conversion (relaxed 2026-08-21).** Lesson #13 (US
    sources only, never convert) was too strict — it drove two shards to 0/50 and 8/50 priced.
    A foreign listing (CAD, GBP, ¥) may now be converted with a **cited FX rate shown in the
    note**, always at `confidence: low`. Still banned: presenting a foreign figure as if it were
    already USD (no conversion shown), and treating one foreign source as settling the price
    rather than one data point among several. Prefer USD when it exists; don't refuse non-USD
    when it doesn't.
20. **Confirm currency from the page, not from platform reputation.** When WebFetch on a
    TCGplayer product page returns a JS shell, "TCGplayer is always USD" is a reasonable
    inference but not the same as verifying — it's a smaller version of the exact shortcut
    (trusting a claim about currency instead of checking it) that produced lesson #18. Prefer
    the price-guide/category page (often reachable even when the product page isn't) or another
    US source over inferring from the platform's reputation alone.
19. **A "$" figure near a tournament promo may be an entry fee, not a card price.** A $50
    "Runic Reaving" hit turned out to be a Nationals side-event entry fee, not a resale price.
    Confirm what a dollar figure is actually pricing before using it.
18. **Google's synthesized summary is usable, but verify currency first.** As a corroborating
    figure when you have real data, or as the price itself when you have nothing else and it
    reads mid-to-high confidence — but click through and confirm currency before using it. A
    summary said "US$85.90" for MPW025-RF; the real listing was invasioncnc.ca, priced in CAD.
    Verify then use — don't skip the verify, and don't refuse to use it either.
17. **No exact-printing match = blank, full stop (2026-08-20, per Skylar).** "Closest observed
    comparable at `low`" sounded principled but kept sliding back into a guess — cross-treatment,
    cross-printing, and cross-card comparables all turned out to be disguised invention. If real
    search finds nothing for the exact `raw_card_number` + finish, leave the price empty with
    `confidence: unresolved`. A blank cell Skylar fills by hand beats a plausible number with no
    real backing. The one exception stays the genuinely marketless bulk-common shard (a named,
    `low` set-level estimate) — never used to justify a single high-value card's guess.
