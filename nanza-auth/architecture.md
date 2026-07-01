# nanza-auth — Architecture

A small, stateless AWS Lambda service that sits between **Amazon Cognito**, **nanza-api**, **AWS Secrets Manager**, and **EventBridge**. It owns no data store. Its job is to react to Cognito user-pool lifecycle events and to authentication flows, and to keep the platform's user records in sync.

See [[INDEX]] for the file map and [[../_shared/INDEX|Shared brain]] for cross-project decisions.

- **Runtime / framework:** Node.js 20, TypeScript, Serverless Framework v3 (`serverless.yml`).
- **Region:** `us-east-1` (hardcoded in every SDK client and in `serverless.yml`).
- **Package name:** `nanza-auth` (`package.json`). README calls it `oak-auth`; the Serverless `service:` is `nanza-auth`, though the README still describes the service as `tcgx-auth` — naming is inconsistent across the repo.

---

## 1. The two handler families

Every file in `lambdas/` exports a single `handler` and returns the event unchanged (Cognito triggers must echo the event back). They split into two behaviours:

**A. Provisioning handlers** — authenticate as a service user, then call nanza-api to create a user record:
- `lambdas/consumerPostConfirmationHandler.ts`
- `lambdas/adminPreTokenGenerationHandler.ts`

**B. Event-emitting handlers** — publish an EventBridge event and exit:
- `lambdas/forgotPasswordHandler.ts`
- `lambdas/changePasswordHandler.ts`

---

## 2. Cognito trigger lifecycle

### Consumer sign-up → Post Confirmation
`lambdas/consumerPostConfirmationHandler.ts` is typed against `PostConfirmationTriggerEvent`. It guards on `event.triggerSource === 'PostConfirmation_ConfirmSignUp'` and returns early for any other source (e.g. `PostConfirmation_ConfirmForgotPassword`). On a real sign-up it:

1. Reads the new user's `sub` from `event.request.userAttributes.sub`.
2. Fetches the consumer service password via `getConsumerPassword()` (Secrets Manager, §4).
3. Calls `AdminInitiateAuthCommand` with `AuthFlow: 'ADMIN_USER_PASSWORD_AUTH'` against the **consumer** pool (`CONSUMER_USER_POOL_ID` / `CONSUMER_CLIENT_ID`) using a fixed service `DEFAULT_USERNAME`, to obtain an `AccessToken`.
4. `POST /user` to nanza-api (§3) with `authId: <sub>`, `type: 'REGISTERED'`, an empty `profile`, one empty cart, and a primary `Collection` list.

Net effect: a confirmed Cognito consumer gets a matching nanza-api account provisioned automatically.

### Admin first token → Pre Token Generation
`lambdas/adminPreTokenGenerationHandler.ts` is typed against `PreTokenGenerationTriggerEvent`. It guards on `event.triggerSource === 'TokenGeneration_NewPasswordChallenge'` — i.e. it fires the first time an admin completes the forced new-password challenge (typical for admin-created accounts), not on every token issuance. On that source it:

1. Reads `sub` from `event.request.userAttributes.sub`.
2. Fetches the admin service password via `getAdminPassword()`.
3. `AdminInitiateAuthCommand` (`ADMIN_USER_PASSWORD_AUTH`) against the **admin** pool (`ADMIN_USER_POOL_ID` / `ADMIN_CLIENT_ID`) with `DEFAULT_USERNAME`.
4. `POST /user` to nanza-api with `authId: <sub>`, `isAdmin: true`, and `admin.email`.

> **Design note:** admin provisioning is deliberately hung on Pre-Token-Generation + `NewPasswordChallenge` rather than Post-Confirmation, because admin accounts are created by an administrator and confirmed through the new-password flow rather than self-service sign-up.

### Password flows → Forgot / Change
`lambdas/forgotPasswordHandler.ts` and `lambdas/changePasswordHandler.ts` do **not** call nanza-api. Each requires a `sub` (throws if missing) and publishes one EventBridge event via `PutEventsCommand` on bus `default`:

| Handler | Source | DetailType | Detail `type` |
| --- | --- | --- | --- |
| `forgotPasswordHandler.ts` | `tcgx` | `user.forgot-password` | `user.forgot-password` |
| `changePasswordHandler.ts` | `tcgx` | `user.password-changed` | `user.password-changed` |

The `Detail` payload carries `userId` (the `sub`) and `email`. Downstream consumers of the `tcgx` event source (outside this repo) react to these — e.g. sending a password-related email. Both files are typed as `PreTokenGenerationTriggerEvent`, which is almost certainly a copy-paste inheritance rather than an accurate reflection of the Cognito trigger they are meant to serve; the code only touches `event.request.userAttributes`, so it works regardless, but the typing is not authoritative about the real trigger.

> **Wiring gap (verified):** `serverless.yml` only declares the two *provisioning* functions. `forgotPasswordHandler` and `changePasswordHandler` have **no `functions:` entry and are not deployed by this config**. They exist as source but are either wired elsewhere, invoked out-of-band, or currently dormant. Treat their deployment as unconfirmed until the Cognito → Lambda attachment is located.

> **Trigger attachment (verified):** even for the two declared functions, `serverless.yml` contains no `events:` block linking them to a Cognito user pool. The pools are referenced only in IAM. The actual **Cognito trigger → Lambda association is configured on the user pools themselves, outside this repo** (e.g. in the pool definition or console). This service supplies the Lambda; something else points Cognito at it.

