# Skills Catalog

A quick view of the skills we have to offer across repos. Summaries only — the runnable skill lives in each repo's `.claude/skills/`. Add a skill here first, then decide which repos it graduates into.

| Skill | What it does | Deployed in |
|-------|--------------|-------------|
| `principal-engineer` | Principal-level engineering standards & architecture principles to apply when designing/reviewing. Canonical across backends. | nanza-api, nanza-admin, nanza-web-app |
| `deploy` | Deploy workflow for the service. | nanza-api, nanza-admin, nanza-web-app |
| `prisma-migration` | Guardrails/steps for running Prisma migrations. | nanza-api, nanza-admin, nanza-web-app |
| `plan` (command) | Structured pre-code implementation plan with confidence scoring; reads CLAUDE.md + project INDEX/REFERENCE. | nanza-mobile, nanza-email |
| `ui-review` | Applies the design system's tiered rules (P0 style/token/typography = hard-fail, P1/P2 = advisory) when authoring, editing, translating a Figma frame, or before a PR. Verifies against CLAUDE.md rather than restating it. | nanza-mobile |
| `figma-sync` | The code→Figma→code pipeline; code-is-canonical, Figma edits are proposals. Two repo-specific halves of one story: **mobile** documents the in-simulator capture (dev-menu exporter attached with `npm run mobile:on`) and turns a pasted Figma link + "I changed …" into a measured implementation prompt; **web** documents `figma-capture.json` (routes, viewport tiers, scroll waypoints), the scripted Chrome capture — nobody takes a screenshot by hand — and the Dev Mode MCP return path. | nanza-mobile, nanza-web-app |

**Note:** the toolchain both `figma-sync` skills describe lives in a third repo, `nanza-figma-sync` (shared engine + per-project plugins + the web capture runner), which must sit beside the app repos — its README is the team setup guide. The skills are the per-repo halves; the engine is not duplicated. nanza-web-app has no `ui-review` — its design rules are not written down as rules yet, so there is nothing for one to verify against.

**Note:** `principal-engineer` is currently copy-pasted identically across three repos. Candidate for a canonical version documented here, synced out to repos.
