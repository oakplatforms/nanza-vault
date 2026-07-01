---
tags: [nanza-mobile, solution-design, docs-infra]
---

# Documentation vault — Solution Design

## Overview

All Nanza documentation lives in **one shared Obsidian vault**, `nanza-vault`, instead of
scattered per-repo `documentation/` folders. Each repo's `documentation/` is a **relative
symlink** into the vault, so Claude keeps writing to `documentation/…` as before while the files
physically land in the vault, organized per project. Partners **clone and go** — the only rule is
that all repos are checked out side-by-side under one parent folder.

## How it works

- **Relative symlinks.** Each repo's `documentation/` points at `../nanza-vault/<repo>`. Because
  the path is relative (not an absolute `/Users/...`), it resolves on any machine. Git stores the
  symlink natively, so it's committed once and reproduced on every pull. The single onboarding rule
  is: clone `nanza-vault` and every repo into the same parent directory
  (`~/Projects/nanza-vault/`, `~/Projects/nanza-api/`, …).

- **Vault structure.** One folder per project (`nanza-api/`, `nanza-mobile/`, `nanza-web-app/`,
  `nanza-admin/`, `nanza-email/`), each with an `INDEX.md` map, plus a `_shared/` "brain" holding
  cross-project **architecture**, **decisions** (ADR-style), a **capabilities catalog**
  (skills + orchestration patterns as summaries, not runtime copies), and a **glossary**. The
  `_shared` folder is deliberately architecture-and-catalog only — not a mirror of what actually
  runs in each repo's `.claude/`.

- **INDEX hierarchy.** Every project directory has a short `INDEX.md` that links out via Obsidian
  wikilinks to that project's designs. `_shared/INDEX.md` sits at the top and links to each project
  INDEX. This is what lets Claude load a small map first and follow only the relevant links.

- **Claude integration is zero-behavior-change.** Because `documentation/` is a symlink, Claude
  still writes to `documentation/plans/<name>.md` — it just physically lands in the vault. Two
  additive touches make it deliberate: each repo's `.claude/commands/plan.md` reads the project
  `INDEX.md` (and the shared brain when cross-cutting), and each repo's `CLAUDE.md` carries a short
  "Documentation" block pointing at the vault.

## Solution designs vs. plans

This vault holds **living solution designs** — durable "how each feature works and why" writeups —
not a graveyard of day-by-day implementation plans. Per-change plans are synthesized into one
solution design per theme (this `solution-designs/` folder) and the raw plans are removed. "Which files
changed today" is ephemeral; "how Collections works, its rules, its history of decisions" is what
the vault keeps.

## Key decisions & rationale

- **Relative symlinks over submodules or a sync script.** Submodules force
  `git submodule update --init`, create detached-HEAD friction on every doc edit, and fight
  Obsidian's single-root model. A copy/sync script is a source-of-truth nightmare (drift, "which
  copy is real?"). Symlinks give one physical file with zero per-machine config.
- **`_shared` is architecture & catalog, not implementation.** It's a design surface — add a skill
  or pattern summary here first, then decide which repos it graduates into — intentionally not
  one-to-one with runtime `.claude/` contents.
- **Small INDEX maps, not walls of text.** Each INDEX links out so Claude picks the right context
  by reading the map and following only relevant links.

## Related

- [[../INDEX|nanza-mobile]]
- [[../architecture|Architecture]]
- [[../REFERENCE|Frontend Reference]]
