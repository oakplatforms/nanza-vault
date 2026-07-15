---
tags: [nanza-admin, solution-design, sources, insights, pricing]
---

# Sources & Insights (Admin surface) — Solution Design

> **⚠️ DEFERRED (2026-07-05 pivot).** Sources and Insights now live in the separate
> [[../../oak-cortex/solution-designs/insight-engine|oak-cortex]] database, not the main DB —
> so this admin surface is on hold. A future operator UI would read from cortex (or a cortex
> API), not from nanza-api. Kept for the UI patterns it captures.

> The canonical, cross-repo design (data model, loop system, agent orchestration, the current
> price algorithm we're porting) lives in **[[../../nanza-api/solution-designs/sources-and-insights|nanza-api → Sources & Insights]]**. This
> page covers only the **admin console surface** for it. One design, two surfaces.

## What the admin adds

nanza-admin gains the operator UI for configuring **Source**s and reviewing the **Insight**s the
loops produce — same feature shape as the existing catalog pages (`src/app/<Feature>/` with a
`data/` React Query hook folder, `<resource>Service` in `src/services/api/`, Cognito-audited
writes).

- **Sources page** (`src/app/Sources/`) — list + create/edit dialog for sources. Fields mirror the
  `Source` model: `name`, `type` (a `SourceType` select, **default `PRICING`**, with `EVENT` /
  `ARTICLE` / `SOCIAL_POST` available so the section scales beyond pricing without a rebuild),
  `weight` (0–1 reputability, numeric input — the field the AI weighs when judging sources),
  `baseUrl` / `config`, `category` (sources are category-scoped, not brand-scoped — see canonical
  design), and `isActive`. A **type filter** on the list so operators can view just pricing
  sources today.
- **Insights on the product** — the Products page gains an Insights panel showing the `Insight`
  rows associated to the selected entity/product: the chosen `PRICING` insight (with which sources
  backed it — the explainability the orchestration writes) and, later, `ARTICLE` / `EVENT` /
  `SOCIAL_POST` insights. `Product.price` stays the canonical price field the marketplace reads;
  the pricing insight is the "why".
- **Services / types** — a new `Source.ts` (and `Insight` read) `<resource>Service` through the
  shared `fetchData`; DTOs + `*Payload` write shapes added to `src/types/index.ts` from
  `@oakplatforms/types` once the API models ship.

## Navigation

Add `Sources` to the `TopNavbar` `navigation` array and the `Sidebar`, and register a `/sources`
route in `AppLayout` (the two nav definitions are maintained separately from the route table — keep
them in sync). A `create` deep-link (`/sources?create=true`) follows the existing create-dropdown
pattern.

## Related

- [[../../nanza-api/solution-designs/sources-and-insights|Canonical design (nanza-api)]]
- [[../INDEX|nanza-admin]] · [[../architecture|Architecture]]
