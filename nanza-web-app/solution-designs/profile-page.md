---
tags: [nanza-web-app, solution-design, profile, figma]
---

# Profile page — Solution Design

## Overview

`nanza.app/<username>` is a read-only mirror of the mobile profile: a hero, then
**Posts / Collections / Trades** as child routes of `ProfileLayout`
(`src/app/ProfilePage/`). It wears the public site's dark theme, so a creator's bio
link reads as Nanza's own site.

The three tab views were restyled from Figma on 2026-10-12 (frames
`Profile-posts-desktop`, `Profile-collections-desktop`, `Profile-trades-desktop`,
read over the Dev Mode MCP — see [[design-sync]]). The collection screen followed the same day; the
other detail screens have not been redesigned yet.

## How it works

**One call decides the page.** `GET /user/:username/page` confirms the name and
returns the hero data plus per-tab counts. A tab with nothing in it is not offered,
and the toggle itself only appears when at least two kinds of content exist — one
kind, or none, shows no tabs at all,
and the base URL falls through to the first tab that has something. Each tab fetches
its own list.

**The visual language is mobile's, not the marketing site's.** The type is Roobert
and the colours are mobile palette values (`#2F333C` border, `#F0EBE6` text,
`#909090` secondary, `#FF6DFA` action), written as literal hexes so the capture binds
them to Figma variables. `RIM` in `components/Primitives.tsx` is the shared border;
applied top-and-bottom only it is mobile's glass rim (collection cards, social chips).

**A 600px column under a banner that sinks into the page.** The banner sits at the
very top of the window, behind the nav, 794px wide. It is two layers on purpose
(`ProfileBanner`): the image, with its left and right fades, is `fixed`; the bottom
fade is the page's own ground — clear over the banner, page colour from its foot to
the end of the page — and scrolls with the content. On scroll the black rises over an
image that stays put. Everything else on the page is positioned above both layers.
The avatar stands in the banner's lower half, its foot on the banner's; the column's top padding is what places
it, so that number and the banner height move together.

**The nav is the logo and nothing else.** `Navigation`'s `minimal` mode drops the site
menu, Support, the app buttons and the hamburger, and also the bar's fill on scroll —
a band of page colour would cut across the banner fading out beneath it. Below `lg` it
scrolls off with the page and comes back pinned on scroll up, the tab bar's rule, so the
two come and go together; inside a detail sheet it follows the sheet's own scroll. It
also carries the detail screens' way-back button on a phone, moved in step with the
sheet as it rises and drops. The QR card
(pinned bottom-right, as on the homepage) is the page's route to the app
on desktop; below `lg` there is none.

