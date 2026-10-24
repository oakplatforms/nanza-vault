---
tags: [oak-cortex, solution-design, sources, insights, pricing]
---

# oak-cortex — Insight Engine (canonical design)

> **This supersedes the data-model and write-path sections of
> [[../../nanza-api/solution-designs/sources-and-insights|nanza-api → Sources & Insights]]**
> (decision 2026-07-05, evening session). That doc remains the reference for the orchestration
> judgment rules and the legacy price algorithm; the models and the runtime now live *here*.

## The pivot: separate service + DB, one canonical key

The insight engine does **not** live in the main (nanza-api) database. It is its own project —
**oak-cortex** (successor to the legacy `oak-cortex` scraper repo) — with its own **Postgres +
Prisma** schema, its own **Lambda-backed API** (same design conventions as oak-api), and the same
API design conventions as nanza-api. Reasoning:

- **Safety / isolation (the decisive reason, reaffirmed 2026-07-06).** The AI loop makes many
  writes and runs *all day*. A separate DB is a hard **isolation boundary**: a bad write, a
  validation gap, a runaway loop, or an experiment gone wrong is physically contained in cortex and
  **cannot corrupt prod**, because prod is read-only to the entire system. This is a stronger
  guarantee than "we wrote careful endpoints" — it's structural, not procedural, and can't be
  retrofitted cheaply later. Prod stays clean by construction while the AI is free to experiment,
  research, and iterate against every entity. (Considered + rejected: source/insight tables in the
  main DB with scoped write endpoints — simpler, fewer moving parts, but the safety would come from
  the review step alone, not from isolation, and messy 30+-source scrape data would live next to
  the clean catalog. For an always-on AI writer, structural isolation won.)
  The loop holds **zero credentials to the production database**, and **nothing writes up to prod
  at all** except reviewed migrations. Data flows one way: main → cortex.
- **Oak-level, not nanza-level.** Main is *Oak's* canonical catalog; nanza is the first project
  in it, with more to come. Cortex serves any Oak project against the same canonical IDs.
- **History is free here.** Isolated from prod, cortex can keep every raw reading and every
  superseded insight — the historical layer we explicitly kept *out* of the main DB.

**Canonical keys:** `entityId` + `productId` from the main database. Every cortex row that refers
to a catalog item carries main's IDs verbatim, so anything cortex concludes can be referenced back
up later.

### Infrastructure: two databases (dev + prod), mirroring main (decided 2026-07-06)

**`oak-cortex-dev` and `oak-cortex-prod` — two separate Postgres databases**, same split main
already runs (`oak-api-dev` / `oak-api-prod` stacks + their separate data). Cortex down-syncs its
`CatalogItem` rows *from* main, so cortex-dev must sync from main-dev and cortex-prod from main-prod
— one shared cortex DB would let dev test runs pollute prod insights/prices. Same mental model as
the existing dev/prod split; no new pattern. The cortex API is a Lambda stack (e.g. `oak-cortex`
service, `oak-cortex-dev` / `oak-cortex-prod` stacks) paralleling oak-api.

### Admin surface: ONE admin app, two API base URLs (decided 2026-07-06)

**Do NOT fork nanza-admin.** nanza-admin never touches a database — it calls an *API* through a
single `fetchData` wrapper with one base URL (`REACT_APP_API_BASE_URL`), and each service file
(`Brand.ts`, `Entity.ts`, …) just hits a path. So "point the admin at two databases" is really
"point it at two APIs": add an **optional per-call base URL** to `fetchData`, keep the main oak API
as the default, and have the new **`Source` / `Insight` service files pass the cortex API URL**
(`REACT_APP_CORTEX_API_BASE_URL`). One admin app, two backends, cleanly separated — no fork, no
duplicated Cognito auth, no second deploy. The old "Sources in nanza-admin against the main DB"
idea is dropped; the sources/insights admin screens now read/write the cortex API instead.

## Data flow

```
main DB (nanza-api, prod)
   │  1. DOWN-SYNC — read-only spec endpoint (new, nanza-api):
   │     GET a flat ~15–20-field spec object per entity
   ▼
oak-cortex DB  ── 2. FETCH: per-entity, per-source AI page reading → SourceReading rows
   │            ── 3. ORCHESTRATE: judge candidates vs current insights → replace Insight rows
   │
   └─ 4. UP-FLOW (manual for now): generate a reviewed SQL migration
        (UPDATE "Product" SET price = … WHERE id = …) keyed by main's IDs;
        the user reviews + runs it against prod. Automated later via a scoped
        API write once trusted. NOTHING else touches prod.
```

