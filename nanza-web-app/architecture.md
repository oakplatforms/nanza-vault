# nanza-web-app — Architecture

Deeper architecture notes for the Nanza web app. See [[INDEX]] for the map and
[[../_shared/INDEX|Shared brain]] for cross-repo decisions (auth, data model, shareability).

## What it is

A **Create React App** SPA (`react-scripts` 5.0, React 18, TypeScript 5) — confirmed by
`package.json` (`"react-scripts": "5.0.0"`, `start`/`build`/`eject` scripts) and `README.md`
(CRA boilerplate). Not Vite, not Next. Bundles: `react-router-dom` v6 (routing),
`@tanstack/react-query` v5 (server state), `tailwindcss` v3 (styling), `framer-motion` + `gsap` +
`lenis` (animation/scroll), `@oakplatforms/types` (shared DTOs), `amazon-cognito-identity-js` (auth),
and `@stripe/*` (checkout — currently dormant).

All application code is under `src/`. `tsconfig.json` runs `strict` with `noEmit` (CRA/Babel does the
transpile). ESLint is flat-config (`eslint.config.mjs`) extending `react-app`.

## Entry and provider stack

`src/index.tsx` mounts `<App />` (from `src/app/index.tsx`) into `#root`. `App` wraps everything in:

```
QueryClientProvider          # src/app/index.tsx — retry: no 4xx retry, 2x on 5xx; staleTime 5m
  └─ SessionProvider         # src/contexts/SessionProvider.tsx
       └─ CartProvider       # src/contexts/CartProvider.tsx
            └─ AppLayout      # src/app/AppLayout.tsx — the router
```

The `QueryClient` default options live inline in `app/index.tsx`: queries do **not** retry on 4xx
(reads `error.status`), retry up to twice on network/5xx, and cache for 5 minutes.

## Routing

`src/app/AppLayout.tsx` is a single `<BrowserRouter>` with a flat `<Routes>` table, wrapped in
`BrandFilterProvider` and `LayoutAnimationProvider`. Route groups:

- **Marketing / landing** — `/` (`LandingPage`), `/about`, `/buy`, `/sell`, `/collect`, `/pricing`,
  `/privacy-policy`, `/terms`, plus many `ContentPage` slugs (`/how-it-works/*`, per-TCG game pages
  like `/pokemon`, `/magic-the-gathering`, legal pages). These are static and out-rank the dynamic
  share route by React Router v6 specificity ranking.
- **Share landing** — `/:referenceCode` → `app/ShareDetail`. This is the **only live dynamic route**
  and the core purpose of the deployed app.
- **Catch-all** — `path="*"` → `<Navigate to="/" replace />`. Any bare username or unknown path
  bounces home (there is no public profile in the live build).
- **Dormant, commented-out** — auth (`/sign-in`, `/sign-up`, `/forgot-password`), cart
  (`/cart`, `/cart/confirmation`), storefront builder (`/build`, `/build/:storefrontId`),
  profile/user routes (`/:username`, `/:username/listings|bids|lists`), and the `SmartRouter`
  disambiguation route (`/:param1/:param2`). These are preserved as comment blocks, not deleted, so
  the marketplace can be restored.

### Reference codes

The share surface is driven by **6-character reference codes**: exactly five digits (2–9) and one
type letter, validated by `isReferenceCode()` (duplicated in `ShareDetail/index.tsx` and
`SmartRouter.tsx`, mirroring the backend contract). Type letters:

| Letter | Meaning | ShareDetail card |
|--------|---------|------------------|
| `S` | Listing | `FeedDetailCard` |
| `B` | Bid | `FeedDetailCard` |
| `P` | Product | `ProductDetailCard` |
| `C` | List / collection | `ListDetailCard` |
| `K` | Bulk listing | `BulkDetailCard` |
| `G` | Group | `GroupDetailCard` |
| `U` | Profile (`?type=sell\|bid`) | `ProfileDetailCard` |

`app/ShareDetail/index.tsx` fetches by code (`data/fetchByCode.ts`), maps the record into card props,
and renders inside `ShareDetailShell`; invalid codes `Navigate` to `/`. `SmartRouter.tsx` (dormant)
uses the same code check to split `/:a/:b` into a profile-reference view vs. a brand/entity slug view.

## Data / services layer (→ nanza-api)

Every network call funnels through **`src/services/api/index.ts` → `fetchData({ url, method, payload })`**:

- Resolves an auth token via `getJwtToken()`: tries `fetchAuthSession()` (Cognito) first; if there's
  no signed-in session it falls back to `getGuestToken()`.
- **Guest token** is cached in-module with an expiry; sourced from `REACT_APP_GUEST_TOKEN` or fetched
  from `REACT_APP_GUEST_TOKEN_ENDPOINT` (default `/user/guest-token`). This is what keeps the app
  functional with auth disabled — every read rides a guest token.
- Prepends `REACT_APP_API_BASE_URL`; throws a configured error if unset.
- `fetch` with `mode: 'cors'`, `credentials: 'omit'`, `Authorization: Bearer <token>`. Handles
  FormData (skips `Content-Type`), maps `429 → RateLimitError`, attaches `.status`/`.isCorsError` to
  thrown errors so React Query's retry predicate can read them.

