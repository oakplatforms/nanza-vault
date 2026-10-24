---
tags: [shared, onboarding, airbridge, affiliates, attribution]
---

# Onboarding — Airbridge affiliate attribution

**Status:** code built 2026-10-16 in `nanza-api` + `nanza-mobile` (uncommitted). Steps 1 and 2
done 2026-10-16 (both `.well-known` files verified live on go.nanza.app); Step 3 done 2026-10-16 (link path
changed to `/@nanza/affiliate`); next is Step 4. This page is the
**human checklist** to take it from "built" to "live": dashboard setup, credentials, migration,
publish, QA. Design and code map: [[../../nanza-api/solution-designs/airbridge-affiliate-attribution|Airbridge affiliate attribution]].
Do the steps in order — each one unblocks the next. Tick the "Done when" line before moving on.

## What is already in place

- **API:** `Account.affiliateId`, `PUT /user/affiliate` (claim, first touch wins, 7-day window),
  `GET /profile/:id/affiliate-link` (mints `https://go.nanza.app/@nanza/affiliate?sub_id=<accountId>&deeplink_url=https://nanza.app/<username>?ref=<accountId>` — no campaign, no fallbacks; Airbridge uses the App Info store URLs as fallback).
- **Mobile:** Airbridge React Native SDK, native init on both platforms, `go.nanza.app` app links,
  `AttributionBridge` (parks the link's `ref`, claims after sign-up, mirrors identity into
  Airbridge), affiliate own-profile share uses the tracking link.
- **Guard rail:** the SDK is **off until a token is filled in** (empty token = no init), so every
  build keeps working while you do the steps below.

## Step 1 — SDK credentials (Airbridge dashboard → Settings → Tokens) — ✅ done 2026-10-16

1. Copy the **App Name** and the **App SDK Token** (not the API token, not the Web SDK token).
2. Paste the token into both native files — the name is already `nanza`; change it only if the
   dashboard app name differs:
   - iOS: `nanza-mobile/ios/nanza/Info.plist` → key `AirbridgeAppToken` (and `AirbridgeAppName`).
   - Android: `nanza-mobile/android/app/src/main/res/values/airbridge.xml` →
     `airbridge_app_token` (and `airbridge_app_name`).
3. The SDK token is a client-side token like the Mixpanel token — it ships inside the binary
   either way, so committing it is fine.

**Done when:** both files hold the token and `AirbridgeAppName` matches the dashboard app name.
*(Both files filled 2026-10-16. Values are read at native launch — a JS reload does nothing;
rebuild natively, see Step 6.)*

## Step 2 — Custom tracking-link domain `go.nanza.app` and app links — ✅ done 2026-10-16

The dashboard already lists three "apps": **Android**, **iOS** and **Website**. The Website app is
`nanza.app` (the real site, for the Web SDK) — that is correct. `go.nanza.app` is **not** an app;
it is only the tracking-link domain below. If the iOS app page nags about **SKAN / App Store
Connect API authentication**, ignore it: SKAN is for paid-ad install attribution only and
affiliate links do not use it.

### 2a — Domain — ✅ done 2026-10-16

1. Dashboard → **Tracking Link → Custom Domain**: add `go.nanza.app`.
2. Route 53, hosted zone `nanza.app`: record `go`, type **CNAME**, Alias **off** (Alias only
   targets AWS resources; Airbridge is not one), value = the CNAME target the dashboard shows,
   TTL 300. Airbridge issues the TLS cert itself — nothing in ACM.

### 2b — Deep link page (this is where Team ID / package / fingerprint go) — ✅ done 2026-10-16

Dashboard → **Tracking Link → Deep Links** (not App Info — that page only has the store URL).

| Field | Value |
|---|---|
| iOS URI Scheme | `nanza://` |
| iOS App ID | `Y9X5CD9HAV.com.oakplatforms.nanza` (Team ID `.` bundle id) |
| Android URI Scheme | `nanza://` |
| Android Package Name | `com.oakplatforms.nanza` |
| Android SHA256 Cert Fingerprints | Play App Signing key fingerprint (2c) — optionally also the debug fingerprint |

