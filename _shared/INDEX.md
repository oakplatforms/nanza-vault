# _shared — Cross-Project Brain

Architecture and catalog level only. This is where we think about the system as a whole — **not** a mirror of what runs in each repo. Summaries and decisions live here; implementation lives in the repos.

## Architecture
Decisions and designs that span more than one repo (auth/Cognito, data model, infra, shareability, messaging).
- `architecture/` — one note per cross-cutting topic
- [[architecture/mixpanel-playbook|Mixpanel playbook]] — how to set up and read the Nanza Mixpanel projects: the ids-plus-lookup-table model, Lexicon setup, the five boards, refresh, Dev vs Prod, and the measurement gaps
- [[architecture/deploy-pipelines|Deploy pipelines]] — AWS CodePipeline + CodeBuild per repo per stage (defined in `nanza-pipelines`), manual release after a push, CodeArtifact for `@oakplatforms/types`; replaces GitHub Actions.
- [[architecture/codecommit-migration|CodeCommit migration]] — plan (2026-09-29, not yet run) to move the six deployed repos from GitHub to AWS CodeCommit, re-point the pipelines, and the pull request workflow afterwards.

## Decisions
ADR-style records: what we chose, why, and what we rejected.
- `decisions/`
- [[decisions/aws-codepipeline-over-github-actions|AWS CodePipeline over GitHub Actions]] (2026-09-29) — GitHub restricted the org; deploys move into AWS, GitLab is stage 2.

## Capabilities catalog
A quick "what do we have to offer" view of skills and orchestration patterns. Intentionally **not one-to-one** with each repo's `.claude/` — add a summary here first, then decide which repos it graduates into.
- [[capabilities/skills|Skills catalog]]
- [[capabilities/orchestrations|Orchestration patterns]]

## Glossary
- [[glossary|Shared domain vocabulary]]
