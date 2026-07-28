---
tags: [nanza-admin, solution-design, query-lists, homepage]
---

# Query Lists / Homepage (Admin surface) — Solution Design

> **Status:** Design only. The `QueryList` model and endpoints don't exist yet — see the canonical
> cross-repo model in [[../../nanza-api/solution-designs/query-lists|nanza-api — Query Lists]].

## Overview

A **Query List** is an admin-configured *dynamic* collection: instead of curating entities by hand
(as a `List` does), an operator defines the *rules* — which sources to pull from and how to filter
them — and the API resolves the matching listings/bids/bulk listings live at read time.

The admin's job is authoring: a builder for creating and editing Query Lists. Its first home is the
**Homepage** section — because the mobile/web homepage *is* the set of Query Lists flagged
`isPrimary`, ordered by `index`. So "editing the homepage" in the admin literally means creating
primary Query Lists and ordering them. The same builder will be reused elsewhere as Query Lists show
up in more surfaces over time.

## How it works

### The Homepage section

A new **Homepage** nav entry (the `/homepage` route is currently a stub) lists the `isPrimary` Query
Lists in `index` order, with drag-or-numeric reordering that writes `index`. Adding a row opens the
builder with `isPrimary` preset. This is the operator's model of the homepage: an ordered stack of
dynamic rows.

Non-primary Query Lists (used by other surfaces later) are authored with the same builder but managed
outside the Homepage list.

### The builder

Reuses the List editor shell — name, displayName, images (banner/logo/thumbnail), brand, description —
plus two query-specific controls:

- **Types** — a multi-select (checkboxes) over the source types: Listing, Bid, Bulk Listing, Entity
  (Set reserved, hidden until the resolver supports it). One or many; the selection is a union with
  no AND/OR. At least one is required.
- **Criteria** — a repeatable list of rows, each three inputs — **field** (Product Price, Entity Tag,
  Set, Group, Talent), **operator** (equals, not equals, >, ≥, <, ≤), **value** — with add/remove per
  row. A single **combinator** dropdown (`AND` / `OR`) above the rows decides whether an item must
  match *all* criteria or *any*. The operator dropdown is filtered by the chosen field (e.g. only
  `equals`/`not equals` for Group), matching the resolver's allowed matrix.

The canonical "between $10 and $100" case is two criteria rows (`Product Price > 10`,
`Product Price < 100`) with the combinator on `AND`.

### Data layer

Follows the repo's standard shape (see [[../architecture|Architecture]]): a `services/api/QueryList.ts`
module with `get`/`list`/`create`/`update`/`delete` (plus image upload/delete, reused from the List
service), consumed through `app/Homepage/data/` React Query hooks, with DTOs from `@oakplatforms/types`
once the API regenerates them. Reordering the homepage uses a batch update of `index` (mirroring the
existing `PUT /lists/batch` pattern).

## Key decisions & rationale

- **Homepage = primary Query Lists.** Rather than a bespoke homepage model, the homepage is just the
  `isPrimary` slice of Query Lists ordered by `index`. One concept, authored one way, reused
  everywhere — the admin "Homepage" section is a filtered view over Query Lists, not a separate thing.
- **Reuse the List editor shell.** Query Lists share List's presentational fields, so the builder
  reuses that UI and only adds the Types multi-select and the Criteria/combinator block — additive, not
  a new editor.
- **Two dimensions, one combinator (mirrors the model).** Types is a plain multi-select union; the
  `AND`/`OR` knob lives only on the criteria. The UI puts the combinator with the criteria rows, never
  near the type checkboxes, so the boolean has exactly one home.
- **Field-filtered operators.** The operator dropdown narrows to what each field supports, so the admin
  can't build an illegal criterion (e.g. `Group > x`) the resolver would reject.

## Related

- [[../../nanza-api/solution-designs/query-lists|nanza-api — Query Lists]] — the canonical model
  (`QueryList` + `QueryCriterion`, enums, resolver) this UI configures.
- [[../architecture|Architecture]] — the service → React Query → page pattern this builder follows.
- [[../INDEX|nanza-admin]]
