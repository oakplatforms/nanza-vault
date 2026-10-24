---
tags: [nanza-web-app, solution-design, profile, sharing, join-sheet]
---

# Join sheet — Solution Design

## Overview

The phone web's one-time "join" prompt: a bottom sheet that rises once per browser session
over a profile (or share) page and offers **Join the chat**. It is the app's "Ready to join
in?" interstitial (nanza-mobile `AuthModal`) brought to the web, as a fixed gradient lockup
rather than a card, without the app's body copy or login link. It has the web's own copy and opens on its own, where the app's
sheet opens when a guest taps something that needs an account. Code: `src/app/JoinSheet/`.

## How it works

**Two variants, decided by the page under it when it opens.**

| | Affiliate | Generic |
|---|---|---|
| Above the headline | the affiliate's avatar, 48px | — |
| Headline | See more from {username} | Find your FaB community |
| Buttons | Open Nanza (no close; the headline tucks it away) | same |

**Affiliate or generic: whatever page it opens on.** If the sheet opens on an affiliate's
profile (`affiliateLink` set in `GET /user/:username/page`), it is that affiliate's sheet:
their avatar and name, and Join goes to their Airbridge link. Anywhere else (a regular
profile, a share page) it is the generic sheet. The profile page hands its own loaded data
to the sheet through `useJoinSheetPage`, so there is no second request and no validation step.

Nothing about the visitor's path is remembered: no entry page, no stored referrer. A visit
that goes through two affiliates' profiles gets each one's own sheet. The automatic showing
lands on whichever page the trigger fires on, and the tab reopens it as the current page's.
(The first version stuck to the entry page's affiliate via `sessionStorage` and took a
`?ref=` parameter. Both were dropped on 2026-10-21 in favour of this simpler rule.)

**When it opens: all of these hold**

- the window is below `lg` (1024px), the same breakpoint as Join and the nav;
- the visitor is signed out (`useSession().isSignedIn === false`, which is everyone while
  auth is the guest stub);
- the sheet has not opened by itself on this page yet in this page load. It is keyed by
  page (a profile's username, or a share page's path) and held in memory only: nothing is
  stored, so a reload is a fresh load and it opens again (Skylar, 2026-10-21);
- the current page opted in with `useJoinSheetPage`. Today that is a loaded profile page
  (never its not-found page) and a loaded share page. Phones never reach a share page (they
  are sent to the store), so on a real phone this means profiles;
- nothing modal is over the page: an `aria-modal` dialog (the card lightbox) or a locked
  body (the nav menu). The consent banner is not modal, and the prompt opens over it.

**It opens half a second after the page has loaded.** If something modal is up then, it
retries every half second until that closes. (It replaced a 10-second-on-site / one-screen-
of-scrolling trigger kept in `sessionStorage`.) An app action tapped on the page (Make
Offer, like, reply) opens it as well, as often as one is tapped (`trigger: action`).

**The way back: an arrow tab.** Once the sheet has had its showing on a page, a small tab stays on
the bottom edge, centred, on the same pages and widths. It is a
64×36px piece of the same card (glass fill, 1px rim, rounded top corners) with an up
arrow. The height is fixed and does not grow with `safe-area-inset-bottom`: with Safari's
toolbar collapsed that inset is about 34px, and padding by it nearly doubled the tab. The
tab's lower half sits under the home indicator, with the arrow in the clear top half. Tapping it reopens the sheet for the current page. The automatic showing still
happens once per page; the tab can reopen the sheet any number of times
(`join_sheet_shown` with `trigger: tab`; the tap itself is `ui_click` `join_sheet_tab`). It
sits at z-40, under the lightbox and the nav menu, and stays mounted while the sheet is
open so focus can return to it.

**Mounted once, opted into per page.** `<JoinSheet />` sits beside the consent banner at the
router, and opens shortly after each opted-in page has loaded. Pages opt in through a tiny external store
(`useJoinSheetPage` / `useJoinSheetHost`). The same split as `usePublicSite`: nothing hung
off the router reaches a page that didn't ask.

**Where the buttons go.** The web has no sign-up or login (auth is stubbed and its routes are
commented out), so **Join the chat** goes through
`joinTheChat`, the routing the nav's Join button used. On the affiliate sheet that is the
affiliate's Airbridge link. Airbridge opens the app if it is installed (where an existing
user logs in), otherwise the store, and credits the install to the creator, which is how
attribution carries through to sign-up. On the generic sheet it is the phone's store, or
`/get` on a narrow desktop window.

**Looks: the app's fixed-button lockup, not a card** (2026-10-23, after X's "See all
the replies" prompt). A band of page colour rises from the bottom edge and fades to clear
over its top 72px, so it covers what is under the button and blends into the page rather
than sitting on it as a panel. On the band, centred: the affiliate's avatar (affiliate
variant), the headline in Roobert SemiBold 24/28, and one full-width Open Nanza pill in
the trade screen's Make Offer style (52px, `#E9ECF0`, 17px SemiBold). There is no close
button: it is meant to be answered, so the headline is the quiet way out — tapping it
tucks the prompt down to its tab. The width is capped at 520px
for tablets. The card version (glass, 44px corners, backdrop, Keep browsing beside Join)
was dropped; only its bottom-edge tab survives, as the way to reopen.

**Not a modal.** No backdrop, no scroll lock, no focus trap, and it doesn't take focus when
it opens by itself, so the page stays live around it. Only the band's solid part takes
taps; the fade lets them through to the page. It sits at `z-56`: above the profile's detail
sheet, the QR card and the consent banner (it opens over the banner, which is not modal),
below the nav's menu and the image lightbox.

**Phone browser chrome**

- `viewport-fit=cover` in `index.html`; the lockup is `position: fixed` at the bottom with
  `padding-bottom: max(20px, env(safe-area-inset-bottom))`.
- `visualViewport` lifts it by however much of the layout viewport's bottom the toolbar
  covers (not while pinch-zoomed).

**Motion and access.** It rises 48px and fades in over 250ms ease-out; with
`prefers-reduced-motion` it only fades. It is `role="dialog"`, labelled by the headline.
The headline, Escape and a swipe down (past 72px, or a quick flick) all dismiss. The swipe uses
touch events, so the move can be cancelled and the drag belongs to the lockup, not the page.

**Analytics** ([[analytics]]): `join_sheet_shown` (`variant` = `affiliate` / `generic`, `trigger`, `affiliate`) and
`join_sheet_dismissed` (`variant`, `method`) are explicit. The two buttons and the tab are plain
`ui_click`s named by `data-track` (`join_sheet_join` / `_collapse` / `_tab`), carrying
`variant` from the card's `data-track-variant`. The GTM container's event regex and params
include them, so the container must be re-imported or edited to match before they reach GA.

## Decisions (2026-10-21)

- **No sign-up/login on the web**: Join goes to the app via `joinTheChat`.
- **Headline and buttons only**: the subtext ("Post, trade and track your collection") and
  "Already have an account?" were removed after the first device test. Someone who already
  has an account takes Join, which opens the app if it is installed.
- **The nav's Join button is gone**: on a phone the sheet (and its tab) is the only call
  to action on a profile, and on a desktop the QR card is.
- **The variant follows the current page**: an affiliate's profile shows their sheet,
  anything else the generic one. No entry-page memory, no `?ref`.
- **Profiles and share pages only**: the marketing pages, `/get` and the guides already
  carry their own app prompts.
- The app's `AuthModal` is untouched. It is a separate React Native component in another
  app, so the two sheets can never be on screen together.

## Open

- Not yet checked on physical iPhones and Android phones (only emulated, below).
