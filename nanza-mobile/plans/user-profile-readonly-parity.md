# Plan: Read-Only User-Profile Parity

Bring the logged-in **ProfileScreen** experience to the **UserProfileScreen** (viewing someone else), in a read-only form. Three independent pieces.

---

## 1. Problem Analysis

**What & why**

1. **Header spacing.** The count-header below the Listings/Bids list (`X Listings` / `X Bids` row with the share icon) sits too tight. The user wants more breathing room beneath it — bump to **18px**.
   - Lives in `ProfileSectionHeader` → `collectionCountHeaderTight.marginBottom` = `theme.spacing.md` (14px) — [profile.ts:404-406](../../src/styles/components/profile.ts#L404-L406).

2. **Missing count + share on user profile.** On your **own** profile, the Listings/Bids tabs render a `ProfileSectionHeader` (count label + `share-2` action) — [ListingsTab.tsx:92](../../src/screens/profile/ProfileScreen/tabs/ListingsTab.tsx#L92), [BidsTab.tsx:73](../../src/screens/profile/ProfileScreen/tabs/BidsTab.tsx#L73). On a **user's** profile, `renderListingsTab` / `renderBidsTab` are inline and render **no header** — [UserProfileScreen.tsx:226-278](../../src/screens/profile/UserProfileScreen/index.tsx#L226-L278). The share should target the viewed user's profile reference code (sell/bid views), same as own profile.

3. **Collection parity (the big one).** Own profile shows a `CollectionTargetCard` grid → tap opens `CollectionDetail` (the editable Collect/search/add/save flow) — [CollectionTab.tsx:83-110](../../src/screens/profile/ProfileScreen/tabs/CollectionTab.tsx#L83-L110), [CollectionDetailScreen](../../src/screens/collections/CollectionDetailScreen/index.tsx). The user profile currently uses a different inline `UserCollection` list — [UserCollection.tsx](../../src/screens/profile/UserProfileScreen/components/UserCollection.tsx). The ask: **the user profile should show the same card grid**, and tapping a card opens a **read-only** detail: you can **search** the collection, but there is **no plus-circle/add, no edit, no save, no delete, and no edit-menu actions** (share is optional/allowed since these are public collections).

**Confidence: 0.93** — Pieces 1 & 2 are precisely located and low-risk. Piece 3 is well-understood but spans more surface (a new read-only detail path + a nav route). Evidence cited inline above.

---

## 2. Proposed Solution

- **Piece 1:** Add an `18` spacing value and point `collectionCountHeaderTight.marginBottom` at it. There is **no existing 18 token** (`md`=14, `lg`=24), so add `theme.spacing.smd = 18` (or reuse if one is added). Keep the change in the stylesheet bucket (P0 rule 1/16 satisfied).
- **Piece 2:** Reuse the existing `ProfileSectionHeader` component in `UserProfileScreen.renderListingsTab` / `renderBidsTab`. Compute `listingCount` / `bidCount` from the already-fetched `sellFeedData` / `bidsData` page totals (same expression the own-profile tabs use). Share via the **viewed user's** `profile.referenceCode` with `buildShareUrl(code, 'sell' | 'bid')`.
- **Piece 3:** Replace `UserCollection` usage on the collection tab with the **`CollectionTargetCard` grid** (an `accountId`-driven variant of `CollectionTab`'s body), and add a **read-only collection detail** that reuses the **same server-side collection search the propose-trade flow uses** — `listService.getCollection(accountId, '?search=…&page=…&limit=20&include=…')` (`GET /list/collection/{accountId}`) — [List.ts:11-12](../../src/services/api/List.ts#L11-L12), [TradeCardSelector.tsx:43-97](../../src/components/trade/TradeCardSelector/index.tsx#L43-L97). This is the **same endpoint `UserCollection` already calls** via `fetchUserCollection`, just with the `&search=` param added. Render with the read-only `EntityCardList` (`gestures={false}`). **Not** `CollectTab` (the editable add/scan/save experience).

**Confidence: 0.9** — Pieces 1 & 2 are near-certain reuse. Piece 3 now reuses an existing, proven search endpoint (server-side, paginated, debounced) rather than inventing client-side filtering; the only genuinely new surface is a thin screen + nav route.

---

## 3. Alternatives Considered

**Piece 3 — read-only detail implementation:**

- **Option 1 (Recommended):** New lightweight `UserCollectionDetailScreen` that renders a header + a **search input** + the read-only `EntityCardList` grid (reusing `UserCollection`'s data hook & card render). Nav route `UserCollectionDetail { listId, accountId }`. — **Confidence: 0.86** — Cleanest separation; zero risk to the editable Collect flow; matches "search but no add/edit/save."
- **Option 2:** Reuse `CollectionDetailScreen` + `CollectTab` with a new `readOnly` prop that hides add/save/scan and the `EditActionsMenu`. — **Confidence: 0.55** — Rejected: `CollectTab` is a large, stateful editable component ([CollectTab.tsx:72+](../../src/screens/trade/TradeScreen/CollectTab.tsx#L72)); threading `readOnly` through it risks the working own-profile flow (violates P2 "surgical / additive" and P0 rule 8 "don't change behavior of existing components" risk). Names the rejected alternative per CLAUDE.md.
- **Option 3:** Keep `UserCollection` as-is and just add a search bar to it (no card-grid landing, no detail navigation). — **Confidence: 0.50** — Rejected: user explicitly wants the **card grid** ("replace ... with our new collections component ... look the exact same") and a tap-in detail; this option doesn't deliver that.

**Piece 1 — the 18px value:**

- **Option 1 (Recommended):** Add `theme.spacing.smd = 18` token and reference it. — **Confidence: 0.82** — Honors P0 rule 5 (no hardcoded sizes) and keeps it a named token. (Naming TBD with user — `smd`/`md2`; trivial.)
- **Option 2:** Inline `marginBottom: 18` in the style class. — **Confidence: 0.30** — Rejected: violates P0 rule 5 (hardcoded size).

---

## 4. Implementation Steps

1. **Add spacing token + repoint header margin.** Add `18` to `dark.json` spacing and the `ThemeTokens` spacing type; set `collectionCountHeaderTight.marginBottom` to it. — **Confidence: 0.85** — Risks: token-name bikeshed; touching shared tokens (`dark.json`) — additive only, no existing value changed.
2. **User-profile Listings/Bids headers.** In `UserProfileScreen`, import `ProfileSectionHeader` + `buildShareUrl` + `guardedShare`; compute counts from existing page data; render the header above each list; wire share to the viewed user's `profile.referenceCode`. — **Confidence: 0.9** — Risks: ensure `referenceCode` exists on the user profile DTO (own profile reads it from `currentUser.account.profile.referenceCode`; confirm the user-profile fetch returns it — guard the share if absent).
3. **User-profile collection grid.** Render a `CollectionTargetCard` grid on the collection tab driven by `accountId` (public collections only), navigating to the new read-only detail on tap. Reuse the existing public-lists query already in `UserCollection`. — **Confidence: 0.82** — Risks: pagination wiring through the parent `PageLayout` `onEndReached` (mirror `CollectionTab`'s ref pattern).
4. **Read-only collection detail screen + route.** New `UserCollectionDetailScreen`: a header + a **debounced search input** (250ms, like `TradeCardSelector`) backed by `listService.getCollection(accountId, '?search=…&page=…')`, rendering the read-only `EntityCardList` (`gestures={false}`), with infinite-scroll pagination. **No** add/edit/save/delete; share allowed. Register route in `AppNavigator`. — **Confidence: 0.84** — Risks: **navigation edit (P0 rule 7)** — see escalation. Search resolved: **server-side**, reusing the existing trade-collection-search endpoint (no client-side filtering, handles >20 items).
5. **Lint + typecheck.** `tsc`/lint clean, no `any`, no inline styles/colors/sizes. — **Confidence: 0.9**.

---

## 5. Files Modified

| File | Type | Est. lines |
|---|---|---|
| `src/styles/tokens/dark.json` | edit | +1 |
| `src/styles/utils/types.ts` (spacing type) | edit | +1 |
| `src/styles/components/profile.ts` | edit | ~1 |
| `src/screens/profile/UserProfileScreen/index.tsx` | edit | ~30 |
| `src/screens/profile/UserProfileScreen/components/UserProfileCollection.tsx` (grid variant) | **NEW** | ~90 |
| `src/screens/collections/UserCollectionDetailScreen/index.tsx` | **NEW** | ~120 |
| `src/styles/components/*` (search/detail classes as needed) | edit | ~20 |
| `src/navigation/AppNavigator.tsx` + `src/types/navigation.ts` | edit | +4 (**escalation**) |

Reused as-is (no change): `CollectionTargetCard`, `ProfileSectionHeader`, `EntityCardList`, `buildShareUrl`, `guardedShare`.

---

## 6. DRY Check Results

- **Searched for:** existing read-only collection rendering, count+share headers, collection card grid, `readOnly`/`gestures` flags, `lockedCollection`, `CollectionDetail` registration.
- **Found existing:** `ProfileSectionHeader` (count+share row), `CollectionTargetCard` (already presentational/read-only), `UserCollection` (read-only `EntityCardList`, `gestures={false}`, public-lists query, share handler), `buildShareUrl`/`guardedShare`. `CollectTab` exists but is the **editable** experience — not reused.
- **Decision:** Reuse `ProfileSectionHeader`, `CollectionTargetCard`, and `UserCollection`'s read-only render/query patterns; create two thin new files (grid variant + read-only detail) rather than overloading the editable `CollectTab`/`CollectionDetailScreen`. — **Confidence: 0.86**

---

## 7. Escalation Check

**Escalation needed? YES** — Step 4 adds a navigation route (P0 rule 7 / "navigation read-only").

> **ESCALATION REQUIRED**
>
> **Proposed change:** Register a new `UserCollectionDetail` screen + route param in `AppNavigator.tsx` / `types/navigation.ts`.
>
> **Cause for concern:** P0 rule 7 forbids editing `AppNavigator.tsx`. Risk is limited to an **additive** new route (no reorder, no existing param change).
>
> **Why this is the best option:** A read-only collection detail needs a destination screen; reusing the editable `CollectionDetail` would require risky `readOnly` plumbing through `CollectTab`. **Confidence: 0.80.**
>
> **Alternatives considered:** (1) Reuse `CollectionDetail` with a `readOnly` param — Confidence 0.55 — risks the live editable flow. (2) Inline expand/no-detail — Confidence 0.50 — doesn't meet the "tap in and search" requirement.
>
> **Risk mitigation:** Additive route only; no changes to existing screen order or params. Memory notes the user permits nav changes when cleaner ([nav-changes-permitted-when-cleaner]) — confirm before editing nav.

Pieces 1 & 2 are **not** escalations (style bucket + component reuse only).

---

## 8. Overall Confidence Score

**0.86** — High confidence on pieces 1 & 2 (pure reuse, surgical). Piece 3 is sound but carries the open items below.

**Resolved with user:**
- Search is **server-side**, reusing the propose-trade collection-search endpoint (`GET /list/collection/{accountId}?search=…`) — handles collections with >20 items.
- Nav-route addition is **approved**.

**Remaining minor uncertainties (resolve during build, no blockers):**
- Spacing token name for `18` (`smd`? `md2`?) — trivial.
- Whether the user-profile DTO exposes `profile.referenceCode` for the Listings/Bids share — guard the share if absent.
