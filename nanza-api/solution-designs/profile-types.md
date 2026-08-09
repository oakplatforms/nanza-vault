---
tags: [nanza-api, solution-design, profile]
---

# Profile Types — Basic vs Affiliate

> **Status:** Implemented in code 2026-07-30 (`type` column + admin-only `PATCH
> /profile/:id/type`; shipped alongside the v1 tag taxonomy work). No admin UI for
> assigning AFFILIATE yet.

## Overview

Every account today has at most one undifferentiated `Profile` (username, avatar, banner,
description, QR/reference codes). This design introduces a **profile type** so the platform can
distinguish ordinary users from **affiliates** — profiles that will progressively unlock extra
features (content, promotion, deeper storefront tooling) in later phases.

The v1 change is deliberately minimal: an enum column with a safe default. It exists now so the
follow-on affiliate features have something to gate on, and so admin/mobile can start surfacing the
distinction.

## Data model

```prisma
enum ProfileType {
  BASIC
  AFFILIATE
}

model Profile {
  // …existing fields…
  type ProfileType @default(BASIC)

  @@index([type])
}
```

- `@default(BASIC)` — the migration needs no backfill; every existing profile is `BASIC`.
- `@@index([type])` for future "list affiliates" queries.
- Named `ProfileType` (not touching the existing `AccountType` / `SellerType` enums — those
  describe different axes: auth-level role and seller business shape).

## API surface

- `GET /profile*` responses include `type` automatically once the DTO regenerates
  (`packages/types` → `@oakplatforms/types` republish; admin and mobile bump the package).
- **Who can set it:** admin-only for now (`PATCH` guarded by `validateRole 'admin'`). Users do not
  self-select affiliate in v1 — promotion is an operator action until an application flow exists.
- Profile update route rejects `type` unless the caller is an admin; otherwise it's a silent
  privilege escalation vector.

## First consumer

[[posts|Posts]] gates **brand post** creation on `profile.type === 'AFFILIATE'` — the brand feed is
read by everyone and written by affiliates only. That is the first real thing this enum controls.

## Later (affiliate phases, not designed yet)

- Affiliate-only features hang off `profile.type === 'AFFILIATE'` checks (content/posts,
  promotional placement).
- A self-serve application/approval flow would replace direct admin assignment.
- If affiliate needs its own settings blob, prefer a sibling `AffiliateProfile` 1:1 model over
  widening `Profile`.
