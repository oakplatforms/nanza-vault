# nanza-email

Nanza's transactional email service. It is a single AWS Lambda (`nanza-email-<stage>`, Node 20, deployed with the Serverless Framework) that reacts to domain events on the default EventBridge bus, renders a [react-email](https://react.email) template to HTML + plaintext, and sends it through Amazon SES. It has no HTTP API of its own — the only entrypoint is EventBridge. nanza-api publishes events like `order.confirmation.seller` or `seller.verified`; this Lambda then fetches whatever data it needs back from that API (as a guest-token client) and emails the right recipient. Small service: ~13 templates, one dispatcher, one SES helper.

## Structural overview

Real source lives at `nanza-email/` (repo root); `documentation/` there is a symlink into this vault.

- **`handler/`** — Lambda entrypoint. `index.ts` re-exports `handler` from `emailDispatcher.tsx`, a large `if (type === ...)` chain over `event.detail.type` that maps each event to a template + recipient.
- **`templates/`** — one `.tsx` react-email template per email: `SellerOrderConfirmation`, `CustomerInvoiceConfirmation`, `OrderCancelled`, `OrderInTransit`, `OrderDelivered`, `OrderFailed`, the three `OrderRefund*` (Request / Declined / Accepted), `PasswordChange`, `SellerAccountVerified`, `SellerApplicationRejected`, `NewSellerNotification`.
- **`components/`** — shared email pieces: `LogoSection` (S3-hosted logo with cache-busting version param) and `OrderCard` (line-item / totals block).
- **`styles/`** — `theme.ts` (hard-coded hex tokens, since email clients can't use CSS variables) and `index.ts` (the `styles` object + `@font-face` declarations, Figtree).
- **`utils/`** — `sendEmail.ts` (the SES client + `SendEmailCommand` wrapper), `shipping.ts` (rate/quantity math for order emails), `formatPriceAsDollar.ts`.
- **`services/api/`** — thin read-only clients (`Order`, `Invoice`, `Account`, `Seller`) over a shared `fetchData` that authenticates with a cached guest token against `API_BASE_URL`.
- **`types/index.ts`** — re-exports DTOs (`OrderDto`, `InvoiceDto`, `AccountDto`, …) from `@oakplatforms/types`; no local domain types.
- **`local/`** — offline preview harness (`preview.tsx` + `configs.ts` + generated `preview.html`) so templates can be rendered without deploying; run via `npm run preview`.
- **`serverless.yml`** — defines the Lambda, its SES IAM permissions, the `SES_SOURCE_EMAIL` env var, and the EventBridge rule listing every event type it subscribes to.
- **`.github/workflows/`** — `deploy-dev.yml` / `deploy-prod.yml` (branch-push deploys via GitHub OIDC + Serverless).

## How sending works (one line)

EventBridge rule (`source: nanza`) → `emailDispatcher` Lambda → fetch data from nanza-api → render react-email template → `sendEmail()` → SES `SendEmailCommand` with `Source = SES_SOURCE_EMAIL`.

## Plans

`plans/` is empty — no plans have been written for this service yet. New solution designs land here.

## Related

- [[architecture|Architecture]] — deeper walkthrough of triggers, sending, templates, config, and deploy.
- [[../_shared/INDEX|Shared brain]] — cross-project architecture & catalog.

> Keep this file a short map that links out.
