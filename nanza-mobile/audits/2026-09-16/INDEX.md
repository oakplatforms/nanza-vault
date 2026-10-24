---
tags: [nanza-mobile, audit]
---

# nanza-mobile audit — raw reports, 2026-09-16

Supporting material for [[../../solution-designs/codebase-audit-2026-09|Codebase audit — September 2026]].
Six read-only sub-agent sweeps over `src/` on branch `dev`, plus the knip output that seeded them.
Every count cites the command that produced it. Where reports disagree, the synthesis doc is authoritative
(it re-verified the tsconfig, image-reference and Proguard claims).

- [[dead-code|Dead code, unused files & assets, stale artifacts]] — 41 dead files, fonts, lottie/svg orphans, root leftovers.
- [[dependencies|Dependencies, toolchain, app size]] — per-package verdict table, pods, Android R8, asset sizes.
- [[network|Network & data layer]] — cold-start trace, full 97-query catalog, invalidation-key bugs, image widths.
- [[components|Component design standards]] — P0 sweep counts, bucket duplication analysis, standardization opportunities.
- [[performance|Runtime performance]] — style-getter rebuild cost with benchmark, contexts, lists, animation, startup.
- [[obsolete-practices|Obsolete practices & tooling hygiene]] — deprecated APIs, tsc/eslint breakdowns, native project hygiene.
- [[knip-report|knip output]] — the raw unused-files/exports/deps report (contains false positives; see dead-code §1).
