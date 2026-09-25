# Top-Level Overlay System (Search + Account Balance)

**Status:** PLAN — not started
**Date:** 2026-07-26
**Goal:** Replace the navigation-pushed **Search** and **Account Balance** screens with
reusable **top-level overlays** that toggle open/closed above the whole app (no screen
transition, keyboard/input auto-shown), each hosting its **own internal nested navigator**
for sub-screens (Search→Filter/Sort; Balance→Orders/Wallet/Selling/PayoutHistory). This is
the model already proven by the **collection search overlay** — generalize it.

---

## Why (the problem)

The in-collection search feels snappy because it is an **inline overlay** mounted over an
already-live screen with an `autoFocus` input — no navigation, nothing to wait for, keyboard
rises instantly. The global Search feels choppy because it is a **root-stack route** pushed
with a slide-up transition that (a) blocks first paint on `useFetchBrands` (full-screen
spinner), (b) mounts a heavy data component during the transition, and (c) never focuses its
input. The fix is architectural: make Search (and Balance) overlays, not routes.

---

## Target architecture

A new **OverlayHost** modeled on the existing `ModalStackProvider` / `ModalStackRenderer`
(already mounted at the root as a sibling of the navigator — `AppNavigator.tsx:797-798`):

```
OverlayProvider                       (new context: which overlay is open + its params)
  <Stack.Navigator> ... </Stack.Navigator>   (unchanged app)
  <OverlayHost />                      (new root sibling — renders active overlay above all)
```

- **`OverlayProvider`** (new, `src/contexts/OverlayContext.tsx`): holds
  `{ activeOverlay: 'search' | 'balance' | null, params }`, exposes
  `openOverlay(name, params)`, `closeOverlay()`, `updateOverlayParams(patch)`. Modeled on
  `ModalStackContext.tsx` (state + memoized value + `useOverlay()` hook).
- **`OverlayHost`** (new, `src/components/global/OverlayHost/index.tsx`): renders the active
  overlay in a `StyleSheet.absoluteFill` view with a Reanimated slide/fade in-out (reuse the
  entrance style from `BottomSheet`), plus manual Android hardware-back handling (close
  overlay → close its internal stack) — same pattern as `CollectionDetailScreen:76-82`.
- **Each overlay hosts its own `createStackNavigator`** (independent of the root navigator),
  exactly like `WalletNavigator` already does. Closing the overlay unmounts the whole nested
  stack. Sub-screen pushes (Search→Filter) happen *inside* the overlay's navigator.
- **Keyboard/input auto-show:** overlays reuse the collection pattern —
  `KeyboardAvoidingView` + `autoFocus` input + manual keyboard-visibility listeners
  (`CollectionSearchOverlay`).

**Key insight:** the sub-screen navigators already exist as components (`SearchScreen`,
`SearchFilterScreen`, `OrdersScreen`, `WalletNavigator`, `SellingScreen`, `PayoutHistory`).
We are re-hosting them under overlay-local navigators, not rewriting them.

---

## ⚠️ Constraint: `navigation/` is documented read-only

The vault's `architecture.md` marks `src/navigation/` **read-only — escalate instead**. This
refactor *must* touch `AppNavigator.tsx` (mount `OverlayProvider`/`OverlayHost`, later remove
Search/Balance routes) and the deep-link config. **This is a deliberate, escalated
exception** — call it out, don't silently edit. Fold the outcome into a new
`solution-designs/overlays.md` and update `architecture.md`'s navigation section when it lands.

---

## Phase 0 — Foundations (no behavior change yet)

1. `OverlayContext.tsx` + `useOverlay()` (copy shape from `ModalStackContext.tsx`).
2. `OverlayHost` component: absoluteFill, Reanimated in/out, backdrop, Android back handler,
   renders `null` when `activeOverlay === null`.
3. Mount `OverlayProvider` around the navigator and `<OverlayHost />` as a root sibling in
   `AppNavigator.tsx` (next to `ModalStackRenderer`). **[escalated nav edit]**
4. Ship inert (nothing opens it yet). Verify app builds + runs unchanged.

## Phase 1 — Search overlay (highest value, lowest risk — single choke point)

`navigateToSearch.ts` is the **single choke point**: nearly every Search entry point funnels
through it, and no file calls `navigate({name:'Search'})` directly except the helper itself.

