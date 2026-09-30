---
tags: [shared, decision, infra, deploy, github, gitlab]
---

# Decision: AWS CodePipeline over GitHub Actions (2026-09-29)

GitHub placed restrictions on the `oakplatforms` account. Actions, pipelines, and pull requests stopped working. Pushing code still works.
Every repo's deploy was a GitHub Actions workflow, so all deploys were blocked.

**Chosen:** move all deploys into AWS. One CodePipeline + CodeBuild per repo per stage, defined
as a Serverless Framework stack in a new `nanza-pipelines` repo, pulling the branch from GitHub
through an AWS-side connection (with an S3 zip upload as the fallback source). Triggering is
manual after a push. `@oakplatforms/types` moves from GitHub Packages to CodeArtifact.

**Why this shape:**
- Nothing flows from GitHub to AWS anymore. AWS reaches into GitHub (or reads a zip we upload),
  so the deploy path has no dependency on GitHub features the account may lose next.
- Serverless Framework already deploys three of the repos; defining the pipelines the same way
  keeps one tool and one mental model.
- Buildspecs are line-for-line ports of the old workflows, so the first goal (be unblocked) does
  not also become a refactor of how each repo deploys.
- The types registry was the hidden single point of failure: five repos install from GitHub
  Packages. Moving it now removes the last GitHub dependency from the build.

**Rejected:**
- Deploying from laptops with `npm run deploy:*`. Works for the Serverless repos but not the
  web apps' S3 sync steps, and puts prod deploys on one person's machine and credentials.
- Waiting on GitHub support. Unknown timeline; deploys can't wait.
- CodeCommit as the git host. AWS closed it to new customers in 2024; not a foundation.
- Rewriting the deploys around a different framework (CDK, Terraform). Bigger change than the
  problem; revisit only if the Serverless-defined stack turns out to be limiting.

**Follow-on (stage 2):** move the repos to GitLab, then re-point the source action (GitLab is a
supported connection provider) and decide whether the npm registry stays in CodeArtifact or moves
to GitLab's. The pipelines themselves do not change.

**Update (2026-09-29):** the GitHub restrictions also block the AWS Connector for GitHub app, so
the pipelines cannot read from GitHub. Stage 2 is now CodeCommit, not GitLab. The rejection above
is out of date: AWS returned CodeCommit to general availability on 2025-11-24. Plan:
[[../architecture/codecommit-migration|CodeCommit migration]].

Design: [[../architecture/deploy-pipelines|Deploy pipelines]].
