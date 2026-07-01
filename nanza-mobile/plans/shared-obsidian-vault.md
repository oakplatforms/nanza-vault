# Plan — Shared Obsidian Documentation Vault (`nanza-vault`)

## Goal

Replace the scattered, per-repo `documentation/` folders with **one shared Obsidian vault** (`nanza-vault`) that acts as a cross-project "brain." Claude keeps writing solution designs / architecture decisions / plans dynamically — but they land in the vault, organized per project. Partners **clone and go** with zero per-machine configuration.

---

## What exists today (research findings)

| Repo | CLAUDE.md | `documentation/` | Files | Subfolders |
|------|-----------|------------------|-------|------------|
| nanza-mobile | ✓ | ✓ | 27 | `plans/`, `deeplink/` |
| nanza-api | ✓ | ✓ | 9 | `plans/` |
| nanza-web-app | ✗ | ✓ | 4 | `plans/` |
| nanza-admin | ✓ | ✗ | 0 | — |
| nanza-email | ✗ | ✗ | 0 | — |

- **The vault already exists locally**: `/Users/skylarbanks/Projects/nanza-vault` — a real Obsidian vault (`.obsidian/` present, `sync` core plugin on), but **not a git repo** (no commits, no remote) and holds only `Welcome.md`.
- **All 5 repos share one parent folder**: `/Users/skylarbanks/Projects/`.
- **All 5 repos are under the `oakplatforms` GitHub org.**
- **`/plan` is the integration hook**: `.claude/commands/plan.md` in nanza-mobile & nanza-email already instructs Claude to read `documentation/REFERENCE.md`. That file only exists in nanza-mobile today (broken read elsewhere). This is exactly where the vault plugs in.
- The universal doc convention across repos is `documentation/plans/*.md` (kebab-case).

---

## Core design decision — relative symlinks (Confidence: 0.85)

Each repo's `documentation/` becomes a **symlink** into the vault, using a **relative path**:

```
nanza-api/documentation  ->  ../nanza-vault/nanza-api
```

Because the link is relative (`../nanza-vault/...`, not `/Users/skylarbanks/...`), it resolves correctly on **any** machine — the only requirement is that everyone checks out the vault and the project repos into the **same parent directory**. Git stores symlinks natively, so the link is committed once and every `git pull` reproduces it.

**The single documented rule for partners:** clone all repos side-by-side, e.g.

```
~/Projects/
  nanza-vault/
  nanza-api/
  nanza-mobile/
  nanza-web-app/
  nanza-admin/
  nanza-email/
```

Then Obsidian opens `nanza-vault`, and every repo's `documentation/` transparently points at its section. Edit once, visible everywhere. No local config, no absolute paths.

**Rejected alternative — git submodule** (Confidence: 0.55): embedding the vault as a submodule in each repo works but forces partners to run `git submodule update --init`, creates detached-HEAD friction on every doc edit, and fights Obsidian (which wants one flat vault root, not five nested `.git`s). Violates the "clone and go, zero config" constraint.

**Rejected alternative — copy/sync script** (Confidence: 0.4): a script that copies docs both ways is a source-of-truth nightmare (merge conflicts, drift, "which copy is real?"). Symlinks give one physical file.

---

## Vault structure

```
nanza-vault/
├── .obsidian/                 # already present — Obsidian config, committed
├── README.md                  # vault landing page (replaces Welcome.md) — the "brain" index
├── nanza-api/                 # <- nanza-api/documentation symlinks here
│   ├── INDEX.md               # intro/map file for this project (see below)
│   └── plans/
├── nanza-mobile/
│   ├── INDEX.md
│   ├── REFERENCE.md           # existing frontend reference guide moves here
│   ├── plans/
│   └── deeplink/
├── nanza-web-app/
│   ├── INDEX.md
│   └── plans/
├── nanza-admin/
│   └── INDEX.md
├── nanza-email/
│   └── INDEX.md
└── _shared/                   # cross-project brain — ARCHITECTURE & CATALOG ONLY, not implementation
    ├── INDEX.md
    ├── architecture/          # decisions that span repos (auth/Cognito, data model, infra)
    ├── decisions/             # ADR-style cross-cutting decisions
    ├── capabilities/          # catalog of skills & orchestration patterns (summaries, not runtime)
    │   ├── skills.md          # "what skills we have to offer" + which repos use them
    │   └── orchestrations.md  # summaries of orchestration patterns (e.g. migration pipeline)
    └── glossary.md            # shared domain vocab (Listing, Bulk, Reference codes, etc.)
```

### `_shared` is architecture & catalog, NOT implementation (Confidence: 0.9)

