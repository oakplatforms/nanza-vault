---
tags: [plan, landing, migration]
status: PLAN — all four decisions settled, prep landed in nanza-three-js. Only the route-table
  edit remains, which needs the escalation in §7.
---

# Landing — migrating the nanza-three-js front end into nanza-web-app

Plan for dropping the immersive landing surface built in `nanza-three-js` into this app,
replacing the marketing front end **without touching** the share/detail routes, cart, auth
or service layer.

---

## 1. Problem Analysis

The two projects were built to converge, and structurally they do: `nanza-three-js` already
mirrors this repo's `src/app/AppLayout.tsx` + `src/landing/index.tsx` shape, uses the same
React Router v6 flat route table, and its `tailwind.config.js` header says it mirrors this
one. Fonts line up too — Roobert is **already** in `public/fonts/` here, so the
`@font-face` block in `nanza-three-js/src/styles/landing.css` resolves unchanged.

Five things do not transfer as-is.

**a) Build tool mismatch — CRA vs Vite.** This app is `react-scripts` 5.0.0
(`package.json`); nanza-three-js is Vite 5. One hard blocker:
`nanza-three-js/src/scene/vortexCards.ts:19` uses `import.meta.glob` to build the card
manifest, which webpack 5 does not implement. `src/vite-env.d.ts` also carries
`/// <reference types="vite/client" />`, which fails to resolve without Vite installed, and
`src/main.tsx` is a Vite entry with no role here.

**b) Global CSS would leak to every route.** `nanza-three-js/src/styles/landing.css:56-63`
sets `html, body { background:#0d0d0d; color:#f5f5f7 }`. Imported into this app that
repaints ShareDetail, Cart and every other surface. The same file also opens with
`@tailwind base/components/utilities`, which `src/index.css` already declares — importing
both emits Tailwind twice.

**c) Tailwind would purge the new classes.** `tailwind.config.js` scans only
`src/{components,app,landing}`. The ported code lives in `src/sections`, `src/scene`,
`src/pages` and `src/icons.tsx`. Its `theme.extend` also lacks the `page` / `ink` /
`surface` / `hairline` tokens and the `roobert` family the new work is built on.

**d) Two files collide by name.** `src/app/AppLayout.tsx` and `src/landing/index.tsx` exist
in both repos. The first is this app's entire route table including `ShareDetail` — it must
be merged, never overwritten. The second is the intended replacement target.
`src/landing/pages/` also already exists here (About, Buy, Sell, Collect, …), so the
incoming `src/pages/` must not land there.

**e) Seven of eleven incoming routes already exist.** `/buy`, `/sell`, `/collect`,
`/about`, `/privacy-policy`, `/fair-market`, `/refund-and-return-policy` are all live in
`AppLayout.tsx` today. Only `/post`, `/trade`, `/share`, `/support` are new.

Two things checked and **clear**: share routes are two-segment (`/{slug}/:referenceCode`,
`AppLayout.tsx:205`) so single-segment marketing paths cannot collide with them — including
`/post`, even though `post` is itself a share slug (`helpers/shareTaxonomy.ts:14`). And the
Universal Links AASA pattern matches `/??????` (six characters); none of the four new paths
are six characters long.

**Confidence: 0.95** — every claim above was verified by direct inspection, not inferred.

---

## 2. Proposed Solution

Land the whole immersive front end **inside `src/landing/`** as additive files, replace only
`src/landing/index.tsx`, and make exactly one edit to `AppLayout.tsx`.

Nesting under `src/landing/` is what makes this cheap: that path is **already** in the
Tailwind `content` globs, so no file-coverage change is needed — only the `theme.extend`
merge for the new tokens. It also keeps the entire new surface in one reviewable place, and
leaves `src/landing/components/` and `src/landing/pages/` (the current marketing site)
untouched and still serving their routes until each is migrated deliberately.

Target layout:

```
src/landing/
  index.tsx          ← REPLACED: the new LandingPage
  scene/             ← NEW (3D phone, scroll timeline, vortex sources)
  sections/          ← NEW (IntroHero, Navigation, Footer, …)
  PageLayout.tsx     ← NEW (flat file: `pages/` is taken)
  routes.ts          ← NEW (was pages/pages.ts)
  theme.ts           ← NEW
  icons.tsx          ← NEW
  immersive.css      ← NEW (was styles/landing.css, minus @tailwind + globals)
  components/, pages/, styles/   ← UNTOUCHED, still serving the old routes
```

