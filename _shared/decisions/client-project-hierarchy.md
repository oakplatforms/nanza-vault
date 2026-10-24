---
tags: [shared, decision, platform, multi-tenant, client]
---

# Decision: Oak platform → Client → Project hierarchy (2026-07-05)

The Oak backend generalizes from "nanza's API" to **oak-api**, multi-client. **Oak itself is the
platform**; the tenants running on it are its **Clients**:

- **`Client`** = a tenant of the Oak platform (nanza first; client 2, 3 later). All backend
  services, the database, and the Lambdas are shared; every client-level resource carries a
  `clientId` that **defaults to nanza**, so nothing breaks.
- **`Project`** = the *existing* builder model inside a client (the Storefront → Project
  refactor). Unchanged.
- Naming history: "Project" was taken, "Tenant" too technical, "Platform" wrong because Oak
  itself is the platform → **`Client`** (also matches the partner/client business framing).
- **Applications** (nanza-mobile, nanza-web-app, nanza-admin) are client-specific, single-tenant
  apps — they reference their client and never need multi-client awareness.
- **oak-cortex** (the intelligence engine, separate DB) receives the client key from main inside
  the down-synced spec object — it holds no client registry of its own.

Full design + phasing: [[../../nanza-api/solution-designs/oak-api-platform-rescope|nanza-api →
oak-api Platform Re-scope]]. Cortex side:
[[../../oak-cortex/solution-designs/insight-engine|oak-cortex → Insight Engine]].