- **Down-sync (1)** is the only prod involvement: one **read-only** endpoint in nanza-api that
  transforms an entity into the flat spec shape (below). No auth risk — it reads.
- **Up-flow (4)** matches the existing workflow: the user already reviews and runs every
  migration by hand. A bad AI day produces a bad *proposal file*, never a bad prod write. Cortex
  also acts as the recovery store — if prod pricing is ever lost, current prices per entity are
  sitting in cortex ready to migrate up.

## The spec object (main → cortex)

A **new, dedicated endpoint** in nanza-api (not `include=` composition — the point is one flat,
stable, transformed shape). **Field list to be finalized by the user**; working draft (~18 fields):

`entityId, productId, clientKey ("nanza"), gameName (brand), categoryName, entityName, setName,
setCode, productNumber, tags { printing/foil, edition, rarity, color|pitch, cardType, artist, … },
image, currentPrice (Product.price), lowestAsk, highestBid, referenceCode, updatedAt`

This object is what the fetch layer compares against a scraped page to verify "this page really is
our card" (product number, edition, foil, defense values, artist — whatever metadata we hold), and
what the orchestration uses as context.

## Cortex data model (draft)

Same Prisma conventions as nanza-api (cuid ids, `createdAt`/`updatedAt`, `Decimal @db.Decimal(10,2)`
money, enums in their own section, no inline schema comments).

- **`CatalogItem`** — the down-synced spec snapshot **and the flat current-price record**, keyed
  by **main's `entityId`** (PK) + `productId`. Carries **`clientKey`** (e.g. `nanza`) straight from
  the spec object — cortex holds **no client/project registry of its own**; main owns that
  hierarchy (see [[../../_shared/decisions/client-project-hierarchy|Client → Project decision]]).
  Plus the flat spec fields + `spec Json`; `syncedAt`. **Pricing fields live right here** (decided
  2026-07-06 — no separate PriceRecommendation table; CatalogItem is already one row per product,
  so the recommendation is just fields on it):
  - `mainPrice` — Product.price at last sync = the price prod has now / the "original price you
    had" + the cold-start pricing anchor.
  - `recommendedPrice` — today's loop decision.
  - `previousRecommendedPrice` — yesterday's, for at-a-glance drift.
  - `priceInsightId` — → the PRICING `Insight` that justifies the recommendation.
  - `priceStatus` — PENDING / PROPOSED / APPLIED / REJECTED (the migration state machine).
  - `priceDecidedAt`.

  The up-flow migration is then one trivial query over this table:
  `SELECT productId, recommendedPrice FROM CatalogItem WHERE priceStatus='PROPOSED'` → emit
  `UPDATE "Product" SET price=… WHERE id=…`. The review reads naturally — `mainPrice` (original)
  vs `recommendedPrice` (new), side by side, per product. **Why on CatalogItem, not a new table:**
  it's already 1-row-per-product keyed by the same ID; a separate table would be a parallel row to
  keep in sync for no gain. `Insight` stays separate (it's the rich *explanation*: sources,
  confidence, ≤3 kept — wrong shape for a fast price dump); CatalogItem's price fields are the flat
  *decision of record*.
