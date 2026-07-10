---
tags: [nanza-api, nanza-admin, solution-design, sources, insights, pricing]
---



# Sources & Insights — Solution Design

## Overview

nanza's catalog is admin-curated: a **Brand** fans out to **Set** → **Entity** (a card) → 1:1
**Product**, with a taxonomy of **Tag** / **EntityTag** values. What the catalog does *not* yet
have is a systematic way to enrich each product with **live external intelligence** — the current
market price, and (later) the events, articles, and social chatter surrounding a card.

This design introduces two admin-configurable models — **Source** and **Insight** — plus a
**local-Claude loop** that continuously fetches from sources per product and an **agent
orchestration layer** that decides which raw fetches become trusted insights on the product.

**Scope of the first build: `PRICING` only.** The models are typed with an enum that defaults to
`PRICING`, but they are deliberately general so `EVENT`, `ARTICLE`, and `SOCIAL_POST` intelligence
slot in later without a schema change. The rest of this doc leads with how price is derived today
(so we port a known-good algorithm), then generalizes.

---

## How price is calculated today (the algorithm we're porting)

Prices are collected as **snapshots** — one row per (product, source, day) — from several pricing
sources (e.g. a market marketplace, a sold-listings marketplace, aggregators, retail-ask sites).
A rollup job then folds the recent snapshots into one canonical number on the product. The logic,
in order:

1. **Window** — pull all non-ignored snapshots from the last **14 days** for the product.
2. **Partition** — split into **today** (last 24h, latest snapshot per source) and **historical**
   (the rest). The historical set **excludes the noisy sold-search source** — its history is as
   unreliable as its present (a search URL matches different listings day to day: graded slabs,
   wrong card numbers), so using it as a benchmark would spread contamination forward.
3. **Historical median** — compute the median of the historical (non-noisy) prices. This is the
   anchor everything else is judged against.
4. **Per-source tolerance band** — reject a today-price as an outlier if it falls outside its
   source's band around the historical median. The noisy sold-search source gets a **tight** band
   (`[median × 0.5, × 2]`); direct-URL sources get a **loose** band (`[median × 0.25, × 4]`).
5. **Floor rule** — if today's max price is over $1, drop today-sources priced below **25% of
   today's median** (catches "sold out" placeholder prices like $0.10 while real prices exist).
6. **Cross-source tighten** — if 2+ survivors remain but still disagree wildly (max > 4× min),
   keep only those within `[median × 0.5, × 2]`.
7. **Corroboration** — only trust the noisy sold-search source when **3+** total sources survived;
   otherwise drop it and let the direct-URL sources speak.
8. **Weighted average** — blend the survivors by **source reliability weight**, then apply a
   **× 0.945** factor (canonical price set ~5.5% below the weighted average).
9. **Fallbacks** — if every today-source was rejected, fall back to `historical median × 0.945`
   (last-known-good). If there's no history and only one today-source, use it as-is (best we can do
   for a first-time product).

**What we're keeping vs. changing.** The *shape* — snapshots → windowed → outlier-filtered →
weight-blended → discounted — is proven and stays. What changes: the hard-coded per-source bands,
the "which source is noisy" flag, and the reliability weights become **fields on the Source model**
(see below), and the final "which price is right" judgment can be handed to the **orchestration
layer** (step 8+) so the AI reasons about disagreeing sources instead of a fixed median rule
deciding for it. The deterministic algorithm remains the **cheap default and the fallback**; the
AI is the escalation for ambiguous cases.

---

## Data model

Two new models, both `Brand`-scoped and admin-managed (created/edited in nanza-admin, same audit
`createdById` / `lastModifiedById` pattern as the rest of the catalog).

### `Source`

A configured place we fetch intelligence from. Admin-curated, filterable by type.

