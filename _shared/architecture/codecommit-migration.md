---
tags: [shared, architecture, infra, git, codecommit, migration, pipelines]
status: in progress 2026-09-30. Repos created and mirrored (verified), pipeline code done. Remaining is listed under Progress.
---

# CodeCommit migration (git hosting moves from GitHub to AWS)

The plan for moving the six deployed repos from GitHub to AWS CodeCommit, re-pointing the dev
pipelines at them, and the day-to-day workflow afterwards.

## Progress (2026-09-30)

Done:

- Step 1: six `AWS::CodeCommit::Repository` resources added to the shared stack
  (`shared/resources/repositories.js`, `DeletionPolicy: Retain`) and deployed. Repos exist.
- Step 2: credential helper configured on Skylar's Mac, tested against CodeCommit.
- Step 3 and 4: all six repos mirror-cloned from GitHub and pushed. `git ls-remote` on both sides
  is identical for every branch (46 refs in all). Default branch set to `dev`. Scratch mirrors are
  in `~/Projects/_codecommit-migration/`.
- `nanza-mobile` added as a repo-only entry (`pipeline: false` in `repos.js`): repository created,
  39 branches mirrored and verified identical. No pipeline, as before.
- `nanza-vault` added the same way (repo only): created, its single `dev` branch mirrored and
  verified identical.
- `nanza-pipelines` repo created (repo only, never on GitHub). It has no commits yet; Skylar makes
  the first commit and pushes.
- Dev stack deployed in `codecommit` mode by Skylar; `nanza-api-dev` released and **succeeded**.
- GitHub is out of the pipelines code entirely: the `github` source mode and the `nanza-github`
  connection are removed (shared stack redeployed, connection deleted). Modes are now `codecommit`
  and `s3`.
- `AUTO_TRIGGER` defaults to `true`: one EventBridge rule per pipeline starts it when the stage
  branch moves. Takes effect on the next `npm run deploy:dev` / `deploy:prod`.
- Step 6: `codecommit` is now the default `SOURCE_MODE` in `nanza-pipelines`; role permissions,
  optional EventBridge triggers, `release.sh`, and the README are updated. Dev template renders.

Remaining, in order (run by Skylar; these were outside what Claude could execute):

1. `cd ~/Projects/nanza-pipelines && npm run deploy:dev` again — adds the on-push triggers.
2. Step 5 below in each of the eight repos, `nanza-mobile` and `nanza-vault` included (re-point
   `origin` to CodeCommit; the vault has only `dev`, so skip the `prod` line there).
3. `nanza-pipelines`: `git remote add origin https://git-codecommit.us-east-1.amazonaws.com/v1/repos/nanza-pipelines`,
   first commit, `git push -u origin dev`.
4. Commit the pending `buildspec.yml` (and lockfile) changes in `nanza-auth`, `nanza-email`,
   `nanza-admin`, `nanza-web-app`, `oak-web-app` and push to `dev`. With the triggers deployed the
   push starts each dev pipeline.
5. Step 8 (prod) once dev is green, then the GitHub OIDC cleanup below.

SSM parameters are already in place for all six repos on dev, and `@oakplatforms/types` 0.1.105
is in CodeArtifact.

Related: [[deploy-pipelines|Deploy pipelines]],
[[../decisions/aws-codepipeline-over-github-actions|AWS CodePipeline over GitHub Actions]].

## Why

The GitHub restrictions on `oakplatforms` also block the AWS Connector for GitHub app, so the
pipelines cannot use GitHub as their source. The `nanza-github` connection is stuck in `PENDING`
and all six dev pipelines fail at the Source stage. Hosting the repos in CodeCommit removes GitHub
from the deploy path entirely: CodePipeline reads CodeCommit natively, with no connection, app
install, or webhook.

