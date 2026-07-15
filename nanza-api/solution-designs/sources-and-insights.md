---
tags: [nanza-api, nanza-admin, solution-design, sources, insights, pricing]
---



# Sources & Insights — Solution Design

> **⚠️ ARCHITECTURE PIVOT (2026-07-05, evening): the models and runtime move to a separate
> project — see [[../../oak-cortex/solution-designs/insight-engine|oak-cortex → Insight Engine]]
> (now canonical for the data model and write path).** `Source` / `Insight` / readings / history
> live in oak-cortex's own Postgres+Prisma DB, keyed by main's `entityId`+`productId`. nanza-api's
> whole involvement shrinks to: (a) a **new read-only spec endpoint** returning a flat ~15–20-field
> object per entity (field list TBD by the user — a dedicated transformed shape, not `include=`
> composition), and (b) **receiving reviewed SQL price migrations** — the AI never writes to prod.
> Google Shopping as a single meta-source was tried and walked back — fetching stays per-source
> with AI-driven extraction + self-updating `Source.notes`. The admin Sources/Insights surface is
> **deferred**. The sections below remain accurate for the **orchestration judgment rules** and
> the **legacy price algorithm**; read data-model/write-path/admin statements through the pivot.

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
(so the judgment it encodes carries over), then generalizes.

**Scope decision (2026-07-05): no snapshot store, no ported rollup job — build the insight engine.**
The 14-day snapshot history and the deterministic rollup stay in oak-cortex, which keeps running the
old pricing system until cutover. The new system is the **insight engine**: each run, the
orchestration layer judges the fresh values fetched that day — anchored by the *current insights*
already on the entity (see below) — and writes `Product.price` + replaced `Insight` rows directly.
Historical metrics are explicitly out of scope for this database; when we want them later, they'll
live in a separate document store, not here.

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

**What we're keeping vs. changing.** The *judgment* this algorithm encodes — outlier bands, noisy
sources need corroboration, placeholder-price floors, reliability weighting, a ~5.5% discount off
the blend — carries over as **heuristics the orchestration layer applies**, and the "which source is
noisy" flag and reliability weights become **fields on the Source model** (see below). What does
*not* carry over (for now): the snapshot table and the ported rollup job. The new system keeps no
per-day price history — the anchor role the 14-day historical median played is filled by the
**current `PRICING` insight** (what the engine concluded yesterday), which is fed back into each
run with strong weight. oak-cortex keeps running the snapshot system as the safety net until
cutover.

---

## Data model

Two new models, admin-managed (created/edited in nanza-admin, same audit
`createdById` / `lastModifiedById` pattern as the rest of the catalog). **Sources are
`Category`-scoped, not brand-scoped** (decision 2026-07-05): the ~30 pricing sources that serve TCG
serve *every* TCG brand, and the same holds for toys, etc. — scoping per brand would mean
duplicating near-identical source rows across every brand in a category. The existing `Category`
model (`schema.prisma`) is the scope; `Entity.categoryId` already exists, so the per-product loop
resolves an entity's sources via its category. Insights attach to the **Entity**.

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
| `category`      | Category relation   | sources belong to a **category** (TCG, toys, …), not a brand — most brands in a category share the same sources. Brand-specific quirks (a One-Piece-only price site) are still just a source in that category; the fetch spec in `config` can narrow further if needed. |

### `Insight`

A trusted, product-associated intelligence datum the orchestration layer decided to keep.

| Field           | Type                | Notes |
|-----------------|---------------------|-------|
| `id`            | id                  | |
| `type`          | `SourceType` enum   | **default `PRICING`**; the **same** enum the source uses (see below) |
| `entity`        | Entity relation     | the card this insight is about (Entity, not Product — listings/bids/tags/brand scope all hang off Entity, and `include=insights` comes free on the entity fetch) |
| `value`         | json                | type-specific payload — for `PRICING`, the chosen price + provenance; for `ARTICLE`, extra body/summary; etc. |
| `sourceUrl`     | string              | the URL this insight came from — the specific page/listing/post, not the source's base URL. Powers click-through and provenance. |
| `title`         | string (opt.)       | extracted meta title (OG `og:title` / `<title>`) |
| `description`   | string (opt.)       | extracted meta description (OG `og:description` / meta description) |
| `image`         | string (opt.)       | extracted meta image (OG `og:image`) — gives insights a rich card look in the UI |
| `sourceIds`     | source refs         | which sources contributed (audit / explainability) |
| `confidence`    | float (opt.)        | orchestration's confidence in this insight |
| `createdAt`     | timestamp           | insights are time-stamped; see replacement rule below |

**Insights are replaced, not accumulated — capped at ~3 per (entity, type)** (decision
2026-07-05). An insight is the *conclusion* ("this is the price", "these are the stories worth
showing") — not a log of runs. Each entity keeps **up to 3 insights per type, `PRICING`
included** — the recent few double as the engine's short-term reference points (the last three
price conclusions are the sanity anchor; the current three stories are the bar a new story must
clear). Replacement is a **judgment, not a rotation**: when a run surfaces a new candidate, the
orchestration weighs it against the existing three and decides whether it earns a slot — usually
evicting the oldest, but **relevance beats recency**: a story exactly about the entity keeps its
slot over a newer story that's merely about the entity's set. If the new candidate doesn't clear
the bar, nothing is written. The table therefore holds each entity's current intelligence picture,
bounded per entity, and never bloats with per-run history. Historical trend metrics are a future,
separate concern — a document store outside this database.

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
anything touches the product.

