---
tags: [nanza-web-app, solution-design, seo]
---

# SEO — Solution Design

## Overview

The goal: a search for "nanza" returns Nanza's own pages and nothing else. That means the homepage, the marketing pages and creator profiles, each with its own title and description.

The site is a client-rendered CRA SPA on S3 behind CloudFront. Until 2026-10-16:
- Every URL served the same HTML: `<title>Nanza</title>`, the description "Trade TCGs", and an empty `#root`.
- Unknown paths and usernames redirected to `/` with HTTP 200, which search engines count as soft 404s.
- There was no sitemap, canonical or structured data.

This design fixes those gaps without touching ogEdge (the Lambda@Edge in nanza-api).

## How it works

**Two kinds of crawler, two answers.**
- **Google and Bing run JavaScript.** They index what the page writes into `<head>` once it has loaded. That covers every page, profiles included.
- **Crawlers that run no script** (AI search, parts of Bing, link unfurlers) read only the HTML that comes over the wire. They get the static defaults in `public/index.html` on every page. (Prerendering the marketing pages for them was built and shelved; see below.)

**One head per page: `useSeo`** (`src/hooks/useSeo.ts`).
- Writes the title, description, canonical, robots, `og:*`, `twitter:card`, and one `<script type="application/ld+json" data-seo>`.
- Restores every tag on unmount, so a page never inherits the previous page's description.
- **One call per rendered page, from the screen, not its layout.** Effects run child-first, so a layout that also wrote the head would overwrite its screen's.
  - `ProfileLayout` writes nothing except on its own not-found and error branches.
  - Content pages get theirs from `ContentPage` in `AppLayout.tsx`, not from `PageLayout`. `PageLayout` also frames `/get` and the error pages, which set their own.
- `pending: true` writes nothing while data loads, so a half-loaded page never claims a title.

**Title grammar.**
- Most specific first, parts joined by ` – `, ending ` / Nanza` (`seoTitle`).
- The homepage is the one exception. It leads with the brand: `Nanza – Buy, sell, trade and collect trading cards` (`HOME_SEO`, mirrored in `index.html`).
- Profiles (`src/app/ProfilePage/seo.ts`):

  | Page | Title |
  |---|---|
  | Profile (Posts tab) | `skibanks / Nanza` |
  | Tab | `skibanks – Collections / Nanza` |
  | Collection | `<Collection> – skibanks / Nanza` |
  | Listing | `<Card> for sale – skibanks / Nanza` |
  | Bid | `<Card> wanted – skibanks / Nanza` |
  | Post | `skibanks: “<opening words>” / Nanza` |

- `personName` produces `Name (@handle)` once profiles carry a display name distinct from the username. Today there is none, so it is the handle alone.
- Descriptions:
  - **Profile:** the bio.
  - **Tabs:** what the tab lists, then the bio.
  - **Detail screens:** what the thing is, written out (card, set, condition, price).

**Help centre guides** (`src/pages/support/seo.ts`, added 2026-10-19). 75 how-to guides at
`/support/<slug>`, all built from one data file (see [[help-centre]]), so their SEO comes from
one function, `guideSeo`, rather than per page.
- **Title:** `<Guide> – Support / Nanza`, e.g. `How to make an offer on a listing – Support / Nanza`.
- **Description:** the guide's intro, then its opening steps, cut on a word at 155 characters.
  Intros are one short promise ("Look around before you sign up."), too thin to stand alone in
  a result; the steps say what the page walks through. Bold markers are stripped.
- **Canonical:** the guide's own path. The one alias, `save-to-a-collection`, repeats
  `add-a-card-by-searching` word for word (it exists so the `/create` link text matches the
  page title), so its canonical points at the original and it stays out of the sitemap.
- **JSON-LD:** `BreadcrumbList`, Nanza › Support › Guide. The category level is left out
  because it is a fragment of `/support` (`#posting`), not a page, and breadcrumb items should
  be real URLs. No `HowTo`: Google retired HowTo rich results in 2023.
- **`og:type`:** `article`.
- **Drafts:** `noindex` and out of the sitemap. None today.
- The hub pages (`/create`, `/collect`, `/trade`) and `/support` link every published guide,
  so crawlers reach them by links as well as by the sitemap.

**Canonicals.**
- Always `SITE_URL` plus the path, in the stored username casing.
- A wrong-case visit (`/SKIBANKS/collections`) is also rewritten in the address bar, so one profile has one address.
- The Posts tab is the profile's canonical page.
- A noindexed page names no canonical, because the two would contradict each other.

