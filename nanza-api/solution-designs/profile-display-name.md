---
tags: [nanza-api, nanza-mobile, solution-design, plan, profile]
---

# Profile Display Name + username hardening on the user route (Plan)

> **Status: plan, 2026-10-23 — building same day in nanza-api + nanza-mobile.** One additive
> column, one migration (`20261023130000_profile_display_name`), types bump `0.1.124`. Web app
> (`nanza.app/<username>`) keeps showing the username for now (Skylar: "eventually the web app,
> not for now"). Fold into `profile-types.md` / mobile `profile.md` when it lands.

## The ask (Skylar, voice, 2026-10-23)

- The username stays the identity and the URL (`nanza.app/<username>`, the affiliate link). It
  must be URL-safe: no spaces, no odd characters, underscores allowed, **no periods** (TikTok
  and Instagram allow them; decided against). Onboarding still only asks for a username.
- A new **optional `Profile.displayName`**, free-form, edited in profile settings *under* the
  username field. Usually unset.
- Mobile shows the display name **wherever it shows a username today**, falling back to the
  username when unset. @-handles (the composer's "Replying to @user", the affiliate link) stay
  username — they are addresses, not names.

## What exists

- `USERNAME_RE = /^[A-Za-z0-9_]{3,20}$/` + `RESERVED_USERNAMES` in
  `src/constants/reservedUsernames.ts`; `validateUsername` (shape → reserved → one
  case-insensitive uniqueness query) runs on `POST /profile` and `PUT /profile/:id`.
- **The gap:** `PUT /user/:id` — the route the mobile `CompleteProfileModal` and the settings
  form actually call — only checks uniqueness (friendly 409) and then writes
  `sanitizeString(username)`. A username with a space or a reserved word gets through there.
- Mobile's `CompleteProfileModal` enforces the same regex client-side; the settings form
  (`AccountScreen/Profile`) enforces nothing.

## Design — nanza-api

1. **Schema:** `displayName String?` on `Profile`, right under `username`. Migration
   `20261023130000_profile_display_name`: `ALTER TABLE "Profile" ADD COLUMN IF NOT EXISTS
   "displayName" TEXT;`. Skylar runs it.
2. **Validation (`validation/profile.ts`):** `validateProfileDisplayName(value)` — undefined /
   null / empty → `null` (clears), must be a string, trimmed, ≤ 30 graphemes (TikTok's cap;
   mirrors `validatePostDisplayName`). Sanitized with `sanitizeString` like the other text fields.
3. **Routes:** `POST /profile` and `PUT /profile/:id` accept `displayName`. `PUT /user/:id`
   accepts `account.profile.displayName`, and — the hardening — runs `isValidUsernameShape` +
   `isReservedUsername` on `profile.username` **before** the uniqueness query, answering 400 with
   the same copy the profile router uses. Uniqueness stays the friendly 409.
4. **Reads:** every `profile: { select: { … username: true … } }` across routers/services also
   selects `displayName`, and the hand-built rows that copy `username` out (the `/profiles`
   rail, `/profiles/affiliates/latest-posts`, `/profiles/affiliates/recent-posts`,
   `GET /user/:username/page`) copy `displayName` beside it. Full-profile includes pick it up
   from the model.
5. **Types:** `npx prisma generate` → `generate:types` + `build:types` → `0.1.124`.

## Design — nanza-mobile

1. `src/utils/profileName.ts` — `getProfileName(profile, fallback)` = `displayName` (trimmed,
   non-empty) else `username` else fallback. One helper so the rule has one home.
2. Apply it to the name slots: `PostItem` author line, `UserActionRow`, `Badge/Profile`,
   `ProfileBadge` (header), `ProfileHero` title (own + other profile), `AffiliateCircleCarousel`
   label + `AffiliateStoriesScreen` author, the Inbox tabs (All / Active / Friends / System
   / requests sheet), conversation + offer thread titles, `offerCopy`, `GroupMembersScreen`,
   `TradeUserListRow` / `useTradeUserList`, `ActionModal/Feed` copy, `PostInput` "Replying to",
   `ListingScreen` / `BidScreen` seller lines. **Not** changed: `CompleteProfileModal` (gate
   on username), `PostComposer`'s `@username` placeholder, the affiliate link.
3. `AccountScreen/Profile`: a **Display name** `TextField` under Username (optional, `maxLength`
   30, help text "Shown instead of your username. Optional."), in the change detection and the
   `PUT /user/:id` payload; the rail row types (`AffiliateRailItem`, `AffiliateRecentPostsRow`)
   gain `displayName`.
4. `npm run types:update` once `0.1.124` is published.

## Rollout

API → migration (dev) → types publish → mobile `types:update` → device QA: set a display name
in settings, see it on a post, the rail, a conversation, the profile hero; clear it and see the
username return; try `bad name` / `admin` on the completion sheet and the settings form → 400.
