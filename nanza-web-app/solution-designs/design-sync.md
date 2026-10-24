---
tags: [nanza-web-app, solution-design, figma, design-system, tooling]
---

# Design sync — Solution Design

How this app's screens reach Figma, and how a Figma redesign reaches the code.
Set up for web on 2026-09-22; the mobile half predates it.

## Overview

Two directions, deliberately asymmetric.

| | Direction | Automated? |
| --- | --- | --- |
| Capture | code → Figma | Yes — a scripted Chrome walks the live app |
| Redesign | Figma → code | No — read over the Dev Mode MCP, implemented by hand |

**Code is canonical; a Figma edit is a proposal.** Figma's variable-write API is
Enterprise-only, so the return path was never going to be symmetric — and making
it manual is the point, not a limitation being worked around. A design arrives as
a diff someone reviews.

## Where it lives, and why it is split

The toolchain is a third repo, `nanza-figma-sync`, sitting beside this one and
`nanza-mobile` — one engine, one token source, thin per-project plugins. It is the
successor to the git-hidden `.figma/` folder that used to live inside nanza-mobile; its README is the team setup guide.

What stays in **this** repo is only what is a fact about *this app*:

- `figma-capture.json` — the routes, viewport tiers and waypoints to capture.
  It lives here because whoever edits `src/app/AppLayout.tsx` is the person who
  has to edit it. Nothing derives it from the router automatically; a new route
  that nobody adds here is simply never captured.
- `.mcp.json` — registers the Figma Dev Mode MCP on `127.0.0.1:3845`.
- `.claude/skills/figma-sync/` — the workflow, so a session in this repo knows
  the pipeline exists at all.

Tokens come from **nanza-mobile**'s token JSON, not from this app's Tailwind
config. A color here that is not in that palette does not bind to a Figma
variable; it lands in the sync's unmatched-hex report, which is the running
tokenization to-do list for the web app.

## Capture is scripted, not performed

nanza-mobile captures by hand: a dev-menu button in the simulator, because only
the running RN app can serialize its own tree. Web has no such constraint — the
runner drives its own Chrome (Playwright over installed system Chrome), navigates
each route, and walks the DOM plus the React fiber tree.

Three consequences worth stating, because they are what people assume otherwise:

- **A scene is a serialization, not a screenshot.** The browser engine never
  enters the picture, so "Chrome or Safari?" is not a question the pipeline has to
  answer. Safari-specific rendering is a visual-diff problem, separately.
- **Nobody names anything.** Screen names come from the manifest; the runner
  composes the filename and the Figma row.
- **It must run against `npm start`.** A production bundle minifies React
  component names, and the owner→component naming is the entire contract.

## The three axes

A Figma row is one screen name; a column is one color mode.

**Viewport tier** is part of the *name* (`Create-mobile`), never the variant —
the plugin reads `variant` as the color mode and uses it both to lay out columns
and to pin Figma's semantic-variable mode, so anything else in that field breaks
mode binding. Tiers are `desktop` 1512, `compact` 834, `mobile` 390: the site has
exactly one breakpoint, `lg`/1024px ([[compact-mode]]), so compact and mobile are
two samples of the same compact layout rather than two layouts.

**Height.** The walker frames everything in viewport space, so the window *is*
the frame and a naive capture is only ever one fold tall. Default behaviour
(`fullPage`) scrolls the page through once — firing any one-shot reveals — then
grows the window to the document height and captures the whole page as a single
frame. `/terms` at 390px comes to 13,530px of frame.

**Scroll position.** `fullPage` is wrong for a page animated *on* scrub rather
than revealed once, because no single height is correct. The landing page
([[landing-opening]]) is the only such page, and is captured at `waypoints` — its
section anchors — one row per section. The runner scrolls and then waits for the
position to stop moving, because Lenis drives ScrollTrigger and a scroll request
animates rather than jumps. `#footer` is not a waypoint: it settles at the same
maximum scroll as `#download`, so it would be a duplicate row.

**Row order** is the screen's position in the manifest, not capture order.
Re-capturing one screen never moves it; reordering the board is an edit to the
manifest.

## One color mode, measured

The board captures `light` only. Tailwind has no `darkMode` config, so it runs
the default `media` strategy, and `prefers-color-scheme` emulation does reach the
page — but almost nothing answers it. All 302 `dark:` classes sit in
`src/components/Tailwind`, and of those only `Avatar`'s outline ring reaches a
live route: a dark capture of a share page came back identical to the light one
apart from that ring. Capturing both would have doubled the board for one outline
color.

This is a fact about the app's *current live shape*, not a permanent one. The
marketplace routes are commented out in `AppLayout.tsx` ([[auth]]); when they
return they bring the dark-aware components with them, and dark becomes a real
second column again — one line in the manifest.

## The naming contract

This is what makes a captured frame worth more than a screenshot: it can be
diffed against JSX.

- Components are named for the React component that owns them.
- Every other node is `tag · className`, verbatim.
- Text nodes are named by their content.

So a redesigned frame maps back: `ComponentName` ↔ that component's file,
`tag · className` ↔ the element with exactly that className, a *new* layer named
`h2 · text-[20px] font-semibold` ↔ create that element with those classes.
Existing layers must never be renamed — the names are the diff keys. Moving and
reordering them is how a redesign is expressed.

## The return path

One screen frame per prompt; the Dev Mode MCP exposes the whole subtree from a
single node link. Read with `get_metadata` (absolute geometry, so gaps are
measurable), `get_screenshot` (intent) and `get_variable_defs` (token bindings);
`get_code` is a styling reference and never pasted into the repo. Then diff
against the current JSX and present the minimal edit list before editing.

Closing the loop is a re-capture: the synced row refreshes to the new code truth
while `· edit me` frames are never touched by a sync, so the two side by side is
the check that the implementation landed.

## Known gaps

- **Interaction states** — hover, an open menu, a mid-flow modal. The runner
  navigates; it does not act. The fallback is the console snippet in the sync
  repo, pasted into DevTools on the page you are looking at.
- **No authenticated captures**, and none needed: Cognito is stubbed to
  guest-only and every live route is public. Re-enabling the marketplace makes
  this a real gap, and the mobile pipeline's capture-profile model is the answer
  to copy.
- **`bulk`, `profile` and `tag`** share types are routed and render, but have no
  fixture reference code yet, so they are not captured.
- **No web `ui-review`.** nanza-mobile has a tiered P0/P1/P2 design-rule skill;
  this app's design rules are not written down as rules anywhere, so there is
  nothing for a web equivalent to verify against yet.
