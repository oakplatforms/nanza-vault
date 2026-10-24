---
tags: [nanza-web-app, solution-design, landing, content-pages]
---

# Content pages — Solution Design

## Overview

The marketing nav pages (Create, Collect, Buy, Sell, Trade, Share, Support) are **topic
pages**: a display headline and one-line intro, then a run of numbered topics. Each topic
makes a short case for one part of the app and links to the **how-to articles** that
deliver it. The structure borrows from a help center's topic page (a two-column list of
article links, "See more" once a list runs long), but the styling is type and colour only:
big display headings in Roobert, big links in plain dark-mode link blue (`accent`,
`#03A9F4`) set in the body face at normal weight, and no dividers or arrow icons. Space
alone separates the sections. Two brand colours were tried and dropped before the blue:
the mark's pink (`#FF6DFA`) read as brand rather than as something to tap, because the
nav's mark is pink, and the condition lime (`#C8FF8F`) read as a status colour. A link
that looks like a link costs the reader nothing to learn. Each heading carries a lede and a reason, because on these pages
the heading has to sell the idea before it works as an index.

Built this way so far (2026-09-17): **Create** (*Follow what you play* / *Post it* / *Send
it to your people*) and **Collect** (*Scan it in* / *Sort it how you want* / *Know what
it's worth* / *Put a card up*). `StoreButtons` closes both pages.

## How it works

**Page → body.** `src/pages/pages.ts` is still the single list behind the nav links and the
routes. A page with real content registers its body component in `PAGE_BODIES`
(`src/app/AppLayout.tsx`, keyed by path), and `PageLayout` renders it under the headline.
That map is the same mechanism the legal pages use. Adding an entry changes no route,
order or params.

**Topic sections.** `src/pages/topics/TopicSection.tsx` takes an id, step number, title,
lede, optional "why" line and a list of guides. It shows six links (three rows of the grid)
and hides the rest behind "See more". Hidden links stay in the document with a `hidden` class,
not unrendered, so crawlers and find-in-page still see every guide.

**Hubs: one list.** `src/pages/topics/hubs.ts` holds the numbered sections of Create,
Collect and Trade and the guide slugs each links. `HubSections` renders them; each page's
`<Page>Copy.tsx` is just that plus its store buttons. The same list tells every guide where
its "Back to" link goes, so the two can't drift. Every guide lives at `/support/<slug>`, not
under the page that links to it; one guide (e.g. "How to share a collection") can sit on
several hubs. `support` is not a share-type slug, so `/support/:guide` can't collide with
`/{slug}/:referenceCode`, and the AASA file doesn't match it.

## Status

**Every hub link resolves** (2026-10-19). All guides are written, see
[[help-centre|Help centre]]. Hub copy was rewritten to match what the app does today:
Create 01 and 02, Collect hero and 01 to 03, and the whole of Trade (hero subhead plus six
sections). Links for features the app doesn't have yet (following, polls, collection
filters/sort/views, FairMarket, reviews) were taken off the hubs until they ship.

Collect's FairMarket section (*Know what it's worth*) was replaced by *Know your
conditions* while FairMarket is off.
