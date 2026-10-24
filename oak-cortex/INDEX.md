---
tags: [oak-cortex]
---

# oak-cortex

Oak's intelligence engine: a **separate Postgres + Prisma project** that enriches the canonical
catalog (main / nanza-api DB) with external intelligence — pricing first, then events, articles,
and social. AI loops run locally (Claude Code), read a spec object down from main via a read-only
endpoint, fetch and judge per-source data, and store sources, raw readings, insights, and history
in cortex's own database. **Nothing writes to prod** — prices flow up as reviewed SQL migrations.
Successor to the legacy `oak-cortex` scraper repo (which keeps running until cutover).

Canonical keys shared with main: `entityId` + `productId`.

## Solution designs

- [[solution-designs/insight-engine|Insight Engine]] — the canonical design: two-DB architecture,
  cortex schema (CatalogItem / Source / SourceReading / Insight / MainSyncBatch — client key
  comes down from main, no registry here), AI-driven per-source fetching with self-updating
  source notes, migration-based up-flow.

## Related

- [[../nanza-api/solution-designs/sources-and-insights|nanza-api → Sources & Insights]] —
  orchestration judgment rules + legacy price algorithm reference.
