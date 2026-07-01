---
tags: [nanza-web-app, solution-design, auth]
---

# Auth — Solution Design

## Overview

Authentication for the web app is a **ported copy of the nanza-mobile Cognito
wrapper** — the same `amazon-cognito-identity-js` function-per-operation API, the
same dev user pool, the same session-context shape. It is designed so a signed-in
visitor makes API calls with a real Cognito JWT while everyone else rides a cached
guest token.

**Current live reality:** the marketplace is switched off. `services/auth/cognito.ts`
ships as a deliberate **guest-only no-op stub**, and the auth routes in
`app/AppLayout.tsx` are commented out. So in the deployed build nobody is ever signed
in — every request uses the guest token. The full design below is what exists in git
history and re-activates when the marketplace is turned back on; it is documented here
as the settled design, not as pending work.

## How it works

### The Cognito wrapper — `src/services/auth/cognito.ts`

A direct port of `nanza-mobile/src/services/auth/cognito.ts`, minus the React Native
`AsyncStorage` adapter — the browser uses `window.localStorage` as its token store
natively, which is what `amazon-cognito-identity-js` defaults to. Same exports, same
signatures:

- `signUp` — email + password → `{ userId }`
- `confirmSignUp` — 6-digit email verification code
- `resendSignUpCode`
- `signIn` → `{ isSignedIn }`
- `signOut`
- `resetPassword` / `confirmResetPassword` — forgot-password request + confirm
- `updatePassword`
- `getCurrentUser` → `{ userId, username }`
- `fetchAuthSession` → `{ tokens: { accessToken }, userSub }`

Pool config reads from `process.env.REACT_APP_COGNITO_USER_POOL_ID` and
`REACT_APP_COGNITO_USER_POOL_CLIENT_ID`, matching the CRA env-var convention. The pool
is the shared dev pool (`us-east-1_BcRP6qegV`); the app client is reused from mobile
for now (a dedicated web client is a later cleanup).

**Stubbed state:** the live build swaps this for a no-op — every mutating export throws
`AUTH_DISABLED_MESSAGE`, `signOut` is a no-op, and `fetchAuthSession()` returns
`{ tokens: undefined, userSub: undefined }`. That empty session is exactly what makes
the API layer fall through to the guest token.

### Session context — `src/contexts/SessionProvider.tsx`

Mirrors the mobile `SessionContext`, trimmed for web. On mount it calls
`fetchAuthSession()` to derive `isSignedIn`; when signed in it loads the current user
via `getCurrentUser()` → `userService.get(userId, include=…)` (pulling
`account.profile/seller/customer/carts/lists`). It exposes through `useSession()`:

```ts
{
  isSignedIn: boolean | undefined,   // undefined = still checking
  currentUser: UserDto | undefined,
  refreshCurrentUser,
  signOut,                            // wraps the wrapper + clears state
}
```

With the stub in place `isSignedIn` stays `false` and `currentUser` stays `undefined`.
The provider is mounted in the `app/index.tsx` provider stack
(`QueryClientProvider → SessionProvider → CartProvider → AppLayout`).

### JWT on API requests — `src/services/api/index.ts`

`getJwtToken()` prefers the Cognito access token when a session exists and otherwise
falls back to the guest token — strictly additive over the original guest-only path:

```ts
async function getJwtToken(): Promise<string | null> {
  try {
    const session = await fetchAuthSession()
    const token = session.tokens?.accessToken?.toString()
    if (token) return token
  } catch { /* fall through */ }
  return await getGuestToken()
}
```

Because the stub's session is always empty, live traffic always takes the guest-token
branch. The guest token is cached in-module with an expiry, sourced from
`REACT_APP_GUEST_TOKEN` or fetched from `REACT_APP_GUEST_TOKEN_ENDPOINT`
(default `/user/guest-token`).

### Auth screens — `src/app/Auth/`

Full-page screens (not a modal — the web app's navigation idiom is full pages, unlike
mobile's modal flow), built from the app's Tailwind/Catalyst primitives (`Button`,
`Input`, `Dialog`, `Heading`, `Text`) to stay visually consistent:

- `Auth/SignUp/index.tsx` — email + password → `signUp` → confirmation step
- `Auth/SignUp/ConfirmationCode.tsx` — 6-digit code → `confirmSignUp` then `signIn`
- `Auth/SignIn/index.tsx` — email + password → `signIn`, stays on the current page
- `Auth/ForgotPassword/index.tsx` — a single page with progressive steps
  (email → reset code → new password), matching mobile's flow

The auth surface lives in the web app **shell** (the storefront/profile Layout variant),
not on the marketing landing page — a sign-in / avatar pill sits in the left `Toolbar`
above the gear icon. Post-sign-in the user stays on the current page and the header
avatar becomes visible.

The routes (`/sign-in`, `/sign-up`, `/forgot-password`) are additive entries in
`app/AppLayout.tsx`; in the live build they are **commented out** alongside the rest of
the marketplace.

### Env vars

```
REACT_APP_COGNITO_USER_POOL_ID=us-east-1_BcRP6qegV
REACT_APP_COGNITO_USER_POOL_CLIENT_ID=<web app client>   # reuses mobile's for now
```

Keys are declared in `.env`; values are injected at build time from GitHub Actions vars
(dev workflow), never committed. The prod workflow omits them, matching the disabled
state.

## Key decisions & rationale

- **Reuse the mobile pool + package rather than a hosted-UI redirect flow.** One mental
  model across mobile/web/admin, a single-file port, and a self-hosted form UX that
  matches the mobile look-and-feel. The Cognito Hosted UI / federated-redirect flow was
  rejected as heavier and needing extra hosting config.
- **`window.localStorage`, no storage adapter.** `amazon-cognito-identity-js` is
  platform-agnostic; the only RN-specific piece (the AsyncStorage wrapper) is dropped
  and the browser default is used.
- **JWT preference is additive.** The guest-token fallback is preserved, so nothing
  breaks when no one is signed in — which is precisely the live state.
- **Full-page auth screens, not a modal.** Fits the web app's page-navigation idiom;
  the mobile modal pattern was rejected for web.
- **Ship the stub, keep the wiring.** Rather than deleting auth, the live build stubs
  the wrapper and comments the routes, so the marketplace re-enables by restoring the
  real wrapper from git history and uncommenting the routes — no rebuild.

## Related

- [[../INDEX|nanza-web-app]]
- [[../architecture|Architecture]]
- [[../fe_spec|Frontend Spec]]
</content>
</invoke>