The earlier decision record rejected CodeCommit because AWS closed it to new customers in 2024.
That is out of date. AWS returned CodeCommit to general availability on 2025-11-24 and new
accounts can create repositories
([announcement](https://aws.amazon.com/blogs/devops/aws-codecommit-returns-to-general-availability/)).
This replaces GitLab as stage 2.

## Where things stand (checked 2026-09-29)

| Item | State |
|------|-------|
| AWS account | `992382609315`, `us-east-1`, IAM user `skibanks` (group `admins`, AdministratorAccess) |
| CodeCommit | API reachable, zero repositories |
| Stacks | `nanza-pipelines-shared-shared` and `nanza-pipelines-dev` deployed; prod not deployed |
| Dev stack source mode | `github` |
| `nanza-github` connection | `PENDING` (cannot be completed) |
| Dev pipelines | all six `Failed` |
| GitHub | still readable over git, so a full copy of every branch is possible |

Local clones are **not** a complete copy and must not be the migration source:

- `nanza-api`: local `prod` is 16 commits behind the last fetch, and GitHub `prod` has moved again
  since. Local `stage` is 710 behind.
- `nanza-auth` and `nanza-email` have no local `prod` branch at all.
- `nanza-admin`: local `main` and `stage` are behind.

No local branch is ahead of GitHub, so no committed work exists only on the laptop, with two
exceptions that were never pushed: `nanza-api` `dev-before-auth` and `nanza-web-app`
`landing-replacement`. Decide tomorrow whether to push them to CodeCommit.

Uncommitted work sits in five repos (mostly the new `buildspec.yml` and lockfile changes):
`nanza-auth` 5 files, `nanza-email` 7, `nanza-admin` 8, `nanza-web-app` 6, `oak-web-app` 1.
The migration does not touch working trees, but these need committing before the first release.

## Scope

In: `nanza-api`, `nanza-auth`, `nanza-email`, `nanza-admin`, `nanza-web-app`, `oak-web-app`, and
`nanza-mobile` (repo only, no pipeline; it is a production repo), plus `nanza-vault` (repo only,
this documentation). `dev` and `prod` are the branches that matter. The mirror copies every branch and tag anyway, at
no extra cost, so nothing has to be chosen up front.

Out for now: `nanza-pipelines` (which has no remote yet), `oak-api`,
`oak-cortex-api`, and the `oneoff-*` repos. Same steps apply when they move.

## Plan

### 1. Create the six repositories

Add one `AWS::CodeCommit::Repository` per entry in `repos.js` to the shared stack, each with
`DeletionPolicy: Retain` so removing a stack can never delete code. Then:

```bash
cd ~/Projects/nanza-pipelines
npm run deploy:shared
aws codecommit list-repositories --region us-east-1
```

This is also the real test that the account may create repositories.

### 2. Set up git auth for CodeCommit on the Mac

CodeCommit authenticates with AWS credentials, not a GitHub login. Use the AWS CLI credential
helper, scoped to the CodeCommit host:

```bash
H=credential.https://git-codecommit.us-east-1.amazonaws.com
git config --global $H.helper ''
git config --global --add $H.helper '!aws codecommit credential-helper $@'
git config --global $H.UseHttpPath true
```

The empty first value matters. Xcode's git ships with `osxkeychain` as a system helper, which
would cache the CodeCommit credentials. They expire after 15 minutes, so pushes would start
failing with 403. The empty value clears the helper list for this host only.

Fallback if that misbehaves: `pip install git-remote-codecommit` and use URLs of the form
`codecommit::us-east-1://nanza-api`.

### 3. Mirror from GitHub

Work in a scratch folder, from fresh mirror clones, so local state is irrelevant:

```bash
mkdir -p ~/Projects/_codecommit-migration && cd ~/Projects/_codecommit-migration
CC=https://git-codecommit.us-east-1.amazonaws.com/v1/repos

for r in nanza-api nanza-auth nanza-email nanza-admin nanza-web-app oak-web-app; do
  git clone --mirror https://github.com/oakplatforms/$r.git $r.git
  git -C $r.git push $CC/$r 'refs/heads/*:refs/heads/*' 'refs/tags/*:refs/tags/*'
  aws codecommit update-default-branch --repository-name $r --default-branch-name dev
done
```

Explicit refspecs instead of `push --mirror`, which would also push GitHub's internal
`refs/pull/*` refs. Largest file in any repo is a 12 MB video in `nanza-web-app`, far under
CodeCommit's limits.

### 4. Verify the copy

Every branch must have the same commit on both sides. No output means identical:

```bash
for r in nanza-api nanza-auth nanza-email nanza-admin nanza-web-app oak-web-app; do
  echo "== $r"
  diff <(git ls-remote --heads --tags https://github.com/oakplatforms/$r.git | sort -k2) \
       <(git ls-remote --heads --tags $CC/$r | sort -k2)
done
```

Do not move on until all six are clean. Keep the scratch mirrors until prod is green.

### 5. Re-point the local clones

In each of the six repos. Working trees and uncommitted changes are not affected:

```bash
git remote rename origin github
git remote set-url --push github DISABLED      # guards against pushing to GitHub by habit
git remote add origin $CC/<repo>
git fetch origin
git branch -u origin/dev dev
git branch -u origin/prod prod                 # where a local prod exists
```

GitHub stays as a read-only backup remote named `github`.

### 6. Add a `codecommit` source mode to the pipelines

Changes in `nanza-pipelines`:

- `resources/pipelines.js`: accept `SOURCE_MODE=codecommit` and make it the default. Source action
  is provider `CodeCommit` with `RepositoryName`, `BranchName: <stage>`,
  `PollForSourceChanges: false`, `OutputArtifactFormat: CODE_ZIP`.
- `PipelineRole`: add `codecommit:GetBranch`, `GetCommit`, `GetRepository`, `UploadArchive`,
  `GetUploadArchiveStatus`, `CancelUploadArchive` on the six repositories.
- `scripts/release.sh`: `codecommit` behaves like `github` (start the pipeline, no zip upload).
- `AUTO_TRIGGER=true` in this mode needs an EventBridge rule per pipeline. Keep manual release as
  the default and add this later if wanted.
- Leave the GitHub connection in the shared stack for now. Remove it once prod is green.

### 7. Get dev green

```bash
npm run deploy:dev
scripts/release.sh nanza-api dev       # api first, it publishes @oakplatforms/types
scripts/release.sh nanza-auth dev
scripts/release.sh nanza-email dev
scripts/release.sh nanza-admin dev
scripts/release.sh nanza-web-app dev
scripts/release.sh oak-web-app dev
```

Before releasing, commit the pending `buildspec.yml` and lockfile changes in each repo and push to
`dev`. The remaining items from the deploy-pipelines checklist still apply first: seed the types
package, switch consumer lockfiles, and fill the SSM parameters.

### 8. Prod

Only after every dev pipeline has deployed successfully: `npm run deploy:prod`, merge to `prod`
in each repo, release in the same order.

### 9. Clean up

The GitHub connection is already gone from the shared stack. Left: delete the scratch mirrors in
`~/Projects/_codecommit-migration/`, update [[deploy-pipelines|Deploy pipelines]] and the decision
record to describe CodeCommit as the source, and remove the GitHub Actions OIDC integration from IAM.

#### Removing the GitHub Actions OIDC integration (inventory 2026-09-30)

None of it is CloudFormation-managed; it was created by hand. The pieces:

- OIDC identity provider `arn:aws:iam::992382609315:oidc-provider/token.actions.githubusercontent.com`
- Roles that trust it, all with `AdministratorAccess` attached:
  `github-actions-nanza-api-dev`, `github-actions-nanza-api-prod` (inline `github-auth-policy-*`),
  `github-actions-oak-api-dev`, `github-actions-oak-api-prod`, `github-actions-oak-cortex-dev`,
  `github-actions-oneoff-api-dev`.

Only the `nanza-api` pair belongs to a migrated repo. The `oak-api`, `oak-cortex`, and `oneoff-api`
roles are the deploy path for repos that have not moved yet, so delete those when those repos
move (or now, if their GitHub Actions deploys are considered dead). Deleting the provider kills
every role's trust at once, so do it last.

```bash
# per role: detach managed policies, delete inline policies, delete the role
for r in github-actions-nanza-api-dev github-actions-nanza-api-prod; do
  for p in $(aws iam list-attached-role-policies --role-name $r --query 'AttachedPolicies[].PolicyArn' --output text); do
    aws iam detach-role-policy --role-name $r --policy-arn $p; done
  for p in $(aws iam list-role-policies --role-name $r --query 'PolicyNames' --output text); do
    aws iam delete-role-policy --role-name $r --policy-name $p; done
  aws iam delete-role --role-name $r
done

# last, once no role trusts it
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::992382609315:oidc-provider/token.actions.githubusercontent.com
```

Also in the repos: delete `.github/workflows/`, and drop the `npm.pkg.github.com` lines and tokens
from every `.npmrc` (the types package comes from CodeArtifact now). Revoke those GitHub tokens in
GitHub if the account still allows it.

## Working with CodeCommit day to day

Git itself does not change. `origin` is CodeCommit, and `git pull`, `git push`, branches, and
merges work as before.

| Task | How |
|------|-----|
| Clone | `git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/<repo>` |
| Browse code, commits, branches | Console: Developer Tools, CodeCommit, Repositories |
| Open a pull request | Console: repository, Pull requests, Create pull request. Or the CLI command below |
| Review | Inline and general comments on the pull request in the console |
| Require approvals | Approval rule templates, for example one approval for anything targeting `prod` |
| Merge | Merge button on the pull request: fast-forward, squash, or three-way |
| Deploy | `scripts/release.sh <repo> <dev\|prod>` or Release change in the CodePipeline console |

```bash
aws codecommit create-pull-request \
  --title "Short description" \
  --targets repositoryName=nanza-api,sourceReference=my-branch,destinationReference=dev
```

Suggested flow: branch from `dev`, push the branch, open a pull request into `dev`, merge, release
the dev pipeline. When dev is verified, open a pull request from `dev` into `prod`, merge, release
the prod pipeline.

What is different from GitHub:

- **Access is IAM.** Teammates need an IAM user or role with `AWSCodeCommitPowerUser`, plus the
  credential helper from step 2. There are no GitHub accounts or teams.
- **No checks on pull requests.** Nothing runs automatically when one is opened. Tests on pull
  requests can be added later with EventBridge and CodeBuild.
- **Branch protection is an IAM policy.** Protecting `prod` means a policy that denies direct
  pushes to that branch, not a repository setting.
- **No issues, forks, or wikis.**

## Open questions for tomorrow

1. Push the two local-only branches (`nanza-api` `dev-before-auth`, `nanza-web-app`
   `landing-replacement`) to CodeCommit, or leave them local?
2. Who else needs access, and do they have IAM users yet?
3. Protect `prod` with an approval rule and a deny-direct-push policy now, or after dev is green?
