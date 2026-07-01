# nanza-admin — Architecture

The admin console (`oak-admin`) is a React 18 + TypeScript single-page app scaffolded with Create React App (`react-scripts` 5). It is the operations back office for the Nanza marketplace: staff sign in with Cognito and manage the catalog and order pipeline against the shared `nanza-api`. There is no server of its own — it is static assets on S3 talking to the API over HTTPS.

Back to [[INDEX|the project map]] · cross-project context in [[../_shared/INDEX|Shared brain]].

## Entry & bootstrapping

- `src/index.tsx` — creates the React root and renders `<App />`. Imports global CSS (`index.css`).
- `src/app/index.tsx` — the `App` component. At module load it calls `Amplify.configure(awsExports)` (from `src/aws-exports.js`) and creates a single `QueryClient`. It renders the provider stack:
  ```
  QueryClientProvider (React Query)
    └─ SessionProvider (auth/session)
        └─ AppLayout (routing + auth gate)
  ```
- `src/app/AppLayout.tsx` — the auth gate. Reads `isSignedIn` from `useSession()`:
  - `undefined` → render `null` (session still resolving; avoids a sign-in flash).
  - `false` → render `<SignIn />`.
  - `true` → render `<Layout navbar={<TopNavbar/>} sidebar={<SidebarNavigation/>}>` wrapping the router.

## Routing & pages

Routing uses `react-router-dom` v6 `BrowserRouter` declared inside `AppLayout`. Because it is a browser (HTML5 history) router served from S3, deep links depend on the bucket/CloudFront rewriting unknown paths to `index.html`.

Routes registered in `AppLayout`:
`/dashboard`, `/tags`, `/brands`, `/categories`, `/products`, `/shipping-methods`, `/shipping-options`, `/conditions`, `/sets`, `/sellers`, `/orders`.

Navigation surfaces are hand-maintained, separate from the route table:
- `src/components/TopNavbar.tsx` — the primary nav bar; `navigation` array of `{ name, to }` plus a `createMenuItems` "create" dropdown (deep-links like `/products?create=true` that pages read via `useSearchParams` to auto-open a create dialog). Includes a `Homepage` link that is not a registered route (stub).
- `src/components/Sidebar.tsx` — the mobile/off-canvas nav (`SidebarNavigation`) rendered inside `Layout`'s `MobileSidebar`.

### Page/feature shape

Each feature is a folder under `src/app/<Feature>/`:
- `index.tsx` — the page component (list + dialogs + pagination + search).
- `data/` — one React Query hook per query it needs (e.g. `Products/data/fetchProductEntities.ts`, `fetchBrands.ts`, `fetchTags.ts`, …).
- Optionally `CreateOrEditProduct.tsx`-style form components for the create/edit dialogs.

`src/app/Products/index.tsx` is the richest example: it composes `fetchProductEntities`, `fetchTags`, `fetchCategories`, `fetchBrands`, `fetchSets`, keeps local UI state for the selected entity / tag edits / dialog visibility, drives server-side search through `entityService`, and reads `currentUser` from the session for audit fields. Features with a `data/` folder: Products, Brands, ShippingMethods, ShippingOptions, Tags, Sets, Conditions, Orders, Categories, Sellers.

## SessionContext & auth (Cognito)

Auth is AWS Cognito accessed through Amplify Auth v6 (`aws-amplify` / `@aws-amplify/auth`). There is no custom auth server.

`src/context/SessionContext.tsx` owns session state and exposes `useSession()`:
- State: `isSignedIn?: boolean` (undefined = unknown), `setIsSignedIn`, `currentUser?: UserDto`, `setCurrentUser`.
- On mount, `getCurrentUser()` is attempted: success → `isSignedIn = true`; throw → `false`.
- When `isSignedIn` becomes true, it fetches the admin profile with `userService.get(userId, '?include=admin')` and stores it as `currentUser` (used for `createdById` / `lastModifiedById` audit fields on writes).
- `useSession()` throws if used outside `SessionProvider`.