Per the design intent: the shared folder holds **architectural thinking and summaries**, deliberately *not* a mirror of what runs. Skills and orchestrations are per-project at runtime (they live in each repo's `.claude/`), but the vault keeps a **catalog** — a quick "what do we have to offer" view:

- `capabilities/skills.md` — one short entry per skill (e.g. `principal-engineer`, `deploy`, `prisma-migration`), what it does, and which repos it's deployed in. Today `principal-engineer` is copy-pasted identically across api/admin/web-app and `nanza-api` has 7 migration agents documented nowhere central — the catalog fixes that.
- `capabilities/orchestrations.md` — summaries of orchestration patterns (e.g. the migration orchestrator that fans out to 7 sub-agents). The *pattern* is documented as a reusable playbook; the runnable agents stay per repo.

This is intentionally **not one-to-one** with the actual `.claude/` contents. It's a design surface: add a skill summary to the vault first, then decide over time which repos it should graduate into.

### The `INDEX.md` hierarchy (Confidence: 0.8)

Each project directory gets an `INDEX.md` — a short map, not a wall of text. It links out (Obsidian `[[wikilinks]]`) to the plans and reference docs in that section. Start flat (just a list), and as a section grows the INDEX becomes the table of contents. This is what lets Claude "always pick the right context": it reads the small INDEX first, then follows only the relevant links instead of loading every doc.

`_shared/INDEX.md` is the top of the brain — it links to each project INDEX plus the cross-project architecture/decisions/glossary.

---

## Claude integration — how docs keep landing in the vault (Confidence: 0.85)

Because `documentation/` is a symlink, **nothing about how Claude writes changes** — it still writes to `documentation/plans/foo.md`, which now physically lives in the vault. That's the elegance: zero behavior change, automatic redirection.

Two additive touches make it deliberate rather than accidental:

1. **Update `.claude/commands/plan.md` in every repo** so step 2 reads the project INDEX (`documentation/INDEX.md`) and, when cross-cutting, the shared brain (`documentation/../nanza-vault/_shared/INDEX.md`). Also fixes the currently-broken `REFERENCE.md` read in nanza-api/nanza-email.

2. **Add a short "Documentation" block to each repo's CLAUDE.md** (create CLAUDE.md for nanza-web-app & nanza-email which lack one) stating: "Solution designs, architecture decisions, and plans go in `documentation/` (a symlink into the shared `nanza-vault`). Plans → `documentation/plans/<kebab-name>.md`. Cross-project decisions → the vault's `_shared/`. Read `documentation/INDEX.md` before starting." This is additive and touches no P0 style rules.

---

## Implementation steps

1. **Init the vault as a git repo.** `git init` in `nanza-vault`, add `.gitignore` (ignore `.obsidian/workspace.json`, `.obsidian/cache`, but keep the vault config). — Confidence: 0.9
2. **Scaffold vault directories** (`nanza-api/`, `nanza-mobile/`, `nanza-web-app/`, `nanza-admin/`, `nanza-email/`, `_shared/`) and write each `INDEX.md` + `_shared/glossary.md`. Replace `Welcome.md` with `README.md`. — Confidence: 0.9
3. **Move existing docs into the vault.** `git mv`/copy each repo's current `documentation/*` into the matching vault folder (mobile's `REFERENCE.md`, `plans/`, `deeplink/`; api's `plans/`; web-app's `plans/`, `fe_spec.md`). Preserve structure. — Confidence: 0.85
4. **Replace each repo's `documentation/` with a relative symlink** → `../nanza-vault/<repo>`. For nanza-admin & nanza-email (no folder today), just create the symlink. Commit the symlink in each repo. — Confidence: 0.8 *(needs verification that git records the symlink cleanly on this machine)*
5. **Update `.claude/commands/plan.md`** in all 5 repos to read the INDEX and fix broken REFERENCE reads. — Confidence: 0.9
6. **Add/patch CLAUDE.md** "Documentation" block in all 5 repos (create for web-app & email). — Confidence: 0.9
7. **Create the GitHub repo** `oakplatforms/nanza-vault` (private), push. — Confidence: 0.9
8. **Write a one-paragraph onboarding note** (`nanza-vault/README.md`) telling partners the one rule: clone all repos into the same parent folder. — Confidence: 0.95

---

## Open questions for you

1. **Repo scope** — I listed all 5 projects (api, mobile, web-app, admin, email). Confirm that's the full set, or if any should be left out.
2. **Move vs. copy** — Do you want the existing docs **moved** out of each repo (clean, single source in the vault) or **copied** (leave originals in place as a fallback during transition)? I recommend move.
3. **Should I do step 1–2 (init vault + scaffold) now** so you can open it in Obsidian and see the shape, before I touch any of the app repos? Lowest-risk starting point.

---

## Risk / escalation notes

- **Symlink + git**: the one thing I want to verify live is that committing a `documentation` symlink behaves cleanly (git stores it as a symlink, not a copy) on your setup. Low risk, but I'll confirm on the first repo before doing all five.
- **Touches `.claude/commands/plan.md` and CLAUDE.md** across repos — additive, no P0 style-rule impact, no navigation/API/context changes.
- **No app code touched.** This is purely docs + config plumbing.

**Overall confidence: 0.85** — the approach is sound and low-risk; the only real unknown is the git-symlink behavior, which I'll verify on repo #1 before rolling out.
