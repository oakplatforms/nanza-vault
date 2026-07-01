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
└── nanza-email/
```

Each repo's `documentation/` folder is a **relative symlink** into this vault (e.g. `nanza-api/documentation → ../nanza-vault/nanza-api`). As long as everything sits side-by-side, it just works — no per-machine setup, no absolute paths. Pull the vault, pull the repos, start working.

## Map

- [[nanza-api/INDEX|nanza-api]] — backend (Node/Prisma, Lambda, migrations)
- [[nanza-mobile/INDEX|nanza-mobile]] — React Native app
- [[nanza-web-app/INDEX|nanza-web-app]] — web app
- [[nanza-admin/INDEX|nanza-admin]] — admin console
- [[nanza-email/INDEX|nanza-email]] — email service
- [[_shared/INDEX|_shared]] — cross-project architecture, decisions & capability catalog

## How docs get here

Claude writes plans exactly where it always did (`documentation/plans/<name>.md`) — because that folder is a symlink, the file lands in this vault automatically. Cross-project architecture and decisions go in `_shared/`.
