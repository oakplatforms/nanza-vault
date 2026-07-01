---
tags: [nanza-mobile, solution-design, ui-fixes]
---

# UI / UX Fixes — Solution Design

## Overview

Cross-cutting UI consistency and lifecycle behaviors that don't belong to a single feature area:
the standardized "more" ellipsis rendering, and the on-load **"Update Available"** bottom sheet.
These are small, additive, and mostly about making shared primitives behave uniformly across
every screen.

## The "more" ellipsis — consistent size & weight

### The problem

The three-dot "more" glyph (`assets/icons/more.svg`) rendered inconsistently across screens —
the messages-thread ellipsis looked visibly **bigger** than elsewhere, and the own-profile vs
user-profile ellipsis looked **lighter/darker**. Two root causes:

1. **The glyph is non-square** — its viewBox is **16×3** (a wide, short horizontal ellipsis).
   Call sites sized it with numbers tuned for square icons, and via two different prop paths
   (`size`, which makes a square box, vs `width`, whose height defaults to 24). Because the glyph
   is 16:3, the box's **width** drives the visual dot size — so widths ranging 16→20 across call
   sites read as different sizes.
2. **Color divergence between the two Header branches** — the `rightIcons` default color differed
   (`secondary[50]` vs `secondary[300]`) between the own-profile and collapsing-header render
   paths, so identical sizes still looked different in weight.

### The fix (settled design)

- **Normalize `more` inside the `Icon` component**: when `name === 'more'`, derive height from
  width via the 16:3 ratio, so any width/size value yields a consistent ellipsis regardless of
  which prop path a call site used.
- **Every `more` call site uses one width**: `theme.icons.md - theme.spacing.min` (**18**), the
  already-most-common value — messages thread, feed cards (bid/bulk/listing), and the `Header`
  `rightIcons` triggers across BulkItems, CreateBid, ScanItemEdit, GroupDetail, CreateListing,
  CollectionDetail, UserCollectionDetail.
- **Color stays intentionally surface-specific** — dark dots on light cards
  (`secondary[800]` on EntityCard/EntityModal), light dots on dark surfaces (`secondary[50]` on
  feed cards + profile header), `text` on dark headers. Only the **accidental** header-branch
  divergence was removed: `UserProfileScreen` now passes an **explicit** `secondary[50]` so it no
  longer rides the differing Header-branch default. Color was **not** flattened to one value — the
  divergence was the bug, not the surface-appropriateness.

Result: every ellipsis renders at the same visual width/weight regardless of screen or render
path.

## "Update Available" bottom sheet

### What it does

On app **cold start**, if the installed app version is older than the latest released version, a
bottom sheet prompts the user to update:

> **A new version of Nanza is available** — New features and improvements are ready. Update to
> get the latest. **[ Update ]**

- **Update** → opens the App Store (iOS) / Play Store (Android) listing.
- **Dismissable** — backdrop tap / drag-down closes it.
- **Once per app load** — shows at most once per cold start; if dismissed it doesn't reappear
  until a fresh launch.

This reuses the usage-caps sheet stack wholesale: `useModalStack` + `ConfirmSheetContent`, so
**no new styles** are needed — the same themed component with different copy.

### The mechanism

This is **not** an error path. On cold start the app: (1) reads its own installed version via
`react-native-device-info`'s `getVersion()`; (2) reads the **latest** released version from the
backend; (3) compares with a tiny in-app semver comparator (`isOutdated`), and pushes the sheet
only if `installed < latest`.

- **Latest version comes from a dedicated `GET /app-version`** returning `{ latest, minimum? }`,
  called **once** on load (`useAppVersion`, `staleTime: Infinity`, no polling). A dedicated
  endpoint was chosen over piggybacking on the user GET because it fires for **everyone**,
  including guests on the splash/onboarding flow — the first nanza-api call is otherwise gated
  behind sign-in. The `{ latest, minimum? }` shape is an infra response, not a domain DTO (like
  `InboxCountResponse`), so a local type is appropriate.
- **The comparison logic ships inside the app**, so only builds from this release forward can be
  prompted — users on older builds without the check simply never see it, by design.
- **Trigger placement**: `useModalStack` only resolves inside `ModalStackProvider`, which lives in
  `AppNavigator`. So a headless `UpdateGate` component is mounted inside the navigator tree (next
  to `ModalStackRenderer`); it runs the check once per load via a `useRef` guard (new cold start =
  new instance = ref resets = checks again). This is an additive one-line `AppNavigator` mount, not
  a route/registration change — flagged and approved as such. Firing from `App.tsx`'s startup
  effect was rejected (`useModalStack` isn't in scope there).

### Files

`src/utils/version.ts` (compare util), `src/services/api/AppVersion.ts`, `src/hooks/useAppVersion.ts`,
`src/hooks/useUpdateSheet.tsx` (sibling of `useCapSheet`), `src/components/global/UpdateGate.tsx`
(headless trigger), a one-line `AppNavigator` mount, and the backend `GET /app-version` route.
Remaining operational to-dos: set the `LATEST_APP_VERSION_DEV/PROD` config values, fill the Apple
App Store numeric ID in the store-link helper (Android package `com.oakplatforms.nanza` is known),
and redeploy the API.

## Key decisions & rationale

- **Normalize the non-square icon at the `Icon` level.** `more` is the only non-square glyph these
  call sites size; deriving height from width inside `Icon` means every call site only needs a
  consistent width number, regardless of the `size` vs `width` prop path.
- **Fix the divergence, keep the intent.** Color is surface-dependent on purpose (dark dots on
  light cards, light on dark); only the accidental header-branch default mismatch was removed.
- **Dedicated `/app-version` over piggybacking.** It reaches guests, keeps app-version metadata
  out of domain payloads, and is one cheap cacheable call.
- **In-app version comparison, forward-only.** Ships the check inside the build so the prompt is
  self-consistent and can't misfire on pre-check builds.
- **Headless `UpdateGate` in the navigator tree.** The only place `useModalStack` resolves;
  additive and returns null, so no visual/layout impact.

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
