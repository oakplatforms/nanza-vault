# Nanza Vault 🧠

The shared documentation "brain" for all Nanza projects. Solution designs, architecture decisions, and implementation plans for every repo live here, in one place, so the whole team — and Claude — can see the system in its entirety instead of doc silos scattered across repos.

## One rule for setup (zero config)

Clone the vault **and all project repos into the same parent folder**:

```
~/Projects/
├── nanza-vault/      ← this repo (open THIS in Obsidian)
├── nanza-api/
├── nanza-mobile/
├── nanza-web-app/
├── nanza-admin/
├── nanza-email/
└── nanza-auth/
```

Each repo's `documentation/` folder is a **relative symlink** into this vault (e.g. `nanza-api/documentation → ../nanza-vault/nanza-api`). As long as everything sits side-by-side, it just works — no per-machine setup, no absolute paths. Pull the vault, pull the repos, start working.

## Projects

Each links to a project map (`INDEX`) with a structural overview, an `architecture.md`, and themed plans.

- [[nanza-api/INDEX|nanza-api]] — marketplace backend: Node/Express + Prisma on Lambda, Cognito, Stripe, Shippo, OG share images.
- [[nanza-mobile/INDEX|nanza-mobile]] — React Native app (iOS + Android); the reference styling architecture.
- [[nanza-web-app/INDEX|nanza-web-app]] — public web app (CRA); share-link surface + marketing site.
- [[nanza-admin/INDEX|nanza-admin]] — internal admin console (CRA + Tailwind) for catalog & back office.
- [[nanza-email/INDEX|nanza-email]] — transactional email service: EventBridge-triggered Lambda + SES + react-email.
- [[nanza-auth/INDEX|nanza-auth]] — auth glue: Cognito user-pool trigger Lambdas, token & password flows.
- [[_shared/INDEX|_shared]] — cross-project architecture, decisions & the capabilities catalog.

## Capabilities

What the system has to offer, at a glance — summaries in `_shared/`, the runnable versions live per-repo in each `.claude/`.

- [[_shared/capabilities/skills|Skills catalog]] — `principal-engineer`, `deploy`, `prisma-migration`, `plan`, and which repos use them.
- [[_shared/capabilities/orchestrations|Orchestration patterns]] — reusable multi-agent playbooks (e.g. the card-data migration pipeline).
- [[_shared/glossary|Glossary]] — shared domain vocabulary.

## Solution designs, not plans

The vault holds **living solution designs** — durable overviews of how each feature works and why (Collections, Sharing, Auth, …) — in each project's `solution-designs/` folder. It is **not** a log of day-by-day code changes.

- **Ephemeral implementation plans** stay local in each repo's gitignored `.plans/` folder — never committed, never synced here.
- When work lands, its durable insight **graduates** into the relevant `documentation/solution-designs/<theme>.md` (created if missing). So each project's `solution-designs/` grows into a complete, current picture of its features over time.
- **`solution-designs/` is the main living surface**, but `INDEX.md` and `architecture.md` are living too — when a solution design is added or a feature changes the wiring, update the INDEX link and the architecture doc to match.
- Cross-project architecture, decisions, and the capabilities catalog live in `_shared/`.
