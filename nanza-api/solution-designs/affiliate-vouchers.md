---
tags: [nanza-api, nanza-admin, plan, affiliates, vouchers, attribution]
---

# Affiliate Vouchers

**Status:** **plan — API built 2026-10-17 (uncommitted; migration `20261017120000_affiliate_vouchers`
to run manually; types 0.1.113 published). Admin dashboard built 2026-10-17 (uncommitted). Fourth
tier (ten collection cards) built 2026-10-20 in API + admin, uncommitted; migration
`20261020120000_affiliate_collection_voucher` to run; types 0.1.120 to publish. QA next.**
Part two of
[[airbridge-affiliate-attribution|Airbridge affiliate attribution]]: that design writes
`Account.affiliateId` on a referred account; this one pays the affiliate for what the referred
account then does. Spans nanza-api (one model, three counters, four hooks, two admin endpoints) and
nanza-admin (one dashboard). Paying the money stays **manual**. When it lands, fold into
[[profile-types]] ("Affiliate vouchers" section) and retire this file.

## The ask (Skylar, 2026-10-17, voice)

We need a Voucher model. A voucher records three things: the **affiliate**, the **referred user**
and the **type**. Vouchers are created programmatically by three hooks, each on a *creation*:

1. the referred user's **fifth post** → `AFFILIATE_POST`
2. the referred user's **fifth sent offer, to five different recipients** → `AFFILIATE_OFFER`.
   Sent is enough — the offer does not need to be accepted, and the order under it does not need
   to complete. Buying (a listing offer) and selling (a bid offer) both count.
   *(Settled 2026-10-17 after several passes — "offers" → "orders" → "orders with distinct
   sellers" → back to **offers to distinct recipients**, which is what Skylar wanted tracked all
   along. An offer is Nanza's "I want to deal with you" action; orders are what fall out of it.)*
3. the referred user's **fifth listing** as a seller → `AFFILIATE_LISTING`
4. the referred user's **tenth card added to their collection** → `AFFILIATE_COLLECTION`
   *(added 2026-10-20, Skylar: "if a user that came in from an affiliate adds 10 items to their
   collection they get a fourth $5 payment")*

Each hook asks: has the user reached the tier? Do they have an `affiliateId`? If both, create the
voucher.
**Each type fires once per user, ever.** Deleting a listing and creating a fifth again must not
produce a second voucher. So an affiliate earns at most four vouchers per referred user — $5 each,
$20 if the user reaches all four tiers. *(Skylar's 2026-10-20 ask said "up to $15"; four $5 tiers
are $20, and the copy in the deck and the agreement says $20. Confirm before go-live.)*

nanza-admin gets a **Vouchers dashboard**: every voucher with its type, the referred user and the
affiliate, and an **accept / decline** control (a dropdown). The $5 goes to the affiliate by hand
for now; the system only needs to track vouchers so we know whom to pay for what.

## Decisions

- **Lifetime counters for posts and listings, not live counts.** "Fifth" means the fifth *ever
  created*, which is exactly what the existing backend-managed `Account.*CreatedCount` columns
  measure (freemium caps, `validation/usageCap.ts`): they increment on the success path of a
  create and **never decrement on delete**. `listingsCreatedCount` already exists. Add
  **`postsCreatedCount`** the same way (backend-managed, listed in
  `BACKEND_MANAGED_ACCOUNT_FIELDS`, stripped from every user-facing account write). Counting rows
  at hook time would re-fire after a delete; the counters cannot.
- **Offers: a live distinct count, no counter.** "Five offers to five different recipients" is
  not something a scalar counter can express. Sent offers are never hard-deleted (only
  never-sent `DRAFT`s are, in `cancelOffer`; everything else moves to a terminal status), so
  counting rows is safe here: at hook time, `count(distinct recipientId)` over the user's **sent**
  offers (`senderId = user`, `status not in (DRAFT, CANCELED)` — a send the sender withdrew does
  not count, decided 2026-10-17; `DECLINED` and `EXPIRED` do count, the sender did their part)
  `>= 5`. Five offers to one person is four short. Counter-offers are rows too, but a counter goes
  back to the same recipient, so distinctness already neutralises them — no `parentId` filter
  needed. Should later cancellations drop the distinct count under five, the voucher already
  issued stands (it is a row, not a recomputation).
- **"Sent" = the offer's `DRAFT → PENDING` transition** (`sendOffer`, `services/offer.ts`), which
  both `POST /offer` with `send: true` and `PUT /offer/:id/send` pass through. For a listing offer
  this is also the moment the buyer's card is authorised and the order goes `PENDING`; for a bid
  offer the order stays `CREATED` until the bidder accepts. Neither fact matters to the tier —
  the send is the action we reward.
- **Once per (user, type), enforced by the database.** `@@unique([accountId, type])` on `Voucher`.
  The hook creates and swallows the unique-violation (P2002). No flag column on `Account`, no
  "already hit this tier" toggle to keep in sync — the voucher row *is* the toggle.
- **The hook fires when the counter reaches the threshold (`>= 5`), not only `=== 5`.** With the
  unique constraint, `>= 5` is idempotent and survives a race or a counter that skipped a value;
  `=== 5` would silently lose the voucher if the fifth create's hook ever failed. Threshold is a
  constant (`AFFILIATE_VOUCHER_THRESHOLD = 5`), env-overridable like the caps.
- **Eligibility is read at hook time.** If `Account.affiliateId` is null when the fifth create
  happens, no voucher, and none later: attribution claimed after the fact does not back-fill.
  (`affiliateId` can only be set within 7 days of sign-up, so this is a narrow edge.)
- **What does not count:** offers the user *received*; cart checkouts (no offer row — the cart
  is being retired anyway, see [[offer-engine]]); drafts never sent.
- **What counts as a post:** any post that goes live — **root posts and replies alike**
  (decided 2026-10-17). Two moments make a post live: `POST /post` without `draft` (root or
  reply; replies cannot be drafted), and `PUT /post/:id/publish` (the one saved draft going
  `DRAFT → ACTIVE`). Saving a draft does not count, the same way it does not spend the existing
  total-post limit. `postsCreatedCount` increments at both moments.
- **What counts as a listing:** a single listing via `POST /listing`, **a scan converted to a
  listing** via `POST /scan-items/batch/convert` (each converted scan is one single — it lands
  in `listingsCreatedCount` like any other single and is cap-checked as a batch; added
  2026-10-19 after QA found scanned listings were invisible to the tier), **or a published bulk
  lot** (decided 2026-10-17). A lot counts as **one**, not as its child singles. Lots get their own
  counter, **`bulkListingsCreatedCount`**, incremented when a lot becomes `PUBLISHED` — on
  `POST /bulk-listing` (created published) or on the `PUT /bulk-listing/:id` that flips a draft
  to `PUBLISHED` (once; a re-save of a published lot does not count). The tier is reached when
  `listingsCreatedCount + bulkListingsCreatedCount >= 5`. A separate counter rather than folding
  lots into `listingsCreatedCount` so the freemium **listing cap** (100 singles) keeps meaning
  what it means today.
- **What counts as a collection card (2026-10-20):** a **new `EntityList` row** the user creates
  in one of their **`COLLECTION`-type lists**, counted by a lifetime counter
  **`Account.collectionItemsAddedCount`** (same backend-managed rule as the others; never
  decrements, so remove-and-re-add cannot re-fire). Row = card: adding the same card to two
  collections is two; bumping the quantity of a card already in the collection is zero; the
  `quantity` on the row is ignored. Four user actions count — `POST /entity-list`,
  `POST /entity-list/batch` (its `created` rows), `POST /entity-list/batch-by-entity` (created rows
  whose list is a `COLLECTION`), `POST /scan-items/batch/commit-collection` (rows
  `addEntitiesToList` newly created; it now returns that count), and cards seeded through
  `POST /list` `entityList.create` on a new `COLLECTION`. What does **not** count: `FAVORITE` /
  `CUSTOM` lists, and the automatic mirrors — a listing landing in the seller's collection and an
  order moving cards seller → buyer (`services/collection.ts`, [[data-sync]]). Those are side
  effects of other tiers, not the user curating a collection. Threshold **10**, its own constant
  (`AFFILIATE_VOUCHER_COLLECTION_THRESHOLD`, env-overridable) — `getAffiliateVoucherThreshold(type)`
  now takes the type; the other three stay at 5.
- **The voucher is created in the same request as the fifth create, after the counter
  increment, fail-safe.** It is a side effect, so a failure logs (`AFFILIATE_VOUCHER_ERROR`) and
  never fails the user's create. No EventBridge hop: the three call sites already own the
  success path and the counter, and a queue would only add a place for the voucher to get lost.
- **Payout is manual; the voucher carries its value.** The dashboard's accept / decline is the
  whole workflow — no `Payout` row, no Stripe transfer. The voucher does carry **`amount`**
  (`Decimal(10,2)`, default `5.00`, decided 2026-10-17) so the pay-run reads the number off the
  row and a later change of tier values never rewrites history. Set at creation from a constant
  (`AFFILIATE_VOUCHER_AMOUNT`, env-overridable); the admin does not edit it in this round.
- **Status lifecycle:** `PENDING` → `ACCEPTED` | `DECLINED`, admin-only, reversible (an admin can
  move it back). `reviewedAt` + `reviewedById` (Admin) record who decided.

## Data model

```prisma
enum VoucherType {
  AFFILIATE_POST
  AFFILIATE_OFFER
  AFFILIATE_LISTING
  AFFILIATE_COLLECTION
}

enum VoucherStatus {
  PENDING
  ACCEPTED
  DECLINED
}

model Voucher {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  type   VoucherType
  status VoucherStatus @default(PENDING)
  amount Decimal       @db.Decimal(10, 2)

  reviewedAt   DateTime?
  reviewedBy   Admin?    @relation(fields: [reviewedById], references: [id], onDelete: SetNull)
  reviewedById String?

  account     Account @relation("VoucherAccount", fields: [accountId], references: [id], onDelete: Cascade)
  accountId   String
  affiliate   Account @relation("VoucherAffiliate", fields: [affiliateId], references: [id], onDelete: Cascade)
  affiliateId String

  @@unique([accountId, type])
  @@index([affiliateId, status])
  @@index([status, createdAt])
}
```

- `account` = the referred user (who did the fifth thing); `affiliate` = who gets paid
  (`account.affiliateId` at hook time, copied onto the row so a later `SetNull` on the account's
  `affiliateId` does not orphan the voucher). `Account` gains `vouchers Voucher[]
  @relation("VoucherAccount")` and `affiliateVouchers Voucher[] @relation("VoucherAffiliate")`.
- `Account` gains `postsCreatedCount Int @default(0)` and `bulkListingsCreatedCount Int @default(0)`
  beside the existing counters (same section, same backend-managed rule), and, from 2026-10-20,
  `collectionItemsAddedCount Int @default(0)`. No counter for offers — see Decisions.
- Enums go in the enums block of `schema.prisma`, no inline comments (repo rule). Migration SQL
  written by hand into `prisma/migrations/`, run manually by Skylar (dev first, prod with the deploy).
- Types package: `VoucherDto`, the two enums, `AccountDto` gains the two counters → bump
  `@oakplatforms/types` (next free version after 0.1.112).

## Hooks

One service does the work, the call sites below invoke it:

```
services/affiliateVoucher.ts
  issueAffiliateVoucherIfEarned(accountId, type)   // writes — a service
    account = select affiliateId, postsCreatedCount, listingsCreatedCount, bulkListingsCreatedCount, collectionItemsAddedCount
    reached = POST:       postsCreatedCount >= THRESHOLD (5)
              LISTING:    listingsCreatedCount + bulkListingsCreatedCount >= THRESHOLD (5)
              OFFER:      count(distinct recipientId) of sent, non-canceled offers >= THRESHOLD (5)
              COLLECTION: collectionItemsAddedCount >= COLLECTION_THRESHOLD (10)
    if !affiliateId or !reached → return null
    try create Voucher { accountId, affiliateId, type, amount: AMOUNT }  → return it
    catch P2002 (already has this type) → return null
    catch anything → console.error('AFFILIATE_VOUCHER_ERROR', …) → return null   // never throws
```

| Hook | Where | Counter | Call |
|---|---|---|---|
| Listing (single) | `routers/listing.ts` success path, right after `incrementLifetimeCounter(accountId, 'listing')` | `listingsCreatedCount` (exists) | `issueAffiliateVoucherIfEarned(accountId, 'AFFILIATE_LISTING')` |
| Listing (scan convert) | `routers/scanItem.ts` `POST /scan-items/batch/convert` success path, after `incrementLifetimeCounter(accountId, 'listing', createdListings.length)`; `enforceLifetimeCap(accountId, 'listing', n)` runs before the convert | `listingsCreatedCount` (+N per batch) | `issueAffiliateVoucherIfEarned(accountId, 'AFFILIATE_LISTING')` once per batch |
| Listing (lot) | `routers/bulkListing.ts`: `POST /bulk-listing` success path, and the `PUT /bulk-listing/:id` branch where `publishing` is true and the lot was not already `PUBLISHED` | `bulkListingsCreatedCount` (new) — `incrementLifetimeCounter(accountId, 'bulkListing')` | `issueAffiliateVoucherIfEarned(accountId, 'AFFILIATE_LISTING')` |
| Offer | `services/offer.ts` `sendOffer`, after the `DRAFT → PENDING` transaction commits (both `POST /offer?send=true` and `PUT /offer/:id/send` land here) | none — live `count(distinct recipientId)` of the sender's sent offers | `issueAffiliateVoucherIfEarned(senderId, 'AFFILIATE_OFFER')` |
| Post | `POST /post` success path when not a draft (root **or reply**), and `PUT /post/:id/publish` success path | `postsCreatedCount` (new) — `incrementLifetimeCounter(accountId, 'post')` | `issueAffiliateVoucherIfEarned(accountId, 'AFFILIATE_POST')` |
| Collection card (add) | `routers/entityList.ts`: `POST /entity-list` (+1), `POST /entity-list/batch` (+created rows), `POST /entity-list/batch-by-entity` (+created rows in `COLLECTION` lists) — all via a local `countCollectionCardsAdded` helper that warns instead of throwing | `collectionItemsAddedCount` (new, 2026-10-20) — `incrementLifetimeCounter(accountId, 'collectionItem', n)` | `issueAffiliateVoucherIfEarned(accountId, 'AFFILIATE_COLLECTION')` |
| Collection card (scan) | `routers/scanItem.ts` `POST /scan-items/batch/commit-collection` after the commit transaction, `+N` where N is what `addEntitiesToList` newly created across the target lists | `collectionItemsAddedCount` | same |
| Collection card (seeded) | `routers/list.ts` `POST /list` when a user-owned `COLLECTION` is created with `entityList.create`, `+N` cards | `collectionItemsAddedCount` | same |

- `CappedFeature` gains `'post'` and `'bulkListing'` so the counter plumbing is reused (decided
  2026-10-17), and `'collectionItem'` (2026-10-20). None is capped by it: `enforceLifetimeCap` is
  simply not called for them (posts keep their existing live-count `validateTotalPostLimit`; lots
  and collection cards have no cap) — the counter is what we want, not the ceiling.
- The offer hook runs **after** the send transaction commits, not inside it: a voucher must
  never roll back a send (which, for a listing offer, includes the card authorisation), and the
  distinct count should see the offer it is counting.
- Validation vs services rule: the "has affiliate / reached threshold / already has voucher"
  checks are inside the service because they are read-then-write in one place; nothing in
  `validation/` is needed beyond the admin status-transition check below.

## Admin API (nanza-api)

- **`GET /vouchers`** — admin only (`validateRole(…, 'admin')`), paginated (`paginatePrisma`),
  filters `?status=`, `?type=`, `?affiliateId=`, `?accountId=`; includes
  `account.profile` and `affiliate.profile` (username, avatar) and `account.email` for the table.
  Default order `createdAt desc`.
- **`PUT /voucher/:id`** — admin only, body `{ status }`; `validation/voucher.ts` checks the
  status is `ACCEPTED` or `DECLINED` (or back to `PENDING`), the service sets `reviewedAt` /
  `reviewedById` from the admin. PUT, never PATCH (repo rule).
- No user-facing voucher endpoints in this round. *(An affiliate-facing "your referrals and
  vouchers" view in mobile is a later design.)*

## Admin dashboard (nanza-admin)

`src/app/Vouchers/` + `/vouchers` route + sidebar entry, mirroring `Posts` (table + filters +
pagination; `SimpleTable`, `Badge`, `PaginationControls`, one `data/fetchVouchers.ts` hook):

| Column | Shows |
|---|---|
| Created | date |
| Type | badge: Post / Offer / Listing |
| User | referred account's `@username` (fallback email) |
| Affiliate | affiliate's `@username` |
| Amount | `$5.00` from the row |
| Status | **dropdown** Pending / Accepted / Declined → `PUT /voucher/:id` → refetch |

Filters: status (default **Pending**), type, affiliate. A per-affiliate total of accepted vouchers
(= what to pay) is a nice later addition; for the first cut the filter by affiliate plus the
status column is the pay-run list.

## Guard rails

- Counters are backend-managed (`BACKEND_MANAGED_ACCOUNT_FIELDS`) → a client cannot inflate its
  way to a voucher.
- `@@unique([accountId, type])` → one voucher per tier per user, race-safe, delete-safe.
- Self-referral is already impossible (`affiliateId` cannot equal the account's own id), so an
  affiliate cannot voucher themselves. Sock-puppet accounts through one's own link remain a
  human review question — that is what the dashboard's **decline** is for.
- The hook never throws into the user's request.

## Note: offers vs orders

Nanza's `Offer` rows (`OfferType.LISTING|BID`, counter-offers via `parentId`) are the live
negotiation layer over orders from [[offer-engine|Offer Engine & Trade Removal]] (2026-08) — not
the retired card-for-card **trading** (`Trade*` tables), which is unrelated. Every purchase is
meant to go through an offer, so "five offers to five recipients" and "five orders with five
sellers" are nearly the same measure for a buyer. They differ at the edges, and the offer count
is the one Skylar wants: it also credits a **seller** who answers five bidders (no order exists
until the bidder accepts), and it ignores cart checkouts (no offer row; the cart is being retired).

## Decided (Skylar, 2026-10-17)

1. **Posts:** replies count, not just root posts.
2. **Listings:** bulk lots count, one per lot, via their own counter.
3. **Offers:** sender-canceled offers are excluded; declined and expired count.
4. **`amount` column:** yes — `Decimal(10,2)`, default `5.00`, set at creation.
5. **Counter plumbing:** reuse `CappedFeature` / `incrementLifetimeCounter` for posts and lots,
   with no cap enforced for them.
6. **Notifications:** none. No email, no in-app message when a voucher is created or accepted.

## As built (API, 2026-10-17)

- `prisma/schema.prisma`: `VoucherType`, `VoucherStatus` in the enums block; `Voucher` model after
  `Payout` (`amount Decimal(10,2)`, `reviewedAt`, `reviewedBy → Admin` via relation
  `VoucherReviewer`, `account` / `affiliate → Account` via `VoucherAccount` / `VoucherAffiliate`,
  `@@unique([accountId, type])`); `Account.postsCreatedCount` + `Account.bulkListingsCreatedCount`;
  `Admin.reviewedVouchers`. Migration `20261017120000_affiliate_vouchers` written by hand.
- `constants/affiliateVouchers.ts`: `getAffiliateVoucherThreshold()` (5, env
  `AFFILIATE_VOUCHER_THRESHOLD`) and `getAffiliateVoucherAmount()` (5, env `AFFILIATE_VOUCHER_AMOUNT`).
- `constants/usageCaps.ts` + `validation/usageCap.ts`: `CappedFeature` gains `'post'` and
  `'bulkListing'` (caps set to `MAX_SAFE_INTEGER`, never enforced); counters mapped, added to
  `BACKEND_MANAGED_ACCOUNT_FIELDS`.
- `services/affiliateVoucher.ts`: `issueAffiliateVoucherIfEarned(accountId, type)` (reads
  `affiliateId` + counters; OFFER tier counts `distinct recipientId` over sent, non-canceled
  offers; creates with `amount`; swallows P2002 / P2022; logs `AFFILIATE_VOUCHER_ISSUED` /
  `AFFILIATE_VOUCHER_ERROR`; never throws), `reviewVoucher(id, status, adminId)`, `voucherInclude`.
- `validation/voucher.ts`: status / type checks for the admin routes.
- Hooks: `routers/post.ts` `POST /post` (non-draft) and `PUT /post/:id/publish`;
  `routers/listing.ts` after the listing counter; `routers/scanItem.ts` batch convert after a
  `+N` listing counter bump (2026-10-19; `enforceLifetimeCap` / `incrementLifetimeCounter` gained a
  count argument for it); `routers/bulkListing.ts` `POST /bulk-listing` and
  the `PUT /bulk-listing/:id` publish flip (`existing.status !== 'PUBLISHED'`, `status` added to the
  select); `services/offer.ts` `sendOffer` after the send transaction.
- `routers/voucher.ts`: `GET /vouchers` (admin via `validateAdmin(adminId)`, paginated, filters
  `status` / `type` / `affiliateId` / `accountId`) and `PUT /voucher/:id` (`{ adminId, status }`);
  mounted in `all_routes.ts` after reports. Types regenerated: `VoucherDto`, `AccountDto` counters.

## As built (API, fourth tier, 2026-10-20)

- `prisma/schema.prisma`: `VoucherType.AFFILIATE_COLLECTION`; `Account.collectionItemsAddedCount`.
  Migration `20261020120000_affiliate_collection_voucher` written by hand (`ALTER TYPE … ADD VALUE`
  + the column).
- `constants/affiliateVouchers.ts`: `getAffiliateVoucherThreshold(type)` — 10 for
  `AFFILIATE_COLLECTION` (env `AFFILIATE_VOUCHER_COLLECTION_THRESHOLD`), 5 otherwise.
- `constants/usageCaps.ts` + `validation/usageCap.ts`: `CappedFeature` gains `'collectionItem'`
  (uncapped), mapped to `collectionItemsAddedCount`, added to `BACKEND_MANAGED_ACCOUNT_FIELDS`.
- `services/collection.ts`: `addEntitiesToList` returns the number of rows it newly created.
- `services/affiliateVoucher.ts`: reads the new counter; `AFFILIATE_COLLECTION` tier.
- Hooks: `routers/entityList.ts` (three add endpoints, `COLLECTION` lists only, via
  `countCollectionCardsAdded`), `routers/scanItem.ts` commit-collection, `routers/list.ts` seeded
  cards on a new collection. `routers/voucher.ts` OpenAPI enum updated → types 0.1.120.
- Admin: `VOUCHER_TYPE_LABELS.AFFILIATE_COLLECTION = 'Collection'`, amber badge. Needs types
  0.1.120 installed (`npm run types:update`) before it typechecks.

## As built (admin, 2026-10-17)

`nanza-admin`: `src/app/Vouchers/index.tsx` + `data/fetchVouchers.ts`, `services/api/Voucher.ts`
(`list`, `update` via **PUT** `/voucher/:id { adminId, status }`; `VoucherStatus` / `VoucherType`
derived from `VoucherDto`), route `/vouchers`, "Vouchers" in `TopNavbar`. `types/index.ts` re-exports
`VoucherDto` and adds `VoucherRowDto` (the hydrated party + reviewer shape `voucherInclude` returns).
Page: filters status (default **Pending**) and type; the **affiliate cell is a button** that narrows
the list to that affiliate (a removable chip shows the active one) — the pay-run view; columns
Created / Type badge / User / Affiliate / Amount / Status **dropdown** (Pending · Accepted · Declined,
colour-coded, disabled while saving, refetch after) / Reviewed (when · by whom). Requires
`@oakplatforms/types ^0.1.113`.

## Build order

1. ✅ API (2026-10-17): schema (enums + model with `amount` + `postsCreatedCount` + `bulkListingsCreatedCount`)
   → migration SQL → `services/affiliateVoucher.ts` → hooks (post create + publish, single
   listing, lot create + publish, offer send) → `GET /vouchers`, `PUT /voucher/:id` +
   `validation/voucher.ts` → types bump.
2. ✅ Admin (2026-10-17): `Vouchers` page, route, nav entry, service module, hook.
3. QA on dev with a referred account: five posts (mix roots, replies, one published draft) /
   five sent offers to five recipients / three singles + two lots → three PENDING vouchers at
   $5.00; five offers to one recipient → no offer voucher; a canceled offer does not count;
   delete + recreate a listing → still three; accept/decline from the dashboard.
4. ✅ Fourth tier (2026-10-20): schema + migration → threshold per type → counter plumbing →
   `addEntitiesToList` created-count → hooks (entity-list ×3, scan commit, seeded list) → types
   0.1.120 → admin label + badge.
5. QA the fourth tier: add ten cards by hand across one or two collections (mix single adds,
   batch, scan commit) → one PENDING Collection voucher; bump a quantity → no change; remove and
   re-add a card → still one; a card arriving via a listing or an order → no change; a FAVORITE
   list add → no change.
   - 2026-10-19: posts fired on the 5th; listings needed a 6th because one had been created via
     scan convert, which bypassed the counter. Fixed (hook above); re-test the listing tier with
     a scanned listing in the mix.
