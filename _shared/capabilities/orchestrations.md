# Orchestration Patterns

Summaries of multi-agent / multi-step orchestration patterns we use. The *pattern* is a reusable playbook documented here; the runnable agents live per repo in `.claude/agents/`.

## Card-data migration pipeline (nanza-api)
A staged pipeline coordinated by `migration-orchestrator`, fanning out to dedicated sub-agents in dependency order:

1. `migration-sets` — insert `Set` records (runs first).
2. `migration-entities-products` — insert `Entity` + one `Product` each (looks up `Set.id`).
3. `migration-supported-tag-values` — add new `SupportedTagValue` rows (diff-shaped).
4. `migration-entity-tags` — attach per-card tag values as `EntityTag` rows (batched).
5. `migration-images` — download, rename to `<entityId>.<ext>`, emit `UPDATE Entity.image`.

Plus `migration-fix-comma-values` as a one-off cleanup for comma-joined tag values.

**Pattern shape:** orchestrator → ordered stages with hard dependencies → each stage is idempotent DBeaver SQL, generated not executed (human runs it). Good template for any staged, dependency-ordered data migration.

## Tag-metadata pipeline (nanza-api)
A second, separate orchestration (`migration-metadata-orchestrator`) added 2026-08-18. It runs off a different source — a curated supported-tags metadata CSV (`Brand Tag, Supported Tag Value, Description, …`) — and *enriches* the SupportedTagValues the card pipeline created, rather than inserting anything:

1. CSV prep (inline) — split `Set` rows into `sets.csv`, drop comma-joined values (set names legitimately contain commas, so sets split first).
2. `migration-tag-value-metadata` — idempotent `UPDATE` SQL setting `description` + `isPrimary`, matched by brand name + tag name + value slug with a case-insensitive `displayName` fallback (values like `*` or Japanese text slugify to empty). Unmatched CSV values surface in a MISSING report — reported, never inserted.
3. Planned: children (`SupportedTagValueParent`) and set metadata stages.

**Why separate:** different source CSV, different cadence, and it grows its own stages — keeping it out of the card pipeline's dependency table. Both pipelines share the same conventions (per-release folders, prevalidate → batches → postvalidate, names-not-ids, generate-don't-execute).

## Card-pricing pipeline (nanza-api)
A third orchestration (added 2026-08-20), unrelated to the DB migrations — an operator-side *research* pipeline over the open web. `pricing-orchestrator` takes a CSV (≤50 cards) or a zip of such CSVs, splits/validates, and fans out one `card-pricer` agent per CSV **in parallel**; each pricer web-searches current market value and emits a single recommended `price_usd` per card (fair NM midpoint × 0.90, per the methodology). Results merge into `pricing/runs/<date>-<name>/` with `all-priced.csv` + `run-summary.md`.

**Pattern shape:** orchestrator → parallel identical workers over ≤50-row shards (vs. the migrations' *ordered dependency* stages) → methodology lives in the vault ([[../nanza-api/solution-designs/card-pricing|Card Pricing]]) and is read by workers at runtime, so tuning the rules is a doc edit, not an agent edit. Good template for any fan-out-over-shards research task. No DB writes, no SQL — files only.

**Model tiering (2026-08-20):** workers pin `model: haiku`, the orchestrator pins `model: sonnet`, and flagged/low-confidence rows are escalated to one sonnet-override worker pass — cheap models do the bulk, the expensive model only sees the hard tail. First pipeline to tier explicitly (migration agents inherit the session model); apply the same pattern there if migration runs ever feel expensive.
