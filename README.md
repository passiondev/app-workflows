# app-workflows

Shared GitHub Actions workflows that deploy and roll back the apps the app registry creates.
Each app repo calls them with its `PLATFORM` variable, which the registry's Terraform writes.
The workflows hold no app settings of their own, so one `v1` tag serves every app.

## Using the workflows

An app repo calls each workflow from a small workflow of its own.

`.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main, staging]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

jobs:
  deploy:
    uses: <owner>/app-workflows/.github/workflows/deploy.yml@v1
    with:
      platform: ${{ vars.PLATFORM }}
```

`.github/workflows/rollback.yml`:

```yaml
name: Roll back

on: workflow_dispatch

permissions:
  id-token: write

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

jobs:
  rollback:
    uses: <owner>/app-workflows/.github/workflows/rollback.yml@v1
    with:
      platform: ${{ vars.PLATFORM }}
```

An app whose staging branch has another name lists that branch instead of `staging`.
An app without a staging service can drop it, since a push to it fails the plan.
Both callers share one concurrency group per branch, so a rollback never runs alongside a deploy of main.
GitHub keeps only the newest pending run in a group, so a deploy queued behind a running one replaces a rollback that was waiting too.

## What the registry provides

`PLATFORM` is JSON with these fields:

- `project`, `region` and `registry`, the app's project and its Artifact Registry repository.
- `workload_identity_provider` and `service_account`, the app's deploy identity.
- `images`, each image key and its Dockerfile path, where `app` is always `Dockerfile`.
- `staging_branch`, or null when the app has no staging service.
- `build_args`, the extra Docker build args for `production` and for `staging`.
- `services`, each service's name, environment, runtime service account, full environment, Cloud SQL connection, health path and URL.
- `jobs`, each job's name, the image key it runs, and its `command` and `args`, each null or a list.

The workload identity pool only issues tokens to runs of these two workflows at `refs/tags/v1`, from repositories in the organization.
The app's deploy service account only trusts its own repository.
It can push and tag images in the app's registry, deploy the app's services, update the app's jobs, and run as the app's runtime service accounts, and nothing else.

## What a deploy does

A push to main deploys production, and a push to the staging branch deploys staging.
Any other ref fails.

1. **Plan** checks `PLATFORM` and the branch, then works out the environment, images, services and jobs to deploy.
2. **Scan** runs gitleaks over the commit's full history and `npm audit --audit-level=critical` in every folder with a tracked `package-lock.json`.
3. **Deploy** runs once both pass:
   1. It builds every image, or only `app` for staging, with the environment's build args plus `APP_VERSION`, the commit, and `APP_ENV`, the environment.
   2. It signs in to Google Cloud as the app's deploy service account.
   3. It pushes each image, tagged `<commit>-<environment>`.
   4. It deploys each of the environment's services as a new revision without traffic, with the new image by digest, the runtime service account, the full environment, the Cloud SQL connection and a startup probe on the health path.
      An app without a health path gets Cloud Run's default TCP probe.
      Cloud Run waits for the probe, so a release that never gets healthy stops here, while the old revisions keep serving.
   5. In production, it points every job at its new image, with the job's `command` and `args`.
   6. It sends all traffic to the new revisions.
   7. It tags the release, as described below.
   8. It writes a summary with the services' URLs and the images' digests.

Everything else about services and jobs, such as scaling, ingress, sign-in, schedules and job settings, belongs to the registry's Terraform.

## Releases and tags

Each push of an image is tagged `<commit>-<environment>`.
After a deploy, the tag `<environment>` marks the image the environment runs, and `<environment>-previous` the one it ran before.
The registry keeps every image tagged `production`, `production-previous`, `staging` or `staging-previous`, so the current and previous release of each environment stay available.
Other images are deleted once they are 30 days old, except the ten newest.
Rerunning a deploy of the commit an environment already runs only moves `<environment>`, so `<environment>-previous` still points at the release before it.

## Rolling back

To roll production back, run **Roll back** from main in the app repo's Actions tab.
It moves each image back to its `production-previous` tag, deploys that image to every production service and job that uses it, and moves `production` to it.
It changes the code only, so the services keep the environment, Cloud SQL connection and probe of the latest deploy, and the jobs keep its `command` and `args`.
`production-previous` stays where it is, so a second rollback in a row finds nothing to roll back to and fails.
To go back further, revert the change on main, which deploys like any other change.
The next deploy from main moves forward as usual.

To roll staging back, push the commit you want to the staging branch:

```sh
git push --force-with-lease origin <commit>:staging
```

## Security model

Each run splits into jobs with only the access they need.
Plan has no permissions and never checks out code.
Scan can read the repo but can't reach Google Cloud.
Deploy is the only job that can get a Google token, and it runs only after both pass.

Deploy builds every image before it signs in.
That keeps Google credentials away from every Dockerfile step and dependency install, and keeps the credentials file the auth action writes into the workspace out of every build context.

`PLATFORM` is untrusted, since anyone who can change the app repo's variables can change it.
Plan checks every field the workflows use against a strict pattern and fails closed, with errors that name fields but never print values.
Values reach scripts only through environment variables and jq, never through expressions inside a script.

The real boundary is the deploy identity, which can only act for its own app.
A changed `PLATFORM` can at worst break that app's own deploys.
The branch checks only catch mistakes, since the staging branch itself comes from `PLATFORM`.
Likewise, the scans catch accidents, not the app's own maintainers.

Actions are pinned to full commit SHAs, checkouts don't keep the GitHub token, the gitleaks download is checked against a pinned SHA-256, and every job has a timeout.
Only the tech team can push to this repo or move `v1`, because every app's deploy identity trusts whatever `v1` points at.
This repo is public, so any repository can call its workflows, but Google issues a token only to runs from repositories in the organization.
That keeps the workflows free to share without an Actions access setting.

gitleaks honors a `.gitleaks.toml` and a `.gitleaksignore` in the app repo.
When it finds a real secret, rotate the secret first, then add the finding's fingerprint to `.gitleaksignore`, since the commit stays in the history.

## Releasing a change

Lint every change with [actionlint](https://github.com/rhysd/actionlint), with shellcheck installed so it checks the scripts too.
Then move `v1` to the reviewed commit on main:

```sh
git tag --force v1
git push --force origin v1
```

Every app picks up the change on its next deploy or rollback.
A breaking change needs a new major tag and a registry change, because the workload identity pool trusts `v1` only.

## Upgrades

To upgrade an action, pin the new release's full commit SHA and update the version comment beside it.
To upgrade gitleaks, set `GITLEAKS_VERSION`, and set `GITLEAKS_SHA256` to the `linux_x64` tarball's line in that release's checksums file.
The runner image is pinned to `ubuntu-24.04`, so moving to a newer one is a deliberate change.
Check that the new image still ships jq 1.7 or newer, bash 5, Docker, Node with npm, and the Google Cloud CLI.
