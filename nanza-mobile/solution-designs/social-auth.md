---
tags: [nanza-mobile, solution-design, auth, social-auth, cognito]
---

# Social Sign-In (Apple / Google) — Solution Design

## Overview

Add **Sign in with Apple** and **Sign in with Google** to nanza-mobile as one-tap
sign-up / sign-in options alongside the existing email + password flow. The design goal
is a **fully native, in-app experience** — no bounce to Safari — while still landing every
user in the same **consumer Cognito user pool** and the same app-side `User`/`Account`
row that email sign-ups produce today.

Discord and Facebook are explicitly **phase 2** — the same federation mechanism extends to
them, so this doc frames the pattern once and calls out where the extra providers slot in.

The current email/password path is unchanged. This is purely additive: two new buttons
that feed the *same* Cognito pool through a *different* front door.

## The core decision: native SDKs, not Cognito Hosted UI

Cognito supports federated identity providers, but there are two ways to drive them:

1. **Hosted UI (OAuth redirect).** Cognito hosts a web login page under a
   `*.auth.us-east-1.amazoncognito.com` domain; the app opens it in a browser session,
   the user signs in, Cognito redirects back with a code. **Rejected** — it makes social
   login feel like leaving the app, and it's a second, divergent auth path from our
   direct-SDK email flow.
2. **Native provider SDKs → federate the resulting token into Cognito.** The app uses
   Apple's native `Sign in with Apple` sheet and Google's native account picker to obtain
   a provider **identity token**, then exchanges that token for Cognito sessions. This is
   the chosen path — it keeps the whole flow in-app.

> Today mobile talks to Cognito **directly** via `amazon-cognito-identity-js` (see
> [[../architecture|Architecture]] and `src/services/auth/cognito.ts`) — no Amplify, no
> hosted UI. Native social login preserves that "SDK-first, no web redirect" character.

### What each platform shows

| Platform | Apple button | Google button |
| --- | --- | --- |
| **iOS** | Native "Sign in with Apple" system sheet (Face ID). **Required** by App Store guidelines whenever any other social login is offered. | Native Google account-picker sheet. |
| **Android** | No native sheet — falls back to a web popup for the Apple button only (phase-2 optional). | Native Google account-picker sheet. |

Cross-provider works both directions: Google on an iPhone is native; Apple on Android is a
web view. This asymmetry is normal and expected — it is Apple's platform rule, not a bug.

## How it works — the federation flow

The seam is: **native SDK gives us a token → Cognito trusts that token because the provider
is registered on the pool → Cognito issues its own tokens → first-time users get an app row.**

```
[Native Apple/Google sheet]  ──id_token──▶  [Cognito consumer pool, provider registered]
                                                    │
                                          Cognito issues its own
                                          access / id / refresh tokens
                                                    │
                                    first federated login for this identity?
                                          │yes                   │no
                                          ▼                       ▼
                              create app User/Account row     just sign in
                              (same shape as email signup)
```

Two independent registration steps make this work — one in the pool (infra), one in the
device (app):

### 1. Register Apple & Google as identity providers on the consumer pool

Lives in **[[../../nanza-auth/solution-designs/social-identity-providers|nanza-auth]]** (the
repo that owns the Cognito pools; see `serverless.yml`). Each provider needs:

- **Apple:** a Services ID, an Apple Team ID, a Key ID, and a private key (`.p8`) — used by
  Cognito to validate the Apple identity token. Attribute mapping: Apple `email` →
  Cognito `email`, `sub` → the federated username.
- **Google:** an OAuth **client ID** (one per platform — iOS, Android, and a "server"/web
  client Cognito validates against). Attribute mapping: Google `email` → `email`.
- The consumer app client must list Apple and Google in **`SupportedIdentityProviders`**.

> **Note — a Cognito domain is still required even for native SDKs.** Cognito's federation
> machinery is attached to the pool's hosted domain, so we must create a domain on the
> consumer pool even though users never *see* the hosted page in the native flow. This is
> pure config, not UX.

### 2. Wire the native SDKs in mobile and exchange tokens

