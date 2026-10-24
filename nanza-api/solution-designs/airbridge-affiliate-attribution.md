---
tags: [nanza-api, nanza-mobile, plan, affiliates, attribution, airbridge]
---

# Airbridge — Affiliate Install Attribution

**Status:** **plan — API + mobile built 2026-10-16, both uncommitted; pending migration, types
0.1.112 publish, Airbridge dashboard setup + SDK tokens, QA.** Spans nanza-api (one column, one
claim endpoint, one link endpoint) and nanza-mobile (Airbridge SDK, deferred deep link, claim call). Builds on [[profile-types]] (`AFFILIATE`) and the mobile
[[../../nanza-mobile/solution-designs/sharing|Sharing]] design (share links on `nanza.app`).
Scope is **the Airbridge attribution loop only** — the Mixpanel wiring, the web-SDK landing-page
work and ad campaigns from the partner notes are listed under "Out of scope" and not designed
here. When it lands, fold into [[profile-types]] (an "Affiliate attribution" section) and retire
this file.

## The ask (Skylar, 2026-10-15, voice)

We already have an Airbridge account and deep linking is set up. What we need is the affiliate
loop: every affiliate gets a link tied to their account. When someone installs through that link
and signs up, the new account is associated with the affiliate's account ID in our database.
Figure out what identifier travels with the link, how it survives install + Cognito sign-up, and
where it lands.

## Design in one paragraph

Each affiliate has one Airbridge tracking link whose sub-publisher and deep-link `ref` parameter
both carry the affiliate's **`Account.id`**. The Airbridge React Native SDK resolves the deferred
deep link on first open after install and hands the app the deep-link URL; the app stores the
`ref` locally as a *pending affiliate*. Once the user has an account (email or Apple/Google),
the app calls one new endpoint, `PUT /user/affiliate`, which writes `Account.affiliateId` once
and never again. First touch wins; the server enforces it with a null check and a freshness
window.

## Why not Cognito

The tempting path is Cognito `clientMetadata` on `confirmSignUp` → Post Confirmation trigger →
`POST /user` with the affiliate id in the body. It fails for two reasons:

- **Social sign-in never calls `confirmSignUp`.** Apple/Google users federate through the hosted
  domain; there is no client call that carries metadata into the trigger.
- **`POST /user` is a service-to-service call** from nanza-auth. Threading one more field through
  Cognito, the trigger and the auth Lambda spreads a product concern across three repos for one
  write.

A post-sign-up claim from the app covers both auth paths with one endpoint and no auth-repo change.

## Claim endpoint hardening (2026-10-16)

`PUT /user/affiliate` answers every refusal of a well-formed claim with one generic **409** and one
message ("Affiliate attribution is not available for this account"), whether the target id is
unknown, not an active AFFILIATE account, the caller is already attributed elsewhere, or the
caller's account is past the 7-day window. Before this an unknown target was a 404, which let any
signed-in user probe cuids and learn which ids are affiliates. The real reason is logged
server-side (`CLAIM_AFFILIATE_REFUSED: …`). 400 remains for malformed input (bad cuid shape,
self-referral, missing accountId); 200 for success and for an idempotent repeat of the same
affiliate. The app treats 400 and 409 alike: drop the parked ref. Threat model otherwise:
caller can only attribute their own account (authId check), affiliateId is cuid-shape-checked and
parameterized, write-once via `updateMany where affiliateId: null`, `sub_id` is reporting only —
`Account.affiliateId` is the payout source of truth, never the Airbridge report.

## The link

Airbridge supports **custom channels** and builds tracking links from URL parameters, so the
affiliate link is deterministic — nothing is stored per affiliate. Airbridge only resolves
parameter-built links on the path `/@<app>/<channel>`; a bare `/<username>` path returns 404
(probed live 2026-10-16 with curl against go.nanza.app). So the path is fixed and the affiliate's
username travels in the deep link:

```
https://go.nanza.app/@nanza/affiliate
  ?sub_id=<affiliateAccountId>
  &deeplink_url=https://nanza.app/<username>?ref=<affiliateAccountId>
```

**Deliberately minimal** (2026-10-16, probed live): no `campaign` — the channel segment already
classifies the link in reports — and no `fallback_ios` / `fallback_android` — Airbridge falls back
to the store URLs on the dashboard App Info pages (verified: iOS → App Store listing, Android →
Play listing). ~134 characters instead of ~330. The deep link is the remaining bulk because it
carries its own `?ref=` and must be URL-encoded. The only way to go materially shorter is one
registered Airbridge short link per affiliate (`go.nanza.app/<code>`) via the Tracking Link API —
deferred; the deterministic link needs no storage and no API round-trip.