1. Build `SearchOverlay` (`src/components/global/OverlayHost/SearchOverlay/…`): a
   `createStackNavigator` with two screens — `Search` (existing `SearchScreen`) and
   `SearchFilter` (existing `SearchFilterScreen`). Reuse the components as-is.
   - Fix the choppiness at the source while re-hosting: **drop the first-paint spinner**
     (don't gate on `useFetchBrands`; render shell immediately, load brands in background)
     and **`autoFocus` the input** on mount (collection pattern).
2. **Rewrite the body of `navigateToSearch(navigation, filters)`** to drive the overlay:
   - No overlay open → `openOverlay('search', { filters })`.
   - Overlay already open → `updateOverlayParams` to merge filters by key and pop the
     internal stack back to the Search root. **This reproduces the load-bearing "refine
     existing search vs open fresh + merge filters" behavior** (`navigateToSearch.ts:35-63`)
     with overlay state instead of `CommonActions`/`StackActions`.
   - Keep the `navigation` arg in the signature (call sites pass it) even if unused, to avoid
     touching ~11 call sites: `useStandardHeaderIcons`, `SearchFilterScreen` (Apply),
     `Favorites` CTA, `EntityCardBase`, `EntityTags`, `EntityCard/Cart`, `FeedBulkCard` (×2),
     `FeedBidCard`, `FeedListingCard`, `DetailTitleBlock`.
3. Internal Search→`SearchFilter` push (`SearchScreen:380`) targets the **overlay's**
   navigator, not root. Update that one call to use the overlay-local navigation.
4. Preserve deep link: keep `search` resolving — either map `linking` to
   `openOverlay('search')` or leave a thin redirect route. **[escalated nav edit]**
5. Remove root `Search` + `SearchFilter` Stack.Screens once the overlay path is verified.
   **[escalated nav edit]**
6. **Verify** end-to-end: open from header, from a tag tap (filter pre-applied), Apply
   filters round-trip, close, Android back, deep link. Compare feel vs collection search.

## Phase 2 — Account Balance overlay (bigger surface, more call sites)

Balance is **not one screen** — it's the Account/Payouts panel plus a cluster of routes
(`Wallet`, `Selling`, `Orders`, `PayoutHistory`), entered by **direct `navigate` calls with
no helper**, and `Wallet` carries a `{ returnOnComplete: true }` contract.

1. Build `BalanceOverlay`: a `createStackNavigator` with the balance root (Payouts/Account
   balance panel) + `Orders`, `Wallet` (already a nested navigator — nest it), `Selling`,
   `PayoutHistory`. Reuse existing screens.
2. Introduce a **`openBalanceOverlay(target?)` helper** (mirror of `navigateToSearch`) so
   future call sites have a choke point. Migrate the direct call sites to it:
   - `Wallet`: AccountScreen §pill, `PaymentGateSheetContent`, `EditBid` (×2),
     `ListingCard`, `CustomerGateModal`, `FeedBulkCard`.
   - `Selling`: AccountScreen §pill, `SellerGateModal`, `useShareActions`.
   - `Orders` / `PayoutHistory`: AccountScreen §pill, `PayoutsScreen` History pill.
3. **Preserve the `{ returnOnComplete: true }` "return to caller after finishing" contract** —
   overlay close must fire the same completion callback the pushed route did.
4. Deep links `wallet` / `orders` / `selling` → `openBalanceOverlay(target)`.
   **[escalated nav edit]**
5. Remove the corresponding root Stack.Screens after verification. **[escalated nav edit]**
6. **Verify:** each entry point, the returnOnComplete flows (bid/payment gates), Android back,
   deep links.

## Phase 3 — Point collection search at the shared overlay (optional, per user's ask)

User wants the collection search to *use the same overlay component*, just pointed at "my
collection" instead of Nanza/group results. Evaluate: the `SearchOverlay` gains a `scope`
param (`'global' | 'collection'` + collectionId). Only do this once Phases 1–2 are stable —
it's a consolidation, not a prerequisite.

## Phase 4 — Docs

- New `documentation/solution-designs/overlays.md`: the overlay-host model, the nested-nav
  contract, the `navigateToSearch`/`openBalanceOverlay` choke points, keyboard/autofocus rule.
- Update `architecture.md` navigation section (overlays now sit beside the navigator).
- Add the design to `INDEX.md`.
- Update memory `global-search-rework` / add an `overlay-system` memory.

---

## Load-bearing behaviors to NOT break

- **`navigateToSearch` "refine vs fresh + merge filters by key"** — the single most important
  contract to reproduce (`navigateToSearch.ts:35-63`).
- **`{ returnOnComplete: true }`** on Wallet entries (return to caller after finishing).
- **Deep links:** `search`, `wallet`, `orders`, `selling` must still resolve.
- **Android hardware back** closes the overlay's internal stack step, then the overlay — never
  pops the underlying app screen (collection pattern, `CollectionDetailScreen:76-82`).
- **Keyboard-on-open** via `autoFocus` + no first-paint spinner (the actual snappiness fix).

## Risks / open questions

- Overlay-local `createStackNavigator` nested inside a root `NavigationContainer`: confirm
  react-navigation v7 supports an independent navigator rendered as a root sibling (likely
  needs its own `NavigationContainer` or an independent nav ref). **Prototype this in Phase 0
  before committing.** ← biggest technical unknown.
- Deep-link + cold-start: opening `search`/`wallet` from a killed app must open the overlay
  after the base screen mounts.
- `gestureEnabled`/back-gesture interplay between overlay and underlying screen.
