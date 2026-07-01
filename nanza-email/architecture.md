# nanza-email — Architecture

Transactional email for Nanza. One AWS Lambda, event-driven, renders [react-email](https://react.email) templates and sends them through Amazon SES. It is deliberately small — a dispatcher, a set of templates, and a SES helper — so this note stays proportional to the repo.

See the [[INDEX]] for the file-by-file map. For cross-project concerns see the [[../_shared/INDEX|shared brain]].

---

## Shape of the service

- **Runtime:** Node 20 (`nodejs20.x`), region `us-east-1`.
- **Packaging:** Serverless Framework v3 + `serverless-esbuild` (bundled, sourcemaps on, `aws-sdk` excluded) + `serverless-dotenv-plugin`.
- **Single function:** `emailDispatcher`, named `nanza-email-${stage}`, handler `handler/index.handler`.
- **No HTTP surface.** The Lambda is invoked only by EventBridge — there is no API Gateway, no scheduled trigger.

Defined in `serverless.yml`.

---

## Trigger: EventBridge → Lambda

nanza-api publishes domain events onto the **default** EventBridge bus with `source: nanza`. `serverless.yml` declares an `AWS::Events::Rule` (`nanza-email-template-rule-${stage}`) whose `EventPattern` filters on `source: [nanza]` and an explicit `detail-type` allow-list. A companion `AWS::Lambda::Permission` lets EventBridge invoke the function.

The `detail-type` list in the rule is the source of truth for what this service reacts to. As of now:

- `invoice.confirmation.customer`
- `order.confirmation.seller`
- `order.canceled.customer`, `order.canceled.seller`
- `order.transit.customer`
- `order.delivered.customer`
- `order.delivery.failed.customer`, `order.delivery.failed.seller`
- `order.failed.customer`, `order.failed.seller`
- `order.refund.request.seller`
- `order.refund.declined.customer`, `order.refund.accepted.customer`
- `seller.verified`, `seller.completed`, `seller.application.rejected`
- `user.password.changed`

> Two events in the rule's allow-list are not (yet) handled in code: `order.delivered.seller` appears in the local preview config but not in the dispatcher, and the rule's `detail-type` and the dispatcher's `type` checks must be kept in sync by hand. If a new email type is added, both `serverless.yml` (the rule) and `handler/emailDispatcher.tsx` (the branch) need updating.

---

## Dispatch: `handler/emailDispatcher.tsx`

`handler/index.ts` is a one-liner: `export { handler } from './emailDispatcher'`.

The handler reads `event.detail` and destructures the fields the publisher may include:
`{ orderId, invoiceId, accountId, sellerId, type, cancellationReason, rejectionReason, sellerName, sellerEmail }`.

It then runs a flat chain of `if (type === '...')` branches. Each branch typically:

1. Calls a read-only API client (`orderService` / `invoiceService` / `accountService` / `sellerService`) to fetch the entity, passing a hand-built `?include=...` query so nested relations (customer/seller account+profile, order listings + entity, shipping options, shipments, refund) come back in one call.
2. Picks the recipient email off the fetched object (customer vs. seller depending on the event suffix).
3. Renders the matching template with the data and calls `sendEmail(to, <Template … />, subject)`.

A few branches are special:
- **`seller.completed`** emails a hard-coded admin list (`skylar@`, `tahir@oakplatforms.com`) with `NewSellerNotification` — an internal "new seller pending verification" alert, not a customer email.
- **`seller.application.rejected`** takes `sellerEmail`, `sellerName`, and `rejectionReason` straight off the event (no fetch needed).
- Shared-template events (cancel, delivery-failed, failed) pass a `recipientType` prop so one template serves both customer and seller.

The handler is intentionally sequential (`await` per send) and does minimal error handling — a failed fetch or send bubbles up and the Lambda invocation errors.

---

## Sending: `utils/sendEmail.ts`

The one place SES is touched.

```ts
const ses = new SESClient({ region: 'us-east-1' })
```

`sendEmail(to, email, subject?)`:
- returns early if `to` is empty (guards against missing recipient emails);
- renders the react-email element **twice** — once as HTML, once as `plainText` — via `@react-email/render`;
- builds a `SendEmailCommand` with both an `Html` and `Text` body (UTF-8) and the subject (default `'Your Nanza Order Confirmation'`);
- sets `Source: process.env.SES_SOURCE_EMAIL || ''` and calls `ses.send(command)`.

This is `SendEmail` (not `SendRawEmail` / templated SES) — HTML is generated in-process from React, SES just delivers it.

---

## Templates & shared UI

Each file in `templates/` is a react-email component exporting a named function (e.g. `SellerOrderConfirmation`, `PasswordChange`). A template:
- opens `<Html>` with a `<head>` that pulls the **Figtree** Google Font and injects `fontFaces` from `styles/`;
- renders a `<Container>` built from `@react-email/components` primitives (`Section`, `Text`, `Container`, `img`);
- leans on the shared `styles` object rather than ad-hoc inline CSS.

Shared building blocks live in `components/`:
- **`LogoSection`** renders the brand logo from `${REACT_APP_S3_BASE_URL}/logo.png?v=<LOGO_VERSION>`. The `?v=` query param exists specifically to bust Gmail/Google's image proxy cache (`ci*.googleusercontent.com`) when the logo changes.
- **`OrderCard`** renders order line items and totals, using `utils/shipping.ts` for rate/quantity math.

`styles/theme.ts` hard-codes hex/rgba tokens (mapped from the app's brand theme) because email clients don't support CSS variables; `styles/index.ts` assembles the `styles` object and `@font-face` block consumed by templates.

Domain types come only from `@oakplatforms/types`, re-exported through `types/index.ts` (`OrderDto`, `InvoiceDto`, `AccountDto`, `UserDto`, shipping DTOs). No local DTO definitions.

---

## Talking back to the API: `services/api/`

The Lambda is a **read-only client** of nanza-api. `services/api/index.ts` defines a shared `fetchData<T>` that:
- lazily fetches and **caches a guest token** (`GET ${API_BASE_URL}/user/guest-token`) for the life of the warm container;
- attaches `Authorization: Bearer <token>`;
- throws a typed `RateLimitError` on HTTP 429 and a generic error otherwise.

Thin per-resource wrappers sit on top: `Order.ts`, `Invoice.ts`, `Account.ts`, `Seller.ts` — each just builds a URL and calls `fetchData`. This is where all outbound API endpoints and the guest-token auth convention live.

---

## Config / environment

Three environment variables flow through `serverless.yml` → Lambda:

| Var | Purpose |
| --- | --- |
| `API_BASE_URL` | Base URL of nanza-api, used by every `fetchData` call and the guest-token fetch. |
| `REACT_APP_S3_BASE_URL` | Public asset base (logo etc.) referenced from `LogoSection`. |
| `SES_SOURCE_EMAIL` | The **From** address SES sends as. This is the sender-domain lever. |

`SES_SOURCE_EMAIL` is the sender identity the whole service ships under; the broader Nanza direction is to send from a verified SES domain (e.g. `no-reply@nanza.app`) to improve deliverability / avoid spam. That address is *only* an env var here — nothing in code hard-codes it, so switching the sender domain is a config change plus a verified SES identity, no code change.

`.env` (local) holds `API_BASE_URL` and `REACT_APP_S3_BASE_URL`. In CI these come from repo/environment **Variables**, not secrets (see below). `LOGO_VERSION` is an optional extra used by `LogoSection`.

The Lambda's IAM role grants only `ses:SendEmail` and `ses:SendRawEmail` on `Resource: '*'`.

---

## Local preview

`local/` is an offline harness so templates can be iterated without deploying:
- `configs.ts` enumerates preview event types (mirroring the dispatcher) and shared `include` lists;
- `preview.tsx` renders the chosen scenario to `preview.html`;
- `npm run preview` runs `nodemon` on the templates + `live-server` on `local/` for hot reload.

---

## Deploy

Serverless-based, one stack per stage.

- **Scripts:** `npm run deploy:dev` / `deploy:prod` (`serverless deploy --stage …`); `npm run build` runs `tsc`.
- **CI:** `.github/workflows/deploy-dev.yml` deploys on push to `dev`; `deploy-prod.yml` on push to `prod`. (README also mentions a `stage` branch → stage environment; only dev and prod workflows exist in the repo.)
- **Auth:** GitHub **OIDC** assumes an AWS role (`AWS_ROLE_ARN_DEV` / `_PROD` repo vars) — no long-lived AWS keys. `@oakplatforms` packages install from GitHub Packages using the workflow `GITHUB_TOKEN`.
- **Env in CI:** `API_BASE_URL`, `REACT_APP_S3_BASE_URL`, `SES_SOURCE_EMAIL` are injected from GitHub **Variables** (`vars.*`) at deploy time.
- The Serverless dashboard/telemetry is explicitly disabled in both workflows.

Deploying replaces the Lambda **and** re-applies the EventBridge rule + permission from `serverless.yml`, so adding a new event type is a code + config change followed by a redeploy.

---

## Notes / sharp edges

- **Rule and dispatcher must stay in sync.** A `detail-type` present in the rule but missing a branch in `emailDispatcher.tsx` silently does nothing; a branch whose `detail-type` isn't in the rule never fires.
- **Admin recipient list is hard-coded** in the `seller.completed` branch — not env-driven.
- **Guest-token caching** is per warm container; a cold start re-fetches. There is no retry/back-off beyond the `RateLimitError` throw.
- **No dedicated test suite** in the repo; correctness of rendered emails is checked via the `local/` preview.
