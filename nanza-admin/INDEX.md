# nanza-admin

The internal admin console for the Nanza marketplace — a React 18 single-page app (Create React App + TypeScript) used by the Nanza team to manage the catalog and marketplace back office: brands, categories, tags, sets, products (entities), conditions, shipping methods/options, sellers, and orders. It authenticates admins through AWS Cognito (via Amplify Auth), reads and writes through the shared `nanza-api`, and is styled entirely with Tailwind + Headless UI. The package name is `oak-admin` and the UI still self-identifies as "TCGx"/"TCGX" in a few places.

## Structural overview

Source lives entirely under `src/` (CRA convention). Entry is `src/index.tsx` → `src/app/index.tsx` (`App`), which wires the two global providers and Amplify.

- **App shell & providers** — `src/app/index.tsx` mounts `QueryClientProvider` (React Query) around `SessionProvider` around `AppLayout`, and calls `Amplify.configure(awsExports)` once at module load. `src/app/AppLayout.tsx` is the auth gate: it renders `null` while the session is unknown, `<SignIn />` when signed out, and the routed `<Layout>` when signed in.
- **Routing / pages** — `react-router-dom` v6 `BrowserRouter` inside `AppLayout`. Each feature is a folder under `src/app/<Feature>/` with an `index.tsx` page and a `data/` folder of React Query hooks. Live routes: Dashboard, Tags, Brands, Categories, Products, ShippingMethods, ShippingOptions, Conditions, Sets, Sellers, Orders. Navigation is defined in `TopNavbar` and `Sidebar` (some links, e.g. `/homepage`, are stubs).
- **Components** — `src/components/` holds app-level pieces (`Layout`, `Sidebar`, `TopNavbar`, `SignIn`, `SimpleTable`, `PaginationControls`, `SearchInput`, dialogs). `src/components/Tailwind/` is a ~24-file Catalyst-style UI kit (Button, Input, Table, Dialog, Dropdown, Select, Badge, RichTextEditor, etc.) built on `@headlessui/react`, `clsx`, and `forwardRef`, re-exported from `Tailwind/index.ts`.
- **SessionContext & auth (Cognito)** — `src/context/SessionContext.tsx` exposes `useSession()` with `isSignedIn` and `currentUser`. On mount it calls Amplify's `getCurrentUser()` to derive `isSignedIn`, then loads the admin user via `userService.get(userId, '?include=admin')`. `SignIn.tsx` drives the Amplify `signIn` / `confirmSignIn` flow including new-password and email/SMS MFA challenges. Cognito pool IDs come from `src/aws-exports.js` (generated, never hand-edited).
- **React Query data layer** — one hook per query in `src/app/<Feature>/data/` (e.g. `useFetchBrands`), each defining its own `queryKey`, `queryFn` calling a service, and `staleTime`. Hooks return renamed fields (`brands`, `refetchBrands`, `isLoadingBrands`, `errorBrands`). The `QueryClient` is created with defaults in `app/index.tsx`.
- **Services / API** — `src/services/api/` has one module per resource (`Brand.ts`, `Entity.ts`, `Order.ts`, `Seller.ts`, `Set.ts`, `Tag.ts`, `User.ts`, …) exposing `get/list/create/update/delete` (plus image upload/delete where relevant). All go through `fetchData` in `src/services/api/index.ts`, which attaches the Cognito access-token bearer header and prefixes `REACT_APP_API_BASE_URL`.
- **Types** — `src/types/index.ts` re-exports DTOs from `@oakplatforms/types` and adds admin-only `*Payload` shapes and a few extended DTOs. No hand-rolled domain types.
- **Styling** — Tailwind CSS 3 (`tailwind.config.js` scans `src/components` and `src/app`), with Plus Jakarta Sans / Figtree fonts, PostCSS + autoprefixer, and Prettier's Tailwind class-sorter. Utility-first classes inline in JSX; no theme-token abstraction like the mobile app.

## Plans

No plans yet. New plans land in `plans/`.

## Related

- [[architecture|Architecture]] — entry & routing, Cognito auth flow, the `fetchData` service layer, React Query patterns, and S3 + GitHub Actions (OIDC) deploy.
- [[../_shared/INDEX|Shared brain]] — cross-project architecture, decisions, and capabilities catalog.

> Keep this file a short map that links out.
