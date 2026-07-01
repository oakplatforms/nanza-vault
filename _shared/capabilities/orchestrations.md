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