**What stays out of search** (`noindex, follow`):
- the not-found and error pages
- `/get` and the share routes (`/listing/<code>` …), which send phones (Googlebot's smartphone crawler included) to a store
- draft guides
- registry pages marked `noindex` (FairMarket "under construction", the placeholder About and Refunds pages)

**Not-found vs error** (`src/pages/ErrorPages.tsx`).
- An API 400/404 means the name or id doesn't exist, so it shows `NotFound`.
- Anything else (5xx, network, rate limit) shows `ErrorPage` with a retry.
- They are kept apart so an outage never tells Google a profile is gone.
- CloudFront still answers 200 (the SPA fallback). Google honours a noindex set by script.

**Structured data.**
- **Static in `index.html`:** `Organization` (logo, `sameAs` the store listings), `WebSite` (`name: "Nanza"`, which drives the site name Google shows) and `MobileApplication`.
- **Profiles:** `ProfilePage` → `Person`, with `sameAs` set to the creator's own social links.
- **Posts:** `SocialMediaPosting`.
- **Detail screens:** `BreadcrumbList`.
- Listings deliberately carry no `Product`/`Offer`. Peer listings without stable pricing data would draw structured-data warnings.

**No `og:*` defaults in `index.html`.** ogEdge injects its tags into that same file for share links, and a default there could be read before its tags. So:
- Share links keep their personalised previews.
- A pasted profile URL unfurls with no image. Only the edge could change that.

**robots.txt and sitemap are kept by hand** (`public/robots.txt`, `public/sitemap.xml`).
- The sitemap lists `/` plus the `pages.ts` entries without `noindex`, then every published guide that is its own canonical (75 as of 2026-10-19, in `/support` order). Add a page there when it's added to the registry or to `guides.ts`.
- Profiles aren't in it; they're found through links.
- `SITE_URL` (`src/constants.ts`) is the constant `https://nanza.app`.

**Prerendered marketing pages: built, then shelved** (git stash "SEO phase 7"; Puppeteer was judged too heavy for now). How it worked, for when it comes back:
- Puppeteer loads every sitemap URL except `/` against `build/` (now including the 75 guides, which are the pages that would gain the most: plain text, no data fetch) and writes `build/__prerender/<path>.html`. The app sees `window.__PRERENDER__` (`src/prerender.ts`), which switches off GTM and the consent banner.
- The deploy (`buildspec.yml`) uploads each snapshot to its **extensionless key** (`collect`, `support/add-a-photo`) as `text/html`. CloudFront serves an existing key directly; only a missing key falls back to `index.html`. This was confirmed on prod: `/.well-known/apple-app-site-association` is `Miss from cloudfront`, `/collect` was `Error from cloudfront`.
- The sync's `--delete` removes last deploy's copies first, so a page dropped from the sitemap falls back to the SPA.
- React uses `createRoot`, not hydrate, so it simply re-renders over the snapshot, and the two can never disagree. The content pages don't animate in, so there is no flash.
- **`/` is never snapshotted.** `index.html` is also the fallback for every profile, so home content there would show as the content of every profile.
- If Chrome won't start in CodeBuild, the script warns, writes nothing, and the build passes. `PRERENDER_STRICT=1` makes failures fatal.

## Key decisions

- **Client-side heads, not prerender or SSR.** Google and Bing render JavaScript, which covers every page. Build-time snapshots could only ever serve the static pages, because profiles share one fallback HTML, and they were shelved. A Next.js move was rejected as a rewrite.
- **ogEdge untouched.** It keeps share-link previews. Profile URL previews stay generic.
- **Every hub link is a real guide** (2026-10-19). The earlier stopgap, unwritten article links landing on a noindexed not-found page, no longer applies: links for features the app lacks were taken off the hubs instead (see [[help-centre]]).
- **One SEO function for all guides.** Guides are data, so their titles, descriptions and breadcrumbs are derived, not written per page. Editing a guide's title or intro updates its search result with it.

## Not done yet

- **After the first deploy:**
  - `curl -A facebookexternalhit` on a real share link should still show ogEdge's tags.
  - Run the Rich Results Test on a profile.
  - Check `https://nanza.app/sitemap.xml` and `/robots.txt` are served.
- **Off-site:**
  - Verify nanza.app in Google Search Console and Bing Webmaster Tools, and submit `/sitemap.xml`.
  - Make `www.nanza.app` 301-redirect to the apex. As of 2026-10-16 it serves the site with a 200.
  - Nanza has no social accounts yet (2026-10-16). When one exists, add it to `sameAs` in `index.html` and link it back to nanza.app.
- **Sitemap drift.** With 75 guides the hand-kept sitemap is the easiest thing to forget. If guides keep growing, generate it from `GUIDES` at build time instead.
- **Phase 3:**
  - A profile sitemap needs an API endpoint that lists public usernames.
  - Competitor and game landing pages, card pages and personalised profile unfurls are deferred.
- **Seen while testing (2026-10-16):** one request to `nanza.app/listing/32S392` got CloudFront's 503 "Lambda function associated with the CloudFront distribution is invalid". Retries were fine. Worth checking ogEdge's association and permissions.

## Related

- [[profile-page]] · [[sharing]] · [[help-centre]] · [[../architecture|Architecture]]
- [[../../nanza-api/solution-designs/og-sharing|nanza-api OG sharing (ogEdge)]]
