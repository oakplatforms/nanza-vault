# Collections: "Add to collection" button + share parity

## Goal (from voice)
1. **Remove** the header `+` create action (and the "single-collection-with-plus" affordance) from the collections section.
2. **Add** a standard full-width button — plus icon + "Add to collection" — at the **end of the collections list** on the **own profile only** (scrolls with content, not pinned). Mirror the "Add a group" button pattern but using the bigger reusable `Button`.
3. **Share a collection** from the collection detail view in **both** the own profile and the read-only user profile via the ellipsis menu. (Investigation shows this already exists; we verify and, if missing, surface it.)

---

## Findings (DRY check)

- Own-profile collections list: [CollectionTab.tsx](../../src/screens/profile/ProfileScreen/tabs/CollectionTab.tsx) — header has `ProfileSectionHeader` with `actionIcon="add" actionVariant="circle"` (the `+`).
- Read-only collections list: [UserProfileCollection.tsx](../../src/screens/profile/UserProfileScreen/components/UserProfileCollection.tsx) — header already has **no** action; no `+` to remove there.
- Header component: [ProfileSectionHeader/index.tsx](../../src/components/profile/ProfileSectionHeader/index.tsx) — `actionIcon`/`onActionPress` are optional; omitting them renders the count label alone. **No change needed to this component.**
- Reusable button: [Button/index.tsx](../../src/components/global/Button/index.tsx) — supports `iconLeft={{ name: 'add', ... }}`, `size="xxl"`, `fullWidth`. The app's "standard white button" on this dark theme is `variant="secondary"` (light `secondary['50']` bg, dark text) — used in Payouts/CollectionPicker.
- Pattern reference: [CreateGroup/index.tsx](../../src/components/groups/CreateGroup/index.tsx) wraps `Button` in a submit row; [CollectionPickerScreen](../../src/screens/collections/CollectionPickerScreen/index.tsx) uses `size="xxl"`.
- Own-profile detail ellipsis already has Share: [CollectionDetailScreen/index.tsx:122-136](../../src/screens/collections/CollectionDetailScreen/index.tsx#L122-L136) — `canShare`, `shareLabel="Share collection"`.
- Read-only detail ellipsis already has Share: [UserCollectionDetailScreen/index.tsx:196-206](../../src/screens/collections/UserCollectionDetailScreen/index.tsx#L196-L206) — `canShare={!!referenceCode}`, `shareLabel="Share collection"`, gated on `referenceCode`.

> **No "single-collection-with-plus" inline component was found** outside the header `+`. The only `+` near the collections section is the header create button. Removing that satisfies "remove the plus + the one-collection thing."

---

## Plan

### Change 1 — Own profile: drop header `+`, add bottom button
File: `src/screens/profile/ProfileScreen/tabs/CollectionTab.tsx`
- Remove `actionIcon`/`actionVariant`/`onActionPress` from `ProfileSectionHeader` (keep the count label).
- After the mapped collection cards (inside `collectionCardsContent`, below the last card / pagination spinner), render a `Button`:
  - `title="Add to collection"`, `iconLeft={{ name: 'add', fill: <theme token>, size: theme.icons.sm }}`
  - `variant="secondary"`, `size="xxl"`, `fullWidth`
  - `onPress={handleCreate}` (reuses existing `navigation.navigate('CreateCollection')`)
- Wrap it in a spacer style so it sits below the last card with list-consistent spacing.
- **Confidence: 0.92** — additive, reuses existing `Button` + `handleCreate`. Rejected alternative: building a bespoke add-tile card (Confidence 0.6) — more code, diverges from the requested "standard button".

### Change 2 — Read-only profile: nothing to add
- `UserProfileCollection` already has no `+` and (correctly) gets **no** add button — you can't add to another user's collections. **No code change** beyond confirming the header stays action-less.
- **Confidence: 0.95.**

### Change 3 — Share in collection detail (both)
- Both detail screens already render `EditActionsMenu` with `Share collection`. The read-only one is gated on `referenceCode`. Verify a public collection returns `referenceCode` (it does — `listService.get` returns the full record, same source the own-profile share uses).
- **No code change expected.** If `referenceCode` is reliably present, the read-only ellipsis already shares. Will confirm at runtime; if the ellipsis is hidden because `referenceCode` is missing for the viewed list, escalate (backend include), not a frontend fix.
- **Confidence: 0.85** — pending the runtime confirmation the user asked for ("if that's there, I just don't see it").

### New style
- Add one class to `src/styles/components/profile.ts`, e.g. `collectionAddButtonSpacer` (top margin = `theme.spacing.lg` to match `collectionCardSpacer`), per the **no-inline-style** P0 rule. Possibly drop the now-unused `collectionAddCircleButton`? **No** — `ProfileSectionHeader` still supports the circle variant generically; leave it.

---

## Files touched
1. `src/screens/profile/ProfileScreen/tabs/CollectionTab.tsx` (header `+` removed, bottom button added)
2. `src/styles/components/profile.ts` (one spacer class)

(2 files — well under the 5-file escalation threshold. No nav/API/context/shared-util changes.)

## P0 compliance
- No inline styles → new class in `profile.ts`.
- No hardcoded colors/sizes → `add` icon fill uses a theme token; button uses theme-driven `secondary` variant + `xxl` size.
- No `any`, no custom DTOs, uses existing `Button`/`Icon`.
- No navigation changes.