- **iOS Apple:** `@invertase/react-native-apple-authentication` (or the RN community lib)
  for the native sheet → returns an Apple `identityToken`.
- **Google:** `@react-native-google-signin/google-signin` → returns a Google `idToken`.
- The app federates that token into the consumer pool. With `amazon-cognito-identity-js`
  alone this is awkward (the lib is built for USER_PASSWORD flows), so the token exchange
  is the one place we reach for the Cognito **Identity Provider** federated call —
  isolated behind our own `services/auth` module so screens stay SDK-agnostic. New
  functions live alongside the existing `signIn` / `signUp` in
  `src/services/auth/cognito.ts` (e.g. `signInWithApple`, `signInWithGoogle`), returning
  the same `{ isSignedIn }` shape so `LoginForm` / `SessionContext` don't care how the user
  arrived.

### 3. Create the app-side user row on first social sign-in

This is the subtle part. Email signups create the `User`/`Account` row via the
**`consumerPostConfirmationHandler`** Lambda (`PostConfirmation_ConfirmSignUp` trigger) in
nanza-auth, which POSTs to nanza-api `/user` with the standard account shape (email, a
`collection` list, an empty cart — see [[../../nanza-api/solution-designs/users|nanza-api —
Users]]).

**Federated users do NOT fire `PostConfirmation_ConfirmSignUp`** — they never "confirm a
signup," they're linked from an external IdP. So we need a trigger that fires for them. The
handler is extended (or a sibling trigger added) to also run on the federated first-login
path so that the *first* time an Apple/Google identity authenticates, the same `/user` POST
runs with the email from the provider claims. Idempotency: keying the app row on the Cognito
`sub` (already the pattern — `authId: userSub`) means a repeat login is a no-op create.

> Detail of *which* trigger + how first-login is detected for federated identities is owned
> by [[../../nanza-auth/solution-designs/social-identity-providers|nanza-auth]] — it's an
> infra concern. Mobile's contract is just: "after a successful social auth, a `/user` row
> exists for this sub."

## UI surface

- **Where:** the existing auth entry (`src/screens/auth/SignUpScreen/LoginForm.tsx` and the
  sign-up `Welcome`/`Email` steps). Add an "or continue with" divider under the primary
  Log In / Sign Up button, then the Apple + Google buttons.
- **Styling:** buttons follow the theme system — no inline styles, tokens from
  `dark.json`, style buckets in `styles/components/` (see [[design-system|Design System]]).
  Apple's button must follow Apple's Human Interface brand rules (black/white, Apple logo);
  Google's follows Google's branding guidelines.
- **Session:** on success the handlers call `setIsSignedIn(true)` exactly like `login()`
  does today, so the app drops the user into the authenticated stack with no new navigation
  wiring. `navigation/` stays read-only.

## Boundaries & phase 2

- **Phase 1 (actionable now):** Apple + Google, native, iOS-first polish; Android gets
  native Google and (optionally) web-popup Apple.
- **Phase 2:** Discord and Facebook — same "register provider on pool + native/web SDK +
  federate token" recipe. Facebook has a native RN SDK; Discord is OAuth-only (web popup or
  a generic OIDC provider on Cognito).
- **Account linking** (same email arriving via email-signup *and* Google) is a known
  Cognito sharp edge — by default it creates *separate* identities for the same email. If
  we want them merged, that's an explicit `AdminLinkProviderForUser` step, called out here
  as a decision to make, not silently assumed.

## Cross-repo map

- **[[../../nanza-auth/solution-designs/social-identity-providers|nanza-auth]]** — pool
  provider registration, Cognito domain, app-client config, and the federated user-row
  trigger. *(Infra owner.)*
- **[[../../nanza-api/solution-designs/users|nanza-api — Users]]** — the `/user` POST
  contract the trigger calls; unchanged by this feature.
- **This doc (nanza-mobile)** — native SDK integration, the `services/auth` seam, and the
  UI surface.

## Related

- [[../architecture|Architecture]] — how auth is wired (direct Cognito SDK, `SessionContext`).
- [[../../_shared/decisions/|Shared decisions]] — cross-project auth stack lives here.
