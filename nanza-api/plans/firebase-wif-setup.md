# Firebase Push — Keyless Auth Setup (Workload Identity Federation)

This is the **keyless** way for the nanza-api Lambda to authenticate to Firebase Cloud
Messaging — **no service-account key file to download or store.** Instead, the Lambda's AWS IAM
role *federates* into Google: Google trusts "anything running as this AWS role" and lets it
impersonate a Firebase service account. The only thing committed to the repo is a **non-secret**
descriptor (`config/firebase-wif.json`) that just names the pool/provider/service-account — it
contains no keys.

The code already expects this. From `src/services/fcm.ts`:
- `FIREBASE_PROJECT_ID` = `nanza-a1ff7` (already set in serverless.yml)
- `GOOGLE_APPLICATION_CREDENTIALS` = `./config/firebase-wif.json` (already set; file must be created)

Known values you'll need:
| Thing | Value |
|---|---|
| Firebase / GCP project id | `nanza-a1ff7` |
| GCP project number | `56171621239` |
| AWS account id | `992382609315` |
| Lambda role name (dev) | `nanza-api-dev-us-east-1-lambdaRole` |
| Firebase SA email | `firebase-adminsdk-fbsvc@nanza-a1ff7.iam.gserviceaccount.com` |

---

## Step 1 — Enable APIs + get the project number (~3 min)
1. Go to https://console.cloud.google.com and select project **nanza-a1ff7** (top bar).
2. Note the **Project number** (Cloud Console → ☰ → **Cloud overview → Dashboard**, "Project number"). You'll need it below.
3. Enable these APIs (☰ → **APIs & Services → Enable APIs**, search + enable each):
   - **IAM Service Account Credentials API** (`iamcredentials.googleapis.com`)
   - **Security Token Service API** (`sts.googleapis.com`)
   - **Firebase Cloud Messaging API** (`fcm.googleapis.com`) — usually already on.

## Step 2 — Create a Workload Identity Pool (~3 min)
1. ☰ → **IAM & Admin → Workload Identity Federation**.
2. **Create Pool**:
   - Name / ID: `aws-lambda-pool`
   - Description: `AWS Lambda federation for nanza-api`
   - **Continue**.

## Step 3 — Add an AWS provider to the pool (~2 min)
1. In the pool's **Add a provider** step:
   - Provider type: **AWS**
   - Provider name / ID: `aws-provider`
   - **AWS Account ID:** `992382609315`
   - **Continue → Save.**
2. (Default attribute mapping is fine: `google.subject = assertion.arn`.)

## Step 4 — Use the existing Firebase Admin SDK service account (~1 min)
**No new service account, no role grant.** Firebase already created an admin SDK service account in
this project that has full FCM send permission by default. Just find its email:
1. ☰ → **IAM & Admin → Service Accounts**.
2. Copy the one named **`firebase-adminsdk-XXXXX@nanza-a1ff7.iam.gserviceaccount.com`**
   (the `XXXXX` is a random suffix).
