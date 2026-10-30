# shared-github-workflows

Reusable GitHub Actions workflows for CI/CD pipelines, designed to streamline automation across multiple projects and promote consistency.

Each workflow lives in `.github/workflows/` and is triggered with `workflow_call`, so other repositories can call it with a few lines of YAML instead of copying the whole pipeline.

## Workflows

### `docker-build-push.yml`

Builds a Docker image from the repository root, scans it for vulnerabilities and publishes it to a container registry.

It runs these steps:

1. Checks out the repository.
2. Logs into the container registry.
3. Generates the image tags and labels with [docker/metadata-action](https://github.com/docker/metadata-action), using its default rules (the Git tag on tag pushes, `pr-<number>` on pull requests, the branch name on manual runs).
4. Builds the image with [docker/build-push-action](https://github.com/docker/build-push-action).
5. Scans the image with [Grype](https://github.com/anchore/grype), through [anchore/scan-action](https://github.com/anchore/scan-action).
6. Uploads the scan results in SARIF format to the repository's code scanning alerts.

#### Pull requests vs. published images

- **Pull requests** only build the image. It is loaded into the runner's Docker daemon and scanned there, and nothing is pushed to the registry.
- **Any other event** (a tag push or a manual run, for example) builds the image, pushes it to the registry and scans the published image. Its scan results are reported against the default branch, so they show up in the repository's main Security view.

The scan never fails the build (`fail-build: false`) and only reports vulnerabilities that already have a fix (`only-fixed: true`). Findings are meant to be reviewed in the Security tab, not to block a release because of an unfixable issue in a base image.

#### Inputs

| Name | Required | Description |
|---|---|---|
| `container-registry` | yes | Registry host, e.g. `docker.io` or `ghcr.io` |
| `image-name` | yes | Image name without the registry, e.g. `myuser/myimage` |

#### Secrets

| Name | Required | Description |
|---|---|---|
| `registry-username` | yes | User to log into the registry |
| `registry-access-token` | yes | Access token (or password) for that user |

#### Permissions

The job requests `contents: read`, `packages: write` and `security-events: write`. A called workflow can't get more permissions than the caller allows, so if your repository restricts the default `GITHUB_TOKEN` permissions, grant them in the calling job (see the example below).

#### Usage

```yaml
name: Docker image build and push

on:
  workflow_dispatch:
  push:
    tags:
      - '*'
  pull_request:
    branches:
      - main

jobs:
  build-and-push:
    uses: mablanco/shared-github-workflows/.github/workflows/docker-build-push.yml@main
    # Only needed if the repository restricts the default GITHUB_TOKEN permissions
    permissions:
      contents: read
      packages: write
      security-events: write
    with:
      container-registry: docker.io
      image-name: myuser/myimage
    secrets:
      registry-username: ${{ secrets.DOCKERHUB_USERNAME }}
      registry-access-token: ${{ secrets.DOCKERHUB_TOKEN }}
```

Referencing `@main` always runs the latest version of the workflow. If you prefer to decide when to pick up changes, pin a tag or a commit SHA instead.

## Maintenance

[Dependabot](https://docs.github.com/en/code-security/dependabot) checks the actions used by the workflows every week, so updates are reviewed and merged once here instead of in every repository that uses them.

## License

[GPL-3.0](LICENSE)
