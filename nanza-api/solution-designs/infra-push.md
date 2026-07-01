---
tags: [nanza-api, solution-design, infra-push]
---

# Infra & Push — Solution Design

## Overview

Push notifications (FCM) were prototyped in nanza-api and then **removed**. This design records the
end state — there is **no push in the backend today** — along with the setup history (APNs auth key,
Firebase Workload Identity Federation) that is preserved so the feature can be rebuilt cleanly if
revisited. It also captures why removal was safe and what teardown it required.

## How it works (current state: push is REMOVED)

**There is no push infrastructure in nanza-api.** No `services/fcm.ts`, no push Lambda, no device-
token router, no `DeviceToken` model in the active schema. Push was never working end-to-end
(EventBridge events weren't reliably invoking the sender Lambda), so it was pulled out and the Apple
APNs key revoked.

### What the push prototype consisted of (all removed)

Push was added across three commits (`fc6ebb2`, `662789d`, `e4fd6d1`, plus a `69500cf` test probe).
Those commits **also** contained unrelated work (`metaHandler.ts`, `referenceResolver.ts`,
`og/meta.ts`, `validation/referenceCode.ts`) that was deliberately **kept** — only the push code was
reverted.

- **New files deleted:** `src/services/fcm.ts`, `src/services/notificationEvent.ts`,
  `src/routers/deviceToken.ts`, `lambdas/pushNotificationHandler.ts`,
  `scripts/test-eventbridge-publish.ts`, `config/firebase-wif.json`, and the
  `add_device_tokens` migration.
- **Push hunks reverted (files kept):** `all_routes.ts` (device-token router mount),
  `message.ts` / `connection.ts` / `services/systemMessage.ts` (the `publishNotificationEvent` /
  `publishNotification` calls), `serverless.yml` (the `pushNotification` function block, the
  `FIREBASE_PROJECT_ID` + `GOOGLE_APPLICATION_CREDENTIALS` env vars, the `firebaseProjectId` custom
  map), and `prisma/schema.prisma` (the `DeviceToken` model, `DevicePlatform` enum, and
  `Account.deviceTokens`). The `firebase-admin` dependency was removed if unused elsewhere.
- **Generated files** (`packages/types/src/dto.ts`, `oak-api-schemas.ts`, the json-schema) were
  **regenerated** via `npm run generate:types`, not hand-edited.

### Teardown that code-deletion alone did NOT cover

- **Live AWS resources:** the `nanza-api-push-notification-dev` Lambda + its IAM role and the
  EventBridge rule targeting it are **only** torn down by a **redeploy** after removing the
  `pushNotification:` function from `serverless.yml` (CloudFormation deletes the orphaned
  resources). Deleting code locally leaves the live Lambda + rule running.
- **APNs key:** revoke the `.p8` in Apple Developer → Keys — the one real credential, and it lives
  in the Firebase console, not the repo.
- **Database:** the `DeviceToken` table (from the ran `add_device_tokens` migration) is harmless to
  leave as an empty, unused table; dropping it needs a separate migration (owned by the user).
- **Optional GCP cleanup:** the Firebase project `nanza-a1ff7` (or just its Cloud Messaging config)
  and the Workload Identity binding can be left inert once the AWS role/Lambda are gone, or removed
  for tidiness.

### Security assessment (why removal was low-risk)

**No credential was leaked in git history.** The committed `config/firebase-wif.json` is the
**non-secret** Workload Identity Federation descriptor — pool/provider/service-account *names* and
public Google URLs only, no private key or client secret (verified by grepping the file and its full
history). No `.p8`, service-account JSON key, or FCM server key was ever committed in the push work.
Therefore **no history rewrite (filter-repo / BFG) was needed** — a plain revert commit is safe. The
only real credential (the APNs `.p8`) lives in Apple/Firebase, not the repo, and was revoked.

> Separately flagged (pre-existing, unrelated to push): `nanza-mobile` has an App Store Connect
> `.p8` committed under `ios/fastlane/`. Not part of this work.

## Setup history (kept for a future rebuild)

If push is ever rebuilt, two external setups are the historically fiddly parts. They are recorded so
they don't have to be re-derived.

### APNs auth key (iOS delivery)

The APNs Authentication Key (`.p8`) is what lets Firebase deliver to iPhones — without it, Android
works but **iOS silently delivers nothing**. It is generated once in Apple Developer → Keys (checking
Apple Push Notifications service), downloadable **only once**, and uploaded once to Firebase
(Project settings → Cloud Messaging → Apple app configuration → APNs Authentication Key) along with
its **Key ID** and the Apple **Team ID**. Key facts:

- Bundle ID `com.oakplatforms.nanza`, Firebase project `nanza-a1ff7`.
- **One key covers both dev and prod** — no per-stage key.
- iOS push only works on a **physical device** (the Simulator has no APNs).

### Firebase Workload Identity Federation (keyless auth)

The chosen server-auth approach was **keyless**: the Lambda's AWS IAM role federates into Google, so
Google trusts "anything running as this AWS role" and lets it impersonate the existing Firebase
Admin SDK service account (`firebase-adminsdk-…@nanza-a1ff7.iam.gserviceaccount.com`, which already
has FCM send permission). The only thing committed is the non-secret `config/firebase-wif.json`
descriptor — no key file to store. Known values: GCP project `nanza-a1ff7` (number `56171621239`),
AWS account `992382609315`, dev Lambda role `nanza-api-dev-us-east-1-lambdaRole`.

The setup: enable IAM Service Account Credentials + STS + FCM APIs; create a Workload Identity Pool
(`aws-lambda-pool`) with an AWS provider (`aws-provider`) for the AWS account; bind the **federated
assumed-role** identity to the Firebase SA with `roles/iam.workloadIdentityUser`; and commit the
downloaded descriptor as `config/firebase-wif.json`.

The failure modes that made it "not work" — worth remembering:

- **Wrong member ARN** — the binding must use the STS `assumed-role` form
  (`arn:aws:sts::…:assumed-role/<ROLE_NAME>`), not the plain IAM role ARN.
- **APIs not enabled** — STS / IAM Credentials off → token exchange 403s.
- **Descriptor missing `service_account_impersonation_url`** — the console download often omits it;
  without it the config auths as the raw AWS identity (no FCM permission).
- **Wrong SA** — a hand-made SA without the FCM role lets federation succeed but the send is denied.
- **Prod** needs the binding re-run for the prod Lambda's role (`…-prod-…-lambdaRole`); everything
  else (Firebase project, descriptor, APNs key) is shared with dev.

A service-account-**key** path (`cert(JSON.parse(env))` instead of `applicationDefault()`) was noted
as a faster fallback if WIF kept fighting.

## Key decisions & rationale

- **Push removed, not fixed.** It never worked end-to-end and was well-isolated (mostly new files +
  additive hunks), so a surgical revert restored prior behavior exactly — the only care needed was
  not touching the unrelated `metaHandler` / `referenceResolver` / `og/meta` / `referenceCode`
  changes sharing the same commits.
- **Redeploy is mandatory, not optional.** The live push Lambda and EventBridge rule only disappear
  when CloudFormation runs after `serverless.yml` loses the function block. Local deletion is not
  enough.
- **No history rewrite.** Nothing secret was ever committed; the WIF descriptor is public-safe. A
  plain revert commit suffices.
- **Keyless (WIF) was preferred over a key file** — no long-lived FCM/service-account secret to
  store, at the cost of a fiddly one-time federation binding. The setup notes are retained precisely
  because those bindings are the historically error-prone part.

## Related

- [[../INDEX|nanza-api]]
- [[../architecture|Architecture]]
