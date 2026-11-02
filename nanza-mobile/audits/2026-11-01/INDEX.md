---
tags: [nanza-mobile, audit]
---

# nanza-mobile audit — raw reports, 2026-11-01

Supporting material for [[../../solution-designs/codebase-audit-2026-11|Codebase audit — November 2026]].
A narrower follow-up to [[../2026-09-16/INDEX|2026-09-16]]: unused npm packages, directories,
components, exports, style classes, assets and repo leftovers only. One session, no
sub-agents; `knip` ran first, every finding was re-grepped by hand. Tree: `dev` at
`4f710f4d` + the uncommitted messaging-improvements edits.

- [[dead-code|Dead code, unreachable surfaces, assets, repo leftovers]] — the per-item tables: packages (none removable), 3 dead files, 3 barrel-hidden components, the unreachable Favorites screen, the unread ListingQuantityContext, the dead Header sticky-tabs path, i18n, fonts, `amplify/` + `vendor/`, dead exports, size since September.
- [[style-bucket-classes|Unreferenced style-bucket classes]] — per-bucket lists of the ~1,760 classes nothing references (cards.ts alone 545 of 729).
- [[knip-report|knip output]] — raw, with the false positives named up top.
