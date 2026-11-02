---
tags: [shared, onboarding, push, firebase, apns, notifications]
---

# Onboarding — Push notifications (Firebase Cloud Messaging)

**Status:** written 2026-10-30. 2026-10-31: Steps 1–4 done (Step 3: one Sandbox & Production key, uploaded to both Firebase slots; Step 4 done by Claude in the entitlements + Info.plist, no Xcode clicks) — the project is **`nanza-8889`** (a new one, not the old `nanza-a1ff7`); both config files are in the repo. The Step 5 service-account key must come from `nanza-8889` too. This page is the **human checklist** for the
console work that the code cannot do: the Firebase project, the Apple push key, the two config
files, the server credential, and the first test message. Design and code map:
[[../../nanza-api/solution-designs/push-notifications|Push notifications]]. Do the steps in
order — each one unblocks the next. Tick the "Done when" line before moving on.

## Where this sits in the build

Steps 1–6 are yours and can happen **now, in parallel** with the code — nothing in them waits
on a build. Step 7 (the first test message) needs the mobile app rebuilt with the Firebase SDK,
so it waits on Claude's mobile half. Step 8 is after the API half is deployed. Prod (Step 9) is
last, and only after the dev test script in the design passes on both phones.

## What is already in place

- A Firebase project, `nanza-a1ff7`, from the 2026-06 prototype — *probably* still exists; its
  APNs key was **revoked**, so a new one is needed (Step 3). Nothing else from that prototype
  survives in code.
- App identifiers: iOS bundle `com.oakplatforms.nanza`, Android package
  `com.oakplatforms.nanza`, Apple Team ID in the Apple developer account (Membership page).
- Secrets Manager secret `nanza-credentials-dev` / `-prod`, read by the API through
  `secretsManager.ts` with a 30-minute cache — the new key goes in there (Step 5).
- Mobile already has `applinks:nanza.app` entitlements and the `nanza://` scheme; push adds two
  more capabilities (Step 4) and does not touch these.

Everything on this page is done in a browser or Xcode by you. Claude never reads or writes
Secrets Manager (CLAUDE.md).

## Step 1 — Firebase project

