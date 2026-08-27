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

## ⚠️ CORRECTION (2026-08-24): PostConfirmation DOES fire for federated users

Verified on device with a first-time Sign in with Apple: **the existing
`consumerPostConfirmationHandler` fired and created the app User/Account row.** The
section below (and the Pre-Token-Generation plan) was built on the assumption that it
never would — that assumption is wrong for the hosted-UI OAuth path the app actually
uses. **No new trigger is needed.** The `/user` idempotency work stands on its own merits.

What the created row looks like: `type: REGISTERED`, an **empty profile (no username)**,
and **no email when the provider attribute mapping is missing** — completing it is the
mobile side's job (the generic completion screen in nanza-mobile routes every social
sign-in with a missing username/email through an email+username form before the app
becomes usable).

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

## Account linking — DECIDED: block, don't link (2026-08-26)

By default Cognito treats "email signup with `a@x.com`" and "Google login with `a@x.com`"
as **two separate identities**. Merging would need an explicit `AdminLinkProviderForUser`
step. **Skylar's decision: don't merge — refuse the social sign-up** and tell the user to
log in with their email and password.

Implemented as `lambdas/consumerPreSignUpHandler.ts`, attached as the consumer pool's
**Pre sign-up** trigger:
- Acts only on `PreSignUp_ExternalProvider` (native sign-ups already run the same check
  from the app). Fires only on a federated identity's FIRST sign-in.
- Calls nanza-api's **`/auth/check-for-user`** — the very lookup the manual form uses
  (account by email, case-insensitive) — with the service-user token, same pattern as
  PostConfirmation. One source of truth for "does this email have an account".
- On 409 → throws `An account with this email already exists. Please log in with your
  email and password instead.` Cognito aborts creation (no pool user, no PostConfirmation,
  no app row) and returns the message on the OAuth redirect as
  `error_description="PreSignUp failed with error <msg>."`; `SocialAuthButtons` strips the
  wrapper and shows the sentence.
- **Fails CLOSED** on API/network errors — a duplicate account is harder to undo than a
  retried sign-up.
- Apple: relay addresses never collide; a shared real email that does is blocked the same
  way. Provider-agnostic on purpose.
- Sets `autoVerifyEmail` so the pool user is born verified.

**Attachment is manual, like the other triggers:** Cognito console → pool → Extensions /
Lambda triggers → Sign-up → **Pre sign-up** → pick `nanza-consumer-pre-sign-up-<stage>`.
Needs a nanza-auth deploy first. Repeat per pool (dev, then prod — see the checklist).

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

**Status: DEV COMPLETE (2026-08-25).** Apple and Google both live end-to-end on the dev
pool — sheet → provider → locked username sheet → in, verified on device. No Pre-Token
handler was ever needed (see the correction above). The checklist below is kept as the
recipe; **the next action is the prod rollout at the bottom of this doc.**

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

## Prod rollout checklist (repeat for the prod pool)

What carries over and what doesn't, learned the hard way in dev:

**Already done once, never repeated:**
- The `.p8` Apple key (keys aren't environment-scoped — same file, same Key ID, same
  Team ID for both pools).
- Private Email Relay sender registration + all the SPF/DKIM/DMARC DNS (domain-level).
- The Google consent screen (project-level) — but see the PUBLISH step below.
- All mobile and api code.

### A. Prod Cognito domain first
- [ ] Prod pool → App integration → Domain. If none exists, create the Cognito-prefix
      domain; either way **note the full host** — every Apple/Google URL below uses it.

### B. Apple portal (~5 min, step by step)

Two Services IDs on purpose: the Services ID is the client ID Cognito validates tokens
against, and sharing dev's would make dev tokens structurally valid for prod.

1. [ ] developer.apple.com → Account → **Certificates, Identifiers & Profiles** →
       **Identifiers** (left sidebar)
2. [ ] Blue **⊕** next to "Identifiers" → choose **Services IDs** (second option, under
       App IDs) → Continue