- **`Source`** — category-scoped fetch target: `name`, `type SourceType @default(PRICING)`,
  `weight`, `isNoisy`, `isActive`, `baseUrl`, `config Json`, `category` (main's category key),
  `clientKey`, and **`notes`** — a free-text/JSON field the **orchestration writes back
  dynamically**: learned extraction hints ("price lives in the `var meta` Shopify blob", "search
  URL pattern is …", "this site needs a rendered page"). Notes make repeat fetches cheap and
  self-healing — the AI reads them before fetching and updates them when a site changes.
- **`SourceReading`** — raw per-fetch observation (the snapshot layer, back for free): `catalogItemId`,
  `sourceId`, `value Json` (price for PRICING), `url`, `matched Boolean` + `matchNotes` (did the
  page verify against the spec?), `accepted Boolean` + `reason` (did orchestration use it?),
  `observedAt`. Append-only; this is cortex's history.
- **`Insight`** — the current conclusions, exactly as designed in the nanza-api doc: `type`
  (`SourceType`), `catalogItemId`, `value Json`, `sourceUrl`, `title`/`description`/`image` (OG
  meta), contributing source refs, `confidence`. **≤ ~3 per (item, type), pricing included**;
  replacement is a judgment (relevance to the entity beats recency; sanity check against the last
  3 price conclusions + `mainPrice` — a $1.50 → $72 jump means re-examine sources, don't write).
  Because history is free in cortex, replaced insights are **archived (`archivedAt`), not
  deleted** — the ≤3 *active* rows are the current picture; archived rows are the trend history
  that will feed future historical metrics.
- **`MainSyncBatch`** — each generated up-flow migration run: `payload Json` (the set of
  productId → recommendedPrice included), `sqlPath`, `status` (PROPOSED / APPLIED), `appliedAt` —
  the audit trail of exactly what cortex proposed to prod and when it was applied. Flipping a batch
  to APPLIED flips the included `CatalogItem` rows' `priceStatus` to APPLIED too.

`SourceType` enum unchanged: `PRICING` (default) / `EVENT` / `ARTICLE` / `SOCIAL_POST`.

## Fetching: AI-driven extraction, per source

Google Shopping as a single meta-source was **tried and walked back** (results too weak) — cortex
stays **per-source**. But unlike legacy oak-cortex, fetching is not hand-coded selectors per site:

1. The loop takes the entity's spec object + the source's `notes`.
2. It reaches the card's page (search → click-through, or a stored per-item URL), rendering via
   headless browser when the site needs it.
3. The AI reads the page against the spec object — verifies product number / edition / foil /
   set / other metadata dynamically, no per-site selectors — and extracts the price (or article,
   etc.).
4. What it learned about the page's structure goes back into `Source.notes`; the raw observation
   becomes a `SourceReading`.

Escalation ladder per page: cached hint from `notes` → AI reads fetched HTML/embedded JSON →
rendered-page read → screenshot + vision (the legacy fallback that already works).

## What stays where

| Concern | Home |
|---|---|
| Catalog truth (Entity/Product/tags/price the marketplace reads) | main (nanza-api) |
| Spec read endpoint (new, read-only) | nanza-api |
| Source, Insight, SourceReading, history, source notes | **oak-cortex DB** |
| Fetch loop + orchestration (local Claude) | **oak-cortex repo** |
| Price → prod | reviewed SQL migration (manual now, automated later) |
| Admin UI for sources/insights | **existing nanza-admin**, new Source/Insight service files pointed at the **cortex API base URL** (no fork) |
| Cortex API (sources/insights CRUD + sync + up-flow) | **oak-cortex** Lambda stack (dev + prod) |
| `@oakplatforms/types` | main schema **unchanged**; cortex models generate cortex's own types (separate package or local) |

## Status / build order

✅ (1) SCAFFOLDED 2026-07-06 at `~/Projects/oak-cortex-service` (sibling repo). Prisma schema
(CatalogItem w/ price fields, Source w/ notes, SourceReading, Insight, MainSyncBatch + SourceType /
PriceStatus / SyncBatchStatus enums), Express app + serverless-express Lambda handler, pg/Secrets-
Manager prisma client (secret `oak-cortex-credentials-<stage>`), serverless.yml (`oak-cortex`
service, dev+prod). Routers: `/sources` `/insights` `/catalog-items` CRUD + `POST /catalog-item/sync`
(down-sync upsert) + `GET /price-recommendations` (flat migration feed). `prisma validate` + `tsc`
clean. NOT git-init'd, no DB provisioned, not deployed. Remaining below.

Build order. (1) ✅ scaffold — done; provision **dev + prod cortex Postgres** + set the secrets; (2) the read-only spec endpoint in
main + down-sync into `CatalogItem`; (3) `Source` seeding (category-scoped, weights from the legacy
system) + the AI fetch loop with `notes` write-back; (4) orchestration for `PRICING` (judgment
rules per the nanza-api doc) + the `MainSyncBatch` migration generator; (5) wire the
sources/insights CRUD into **nanza-admin** via the cortex API base URL (no fork); later,
EVENT/ARTICLE/SOCIAL_POST against the same machinery, then automated up-flow once trusted.

The loop start-to-finish, restated: fetch entities + tags + canonical info from **main** (spec
endpoint) → build the flat spec object → pass it down to the **cortex** service (Lambda on the
cortex DB) → cortex finds sources for that entity, fetches + AI-reads each, writes `SourceReading`s,
orchestrates `Insight`s, and proposes prices back up as a reviewed migration.

## Related

- [[../../nanza-api/solution-designs/sources-and-insights|nanza-api → Sources & Insights]] —
  orchestration judgment rules + the legacy price algorithm (heuristics reference)
- [[../INDEX|oak-cortex]] · [[../../_shared/architecture|Shared architecture]]
