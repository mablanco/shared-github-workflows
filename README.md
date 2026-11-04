# shared-github-workflows

Reusable GitHub Actions workflows for CI/CD pipelines, designed to streamline automation across multiple projects and promote consistency.

Each workflow lives in `.github/workflows/` and is triggered with `workflow_call`, so other repositories can call it with a few lines of YAML instead of copying the whole pipeline.

Referencing a workflow `@main` always runs its latest version. If you prefer to decide when to pick up changes, pin a tag or a commit SHA instead.

## Workflows

### `docker-build-push.yml`

Builds a Docker image from the repository root, scans it for vulnerabilities and publishes it to a container registry.

It runs these steps:

1. Checks out the repository.
2. Logs into the container registry.
3. Generates the image tags and labels with [docker/metadata-action](https://github.com/docker/metadata-action), using its default rules (the Git tag on tag pushes, `pr-<number>` on pull requests, the branch name on manual runs).
4. Sets up a BuildKit builder with [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action), which the layer cache needs.
5. Builds the image with [docker/build-push-action](https://github.com/docker/build-push-action), reusing and saving layers in the GitHub Actions cache (`type=gha`, `mode=max`).
6. Scans the image with [Grype](https://github.com/anchore/grype), through [anchore/scan-action](https://github.com/anchore/scan-action).
7. Uploads the scan results in SARIF format to the repository's code scanning alerts.

#### Pull requests vs. published images

- **Pull requests** only build the image. It is loaded into the runner's Docker daemon and scanned there, and nothing is pushed to the registry.
- **Pull requests from forks** get no secrets and a read-only token from GitHub, so they also skip the registry login and the upload of the scan results. The scan report is still printed in the job log.
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

### `ansible-role-ci.yml`

Lints an Ansible role and tests it with [Molecule](https://ansible.readthedocs.io/projects/molecule/).

It runs two jobs:

1. **lint**: installs the role's `requirements-dev.txt` and runs [ansible-lint](https://ansible.readthedocs.io/projects/lint/), once.
2. **molecule**: runs `molecule test` once for each entry of `molecule-matrix`, in parallel. A failing run doesn't cancel the others.

The tools come from the role's `requirements-dev.txt`, so each role pins its own versions of ansible-core, ansible-lint and Molecule, and uses the same ones locally. ansible-lint and Molecule read their usual configuration from the role (`.ansible-lint`, `.yamllint`, `molecule/`).

#### Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `python-version` | no | `3.14` | Python version for the tools |
| `molecule-matrix` | no | `[{"name": "default"}]` | JSON list of Molecule runs (see below) |

Each entry of `molecule-matrix` has:

- `name`: shown in the job name, `Molecule (<name>)`.
- `scenario` (optional): the Molecule scenario to run. Defaults to `default`.
- `env` (optional): environment variables for the run, as `KEY=value` lines, for roles whose scenario reads them.

#### Usage

A role with only the `default` scenario:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  # Weekly, to catch changes in the distributions' packages
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:

jobs:
  ci:
    uses: mablanco/shared-github-workflows/.github/workflows/ansible-role-ci.yml@main
```

A role with several scenarios, or one scenario run with different variables:

```yaml
jobs:
  ci:
    uses: mablanco/shared-github-workflows/.github/workflows/ansible-role-ci.yml@main
    with:
      molecule-matrix: >-
        [{"name": "default"},
         {"name": "upgrade", "scenario": "upgrade"},
         {"name": "git", "env": "DEPLOY_METHOD=git"}]
```

## Maintenance

[Dependabot](https://docs.github.com/en/code-security/dependabot) checks the actions used by the workflows every week, so updates are reviewed and merged once here instead of in every repository that uses them.

## License

[GPL-3.0](LICENSE)