`src/components/SignIn.tsx` renders the login screen and drives the Amplify challenge state machine:
- `signIn({ username, password })`; on success sets `isSignedIn`.
- Handles `CONFIRM_SIGN_IN_WITH_NEW_PASSWORD_REQUIRED` (force new password) via `confirmSignIn`.
- Handles `CONFIRM_SIGN_IN_WITH_EMAIL_CODE` / `CONFIRM_SIGN_IN_WITH_SMS_CODE` (MFA) via `confirmSignIn`.

Sign-out lives in `TopNavbar`/`Sidebar`: Amplify `signOut()`, then reset `isSignedIn` and `currentUser`.

### Cognito config

`src/aws-exports.js` holds the pool id / web client id (`aws_project_region`, `aws_cognito_region`, `aws_user_pools_id`, `aws_user_pools_web_client_id`). It is **generated, not hand-edited**:
- Local: `npm run setup` → `scripts/setup-local.sh` writes a dev-pointing `aws-exports.js`.
- CI: the deploy workflow regenerates it inline from GitHub Actions repo variables per environment (see Build/deploy).

## Data / services layer (talking to nanza-api)

All network I/O funnels through `fetchData` in `src/services/api/index.ts`:
- Signature `{ url, method = 'GET', payload }`.
- Pulls the Cognito **access token** via Amplify `fetchAuthSession()` and sets `Authorization: Bearer <token>` when present.
- Detects `FormData` payloads and omits `Content-Type` so the browser sets the multipart boundary (image uploads); otherwise `application/json` + `JSON.stringify`.
- Prefixes `process.env.REACT_APP_API_BASE_URL`.
- Maps HTTP 429 to a dedicated `RateLimitError`; other non-2xx throws `errorMessage` from the body (falling back to `HTTP Error: <status>`).

Per-resource service modules sit alongside it, one file per resource: `Brand.ts`, `BrandTag.ts`, `Category.ts`, `Condition.ts`, `Entity.ts`, `List.ts`, `Order.ts`, `Seller.ts`, `Set.ts`, `ShippingMethod.ts`, `ShippingOption.ts`, `SupportedTagValue.ts`, `Tag.ts`, `User.ts`. Each exports a `<resource>Service` object with `get / list / create / update / delete` methods (plus `uploadImage` / `deleteImage` on image-bearing resources such as `Brand`). URLs follow the API's singular/plural convention (`/brand/:id` for one, `/brands` for the list) and accept a raw query-string `parameters` argument for `?include=…`, pagination, and search.

The CRA dev server also sets a `proxy` in `package.json` to the API's API Gateway origin so local dev can call the API without CORS setup.

## React Query patterns

- One `QueryClient` created in `app/index.tsx` with library defaults (no global option overrides).
- **One hook per query**, colocated under the feature's `data/` folder. Convention (see `Products/data/fetchBrands.ts`):
  ```ts
  function useFetchBrands() {
    const query = useQuery<PaginatedResponse<BrandDto>>({
      queryKey: ['productBrands'],
      queryFn: () => brandService.list(''),
      staleTime: 1000 * 60 * 5,
    })
    return {
      brands: query.data,
      refetchBrands: query.refetch,
      isLoadingBrands: query.isLoading,
      errorBrands: query.error,
    }
  }
  export const fetchBrands = useFetchBrands
  ```
- Each hook owns its `queryKey` (string-array), a `queryFn` delegating to a service, and a `staleTime` (commonly 5 minutes). Return fields are renamed per-resource so a page can pull several hooks without collisions.
- Reads dominate; writes are done by calling the service `create/update/delete` directly and then `refetch`-ing the relevant query rather than via `useMutation` cache invalidation. Server-side search (e.g. Products) bypasses React Query and calls the service imperatively, holding results in local `useState`.

## Types