1. Open [console.firebase.google.com](https://console.firebase.google.com) with the Oak Google
   account. If **nanza-8889** is listed, open it. If not, **Add project** → name `nanza`, Google
   Analytics **off** (Mixpanel is the analytics; Firebase here is only for messaging).
2. **Project settings (gear) → Cloud Messaging**: under *Firebase Cloud Messaging API (V1)* it
   must read **Enabled**. If it reads Disabled, click the ⋮ → *Manage API in Google Cloud
   Console* → **Enable**. (The legacy "Cloud Messaging API" row can stay disabled — the server
   uses V1.)

**Done when:** the project opens and *Cloud Messaging API (V1)* shows **Enabled**.

## Step 2 — Register the two apps and download their config files

1. Project overview → **Add app → iOS**: bundle ID `com.oakplatforms.nanza`, nickname
   `nanza iOS`, App Store ID can stay blank → Register → **Download GoogleService-Info.plist**.
   Skip the remaining wizard screens (the SDK steps are Claude's).
2. **Add app → Android**: package name `com.oakplatforms.nanza`, nickname `nanza Android`,
   SHA-1 can stay blank (only needed for Google sign-in / Dynamic Links, not messaging) →
   Register → **Download google-services.json**. Skip the rest.
3. Put the files in the repo:
   - `nanza-mobile/ios/nanza/GoogleService-Info.plist`
   - `nanza-mobile/android/app/google-services.json`
   Both are **client config, not secrets** (same class as the Airbridge app token in
   `Info.plist`): the API key inside is restricted to this app and is shipped in the binary
   either way. Commit them.
4. ~~Add the plist to the Xcode target by hand.~~ **Done by Claude 2026-10-31** — the project
   file now references `ios/nanza/GoogleService-Info.plist` and copies it into the app, the same
   way it handles `Info.plist`. (Why it matters: Xcode only bundles files the project lists; a
   file that is just sitting in the folder never reaches the phone, and Firebase would stay off.)
   In Xcode it shows in the left sidebar under the `nanza` folder, next to `Info.plist`.

**Done when:** both files are in place, the plist shows under the `nanza` group in Xcode, and
Firebase's project overview lists two apps.

## Step 3 — Apple push key (APNs) → Firebase

Without this, Android works and **iOS silently delivers nothing**. One key covers dev and prod.

1. [developer.apple.com](https://developer.apple.com/account) → **Certificates, Identifiers &
   Profiles → Keys → +**. Name `Nanza push`, tick **Apple Push Notifications service (APNs)** →
   Continue → Register → **Download** the `.p8`. **It downloads exactly once** — keep it
   somewhere safe outside the repo (1Password). Note the **Key ID** shown on that page.
2. Note the **Team ID**: Account → Membership details.
3. Also under **Identifiers → App IDs → `com.oakplatforms.nanza`**: make sure the **Push
   Notifications** capability is ticked (Xcode's automatic signing usually does this in Step 4;
   if it didn't, tick it here and save).
4. Firebase → **Project settings → Cloud Messaging → Apple app configuration → APNs
   Authentication Key → Upload**: the `.p8`, the Key ID, the Team ID.

**Done when:** Firebase's Apple app configuration shows the key with its Key ID, and the old
revoked key (if listed) is removed.

## Step 4 — Xcode capabilities (iOS)

In `ios/nanza.xcworkspace`, target `nanza` → **Signing & Capabilities**:

1. **+ Capability → Push Notifications**. This writes `aps-environment` into
   `nanza.entitlements` (`development` in debug, Xcode switches it to `production` for App Store
   builds).
2. **+ Capability → Background Modes** → tick **Remote notifications**. This writes
   `UIBackgroundModes: remote-notification` into `Info.plist`.
3. Commit the entitlements + plist diff.

**Done when:** both capabilities are listed on the target and `git diff` shows the two files.
(Android needs no console capability — the `POST_NOTIFICATIONS` permission is in the manifest
diff Claude makes.)

## Step 5 — Server credential → Secrets Manager

The API's `push` Lambda signs its calls to Firebase with a service-account key.

1. Firebase → **Project settings → Service accounts → Firebase Admin SDK → Generate new
   private key** → a JSON file downloads. Treat it like the database password.
2. AWS console → **Secrets Manager → `nanza-credentials-dev` → Retrieve secret value → Edit**:
   add a key **`firebaseServiceAccount`** whose value is the **entire JSON file as one string**
   (paste the file contents; the console stores it as a string value — do not split its fields
   into separate keys). Save.
3. Delete the downloaded JSON from your Downloads folder.
4. Prod gets the same key under `nanza-credentials-prod` in Step 9 (the same service account
   works for both stages; one key, pasted twice).

**Done when:** `nanza-credentials-dev` has a `firebaseServiceAccount` key whose value starts
with `{"type": "service_account"`.

## Step 6 — Secret shape in code (Claude)

`NanzaSecrets` in `src/utils/secretsManager.ts` gains `firebaseServiceAccount: string`; the
`push` Lambda does `initializeApp({ credential: cert(JSON.parse(secrets.firebaseServiceAccount)) })`
once per container. No env vars, no `custom.firebaseProjectId` — the JSON names the project.

**Done when:** `tsc` is clean and the Lambda logs `[push] firebase ready <project_id>` on cold
start in dev.

## Step 7 — First test message (transport only)

Needs the mobile app rebuilt natively with the Firebase SDK (`npx pod-install`, then a device
build on both phones — a JS reload does nothing for native modules).

1. Run the debug build on the **iPhone** (physical — the Simulator has no APNs) and the **Moto**.
   Accept the permission prompt when the app asks (the first-message / first-friend-request
   moment; in a debug build Claude also logs the token at launch).
2. Copy the FCM token from the Metro console line `[push] token <…>`.
3. Firebase → **Engage → Messaging → Create your first campaign → Firebase Notification
   messages** → title/body anything → **Send test message** → paste the token → Test.
4. Lock the phone first; the banner should arrive within seconds. Repeat for the other phone.

**Done when:** a banner from the Firebase console lands on both locked phones. If iOS gets
nothing and Android does, Step 3 is wrong (key, Key ID, Team ID, or the Xcode capability).

## Step 8 — Dev end to end

API deployed to dev (`nanza-api-dev` pipeline release) with the `Device` migration run. Then
run the **Testing in dev** script in the design — stage 1 (one real message), stage 2 (the nine),
stage 3 (the rules), both phones.

**Done when:** every row passes on both phones and `oak-api-push-dev` logs show no
`post failed` for valid tokens.

## Step 9 — Prod

1. Paste the same `firebaseServiceAccount` value into `nanza-credentials-prod`.
2. Release `nanza-api-prod`.
3. Ship the mobile build (TestFlight / internal track) and repeat Step 7's console test against
   a prod-signed build — this is the only way to confirm the `production` APNs environment.

**Done when:** a console test message lands on a TestFlight iPhone build and a Play internal
Android build.

## Related

- [[../../nanza-api/solution-designs/push-notifications|Push notifications — design]]
- [[../../nanza-api/solution-designs/infra-push|Infra & push — the 2026-06 prototype and why the APNs key was revoked]]
- [[airbridge|Airbridge onboarding]] — the same checklist shape