**Confidence: 0.88** — the structure is verified; the uncertainty is in §3's open decisions,
not in the shape.

---

## 3. Alternatives Considered

**Option 1 (Recommended) — additive port under `src/landing/`, one route-table edit.**
**Confidence: 0.88.** Smallest blast radius that still meets the goal. Nothing outside
`src/landing/` changes except `AppLayout.tsx` (routes), `tailwind.config.js` (tokens) and
`package.json` (three deps). The old landing pages keep working, so the migration can be
finished route by route instead of in one commit.

**Option 2 — migrate this app from CRA to Vite first.** **Confidence: 0.35.** It would make
the port literally copy-paste, and CRA is deprecated so this is owed eventually. Rejected
for now: it rewrites the build, the `REACT_APP_*` env contract, both GitHub Actions
workflows and the S3/CloudFront deploy — the exact "rest of the app" this migration is
supposed to leave alone. Worth its own project afterwards.

**Option 3 — keep nanza-three-js deployed separately at a path or subdomain.**
**Confidence: 0.30.** Zero migration risk, but it doesn't replace the front end: two
deploys, two navs, and share links landing in a different app from the marketing site.

### Decisions (settled 2026-08-13)

**D1 — the seven colliding routes: OURS WIN.** `/buy`, `/sell`, `/collect`, `/about`,
`/privacy-policy`, `/fair-market`, `/refund-and-return-policy` are all replaced by the new
`PageLayout`. Replacing the marketing site is the point of the migration; the incoming copy
is placeholder and gets written, it does not get deferred to. The old `landing/pages/` and
`landing/components/` are deleted once their routes are cut over.

**D2 — the hero video: RESOLVED by re-encoding.** Was 144MB at 19.4 Mbps; re-exported at
1.6 Mbps to **12.2 MB**, same 59.6s, faststart intact. It now ships in the build like any
other asset — no external host, no second bucket, no CloudFront origin. Egress falls from
~$0.012 to ~$0.001 per visitor. The hosting notes below are kept only in case the video
grows again.

**D3 — transition period: not applicable.** D1 removes it; there is no window where the two
designs are both live.

**D4 — card artwork: RESOLVED, already done.** `import.meta.glob` is gone. `src/assets/cards/manifest.ts`
lists the images as static imports — the one form both Vite and webpack understand, and both
still fingerprint them. Adding artwork is now "drop the file in **and add its two lines**".

### Share paths are untouchable

Standing constraint, above everything else in this plan: nothing may disrupt
`/{slug}/:referenceCode`. Acted on already — the marketing "Post" page was moved off `/post`
to **`/create`**, because `post` is a share-type slug (`helpers/shareTaxonomy.ts:14`). The two
could not actually collide (matching is by segment count), but the share links are the live
product and a marketing page is not worth the question.

### Prep already landed (nanza-three-js `89ee9e9`)

Three of the five problems in §1 are now solved **on the source side**, so the migration no
longer has to solve them mid-flight:

- **(a) `import.meta.glob`** — replaced by the static manifest above.
- **(b) global CSS leak** — the stylesheet is split in two. `styles/tailwind.css` holds the
  three `@tailwind` directives and is the file that **does not migrate**;
  `styles/public-site.css` holds everything that does, with every rule that would reach the
  whole document scoped under `html.public-site`. `@font-face` and Lenis's classes stay
  unscoped on purpose — one declares a face without applying it, the other only matches while
  Lenis has put the class there itself.
- **(c)/(scroll)** — `app/usePublicSite.ts` owns that class *and* `scrollRestoration`, and is
  called by the two marketing surfaces rather than mounted at the router. Anything hung off
  the router would reach share cards, cart and detail screens; anything hung off those two
  components cannot. Both effects are undone on unmount.

What remains for migration day: install the three-stack, copy the files, merge the Tailwind
tokens, and edit the route table.

### Video hosting (D2 — no longer needed, kept for reference)

Resolved by re-encoding rather than by hosting: 144MB -> 12.2MB. What follows is the
reasoning, in case the asset ever grows back past what a static build should carry.

**The size was the problem, not the host.** 138MB for a hero loop is roughly ten times what
it should be; a full-bleed 1080p loop lands at 8-15MB. On a 25 Mbps connection 138MB is ~45
seconds of download, and on CloudFront's ~$0.085/GB it is about **$0.012 of egress per
visitor** — 10,000 visitors is ~$120 for one background video. Re-encode first and the
hosting question stops mattering much:

