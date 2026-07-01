# _shared — Cross-Project Brain

Architecture and catalog level only. This is where we think about the system as a whole — **not** a mirror of what runs in each repo. Summaries and decisions live here; implementation lives in the repos.

## Architecture
Decisions and designs that span more than one repo (auth/Cognito, data model, infra, shareability, messaging).
- `architecture/` — one note per cross-cutting topic

## Decisions
ADR-style records: what we chose, why, and what we rejected.
- `decisions/`

## Capabilities catalog
A quick "what do we have to offer" view of skills and orchestration patterns. Intentionally **not one-to-one** with each repo's `.claude/` — add a summary here first, then decide which repos it graduates into.
- [[capabilities/skills|Skills catalog]]
- [[capabilities/orchestrations|Orchestration patterns]]

## Glossary
- [[glossary|Shared domain vocabulary]]
