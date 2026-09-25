# Plan — Client model + serverless oak rename (nanza-api)  ✅ IMPLEMENTED 2026-07-05

Status: schema + serverless edits done, `prisma validate` passes. Pending on user: run the
migration, seed nanza Client + backfill, deploy serverless to dev then prod, RDS/secret rename.


Scope this session: (A) add the `Client` model + `clientId` to catalog roots, (B) rename lambdas
to oak **in-stack only**. Cortex is later. User runs the migration + the RDS/DB rename.

---

## Part A — `Client` model (schema only; user runs the migration)

**New model** (top of models section, near Brand/Category; enums none needed):

```prisma
model Client {
  id        String   @id @unique @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  name        String  @unique   // "nanza"
  displayName String?
  config      Json?

  createdBy        Admin?  @relation(name: "CreatedBy", fields: [createdById], references: [id], onDelete: SetNull)
  createdById      String?
  lastModifiedBy   Admin?  @relation(name: "LastModifiedBy", fields: [lastModifiedById], references: [id], onDelete: SetNull)
  lastModifiedById String?
  brands           Brand[]
  categories       Category[]
}
```

**`clientId` on client roots — one-to-many, nullable first.** Add to **Brand** and **Category**
(the two catalog roots; Entity/Product/Set inherit client through Brand, so they do NOT get their
own `clientId` — keeps the delta small):

```prisma
// in Brand and Category
client   Client? @relation(fields: [clientId], references: [id], onDelete: SetNull)
clientId String?
@@index([clientId])
```

**Admin back-relations** (required by the audit-relation convention) — add to `Admin`:

```prisma
createdClients      Client[] @relation(name: "CreatedBy")
lastModifiedClients Client[] @relation(name: "LastModifiedBy")
```

**Confidence 0.9** — additive only, no infra risk, matches existing Brand/Category shape.
Nullable-first is deliberate: the user then seeds one `Client{name:"nanza"}`, backfills every
Brand/Category to it, and can tighten to required in a later migration. **`clientId` stays
nullable in this pass** so the migration is non-breaking on existing rows.

Decision to confirm: **Brand + Category only** carry `clientId` now (vs. adding to Project,
Group, List too). Recommend Brand+Category only for the minimal first cut — 0.8; the rejected
alternative (all client roots at once) is a bigger migration with no immediate consumer.

---

## Part B — serverless rename to oak (IN-STACK, no stack swap)

**`service: nanza-api` STAYS** (per your call — avoids a new CloudFormation stack / new API URL /
webhook + edge cutover).

**Rename these `name:` fields** `nanza-api-*` → `oak-api-*` (CloudFormation replaces each function
in place; API Gateway + its URL survive, so apps/webhooks are unaffected):

- `api`: `nanza-api-${stage}` → `oak-api-${stage}`
- `authorizer`, `guestToken`, `websocket`, `fanout`, `og`, `meta`, `stripeWebhook`,
  `shippoWebhook`, `cancelOrder`, `reviewOrder` — same pattern.

**⚠️ DO NOT rename (leave as nanza) — these two are the landmines we discussed:**
- **`ogEdge`** (`nanza-api-og-edge`) — Lambda@Edge, associated to CloudFront. Renaming forces a
  replace; edge replicas take *hours* to delete and the deploy will error / can break the
  distribution. Leave its `name:` as-is.
- **`OgEdgeExecutionRole`** (`RoleName: nanza-api-og-edge-${stage}-role`) — renaming an IAM role
  forces replace and is tied to the edge function. Leave as-is.

**Also leave untouched (not lambda names — runtime couplings):**
- `fanout` EventBridge pattern `source: [nanza]` (line 163) + `fanoutHandler` `Source: 'nanza'`
  publisher. Changing the event source string is a coordinated publisher+consumer change, not a
  rename — out of scope this pass.
- Secrets ARN `nanza-credentials-*` (line 65) — the DB credentials secret; you own renaming that
  alongside the RDS work.
- `s3Bucket`, `apiBaseUrl`, `webBaseUrl` (nanza.app domains, nanza-static buckets) — real
  resources, not labels; leave.

**Confidence 0.85** on the in-stack function renames being safe (API Gateway/URL preserved).
The one caveat: each renamed function is delete+recreate, so there's a few-seconds cold window
per function on deploy — do it during a quiet window, dev first.

---

