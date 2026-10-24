---
tags: [nanza-web-app, nanza-web-app-update, solution-design, landing, marketing, plan]
status: PLAN — implemented in the `landing-replacement` branch of nanza-web-app (off origin/dev), pending review/deploy (2026-08-27)
---

# Landing Replacement — drop `nanza-web-app-update` in as the public site (Plan)

## Goal

Replace the whole marketing surface of **nanza-web-app** — `src/landing/` (the static landing page,
its 23 section components, the `LandingLayout` nav/footer shell, and the thin content pages) — with
the front end built in **nanza-web-app-update**: a scroll-driven landing (Lenis + GSAP timeline,
React-Three-Fiber phone scene, hero video) plus a generated set of marketing pages.

The old landing and its components are **deprecated and deleted**, not kept alongside. The live
product surface — the type-scoped share routes `/<slug>/:referenceCode` → `ShareDetail`, the provider
stack, services, dormant auth/cart/storefront — is **untouched**.

**Decisions (Skylar, 2026-08-27, voice):**
- The two repos stay separate for now; the update repo's front end becomes the landing inside
  nanza-web-app; the existing landing + components are deprecated.
- **Fresh and clean**: every old-only marketing route and its static files go (`/pricing`,
  `/how-it-works/*` + `PayoutCalculator`, `/contact-nanza`, `/sell-on-nanza`,
  `/transaction-integrity`, the 8 TCG pages, `/terms-of-use`, `/legal`). Re-add later under
  `pages.ts` if a real page is needed. Only real legal copy (Privacy, Terms) is carried over.
- **The React app is untouched**: share routes/`ShareDetail`, providers, services, and the dormant
  auth/cart/storefront code are not part of this change.

## What's moving, in one picture

```
nanza-web-app-update/src            →   nanza-web-app/src
├── app/AppLayout.tsx               →   (merged INTO existing app/AppLayout.tsx — not copied)
├── app/usePublicSite.ts            →   app/usePublicSite.ts
├── landing/index.tsx               →   landing/index.tsx         (replaces the old one)
├── pages/{pages.ts,PageLayout.tsx} →   pages/                    (new dir)
├── sections/* (17 files)           →   sections/                 (new dir)
├── scene/*    (R3F + timeline)     →   scene/                    (new dir)
├── assets/cards/* (39 webp+manifest)→  assets/cards/             (src/assets exists; no name clash)
├── styles/public-site.css          →   styles/public-site.css
├── styles/tailwind.css             ✗   NOT copied — target's index.css already has @tailwind
├── theme.ts, icons.tsx             →   theme.ts, icons.tsx
├── vite-env.d.ts                   ✗   NOT copied (Vite-only)
└── App.tsx / main.tsx              ✗   NOT copied — target's app/index.tsx + index.tsx stay

nanza-web-app-update/public         →   nanza-web-app/public
├── collecting-is-an-art-rc1.mp4    →   (12 MB hero video)
├── bulk-cards.png, icons/, logos   →
└── fonts/Roobert-*.otf             ✗   already present in target public/fonts — skip
```

