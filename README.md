# BedrockNexus GitHub Actions

Central, reusable CI/CD workflows for the BedrockNexus apps: [hub](https://github.com/BedrockNexus/hub), [plugins](https://github.com/BedrockNexus/plugins) and [api](https://github.com/BedrockNexus/api).

Every BedrockNexus app ships as a Docker image. This repository owns *how* that image is built, published and deployed, so application repositories contain almost no CI/CD logic: just a `Dockerfile` and a few lines of workflow YAML.

This repository is **public** on purpose. The application repositories are public, and GitHub only lets public repositories call reusable workflows that live in public repositories. The workflows contain no secrets: Coolify credentials stay in GitHub Actions secrets and are passed in at run time.

## Architecture

```
GitHub repository (push to main)
        ↓
Reusable GitHub Actions (this repository)
        ↓
Docker Buildx / BuildKit (GitHub Actions cache)
        ↓
GitHub Container Registry: ghcr.io/bedrocknexus/<repo>
        ↓
Coolify deploy webhook
        ↓
Hetzner production server (Coolify pulls the image)
```

Coolify **never builds source code**. It is configured as a *Docker Image* deployment and only pulls the image that GitHub Actions has already built and pushed.

The deployment sequence is strictly:

```
checkout → validation (optional) → docker build → docker push → Coolify webhook
```

Coolify is triggered in a separate job that depends on the build job. If checkout, validation, the build, GHCR login or the push fails, the webhook is never called.

## Workflows

| Workflow | Purpose | Pushes | Deploys | Secrets |
| --- | --- | --- | --- | --- |
| [`docker-deploy.yml`](.github/workflows/docker-deploy.yml) | Production deployment | ✅ | ✅ | `COOLIFY_WEBHOOK`, `COOLIFY_TOKEN` |
| [`docker-pr.yml`](.github/workflows/docker-pr.yml) | Pull-request validation | ❌ | ❌ | none |
| [`docker-build.yml`](.github/workflows/docker-build.yml) | Shared build engine used by both of the above | optional | ❌ | none (uses `GITHUB_TOKEN`) |

`docker-pr.yml` and `docker-deploy.yml` both call `docker-build.yml`, so all build logic lives in one place.

## Quick start

### Production deployment

`.github/workflows/deploy.yml` in the application repository:

```yaml
name: Deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  deploy:
    uses: BedrockNexus/github-actions/.github/workflows/docker-deploy.yml@main
    secrets: inherit
```

The `permissions` block is required. A reusable workflow can only use the permissions its caller grants, and GitHub's default token permissions for new repositories and organizations are read-only, so `packages: write` must be granted here for the push to GHCR to work.

### Pull-request validation

`.github/workflows/pull-request.yml`:

```yaml
name: Pull Request

on:
  pull_request:

permissions:
  contents: read

jobs:
  docker:
    uses: BedrockNexus/github-actions/.github/workflows/docker-pr.yml@main
```

This builds the production image exactly as a deployment would, and fails the check if the image does not build. It never logs in to GHCR, never pushes, never receives secrets and never contacts Coolify. Protect `main` with a branch rule that requires this check.

## Inputs

All inputs are optional. `docker-deploy.yml` and `docker-pr.yml` accept the same build inputs and forward them to `docker-build.yml`.

| Input | Default | Available in | Description |
| --- | --- | --- | --- |
| `image-name` | `ghcr.io/<owner>/<repo>` | all | Full image name without a tag. Lowercased automatically. |
| `dockerfile` | `./Dockerfile` | all | Dockerfile path relative to the repository root. |
| `context` | `.` | all | Build context relative to the repository root. |
| `platforms` | `linux/amd64` | all | Comma-separated platforms, e.g. `linux/amd64,linux/arm64`. QEMU is set up automatically for non-default platforms. |
| `target` | last stage | all | Dockerfile stage to build as the image. |
| `test-target` | none | all | Dockerfile stage built **before** the image as a validation step (see [Validation](#validation)). |
| `build-args` | none | all | Newline-separated `KEY=VALUE` build arguments. |
| `runner` | `BEDROCKNEXUS_RUNNER` variable, else `ubuntu-latest` | all | Runner label or JSON array of labels for the build job. |
| `environment` | `production` | deploy | Deployment environment name; scopes concurrency. |
| `push` | `false` | build | Push to GHCR. Only relevant when calling `docker-build.yml` directly. |

### Outputs

`docker-build.yml` exposes `image`, `tags`, `version` and `digest`. `docker-deploy.yml` exposes `image`, `tags` and `digest`. `digest` is empty when the image was not pushed.

## Examples

### Custom Dockerfile location

```yaml
jobs:
  deploy:
    uses: BedrockNexus/github-actions/.github/workflows/docker-deploy.yml@main
    with:
      dockerfile: ./docker/production.Dockerfile
    secrets: inherit
```

### Custom build context (monorepo)

```yaml
jobs:
  deploy-api:
    uses: BedrockNexus/github-actions/.github/workflows/docker-deploy.yml@main
    with:
      context: ./apps/api
      dockerfile: ./apps/api/Dockerfile
      image-name: ghcr.io/bedrocknexus/hub-worker
      environment: production-api
    secrets:
      COOLIFY_WEBHOOK: ${{ secrets.COOLIFY_WEBHOOK_API }}
      COOLIFY_TOKEN: ${{ secrets.COOLIFY_TOKEN }}
```

When one repository deploys several images, give each its own `image-name` (separate GHCR packages and build caches) and its own `environment` (separate concurrency groups, so they do not cancel each other).

### Custom image name

```yaml
    with:
      image-name: ghcr.io/bedrocknexus/hub-preview
```

The image must live under `ghcr.io/bedrocknexus/...`: the workflow authenticates to GHCR with the repository's `GITHUB_TOKEN`.

### Build arguments

```yaml
    with:
      build-args: |
        NEXT_PUBLIC_SITE_URL=https://example.com
        NODE_VERSION=22
```

Build arguments are stored in the image metadata and visible to anyone who can pull the image. **Never pass secrets as build arguments.** Runtime secrets belong in Coolify's environment variables.

### Explicit Coolify secrets

Use this when the secret names in the application repository differ, or when you prefer to be explicit:

```yaml
jobs:
  deploy:
    uses: BedrockNexus/github-actions/.github/workflows/docker-deploy.yml@main
    secrets:
      COOLIFY_WEBHOOK: ${{ secrets.COOLIFY_WEBHOOK }}
      COOLIFY_TOKEN: ${{ secrets.COOLIFY_TOKEN }}
```

### Inherited secrets

```yaml
jobs:
  deploy:
    uses: BedrockNexus/github-actions/.github/workflows/docker-deploy.yml@main
    secrets: inherit
```

`secrets: inherit` passes all of the caller's repository and organization secrets to the reusable workflow. It works because the caller and this repository belong to the same organization. It is the simplest option when `COOLIFY_WEBHOOK` and `COOLIFY_TOKEN` exist with exactly those names. A common setup is `COOLIFY_TOKEN` as an organization secret and `COOLIFY_WEBHOOK` as a repository secret, since the webhook is different for each application.

### Staging environment

```yaml
on:
  push:
    branches:
      - develop

permissions:
  contents: read
  packages: write

jobs:
  deploy:
    uses: BedrockNexus/github-actions/.github/workflows/docker-deploy.yml@main
    with:
      environment: staging
    secrets:
      COOLIFY_WEBHOOK: ${{ secrets.COOLIFY_WEBHOOK_STAGING }}
      COOLIFY_TOKEN: ${{ secrets.COOLIFY_TOKEN }}
```

Point the staging Coolify application at the `develop` tag.

### Publishing release images

Git tags produce version tags (`v1.0.0`) but do not move `latest`. To publish release images without deploying:

```yaml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read
  packages: write

jobs:
  image:
    uses: BedrockNexus/github-actions/.github/workflows/docker-build.yml@main
    with:
      push: true
```

## Validation

The Dockerfile is the only interface between an application and this pipeline, and it is also where validation lives. Choose either approach:

1. **Validation inside the build.** Run lint, type checks or tests in a `RUN` step of the build stage. If they fail, the build fails and nothing is pushed or deployed.
2. **A dedicated test stage.** Add a stage to the Dockerfile and pass `test-target`:

   ```dockerfile
   FROM base AS test
   RUN bun run lint && bun run typecheck && bun test
   ```

   ```yaml
       with:
         test-target: test
   ```

   The test stage is built first. Only if it succeeds is the production image built, pushed and deployed. Both stages share the build cache.

If a project needs checks that cannot run in Docker, add its own job in the caller and gate the deployment with `needs:`:

```yaml
jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: ./scripts/check.sh

  deploy:
    needs: checks
    uses: BedrockNexus/github-actions/.github/workflows/docker-deploy.yml@main
    secrets: inherit
```

## Required project files

Each application repository provides:

- **`Dockerfile`** - defines how the application is built and run. This repository does not know or care whether the project uses Next.js, Bun, Hono, Payload CMS, React, a static site or a worker.
- **`.dockerignore`** - keeps `node_modules`, `.git`, `.env*`, build output and other local artifacts out of the build context. This makes builds faster, keeps cache hits stable and prevents local secrets from leaking into images.

If `docker build .` works locally, the pipeline can build it.

## Required secrets

| Secret | Used by | Description |
| --- | --- | --- |
| `COOLIFY_WEBHOOK` | `docker-deploy.yml` | The application's deploy webhook URL. In Coolify: **Application → Webhooks → Deploy Webhook**. It has the form `https://<coolify-host>/api/v1/deploy?uuid=<application-uuid>&force=false`. |
| `COOLIFY_TOKEN` | `docker-deploy.yml` | A Coolify API token with **deploy** permission. In Coolify: **Keys & Tokens → API tokens**. Sent as `Authorization: Bearer <token>`. |

Store both as GitHub Actions secrets (organization or repository level). Never commit them, and never hardcode Coolify URLs, UUIDs or tokens in workflows.

GHCR authentication needs **no secret**: the workflow uses the built-in `GITHUB_TOKEN`, scoped to `packages: write` for the push job only. No personal access token is involved in CI.

## GHCR

Images are published to:

```
ghcr.io/<organization>/<repository>
```

For example, `BedrockNexus/hub` becomes `ghcr.io/bedrocknexus/hub`. Image names are lowercased automatically because registries require it.

The first push creates the package in the organization and links it to the source repository through the `org.opencontainers.image.source` label. The package inherits access from that repository.

## Image tagging

Tags are generated by [`docker/metadata-action`](https://github.com/docker/metadata-action):

| Event | Tags |
| --- | --- |
| Push to the default branch (`main`) | `latest`, `main`, `sha-<short-sha>` |
| Push to another branch (`develop`) | `develop`, `sha-<short-sha>` |
| Git tag `v1.0.0` | `v1.0.0`, `sha-<short-sha>` |
| Pull request (built, never pushed) | `pr-<number>`, `sha-<short-sha>` |

- **`latest`** / **`main`** are moving tags. Coolify tracks one of them.
- **`sha-xxxxxxx`** is immutable. It identifies exactly which commit is inside an image, which makes it the tag to use for audits, debugging and **rollbacks**: set Coolify's image tag to a previous `sha-…` tag and redeploy. No rebuild is needed.
- **`v1.0.0`** mirrors Git tags and never moves `latest`.

Images also carry OCI labels and annotations such as `org.opencontainers.image.source`, `org.opencontainers.image.revision`, `org.opencontainers.image.created` and `org.opencontainers.image.version`.

## Coolify setup

For each application:

1. Create the resource as a **Docker Image** deployment, not a Git-based or Dockerfile build. Coolify must never build source code.
2. Set the image to `ghcr.io/bedrocknexus/<repository>` and the tag to `latest` (production) or to the branch tag for other environments (e.g. `develop`).
3. If the GHCR package is private, log the Coolify server in to GHCR once (see [Private GHCR access](#private-ghcr-access)).
4. Copy the **Deploy Webhook** URL into the repository's `COOLIFY_WEBHOOK` secret.
5. Create an API token with deploy permission and store it as `COOLIFY_TOKEN`.
6. Disable Coolify's own Git auto-deploy for the resource, so GitHub Actions is the only thing that triggers deployments.

When the webhook fires, Coolify pulls the configured tag, which the workflow has just pushed, and restarts the application.

## Concurrency

Deployments are grouped by repository and `environment`. When a newer commit is pushed while an older deployment is still running, the older run is cancelled. The newest commit is always the one that gets built, pushed and deployed, and two deployments of the same environment never race. Different environments (`production`, `staging`) do not affect each other.

## Security

- **Minimal permissions.** Each job requests only what it needs: the build job gets `contents: read` plus `packages: write` when pushing, and the Coolify job gets no token permissions.
- **Secrets only where needed.** Only the final Coolify job receives `COOLIFY_WEBHOOK` and `COOLIFY_TOKEN`. The build job, which runs application code during `docker build`, never sees them.
- **No secrets in logs.** Secrets are passed through environment variables, never interpolated into scripts, and the bearer token is sent to `curl` over stdin so it does not appear in the runner's process list.
- **Pull requests cannot deploy.** `docker-pr.yml` takes no secrets and never pushes. `docker-build.yml` refuses to push from `pull_request` events and refuses to run under `pull_request_target` at all.
- **Forks are isolated.** Fork pull requests always build on GitHub-hosted runners, even if a self-hosted runner is configured.
- **Checkout credentials are not persisted.** The repository token is not left on disk for the Docker build to read.
- **Pinned actions.** Third-party actions are pinned to full commit SHAs, and Dependabot keeps them updated (see [Maintenance](#maintenance)).

## Self-hosted build runner

The build job's runner is resolved in this order:

1. the `runner` input, if a caller sets it
2. the `BEDROCKNEXUS_RUNNER` Actions variable (organization or repository level)
3. `ubuntu-latest`

To move every BedrockNexus project to a self-hosted builder at once, **without changing any application repository or this repository**, create an organization Actions variable:

```
BEDROCKNEXUS_RUNNER = ["self-hosted","linux","x64","bedrocknexus-builder"]
```

A plain label (`bedrocknexus-builder`) or a JSON array of labels are both accepted. To roll back, delete the variable. To pilot the change on a single project first, set the variable on that repository only, or pass `runner:` in its workflow.

Notes for the runner:

- It needs Docker installed and GitHub Actions runner v2.327.1 or newer (the pinned actions run on Node 24). Buildx is set up by the workflow.
- The GitHub Actions cache works from self-hosted runners, so cache behavior is unchanged.
- Fork pull requests are forced onto `ubuntu-latest` so untrusted code never runs on BedrockNexus infrastructure.
- The Coolify trigger job always runs on `ubuntu-latest`. It only makes one HTTPS request.

## Troubleshooting

### `denied: permission_denied` / `installation not allowed to Write organization package`
The push job lacks `packages: write`. Add the `permissions` block from the [quick start](#production-deployment) to the caller. If the package already exists, open it in GitHub (**Package settings → Manage Actions access**) and give the repository **Write** access. Packages created by a different repository are not writable by default.

### `The nested job 'build' is requesting 'packages: write', but is only allowed 'packages: read'`
Same cause: the calling workflow must grant `packages: write`.

### GHCR image inaccessible / `unauthorized` when Coolify pulls
The package is private and the Coolify server is not logged in. See [Private GHCR access](#private-ghcr-access). Also check that the image name and tag configured in Coolify exist in GHCR.

### Private GHCR access
Private packages need registry credentials on the Coolify server. Create a GitHub token with only the `read:packages` scope (ideally from a dedicated machine user) and log in on the server:

```bash
echo "<token>" | docker login ghcr.io -u <github-username> --password-stdin
```

Coolify uses the server's Docker credentials when pulling. This token is only used on the server and never in CI.

### Dockerfile build failures
Reproduce locally with `docker build -f Dockerfile .` from the same context. Common causes are a missing `.dockerignore` (huge contexts, leaked `node_modules`), files excluded by `.dockerignore` that the build needs, or build-time environment variables that the application expects but that are only defined in Coolify at runtime.

### `Unable to resolve action` / `workflow was not found`
Check the `uses:` path and the ref (`@main`). This repository must stay public for the public application repositories to call it.

### Coolify webhook authentication failures (HTTP 401/403)
`COOLIFY_TOKEN` is missing, revoked or lacks **deploy** permission, or it belongs to a different Coolify team than the application. The image was pushed. After fixing the token, re-run the failed job to redeploy without rebuilding.

### Coolify returns 404
The application UUID in `COOLIFY_WEBHOOK` is wrong or the application was recreated. Copy the webhook again from Coolify.

### `COOLIFY_WEBHOOK and COOLIFY_TOKEN must both be set`
The secrets are not visible to the caller. Check their names and, for organization secrets, that the repository is in the secret's access list. When mapping explicitly, check the right-hand side names.

### Coolify deployed an old version / incorrect image tags
Coolify is tracking a tag the workflow did not move. `latest` only moves on default-branch builds, and Git tags only create `vX.Y.Z` tags. Check the job summary for the exact tags pushed, and make sure Coolify's tag matches the branch that deploys.

### Docker build cache behavior
Layers are cached in the GitHub Actions cache (`type=gha`, `mode=max`), scoped per image and target. Things to know:

- Caches are scoped by branch. Pull requests read the base branch's cache but cannot overwrite it.
- The first build on a new repository or image name is cold.
- A single `COPY . .` early in the Dockerfile invalidates every later layer on each commit. Copy lockfiles and install dependencies *before* copying the source.
- The repository cache limit is 10 GB by default. Old entries are evicted automatically.
- A cache export failure is ignored, so it never fails an otherwise successful deployment.
- To force a clean build, delete the cache entries under **Actions → Caches** in the application repository.

## Maintenance

- **Versioning.** Callers reference `@main`, so a change here applies to every app on its next run. Test changes on a branch first by pointing one app's workflow at `@<branch>`. If you ever need stricter stability, tag a release (for example `v1`) and point the callers at it.
- **Linting.** [`lint.yml`](.github/workflows/lint.yml) runs actionlint on every push and pull request.
- **Action updates.** Actions are pinned to commit SHAs with the version in a trailing comment. Dependabot ([`.github/dependabot.yml`](.github/dependabot.yml)) opens a weekly grouped PR that updates both the SHA and the comment.
- **Scope.** Keep this repository framework-agnostic. Anything specific to one application belongs in that application's `Dockerfile`.

## License

[MIT](LICENSE)