Sources: Team ID and bundle id are in `ios/nanza.xcodeproj/project.pbxproj`
(`DEVELOPMENT_TEAM`, `PRODUCT_BUNDLE_IDENTIFIER`); the `nanza` scheme is `CFBundleURLSchemes` in
`Info.plist` and the `android:scheme="nanza"` filters in `AndroidManifest.xml`. The entitlement
already lists `applinks:go.nanza.app`; the manifest already has the `go.nanza.app` intent filter.

### 2c — Android fingerprint (copy, do not create) — ✅ obtained 2026-10-16

Play App Signing key SHA-256 (from the signed universal APK of bundle 42 / 0.3.8, signer DN
`O=Google Inc.`):
`41:53:31:7F:CF:85:99:92:A3:A4:DB:5E:E4:05:63:C3:DE:46:4E:7F:12:98:AE:F6:1B:28:BC:D5:C8:3B:66:67`

Google re-signs every Play release with the **App Signing key** it holds, so the fingerprint comes
from the Play Console, not from a local keystore. In the 2026 console layout the App integrity page
no longer shows certificates; the reliable route is **App bundle explorer → active bundle →
Downloads → Signed, universal APK**, then
`$ANDROID_HOME/build-tools/<ver>/apksigner verify --print-certs <apk>` and take the
`V3.0 Signer: certificate SHA-256 digest` (ignore the Source Stamp signer). The console path, when
present, is:

1. Play Console → the Nanza app → **Test and release → Setup → App signing** (older layout:
   *App integrity*).
2. Under **App signing key certificate** copy the **SHA-256 certificate fingerprint**
   (hex pairs with colons).
3. Paste into the SHA256 field in 2b.
4. **Do not** use the *Upload key certificate* block below it — that is the
   `NANZA_UPLOAD_STORE_FILE` key, stripped before install, and it breaks App Links.

Optional, for sideloaded debug builds during QA — add as a second fingerprint (standard React
Native debug key from `android/app/debug.keystore`, public):
`FA:C6:17:45:DC:09:03:78:6F:B9:ED:E6:2A:96:2B:39:9F:73:48:F0:BB:6F:89:9B:83:32:66:75:91:03:3B:9C`
(`keytool -list -v -keystore android/app/debug.keystore -alias androiddebugkey -storepass android`).

### 2d — airbridge.json

`nanza-mobile/airbridge.json` declares `trackingLinkCustomDomains: ["go.nanza.app"]` — already
matches the dashboard spelling. ✅

**Done when:** `https://go.nanza.app/.well-known/apple-app-site-association` returns JSON
containing `Y9X5CD9HAV.com.oakplatforms.nanza`, and
`https://go.nanza.app/.well-known/assetlinks.json` contains `com.oakplatforms.nanza` plus the
Play signing fingerprint.

## Step 3 — Confirm the link format — ✅ done 2026-10-16

Probed live with curl. Airbridge resolves parameter-built links on the custom domain **only on the
`/@nanza/<channel>` path**; the flat `go.nanza.app/<username>` path returned 404. The API link
builder (`src/utils/affiliateLink.ts`) now mints `https://go.nanza.app/@nanza/affiliate?sub_id=…&deeplink_url=…`
(campaign and fallbacks dropped the same day — Airbridge falls back to the App Info store URLs,
verified with curl for both platforms) and the design's "The link" section was updated. Observed redirect for the new shape: 302 →
`nanza.airbridge.io/nanza.app/<username>?ref=<accountId>&airbridge_referrer=…channel=affiliate…
campaign=affiliate…sub_id=<accountId>…&fallback_ios=…&fallback_android=…` — every parameter passes
through. `sub_id` lands as Sub Publisher 1; `affiliate` is the channel to filter by.

Dashboard: `go.nanza.app` is set as the **primary** custom domain. No per-affiliate link is
registered anywhere.

## Step 4 — Database migration

Run `nanza-api/prisma/migrations/20261016120000_account_affiliate_id/migration.sql` in DBeaver
against **dev** first (adds the nullable `affiliateId` column, index and self-referencing FK;
no backfill). Prod follows with the prod deploy.