## What I will NOT touch
- Navigation / AppNavigator (mobile P0) — n/a here anyway.
- `src/contexts/*`, `fetchData`, `src/styles/utils/*`.
- The RDS instance rename + secret host update — **you** run these; nothing in serverless.yml
  references the DB identifier, so no config change is needed for it.
- Running `prisma migrate` — you run migrations.

## Order of operations
1. Schema edits (Part A) → you review → you run migration → seed nanza Client → backfill.
2. serverless.yml edits (Part B) → deploy to **dev** first, verify, then prod.

## 🔀 COURSE CHANGE (2026-07-05, evening) — went full stack swap to oak-api after all

During the dev deploy the plan changed. What happened, in order:
1. An intermediate deploy had `service: oak-api` → created a healthy **`oak-api-dev`** stack
   (`UPDATE_COMPLETE`) with all `oak-api-*` functions.
2. Config got reverted to `service: nanza-api` while function `name:` fields stayed `oak-api-*`
   → the old `nanza-api-dev` stack then tried to create `oak-api-*` functions **already owned by
   the oak-api-dev stack** → cross-stack name collision, "failed to deploy".
3. **Decision (user): keep `oak-api-dev`, do the full stack swap** (reverses the earlier
   "keep stack name nanza-api" call). `service:` set to `oak-api`; **ogEdge function + its
   OgEdgeExecutionRole also renamed to `oak-api-og-edge-*`** (user chose the clean end state over
   letting the old stack linger just for the edge lambda).

**serverless.yml is now fully oak:** `service: oak-api`, all 12 functions `oak-api-*`, edge role
`oak-api-og-edge-*-role`. Intentionally-kept nanza refs (real external resources, NOT labels):
`nanza-credentials-*` secret, `nanza-static-*` S3 buckets, `*.nanza.app` domains.

### Remaining manual cutover steps (dev, then repeat for prod) — ORDER MATTERS
1. `rm -rf .serverless && npm run deploy:dev` → reconcile the oak-api-dev stack (creates the new
   oak-named edge lambda + role fresh — no collision since names are new).
2. **Re-point CloudFront** to the new `oak-api-og-edge` version (share-image previews).
   ⚠️ The CloudFront distribution is NOT in this repo — it's console-managed, on the **web-app**
   distribution (`d17uau8qojkras.cloudfront.net`), origin-request trigger. Renaming ogEdge in
   serverless only *creates* the new oak edge function; it does NOT move CloudFront's Lambda@Edge
   association, which still references the OLD `nanza-api-og-edge` version. Must be swapped by hand
   in the CloudFront console (distribution → behavior → Function/Lambda associations) BEFORE the
   old stack is deleted, or share previews break the moment the old edge function is gone.
   DEV STATUS 2026-07-05: domain re-mapped to oak (requests confirmed hitting oak lambdas); edge
   association swap still pending (user handling).
3. **Re-map the custom domain** `api-dev.nanza.app` → the oak-api-dev HTTP API (API Gateway →
   Custom domain names → API mappings). No serverless-domain-manager plugin, so the raw
   execute-api URL differs per stack; the custom domain mapping is what keeps clients/webhooks
   working. After this, mobile + Stripe/Shippo webhooks hit oak with no client change.
4. **Only after** domain + edge are moved and `api-dev.nanza.app` verified: disassociate the old
   edge lambda from CloudFront, wait hours for edge replicas to drain, then
   `aws cloudformation delete-stack --stack-name nanza-api-dev`.
5. **PROD:** identical dance for `api.nanza.app` — higher stakes (live users, Stripe webhooks,
   the edge lambda). Move the domain mapping so webhooks keep working; drain the old edge lambda
   before deleting `nanza-api-prod`.

## ⚠️ Deploy gotcha — log-group collision on the lambda rename (hit on dev, 2026-07-05)

`UPDATE_FAILED: <Fn>LogGroup … /aws/lambda/oak-api-<fn>-dev already exists in stack oak-api-dev`,
one function at a time; partial deploy rolls back to `UPDATE_ROLLBACK_COMPLETE` (updatable again,
no `continue-update-rollback` needed). NOTE: no orphaned `oak-api-*` groups were found in
CloudWatch — so the "already exists" is WITHIN the stack (duplicate managed LogGroup resource from
the rename), not an AWS-side auto-created group. See the investigation notes for the resolved
cause. PROD will likely hit the same on first rename deploy.