```
ffmpeg -i collecting-is-an-art-rc1.mp4 \
  -vf "scale=1920:-2,fps=30" \
  -c:v libx264 -profile:v high -crf 28 -preset slow -pix_fmt yuv420p \
  -c:a aac -b:a 96k \
  -movflags +faststart \
  hero-1080.mp4
```

`+faststart` is the non-optional flag: it moves the moov atom to the front so the file plays
before it has finished downloading. Without it the hero waits for the whole file however it
is hosted. Keep the audio track — the controls have a mute toggle and a scroll-driven fade.

**Temporary host — recommended: Cloudflare R2**, public bucket on a custom domain.
**Confidence: 0.85.** Egress is free, which removes the cost tail entirely while the video
is still being iterated on; storage is ~$0.015/GB-month, so this is cents. Setup is minutes
and nothing in AWS has to change.

**All-AWS alternative: a second S3 bucket** as an extra CloudFront origin under a `/media/*`
path pattern. **Confidence: 0.8.** Keeps one domain and one distribution, reuses the existing
OIDC role. The trap: it must not be the bucket or prefix that `aws s3 sync build/ --delete`
targets, or the next deploy silently removes the video.

**Long-term: Cloudflare Stream, Mux or Bunny Stream** — adaptive bitrate, so phones fetch a
small rendition instead of the desktop master. Right answer eventually, overkill for now.

**Avoid:** GitHub Releases/raw (not a CDN, hot-linking is against the spirit of the ToS),
Drive/Dropbox share links (unreliable range requests), and YouTube/Vimeo embeds — an iframe
cannot do the scrubbing, masking and audio-fade the opening is built on.

Whatever the host: it must support **HTTP range requests** (the scrubber and seeking depend
on them) and should send a long `Cache-Control`. S3, R2 and CloudFront all do by default.

## 4. Implementation Steps

1. **Install the 3D stack** — `three`, `@react-three/fiber@^8`, `@react-three/drei@^9`,
   `@types/three`. **Confidence: 0.9.** Risks: `@react-three/drei` types can want
   `moduleResolution: bundler`/`node16`; this app is on `node`. If types complain, the fix
   is a `skipLibCheck` reliance (already on) or a narrow tsconfig bump — verify before
   assuming. `gsap`, `lenis`, `react-router-dom`, `react`, `tailwindcss` versions already
   match exactly, so nothing else moves.

2. **Copy the code under `src/landing/`** per §2, rewriting relative imports and renaming
   `pages/` → `PageLayout.tsx` + `routes.ts`. Drop `main.tsx`, `App.tsx` and
   `vite-env.d.ts` — this app already has an entry and a provider stack.
   **Confidence: 0.9.** Risks: import churn is mechanical but wide; `noUnusedLocals` is on
   in nanza-three-js and off here, so dead imports will pass silently.

3. ~~**Replace `import.meta.glob`**~~ — **DONE** in prep. Copy
   `src/assets/cards/manifest.ts` across with the images.

4. ~~**Split the CSS**~~ — **DONE** in prep. Copy `styles/public-site.css` only; leave
   `styles/tailwind.css` behind. **Confidence: 0.95.** Risk retired: the scoping and its
   cleanup are already written and running.

5. **Merge `tailwind.config.js` `theme.extend`** — add `page` / `ink{,-muted,-faint}` /
   `surface` / `hairline` colours and the `roobert` family. **Confidence: 0.95.** Risks:
   none of these names are currently in use here (verified); the file is CJS while the
   source is ESM, so it is a transcribe, not a copy.

6. **Merge routes into `AppLayout.tsx`** — point `/` at the new landing page, repoint the
   seven replaced paths, and add `/create`, `/trade`, `/share`, `/support`. Static paths
   must stay above the `/{slug}/:referenceCode` block. **Confidence: 0.85.** Risks: this is
   a documented Non-Negotiable — see §7. The scroll-restoration side effect is no longer
   one of them: `usePublicSite` scopes it to the marketing surfaces.

7. **Verify the untouched surfaces** — load a share link, the cart route and a legacy
   landing page, and confirm colour, scroll restoration and fonts are unchanged.
   **Confidence: 0.9.** Risks: this is the step that catches 4 and 6 leaking.

---

## 5. Files Modified