3. Use that email wherever Steps 6 and 7 say the service account. (Skip the old "create fcm-sender +
   grant a role" approach — this existing SA already has the FCM role.)

> Reference below uses the placeholder `FIREBASE_SA_EMAIL` for this address.

## Step 5 — Deploy the API once, then find the Lambda role (~5 min)
The WIF binding needs the exact AWS role ARN of the `pushNotification` Lambda. Serverless creates it on deploy.
1. Deploy: `npm run deploy:dev` (from nanza-api).
2. Find the role: AWS Console → **Lambda → nanza-api-push-notification-dev → Configuration → Permissions → Execution role**. Copy the **Role ARN**, e.g.
   `arn:aws:iam::992382609315:role/nanza-api-dev-us-east-1-lambdaRole`
   (serverless often uses one shared `lambdaRole` for all functions — that's fine.)

## Step 6 — Let the AWS role impersonate the service account (~3 min)
Bind the federated AWS identity to the `fcm-sender` SA. Replace `PROJECT_NUMBER` and the role ARN.

Easiest via Cloud Shell (☰ → Activate Cloud Shell). Replace `FIREBASE_SA_EMAIL` with the
`firebase-adminsdk-XXXXX@nanza-a1ff7.iam.gserviceaccount.com` from Step 4, then run:
```bash
gcloud iam service-accounts add-iam-policy-binding \
  FIREBASE_SA_EMAIL \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/aws-lambda-pool/attribute.aws_role/arn:aws:sts::992382609315:assumed-role/nanza-api-dev-us-east-1-lambdaRole"
```
- `PROJECT_NUMBER` → from Step 1.
- Role name is `nanza-api-dev-us-east-1-lambdaRole` (the dev Lambda execution role; ARN `arn:aws:iam::992382609315:role/nanza-api-dev-us-east-1-lambdaRole`). For prod, swap `dev`→`prod`.
- Note it's `assumed-role` and `sts::`, not `iam::` — that's the federated form.

## Step 7 — Download the credential config descriptor (~2 min)
This is the **non-secret** JSON the code points at. In the Workload Identity Federation pool page:
1. Open pool `aws-lambda-pool` → **Connected service accounts** (or the **Download config** action on the AWS provider).
2. **Download configuration**, choosing:
   - Service account: the `firebase-adminsdk-XXXXX@nanza-a1ff7.iam.gserviceaccount.com` from Step 4
   - Provider: `aws-provider`
3. Save the downloaded file to the repo as **`config/firebase-wif.json`** (create the `config/` dir).
   Google names the download `clientLibraryConfig-aws-provider.json` — rename it.
   ⚠️ The console download often OMITS `service_account_impersonation_url`. Without it the config
   auths as the raw AWS identity (no FCM permission). The committed file MUST include:
   `"service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/firebase-adminsdk-fbsvc@nanza-a1ff7.iam.gserviceaccount.com:generateAccessToken"`
   It looks like:
```json
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/aws-lambda-pool/providers/aws-provider",
  "subject_token_type": "urn:ietf:params:aws:token-type:aws4_request",
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/firebase-adminsdk-XXXXX@nanza-a1ff7.iam.gserviceaccount.com:generateAccessToken",
  "token_url": "https://sts.googleapis.com/v1/token",
  "credential_source": {
    "environment_id": "aws1",
    "region_url": "http://169.254.169.254/latest/meta-data/placement/availability-zone",
    "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials",
    "regional_cred_verification_url": "https://sts.{region}.amazonaws.com?Action=GetCallerIdentity&Version=2011-06-15"
  }
}
```
It contains **no private key** — safe to commit. (It's already wired: `GOOGLE_APPLICATION_CREDENTIALS=./config/firebase-wif.json`, and serverless packages it into the push function.)

## Step 8 — Redeploy + test (~5 min)
1. `npm run deploy:dev`.
2. Trigger a push (send a message / make an offer to a test account that has a registered device token).
3. Watch the consumer logs: AWS → **CloudWatch → Log groups → /aws/lambda/nanza-api-push-notification-dev**.
   - Success: no errors, device receives the push.
   - `FIREBASE_PROJECT_ID is not configured` → env not set (it is, in serverless.yml — re-deploy).
   - `unable to impersonate` / 403 → the Step 6 binding member string is wrong (check PROJECT_NUMBER + the exact assumed-role ARN / role name).
   - `GOOGLE_APPLICATION_CREDENTIALS ... not configured` or file-not-found → `config/firebase-wif.json` wasn't packaged (confirm it exists and is committed).

---

## Why it "wasn't working" before
The most common failures are all in Steps 6–7:
- **Wrong member ARN** — it must be the `assumed-role` STS form (`arn:aws:sts::992382609315:assumed-role/<ROLE_NAME>`), not the IAM role ARN. Serverless's role name must match exactly.
- **APIs not enabled** (Step 1) — STS / IAM Credentials off → token exchange 403s.
- **`config/firebase-wif.json` missing from the package** — now handled: serverless.yml packages it into the push function, but the file must actually exist in the repo.
- **Wrong SA** — make sure you used the existing `firebase-adminsdk-XXXXX` SA (it already has the FCM role); a hand-made SA without the role lets federation succeed but the send is denied.

## Going to production (same Firebase project, repeat ONE step)
Dev and prod share the same Firebase project (`nanza-a1ff7`), the same `config/firebase-wif.json`,
and the same APNs key. The **only** prod-specific step is re-running the Step 6 binding for the prod
Lambda's role (prod runs as a different IAM role):
```bash
gcloud iam service-accounts add-iam-policy-binding \
  firebase-adminsdk-fbsvc@nanza-a1ff7.iam.gserviceaccount.com \
  --project nanza-a1ff7 \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/56171621239/locations/global/workloadIdentityPools/aws-lambda-pool/attribute.aws_role/arn:aws:sts::992382609315:assumed-role/nanza-api-prod-us-east-1-lambdaRole"
```
(Only difference vs dev: `dev` → `prod` in the role name. Confirm the prod role name after `deploy:prod`.)

## Fallback if WIF keeps fighting you
If this stays broken, the service-account-key path is a 5-minute alternative: generate a key
(Firebase → Project settings → Service accounts → Generate new private key), store the JSON value
(env var / your secret store — I only reference the key name in code), and I swap `fcm.ts` to
`cert(JSON.parse(env))` instead of `applicationDefault()`. Say the word and I'll make that change.
