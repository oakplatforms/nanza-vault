---
tags: [nanza-web-app, solution-design, support, help-centre]
---

# Help centre — Solution Design

## Overview

`/support` is the help centre index and `/support/<slug>` is a **how-to guide**: a title,
a one-line promise, numbered steps, then where to go next. The hub pages (Create, Collect,
Trade) sell the idea and link the guides; the guides do the telling. See
[[content-pages|Content pages]] for the pages that link in.

There are seventy-odd guides, all the same shape, so a guide is **data rather than a page**:
it declares its content and one component renders every one of them. Written as free JSX
per guide, seventy of them would each be a fresh chance to set a heading at a different size
or number the steps a different way. The legal pages ([[content-pages|Privacy, Terms]])
deliberately go the other way: two one-off documents inside `LegalArticle`.

**Text only.** Guides carry no screenshots (decided 2026-10-19 with the support spec). A
screenshot goes stale every time the app changes; a sentence that names the screen and the
label to tap is one edit. Each step names the exact UI label in bold, and icons with no text
are named by shape (share icon, pencil icon), never by colour.

Content source: the *Nanza support guides: build spec (text only)*, checked against
nanza-mobile `aa5e6e75` (app 0.3.9). Its copy rules (no dashes, "tap" not "click", sentence
case, one action per step, no "marketplace") apply to every edit.

## How it works

**One data file.** `src/pages/support/guides.ts` holds every guide keyed by slug: title,
category, intro, `related` slugs, and a `body` written in the same plain format the spec
uses (`## heading`, `01.` steps, `-` bullets, `**label**`). `guide.ts` parses the body into
sections at load. Writing guides as near-verbatim spec text is the point: keeping the copy in
sync with the app is a text edit, not a JSX edit.

**Labels are derived.** Each guide's `labels` field is every `**label**` in its body. When
the app renames a button, search the labels (or just the file) for the old name to find
every guide that needs an edit. `appVersion` (default `CHECKED_APP_VERSION`, 0.3.9) records
what the steps were checked against and renders as "Last checked with Nanza …" at the foot.

**Slugs are types.** `GuideSlug` is the key set of the data, so hubs (`topics/hubs.ts`) and
navigation (`navigation.ts`) name guides by slug and a typo is a type error. Link text is
always the guide's own title, so a hub link and the heading it lands on can't disagree.

**Aliases.** `save-to-a-collection` renders `add-a-card-by-searching`'s content under its own
title, because `/create` links it by that title. A redirect would have broken the
link-text-matches-title rule.

**Navigation block (`navigation.ts`).** Every guide ends with Next step, Previous, Part of,
Related (max 3), and Back to, then the "Still stuck?" email block.
- *Journeys* (Getting started, Posting, Collecting, Buying, Selling, Selling in bulk,
  Running a group) give their members Next/Previous/"step n of m"; the last step hands on to
  the journey's `then`, if any.
- Guides outside a journey take Next from the `NEXT_STEP` table. No entry, no Next
  (`delete-your-account` on purpose).
- *Back to* is the first hub section that lists the guide (Create, then Collect, then Trade,
  in section order), e.g. "Back to Sell it on Trade" → `/trade#sell-it`. It's derived from
  `hubs.ts` rather than stored, so it can't point at a section that no longer links the guide.
  Guides no hub lists go back to their category shelf on `/support`.
- Every link goes through `publishedGuide()`, so a draft or missing slug is skipped rather
  than rendered dead.

**Anchors.** Hub sections (`/trade#sell-it`) and `/support` category shelves
(`/support#posting`) carry ids. `usePublicSite` scrolls to the hash on navigation instead of
to the top when one is present.

**The page (`GuidePage.tsx`).** Guides don't use `PageLayout`: that layout opens on a
display headline a quarter of the way down the screen, which suits a page selling something
and not someone mid-task with a phone in their other hand. The breadcrumb (Support /
Category, both links) sits above a smaller title, and the first step is on the first screen.
Steps are numbered `01`, `02` in the left gutter (restarting each section) in the same faint
tabular numerals `TopicSection` uses. The "On this page" rail is generated from the H2s and
shown on desktop only.

**Search.** Every guide gets its title, description, canonical and breadcrumb JSON-LD from
`guideSeo` (`src/pages/support/seo.ts`); see [[seo]]. The `save-to-a-collection` alias
(`canonicalOf`) points its canonical at the guide it repeats. Guides are listed in
`public/sitemap.xml` by hand.

**`draft`.** A draft guide is routed and reviewable at its real address but left off the
index, the hubs and every other guide's links.

**Adding a guide:** add an entry to `guides.ts`, add its slug to a hub section in
`topics/hubs.ts` if it belongs on one, give it a Next step in `navigation.ts` (journey or
`NEXT_STEP`), and add its URL to `public/sitemap.xml`. Route, index entry, search tags and
back-links follow.

## Status

**75 guides plus one alias, all published** (2026-10-19), every link on `/create`, `/collect`
and `/trade` resolves. `/trade` became a hub with six sections.

**On hold (no page, no hub link) until the feature ships:** follow a hero/set/card, manage
following, polls, scan into a collection, filter/sort/views inside a collection, FairMarket
prices and card value (`SHOW_FAIR_MARKET = false`), condition and value, reviews. When
FairMarket returns, `fairmarket-prices` and `check-a-cards-value` are first.

**Open:**
- "See more" on the hubs still expands in place (current markup). The spec also suggests it
  could link to the matching `/support#category`; hub sections span categories, so that was
  left as is.
- `what-people-see` describes link previews in one generic line; check a real Discord and
  Facebook preview and describe it.
