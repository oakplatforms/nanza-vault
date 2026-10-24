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
  and entity-slug routes. Live screens: `ShareDetail/` (the reference-code landing pages) and `ProfilePage/`
  (`nanza.app/<username>`). Dormant
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

**Routing:** flat route table in `AppLayout.tsx`. Static marketing paths out-rank the dynamic
`/:referenceCode` share route; anything unmatched `Navigate`s to `/`. 6-char reference codes
(`S/B/C/P/K/G/U` + five digits 2–9) are the core dynamic surface.

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

## Plans (in progress)

- [[solution-designs/landing|Landing — migrating the nanza-three-js front end]] — porting the
  immersive landing surface into `src/landing/`, additive except for the route table. Blocked on
  four decisions (colliding routes, the 138MB hero video, card-artwork location, transition-period
  look) and on routing escalation.

## Solution designs

- [[solution-designs/analytics|Analytics]] — GTM and GA4 from one module: capture-phase
  listeners for every click and key, `data-track` attributes where a label is dynamic, page types
  from the route registries, a consent banner on Consent Mode v2, attached bottom-up (no effects,
  no fetch hooks, toggled by env or GTM) except the two store redirects held for their hit, the
  importable container and the GA/GTM setup steps.
- [[solution-designs/auth|Auth]] — the ported mobile Cognito wrapper (session context, JWT-on-requests,
  sign-up/in/out + forgot-password screens); currently a guest-only stub with the auth routes
  commented out.
- [[solution-designs/compact-mode|Compact mode]] — what the public site does below
  1024px: one breakpoint for the copy columns, the 3D scene and the nav, the static
  phone that now shows real captures behind a scrim, and the touch details (hit areas,
  a menu that pins the page).
- [[solution-designs/content-pages|Content pages]] — the nav's topic pages: numbered sections that
  make a case and link to how-to guides, driven by one `hubs.ts` list for Create, Collect and
  Trade, and the `PAGE_BODIES` hookup.
- [[solution-designs/design-sync|Design sync]] — how screens reach Figma and how a Figma
  redesign reaches the code: the scripted capture (no screenshots taken by hand), the three
  axes of the board (viewport tier in the name, color mode in the variant, scroll waypoints
  for the landing page), and the manual Dev Mode MCP return path.
- [[solution-designs/get-the-app|Get the app]] — the `/get` address that resolves to the right
  store, the QR code that encodes it, and the homepage scan card: geometry matched to X's, a
  4% hover lift, and a dismissal remembered per browser.
- [[solution-designs/help-centre|Help centre]] — `/support` and the `/support/<slug>` how-to
  guides: text only, every guide in one data file written in the spec's plain format, labels
  derived from the bold text, and the journey / next-step / back-to navigation block.
- [[solution-designs/join-sheet|Join sheet]] — the phone web's once-a-session "Join the chat"
  bottom sheet: the app's "Ready to join in?" sheet on web tokens, the affiliate's own sheet
  on an affiliate's profile and a generic one elsewhere, opened by 10s on site or a screen of
  scrolling, reopened from an arrow tab, sent to the affiliate's Airbridge link or the store.
- [[solution-designs/landing-opening|Landing opening]] — the homepage hero: centred copy in the eye
  of a full-bleed card spiral that drains inward and becomes the 3D phone. Previous video opening
  archived as `IntroHeroOld.tsx` / `sectionsOld.ts`.
- [[solution-designs/profile-page|Profile page]] — `nanza.app/<username>`: the hero under a
  bleeding banner, the segmented Posts / Collections / Trades tabs, and the one thumb card
  shared by the Trades tab and embedded post items — mobile's visual language on the web.
- [[solution-designs/seo|SEO]]: per-page titles, descriptions, canonicals and JSON-LD from
  `useSeo`, with the `Name – Tab / Nanza` grammar. Not-found and error pages instead of
  redirecting home. Hand-kept robots.txt and sitemap. ogEdge untouched. Prerendering shelved in a
  git stash.
- [[solution-designs/sharing|Sharing]] — the reference-code landing pages, the app-store redirect for
  app-less mobile visitors, centralized store URLs + card headers, and the profile (`U`)
  `?type=sell|bid` share views.
