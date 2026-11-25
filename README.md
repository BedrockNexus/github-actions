# BedrockNexus GitHub Actions

Central, reusable CI workflows for the BedrockNexus apps: [hub](https://github.com/BedrockNexus/hub) and [api](https://github.com/BedrockNexus/api).

Every BedrockNexus app ships as a Docker image. This repository owns *how* that image is built, validated and published, so application repositories contain almost no CI logic: just a `Dockerfile` and a few lines of workflow YAML.

This repository is **public** on purpose. The application repositories are public, and GitHub only lets public repositories call reusable workflows that live in public repositories. The workflows take no secrets.

## Architecture

```
GitHub repository (push to main)
        ↓
Reusable GitHub Actions (this repository)
        ↓
Docker Buildx / BuildKit (GitHub Actions cache)
        ↓
GitHub Container Registry: ghcr.io/bedrocknexus/<repo>
```

The pipeline stops at the registry. Deploying is not part of CI: whoever runs an app pulls the published image (or builds the `Dockerfile` themselves) and runs it. The sequence is:

```
checkout → validation (optional) → docker build → docker push
```

If checkout, validation, the build or the GHCR login fails, nothing is pushed.

## Workflows

| Workflow | Purpose | Pushes | Secrets |
| --- | --- | --- | --- |
| [`docker-build.yml`](.github/workflows/docker-build.yml) | Build the image, optionally pushing it to GHCR | optional | none (uses `GITHUB_TOKEN`) |
| [`docker-pr.yml`](.github/workflows/docker-pr.yml) | Pull-request validation | ❌ | none |

`docker-pr.yml` calls `docker-build.yml`, so all build logic lives in one place.

## Quick start

### Publishing the image

`.github/workflows/image.yml` in the application repository:

```yaml
name: Image

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  packages: write

concurrency:
  group: image-${{ github.ref }}
  cancel-in-progress: true

jobs:
  image:
    uses: BedrockNexus/github-actions/.github/workflows/docker-build.yml@main
    with:
      push: true
```

The `permissions` block is required. A reusable workflow can only use the permissions its caller grants, and GitHub's default token permissions for new repositories and organizations are read-only, so `packages: write` must be granted here for the push to GHCR to work.

The `concurrency` block cancels an older run when a newer commit lands, so the newest commit is always the one published.

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

This builds the production image exactly as `main` would, and fails the check if the image does not build. It never logs in to GHCR, never pushes and never receives secrets. Protect `main` with a branch rule that requires this check.

## Inputs

All inputs are optional. `docker-pr.yml` accepts the same build inputs, except `push`, and forwards them to `docker-build.yml`.

| Input | Default | Description |
| --- | --- | --- |
| `image-name` | `ghcr.io/<owner>/<repo>` | Full image name without a tag. Lowercased automatically. |
| `dockerfile` | `./Dockerfile` | Dockerfile path relative to the repository root. |
| `context` | `.` | Build context relative to the repository root. |
| `platforms` | `linux/amd64` | Comma-separated platforms, e.g. `linux/amd64,linux/arm64`. QEMU is set up automatically for non-default platforms. |
| `target` | last stage | Dockerfile stage to build as the image. |
| `test-target` | none | Dockerfile stage built **before** the image as a validation step (see [Validation](#validation)). |
| `build-args` | none | Newline-separated `KEY=VALUE` build arguments. |
| `runner` | `BEDROCKNEXUS_RUNNER` variable, else `ubuntu-latest` | Runner label or JSON array of labels for the build job. |
| `push` | `false` | Push to GHCR. Refused on `pull_request` events. |

### Outputs

`docker-build.yml` exposes `image`, `tags`, `version` and `digest`. `digest` is empty when the image was not pushed.

## Examples

### Several images from one repository

```yaml
jobs:
  api:
    uses: BedrockNexus/github-actions/.github/workflows/docker-build.yml@main
    with:
      push: true
      target: runner
      test-target: test

  validator:
    needs: api
    uses: BedrockNexus/github-actions/.github/workflows/docker-build.yml@main
    with:
      push: true
      target: validator
      image-name: ghcr.io/bedrocknexus/api-validator
```

Give each image its own `image-name`, so they get separate GHCR packages and build caches.

### Custom Dockerfile location or context

```yaml
    with:
      push: true
      context: ./apps/api
      dockerfile: ./apps/api/Dockerfile
```

The image must live under `ghcr.io/bedrocknexus/...`: the workflow authenticates to GHCR with the repository's `GITHUB_TOKEN`.

### Build arguments

```yaml
    with:
      build-args: |
        NEXT_PUBLIC_SITE_URL=https://example.com
        NODE_VERSION=22
```

Build arguments are stored in the image metadata and visible to anyone who can pull the image. **Never pass secrets as build arguments.** Runtime secrets belong in the environment of whatever runs the container.

### Release images

Git tags produce version tags (`v1.0.0`) but do not move `latest`:

```yaml
on:
  push:
    tags:
      - "v*"
```

## Validation

The Dockerfile is the only interface between an application and this pipeline, and it is also where validation lives. Choose either approach:

1. **Validation inside the build.** Run lint, type checks or tests in a `RUN` step of the build stage. If they fail, the build fails and nothing is pushed.
2. **A dedicated test stage.** Add a stage to the Dockerfile and pass `test-target`:

   ```dockerfile
   FROM base AS test
   RUN bun run lint && bun run typecheck && bun test
   ```

   ```yaml
       with:
         test-target: test
   ```

   The test stage is built first. Only if it succeeds is the image built and pushed. Both stages share the build cache.

If a project needs checks that cannot run in Docker, add its own job in the caller and gate the image with `needs:`.

## Required project files

Each application repository provides:

- **`Dockerfile`** - defines how the application is built and run. This repository does not know or care whether the project uses Next.js, Bun, Hono, a static site or a worker.
- **`.dockerignore`** - keeps `node_modules`, `.git`, `.env*`, build output and other local artifacts out of the build context. This makes builds faster, keeps cache hits stable and prevents local secrets from leaking into images.

If `docker build .` works locally, the pipeline can build it.

## GHCR

Images are published to `ghcr.io/<organization>/<repository>`. For example, `BedrockNexus/hub` becomes `ghcr.io/bedrocknexus/hub`. Image names are lowercased automatically because registries require it.

The first push creates the package in the organization and links it to the source repository through the `org.opencontainers.image.source` label. The package inherits access from that repository.

GHCR authentication needs **no secret**: the workflow uses the built-in `GITHUB_TOKEN`. No personal access token is involved in CI.

## Image tagging

Tags are generated by [`docker/metadata-action`](https://github.com/docker/metadata-action):

| Event | Tags |
| --- | --- |
| Push to the default branch (`main`) | `latest`, `main`, `sha-<short-sha>` |
| Push to another branch (`develop`) | `develop`, `sha-<short-sha>` |
| Git tag `v1.0.0` | `v1.0.0`, `sha-<short-sha>` |
| Pull request (built, never pushed) | `pr-<number>`, `sha-<short-sha>` |

- **`latest`** / **`main`** are moving tags.
- **`sha-xxxxxxx`** is immutable. It identifies exactly which commit is inside an image, which makes it the tag to use for audits, debugging and **rollbacks**: run an earlier `sha-…` tag. No rebuild is needed.
- **`v1.0.0`** mirrors Git tags and never moves `latest`.

Images also carry OCI labels and annotations such as `org.opencontainers.image.source`, `org.opencontainers.image.revision`, `org.opencontainers.image.created` and `org.opencontainers.image.version`.

## Security

- **Minimal permissions.** The build job gets `contents: read`, plus `packages: write` only when the caller grants it for pushing.
- **No secrets.** The workflows take none, so application code run during `docker build` has nothing to read.
- **Pull requests cannot push.** `docker-pr.yml` never pushes. `docker-build.yml` refuses to push from `pull_request` events and refuses to run under `pull_request_target` at all.
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
- Do not use a production server as the runner: image builds need far more memory than the apps do.

## Troubleshooting

### `denied: permission_denied` / `installation not allowed to Write organization package`
The build job lacks `packages: write`. Add the `permissions` block from the [quick start](#publishing-the-image) to the caller. If the package already exists, open it in GitHub (**Package settings → Manage Actions access**) and give the repository **Write** access. Packages created by a different repository are not writable by default.

### `The nested job 'build' is requesting 'packages: write', but is only allowed 'packages: read'`
Same cause: the calling workflow must grant `packages: write`.

### `unauthorized` when pulling the image
The package is private. Make it public in **Package settings**, or log the pulling machine in to GHCR with a token that has only the `read:packages` scope:

```bash
echo "<token>" | docker login ghcr.io -u <github-username> --password-stdin
```

### Dockerfile build failures
Reproduce locally with `docker build -f Dockerfile .` from the same context. Common causes are a missing `.dockerignore` (huge contexts, leaked `node_modules`), files excluded by `.dockerignore` that the build needs, or build-time environment variables that the application expects but that are only defined at runtime.

### `Unable to resolve action` / `workflow was not found`
Check the `uses:` path and the ref (`@main`). This repository must stay public for the public application repositories to call it.

### Docker build cache behavior
Layers are cached in the GitHub Actions cache (`type=gha`, `mode=max`), scoped per image and target. Things to know:

- Caches are scoped by branch. Pull requests read the base branch's cache but cannot overwrite it.
- The first build on a new repository or image name is cold.
- A single `COPY . .` early in the Dockerfile invalidates every later layer on each commit. Copy lockfiles and install dependencies *before* copying the source.
- The repository cache limit is 10 GB by default. Old entries are evicted automatically.
- A cache export failure is ignored, so it never fails an otherwise successful build.
- To force a clean build, delete the cache entries under **Actions → Caches** in the application repository.

## Maintenance

- **Versioning.** Callers reference `@main`, so a change here applies to every app on its next run. Test changes on a branch first by pointing one app's workflow at `@<branch>`. If you ever need stricter stability, tag a release (for example `v1`) and point the callers at it.
- **Linting.** [`lint.yml`](.github/workflows/lint.yml) runs actionlint on every push and pull request.
- **Action updates.** Actions are pinned to commit SHAs with the version in a trailing comment. Dependabot ([`.github/dependabot.yml`](.github/dependabot.yml)) opens a weekly grouped PR that updates both the SHA and the comment.
- **Scope.** Keep this repository framework-agnostic. Anything specific to one application belongs in that application's `Dockerfile`.

## License

[MIT](LICENSE)