- **Path = `/@nanza/affiliate`, fixed.** `@nanza` is the Airbridge app name
  (`AIRBRIDGE_APP_NAME`), `affiliate` is the Airbridge **channel** — the thing reports filter by.
  Decided 2026-10-16, replacing the earlier flat-username path once the probe showed Airbridge
  rejects it. `/@nanza/<username>` would also resolve but makes every affiliate its own channel
  and was rejected for that. A true `go.nanza.app/<username>` would need one registered short link
  per affiliate via the Tracking Link API — not worth it today. Later use cases (QR, NFC,
  outreach) get their own channel segment, e.g. `/@nanza/qr`, with their own campaign value.
- **The channel `affiliate` classifies the link** (no `campaign` parameter). **`sub_id` (sub
  publisher 1) = affiliate `Account.id`** — matches the partner naming (sub publisher 1 =
  affiliate id).
- **`deeplink_url`** points at the affiliate's existing public profile page
  ([[public-profile-page]]) so a new install lands on the affiliate, not a blank Home. The `ref`
  query parameter is what the app reads; it is the same account id, duplicated so attribution does
  not depend on Airbridge reporting fields reaching the SDK. Deep link, fallbacks and the `ref`
  handling are unchanged by the path decision.
- **`go.nanza.app`** is the Airbridge link domain (`AIRBRIDGE_LINK_DOMAIN`, per the partner
  notes). Associated Domains / App Links for it are part of the SDK install, below.
