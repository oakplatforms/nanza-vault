---
tags: [nanza-api, solution-design, platform, multi-tenant, oak, client]
---

# oak-api — Platform Re-scope (nanza-api → Oak's API)

**Status: design only (2026-07-05). Nothing built. Big change — plan carefully, ship in phases,
zero breaking changes.**

## The idea

nanza-api is really **Oak's API** wearing nanza's name. The database, the Lambdas, the serverless
service are all scoped/named "nanza", but nanza is just the *first client* running on them.
**Oak itself is the platform** — so the tenant concept is a **`Client`** of the platform. The
re-scope: re-name the backend to **oak-api** (in-stack renames only — see phase 4; the
CloudFormation stack name stays), introduce a **`Client`** model, and make every client-level
resource carry a `clientId` that **defaults to nanza** — so everything works exactly as it does
today, and client 2 / client 3 later reuse the same services, same DB, same Lambdas.

## Naming: Oak platform → `Client` → `Project`

Naming history (settled 2026-07-05, same session):
- **Not "Project"** — the schema **already has `Project` / `ProjectEdit` / `ProjectType`** (the
  Storefront → Project builder refactor, see [[projects|Projects / AI builder]]). Taken, stays.
- **Not "Tenant"** — too technical (user call).
- **Not "Platform"** — Oak *itself* is the platform; the things running on it are its clients
  (matches the partner/client business framing). Final: **`Client`** / `clientId`. No schema
  collision (`grep "model Client\|clientId"` → nothing; `s3Client` etc. are code utils, not
  models).

```
Oak platform (oak-api + shared DB + Lambdas)
└── Client (nanza, client-2, …)
    ├── catalog: Brand → Set → Entity → Product …   (client-scoped)
    ├── Project (builder projects — the existing model, children of a client)
    └── applications (nanza-mobile, nanza-web-app — client-specific apps)
```

Client apps stay client-branded and single-tenant (nanza-mobile talks nanza) — no app changes;
only the backend generalizes.

## What changes (phased, non-breaking)

1. **`Client` model** — `id`, `name` (unique key, e.g. `nanza`), `displayName`, `config`.
   Seed one row: nanza.
2. **`clientId` on top-level catalog roots** — start minimal: `Brand`, `Category`, `Project`,
   and other client roots; children inherit scope through their existing parent relations
   (Entity → Brand already implies the client). Nullable first → backfill every row to nanza →
   then required. One migration per step; the user reviews/runs all migrations manually.
3. **Default-to-nanza resolution** — the API resolves the client from (in order) an explicit
   header/param, the caller's app credentials, else the nanza default. Existing apps send
   nothing and keep working untouched.
4. **Renames — in-stack only, NO stack swap (user call, 2026-07-05).** Changing
   `service: nanza-api` would create a brand-new CloudFormation stack (new ARNs, new API Gateway
   URL) and force a cutover across baked-in mobile API URLs, Stripe/Shippo webhook registrations,
   and the Lambda@Edge wiring — explicitly rejected. Instead the **stack name stays `nanza-api`
   as accepted cosmetic debt**, and resources are renamed to oak *inside* it, by tier:
   - **Free label changes (safe any time):** Cognito user-pool display names (pool IDs are
     unchanged), env var / secret *names*, repo + package names, all code-level naming, docs.
   - **In-stack replacements (safe, brief deploy risk):** individual Lambda function names via
     explicit `functions.<fn>.name: oak-…` overrides — CloudFormation deletes/recreates each
     function inside the same stack; API Gateway (and its URLs) survives, so apps and webhooks
     are untouched.
   - **RDS rename (user-run, quiet window):** the instance identifier CAN be renamed in the AWS
     console, but the connection **endpoint hostname changes** and the instance briefly reboots —
     rename, update the connection secret/config, redeploy. Renaming the logical Postgres DB
     (`nanza` → `oak`) is a separate `ALTER DATABASE … RENAME` needing drained connections.
   - **Leave alone:** the Lambda@Edge share-preview function (CloudFront association +
     replicated-function deletion makes replacement painful) and the stack name itself.
   - **Everything NEW is named oak from day one.**
5. **oak-cortex alignment** — cortex holds no client registry; the client key + IDs come *down*
   from main inside the spec object / `CatalogItem` (see
   [[../../oak-cortex/solution-designs/insight-engine|oak-cortex → Insight Engine]]).

## Constraints

- **Zero breaking changes at every step.** nanza-mobile / nanza-web-app / nanza-admin keep
  working unmodified throughout; nanza is the default client everywhere a client isn't named.
- **`@oakplatforms/types`** — `Client` + `clientId` fields flow into the types package;
  additive only.
- Repo/package renames (nanza-api → oak-api repo name, docs) are cosmetic and last.

## Open questions (to settle before building)

- Exact set of models that carry `clientId` directly vs inherit via parent.
- Whether Cognito user pools are per-client or shared with client membership.
- S3 bucket/key scoping and OG-share domains per client (nanza.app is client-level).
- Timing of the RDS instance-identifier rename (user runs it in the console, quiet window).

## Related

- [[projects|Projects / AI builder]] — the existing `Project` model that forced the naming call
- [[../../oak-cortex/solution-designs/insight-engine|oak-cortex → Insight Engine]]
- [[../../_shared/decisions/client-project-hierarchy|Shared decision: Client → Project hierarchy]]
- [[../INDEX|nanza-api]] · [[../architecture|Architecture]]
