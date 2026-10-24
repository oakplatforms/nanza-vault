---
tags:
  - nanza-mobile
  - plan
  - search
  - posts
status: PLAN (2026-08-18) — awaiting approval. Fold into a `search-filters` design on landing and delete.
---

# Collapse-to-6 chip grid — one implementation, three call sites

> **Status: PLAN.** Not implemented.

## The ask

The composer's tag screen (`TagPickerLayer`) renders EVERY value in every section, so
"Defense" runs 11 chips deep and "Talent" further still before the next section is reachable.
The search filter screen already solves this: 6 chips, then a "Show more" pill that toggles
to "Show less".

## What exists

The pattern is already written TWICE, in near-identical form:

| Site | Count const | Extra behaviour |
|---|---|---|
| `SearchScreen/OpenTagSection.tsx:55` | `COLLAPSED_COUNT = 6` | paginates the rest in on expand; per-section Reset |
| `SearchScreen/Filter.tsx:126` (`renderSetSection`) | `SET_COLLAPSED_COUNT = 6` | same, for sets |
| `composer/TagPickerLayer.tsx` | — | none: renders all values |

Both existing copies share the same styles (`filters.ts` `filterGrid` / `filterGridItem` /
`filterGridShowMoreButton` / `filterGridShowMoreText`) and the same markup: section header
row, 2-up grid, pill with a chevron that hides when expanded. Adding a third hand-rolled copy
would be the wrong move under the "a pattern used 2+ times is a component" rule.

## The shape

One presentational component — `ChipGridSection` — owning: the section header row, the
collapse state, the 6-chip cut, and the Show more/less pill. Consumers pass items, a
selected-test, a press handler, and optional slots (`headerAccessory` for search's Reset,
`footer` for its loading spinner) plus `hasMoreBeyondLoaded` for the paginated cases and an
`onExpand` callback so pagination stays where the data lives.

Collapse state belongs INSIDE the component — the composer maps over N sections, so
per-section state in the parent would mean a keyed Set; a child instance carries its own.

## Selected chips must not hide

A section's 9th chip can be a committed pick. Collapsed, it disappears — the search screens
have this today. The composer feels it harder: it already floats sections holding a committed
pick to the top, so it would surface a section and then hide the pick inside it.

Decision: a section opens EXPANDED when it holds a pick that sits past the cut. It is a
deviation from search's behaviour, in the direction search should probably follow.