---

## 3. Calling nanza-api

All outbound calls to nanza-api go through the single generic client `src/services/api/index.ts` (`fetchData<T>()`):

- Base URL comes from `process.env.API_BASE_URL`; the helper validates it's a real URL, trims a trailing slash, and ensures the path is slash-prefixed before `fetch()`.
- Headers are always `Accept`/`Content-Type: application/json`; when a `token` is passed it adds `Authorization: Bearer <token>`.
- HTTP `429` throws a dedicated `RateLimitError`; other non-2xx responses throw with `errorMessage`/`error` from the JSON body, else `HTTP Error: <status>`.

Both provisioning handlers pass the Cognito `AccessToken` from `AdminInitiateAuth` as `token`, so nanza-api sees a bearer token minted for the service user. The only endpoint used today is `POST /user`.

---

## 4. Secrets Manager (key names only)

`src/utils/secretsManager.ts` wraps `@aws-sdk/client-secrets-manager`:

- **Secret name:** derived at runtime as `nanza-credentials-${process.env.STAGE || 'dev'}` (e.g. `nanza-credentials-dev`, `nanza-credentials-prod`). Fetched via `GetSecretValueCommand`.
- **JSON shape (`NanzaAuthSecrets`):** the secret string is parsed to an object with keys **`adminPassword`** and **`consumerPassword`**. These are *key names only* — no values are read, logged, or stored in this repo, and none should be.
- **Caching:** the parsed secret is held in a module-level cache with a **30-minute TTL** (`CACHE_TTL_MS`), so warm Lambda invocations skip the Secrets Manager round-trip. `clearSecretsCache()` exists for retry-on-auth-failure paths.
- **Accessors:** `getAdminPassword()` and `getConsumerPassword()` return the respective service-user password for the `AdminInitiateAuth` call.

These passwords are the credentials for the fixed `DEFAULT_USERNAME` service user in each pool — the mechanism by which the Lambda mints a token it can present to nanza-api.

---

## 5. Environment & configuration

Env vars are injected by `serverless-dotenv-plugin` locally (from `.env.<stage>`, not committed) and by GitHub Actions `vars` in CI (§7).

| Var | Used by | Purpose |
| --- | --- | --- |
| `STAGE` / `NODE_ENV` | provider-wide | stage name; `STAGE` also builds the secret name |
| `ADMIN_USER_POOL_ID`, `ADMIN_CLIENT_ID` | admin handler | admin pool + client for `AdminInitiateAuth` |
| `CONSUMER_USER_POOL_ID`, `CONSUMER_CLIENT_ID` | consumer handler | consumer pool + client for `AdminInitiateAuth` |
| `DEFAULT_USERNAME` | both provisioning handlers | fixed service-user username |
| `API_BASE_URL` | `fetchData` | nanza-api base URL |

`serverless.yml` sets `NODE_ENV`/`STAGE` provider-wide and passes the pool/client/username/API vars into each function's `environment` block individually.

---

## 6. IAM

The provider-level IAM role (`serverless.yml`) grants exactly two things:

- `secretsmanager:GetSecretValue` on `arn:aws:secretsmanager:us-east-1:<account>:secret:nanza-credentials-*` — read the staged credentials secret.
- `cognito-idp:AdminInitiateAuth` on the admin and consumer user pool ARNs (built from `${aws:accountId}` and the pool-id env vars) — mint service-user tokens.

No database, queue, or (notably) explicit `events:PutEvents` permission is granted here — another reason the EventBridge-emitting handlers appear to run under a different deployment/role than what `serverless.yml` defines.

---

## 7. Deploy (Serverless + GitHub Actions)

- **Framework:** Serverless v3, `package: individually`, esbuild bundling (`bundle`, `minify`, `target: node20`).
- **CI:** `.github/workflows/deploy-dev.yml` (push to `dev`) and `deploy-prod.yml` (push to `prod`). Both:
  1. Checkout, Node 20, `npm ci`.
  2. Authenticate to AWS via **OIDC** (`aws-actions/configure-aws-credentials@v4`, `role-to-assume: vars.AWS_ROLE_ARN_{DEV|PROD}`) — no static access keys.
  3. Install Serverless globally, disable dashboard/telemetry.
  4. `serverless deploy --stage {dev|prod} --verbose`, injecting the pool/client IDs, `DEFAULT_USERNAME` (`vars.USERNAME`), and `API_BASE_URL` from stage-specific GitHub Actions `vars`.
- **Discrepancy (verified):** `package.json` scripts reference `serverless.dev.yml` / `serverless.prod.yml` (`--config`), and the README mentions per-env configs, but only a single `serverless.yml` exists in the repo. The CI workflows do **not** use `--config`; they rely on `serverless.yml` + `--stage`. The package scripts are stale relative to the actual deploy path.

---

## Open questions / honest gaps

- **Where are the Cognito triggers actually attached?** Not in this repo — the pool-side association must be located to confirm which handler fires on which pool event.
- **Are `forgotPasswordHandler` / `changePasswordHandler` deployed at all?** They are absent from `serverless.yml` and their event-emit path has no matching IAM in this repo. Their deployment and `events:PutEvents` grant live elsewhere (or they are dormant).
- **`PreTokenGenerationTriggerEvent` typing on the password handlers** looks like a copied type, not a statement of the real trigger source; don't treat it as ground truth.