| File | Kind | Est. |
|---|---|---|
| `src/landing/index.tsx` | **REPLACED** | ~45 |
| `src/landing/scene/*` (21 files) | **NEW** | ~2,600 |
| `src/landing/sections/*` (16 files) | **NEW** | ~2,300 |
| `src/landing/PageLayout.tsx`, `routes.ts`, `theme.ts`, `icons.tsx` | **NEW** | ~220 |
| `src/landing/immersive.css` | **NEW** | ~70 |
| `src/app/ScrollToTop.tsx` | **NEW** | ~35 |
| `src/app/AppLayout.tsx` | edit | ~+12 / −1 |
| `tailwind.config.js` | edit | ~+25 |
| `package.json` | edit | ~+4 |
| `public/cards/*.webp` | **NEW** (moved) | 39 files |
| `public/bulk-cards.png` | **NEW** (moved) | 2.5MB |
| video | **decision D2** | 138MB |

Roughly 60 new files, but only **three existing files edited** and one replaced.

---

## 6. DRY Check Results

**Searched for:** an existing landing shell, nav and footer (`src/landing/components/`,
`LandingLayout`); existing route table and share-slug handling (`AppLayout.tsx`,
`helpers/shareTaxonomy.ts`); Roobert `@font-face` declarations; the theme token palette
(`src/index.css`, `src/styles/globals.css`, `tailwind.config.js`); already-installed
animation deps.

**Found existing:** `LandingLayout` + section components (the current marketing site);
Roobert **already** in `public/fonts/` with all four needed weights; `gsap` and `lenis`
already installed at the exact versions nanza-three-js uses; a flat route table that already
follows the shape the incoming `AppLayout` mirrors.

**Decision:** Reuse the fonts, the animation deps, the router and the provider stack — none
of these are re-added. Create new for the sections/scene, because they are a different
design system (dark, Roobert display, scroll-scrubbed 3D) with no overlap in behaviour with
`landing/components/*`; the old components stay in place serving the routes that still use
them, and are deleted only as each route migrates. **Confidence: 0.9.**

---

## 7. Escalation Check

**Escalation needed? YES.**

```
**ESCALATION REQUIRED**

Proposed change:
- Files: src/app/AppLayout.tsx (route table), src/app/ScrollToTop.tsx (new),
  tailwind.config.js, package.json
- Impact: `/` renders the new immersive landing page instead of the current one;
  four new static routes; app-wide scroll restoration set to manual.

Cause for concern:
- What could break: fe_spec.md names AppLayout.tsx a Non-Negotiable ("Do not modify
  routing"). Static/dynamic route ranking protects the share surface — a new route
  inserted below the `/{slug}/:referenceCode` block, or a stray single-segment dynamic
  route, would shadow live share links. `history.scrollRestoration = 'manual'` is global
  and changes back-button behaviour on every route, not just the landing.
- Affected areas: every share/detail link (the app's core live surface), plus the seven
  marketing routes in D1.

Why this is the best option:
- The route table is the only place a landing page can be swapped; there is no way to
  replace the front end without touching it. Everything else is additive.
  Confidence: 0.85

Alternatives considered:
1. Vite migration first - Confidence: 0.35 - Why rejected: rewrites build, env contract
   and both deploy workflows; far larger blast radius than the routing edit it avoids.
2. Separate deployment - Confidence: 0.30 - Why rejected: does not replace the front end;
   splits nav and share links across two apps.

Risk mitigation:
- Keep the edit additive: one `element` swap on `/` plus four appended static routes;
  no reordering, no changes to the share block or the `*` catch-all.
- Scope scroll restoration to the landing routes, or accept it app-wide as an explicit
  decision.
- Verify a share link, the cart route and a legacy landing page after the change (step 7).
```

---

## 8. Overall Confidence Score

**0.82** — The structure genuinely does transfer: same router, same React, same
`gsap`/`lenis` versions, fonts already present, and the incoming code already mirrors this
repo's `app/` + `landing/` shape. Nesting under `src/landing/` means Tailwind needs no
content change and only three existing files get edited.

Key uncertainties, in order:

1. **`@react-three/drei` typings** under `moduleResolution: node` — the only item still
   unverified, and it will not be known until the install runs.
2. **The route-table edit** — low technical risk, but it is a documented Non-Negotiable and
   needs sign-off (§7).

Retired since this plan was written: the video (re-encoded to 12.2MB, ships in the build),
`import.meta.glob` (replaced by a static manifest), CSS leakage and scroll restoration (both
scoped behind `usePublicSite`), and the colliding routes (ours win).
