---
tags: [nanza-api, nanza-admin, nanza-mobile, plan, affiliates, agreements]
---

# Agreements — affiliate terms before the link

**Status:** **plan — API built + committed 2026-10-17 (`28ba43e`), types published as 0.1.116 and installed in admin + mobile (local shims removed); admin + mobile built 2026-10-17, uncommitted; pending migration `20261019120000_agreements` and QA.** Part three of the
affiliate loop: [[airbridge-affiliate-attribution|attribution]] mints the link,
[[affiliate-vouchers|vouchers]] pay for what referrals do, and this design makes the affiliate
**accept the terms first**. Spans nanza-api (two models, one flag, one admin resource, one accept
endpoint, one gate), nanza-admin (an Agreements page) and nanza-mobile (an agreement screen +
the Settings gate). When it lands, fold into [[profile-types]] ("Affiliate agreement" section)
and retire this file.

## The ask (Skylar, 2026-10-17, voice)

Affiliates can already see and copy their affiliate link in Account → Settings. Before they get
it, they must **agree to a document**. The document is written in nanza-admin — "forms and
agreements" — as a model with a copy field, and whatever is in that field is what the app shows
the affiliate to accept. When they accept, we record **who** (profile) and **when** (timestamp)
against the agreement, and a generic **has-signed** value on the profile flips to true. The API
must refuse the affiliate link until that value is true, and the app must not show the link
either.

Terms are **immutable once anyone has accepted**: the API rejects edits after the first
acceptance, and the admin form disables the fields. To change the terms you create a new
agreement. For v1, one acceptance of the first agreement is enough — no re-acceptance logic yet.

## Design in one paragraph

An `Agreement` is a typed document (`AgreementType.AFFILIATE` today) with a `displayName` (the
title) and a `body` (the copy), created and edited in nanza-admin until the first acceptance locks
its text. An `AgreementAcceptance` is one row per (agreement, profile) with an `acceptedAt`
stamp. Accepting runs in one transaction: create the acceptance, set `Profile.hasSigned = true`.
The affiliate-link mint (`GET /profile/:id/affiliate-link`) and the public page's
`affiliateLink` both require `hasSigned`. Mobile reads `hasSigned` off the session profile: an
affiliate who has not signed sees an **Affiliate agreement** shelf in Settings instead of the
link; it opens a screen showing the current agreement with one **I agree** button.

## Decisions

- **Two models, not one.** The document and the acceptance are different lifecycles: the document
  is authored once and locked; acceptances accumulate. Folding `acceptedAt` onto `Profile` would
  lose *which* agreement was accepted — the thing we need when terms change.
- **`Profile.hasSigned` is a denormalized flag**, named generically as asked. It is what the gate
  and the app read; the acceptance rows are the audit trail. v1 sets it on any acceptance; when a
  second agreement type or re-acceptance arrives, the flag's meaning narrows to "signed the
  current AFFILIATE agreement" and a service recomputes it.
- **Acceptance hangs off `Profile`, not `Account`.** `ProfileType` and the affiliate link both live
  on the profile, so the gate reads one row. `Account` is reachable through `profile.accountId`.
- **Current agreement = newest ACTIVE row of the type.** Retiring one (status `INACTIVE`/`DELETED`)
  is allowed after acceptances; editing text is not. Creating a new ACTIVE row of the same type
  makes it current; old acceptances still count for v1 (hasSigned stays true).
- **Locking is enforced server-side** (409 when `displayName` or `body` changes and
  `_count.acceptances > 0`), and mirrored client-side (fields disabled, note shown). Status may
  still change.
- **The gate is server-side first.** `assertAffiliateLinkable` gains the `hasSigned` check
  (403, "Accept the affiliate agreement first"). Admin callers bypass it (they are reading, not
  sharing). `GET /user/:username/page` returns `affiliateLink: null` until signed, so the web Join
  button falls back to the store rather than attributing to an unsigned affiliate.
- **`name` + `displayName`** per the standard fields; `name` is a server-derived slug of
  `displayName` (unique per type is not enforced — two agreements can share a title across time).
- **Body is plain text** (`@db.Text`), rendered with the legal typography styles. Rich text is a
  later concern; the admin field is a `Textarea`.

## Data model

```prisma
enum AgreementType {
  AFFILIATE
}

model Agreement {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  status    Status   @default(ACTIVE)

  type        AgreementType
  name        String
  displayName String
  body        String        @db.Text

  createdBy        Admin?  @relation(name: "CreatedBy", fields: [createdById], references: [id], onDelete: SetNull)
  createdById      String?
  lastModifiedBy   Admin?  @relation(name: "LastModifiedBy", fields: [lastModifiedById], references: [id], onDelete: SetNull)
  lastModifiedById String?

  acceptances AgreementAcceptance[]

  @@index([type, status, createdAt])
}

model AgreementAcceptance {
  id         String   @id @default(cuid())
  createdAt  DateTime @default(now())
  acceptedAt DateTime @default(now())

  agreement   Agreement @relation(fields: [agreementId], references: [id], onDelete: Cascade)
  agreementId String
  profile     Profile   @relation(fields: [profileId], references: [id], onDelete: Cascade)
  profileId   String

  @@unique([agreementId, profileId])
  @@index([profileId])
}

model Profile {
  // …
  hasSigned   Boolean               @default(false)
  acceptances AgreementAcceptance[]
}
```

Migration `20261019120000_agreements`: enum, two tables, one column with default — additive, no
backfill. Types bump to **0.1.115** (`AgreementDto`, `AgreementAcceptanceDto`, `ProfileDto.hasSigned`).

## API

