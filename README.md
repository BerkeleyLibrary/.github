# Berkeley Library: Shared GitHub Actions Workflows

Shared workflows for Berkeley Library GitHub repositories.

## Docker Workflows

Reusable `workflow_call` workflows for multi-arch (ARM64/x64) Docker image
CI/CD. Calling repos compose these stages in their own workflows.

### CI Pipeline

The CI pipeline builds, tests, and pushes images on every branch push. The
[`docker-ci` workflow template](workflow-templates/docker-ci.yml) demonstrates
how to weave together the shared build and push workflows with a custom test
job.

| Workflow | Inputs | Secrets | Outputs | Purpose |
|----------|--------|---------|---------|---------|
| [`docker-build`](.github/workflows/docker-build.yml) | `image` | `registry-auth` (optional) | `image`, `image-arm64`, `image-x64` | Build on native ARM64 + x64 runners, merge into a multi-platform manifest with a temporary build tag |
| [`docker-push`](.github/workflows/docker-push.yml) | `image`, `build-image-arm64`, `build-image-x64` | — | — | Retag the build digests with permanent tags (`sha-*`, branch name, `latest` on default branch) |

The **test** job is defined by the calling repo (not a shared workflow) so each
project can customize its test setup. The `docker-ci` template provides a
starting point that pulls the built image via `docker compose` and runs a `test`
command in the app container.

#### Private base images

`docker-build` always authenticates to GHCR with the calling workflow's
`GITHUB_TOKEN`. To pull private base images, pass any additional credentials as
YAML through the optional `registry-auth` secret:

```yaml
jobs:
  build:
    uses: BerkeleyLibrary/.github/.github/workflows/docker-build.yml@main
    with:
      image: ghcr.io/${{ github.repository }}
    secrets:
      registry-auth: |
        - registry: ghcr.io
          username: ${{ secrets.PRIVATE_REGISTRY_USERNAME || github.actor }}
          password: ${{ secrets.PRIVATE_REGISTRY_TOKEN }}
          scope: someorg/private/reg@pull
```

`registry-auth` accepts any number of entries. Each entry supports the
`registry`, `username`, `password`, `scope`, and `ecr` fields from
[`docker/login-action`](https://github.com/docker/login-action#customizing).
Use a `scope` ending in `@pull` when the credential only needs to pull a private
base image. Unscoped credentials are available to all Docker commands in the
build job; scoped credentials are available only to Buildx.

### Release Pipeline

The release pipeline runs on git tags. It finds the existing `sha-*` image
(previously pushed by CI) and retags it with semver tags — no rebuild required.

| Workflow | Inputs | Outputs | Purpose |
|----------|--------|---------|---------|
| [`docker-release`](.github/workflows/docker-release.yml) | `image` | — | Retag the `sha-*` image with version tags (`v1`, `v1.2`, `v1.2.3`, full tag) |

---

> **Semver Suffixes:** For prerelease tags (e.g. `v1.2.3-rc.1`),
> `docker/metadata-action@v6`'s `semver` tag type deliberately emits only the
> full prerelease version; it does not generate major, minor, or patch tags.
> The release workflow uses custom `match` regexes to generate those tags as
> well as the full tag.

---

> **Optional "v" Prefix:** docker-release allows but ignores a leading "v" on a semver
> tag. For example, the tags "v1.2.3" and "1.2.3" will both result in retagging
> the underlying image with "1", "1.2", and "1.2.3" (without the leading "v").

---

#### Example release workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - '**'
  workflow_dispatch:

jobs:
  release:
    uses: BerkeleyLibrary/.github/.github/workflows/docker-release.yml@main
    with:
      image: ghcr.io/${{ github.repository }}
```

### Full Lifecycle

```mermaid
flowchart LR
    subgraph Commits
        build[docker-build]

        trigger{git-commit} --> |triggers| build

        build -->buildx64[[build-x64]]
        build -->buildarm64[[build-arm64]]

        merge[[merge]]
        buildx64 -->|x64-image-digest|merge
        buildarm64 -->|arm64-image-digest|merge
        merge -->|"build.image"|test{{"test\n(implemented by caller)"}}

        test -->push[docker-push]

        push --> commitTags[["permatag\n(sha,branch,latest)"]]
        commitTags --> reg1[("registry (ghcr.io)")]
    end

    subgraph Tags
        git-tag{git-tag} -->|triggers| release[docker-release]
        release --> verify[[sha-tag exists?]]
        verify --> semverTags[[retag]]
        semverTags --> reg2[("registry (ghcr.io)")]
    end

    Commits .- Tags
```

### Caller Requirements

- A `Dockerfile` at the repo root
- A `compose.yml` and `compose.ci.yml` for testing (if using the compose-based test pattern)
- `packages: write` permission on the calling workflow

The reusable workflows receive the caller's `GITHUB_TOKEN` automatically, so
callers do not need `secrets: inherit`. Pass only the optional `registry-auth`
secret when the build requires additional registry credentials.
