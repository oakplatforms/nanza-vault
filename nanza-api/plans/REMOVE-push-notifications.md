# How to Remove Push Notifications from nanza-api — Full Report

Status as of 2026-06-29. Scope: **backend (nanza-api) only**. Mobile is handled separately
(uncommitted there — stash it). Push was never working end-to-end (events weren't invoking the
sender Lambda); you've decided to pull it out and revoke the Apple APNs key.

---

## 1. Security assessment (read first)

**Good news: there is NO credential leak in git history.** The audit:

- `config/firebase-wif.json` **is committed**, BUT it is the **non-secret** Workload Identity
  Federation descriptor — it contains only pool/provider/service-account *names* and public Google
  URLs. **No private key, no client secret.** Verified: grepped the file + its full history for
  `private_key` / `BEGIN ... PRIVATE` / `client_secret` → none.
- No `.p8`, no service-account JSON key, no FCM server key was ever committed in the push work.
- (Separate pre-existing issue, NOT push-related: `nanza-mobile/ios/fastlane/AuthKey_J3N69NK8BQ.p8`
  is committed in the mobile repo. That's an App Store Connect key, unrelated to this work — flagged
  for you separately.)

**Conclusion:** removing push is **not** security-urgent from a leaked-secret standpoint. History
rewriting (git filter-repo / BFG) is **NOT required**. A normal revert commit is sufficient and safe.

What you SHOULD still do for hygiene:
- **Revoke the APNs `.p8` key** in Apple Developer (you said you would) — good, do it. It's the one
  real credential, and it lives in the Firebase console, not the repo.
- Optionally delete/disable the Firebase project `nanza-a1ff7` (or just the Cloud Messaging config)
  if you don't plan to reuse it. Not required; harmless if left.
- The GCP Workload Identity binding (the `gcloud add-iam-policy-binding` you may have run) can be
  left or removed — it grants an AWS role permission to impersonate the Firebase SA. Once the AWS
  role / Lambda is deleted (step below) it's inert. Remove it for tidiness if you like.

---

## 2. What exists (the complete footprint)

Push was added across **3 commits**: `fc6ebb2 add push notification support`,
`662789d fix issue with dev firebase`, `e4fd6d1 fix push notifications`
(plus `69500cf` = the test probe + serverless tweak).

⚠️ Those commits ALSO contain unrelated work (`metaHandler.ts`, `referenceResolver.ts`,
`og/meta.ts`, `validation/referenceCode.ts`). **Do NOT revert those files** — only the push files.

### New files (delete these)
- `src/services/fcm.ts`
- `src/services/notificationEvent.ts`
- `src/routers/deviceToken.ts`
- `lambdas/pushNotificationHandler.ts`
- `scripts/test-eventbridge-publish.ts`
- `config/firebase-wif.json`  (non-secret, but no longer needed)
- `prisma/migrations/20260629122643_add_device_tokens/migration.sql`
- `documentation/plans/firebase-wif-setup.md`, `apns-key-setup.md`, `firebase-push-setup.md`,
  and this file (docs — delete or keep for reference; harmless)

### Edited files (revert ONLY the push hunks)
- `src/routers/all_routes.ts` — remove `import { deviceTokenRouter }` + `router.use(deviceTokenRouter)`
- `src/routers/message.ts` — remove the `publishNotificationEvent` import + the `await Promise.all(...)` block
- `src/routers/connection.ts` — remove the import + the two `await publishNotificationEvent(...)` calls
- `src/services/systemMessage.ts` — remove the `publishNotification` import + the `await publishNotification(...)` call
- `serverless.yml` — remove: the `pushNotification:` function block; the `FIREBASE_PROJECT_ID` +
  `GOOGLE_APPLICATION_CREDENTIALS` provider env vars; the `firebaseProjectId` custom map
- `prisma/schema.prisma` — remove `model DeviceToken`, `enum DevicePlatform`, and
  `deviceTokens DeviceToken[]` from `model Account`
- `package.json` / `package-lock.json` — push added one dep (firebase-admin). Remove if unused elsewhere.
- Generated: `packages/types/src/dto.ts`, `oak-api-schemas.ts`, `src/generated/json/json-schema.json`
  — DO NOT hand-edit; these regenerate via `npm run generate:types` after the schema change.

### Deployed AWS resources (must be torn down, not just code-deleted)
- Lambda `nanza-api-push-notification-dev` + its IAM role
- The EventBridge rule on the `default` bus targeting that Lambda
- These are removed automatically when you redeploy after deleting the `pushNotification:` function
  from serverless.yml (CloudFormation deletes the orphaned resources). **You must redeploy** — just
  deleting code locally leaves the live Lambda + rule running.

### Database
- The `DeviceToken` table exists in the dev DB (migration `20260629122643_add_device_tokens` ran).
- Dropping it needs a new migration (you handle migrations). It's harmless to leave — an empty,
  unused table. Recommend: leave it for now, drop later if desired. If you remove the model from
  schema without a drop migration, `prisma migrate` will want to create one.

---

## 3. Removal procedure (recommended order)

1. **Revoke the APNs key** in Apple Developer → Keys (the one real credential).
2. **Delete the new files** listed above.
3. **Revert the push hunks** in the 6 edited code files (NOT the unrelated files).
4. **schema.prisma**: remove DeviceToken model + DevicePlatform enum + Account.deviceTokens.
5. **serverless.yml**: remove the pushNotification function + the 2 env vars + firebaseProjectId.
6. `npm run generate:types` (regenerates dto/schemas cleanly without DeviceToken).
7. `npm run lint` + `npx tsc --noEmit` → expect 0 errors.
8. **Redeploy** `npm run deploy:dev` → CloudFormation tears down the push Lambda + EventBridge rule.
9. Commit as a single "remove push notifications" revert.
10. (Optional) New migration to drop the DeviceToken table; (optional) delete Firebase project / WIF binding.

### Fastest path
Because the push code is well-isolated (mostly new files + additive hunks), an agent can do steps
2–7 surgically in a few minutes, then you run 8–10. The risk is low: the edits were additive, so
reverting them restores the prior behavior exactly. The ONLY thing that needs care is not touching
the unrelated changes (`metaHandler`, `referenceResolver`, `og/meta`, `referenceCode`) that share
the same commits.

---

## 4. TL;DR
- **No secret leaked** → no history rewrite needed; a plain revert commit is safe.
- **Revoke the APNs key** (only real credential, lives in Apple/Firebase, not the repo).
- Delete ~8 new files, revert push hunks in 6 files, drop schema model, remove serverless block,
  regen types, **redeploy** (to delete the live Lambda + EventBridge rule), commit.
- Leave the empty `DeviceToken` table (drop later) and the non-secret WIF config in history (harmless).
