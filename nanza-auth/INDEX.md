# nanza-auth

Authentication/authorization Lambda service for the Nanza / TCGX platform. It is a thin layer of AWS Lambda handlers wired to **Amazon Cognito user-pool triggers**: when a consumer confirms sign-up, or an admin is issued a token for the first time, the matching handler authenticates as a service user and calls back to **nanza-api** to provision the corresponding user record; separate handlers emit **EventBridge** events on forgot-password / change-password so the rest of the platform can react (e.g. notifications). It holds no database of its own — it is glue between Cognito, Secrets Manager, nanza-api, and EventBridge. Package name is `nanza-auth`; the README calls it `oak-auth`; the Serverless service is `nanza-auth` (README/service naming still references `tcgx-auth`).

Deeper flow-by-flow detail lives in [[architecture|Architecture]]. Cross-project decisions live in the [[../_shared/INDEX|Shared brain]].

## Structural overview

### `lambdas/` — Cognito triggers & auth flows
Each file is a single `handler` export typed against `aws-lambda` trigger events. Two families: **provisioning** handlers (call nanza-api) and **event-emitting** handlers (publish to EventBridge).

- **`consumerPostConfirmationHandler.ts`** — Cognito **Post Confirmation** trigger on the *consumer* user pool. On `PostConfirmation_ConfirmSignUp`, authenticates as the service user (`AdminInitiateAuth`, `ADMIN_USER_PASSWORD_AUTH`) and `POST`s `/user` to nanza-api to create a `REGISTERED` account seeded with an empty cart and a primary `Collection` list.
- **`adminPreTokenGenerationHandler.ts`** — Cognito **Pre Token Generation** trigger on the *admin* user pool. On `TokenGeneration_NewPasswordChallenge`, authenticates as the service user and `POST`s `/user` to nanza-api with `isAdmin: true` to create the admin record.
- **`forgotPasswordHandler.ts`** — emits an EventBridge event `tcgx / user.forgot-password` (bus `default`) with the user `sub` and email. Typed as a `PreTokenGenerationTriggerEvent`; **not wired in `serverless.yml`** (see Architecture).
- **`changePasswordHandler.ts`** — emits an EventBridge event `tcgx / user.password-changed` (bus `default`) with the user `sub` and email. Also typed as `PreTokenGenerationTriggerEvent`; likewise **not wired in `serverless.yml`**.

### `src/services/api/` — the nanza-api client
- **`index.ts`** — a single generic `fetchData<T>()` helper. Reads `API_BASE_URL` from env, normalizes slashes, sets JSON headers, adds `Authorization: Bearer <token>`, and throws a dedicated `RateLimitError` on HTTP 429. This is the only path the provisioning handlers use to reach nanza-api.

### `src/utils/` — secrets
- **`secretsManager.ts`** — wraps `@aws-sdk/client-secrets-manager`. Fetches a single JSON secret (name derived from `STAGE`) once and caches it in memory for 30 minutes, exposing `getAdminPassword()` / `getConsumerPassword()` / `clearSecretsCache()`. Only key *names* are referenced in code; no values are stored in the repo.

### `serverless.yml` — Lambda + IAM wiring
Serverless Framework v3, `nodejs20.x`, `us-east-1`, staged via `${opt:stage, 'dev'}`. Bundles with `serverless-esbuild`, loads env via `serverless-dotenv-plugin`. Defines **two** functions (admin pre-token-generation, consumer post-confirmation) and an IAM role granting `secretsmanager:GetSecretValue` on `nanza-credentials-*` and `cognito-idp:AdminInitiateAuth` on the two user pools. The Cognito **trigger attachment itself is configured on the user pools out-of-band**, not in this file.

### `.github/workflows/` — deploy
- **`deploy-dev.yml`** — on push to `dev`, assumes an AWS role via **OIDC** (no static keys) and runs `serverless deploy --stage dev`, injecting pool/client IDs and `API_BASE_URL` from GitHub Actions `vars`.
- **`deploy-prod.yml`** — same, on push to `prod`.

## Plans
None yet. New plans land in `plans/`.

## Related
- [[architecture|Architecture]]
- [[../_shared/INDEX|Shared brain]]