- **Minted by the API, not the dashboard:** `GET /profile/:id/affiliate-link` (own profile or
  admin; 403 unless `profile.type === 'AFFILIATE'`). Mobile shows it in **Account → Settings** as
  a read-only field with a Copy action (between Email and Update Password — Skylar, 2026-10-16;
  the own-profile share circle shares the plain profile URL like everyone else's). The public
  web page also gets it, server-built, as `affiliateLink` on `GET /user/:username/page` for its
  Join button. nanza-admin can show it on the profile picker. Keeping the format in one server
  function means the app name, channel, campaign name and parameter spelling live in one place.
  The username in the deep link comes from the profile; if a username changes, the minted link
  changes with it, and the `sub_id` / `ref` (account id) keep attribution stable.
- **Resolved 2026-10-16:** parameter-built links on the custom domain work with no
  dashboard-registered link per affiliate, as long as the path is `/@nanza/<channel>`. The
  redirect carries `deeplink_url`, `campaign`, `sub_id` and both fallbacks through unchanged.

Why `Account.id` and not `Profile.referenceCode`: the ask is an account association, `Account.id`
is immutable and already the key `AnalyticsBridge`-style identity calls use, and it saves one
lookup on claim. The profile reference code is a reasonable alternative (opaque, short, already in
share URLs) — decide in review.

## Data model

```prisma
model Account {
  // …existing…
  affiliate   Account?  @relation("AccountAffiliate", fields: [affiliateId], references: [id], onDelete: SetNull)
  affiliateId String?
  referrals   Account[] @relation("AccountAffiliate")

  @@index([affiliateId])
}
```

- Additive, nullable, no backfill. One migration, run manually.
- `onDelete: SetNull` — deleting an affiliate account must not cascade into its referrals.
- No `attributedAt`: `Account.createdAt` is within the freshness window by construction, so the
  claim time is recoverable. Add it only if reporting needs it.
- Types: `packages/types` regenerates `AccountDto.affiliateId`; bump the package version.

## API

### `PUT /user/affiliate` — claim

Body `{ accountId: string, affiliateId: string }`. Caller: the signed-in user, for their own
account — `accountId` follows the codebase convention (body id validated against the Cognito
principal by `validateAccount`).

Validation (`validation/affiliate.ts`, pure):
- `affiliateId` is a cuid-shaped string.
- Not the caller's own account id (self-referral → 400).
- Target account exists and `profile.type === 'AFFILIATE'` (read-only existence check; 404
  otherwise).
- Caller's `account.affiliateId` is null (409 if already set — first touch wins).
- Caller's `account.createdAt` is within **7 days** (409 otherwise). This is what stops an
  existing user who taps an affiliate link later from being re-attributed; the window is
  generous because the claim fires on the first signed-in session, which can be days after the
  install.

Service (`services/affiliate.ts`): a single `update` guarded by `where: { id, affiliateId: null }`
so two racing claims cannot both win. Returns the account. Idempotent for the same affiliate id
(a repeat with the same id after success returns 200, not 409).

### `GET /profile/:id/affiliate-link` — mint

Returns `{ url }` built from the format above. Own profile or admin.

### Reads

`affiliateId` rides along on the existing `GET /user` payload via the DTO. nanza-admin later
gets a "Referred by" column; no new list endpoint now. A per-affiliate count
(`referrals: { _count }`) is one include away when the affiliate dashboard needs it.

## Mobile

- **SDK:** add `airbridge-react-native-sdk`; init at root beside `initAnalytics()`. Associated
  Domains (iOS, via xcconfig) and App Links (Android, Gradle manifest placeholder) for
  `go.nanza.app`, per the partner notes. The existing `nanza.app` share-link `prefixes` in
  `AppNavigator` stay as they are.
- **Deep link in:** `Airbridge.setOnDeeplinkReceived(url)` fires for both direct opens and
  deferred deep links after install. The handler parses `ref`; if present, persist
  `pendingAffiliateId` (MMKV/AsyncStorage) and hand the URL to the normal `Linking` path so the
  profile page still opens. One handler, no change to `getStateFromPath`.
- **Claim:** in `SessionContext`, when `currentUser` first resolves and `pendingAffiliateId`
  is set and `currentUser.account.affiliateId` is null → `PUT /user/affiliate`, then clear the
  pending value on 2xx **or** 4xx (a 409/404 means it will never succeed; keep it only on
  network failure). This is the same spot `AnalyticsBridge` identifies the user, so it works for
  email and social sign-in alike.
- **Airbridge identity:** `Airbridge.setUserID(account.id)` on sign-in, `clearUser()` on
  sign-out, so Airbridge's own sign-up attribution reports line up with our account ids. Send
  `signUp` / `signIn` as Airbridge events at the same points `SIGNUP_COMPLETED` fires today.

## Abuse and edge cases

- **Self-referral** rejected server-side; the link itself cannot be trusted.
- **Clipboard / link spoofing:** anyone can craft `?ref=<id>`. The only thing it buys is an
  attribution to a real affiliate within 7 days of creating an account, which is the intended
  outcome anyway. Payouts, if any later, reconcile against Airbridge's own install attribution,
  not just our column.
- **Reinstall by an existing user:** no pending ref (Airbridge does not deliver a deferred deep
  link for a non-fresh install), and the window check blocks it regardless.
- **Airbridge down / SDK fails:** the app never blocks on it; no pending ref means no claim.
  Attribution is lost for that install, same as today.

## Web "Join" button and the Settings copy (2026-10-16)

The deep link inside every affiliate link is `nanza.app/<username>?ref=…`, so a visitor **without
the app on a phone, or on a laptop, lands on the creator's web page**. That page now carries one
call to action, **Join the chat**, far right in the nav (white pill, black type — the site's filled-pill
style at the nav's 44px; the marketing menu, hamburger and store pills are hidden on this route):

| Visitor | Join goes to |
|---|---|
| AFFILIATE page, any device | `page.affiliateLink` — the creator's tracking link, so the click and install are attributed (desktop too — Skylar, 2026-10-17; Airbridge serves its own desktop landing) |
| Other profile, phone | the platform's store (same UA sort as `/get`) |
| Other profile, desktop | `/get` — our store-buttons page |

The desktop **QR card** on the profile page ("Scan to get the app", bottom right) scans to the same
affiliate link when `page.affiliateLink` is set — drawn live with `qrcode.react` (added 2026-10-17)
instead of the homepage's static OneLink art — so a phone scanning off a laptop goes through the
attributed link too. Other profiles keep the general code. The button label is **"Join the chat"**
(2026-10-17).

`GET /user/:username/page` returns `affiliateLink` (string | null), built by the same
`buildAffiliateLink` the mint endpoint uses, so the format still lives in one place. Web files:
`src/app/ProfilePage/JoinButton.tsx`; `Navigation` gained an `action` slot on its `minimal` mode (the partner's logo-only bar for the profile page; Join is the one control it may carry),
`ProfileLayout` passes it.

**Mobile placement:** the affiliate's link is in **Account → Settings** between Email and Update
Password — `ShelfHeading "Affiliate link"` with a **Copy** action (`@react-native-clipboard/clipboard`,
added 2026-10-16, pod installed) over a disabled `TextField` showing the URL, exactly the Email
treatment. Shown only when `isAffiliate`. The profile share circle went back to the plain profile
URL for everyone.

## Next: vouchers

What the affiliate earns once the referred account is active is
[[affiliate-vouchers|Affiliate Vouchers]] (plan, 2026-10-17): hooks on the referred user's fifth
post / fifth sent offer to five distinct recipients / fifth listing issue one `Voucher` per tier, reviewed in nanza-admin, paid by hand.

## Out of scope (from the partner notes, not designed here)

- Mixpanel integration, `acq_*` property replacement, distinct-id ↔ Airbridge device alias.
- Airbridge Web SDK on the `nanza.app` share landing pages — affiliate links are Airbridge links
  directly, so the web SDK is not needed for this loop. (The page's Join button re-enters the
  tracking link instead — see above.)
- NFC / QR / outreach / ad-network channels. They reuse the same link format with a different
  `campaign` and, when needed, a second path tier under `go.nanza.app/<username>`; nothing here
  prevents them.
- Optimizing paid campaigns on outcome events.

## Build order

1. ✅ API (2026-10-16): `Account.affiliateId` + migration `20261016120000_account_affiliate_id`,
   `validation/affiliate.ts` (cuid shape, self-referral, AFFILIATE target → 404, first touch /
   7-day window → 409), `services/affiliate.ts` (`updateMany` guarded on `affiliateId: null`),
   `PUT /user/affiliate` (body `{ accountId, affiliateId }`, registered before `PUT /user/:id`),
   `GET /profile/:id/affiliate-link` (own or admin, 403 unless AFFILIATE, needs a username),
   `utils/affiliateLink.ts` (the one place the link format lives; `AIRBRIDGE_LINK_DOMAIN` from
   `serverless.yml custom.airbridgeLinkDomain`, store URLs defaulted in code),
   `generatePrismaError` now honours a `statusCode` pinned on a plain `Error`, types 0.1.112.
   Migration not yet run; types not yet published.
2. ✅ Mobile (2026-10-16): `airbridge-react-native-sdk` 4.11 + `airbridge.json`
   (`trackingLinkCustomDomains: [go.nanza.app]`, ATT wait 0 — the app never asks ATT); native
   init reads the app name + SDK token from `Info.plist` (`AirbridgeAppName` /
   `AirbridgeAppToken`) and `res/values/airbridge.xml`, and **skips init when the token is
   empty** so credential-less builds still run; `AppDelegate.mm` tracks `openURL` +
   `continueUserActivity`, `MainActivity.onResume` tracks the intent; `applinks:go.nanza.app`
   entitlement + an `autoVerify` App Links filter for `go.nanza.app`. JS:
   `services/attribution/` (the only Airbridge import, fail-safe; `pendingAffiliate.ts` parks the
   `ref` in AsyncStorage, first touch wins), `AttributionBridge` beside `AnalyticsBridge` (deep
   links from Airbridge **and** RN Linking → park ref, resolve `nanza.app/<username>` →
   `UserProfile`; claim via `PUT /user/affiliate` once `account.id` resolves, clear on any server
   answer, keep on network failure; `setUserID` / `clearUser`), `trackEvent('airbridge.user.signup')`
   beside `SIGNUP_COMPLETED` (email only — social sign-up has no completion hook today),
   `userService.claimAffiliate`, `profileService.affiliateLink` + `getByUsername`, and the
   ~~own-profile share circle on `UserProfileScreen` shares the affiliate link~~ → moved 2026-10-16 to
   Account → Settings (read-only + Copy); the share circle shares the plain profile URL.
   Not changed: `SessionContext`, `AppNavigator` (bridge imports `navigationRef` only).
3. Admin (later): "Referred by" on the profile picker, referral count per affiliate.

## Open questions

1. `Account.id` vs `Profile.referenceCode` as the public identifier in the link.
2. Confirm parameter-built links on a custom channel (with a `<username>` path) vs one dashboard
   link per affiliate.
3. Is 7 days the right claim window, or should it match the Airbridge attribution window?
4. Should the affiliate see their referral count in-app now, or is admin-only enough for v1?