The update repo already did the bundler-portability work in commit `89ee9e9` ("Prepare the front
end to drop into nanza-web-app"): static card manifest instead of `import.meta.glob`, styles scoped
under `html.public-site`, `usePublicSite()` called from the marketing surfaces (not the router) so it
cannot leak onto share/cart/detail screens, no `import.meta.env`/`process.env` in shared code, and
the marketing "Post" page at `/create` to stay out of the `post/:referenceCode` share namespace.

## Design

### 1. Routing — one flat table, marketing routes generated

`src/app/AppLayout.tsx` keeps its shape (`BrowserRouter → BrandFilterProvider →
LayoutAnimationProvider → flat Routes`). The marketing block is replaced:

```tsx
<Route path="/" element={<LandingPage />} />
{[...PAGES, ...FOOTER_PAGES].map((page) => (
  <Route key={page.path} path={page.path}
         element={<PageLayout title={page.title} intro={page.intro} />} />
))}
{/* real-copy pages, rendered inside the new shell — see §5 */}
<Route path="/privacy-policy" element={<PageLayout title="Privacy Policy"><PrivacyCopy /></PageLayout>} />
<Route path="/terms" ... />
{SHARE_TYPE_SLUGS.map(slug => <Route path={`/${slug}/:referenceCode`} element={<ShareDetail shareType={slug} />} />)}
<Route path="*" element={<Navigate to="/" replace />} />
```

`pages.ts` stays the single source of truth for nav links **and** routes (a link cannot point at a
missing route). Route overlap between the two apps, and what happens to each:

| Route | Old app | New app | Plan |
|---|---|---|---|
| `/` | `LandingPage` | `LandingPage` | new |
| `/collect` `/buy` `/sell` | thin `ContentPage` wrappers | `PAGES` → `PageLayout` | new |
| `/create` `/trade` `/share` `/support` | — | `PAGES` | new |
| `/about` `/fair-market` `/refund-and-return-policy` | `ContentPage` (placeholder) | `FOOTER_PAGES` | new |
| `/privacy-policy` | `Privacy.tsx` — **544 lines of real legal copy** | `FOOTER_PAGES` placeholder | **keep the copy**, render in `PageLayout` |
| `/terms` `/terms-of-use` `/legal` | `Terms`/`ContentPage` | — | keep `/terms` (legal); collapse aliases → **decision A** |
| `/pricing` | `Pricing` | — | **decision B** |
| `/how-it-works/{buyers,sellers,collectors}` (+ `PayoutCalculator`) | `ContentPage` | — | **decision B** |
| `/contact-nanza` `/sell-on-nanza` `/transaction-integrity` | `ContentPage` placeholders | — | drop (catch-all → `/`) unless linked externally — **decision B** |
| 8 TCG game pages (`/pokemon`, `/magic-the-gathering`, …) | `ContentPage` placeholders | — | drop unless SEO-relevant — **decision B** |
| `/<slug>/:referenceCode` ×9 | `ShareDetail` | (noted as "still to port") | **unchanged** — stays in the target |

Static paths stay declared before the dynamic share routes, as both files already insist on.

### 2. Scroll ownership — exactly one Lenis

Today `LandingLayout` (old) creates a Lenis instance driven off the GSAP ticker; the new
`scene/useScrollTimeline.ts` does the same. `LandingLayout` is deleted with the old landing, so the
new timeline is the only owner. The new `PageLayout` does not use Lenis; content pages scroll
natively. Nothing outside `src/landing` imports `LandingLayout` (verified: only `AppLayout` imports
from `landing/`), so deleting it strands nothing.

### 3. Styling — scoped dark theme over the existing global sheet

- **Tailwind config** (`tailwind.config.js`, CommonJS in the target): extend `content` globs to add
  `./src/pages/**`, `./src/sections/**`, `./src/scene/**` (and `./src/*.{ts,tsx}` for `icons.tsx`),
  otherwise every class in the new tree is purged. Add the semantic dark tokens (`page`, `ink`,
  `ink-muted`, `ink-faint`, `surface`, `hairline`) and the `roobert` font family from the update
  repo's config. Keep the existing `sans`/`figtree`/`fadeIn` entries.
- **`src/index.css`** stays the global sheet (`@tailwind` + Euclid `@font-face` + `body` font/margin).
  Add `@import './styles/public-site.css'` after the `@tailwind` directives (or import it from
  `src/index.tsx`). Its rules are all scoped under `html.public-site`, so share/detail screens keep
  the light theme; the two unscoped blocks (Roobert `@font-face`, Lenis classes) are inert until
  used.
- **Delete** `src/landing/styles/landing.css` (its Lenis block and Roobert faces are duplicated by
  `public-site.css`; its `.landing-page` / `.hero-video*` rules die with the components).
- **Delete** `src/styles/globals.css` — nothing imports it (dead since before this work; the vault's
  INDEX/architecture mention it and need correcting).
- Roobert `@font-face` in `public-site.css` points at `/fonts/Roobert-*.otf`, which `public/fonts/`
  already serves — no asset copy needed. `src/fonts/Roobert-*.otf` become unreferenced and can go.
- `public/index.html`: add the inline `html { background: #0d0d0d }` + `theme-color`? **No** —
  the target serves light share pages from the same document. Instead rely on
  `usePublicSite`'s `useLayoutEffect` (class lands before paint). Accept a possible single white
  frame on cold load of `/`; revisit only if it's visible.

### 4. Dependencies and the CRA build

Add to `nanza-web-app/package.json`: `three@^0.169`, `@react-three/fiber@^8.17`,
`@react-three/drei@^9.114`, `@types/three@^0.169`. `gsap`, `lenis`, `react-router-dom`, `tailwindcss`
are already present at compatible versions. Nothing is removed in this pass (`framer-motion` is
still used by other screens; confirm before pruning).

CRA 5 / webpack 5 handles `.webp` and `.otf` imports and three's ESM. Two things to verify on the
first build rather than assume: (a) `tsconfig` `target: es5` — `noEmit` + Babel means it only
affects type-checking, but drei's types may need `target: es2015`+ `lib`; (b) the update repo's
`strict` + `noUnusedLocals` is stricter than the target — the copied code passes it, so the target's
looser config is fine. `motionState.ts` uses a hostname check instead of `import.meta.env`, so no
env shim is needed.

Bundle size: three + drei add ~600 KB gz to the single CRA chunk, and that chunk is shared with
share pages. **Lazy-load the scene**: `React.lazy(() => import('../scene/PhoneCanvas'))` behind the
existing `useImmersive` gate (which already turns 3D off for reduced-motion, <768 px, no-WebGL) so
mobile share visitors never download it. This is the one structural change made during the port.

### 5. Real copy vs placeholder

`pages.ts` says every title/intro is **placeholder**. The port must not regress real content:

- `/privacy-policy` — move the legal body out of `landing/pages/Privacy.tsx` into a plain
  `pages/legal/PrivacyCopy.tsx` (JSX only, no layout) and render it as `PageLayout` children.
- `/terms` — same treatment if `Terms.tsx` carries real copy (check; it's a thin wrapper today).
- Everything else in the old `landing/pages` is a titled empty shell (`ContentPage`) and is
  replaced by the new placeholder without loss.

`PageLayout` needs to accept `children` for this (it takes `title`/`intro` today).

### 6. Deprecation — what gets deleted

`src/landing/components/*` (all 23, including dead `LinksEverywhere`, `SellFastSectionV1`,
`LogoMarquee`, `StickyContainer`, `Buy/Collection/GetOffers/ListFast/MultipleCollections/SellLots/
Share*Section`, the unused `Navigation.tsx`, `DownloadCTA`, `LandingLayout`, `Hero`, `Footer`),
`src/landing/pages/*` (after extracting Privacy/Terms copy), `src/landing/styles/landing.css`,
`src/styles/globals.css`, `public/videos/` (~29 MB, only `sequence1d.mp4` was referenced and it
dies with `Hero.tsx`), `src/fonts/Roobert-*.otf`. `PayoutCalculator` survives only if decision B
keeps `/how-it-works/sellers`.

Delete, don't comment out — this differs from the auth/cart/storefront convention because the old
landing is being superseded, not paused; git history keeps it.

## Sequence

Work on a branch off **`dev`** (the working tree is currently on `prod`; do not port there).

1. **Deps + config**: add the three/R3F deps; extend `tailwind.config.js` (globs + tokens + roobert);
   `@import` `public-site.css` from `index.css`. Build still green with the old landing.
2. **Copy the new tree** (`usePublicSite`, `landing/index.tsx`, `pages/`, `sections/`, `scene/`,
   `assets/cards/`, `theme.ts`, `icons.tsx`, `public/` assets). Lazy-load `PhoneCanvas`.
3. **Extract real copy** (Privacy, Terms) into `pages/legal/`; give `PageLayout` `children`.
4. **Rewire `AppLayout`**: swap the marketing block for the generated routes per §1 and the decided
   extras; share routes and catch-all untouched.
5. **Delete** per §6. `npm run build`, `eslint`, `tsc --noEmit` clean.
6. **Update this vault**: fold the durable parts into a `landing` solution design, fix
   `INDEX.md`/`architecture.md` (`src/landing` description, styling section, the dead `globals.css`
   mention, new `pages/sections/scene` dirs), delete this plan.

## Implementation notes (what differed from the design above)

- **Fonts:** CRA's css-loader treats a root-absolute `url('/fonts/…')` as a module path and fails
  the build, so the four Roobert weights the site sets were restored to `src/fonts/` and
  `public-site.css` references them relatively — the same convention `index.css` uses for Euclid.
  The stylesheet is imported from `src/index.tsx` (a CSS `@import` after `@tailwind` rules is
  invalid ordering).
- **`three` in the main bundle:** `StaticPhone` (the no-WebGL / mobile path) read `SCREENS` from
  `scene/screens.ts`, which imports `three` for its texture loader — enough to drag three into the
  main chunk despite the lazy `PhoneCanvas`. The plain-data manifest now lives in
  `scene/screenSpecs.ts` (no three) and `screens.ts` re-exports it.
- **drei:** import from `@react-three/drei/core`, not the barrel — the barrel re-exports
  `FaceLandmarker` → `@mediapipe/tasks-vision`, whose missing source map warns on every build.
- **Store links:** the new sections shipped `href="#"` placeholders; those are `jsx-a11y` warnings,
  and GitHub Actions runs the build with `CI=true` (warnings → errors). They now use
  `APP_STORE_URL` / `PLAY_STORE_URL` from `src/constants.ts`. The hero's "Join the chat" button
  points at the App Store — revisit if it should go elsewhere.
- **Lint:** the target's `spaced-comment: never` / `no-inline-comments` rules were applied to the
  new tree; `react/no-unknown-property` is switched off for `src/scene/**` (R3F JSX props). The
  legal copy's straight quotes are escaped (`&quot;`/`&apos;`).
- **Terms:** `/terms` was added to `FOOTER_PAGES` so the footer links it; `AppLayout` maps
  `/privacy-policy` and `/terms` to their copy via a small `LEGAL_BODIES` table.
- **Also removed:** the eight unused Roobert weights in `src/fonts/`, `src/styles/globals.css`
  (dead), `public/videos/` (~29 MB).

## Verification

- `/` renders the scrubbed landing; scroll drives the phone scene; hero video plays; reduced-motion
  and <768 px get `StaticPhone`; no console errors from three under webpack.
- `/collect`, `/create`, `/about`, `/privacy-policy` (real copy) render in `PageLayout`; nav links and
  routes agree (they're the same list).
- **Share pages are unchanged**: open `/listing/<code>` — light background, Euclid body font, no
  `public-site` class on `<html>`, `history.scrollRestoration` back to `auto`. Navigate `/` →
  `/listing/<code>` → `/` and confirm the class toggles and only one Lenis instance ever exists.
- Production bundle: the three/R3F chunk is a separate lazy chunk and is not requested on a share
  page load.
- Deploy to dev via the existing S3/CloudFront workflow; smoke the above on the dev URL.

## Risks

- **Bundle weight on share pages** — mitigated by the lazy scene chunk; verify, don't assume.
- **Tailwind purge** silently stripping the new tree's classes if a glob is missed — the first
  visual pass will show it immediately (unstyled sections).
- **Route regressions** for URLs in the wild (`/pricing`, `/how-it-works/*`, TCG pages) — they fall
  through to `/`, which the app treats as the correct answer for retired links; still, decide B
  deliberately.
- **CRA + three** compile quirks (ESM/type target) — caught at step 1/2, before anything is deleted.
- **White flash** on cold load of `/` without the inline `<html>` background — accepted, revisit.

## Open decisions (for Skylar)

- ~~**A.** Legal aliases~~ — decided: `/terms` only; `/terms-of-use` and `/legal` fall to the catch-all.
- ~~**B.** Which old-only routes survive~~ — decided: none. Drop them all; re-add under `pages.ts` when
  real copy exists.
- ~~**C.** Fate of `nanza-web-app-update`~~ — decided: it is a temporary repo, not a playground.
  Once the port is verified on dev it can be deleted; nanza-web-app is the only home for this code.
  (Strip the "meant to replace nanza-web-app" docblocks from `AppLayout`/`usePublicSite`/`pages.ts`
  as they're copied — they describe a migration that will have happened.)
- ~~**D.** Placeholder copy~~ — decided: port `pages.ts` exactly as it is (the existing headlines —
  "Every card you own, in one place", "Talk and trade with the people who get it", … — and the
  shared intro line). Copy refinement is a later pass on `pages.ts`, not part of this change.

All four decisions are made (2026-08-27); the plan is ready to execute.