| Route | Who | What |
|---|---|---|
| `GET /agreements?adminId&type&status` | admin | List, newest first, with `_count.acceptances`. |
| `POST /agreement` | admin | `{ adminId, type, displayName, body }` → ACTIVE agreement, `name` slugified, `createdById` stamped. |
| `PUT /agreement/:id` | admin | `{ adminId, displayName?, body?, status? }`. 409 if text changes after any acceptance. |
| `DELETE /agreement/:id` | admin | Soft: status `DELETED`. |
| `GET /agreement/current?type=AFFILIATE` | signed-in | The newest ACTIVE agreement of that type (404 if none). `displayName`, `body`, `id`, `createdAt`. |
| `POST /agreement/:id/accept` | signed-in, own account | `{ accountId }`. Agreement must be ACTIVE; AFFILIATE type requires `profile.type === 'AFFILIATE'`. Idempotent (repeat → 200 with the existing row). Transaction: create acceptance + `hasSigned = true`. |

Layout: `validation/agreement.ts` (pure: type/status parsing, lock check given a count, party
assertion), `services/agreement.ts` (`acceptAgreement` transaction, `currentAgreement`),
`routers/agreement.ts`, registered after `voucherRouter` in `all_routes.ts`.

Gate changes: `assertAffiliateLinkable` takes `hasSigned`; the mint route passes
`bypassSigned = reqUser.role === 'admin'`; the page route checks `profile.hasSigned`.

## Admin

`/agreements` page (nav **Agreements**, after Vouchers): table **Type · Name · Status ·
Acceptances · Created · Updated**, Add New, pencil opens the dialog, trash soft-deletes
(ConfirmDialog). Dialog: Type (`Select`, AFFILIATE only today), Title (`Input` → `displayName`),
Terms (`Textarea`, tall), Status (`Select`) on edit. When `_count.acceptances > 0` the Title and
Terms fields are disabled with the note *"N affiliates have accepted this agreement. Create a new
agreement to change the terms."* `services/api/Agreement.ts`; `AgreementRowDto` typed locally for
the `_count`.

## Mobile

- `SessionContext` exposes `hasSignedAgreement` (= `account.profile.hasSigned === true`).
- **Settings:** the `isAffiliate` block branches. Signed → the existing Affiliate link shelf.
  Unsigned → `ShelfHeading "Affiliate agreement"` with action **Review**, body text *"Accept the
  affiliate agreement to get your affiliate link."* The `affiliateLink` query is `enabled` only
  when signed, so the 403 is never hit.
- **`AffiliateAgreement` screen** in `ProfileNavigator` (vertical slide like Settings): fetches
  `GET /agreement/current?type=AFFILIATE`, renders `displayName` as `legalTitle`, the created
  date as `legalUpdatedText`, `body` as `legalBodyText` paragraphs (split on blank lines), and a
  checkbox row ("I have read and agree to the <title>.") that arms the **I agree** button at the
  end. On press: `POST /agreement/:id/accept`, `refreshCurrentUser()`,
  `alert.success('Agreement accepted')`, `goBack()` → Settings re-renders with the link.
- `services/api/Agreement.ts`: `current(type)`, `accept(id, accountId)`.

## Build order

1. ✅ API (2026-10-17): schema (`AgreementType`, `Agreement`, `AgreementAcceptance`,
   `Profile.hasSigned`, Admin created/lastModified relations) + migration
   `20261019120000_agreements`; `validation/agreement.ts`, `services/agreement.ts`,
   `routers/agreement.ts` (registered after vouchers); `assertAffiliateLinkable` takes
   `hasSigned` with an admin bypass; `GET /user/:username/page` nulls `affiliateLink` until
   signed. Types regenerated; published by the dev pipeline as **0.1.116**. `tsc` + lint clean.
2. ✅ Admin (2026-10-17): `services/api/Agreement.ts`, `app/Agreements/` (page + dialog +
   fetch hook), `/agreements` route, nav item + Create menu entry, local `AgreementDto` /
   `AgreementDraft` in `types/index.ts` (the DTO now comes from the package). `tsc` clean.
3. ✅ Mobile (2026-10-17): `services/api/Agreement.ts`, `hasSignedAgreement` on
   `SessionContext`, Settings branches the affiliate shelf, new
   `AffiliateAgreement` screen in `ProfileNavigator` + param list. `tsc` + lint clean.
4. QA: create an agreement in admin → affiliate sees Review shelf → I agree → link appears;
   admin edit of the accepted agreement is refused; new agreement supersedes.

## Decisions on the open questions (Skylar, 2026-10-17)

1. **Re-acceptance when a new agreement is created — later.** v1 keeps `hasSigned` true once
   any agreement is accepted; recomputing it against the current agreement is a scale-up for
   when terms actually change.
2. **Explicit checkbox — yes.** The mobile screen has a tick ("I have read and agree to the
   <title>.") that arms the **I agree** button; the button is disabled until ticked. Reuses the
   seller-onboarding checkbox row styles.
3. **Who accepted — keep the association, don't show it yet.** `AgreementAcceptance` rows record
   agreement × profile × `acceptedAt`; admin shows only the count for now. A drill-down
   (`GET /agreement/:id/acceptances`) is one endpoint away when wanted.

## Open question

- **Rich text for `body`?** The plumbing exists (admin `RichTextEditor` / Quill, mobile
  `HTMLView` as on entity descriptions); it would need a server-side HTML sanitizer and a
  swap of the admin textarea + mobile paragraph renderer. Undecided; body is plain text today.

## v1 agreement copy

The text affiliates accept lives in the vault at [[../../business/affiliate-agreement-v1|Affiliate Agreement v1]] (title + plain-text body, paste-ready for nanza-admin). Edit it there; an accepted agreement's text is locked in the app, so a change means a new agreement row.
