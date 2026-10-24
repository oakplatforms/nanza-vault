---
tags: [shared, architecture, infra, deploy, pipelines, codepipeline, codeartifact]
status: stage 1 built 2026-09-29, pending first deploy
---

# Deploy pipelines (AWS CodePipeline)

How every Nanza repo gets from a `dev` / `prod` branch to AWS, now that GitHub Actions is
unavailable to the org. Replaces the per-repo `.github/workflows/deploy-*.yml` files, which
stay in the repos but no longer run.

Decision record: [[../decisions/aws-codepipeline-over-github-actions|AWS CodePipeline over GitHub Actions]].
Infra code: `nanza-pipelines` repo (sibling of the other repos).

## Shape

```
push to dev|prod  ──►  (manual) Release change  ──►  CodePipeline  <repo>-<stage>
                                                        │
                                                        ├─ Source   GitHub via AWS connection  (or S3 zip fallback)
                                                        └─ Deploy   CodeBuild  <repo>-<stage>  runs repo/buildspec.yml
```

- **One pipeline per repo per stage**, twelve in all: `nanza-api`, `nanza-auth`, `nanza-email`,
  `nanza-admin`, `nanza-web-app`, `oak-web-app` × `dev`, `prod`. `nanza-mobile` has no server
  deploy and is not in the set.
- **Two CloudFormation stacks per stage** plus one shared stack, all defined with the Serverless
  Framework in `nanza-pipelines`:
  - `nanza-pipelines-shared` — the GitHub connection and the CodeArtifact domain + repository.
  - `nanza-pipelines-dev` / `nanza-pipelines-prod` — artifact bucket, a CodeBuild role, a
    CodePipeline role, and the pipeline + build project for each repo. `resources/pipelines.js`
    generates them from `repos.js`, so adding a repo is one entry plus a `buildspec.yml`.
- **The buildspec lives in the repo it deploys.** Each repo's `buildspec.yml` is a straight port
  of its old workflow (same commands, same order), so the deploy steps stay next to the code they
  deploy and the pipeline repo stays generic.
- **Triggering is manual by design.** Pushing to `dev` or `prod` does nothing. Deploys start from
  *Release change* on the pipeline in the console or `scripts/release.sh <repo> <stage>`. The
  GitHub connection's change detection is off (`AUTO_TRIGGER=false`) because inbound GitHub
  webhooks are the thing we can no longer rely on; flip it when the org is healthy again or after
  the GitLab move.

## Source: GitHub connection, S3 zip fallback

The pipelines pull code *from AWS into GitHub*, never the other way. The source action is a
CodeStar connection (`AWS Connector for GitHub` app on the `oakplatforms` org). The connection is
created PENDING by the shared stack and finished once in the console.

If that app install is blocked too, the same stacks deploy in `SOURCE_MODE=s3`. The source becomes
`s3://nanza-pipelines-<stage>-<account>/source/<repo>/<stage>.zip`, and `scripts/release.sh`
does `git archive` of the local branch, uploads it, and starts the pipeline. Same buildspecs,
same CodeBuild projects. This is also the path that keeps working through the GitLab migration.

## Config: SSM Parameter Store instead of GitHub variables

The old workflows injected `vars.X_DEV` / `vars.X_PROD`. Those values now live in Parameter Store
at `/<repo>/<stage>/<NAME>` (for example `/nanza-api/dev/ADMIN_USER_POOL_ID`). Each buildspec
reads the ones it needs with `aws ssm get-parameter` in `pre_build` and fails fast when a required
one is missing. Optional ones (web app's Cognito and Stripe on prod, CloudFront on dev) are simply
absent. `nanza-pipelines/params/params.env.example` lists every parameter as
`<repo>/<stage>/<NAME>=`; fill a gitignored copy and `scripts/put-params.sh` writes them all in one
run (or a `<repo> [<stage>]` subset).

Database credentials keep coming from Secrets Manager (`nanza-credentials-<stage>`) inside the
build, as before. The CodeBuild role has the same broad rights the OIDC role had
(AdministratorAccess by default, `DEPLOY_POLICY_ARN` to narrow) because `serverless deploy`
creates IAM roles, Lambdas, API Gateways, CloudFront, DynamoDB, EventBridge, and Step Functions.

## The types package: CodeArtifact replaces GitHub Packages

`@oakplatforms/types` used to publish to and install from `npm.pkg.github.com`, with a GitHub
token in five repos' `.npmrc`. Any GitHub restriction on packages would have broken every build,
so the registry moved to **AWS CodeArtifact** (domain `oakplatforms`, repository `oakplatforms`).

- Consumers (`nanza-email`, `nanza-admin`, `nanza-web-app`, `nanza-mobile`) carry only the scope
  line in `.npmrc`; auth comes from `aws codeartifact login` (12h token). Locally:
  `nanza-pipelines/scripts/npm-login.sh`. In CodeBuild the buildspec logs in before `npm ci`.
- Publishing happens in the **api dev pipeline only**, as before. Change detection is a content
  hash of the built `dist/` compared with the latest published tarball, not `git diff HEAD~1`,
  because the pipeline source is a zip and several commits can land between releases. On change it
  bumps a patch above max(latest published, package.json) and publishes.
- `scripts/seed-types.sh` publishes the current version once so consumers can switch before the
  first api pipeline run.
- Consumer lockfiles still resolve to the GitHub tarball URLs until each repo re-installs the
  package after the registry switch (`npm install @oakplatforms/types@latest`).

## Stage 2: GitLab

> Superseded 2026-09-29. Stage 2 is CodeCommit: see [[codecommit-migration|CodeCommit migration]].
> The section below is kept for the record.

Repos move to GitLab later. What changes then: the source action (GitLab is also a CodeStar
connection provider, or keep S3 zips) and, optionally, the registry (GitLab's package registry
speaks npm too, so CodeArtifact could be dropped for it). The buildspecs, roles, params, and the
stage-per-branch model carry over unchanged.

## Migration checklist (stage 1)

Built 2026-09-29, nothing deployed yet. Order matters.

1. `nanza-pipelines`: `npm install`, then `npm run deploy:shared`. This is the stage-neutral
   account stack, not prod: it creates the CodeArtifact registry that step 2 publishes into and
   the GitHub connection whose ARN the dev and prod stacks import (they fail to deploy without
   it). Complete the pending connection in the console; if the app install is refused, plan on
   `SOURCE_MODE=s3`.
2. `scripts/seed-types.sh` to publish `@oakplatforms/types` into CodeArtifact.
3. In each consumer repo: `scripts/npm-login.sh`, then `npm install @oakplatforms/types@latest`
   to rewrite the lockfile's resolved URLs; commit.
4. `cp params/params.env.example params/params.env`, fill every line from the GitHub repo
   variables, then `scripts/put-params.sh` once to write all twelve repo/stage sets.
5. **Dev only first.** `npm run deploy:dev` (prefix with `SOURCE_MODE=s3` if needed). Do not
   deploy the prod stack yet.
6. Commit the `buildspec.yml` changes in each repo, push to `dev`, release the dev pipelines one
   at a time (api first, it publishes types). Fix anything that breaks here, in dev.
7. Only when every dev pipeline is green: `npm run deploy:prod`, merge to `prod`, release the
   prod pipelines in the same order.
8. Once prod is green: narrow `DEPLOY_POLICY_ARN`, revoke the old GitHub tokens that were in
   `.npmrc`, delete the OIDC roles, and remove the `.github/workflows` files.