**Done when:** `select "affiliateId" from "Account" limit 1` runs on dev.

## Step 5 — Publish `@oakplatforms/types` 0.1.112 and bump mobile

`0.1.111` is already on CodeArtifact (the trending work), so the API bump is **0.1.112**.

1. Push `nanza-api` `dev` and release the `nanza-api-dev` pipeline — it publishes the types
   package when the version changed, then deploys the API.
2. In `nanza-mobile`: `npm install @oakplatforms/types@0.1.112` (package.json still says
   `^0.1.111`; for today I copied the local build into `node_modules` so typecheck passes — the
   install replaces that).

**Done when:** `node_modules/@oakplatforms/types/package.json` says `0.1.112` in mobile and
`AccountDto` has `affiliateId`.

## Step 6 — Build the app

- **iOS:** `pod install` already ran. In Xcode confirm the target has the build phase
  **"[Airbridge] Copy airbridge.json"** (the pod adds it) and that **Signing & Capabilities →
  Associated Domains** lists `applinks:nanza.app` and `applinks:go.nanza.app`.
- **Android:** nothing to add — the SDK's gradle script copies `airbridge.json`. A clean build
  (`cd android && ./gradlew clean`) if the manifest change does not pick up.
- Ship a TestFlight / internal build **with the token filled in** (Step 1); otherwise the SDK is
  off and Step 7 cannot pass.

**Done when:** the app launches on both platforms with no Airbridge errors in the console.

## Step 7 — QA script (fresh install → attributed account)

Use an **AFFILIATE** profile with a username as the affiliate, and a phone that has **never**
had the app (or delete the app and clear Safari/Chrome site data).

1. **Mint:** as the affiliate (or admin) call `GET /profile/<profileId>/affiliate-link` → copy
   `url`. On the device, **Account → Settings → Affiliate link → Copy** puts the same link on the
   clipboard (2026-10-16; the profile share circle shares the plain profile URL). On the web,
   `nanza.app/<username>` shows a **Join** button that opens this link on a phone.
2. **No app:** open the link → store page (fallback). Install (TestFlight / internal track).
3. **First open:** the app should land on the **affiliate's profile**, not Home (deferred deep
   link → `AttributionBridge` → `UserProfile`). If it lands on Home, the ref is still parked —
   continue.
4. **Sign up** (email or Apple/Google). Within a few seconds of the session loading,
   `Account.affiliateId` on the new account should equal the affiliate's `Account.id`
   (DBeaver, or `GET /user/<authId>` → `account.affiliateId`).
5. **Repeat tap:** tap the link again on the same device → no change (409 is swallowed, the
   pending value clears).
6. **Self link:** the affiliate opens their own link and signs in → `affiliateId` stays null.
7. **Old account:** an account older than 7 days taps a link → stays null.
8. **Airbridge dashboard:** the install and the `airbridge.user.signup` event show under the
   `affiliate` campaign with Sub Publisher 1 = the affiliate's account id.

**Done when:** steps 3, 4 and 8 pass on iOS and Android.

## Step 8 — Commit, deploy, fold the design in

1. Commit `nanza-api` (schema, migration, routes, types 0.1.112) and `nanza-mobile` (SDK, native,
   bridge) — I never commit; both trees are left for your review.
2. Prod: run the migration on prod, push `prod`, release both pipelines, ship the app build.
3. Vault: once live, fold the design into `nanza-api/solution-designs/profile-types.md` ("Affiliate
   attribution" section) and retire the plan file; keep this onboarding page as the ops record.

## Not in this round

- Mixpanel ↔ Airbridge wiring, the Airbridge **Web SDK** on `nanza.app` pages, NFC / QR / ad
  channels — all deliberately out of scope (see the design's "Out of scope").
- Social sign-up has no completion hook today, so the Airbridge `signup` event fires for **email**
  sign-ups only; the attribution claim itself works for every sign-in method.
- The web profile page (`nanza.app/<username>`) is untouched; it receives `?ref=` but does nothing
  with it. A "Get the app" button that re-enters the tracking link is a web-app follow-up.
