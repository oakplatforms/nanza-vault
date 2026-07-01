# Plan: Add Cognito Auth to nanza-web-app

## Goal

Bring the same Cognito setup we just built for `nanza-mobile` to `nanza-web-app`. The web app currently has **zero authentication** — every request goes through a cached guest token. We want to add:

1. **Sign up** (email + password, with email verification code)
2. **Sign in / sign out**
3. **Forgot password** (request reset code → confirm with new password)
4. **An authenticated session** the rest of the app can consume
5. **Authenticated API requests** (real JWT instead of guest token when signed in)

Out of scope per user instruction: full marketplace flows. **No Stripe / payment card** work this round — auth only. (User mentioned cart for a future pass but explicitly said don't worry about Stripe now.)

---

## Why this matters / context

- Mobile just migrated off `aws-amplify` to `amazon-cognito-identity-js`. The wrapper lives at `nanza-mobile/src/services/auth/cognito.ts` and exports a clean function-per-operation API (`signIn`, `signUp`, `confirmSignUp`, `resendSignUpCode`, `signOut`, `resetPassword`, `confirmResetPassword`, `updatePassword`, `getCurrentUser`, `fetchAuthSession`).
- That wrapper works just as well in a browser as in React Native — `amazon-cognito-identity-js` is platform-agnostic. The only platform difference is the storage adapter: RN needs an `AsyncStorage` wrapper; the web can use `window.localStorage` natively, which is what the package defaults to.
- The same Cognito User Pool (`us-east-1_BcRP6qegV`) should be used. Web App Client ID is the open question (see below).

**Confidence: 0.95** — Reusing the same Cognito pool and the same package is the right call. Dropping in a near-identical wrapper means the auth implementation stays consistent across mobile and web, and there's only one mental model to maintain. Rejected alternative: federated identity / Cognito Hosted UI redirect flow. Rejected because it's heavier, requires hosting config, and we want a self-hosted form-based UX that matches the mobile look-and-feel.

---

## Inventory of what doesn't exist yet

| Surface | Status |
|---|---|
| Auth wrapper module | Missing — must create `src/services/auth/cognito.ts` |
| Session context | Missing — must create `src/contexts/SessionProvider.tsx` |
| Sign up screens | Missing — must create `src/app/Auth/SignUp/` |
| Sign in screen | Missing — must create `src/app/Auth/SignIn/` |
| Forgot password screens | Missing — must create `src/app/Auth/ForgotPassword/` |
| Header sign-in / sign-out UI | Need to check `Navbar`/`Header` components |
| API JWT integration | `src/services/api/index.ts` only handles guest token; need to prefer Cognito access token when signed in |
| Auth routes | Need to add routes in `src/app/AppLayout.tsx` (the spec says routing is non-negotiable — this is an additive route only) |

**Escalation:** Adding routes touches `src/app/AppLayout.tsx`. The fe_spec.md flags routing changes as escalation-worthy. **My read:** the rule is "don't change route order/params" and "don't break existing routes." Adding new additive routes (`/sign-in`, `/sign-up`, `/forgot-password`) is necessary for this feature and doesn't modify any existing route. **Confidence: 0.85** that this is within the spirit of the rule, but I'm flagging it — confirm before I add the routes.

---

## Architecture

### Module 1: `src/services/auth/cognito.ts`

Direct port of the mobile wrapper, **without** the AsyncStorage adapter (use default `window.localStorage`). Same exports, same call signatures:

```ts
export function signUp(input): Promise<{ userId: string }>
export function confirmSignUp(input): Promise<void>
export function resendSignUpCode(input): Promise<void>
export function signIn(input): Promise<{ isSignedIn: boolean }>
export function signOut(): Promise<void>
export function resetPassword(input): Promise<void>
export function confirmResetPassword(input): Promise<void>
export function updatePassword(input): Promise<void>
export function getCurrentUser(): Promise<{ userId: string; username: string }>
export function fetchAuthSession(): Promise<{ tokens: { accessToken: { toString(): string } } | undefined; userSub: string | undefined }>
```

Pool config comes from `process.env.REACT_APP_COGNITO_USER_POOL_ID` and `process.env.REACT_APP_COGNITO_USER_POOL_CLIENT_ID` to match the existing CRA env-var pattern.

**Confidence: 0.95** — Already proven the API shape on mobile. Single-file port.

### Module 2: `src/contexts/SessionProvider.tsx`

Mirrors the mobile `SessionContext` but trimmed to web needs. Exposes:

```ts
{
  isSignedIn: boolean | undefined,   // undefined = still checking
  currentUser: UserDto | undefined,
  setIsSignedIn, setCurrentUser,
  refreshCurrentUser,
  signOut,                            // wraps the wrapper + clears state
}
```

On mount: call `fetchAuthSession()` to determine signed-in state. When signed in, call `userService.get(userId, ...)` to load `UserDto` from the API.

**Confidence: 0.9** — Standard auth context pattern. Mirrors existing `ProfileProvider` style.

### Module 3: Update `src/services/api/index.ts`

Currently `getJwtToken()` always returns a guest token. Change it to prefer Cognito's access token when a session exists:

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

**Confidence: 0.95** — Same pattern as mobile. Strictly additive — existing guest-token fallback stays.

### Module 4: Auth screens

Use the existing Tailwind/Catalyst components (`Button`, `Input`, `Dialog`, `Heading`, `Text`). Keep visual style consistent with the rest of the app. Each step is a screen, mirroring mobile's flow:

- `src/app/Auth/SignUp/index.tsx` — email + password form → calls `signUp` → moves to confirmation
- `src/app/Auth/SignUp/ConfirmationCode.tsx` — 6-digit code → calls `confirmSignUp` then `signIn`
- `src/app/Auth/SignIn/index.tsx` — email + password → `signIn` → redirects home
- `src/app/Auth/ForgotPassword/index.tsx` — three-stage form (email → code → new password) on one route, or three sub-routes — preference question below

**Confidence: 0.85** for the screen structure. Rejected alternative: build it as a modal like mobile. Rejected because mobile uses a modal pattern that doesn't fit the web app's full-page navigation idiom. The fe_spec specifies feature screens under `src/app/`.

### Module 5: Header / nav surface

The header (`src/components/Tailwind/Header.tsx` or wherever sign-in lives) needs a sign-in button when signed out and a user menu / sign-out option when signed in. **Need to inspect the current header before plumbing this** — I haven't read those files yet.

**Confidence: 0.7** — Can't pin this down without reading `Header.tsx` and seeing how the navbar is composed. Will check before implementing.

### Module 6: Mount the provider

Wrap `AppLayout` (inside `QueryClientProvider`) with `<SessionProvider>` in `src/app/index.tsx`. Single-line additive change.

**Confidence: 0.95** — Same shape as mobile.

---

## Env vars

Add to `.env` / docs:
```
REACT_APP_COGNITO_USER_POOL_ID=us-east-1_BcRP6qegV
REACT_APP_COGNITO_USER_POOL_CLIENT_ID=???   # OPEN QUESTION
```

---

## File count

Estimated: ~8–10 new files + 3 modified (`src/services/api/index.ts`, `src/app/index.tsx`, `src/app/AppLayout.tsx`). The fe_spec flags >5 files as escalation-worthy — flagging now.

---

## Answers (decided)

1. **Cognito App Client ID** — Reuse mobile's (`7776n77dtpfth7bv5m7io1e2n4`) for now. Leave a `# TODO:` comment in `.env` (not in code) noting we should create a dedicated web client later.
2. **Forgot password** — Single page with progressive steps. Matches mobile's flow.
3. **Auth UI placement** — In the **web app shell only** (the `/:username` routes via the "modern" Layout variant). Not on the landing page. Add a sign-in / avatar pill to the left `Toolbar` (it currently only has a gear icon at the bottom — add an auth control above it).
4. **Post-sign-in redirect** — Stay on current page. Header avatar circle becomes visible.
5. **Additive routes in `AppLayout.tsx`** — Approved.