| Field           | Type                | Notes |
|-----------------|---------------------|-------|
| `id`            | id                  | |
| `name`          | string              | e.g. "TCG Sold Listings", "eBay Sold", "Card News Feed" |
| `type`          | `SourceType` enum   | **default `PRICING`**; drives which loop fetches it and which insight it produces |
| `weight`        | float (0–1)         | reputability / reliability. More reputable source ⇒ higher weight. Feeds both the deterministic weighted-average AND is passed to the AI so it can reason about trustworthiness when sources disagree. |
| `baseUrl` / `config` | string / json  | how to reach it (URL template, search params, selectors) — the "how to fetch" spec |
| `isActive`      | boolean             | loops only run active sources |
| `brand`         | Brand relation      | sources can be brand-specific (a One-Piece price site vs a Flesh-and-Blood one) |

### `Insight`

A trusted, product-associated intelligence datum the orchestration layer decided to keep.

| Field           | Type                | Notes |
|-----------------|---------------------|-------|
| `id`            | id                  | |
| `type`          | `SourceType` enum   | **default `PRICING`**; the **same** enum the source uses (see below) |
| `product` / `entity` | relation       | the card this insight is about |
| `value`         | json                | type-specific payload — for `PRICING`, the chosen price + provenance; for `ARTICLE`, extra body/summary; etc. |
| `sourceUrl`     | string              | the URL this insight came from — the specific page/listing/post, not the source's base URL. Powers click-through and provenance. |
| `title`         | string (opt.)       | extracted meta title (OG `og:title` / `<title>`) |
| `description`   | string (opt.)       | extracted meta description (OG `og:description` / meta description) |
| `image`         | string (opt.)       | extracted meta image (OG `og:image`) — gives insights a rich card look in the UI |
| `sourceIds`     | source refs         | which sources contributed (audit / explainability) |
| `confidence`    | float (opt.)        | orchestration's confidence in this insight |
| `createdAt`     | timestamp           | insights are time-stamped; pricing insights supersede prior ones |

The `sourceUrl` + `title` / `description` / `image` meta fields are extracted from the fetched page
(same OG-tag scrape the marketplace already does for share previews). They let a non-pricing insight
— an article or social post — render as a rich preview card in admin and on the product page, rather
than a bare link. For `PRICING`, `sourceUrl` points at the listing/page the chosen price came from.

For `type = PRICING`, the orchestration's decision **also writes `Product.price`** directly (that's
the canonical field the marketplace reads) — the `Insight` row is the explainable record of *how*
that price was chosen. Non-pricing insights (article, event, social) live only as `Insight` rows
associated to the product and surface in admin / on the product page.

### `SourceType` enum — one enum for both models, designed to scale beyond pricing

```prisma
enum SourceType {
  PRICING       // default — market/sold/retail price feeds
  EVENT         // tournaments, releases, set drops
  ARTICLE       // news, deck guides, editorial coverage
  SOCIAL_POST   // social chatter / mentions / hype signals
}
```

**One enum, used by both `Source.type` and `Insight.type`** — a `PRICING` source produces a
`PRICING` insight, so there's no reason for a parallel `InsightType`; sharing `SourceType` keeps them
from drifting. **`PRICING` is the default and the only type built first.** The enum's whole point is
that adding `EVENT`, `ARTICLE`, or `SOCIAL_POST` later is a config change (add sources of that type,
register a fetcher, register an orchestration prompt) — **no new tables, no schema migration to the
Source/Insight shape.** A `SOCIAL_POST` source is fetched by the same loop, judged by the same
orchestration layer, and stored as the same `Insight` row — only the `value` payload and the
"which is best" prompt differ.

---

## Per-product loop system (local Claude, not the API)

The fetch/enrich cycle runs on **local Claude (Claude Code)** on our own machine, **not** the
Anthropic API — running a hundred-thousand-product enrichment through the paid API would be far too
expensive. The same local-Claude capability being used to build this design is what runs the loops.

**Loops are per-product, not per-source-type.** Instead of "run all pricing sources, then all news
sources," the loop iterates products in graceful incremental batches. For each product it builds
**one spec object** carrying everything intelligence needs (the entity, its tags, and the set of
active sources relevant to it), then fetches every relevant source type for that product in that
pass. This keeps one card's whole intelligence picture coherent and lets us pace by product count.

