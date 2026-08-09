# Plan — Make `POST /user` idempotent on `authId`

Scratch plan (gitignored). Prereq for social sign-in: the consumer Pre-Token-Generation
trigger (nanza-auth) fires on **every** token issue, so it will POST `/user` repeatedly for
the same Cognito `sub`. Today that creates duplicate rows. Fix that here first.

## The bug (confirmed)

- `POST /user` — `src/routers/user.ts:177` — runs an unconditional `prisma.user.create`. No
  lookup for an existing user by `authId`.
- `schema.prisma:246` — `authId String` with only `@@index([authId])`, **no `@unique`**. So a
  repeat POST does **not** 409 — it **silently inserts a duplicate `User`** (plus a duplicate
  `Account`, cart, and `collection` list).
- Corroborating signal that non-uniqueness is assumed elsewhere: `GET /user/:authId`
  (`user.ts:460`) uses `findMany(...)[0]`, not `findUnique` — i.e. the code already tolerates
  multiple rows per `authId`. We keep that assumption and just stop *creating* new dupes.

Impact today: the existing `consumerPostConfirmationHandler` / `adminPreTokenGenerationHandler`
each fire once per user, so dupes are unlikely in practice — but the federated Pre-Token trigger
fires on every login, which would turn this latent bug into constant duplicate accounts.

## Fix — return the existing user instead of creating a second

In the `POST /user` handler, before the create loop:

```ts
// Idempotent by authId: a repeat signup/first-login trigger (esp. federated Pre-Token
// generation, which fires on every token issue) must not create a duplicate user.
const existing = await prisma.user.findFirst({
  where: { authId },
  include: { account: { select: { id: true } } },
})
if (existing) {
  res.json(existing)
  return
}
```

- Use `findFirst`, **not** `findUnique` — `authId` isn't unique, and matching the existing
  `findMany` usage keeps behavior consistent even if legacy dupes exist.
- Keep the response shape identical to the create path (`include: { account: { select: { id:
  true } } }`) so callers (and the auth Lambda) see the same object either way.
- Runs after `validateRole(..., 'admin')` — the lookup is still an admin-only action; place the
  check inside the existing `try` after validation.

### Race hardening (P2, recommended)

Two near-simultaneous first-logins for the same `sub` could both pass the `findFirst` check and
both create. To close that window we'd want `authId` to be `@unique`, then catch the P2002 and
re-fetch. But:

- **`@unique` on `authId` = a schema migration**, and this repo's rule is **migrations are run
  manually by the user** (see CLAUDE.md — never run `prisma migrate`). Also `authId` is used in
  a few places assuming multiplicity, so flipping to unique needs a dedup pass on existing data
  first.
- **Decision:** ship the `findFirst` guard now (covers the real-world trigger cadence, which is
  serial per user). Track the `@unique` + P2002-catch hardening as a **follow-up** requiring a
  manual migration + a duplicate-row audit. Flag to the user; don't do it silently.

## What this does NOT change

- No new endpoint, no route signature change (still `POST /user`, admin-only).
- Email/password signup path unaffected — first call still creates exactly as today.
- Validation-vs-services boundary: this is a read-then-write inside an existing handler; small
  enough to stay in the handler, consistent with how create already lives there. (If we later
  extract user creation into `services/`, the idempotency check goes with it.)

## Tests (`src/routers/user.test.ts`)

- Existing: `POST /user` with a new `authId` creates a user. (unchanged)
- **Add:** `POST /user` twice with the same `authId` → second call returns the **same** user
  (same id), and the `User` count for that `authId` stays at 1.
- **Add:** the returned object from the idempotent path includes `account.id` (shape parity).

## Verification
- Run the user router tests (`npm test`) — Docker Postgres on 5433.
- Manually: POST the same payload twice, assert one row via a count query.

## Downstream (after this lands)
- Unblocks the nanza-auth consumer Pre-Token-Generation trigger (see
  `nanza-auth/.plans/social-identity-providers.md`).
