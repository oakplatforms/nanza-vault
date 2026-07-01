---
tags: [nanza-api, solution-design, projects]
---

# Projects / AI Builder — Solution Design

## Overview

nanza has an AI-driven page builder: a user describes what they want and Claude generates a
renderable page tree. The original single-purpose `Storefront` model was generalized into a broader
`Project` model carrying a **type**, so the same builder can eventually produce storefronts, gifts,
group pages, listing/collection/bid/lot pages, etc. — with `STOREFRONT` as the first (and currently
only real) type. This design captures that refactor's end state across nanza-api and nanza-web-app.

## How it works

### Data model

- **`Project`** (replaced `Storefront`) — holds the rendering `tree` (JSON page), optional
  `publishedTree`, `theme`, free-text `context`, a unique `name` slug, `displayName`, and
  `account` / `brand` relations. New: `type ProjectType @default(STOREFRONT)`, `status
  ProjectStatus @default(DRAFT)`, and `@@index([type])`.
- **`ProjectEdit`** (replaced `StorefrontEdit`) — logs each AI edit (prompt, before/after tree,
  token counts); relation fields renamed `storefront`→`project`, `storefrontId`→`projectId`.
- **`enum ProjectType { STOREFRONT, GIFT, GROUP, LISTING, COLLECTION, BID, LOT }`** (dropped the
  earlier `market`/`deck` ideas), default `STOREFRONT`.
- **`enum ProjectStatus { DRAFT, PUBLISHED, ARCHIVED }`** (renamed from `StorefrontStatus`, same
  values).
- Back-relations `Account.projects` and `Brand.projects` (were `.storefronts`).
- Reference-code type letter `O` = Project.
- Migration posture was a **clean drop-and-recreate** — there was no storefront data to preserve.
  (The user runs the actual `prisma migrate`.)

The rendering-tree schema is **deliberately kept named `StorefrontTree` / `StorefrontTheme`** (the
~338-line `packages/types/src/storefront.ts`). It describes a *storefront page layout*, which is
exactly the content of the `STOREFRONT` project type; other project types will define their own
content schemas later, so a generic "ProjectTree" name would be misleading. `tree`/`theme` stay
required-with-default so the storefront path works unchanged and non-storefront types just get an
empty default tree for now.

### Router & services (nanza-api)

- `src/routers/storefront.ts` → **`src/routers/project.ts`** — `storefrontRouter` →
  `projectRouter`, paths `/storefronts*` → `/projects*`, `prisma.storefront`/`storefrontEdit` →
  `prisma.project`/`projectEdit`. `create` accepts an optional `type` (default `STOREFRONT`, slug
  base `'project'`); the list endpoint takes an optional `?type=` filter. The former
  `FREE_TIER_STOREFRONT_LIMIT` was renamed `FREE_TIER_PROJECT_LIMIT`.
- **`all_routes.ts`** — the `projectRouter` was previously commented out; it is now uncommented and
  registered, **before** `universalRouter` (which stays last so it can catch bare reference codes).
  This is what brought the AI-orchestration API live.

  > NOTE: the current `architecture.md` still describes the `projectRouter` as intentionally not
  > mounted. This refactor's end state re-enables it; that section of the architecture doc predates
  > this change and is the one point of drift to reconcile.

- `src/services/anthropic.ts` — `generateStorefrontTree` (the Claude client that builds a
  storefront-type page) stays as-is; it generates STOREFRONT-type content.
- `src/validation/storefront.ts` — kept under its filename (it validates the *storefront tree*,
  consistent with keeping the tree schema named `Storefront*`).
- Types regenerated via `npm run generate:types` → `ProjectDto` / `ProjectEditDto` (and a
  `ProjectType` export) in `packages/types` (`dto.ts`, `oak-api-schemas.ts`).

### Web builder (nanza-web-app)

- `services/api/Storefront.ts` → `Project.ts` (`projectService`, `/projects*`, optional `type` on
  create); `Build/data/useStorefronts.ts` → `useProjects.ts`; `Build/components/StorefrontCard.tsx`
  → `ProjectCard.tsx`; `Build/index.tsx` + `Build/Editor/index.tsx` retargeted to Project (route
  param `:storefrontId` → `:projectId`).
- Because the web app consumes the **published** `@oakplatforms/types` (not nanza-api's in-repo
  package), it re-declares `ProjectDto` / `ProjectStatus` / `ProjectType` / `ProjectEditDto` /
  `ProjectPayload` **locally** in `src/types/index.ts` (mirroring the prior Storefront pattern)
  until the package is republished. Renderer tree types stay `Storefront*`.
- **Cognito auth re-enabled** on web: `services/auth/cognito.ts` ported from nanza-mobile (browser
  `localStorage` instead of AsyncStorage), `.env` pointed at the same dev pool as mobile,
  `AppLayout` re-enables `/sign-in`, `/sign-up`, `/forgot-password`, and `Toolbar/AuthControl.tsx`
  restored (avatar + sign-out + Builder link). Only the `/build` and `/build/:projectId` builder
  routes are re-enabled — the public user-profile routes stay commented (the web app is a
  distribution surface, not a public profile storefront).
- **Builder access:** `Build/BuilderGuard.tsx` wraps both `/build` routes — signed-out users get a
  sign-in prompt, any signed-in user gets the builder.

### Monetization — deliberately NOT built

Membership/subscription tiers were prototyped and then **fully reverted** (2026-06-25). The
`MembershipType` enum, `Account.membershipType`, and the paid-builder 402 gate are gone. The
long-term model is **caps/credits** (Discord-style: cap messaging, scanning, and AI features; pay to
raise caps or add commission-free transactions), **not** subscription tiers. The builder is open to
any signed-in user today; cap-based gating on the builder / scanning / messaging / AI comes later.

## Key decisions & rationale

- **Generalize `Storefront` → `Project` + `type`, don't run parallel models.** Keeping both would
  leave two overlapping concepts and violate DRY; the storefront becomes one project type.
- **Keep the tree schema named `Storefront*`.** It genuinely is the storefront-page layout; renaming
  it to a generic `ProjectTree` before a second type exists would mislead, since a gift/deck project
  won't reuse this node set. Surgical to leave it.
- **`ProjectType` default `STOREFRONT`, tree required-with-default.** Keeps the storefront path
  working unchanged and avoids a premature nullable-tree decision for type-less projects.
- **Re-enable the builder route only, not public profile routes.** The web app is a distribution
  surface for the builder, not a public storefront host, so profile routes stay disabled.
- **Monetization stays out.** Membership was tried and reverted; caps/credits is the chosen model
  and is intentionally unbuilt so the refactor doesn't couple to a business model still being
  designed.

### Status / remaining

Shipped on `dev` (tsc- and lint-clean, uncommitted as of the plan). Remaining: run the Prisma
migration (drops Storefront/StorefrontEdit, adds Project/ProjectEdit + `ProjectType`); republish
`@oakplatforms/types` with `ProjectDto` and swap the web app's local DTOs for the published ones;
manual end-to-end test (sign in → `/build` → create STOREFRONT project → AI generate → publish);
then, later, design and layer on the caps/credits gating.

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
