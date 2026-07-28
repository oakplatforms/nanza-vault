---
tags: [nanza-auth, solution-design, auth, social-auth, cognito]
---

# Social Identity Providers (Apple / Google) — Solution Design

## Overview

The infra half of native social sign-in. nanza-auth owns the Cognito user pools and the
Lambda triggers that create app-side user rows (`serverless.yml`,
`lambdas/consumerPostConfirmationHandler.ts`). To let nanza-mobile offer **Sign in with
Apple** and **Sign in with Google**, the **consumer pool** must (a) trust Apple and Google
as federated identity providers, and (b) create the app `User`/`Account` row on a
federated user's *first* sign-in — the same row email signups already get.

The mobile/native side (which SDKs, which buttons, the `services/auth` seam) is owned by
[[../../nanza-mobile/solution-designs/social-auth|nanza-mobile — Social Sign-In]]. This note
is the pool + trigger slice.

## Register the providers on the consumer pool

Three pieces of config on the **consumer** pool (`CONSUMER_USER_POOL_ID`):

- **Apple identity provider** — Services ID, Team ID, Key ID, and a private `.p8` key so
  Cognito can validate Apple identity tokens. Map Apple `email` → Cognito `email`, `sub` →
  federated username.
- **Google identity provider** — a Google OAuth **client ID** Cognito validates tokens
  against (plus the platform client IDs the app uses natively). Map Google `email` →
  `email`.
- **App client** — add `SignInWithApple` and `Google` to the consumer client's
  **`SupportedIdentityProviders`**.

> **A Cognito domain is required even though users never see the hosted page.** Federation
> is bound to the pool's hosted domain, so we create a domain on the consumer pool as pure
> config. In the native flow the app obtains the provider token itself and federates it; the
> hosted login page is never rendered.

Secrets (Apple `.p8`, Google client secret) follow the existing pattern — stored in
Secrets Manager under `nanza-credentials-*`, which the auth Lambda role can already read.

## Creating the app user row for federated users

Email signups create the app row via **`consumerPostConfirmationHandler`** on the
`PostConfirmation_ConfirmSignUp` trigger — it admin-auths as the service user and POSTs to
nanza-api `/user` with `authId: userSub` plus the standard account shape (email, a
`collection` list, one empty cart).

**Federated identities never fire `PostConfirmation_ConfirmSignUp`.** They are linked from an
external IdP, so no "confirm signup" event occurs. To create their row we handle the
federated first-login path — options, in preference order:

1. **Pre-Token-Generation trigger on the consumer pool** (mirrors the admin pool, which
   already uses `adminPreTokenGenerationHandler`). It fires on every token issue including
   federated ones; make the `/user` POST **idempotent on `sub`** so only the first login
   actually creates a row and every later login is a cheap no-op. *Preferred* — one trigger
   covers both signup styles.
2. **Post-Authentication trigger** — similar, fires after each sign-in; same idempotency
   requirement.

Whichever trigger is chosen, it reuses the existing admin-auth-then-POST mechanics from
`consumerPostConfirmationHandler` — the only new logic is "have I already created this sub?"
The mobile contract is simply: after a successful social auth, a `/user` row exists for the
Cognito `sub`.

> **Idempotency is the whole ballgame.** A Pre-Token / Post-Auth trigger fires on *every*
> token issue, not just the first, so the `/user` create must be safe to call repeatedly for
> the same `sub`. **Status:** `POST /user` was originally *not* idempotent — it did an
> unconditional `create`, and `authId` is `@@index`ed but **not** `@unique`, so repeats
> silently made duplicate users. Fixed in nanza-api: `POST /user` now `findFirst`s on `authId`
> and returns the existing user. A follow-up (make `authId` `@unique` to close the concurrent
> first-login race) is tracked but needs a manual migration + dedup pass.

## Account linking (decision, not default)

