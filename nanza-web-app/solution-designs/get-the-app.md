---
tags: [nanza-web-app, solution-design, landing, get-the-app]
---

# Get the app — Solution Design

## Overview

How a visitor on a laptop ends up with the app on their phone. Three pieces, one
address: the **`/get` route** that resolves to the right store, the **QR code** that
encodes it, and the **scan card** pinned in the corner of the homepage.

## How it works

**Two addresses, one job.** Both send a phone to its own store; the difference is who
counts the install.

`GET_APP_URL` (`src/constants.ts`) is `https://nanza.app/get`, answered by
`src/pages/GetTheApp.tsx`: on iOS it `location.replace`s to the App Store, on Android to
Play, and on desktop it falls through to a real page with both store buttons. `replace`,
not `assign`, because the visitor never saw the page and Back should return them to
whatever they scanned from. `/get` is routed outside the `pages.ts` list — a redirect
wearing a page, deliberately absent from the nav. It is ours, it is short, and it is the
one that can be said out loud and typed. `public/qr-get.svg` is its code.

`ONELINK_URL` is `https://onelink.to/auw2kc`, an **AppsFlyer OneLink**, and since
2026-09-21 it is what the homepage card actually scans to
(`public/onelink_to_auw2kc.svg`, exported from AppsFlyer under that name). It resolves the
platform the same way, but a scan that ends in an install is attributed to it — which a
redirect of our own cannot do.

Either way one code serves both platforms: printing two would make the reader do the
sorting, and a code aimed at one store is wrong half the time. Both URLs are absolute and
hard-coded, because a QR is scanned off other people's screens and cannot be relative, and
because nothing can read a URL back out of an SVG — the constant is the only place the app
knows where its own code points, and changing either means regenerating its file.

**The scan card** (`src/sections/QrOverlay.tsx`) is pinned bottom-**right** of the homepage and
of every profile page. It sat bottom-left until 2026-10-12 (the right corner is where browsers
park download bubbles and chat widgets land); the profile design placed it on the right, and
one position everywhere won over the argument for the left. Desktop only (`lg`), because a QR is a thing
you point another device at; phones get the store links in the header, the hero and the
footer. It is not a modal — no backdrop, no focus trap, nothing blocked underneath.

Dismissal is remembered in `localStorage` (`nanza:qr-dismissed`) per browser, not per
session: closing it is the reader saying they are not doing this now, and asking again next
visit is why cards like this are resented. Every storage access is wrapped — Safari in
private mode throws — and the failure mode is deliberately "the card shows anyway". The
dismiss button only appears on hover or keyboard focus, so the resting card is the code and
its line.

**Geometry follows X's scan card** (2026-09-21), which is the reference this was restyled
against: a 128px code in a ~168px card, stacked rather than side by side, with one line of
copy — *"Scan to get the app"* — above it instead of a heading and a subtitle. Ours reaches
that width as 8 + 12 + 128 + 12 + 8, because the white quiet zone the code needs is a panel
inside a dark card rather than the card itself, so the outer padding is halved to pay for
the inner one.

The panel stays even though the AppsFlyer art brings a white rounded field of its own: that
field carries **one** module of quiet zone where the spec asks for four, and the panel's
12px is what makes up the difference. White on white, so it costs nothing to look at and
buys the scan.

**The hover lift** is 4% on a 300ms `cubic-bezier(.22,.61,.36,1)` — the nav's reveal easing,
a fast start that settles, where a symmetric ease feels rubbery at this size. Scaled from
the centre rather than the pinned corner, so it grows into its own gutter instead of sliding
across the page. Nothing else moves: no shadow bloom, no colour shift. The card cannot be
clicked — the code is aimed at the reader's *other* device — so the hover exists only to say
it is a thing rather than part of the background, and anything more reads as a prompt to
click.

## Known gaps

**No logo in the code.** X's carries their mark in the middle. The OneLink code is 25
modules at **EC level M** (decoded from the file), which does not leave enough correction to
punch a hole in the middle of — that is how a QR stops scanning. It would have to be
re-exported at EC level H before the mark could go in.

**`public/qr-get.svg` is now unreferenced.** Nothing renders it since the card moved to the
OneLink code; it is kept because it is the code for `/get`, which still exists. Nothing
links to `/get` either — it is reachable by typing it, which is the point of it.
