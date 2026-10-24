---
tags: [nanza-web-app, solution-design, analytics, gtm]
status: Built 2026-10-15, bottom-up revision the same day. Not live until the accounts in "Setting up the accounts" exist and
  `GTM_ID_DEV` / `GTM_ID_PROD` are set. The privacy-policy copy awaits legal review.
---

# Analytics — Solution Design

## Overview

The whole public site, from the marketing pages and share links to the profile pages (down to
the entity lightbox and a post's reply and like glyphs), reports to **Google Tag Manager**.
**GA4** is configured inside the container, not in the code. (Microsoft Clarity for heatmaps and
recordings was designed in and taken out on 2026-10-15 — not used yet.) Everything the app sends
comes from one module, `src/analytics.ts`. Components carry markup, not tracking calls.

## How it works

**Bottom up: analytics only listens.** The app does its thing and analytics is attached at the
very end of the chain. It never sits inside a data fetch or a route, makes no request
to our API, adds no React effects, and changes nothing about what the app does or when. Every
listener is wrapped so an error in it stays in it. Two off switches: leave `REACT_APP_GTM_ID`
empty and nothing loads or listens (no `dataLayer`, history untouched); or pause/unpublish the
GTM container and nothing is sent, with no deploy. Verified 2026-10-15: the same walk through a
profile made the same seven API requests with analytics on and off.

**The code talks to `dataLayer` and nothing else.** Which tools hear about an event is the
container's business, so adding a destination is a GTM publish, not a deploy.

**Three listeners, set up once in `initAnalytics()`, instead of a call per button:**

- **The browser's history, for page views.** React Router moves with `pushState` /
  `replaceState`; wrapping those (as GTM's own history trigger does) plus `popstate` sees every
  route change with no component. The view goes 100ms later, once the page has rendered and set
  its title — and a route that only redirects has left by then, so it is never counted.
- **`document` in the capture phase, for clicks, keys and copies.** The profile pages stop
  propagation everywhere (a trade tile, the reply/like glyphs, the share glyph, the link rows
  keep their click from opening the post they sit inside). A capture listener sees the event on
  the way down, before React, so nothing in a component can hide a click. Overlays (lightbox,
  menus) are in the same document and need nothing. A button added later is tracked the day it
  ships.
- **Scrolling, for depth and for the named sections reached.**

| Event | Source |
|---|---|
| `page_view` | history wrapper + `popstate` |
| `dead_link` | a `replaceState` to `/` from elsewhere — the app's `<Navigate to="/" replace />` turning a visitor away (unknown path, bad share code, unknown username); carries `from_path` |
| `ui_click`, `app_store_click` | capture `click` — the latter derived from an `apps.apple.com` / `play.google.com` href |
| `ui_key` | capture `keydown` — Enter/Space on the `role=button/link` spans (real buttons turn Enter into a click themselves), Escape while a dialog is open |
| `section_view` | an `IntersectionObserver` on every `section[id]`, fired as each crosses the middle of the window |
| `scroll_depth` | 25/50/75/90%, once each per page view |
| `text_copy` | capture `copy` — where on the page, never what |
| `share` | the one explicit `track()` in a component: `ShareButton`, after the share resolves (`native`, `clipboard`, `prompt`) |
| `app_store_redirect` | `trackThenGo()` on the two automatic store redirects — below |
| `join_sheet_shown`, `join_sheet_dismissed` | explicit `track()` in `JoinSheet` — what a click cannot show: that it opened (`variant` = `affiliate` / `generic`, `trigger` = `time` / `scroll` / `tab`, `affiliate`) and how it went away (`variant`, `method` = `keep_browsing` / `backdrop` / `swipe` / `escape`). See [[join-sheet]] |

**Params travel under `event_params`, cleared before every push.** GTM merges each push into
what came before, so a `post_id` from one click would otherwise ride along on the next. Every
event carries `page_type`.

**What a click records.** The nearest of `[data-track], a[href], button, [role=button|link|tab|menuitem], summary`
above the target. `click_label` is `data-track`, else `aria-label`, else visible text (80 chars),
else `title` or an image's `alt`. `click_region` is the nearest `data-track-region` or landmark
(`nav`, `header`, `footer`, `aside`, `main`, a dialog — by its `aria-label` when it has one, so
the tabs read "Profile sections"). `click_url`, `link_domain` and `outbound` come from the
element's **own** href only: a reply glyph inside a post's link is not a click on that link.
Clicks on nothing interactive are not sent.

**`data-track` is the only way context reaches analytics** — an attribute, not an import. It is
used where an `aria-label` carries a count or a name (`"3 replies — reply in the app"`,
`"Enlarge Spider's Bite"`), and so would make a new label per post, and on the lightbox's
backdrop and image, which are not controls. Every `data-track-foo-bar` on the element or an
ancestor becomes param `foo_bar`, nearest first — so `data-track-post-id` on a post's
`<article>` reaches every click inside it.

| Name | Where | Params |
|---|---|---|
| `post_open`, `post_back` | the feed row's link, the post page's byline | `post_id` |
| `post_reply`, `post_like`, `post_share` | the footer glyphs | `post_id` |
| `post_link` | a link-preview row | `link_domain` |
| `post_trade_item` | a listing/bid tile in a post | `item_type`, `item_id` |
| `entity_open` | collection tiles and rows, cards in posts | `entity_id`, `entity_name` |
| `entity_lightbox_close` / `_backdrop` / `_image` | the lightbox (region `entity_lightbox`) | — |
| `trade_open`, `collection_open` | Trades and Collections tab cards | `item_type`, `item_id` |
| `profile_back`, `profile_social` | the compact hero row, social chips | `network` |
| `trade_back`, `trade_trader`, `trade_offer` | listing / bid / lot screens | — |
| `consent_accept`, `consent_decline` | the banner (region `consent_banner`) | — |
| `join_sheet_join`, `join_sheet_keep_browsing` | the join sheet (region `join_sheet`) | `variant` |
| `join_sheet_tab` | the arrow tab that reopens it | — |

The marketing site needed none: its labels are already stable.

**Page types from the route registries.** `pageTypeOf` reads `PAGES` / `FOOTER_PAGES` and the
share slugs — the lists the route table is built from — so it cannot drift from the routes:
`landing`, `content`, `guide`, `get_app`, `share_<slug>` (+ `share_code`), `profile_posts` /
`_collections` / `_trades` (+ `profile_username`), `profile_post` … `profile_lot` (+ `item_id`).
Read off the address alone — nothing is fetched to find out. Reports group by these, not by
title: a profile's title arrives with its data, which may be after the page view.

**The one exception: the two store redirects wait for their hit.** A share link opened on a
phone without the app (`ShareDetail`), and `/get` (the QR code's address), replace themselves with
the App Store or Google Play as soon as they load — before GTM would have sent anything, so the
visits that say whether sharing and the QR code work would never be counted. There,
`trackThenGo()` takes the place of `location.replace`: it sends the page view and
`app_store_redirect`, then leaves when GTM reports the tags done (about 30ms once loaded) or after
800ms, whichever comes first. It only ever delays the redirect; with analytics off it is a plain
`location.replace`, and a second call (a re-render) is ignored, so it can stay in render where
the redirect always was. Agreed 2026-10-15 for these two pages only.

## Consent

Consent Mode v2 with our own banner (`src/components/ConsentBanner.tsx`), not a consent
platform: there is no advertising, so Google does not require a certified one, and their free
tiers count pages — every profile is one.

- **Defaults**, set before GTM loads: analytics storage **denied in the EEA, UK and Switzerland**
  (Consent Mode's `region` list, which Google applies by location) and granted elsewhere; ad
  storage denied everywhere. A stored answer is applied at once.
- **The banner** appears on a European time zone (`Europe/*`, the Atlantic islands, Arctic) when
  there is no stored answer. The page cannot see the country, and when time zone and location
  disagree the result is still safe: a visitor in Europe on another zone keeps the denied
  default, and a visitor elsewhere on a European zone is asked a question they did not need.
  Bottom-left from `lg`, opposite the QR card, so the two never stack.
- **The answer** goes to `localStorage` (`nanza:consent`) and to a gtag `consent update`.
- **"Cookie settings"** in the footer reopens the banner (only when analytics is on).
- While analytics storage is denied, GA sends cookieless pings (no cookies, modelled in reports).

## The container

`analytics-gtm-container.json`, next to this note, is the import file:

- **Google tag** on Initialization, `send_page_view: false` — the app sends its own page views.
- **One GA4 event tag** on every app event (trigger `Nanza - app events`, `{{Event}}` as the
  name), mapping every `event_params.*` variable. Unset params are dropped by gtag.
- One constant to fill in: **GA4 Measurement ID**.

It was generated from the module's event and param lists. It has **not been import-tested** —
import it into the dev container first and check it in Preview.

## Setting up the accounts

Do it twice — **dev** first, then **prod** — so test traffic never lands in the real property.

**A. GA4** — [analytics.google.com](https://analytics.google.com)
1. **Admin** (gear, bottom-left) → **Create** → **Property**. Name "Nanza Web (dev)", your time
   zone and currency → Next → business details → **Create**.
2. Platform: **Web**. URL: the dev site's domain (prod: `nanza.app`); stream name "Web".
   **Create stream**. Copy the **Measurement ID** (`G-…`) from the stream details.
3. Same screen, **Enhanced measurement** → gear → turn **off**: *Page changes based on browser
   history events*, *Scrolls*, *Outbound clicks* (the app sends its own). Leave the rest. Save.
4. Admin → **Data collection and modification → Data retention** → Event data retention:
   **14 months** → Save. (The privacy policy says 14 months.)
5. Admin → **Data display → Custom definitions** → **Create custom dimension**, once per param.
   Dimension name = Event parameter = the param, Scope **Event**:
   `page_type`, `profile_username`, `item_id`, `item_type`, `share_code`, `share_type`, `source`,
   `click_label`, `click_element`, `click_region`, `click_url`, `link_domain`, `outbound`,
   `post_id`, `entity_id`, `entity_name`, `network`, `section_id`, `store`, `method`,
   `content_type`, `from_path`, `key`, `percent_scrolled` (24 of 50).
6. After the first events arrive (step C): Admin → **Data display → Events** → star / **Mark as
   key event** on `app_store_click`, `app_store_redirect` and `share`.

**B. GTM** — [tagmanager.google.com](https://tagmanager.google.com)
1. **Create account** "Nanza" → container "nanza.app (dev)", target **Web** → Create. Copy the
   **Container ID** (`GTM-…`); close the install-snippet dialog (the app injects GTM itself).
2. **Admin → Import container** → `analytics-gtm-container.json` → workspace **Existing:
   Default Workspace** → **Merge** (Overwrite conflicting) → Confirm.
3. **Variables** → *GA4 Measurement ID* → set to the `G-…` from A2 → Save.
4. **Preview** → enter the site URL (checks below) → when right, **Submit → Publish**.

**C. The site**
1. GitHub → repo **Settings → Secrets and variables → Actions → Variables**: `GTM_ID_DEV`,
   `GTM_ID_PROD` (the workflows already pass them as `REACT_APP_GTM_ID`).
2. Locally: `REACT_APP_GTM_ID=GTM-…` (dev) in `.env`, then restart `npm start`.

**Checks** in GTM Preview, then GA4 **Admin → Data display → DebugView**: landing scrolled to the
bottom (sections and depths) → a store pill (`app_store_click`); a profile → like, share, open a
post, back → a collection → a card, closed by backdrop, button and Escape; the browser's Back
button (a page view); a bad share code (`dead_link`); with a phone, a share link and `/get`
(`app_store_redirect`, then the store). On a European time zone, the banner appears.
Standard reports fill in after 24–48 hours; DebugView and Realtime are immediate.

## Privacy policy

`pages/legal/PrivacyCopy.tsx` gained a **"Do we use cookies and other tracking technologies?"**
section (§4, `#cookies`, which the banner links to; the later sections renumber), an
"Information automatically collected" part in §1, an analytics purpose in §2,
Google as an analytics provider in §3, a pointer from the do-not-track section, a key-points
entry, a revised date, network activity marked collected in the California table, and its
retention, and the California disclosures: Category A (online identifiers, IP addresses) and
Category F (network activity) are disclosed to Google, as a service provider, for a business
purpose; nothing is sold or shared for advertising. It was drafted for **legal review**. The page
mirrors `oakplatforms.com/nanza/privacy`, which needs the same change.

## Known gaps

- **Not tracked in this phase**, by the bottom-up rule: failed API calls (needs a hook inside
  `fetchData`), JavaScript errors, web vitals, video events, the reason a visitor was turned
  away (only where from). Heatmaps and recordings (Clarity) are not used yet.
- **`/get` is slow to redirect at all** (~3.5s locally, with or without analytics): its redirect
  waits for the full marketing page to render. Unrelated to analytics.
- **Profile page views can carry the default title** when the profile's data is slower than the
  100ms wait. Group by `page_type` + `profile_username`.
- **An unknown username is counted as a profile view** before its `dead_link`: the app only
  redirects once the API answers.
- **No user identity yet.** When sign-in returns, set GA's `user_id`, and
  add `sign_up` / `login`; with checkout, `begin_checkout` / `purchase`.
