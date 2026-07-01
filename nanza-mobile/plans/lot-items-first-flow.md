# Lot flow restructure: items-first (mirrors Collections)

## Goal
Make the lot-ITEMS screen the primary view for both editing and creating a lot, with the
DETAILS form (name, cover, where-to-post) demoted to a secondary step. Mirrors the
Collection items-first flow.

### EDIT a lot
- Tapping a lot (owner) opens the lot-ITEMS screen directly (currently opens EditBulk = the form).
- Items screen: search placeholder "Add to lot"; header ELLIPSIS with **Edit lot / Share lot / Delete lot**.
- "Edit lot" → opens the DETAILS form (name, cover). Reached this way it has **no ellipsis** of its own.

### CREATE a lot
- Starts on the lot-ITEMS screen ("Add to lot", add items).
- Bottom button reads **"Next"** (not "Done") → goes to the DETAILS form.
- DETAILS form's final action **creates the lot and returns to the Sell screen** (existing handlePublish behavior).

## Current architecture (confirmed)
- `BulkTab` (src/screens/bulk/CreateListingsScreen/BulkTab.tsx) has two variants:
  - `picker` — the items screen: search "Search cards to add to lot", member rows, summary, **"Done"** button (→ onClose). [lines 376-453]
  - `form` — the details screen: Name, Cover image, GroupTargeting, "Lot items (N)" row that pushes the picker, **"Create Lot"/"Update Lot"** button (→ handlePublish). [lines 454-542]
- Wrappers:
  - `CreateListingsScreen` — renders form (default) or picker (mode='picker'); titles "Create Lot"/"Lot items". Create entry: `navigate('CreateListings')`.
  - `EditBulkScreen` — renders `form` with `existingBulkId`; header ellipsis (Share/Delete) via EditActionsMenu. Edit entry: `navigate('EditBulk', { bulkListingId })` from FeedBulkCard/BulkSaleCard/FeedActionModal.
- Draft model: create mode get-or-creates a DRAFT bulk on mount; items attach to it server-side; handlePublish flips DRAFT→PUBLISHED. Edit uses the existing published id.
- handlePublish already: validates (≥2 items, name, cover), updates, invalidates, and `onClose()`s back to Sell (or Sell tab for scan-deferred). [lines 250-347]

## Approach (reuse the two existing variants; rewire entry + transitions)

### A. New entry screen: lot items as primary — `BulkItemsScreen`
Add a thin wrapper screen `src/screens/bulk/BulkItemsScreen/index.tsx` that renders `BulkTab variant="picker"` against a bulk id and owns the header ellipsis. Two modes via params:
- EDIT mode: `{ bulkListingId }` — picker against the published lot. Header ellipsis = Edit/Share/Delete.
  - Edit → `navigate('EditBulk', { bulkListingId })` (the existing details form).
  - Share/Delete → move the EditBulkScreen's existing share + delete logic here (guardedShare + ChoiceSheetContent delete).
- CREATE mode: `{ create: true }` — picker against a fresh draft (get-or-create). Bottom button "Next" → `navigate('CreateListings', { bulkId: draftId, startStep: 'details' })` (the details form). No ellipsis in create mode.

### B. BulkTab picker changes
- Placeholder: "Search cards to add to lot" → **"Add to lot"** (line 385).
- Bottom button: support a "Next" label + onNext handler in create mode vs "Done" in edit mode. Add props: `primaryActionLabel?: 'Done' | 'Next'` and keep `onClose` for Done; add `onPrimaryAction`. Picker needs to get-or-create the draft in create mode (today only the form does). Simplest: have `BulkItemsScreen` create the draft and pass `existingBulkId`, so BulkTab picker stays draft-agnostic.

### C. Edit entry rewiring
- FeedBulkCard (owner, line 146), BulkSaleCard (line 45), FeedActionModal (line 201): change `navigate('EditBulk', { bulkListingId })` → `navigate('BulkItems', { bulkListingId })` so tapping a lot opens items-first.
- `EditBulk` route stays (now reached only via the items ellipsis "Edit lot"). Remove its header ellipsis (Edit/Share/Delete now live on the items screen).

### D. Create entry rewiring
- Sell tab "create lot" (TradeScreen ~line 500) currently `navigate('CreateListings')` (form-first). Change → `navigate('BulkItems', { create: true })` (items-first). The "Next" button then goes to `CreateListings` with `startStep: 'details'` for the details form; handlePublish unchanged (creates + returns to Sell).
- Scan-deferred create (initialScanItemIds) — KEEP form-first as today (no items to pick pre-conversion; the form hides the items row). Route those straight to `CreateListings` with scan ids as now.

### E. Navigation
- Register `BulkItems` in AppNavigator + add `BulkItems: { bulkListingId?: string; create?: boolean }` to RootStackParamList.
- `CreateListings` gains real use of `startStep: 'details'` (type already allows it) to render the form as a standalone details step in create mode.

## Files
- NEW `src/screens/bulk/BulkItemsScreen/index.tsx` (items-first wrapper + ellipsis + draft create)
- EDIT `src/screens/bulk/CreateListingsScreen/BulkTab.tsx` (placeholder; Next vs Done action)
- EDIT `src/screens/bulk/CreateListingsScreen/index.tsx` (startStep details = details-only form in create; Next handler)
- EDIT `src/screens/bulk/EditBulkScreen/index.tsx` (remove ellipsis — now details-only)
- EDIT `src/components/feed/FeedBulkCard/index.tsx` (owner tap → BulkItems)
- EDIT `src/components/trade/BulkSaleCard/index.tsx` (tap → BulkItems)
- EDIT `src/components/feed/FeedActionModal/...` (edit action → BulkItems) [verify path]
- EDIT `src/screens/trade/TradeScreen/index.tsx` (create lot → BulkItems create)
- EDIT `src/navigation/AppNavigator.tsx` (register BulkItems)
- EDIT `src/types/navigation.ts` (BulkItems param)

~10 files (1 new). Navigation change — pre-approved direction.

## Open questions to confirm before building
1. EDIT delete keeps the existing "keep listings vs delete listings" choice sheet — moved to the items ellipsis. OK?
2. The details "Lot items (N)" row on the EDIT details form — keep it (lets you jump back to items) or remove it since items is now the primary screen? (Lean: remove on edit, keep on create's details step is N/A since create details has no back-to-items.)
3. Confirm BulkDetail (buyer view) is untouched — this is all the OWNER/seller flow.

## P0 compliance
- Styles in buckets; theme tokens; no any; DTOs from @oakplatforms/types. Nav changes pre-approved.

## Confidence: 0.8 (multi-screen + nav; reuses existing variants which lowers risk; main risk is the create draft+Next wiring and not breaking scan-deferred).