By default Cognito treats "email signup with `a@x.com`" and "Google login with `a@x.com`"
as **two separate identities**. If we want one merged account, that requires an explicit
`AdminLinkProviderForUser` step. Flagged here as a deliberate decision — do not assume auto
merge.

## Phase 2

Discord and Facebook extend the same recipe: register the provider on the consumer pool, add
it to `SupportedIdentityProviders`, done. Facebook is a first-class Cognito IdP; Discord is
registered as a generic **OIDC** provider.

## Setup checklist — console + credential gathering (do this first, per environment)

This is the manual, credential-gathering half — done in the Apple, Google, and AWS consoles,
**not** in code. It has no local prerequisites and can be done before any code lands. **Do
`dev` only to start.** Decide the **domain prefix first** (e.g. `nanza-consumer-dev`) because
the Apple/Google return URLs all point back at
`https://nanza-consumer-dev.auth.us-east-1.amazoncognito.com/oauth2/idpresponse`.

**Status: not started — next action for the human. Code side (the Pre-Token handler + serverless
wiring) is intentionally deferred until these credentials exist.**

### A. Apple Developer portal (developer.apple.com)
1. Ensure the app's **App ID** has **Sign in with Apple** enabled.
2. Create a **Services ID** (Identifiers → Services IDs), e.g. `com.nanza.signin` — this is the
   "client ID" Cognito wants.
3. Configure that Services ID: Primary App ID = your app; **Domains** = the Cognito domain;
   **Return URLs** = `https://<cognito-domain>/oauth2/idpresponse`.
4. Create a **Sign in with Apple key** (Keys → +) → download the **`.p8`** (one-time download);
   note the **Key ID**.
5. Note your **Team ID** (top-right, 10 chars).
- **Hand off:** Services ID, Team ID, Key ID, `.p8` contents.

### B. Google Cloud Console (console.cloud.google.com → APIs & Services → Credentials)
1. Create an **OAuth client ID → Web application** (the one Cognito validates against):
   authorized redirect URI = `https://<cognito-domain>/oauth2/idpresponse`. Note **Client ID +
   secret**.
2. Also create the **native** clients for the mobile step: **iOS** (needs bundle ID) and
   **Android** (needs package name + SHA-1). Note both client IDs.
- **Hand off:** Web client ID + secret (for Cognito), iOS + Android client IDs (for mobile).

### C. AWS Cognito console — consumer pool
1. **Add a domain** (App integration → Domain), prefix `nanza-consumer-dev` → gives the
   `…auth.us-east-1.amazoncognito.com` host used in A3/B1.
2. **Add Apple** as a federated IdP: paste Services ID, Team ID, Key ID, `.p8`; scopes
   `email name`; map Apple **email → email**.
3. **Add Google**: paste Web client ID + secret; scopes `openid email profile`; map **email →
   email**.
4. **Update the consumer app client**: tick **Apple** + **Google** under identity providers
   (keep **Cognito user pool** on for email/password); grant type **Authorization code**;
   OpenID scopes `openid email profile`; callback URL = the app deep link (scheme finalized in
   the mobile step, e.g. `nanza://auth`).

### D. Secrets
- The `.p8` contents and Google client secret go into the `nanza-credentials-dev` secret (or the
  IdP config itself) — **never git**. Repeat A–C per environment (dev → stage → prod) later.

### Then (code side, deferred)
- Add `lambdas/consumerPreTokenGenerationHandler.ts` (mirror `adminPreTokenGenerationHandler`;
  gate on `event.request.userAttributes.identities` so only **federated** users create a row,
  since email users are covered by PostConfirmation) + wire it in `serverless.yml` + attach it
  as the consumer pool's **Pre Token Generation** trigger. Full detail in the scratch plan
  `nanza-auth/.plans/social-identity-providers.md`.

## Related

- [[../../nanza-mobile/solution-designs/social-auth|nanza-mobile — Social Sign-In]] — the
  native SDK integration and UI.
- [[../architecture|Architecture]] — the pools, triggers, and service-user auth pattern.