`src/types/index.ts` re-exports DTOs from `@oakplatforms/types` (the shared cross-repo package) — `UserDto`, `BrandDto`, `EntityDto`, `OrderDto`, `PaginatedResponse`, etc. — and layers admin-only additions on top:
- Extended DTOs where the admin needs extra fields (`ShippingMethodDto`, `SetDto`, `SellerDto`, `BrandTagDto`, `SupportedTagValuesDto`).
- `*Payload` write shapes (`EntityPayload`, `BrandPayload`, `TagPayload`, `SetPayload`, `ShippingMethodPayload`, …) that model the API's nested `create/update/delete` write envelopes and audit fields (`createdById`, `lastModifiedById`).

Domain shapes come from `@oakplatforms/types`, not hand-rolled interfaces.

## Styling (Tailwind + Headless UI)

- Tailwind CSS 3 configured in `tailwind.config.js`, scanning `./src/components/**` and `./src/app/**`. `theme.extend.fontFamily` adds `sans: Plus Jakarta Sans` and `figtree: Figtree`. Processed by PostCSS + autoprefixer (`postcss.config.js`), entered via `src/index.css` / `src/styles/globals.css`.
- Utility-first: classes live inline in JSX (no theme-token / stylesheet-bucket abstraction like `nanza-mobile`). Conditional classes use `clsx`.
- `src/components/Tailwind/` is a Catalyst-style UI kit (~24 primitives: `Button`, `Input`, `Table`, `Dialog`, `Dropdown`, `Select`, `Listbox`, `Badge`, `Switch`, `RichTextEditor`, `Pagination`, …) built on `@headlessui/react` for accessible behavior, `@heroicons/react` for icons, and `forwardRef` for composition; all re-exported from `Tailwind/index.ts`. App-level composites (`SimpleTable`, `PaginationControls`, `SearchInput`, `ConfirmDialog`, `SimpleDialog`) wrap these.
- Rich text uses `quill` (v2); animations use `framer-motion`.

## Build & deploy (S3 + GitHub Actions OIDC)

Deployment is static hosting on S3, driven by GitHub Actions — no access keys, OIDC-assumed roles (per the app's `CLAUDE.md`).

- Workflows: `.github/workflows/deploy-dev.yml` (push to `dev`) and `.github/workflows/deploy-prod.yml` (push to `prod`). A `stage` branch/environment follows the same pattern.
- Each job: checkout → Node 20 with the `@oakplatforms` GitHub Packages registry (for `@oakplatforms/types`) → `npm ci` (`--legacy-peer-deps` on dev) → **assume AWS role via OIDC** (`aws-actions/configure-aws-credentials@v4`, `role-to-assume: vars.AWS_ROLE_ARN_<ENV>`, `id-token: write`).
- **Cognito config is generated in-pipeline**: an inline step writes `src/aws-exports.js` from `vars.ADMIN_USER_POOL_ID_<ENV>` / `vars.ADMIN_USER_POOL_CLIENT_ID_<ENV>` (avoids `amplify pull` under OIDC).
- Build: `npm run build` (CRA) with env `REACT_APP_API_BASE_URL` and `REACT_APP_S3_IMAGE_BASE_URL` from per-environment repo variables.
- Publish: `aws s3 sync build/ s3://<vars.S3_ADMIN_BUCKET_<ENV>> --delete`.

Config values (`API_BASE_URL`, `S3_IMAGE_BASE_URL`, pool ids, bucket, role ARN) are GitHub Actions **repo variables**, suffixed by environment. Secret values are never committed; `aws-exports.js` and `.env` are regenerated per build.

## Notable characteristics / gotchas

- The app still labels itself "TCGx 0.1" / "TCGX" in `Sidebar` and `SignIn`; the package is `oak-admin`. Branding predates the Nanza name.
- Two nav definitions (`TopNavbar`, `Sidebar`) are maintained separately from the `AppLayout` route table, so links can drift ahead of routes (e.g. `/homepage`).
- Writes refetch queries rather than invalidating the React Query cache; there is little use of `useMutation`.
- `staleTime` is set per hook, but there is no shared `QueryClient` default config.

## Related

- [[INDEX|nanza-admin map]]
- [[../_shared/INDEX|Shared brain]] — Cognito, data model, and infra decisions that span repos.