3. [ ] Description: `Nanza Sign In Prod` · Identifier: **`com.nanza.signin`** →
       Continue → **Register** (the form closes unconfigured — that's normal)
4. [ ] Back on the Identifiers list, switch the top-right dropdown from **App IDs** to
       **Services IDs** → click `com.nanza.signin`
5. [ ] Tick the **Sign in with Apple** checkbox → a **Configure** button appears → click it
6. [ ] In the panel:
       - **Primary App ID**: the Nanza app
       - **Domains and Subdomains**: the prod Cognito host — host only, no `https://`,
         no path (e.g. `us-east-1XXXXXXX.auth.us-east-1.amazoncognito.com`)
       - **Return URLs**: the same host as a full URL —
         `https://<prod-host>/oauth2/idpresponse`
7. [ ] **Save** in the panel, then **Continue → Save** on the outer page — the two-level
       save; skipping the outer one silently discards everything. Reopen the Services ID
       once to confirm the URLs stuck.
8. [ ] **No new key, no new capability** — the dev round's `.p8`, Key ID, and Team ID are
       reused as-is in step D; the App ID's Sign in with Apple capability is already on.

If Apple rejects the domain: it's almost always a stray `https://` or trailing slash in
the Domains field (URL only belongs in Return URLs).

### C. Google Cloud console (~5 min)
- [ ] Same project. Credentials → new **OAuth client ID → Web application**,
      `Nanza Cognito Prod`, redirect URI `https://<prod-host>/oauth2/idpresponse`.
      (Separate client for the same isolation reason as the Services ID.) Note ID + secret.
- [ ] **PUBLISH the consent screen** (OAuth consent screen → Publish app). Testing mode
      only admits listed test users — fine for dev, fatal in prod. The basic scopes
      (openid/email/profile) need no Google review.

### D. Prod pool config (~10 min)
- [ ] Add **Apple** IdP: Services ID `com.nanza.signin`, same Team ID / Key ID / `.p8`;
      scopes `email name`; **map email → Email** (final decision: mapping ON — relay
      addresses are accepted; senders are registered so they deliver).
- [ ] Add **Google** IdP: the prod web client ID + secret; scopes `profile email openid`;
      **map email → Email**.
- [ ] App client (the existing one — never a new one) → Login pages → Edit:
      tick **Apple + Google** alongside Cognito user pool; callback `nanza://auth`;
      sign-out `nanza://signout`; Authorization code grant; scopes `openid email profile`.
      The console accepts the custom scheme (verified in dev despite docs suggesting
      otherwise).

### D2. Triggers on the prod pool
- [ ] Deploy nanza-auth to prod (GitHub deploy-prod on push to `prod`) so
      `nanza-consumer-pre-sign-up-prod` exists.
- [ ] Prod pool → Lambda triggers → **Pre sign-up** → attach it. (Confirm
      Post confirmation is attached too — it's what creates the app row.)

### E. The two non-console bits
- [ ] Mobile prod env: `COGNITO_DOMAIN=<prod host>` alongside the prod pool/client IDs.
- [ ] nanza-api **prod deploy** — carries `syncCognitoEmail`, its
      `AdminUpdateUserAttributes` IAM grant, and the email-collision 409. (Skylar runs
      deploys.)

### F. Verify
- [ ] Fresh Apple sign-up on a prod build: consent prompt → Apple sheet → locked
      username sheet → in. Repeat for Google (username only; email arrives mapped).
- [ ] Check the created pool user has the email attribute; the DB row has
      email + username.

## Custom auth domain — `auth.nanza.app` instead of `*.amazoncognito.com` (plan, 2026-08-26)

> **Status: plan.** Skylar's ask: the sign-in sheet's consent prompt / Custom Tab should show
> the Route 53 domain, not `nanza.auth.us-east-1.amazoncognito.com`. This is Cognito's
> **custom domain** feature — console + DNS work, with a single code flip at the end. The
> pool is console-managed (nanza-auth's serverless.yml owns only the triggers), so nothing
> here is codified.

**Naming:** `auth.nanza.app` (prod pool) and `auth-dev.nanza.app` (dev pool). One custom
domain per pool; it can coexist with the existing prefix domain, so nothing breaks during
the switch and rollback is just flipping `COGNITO_DOMAIN` back.

**Cognito's preconditions (all already met for `nanza.app`):**
- The domain must be a subdomain — the apex is not allowed.
- The **parent** (`nanza.app`) must resolve to an A record — it does (the CloudFront site).
- The hosted zone is on Route 53 (awsdns name servers) — confirmed with `dig NS`.
- An **ACM certificate in `us-east-1`** covering the subdomain (Cognito fronts the domain with
  CloudFront, which only reads us-east-1 certs). A wildcard `*.nanza.app` cert covers both
  environments; DNS-validate it in the same hosted zone.

**Order of operations (per environment; do dev first):**
1. **ACM (us-east-1):** request `auth.nanza.app` (or `*.nanza.app`), DNS validation → add the
   CNAME it gives you in Route 53 → wait for *Issued*.
2. **Cognito → consumer pool → App integration → Domain → "Use a custom domain":** enter
   `auth.nanza.app`, pick the cert. Cognito returns an **alias target** (a `*.cloudfront.net`
   host). The domain sits in *Creating* for up to ~an hour.
3. **Route 53 → `nanza.app` zone:** add an **A record, Alias = yes**, name `auth`, target =
   that CloudFront host (choose "Alias to CloudFront distribution" and paste it; region is
   implicit). Add a matching **AAAA alias** for IPv6 if the rest of the zone has them.
   *This is the only DNS entry Cognito needs.*
4. **Google Cloud → the Web OAuth client:** ADD `https://auth.nanza.app/oauth2/idpresponse`
   to authorized redirect URIs (keep the amazoncognito one until the cut-over is proven).
5. **Apple Developer → the Services ID:** ADD `auth.nanza.app` under Domains and
   `https://auth.nanza.app/oauth2/idpresponse` under Return URLs (Apple allows several).
6. **Verify the domain answers:** `curl -sI https://auth.nanza.app/.well-known/openid-configuration`
   → 200 once *Active*. (`/oauth2/authorize` without params 400s — that's fine.)
7. **Code flip — the only code change:** nanza-mobile `.env` `COGNITO_DOMAIN=auth.nanza.app`
   (dev block: `auth-dev.nanza.app`). `services/auth/cognito.ts` builds every OAuth endpoint
   from that one value; the `issuer` stays `cognito-idp.us-east-1.amazonaws.com/<pool>` —
   it is the token issuer, not the hosted domain, and must NOT change. Callback URLs
   (`nanza://auth`, `nanza://signout`) are untouched. Restart Metro with `--reset-cache`.
8. **Test both providers on both platforms**, then remove the amazoncognito redirect URIs
   from Google/Apple and (optionally) delete the prefix domain.

**What changes for users:** iOS's one-time prompt reads "Nanza wants to sign in using
nanza.app"; Android's Custom Tab URL bar shows `auth.nanza.app`. Nothing else in the flow
moves.

**Gotchas:**
- The cert must be in **us-east-1** even though the pool is too — the constraint is
  CloudFront's, and a cert in any other region simply won't appear in the picker.
- If the `nanza.app` zone ever loses its apex A record, custom-domain creation fails with an
  opaque error — it is the parent-A-record rule.
- Custom domain status updates lag; don't flip `COGNITO_DOMAIN` until step 6 returns 200.

## Related

- [[../../nanza-mobile/solution-designs/social-auth|nanza-mobile — Social Sign-In]] — the
  native SDK integration and UI.
- [[../architecture|Architecture]] — the pools, triggers, and service-user auth pattern.
