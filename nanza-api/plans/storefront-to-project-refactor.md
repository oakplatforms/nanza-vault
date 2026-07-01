# Storefront → Project Refactor

**Goal (this phase):** Replace the `Storefront` model with a more general `Project` model that
carries a `type` (storefront, gift, group, listing, collection, bid, lot). `storefront` becomes
one project type. Re-skin the router + shared types to Project, uncomment the (currently disabled)
AI-orchestration API so it goes live, and bring the **storefront builder experience** back on
`nanza-web-app` for testing — pointed at `nanza-api`.

**Out of scope (long-term north star, not built now):** subscription/token-metering business model
(free Claude token allowance → paywall, token markup covering Claude cost + Oak + licensee margin,
subscription unlocking zero commission). We keep this in mind while building but write none of it yet.

**Migration posture:** Clean drop-and-recreate. There is **no storefront data to preserve**. User
handles running migrations (Prisma migrate) themselves — we only edit `schema.prisma` and regenerate types.

---

## Key design decisions

| # | Decision | Confidence | Notes / rejected alternative |
|---|----------|-----------|------------------------------|
| 1 | Generalize `Storefront` → `Project` + add `ProjectType` enum; `StorefrontEdit` → `ProjectEdit`. Keep all existing fields (tree, publishedTree, theme, context, name slug, account/brand relations). | 0.9 | Rejected: keep `Storefront` and add a parallel `Project` model — leaves two overlapping concepts, violates DRY, the user explicitly wants Storefront *replaced*. |
| 2 | **Keep the rendering-tree schema named `StorefrontTree`/`StorefrontTheme`** (the 338-line `packages/types/src/storefront.ts`). It describes a *storefront page layout* — which is exactly the `STOREFRONT` project type's content. Other types define their own content later. | 0.85 | Rejected: rename tree → `ProjectTree`. Premature: a "deck" or "gift" project won't use this node set, so a generic name would be misleading. Surgical to leave it. |
| 3 | New enum `ProjectType { STOREFRONT, GIFT, GROUP, LISTING, COLLECTION, BID, LOT }`, default `STOREFRONT`. `tree`/`theme` stay required-with-default so the storefront type works unchanged; non-storefront types simply get the default empty tree for now. | 0.8 | Rejected: make `tree` nullable to accommodate type-less projects. Defers a decision we don't need yet and complicates the storefront path. Revisit when a second type ships. |
| 4 | Keep `StorefrontStatus` enum but rename → `ProjectStatus` (DRAFT/PUBLISHED/ARCHIVED). | 0.9 | Same values, just consistent naming. |
| 5 | Router file `storefront.ts` → `project.ts`; endpoints move `/storefronts*` → `/projects*`. Keep one GET filter `?type=storefront`. Web-app service + hooks renamed to match. | 0.85 | Rejected: keep `/storefronts` path for "compatibility" — nothing live depends on it (it's disabled), so no reason to carry the old name. |
| 6 | Web app: re-enable **only** the `/build` and `/build/:projectId` (storefront builder) routes. **Do NOT** re-enable the user-profile routes (`/:username`, `/listing/:id`, etc.) — user said the web app is a distribution surface, not a public profile storefront. | 0.9 | Those profile routes are a separate disabled concern; leave them commented. |

> Per CLAUDE.md, decisions #2/#3/#5/#6 are between 0.8–0.89; the rejected alternative is named for each.

---

## nanza-api changes

### A. `prisma/schema.prisma`
1. Rename enum `StorefrontStatus` → `ProjectStatus` (lines ~189-193).
2. Add enum:
   ```prisma
   enum ProjectType {
     STOREFRONT
     GIFT
     GROUP
     LISTING
     COLLECTION
     BID
     LOT
   }
   ```
3. Replace `model Storefront` (lines ~1572-1597) with `model Project`:
   - Add `type ProjectType @default(STOREFRONT)`.
   - `status ProjectStatus @default(DRAFT)`.
   - Keep `name @unique`, `displayName`, `tree`, `publishedTree?`, `theme?`, `context?`.
   - Relations: `account` / `brand` (unchanged), `edits ProjectEdit[]`.
   - Add `@@index([type])` alongside existing account/brand indexes.
4. Replace `model StorefrontEdit` (lines ~1599-1616) with `model ProjectEdit` (rename relation field `storefront`→`project`, `storefrontId`→`projectId`).
5. Update back-relations: `Account.storefronts Storefront[]` → `Account.projects Project[]` (line ~314); `Brand.storefronts Storefront[]` → `Brand.projects Project[]` (line ~595).

### B. Shared types (`packages/types/`)
6. `oak-api-schemas.ts` — the `Storefront` / `StorefrontEdit` component schemas (lines ~1173-1207) become `Project` / `ProjectEdit`, add `type` enum field. *(This file is generated from the OpenAPI spec — regenerate per repo convention rather than hand-editing if a generator exists; otherwise edit directly.)*
7. `dto.ts` — `StorefrontDto`/`StorefrontEditDto` → `ProjectDto`/`ProjectEditDto` (add `ProjectType` export).
8. `storefront.ts` (the rendering tree) — **leave the type names as-is** (decision #2). It stays the storefront-page content schema. `index.ts` re-export comment updated to clarify it's the storefront-type page schema.

### C. Router
9. Rename `src/routers/storefront.ts` → `src/routers/project.ts`:
   - `storefrontRouter` → `projectRouter`, paths `/storefronts*` → `/projects*`.
   - `prisma.storefront` → `prisma.project`, `prisma.storefrontEdit` → `prisma.projectEdit`.
   - `FREE_TIER_STOREFRONT_LIMIT` → `FREE_TIER_PROJECT_LIMIT` (keep value 2 for now; metering comes later).
   - On create, accept optional `type` (default `STOREFRONT`); `slugify` fallback base `'project'`.
   - Optional `?type=` filter on the list endpoint.
10. `src/validation/storefront.ts` — keep file (validates the storefront tree); referenced by the storefront project type. Imports in router updated to new path only if file is renamed; **leave filename** to stay surgical (it validates the *storefront tree*, decision #2).
11. `src/services/anthropic.ts` — `generateStorefrontTree` stays (it generates a storefront-type page). Optionally alias/rename later; **no change now**.
12. `src/routers/all_routes.ts` — uncomment import + `router.use(projectRouter)` (lines 49, 96-98), update to project naming. Keep it registered **before** `universalRouter` (which must stay last).

### D. Verify
13. `tsc` / lint clean across api + types package.
14. Prisma schema validates (`prisma validate` / `prisma generate`) — user runs the actual migrate.

---

## nanza-web-app changes

15. `src/services/api/Storefront.ts` → `Project.ts`: `storefrontService` → `projectService`, paths `/storefronts*` → `/projects*`, add optional `type` to `create`.
16. `src/types/index.ts` (lines ~149-200): `StorefrontDto`/`StorefrontEditDto`/`StorefrontPayload`/`StorefrontStatus` → Project equivalents; add `ProjectType`. Keep `StorefrontTree`/`StorefrontTheme` names (mirror decision #2).
17. `src/app/Build/data/useStorefronts.ts` → `useProjects.ts`: hooks renamed (`useStorefronts`→`useProjects`, etc.), call `projectService`, default `type: 'STOREFRONT'` on create.
18. `src/app/Build/**` (dashboard, Editor, StorefrontCard, renderer): update imports/types to Project; **renderer + tree types unchanged** (storefront page format). `StorefrontCard` → `ProjectCard` (or leave name, low value).
19. `src/app/AppLayout.tsx`: uncomment **only** the `Layout` import + `BuildWrapper`/`BuildEditorWrapper` imports and their two routes (`/build`, `/build/:projectId`). Leave all user-profile/auth/cart routes commented.
20. Confirm `.env` `REACT_APP_API_BASE_URL` → points at the nanza-api env we're testing against (currently `https://api-dev.nanza.app`). No code change unless we want a local target.

---

## Open items to confirm before coding
- **A.** Project types confirmed by user: `STOREFRONT, GIFT, GROUP, LISTING, COLLECTION, BID, LOT` (dropped market/deck).
- **B.** Re-enable on web = storefront **builder** (`/build`) only, not public profile routes. Confirmed by user.
- **C.** Frontend project: use existing `nanza-web-app` for now (no new `nanza-ai` project yet). Confirmed by user.
- **D.** `@oakplatforms/types` publish: the **mobile app AND the web app** consume the *published* npm package (web app is on `@oakplatforms/types@0.1.54`); only **nanza-api** consumes its in-repo `packages/types`. So the api-side Project/membership types regenerate locally, but the web app's `AccountDto` won't carry `membershipType` until the package is republished. The web app re-declares `ProjectDto`/`ProjectStatus`/`ProjectType` locally in `src/types/index.ts` (matching the prior Storefront pattern) and reads `membershipType` defensively (see "Shipped" below) so nothing is blocked.

---

## SHIPPED (2026-06-24, membership reverted 2026-06-25)

All of the below is implemented, tsc-clean (exit 0) and lint-clean on touched files, on branch `dev`. **Not committed.** Migration NOT yet run (user runs it).

> **Membership removed 2026-06-25.** The MembershipType enum / `Account.membershipType` / paid builder gate were fully reverted — the long-term model is **caps/credits** (Discord-style: cap messaging, scanning, AI features; pay to raise caps / add commission-free transactions), NOT subscription tiers. The builder is now open to **any signed-in user**; cap-based gating comes later.

### nanza-api
- `prisma/schema.prisma`: `Storefront`→`Project` (+`type ProjectType @default(STOREFRONT)`, `@@index([type])`), `StorefrontEdit`→`ProjectEdit`, `StorefrontStatus`→`ProjectStatus`. Back-relations `Account.projects`/`Brand.projects`. New `enum ProjectType { STOREFRONT, GIFT, GROUP, LISTING, COLLECTION, BID, LOT }`. ~~MembershipType enum + Account.membershipType~~ (removed).
- `src/routers/storefront.ts` → `src/routers/project.ts`: `/storefronts*`→`/projects*`, `prisma.project`/`projectEdit`, optional `?type=` list filter, optional `type` on create. ~~Server-side membership 402 gate~~ (removed).
- `all_routes.ts`: `projectRouter` uncommented/registered (before `universalRouter`).
- Types regenerated via `npm run generate:types`: `ProjectDto`/`ProjectEditDto` (no membership).
- Tree/theme rendering schema kept named `Storefront*` on purpose (it's the STOREFRONT-type page content).

### nanza-web-app
- `services/api/Storefront.ts`→`Project.ts` (`projectService`, `/projects*`); `Build/data/useStorefronts.ts`→`useProjects.ts`; `Build/components/StorefrontCard.tsx`→`ProjectCard.tsx`; `Build/index.tsx` + `Build/Editor/index.tsx` retargeted to Project (route param `:storefrontId`→`:projectId`). `types/index.ts`: local `ProjectDto/ProjectStatus/ProjectType/ProjectEditDto/ProjectPayload` (renderer tree types stay `Storefront*`).
- **Cognito auth re-enabled**: `services/auth/cognito.ts` ported from nanza-mobile (browser localStorage, no AsyncStorage); `.env` sets `REACT_APP_COGNITO_USER_POOL_ID=us-east-1_S924xgl85` / `..._CLIENT_ID=2tms3boiftodahp86cnnpjk5j7` (same dev pool as mobile); `AppLayout` re-enables `/sign-in`,`/sign-up`,`/forgot-password` (cart stays disabled); `Toolbar/AuthControl.tsx` restored (avatar + sign-out + Builder link). SessionProvider/api token logic were already correct.
- **Builder gate**: `Build/BuilderGuard.tsx` wraps both `/build` routes — signed-out → sign-in prompt; signed-in → builder. (`utils/membership.ts` deleted.)

## TODO (next session)
- Run the Prisma migration (drops Storefront/StorefrontEdit, adds Project/ProjectEdit + ProjectType enum).
- Republish `@oakplatforms/types` with `ProjectDto`; then bump web app and replace the local Project DTOs with the published types.
- Manual test: sign in → `/build` → create STOREFRONT project → AI generate → publish.
- (Later) Design the caps/credits model and layer cap-based gating onto the builder + scanning + messaging + AI features.