`services/api/` has ~16 resource modules — one per domain object (`Listing.ts`, `Bid.ts`, `Entity.ts`,
`EntityList.ts`, `List.ts`, `Order.ts`, `Invoice.ts`, `Customer.ts`, `Profile.ts`, `User.ts`,
`Brand.ts`, `Storefront.ts`, `Universal.ts`, `ShippingMethod.ts`, `Auth.ts`). Each exposes a small
object of async methods that call `fetchData` with the REST path (e.g. `userService.get(id, params)`
→ `GET /user/{id}{params}`; `authService.checkForUser(email)` → `GET /auth/check-for-user?email=`).
The dev proxy (`"proxy": "https://api-dev.nanza.app"` in `package.json`) routes local calls to the
dev API.

React Query hooks live next to their screens (e.g. `app/Build/data/useStorefronts.ts`,
`app/BidDetail/data/fetchBid.ts`, `contexts/data/fetchCartOrders.ts`), keyed by resource and unwrapping
the API's `{ data, page, total }` pagination envelope where present.

## Auth flow (Cognito)

**Design (shared with mobile/admin):** the same `amazon-cognito-identity-js` wrapper shape from
`nanza-mobile/src/services/auth/cognito.ts` — a clean function-per-operation API (`signUp`,
`confirmSignUp`, `resendSignUpCode`, `signIn`, `signOut`, `resetPassword`, `confirmResetPassword`,
`updatePassword`, `getCurrentUser`, `fetchAuthSession`) against the **same dev user pool**. The
package is platform-agnostic; the web uses `window.localStorage` as its token store natively (no
AsyncStorage adapter needed). See [[plans/web-cognito-auth|Add Cognito Auth]].

**Current live state:** `src/services/auth/cognito.ts` is a deliberate **no-op stub** — auth is
disabled. Every mutating export throws `AUTH_DISABLED_MESSAGE`; `signOut` is a no-op; and
`fetchAuthSession()` returns `{ tokens: undefined, userSub: undefined }`. That empty session is what
makes `fetchData` fall through to the guest token, so the app runs guest-only at all times.

**Session consumption:** `SessionProvider` calls `fetchAuthSession()` on mount to derive `isSignedIn`,
and when signed in loads the current user via `getCurrentUser()` → `userService.get(userId, include=…)`
(pulling `account.profile/seller/customer/carts/lists`). It exposes `{ isSignedIn, currentUser,
refreshCurrentUser, signOut }` through `useSession()`. With the stub in place `isSignedIn` stays
`false` and `currentUser` stays `undefined`. Auth screens (`app/Auth/SignIn|SignUp|ForgotPassword`)
exist but their routes are commented out in `AppLayout.tsx`.

Env-var keys for Cognito are declared in `.env` (`REACT_APP_COGNITO_USER_POOL_ID`,
`REACT_APP_COGNITO_USER_POOL_CLIENT_ID`) and passed at build time in the dev workflow — values are set
in GitHub Actions vars, never committed. Re-enabling = restore the real wrapper from git history +
uncomment the auth routes.

## Styling

Tailwind CSS drives layout/utility styling. `tailwind.config.js` scans `src/{components,app,landing}`
and extends the theme with the brand font families (`sans` → Euclid Circular B, `figtree` → Figtree)
and a `fadeIn` animation. The full color scale and semantic tokens are defined as CSS custom
properties in `src/styles/globals.css` and `src/index.css` (`--color-primary-*`, font-family vars),
with `@font-face` declarations for Euclid Circular B, Geist, and Figtree — the **same token palette
and font stack shared with nanza-mobile and nanza-admin**. Component styling is Tailwind class names
(with `clsx` for conditionals), not a StyleSheet abstraction as in mobile.

## Build and deploy

CRA build (`npm run build`) emits a static bundle to `build/`, deployed by GitHub Actions:

- **`.github/workflows/deploy-dev.yml`** — triggers on push to `dev`. Node 20, `npm ci
  --legacy-peer-deps` (auth to GitHub Packages for `@oakplatforms/types`), assumes an AWS role via
  **OIDC** (`aws-actions/configure-aws-credentials`, no static keys), builds with `REACT_APP_*` env
  from GitHub vars (API base URL, S3 image base, Cognito pool/client, Stripe key), then
  `aws s3 sync build/ s3://<S3_WEBAPP_BUCKET_DEV> --delete`. A follow-up step re-uploads
  `.well-known/apple-app-site-association` with `Content-Type: application/json`.
- **`.github/workflows/deploy-prod.yml`** — triggers on push to `prod`. Same shape (build env omits
  Cognito/Stripe vars, matching the disabled state), syncs to the prod bucket, fixes the AASA
  content-type, and additionally runs a **CloudFront invalidation** (`create-invalidation … /*`).

**Hosting:** static site in **S3 behind CloudFront**. **Deep-link assets** ship from
`public/.well-known/`: `apple-app-site-association` (iOS Universal Links, matching `/??????` — the
6-char codes, appID `Y9X5CD9HAV.com.oakplatforms.nanza`) and `assetlinks.json` (Android App Links,
`com.oakplatforms.nanza`). With those in place, users who already have the native app never see the
web page — the OS opens the app; the [[plans/share-store-redirect|store redirect]] only fires for
visitors without the app. Store URLs are centralized in `src/constants.ts` (`APP_STORE_URL`,
`PLAY_STORE_URL`).

## Related

- [[fe_spec|Frontend Spec]] — engineering standards, confidence scoring, non-negotiables
- [[../_shared/INDEX|Shared brain]] — cross-repo auth/data-model/infra decisions
- [[../_shared/glossary|Shared glossary]]
