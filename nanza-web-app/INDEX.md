# nanza-web-app

The public web app for Nanza — a **Create React App** (react-scripts 5) single-page app in
TypeScript, using React Router v6, TanStack Query, Tailwind CSS, and Framer Motion. Source lives
entirely in `src/`. It talks to **nanza-api** through a small `fetchData` wrapper and shares its
Cognito auth wrapper and design tokens with **nanza-mobile** and **nanza-admin**.

In its **current live shape** the app is a **public distribution surface for shareable links** plus
a marketing landing site. Sign-in, cart/checkout, and the public storefront/profile experience are
built but **intentionally commented out** in routing (`src/app/AppLayout.tsx`) and Cognito is
**stubbed to guest-only mode** (`src/services/auth/cognito.ts`) — every request rides a cached guest
token. The auth screens, `SessionProvider`, service layer, and env-var keys are all still present so
the marketplace can be re-enabled without rebuilding it.

## Structural overview

Top-level dirs under `src/`:

- **`app/`** — routed screens and the router. `app/index.tsx` mounts the provider stack
  (`QueryClientProvider → SessionProvider → CartProvider → AppLayout`); `app/AppLayout.tsx` holds all
  `react-router-dom` routes; `app/SmartRouter.tsx` disambiguates `/:a/:b` between profile-reference
  and entity-slug routes. Live screens: `ShareDetail/` (the reference-code landing pages). Dormant
  (commented-out) screens: `Auth/` (SignIn/SignUp/ForgotPassword), `Cart/`, `Build/` (storefront
  builder), `UserProfile/`, `UserListings/`, `UserBids/`, `UserLists/`, `UserSaved/`,
  `EntityDetail/`, `ListingDetail/`, `UniversalDetail/`.
- **`landing/`** — marketing site: `landing/index.tsx` + `landing/pages/` (About, Buy, Sell, Collect,
  Pricing, Privacy, Terms, ContentPage) and `landing/components/` section blocks.
- **`components/`** — shared UI: layout shells (`Layout`, `AppShell`, `DetailLayout`, `TopNav`,
  `Sidebar/`, `Toolbar/`), `Tailwind/` primitives, and per-domain folders (`entity`, `listing`,
  `bid`, `list`, `cart`).
- **`services/api/`** — one file per resource (`Listing`, `Bid`, `Entity`, `List`, `Order`, `User`,
  `Storefront`, `Universal`, …), all funneling through `services/api/index.ts`'s `fetchData`.
  `services/auth/cognito.ts` is the auth wrapper (currently no-op stubs).
- **`contexts/`** — `SessionProvider` (current user via Cognito session → `userService`),
  `CartProvider`, `ProfileProvider`.
- **`types/index.ts`** — re-exports DTOs from `@oakplatforms/types` (no local domain types).
- **`helpers/`, `utils/`, `constants.ts`** — slugify, price formatting, cart math, store URLs,
  brand/condition config.
- **Styling** — Tailwind (`tailwind.config.js`) plus CSS custom properties for the theme scale in
  `src/index.css` / `src/styles/globals.css` (EuclidCircularB primary, Geist/Figtree secondary),
  the same token palette used by mobile/admin.

**Routing:** flat route table in `AppLayout.tsx`. Sharing lives at **nine root-level type-scoped
routes** `/<slug>/:referenceCode` (listing, bid, collection, product, bulk, group, profile, post,
tag) → `ShareDetail` (share-route-migration, 2026-08 — the slug names the record type; older
shapes, bare root codes and the interim `/share` prefix, are not honored, no redirect); anything
unmatched `Navigate`s to `/`. These are the core dynamic surface; the reference code is an opaque
token (lenient 4–12 alphanumeric check — still minted as 6 chars, five digits 2–9 + a type
letter, but routing never decodes the letter).

**Data/services → nanza-api:** all reads/writes go through `fetchData`, which attaches a Cognito JWT
when signed in and otherwise a cached guest token. Base URL from `REACT_APP_API_BASE_URL`
(dev proxy `https://api-dev.nanza.app`). TanStack Query handles caching/retry.

**Auth (Cognito):** `amazon-cognito-identity-js` against the shared dev user pool, wrapped in
`services/auth/cognito.ts` — the same function-per-operation shape as nanza-mobile. Live build ships
the guest-only stub; re-enabling restores the real wrapper + the commented auth routes.

**Build/deploy:** `npm run build` (CRA) → `aws s3 sync` to an S3 bucket, fronted by CloudFront, via
GitHub Actions (`.github/workflows/deploy-dev.yml`, `deploy-prod.yml`) using OIDC role assumption.
Universal Links / App Links assets ship under `public/.well-known/`.

## Deeper docs

- [[architecture|Architecture]] — entry, routing, services, auth flow, styling, build/deploy
- [[fe_spec|Frontend Spec]] — engineering standards, non-negotiables, escalation process
- [[../_shared/INDEX|Shared brain]] — cross-project architecture & decisions

## Strategy

- [[nanza-founder-skylar|Founder Strategy Questionnaire — Skylar]] — 136-question strategy capture
  (origin, customer, product vision, moat, licensors, model, growth, risk, philosophy) in a consistent
  first-person voice; all questions answered, session follow-ups folded in.

## Plans (in progress)

- [[solution-designs/share-route-migration|Share Route Migration]] — move the public share URL to
  root-level type-scoped routes `/<typeSlug>/<code>` (listing/bid/collection/…) across web,
  API/edge, and mobile; the slug picks the table, the code is an opaque token; clean cut, no
  redirects (breaking change accepted).

## Solution designs

- [[solution-designs/auth|Auth]] — the ported mobile Cognito wrapper (session context, JWT-on-requests,
  sign-up/in/out + forgot-password screens); currently a guest-only stub with the auth routes
  commented out.
- [[solution-designs/sharing|Sharing]] — the reference-code landing pages, the app-store redirect for
  app-less mobile visitors, centralized store URLs + card headers, and the profile
  (`/profile/<code>?type=sell|bid`) share views.