**Tabs are mobile's segmented toggle, resting under the bio.** Since 2026-10-23 they
follow the Figma `rangeTrack` (node 2758:88484): the column's full width in equal
segments, a `#010101` track with a 1px `glassBorder` rim (`#FFFFFF1C`), the selected
segment filled with that same glass, labels Roobert SemiBold 13/14 in `secondaryText`
(`#BDC9D3`), selected in `#F8F6F4`. The motion below is unchanged. The glass
description that follows is the earlier look; the opaque fill now hides the lens. One
clear but dark-tinted glass track (backdrop dimmed and a 42% black tint, so white
labels hold over anything, a card's white text box included): the page behind it bent by an SVG
displacement filter (Chromium only; a 2px blur elsewhere), a hairline rim, labels
carrying their own shadow for contrast. The selected segment is a deep dimple in the same
clear glass — shadow under the top lip, light on the bottom lip — marked by its
pressed-in shape and a full-white label, not by a fill. (A liquid-glass pill with a
gradient rim and hover sheen was tried on 2026-10-14 and dropped.) It travels between tabs without overshooting and sets off slightly
squashed, springing back to shape with one soft rebound; the squash is anchored so
it never bulges past the track's ends.

They **stay under the bio and come back on request**: scrolling down they leave
with the page like any content; scrolling back up once their spot is out of view
they slide and fade in pinned at the top of the window (in the header bar from
`lg`, under it on a phone), and scrolling down while pinned sends them back out.
They unpin when their spot under the bio scrolls back down to the pin line — the
hero is on screen again by then — and since pinned and resting coincide at that
moment the hand-back is invisible. At rest it is an ordinary in-flow element and
only becomes `fixed` while pinned or hidden; its slot keeps its height so nothing
jumps (`useScrollAway` in `ProfileTabs.tsx`). Faking the resting state with a
permanently fixed element repositioned per frame lagged native scrolling by a frame
and read as stuck to the top — don't go back to that.

**One thumb card, two sizes.** `ThumbCard` is mobile's listing thumb: the image over
a dimmed, enlarged copy of itself, a direction pill (`WTB` pink, `WTS` green), and
name / price pill / quantity under a bottom shade. The Trades tab uses it full width;
a post embeds it at 400px in a horizontal row. Card art is shown as a card; a
seller's own photo is shown whole over a blurred copy. The card-art backdrop is
shifted down a quarter of the tile so it shows the illustration only — centred, the
enlarged card's text box and bottom frame draw a band across the foot of the tile.

**Posts are rows, not cards** — separated by a hairline, with the feed's footer
(reply and like counts, share glyph) shown but not interactive.

**Links in a post are preview rows, not URLs.** A LINK item renders with the card
row's lockup (`RowContent`): the page's Open Graph image in the thumbnail slot, its
OG title as the heading, and the bare hostname (no `www.`, no path) as the grey line.
The preview comes from the API's `/link-preview` — the endpoint mobile uses — because
the browser cannot read most sites' tags itself (no CORS). Until it answers, or if the
page publishes nothing, the hostname is the title and a link glyph fills the slot. In
the feed the row opens the URL in a new tab itself (a link cannot nest inside the
post's own link); on the post page it is a plain link.

**A collection's cover is its banner, else a mosaic** of its first four cards
(mobile's `CollectionThumb` rule). The card shows the count only; value was dropped
in the redesign.

**A collection opens as a grid under the same banner.** `CollectionScreen` heads
itself rather than taking the layout's back row: the row (chevron, avatar, username,
how long ago the collection changed) needs the collection's own date, and returns to
the Collections tab. Under it the name in the hero's type, the card count, and the
cards as a **list of rows** in the same 600px column: mobile's thumb tile with the
picture only (no pill, name or shade) at a quarter of the row, then the name over its
print number and set, with quantity at the far right. Rows are ruled off with
`DIVIDER` (`components/Primitives.tsx`) — the same hairline the posts feed uses, the
one divider on the profile. The six-column grid it replaced is archived, unrouted, as
`screens/CollectionScreenGrid.tsx`. A card only
enlarges; there is no entity page on the web.

**Detail screens are sheets over the tab they came from** (2026-10-23), as in the app,
where screens enter from the bottom. Opened from a tab, a post, collection, listing,
bid or lot rises over that tab; the tab stays drawn underneath, held at its scroll
position, and closing drops the sheet to reveal it. Closing is the way-back button or
the browser's Back. Nothing behind has to load or move, so the only thing in motion is
the sheet. A first version that slid the whole page in and out was dropped: it ran as
two moves (slide, then content landing) and on the way out the tab had to rebuild
behind it.

- **Tabs are drawn by path, not through the outlet.** `ProfileLayout`'s `TabArea`
  renders the tab, and the router's tab routes have no element. That keeps the same tab
  mounted under the sheet; the outlet carries only the detail screens.
- **The profile context is its own** (`profileContext.ts`, not the outlet context), for
  the same reason. A tab under a sheet gets `underlay`, so it doesn't write its head
  over the screen's.
- **The sheet scrolls on its own** (fixed, opaque, scroll contained), under the nav and
  the QR card. Lightboxes are portalled to the body so they still cover both.
- **It waits briefly for data before rising** (up to 450ms, until the calls in flight
  land), so it rises once instead of rising and then filling in.
- **Arriving straight on a detail address** (a shared link) there is nothing to rise
  from, so the screen is the page.
- **The way back** is a glass down-chevron, at the top right below `lg` and the top
  left from it. Over a tab it returns to that tab; on its own page it goes to the tab
  the screen belongs to. Reduced motion skips both animations.

**Search.** Each tab and detail screen sets its own head through `ProfilePage/seo.ts`:
`skibanks – Trades / Nanza`, with the bio as the description, a canonical in the stored
username casing (a wrong-case URL is also rewritten in the address bar), and `ProfilePage` /
`Person` JSON-LD. An unknown username is the not-found page. When the API fails for any other
reason the layout shows the error page instead, so an outage never looks like a missing profile.
See [[seo]].

## Known gaps

- **Post body face.** Mobile sets post copy in Balto; the web does not ship it, so the
  body inherits the public site's text face at the design's size.
- **Lots** have no direction pill in the design; they take a neutral `LOT` pill.
- **Listing quantity copy** ("N available") is ours — the frames only carry bid copy.