- **Active-loop discovery** — the loop system finds active loops and runs them on a schedule across
  the day (the pricing loop in the morning; later, the news loop; etc.). Each loop targets sources
  by `type` and `isActive`.
- **Graceful pacing** — with ~100k products we never fetch all at once. Incremental batches, rate
  limited, so the system is *continuously* fetching and refreshing rather than spiking. Naturally
  prioritizes the stalest products so coverage gaps close on their own.
- **Output** — each fetch produces raw candidate data (e.g. 25 prices from 25 pricing sources, or
  30 candidate articles), handed to the orchestration layer — never written to the product blindly.

---

## Agent orchestration layer

When candidate data comes back, a **team of agents decides which insights are valuable** before
anything touches the product. The structure:

- **Lead ("manager") agent — Opus.** Owns the decision for a product's candidate set. Sees the
  candidates *and the source weights*, and reasons about which to trust.
- **Worker agents — Haiku.** Cheap, parallel workers that do the per-candidate legwork the lead
  delegates (normalize a price, dedupe listings, extract an article's gist, score relevance).

Per type, the orchestration does the type-specific "pick the best" job:

- **`PRICING`** — given e.g. 25 prices back, the AI determines *the right price* rather than
  mechanically averaging. It weighs each source's `weight`, spots outliers (graded-slab
  contamination, wrong-card matches, stale placeholders) the way the deterministic rules do — but
  with judgment for the ambiguous cases the fixed bands can't resolve. Result → `Product.price` +
  a `PRICING` `Insight` recording the chosen value and which sources backed it. The deterministic
  algorithm above stays as the **cheap path and the fallback** when the set is unambiguous.
- **`ARTICLE` / `EVENT` / `SOCIAL_POST`** — given e.g. 30–100 candidate stories/posts, the AI picks
  the **best few** worth associating with the product and writes them as `Insight` rows. Same
  weight-aware, orchestrated selection; only the prompt and the payload shape differ.

The **source `weight`** is the connective tissue: it feeds the deterministic blend *and* is handed
to the orchestration so the AI understands "this source is more reputable than that one" when it
dictates the best possible price/story/insight.

---

## Key decisions & rationale

- **One `Source` + one `Insight` model, both typed by the single `SourceType` enum — not per-domain
  tables, not a parallel `InsightType`.** A `PricingSource` and a separate `NewsSource` would
  duplicate the loop, the weighting, and the orchestration; a second enum would just drift from the
  first. One shared enum keeps it DRY and makes `EVENT` / `ARTICLE` / `SOCIAL_POST` a config addition.
- **`PRICING` is the default and the only type built first.** We ship the known-good price rollup
  end to end before generalizing; the enum just guarantees we won't have to reshape the model to
  grow.
- **`weight` lives on the source, consumed twice.** Deterministic weighted-average uses it; the AI
  is *also* given it so ambiguous disagreements are resolved with reputability in mind.
- **Loops run on local Claude, not the paid API.** Cost — enriching ~100k products via the API is
  prohibitive; the local Claude-Code capability runs the loops for free.
- **Per-product, not per-source-type, batching.** One coherent spec object per card, graceful
  incremental pacing, continuous refresh instead of spikes.
- **Keep the deterministic algorithm as default + fallback; use the AI as escalation.** We don't
  throw away a proven, cheap rollup — the orchestration is the judgment layer for the cases fixed
  bands can't settle, and the safety net when the AI/loop is unavailable.

## Status / remaining

Design stage. Nothing built yet. Build order: (1) `Source` + `Insight` Prisma models + enums,
default `PRICING`; (2) nanza-admin Sources section (type filter, weight editing) + Insights view on
the product; (3) the snapshot store + deterministic rollup ported from the known-good algorithm
above; (4) the local-Claude per-product pricing loop; (5) the Opus-lead / Haiku-worker orchestration
for `PRICING`; then, later, register `EVENT` / `ARTICLE` / `SOCIAL_POST` sources against the same
machinery.

## Related

- [[../INDEX|nanza-api]] · [[../architecture|Architecture]]
- [[../../nanza-admin/solution-designs/sources-and-insights|Admin surface for Sources & Insights]]