**Current insights are an input, with strong weight.** Each run starts by reading the entity's
existing `Insight` rows — every type: the last few price conclusions, the articles/events/posts
already chosen — **plus the current `Product.price`** — and hands them to the orchestration
alongside the fresh candidates. `Product.price` matters especially on the cold start: the very
first run has no prior insights, but the product already carries a price (set by the outgoing
system), so the pricing judgment is never anchor-less. The recent price conclusions +
`Product.price` are the anchor that replaces the old 14-day historical median, and they power a
hard **sanity check**: if the last three pricing insights say ~$1.50 / $1.25 and today's candidates
imply $72, the engine does *not* write — something went wrong (wrong listing matched, graded slab,
bad parse). It goes back to the sources, re-examines the fetches against the current insights, and
only writes once the discrepancy is explained. For non-pricing types, the current insights are the
bar a new candidate must clear (see the replacement rule in the data model): the engine reads the
new story against the three it already trusts and judges whether it's relevant enough to evict
one — entity-specific beats set-level, even when the set-level story is newer. Prior insights
carry strong weight, but they don't veto — genuinely better fresh evidence wins the slot.

The structure:

- **Lead ("manager") agent — Opus.** Owns the decision for a product's candidate set. Sees the
  candidates *and the source weights*, and reasons about which to trust.
- **Worker agents — Haiku.** Cheap, parallel workers that do the per-candidate legwork the lead
  delegates (normalize a price, dedupe listings, extract an article's gist, score relevance).

Per type, the orchestration does the type-specific "pick the best" job:

- **`PRICING`** — given e.g. 25 prices back plus the last ~3 `PRICING` insights and the current
  `Product.price` (the anchor even when no insights exist yet), the AI determines
  *the right price* rather than mechanically averaging. It weighs each source's `weight`, anchors
  against the recent conclusions (with the sanity check above), and spots outliers (graded-slab
  contamination, wrong-card matches, stale placeholders) the way the deterministic rules do — but
  with judgment for the ambiguous cases fixed bands can't resolve. Result → `Product.price` + a
  new `PRICING` `Insight` recording the chosen value and which sources backed it (oldest of the 3
  rolls off).
- **`ARTICLE` / `EVENT` / `SOCIAL_POST`** — given e.g. 30–100 candidate stories/posts plus the
  ~3 current insights of that type, the AI judges which candidates are relevant enough to earn a
  slot (relevance to *this entity* over recency), evicting the weakest/oldest only when beaten.
  Same weight-aware, orchestrated selection; only the prompt and the payload shape differ.

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
- **Sources scope to `Category`, not `Brand`** (2026-07-05). The ~30 pricing sources for TCG serve
  every TCG brand; per-brand rows would be near-duplicates across the whole category. Entity
  already carries `categoryId`, so source resolution per product is a category lookup.
- **No snapshot store, no ported rollup — insights replace history** (2026-07-05). oak-cortex keeps
  the snapshot system running until cutover. In the new engine, the current insights (fed back into
  each run with strong weight) play the anchor role the 14-day median played; historical metrics,
  when we want them, go to a separate document store, never this database.
- **Insights are replaced by judgment, capped at ~3 per (entity, type) — pricing included**
  (2026-07-05). The recent 3 are the short-term reference: pricing's sanity anchor, and the bar a
  new story must clear (relevance to the entity beats recency). New candidate in → weakest/oldest
  evicted only when beaten. Bounded rows per entity, no bloat at 100k products.
- **Loops run on local Claude, not the paid API.** Cost — enriching ~100k products via the API is
  prohibitive; the local Claude-Code capability runs the loops for free.
- **Per-product, not per-source-type, batching.** One coherent spec object per card, graceful
  incremental pacing, continuous refresh instead of spikes. Runs daily, paced through the day.
- **The deterministic algorithm survives as heuristics, not a job.** Its judgment (bands, noisy
  corroboration, floors, the 0.945 discount) informs the orchestration prompts; the safety net
  while the new engine proves out is oak-cortex itself, still running until cutover.

## Status / remaining

Design stage. Nothing built yet. Build order: (1) `Source` (category-scoped) + `Insight`
(entity-scoped) Prisma models + `SourceType` enum, default `PRICING`; (2) nanza-admin Sources
section (type filter, weight editing) + Insights view on the product; (3) the local-Claude
per-product loop (fetch per source, per entity, daily paced); (4) the Opus-lead / Haiku-worker
orchestration for `PRICING` — prior insights in, replaced insights + `Product.price` out; then,
later, register `EVENT` / `ARTICLE` / `SOCIAL_POST` sources against the same machinery. No snapshot
store and no rollup port — oak-cortex keeps running the old pricing system until the new engine is
verified, then it's decommissioned.

## Related

- [[../INDEX|nanza-api]] · [[../architecture|Architecture]]
- [[../../nanza-admin/solution-designs/sources-and-insights|Admin surface for Sources & Insights]]
