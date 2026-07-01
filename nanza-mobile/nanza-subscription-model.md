# Nanza Usage Caps (Freemium Foundation)

A freemium usage-cap system. No payment system yet — when a user hits a cap, the
feature is blocked and a consistent "you've hit your limit" bottom sheet appears.
A paid tier that lifts caps comes later.

## Caps

All caps count **lifetime creations**. Deletions do **not** decrement the count
(prevents gaming: delete-one / create-one to stay under). The counter only goes up.

| Feature              | Cap   | Counted event                          |
| -------------------- | ----- | -------------------------------------- |
| Listings created     | 100   | Each `POST /listing` success           |
| Bids created         | 100   | Each `POST /bid` success               |
| Messages sent        | 2500  | Each message send success              |
| Collections created  | 50    | Each `POST /list` of type COLLECTION   |
| Scans run            | 500   | Each scan success                      |

Explicitly **not** capped:
- **Share** — no cap. (Sharing is the native OS share sheet, no backend call; out of scope.)
- **Items per collection** — unlimited cards per collection.
- **Transactions** — no cap.

### Scan: two guards, both kept
Scan has two separate concerns; keep both:

1. **Pending-items guard (KEEP AS-IS).** `enforceScanItemLimit` (nanza-api
   `src/validation/scanItem.ts`) caps an account at **20 pending scan items per type**
   (COLLECTION vs LISTING counted separately), throwing 409. This is a queue guard so
   users convert/clear scans before scanning more. Unchanged.

2. **Lifetime scan cap (NEW, additive).** Add a **lifetime cap of 500 total scans** per
   account. After 500, the user hits the cap prompt (future: non-freemium upgrade).

   Note: `enforceScanRateLimit` / `DAILY_SCAN_LIMIT` (the rolling 50-per-24h logic in
   `src/services/scan.ts`) is **dead code** — never called anywhere. Repurpose
   `Account.scanCount` as the lifetime counter: drop the reset logic and
   `scanCountResetAt` usage, increment `scanCount` on each successful scan, and reject
   once it reaches 500. Counted at the `/scan/cards` success path.

## Enforcement

- **Backend-enforced** (nanza-api). The API is the source of truth and rejects the
  capped action when the lifetime count is reached.
- A per-account lifetime counter is incremented on each successful creation and never
  decremented.
- When the cap is reached, the API returns a recognizable cap response (e.g. HTTP 403
  with a stable error code per feature) so the client can show the right copy.

### Schema changes (nanza-api `Account`)
Follows the existing precedent on `Account`: `scanCount` / `scanCountResetAt` are
already documented as backend-managed counters that **must not be writable from any
user-facing endpoint**. The new counters follow the same rule.

Add to `model Account`:
- `listingsCreatedCount     Int @default(0)`
- `bidsCreatedCount         Int @default(0)`
- `messagesSentCount        Int @default(0)`
- `collectionsCreatedCount  Int @default(0)`

Reuse the existing `scanCount` as the lifetime scan counter (no new field); stop
resetting it and stop using `scanCountResetAt` for scan limiting.

All counters are `{ increment: 1 }` on successful creation, never decremented, never
settable from request bodies.

### Increment + check points (nanza-api)
Each create path checks the lifetime counter against the configured limit BEFORE
committing; on success, increments. (Prefer a small shared helper, e.g.
`enforceLifetimeCap(accountId, feature)`, mirroring `enforceScanRateLimit`.)
- Listing create → `listingsCreatedCount`
- Bid create → `bidsCreatedCount`
- Message send → `messagesSentCount`
- List create where `type === 'COLLECTION'` → `collectionsCreatedCount`
- Scan → `scanCount` (replaces `enforceScanRateLimit`)

## Cap values are dynamic / configurable

The numbers above are **not hardcoded** — they must be adjustable without a code
deploy, since we'll want to tune them (and later raise/remove them per paid tier).
Store the limits in config the backend reads at validation time (env vars, a config
table, or a per-tier limits record), keyed by feature. Today there is one global
("free") tier; the structure should allow per-account/per-tier overrides later so the
paid tier just supplies different numbers against the same enforcement path.

## Client UX (nanza-mobile)

- On a cap response, show the existing bottom-sheet confirmation
  (`ConfirmSheetContent` via `useModalStack().pushModal`) with copy like
  "You've reached your limit" for that feature, and a single dismiss action.
- The blocked action does not proceed.
- One consistent component/helper handles the cap sheet across all features.

### Relevant client call sites
- Listing create: `src/components/listings/CreateListing/index.tsx` → `listingService.create`
- Bid create: `src/components/bids/CreateBid/index.tsx` → `bidService.create`
- Message send: `src/screens/messages/MessagesScreen/data/mutations.ts` (`useSendMessage`) → `messageService.send`
- Collection create: `src/components/lists/CreateCollection/index.tsx` → `listService.create`

## Future
- Paid tier that raises or removes caps.
- Payment system wired to cap responses.
