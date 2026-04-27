# idvilnius/gha-workflows

Reusable GitHub Actions workflows shared across all idvilnius service
repos. Caller workflows invoke them with
`uses: idvilnius/gha-workflows/.github/workflows/<file>@main`.

## Quick start — adding a new service

**Copy [`examples/build-and-deploy.yml`](examples/build-and-deploy.yml) into your repo as `.github/workflows/build-and-deploy.yml` — verbatim, no values to edit.**

That single file wires up the entire `service-pipeline.yaml` orchestrator
below. Then set five repo secrets (ask ops to paste them — values are
shared across all idvilnius services):

| Secret              | Used for                                      |
| ------------------- | --------------------------------------------- |
| `ACR_USERNAME`      | `cr0shared.azurecr.io` push (stg, prod)       |
| `ACR_PASSWORD`      | `cr0shared.azurecr.io` push (stg, prod)       |
| `ACR_TEST_USERNAME` | `cr0test.azurecr.io` push (dev, future test)  |
| `ACR_TEST_PASSWORD` | `cr0test.azurecr.io` push (dev, future test)  |
| `CHARTS_REPO_TOKEN` | PR write on `idvilnius/private-charts`        |

Push to `main`. On the very first run the pipeline auto-generates a
chart skeleton in `idvilnius/private-charts` and opens a `feat: bootstrap
<repo> chart` PR. Fill in the chart values, merge, and the rest is
GitOps.

The full developer walkthrough lives in [`idvilnius/private-charts/docs/developer-deployment-guide.md`](https://github.com/idvilnius/private-charts/blob/main/docs/developer-deployment-guide.md).

## Available workflows

| Workflow | Purpose |
| --- | --- |
| **`service-pipeline.yaml`** | **Universal orchestrator** — recommended entry point for every new idvilnius service. Combines the five workflows below into one call. |
| `image-version.yaml` | Derive an image tag from git SHA (with optional release tag prefix). |
| `build-env.yaml` | Lint, test, build a Docker image, upload as artifact. |
| `security.yaml` | Trivy + license scan against the built image. |
| `publish-acr.yaml` | Pull the build artifact and push the image to Azure Container Registry. |
| `bump-chart-tag.yaml` | Open (or auto-merge) a PR in a separate Helm chart repo that updates a single image tag field. |

See individual files for the full input/secret schema.

## How `service-pipeline.yaml` routes events

| Event                            | Pushes image to       | Bumps values file                     |
| -------------------------------- | --------------------- | ------------------------------------- |
| push to `dev` branch             | `cr0test.azurecr.io`  | `charts/<repo>/envs/dev-values.yaml`  |
| push to `main` branch            | `cr0shared.azurecr.io`| `charts/<repo>/envs/stg-values.yaml`  |
| GitHub `release` (publish a tag) | `cr0shared.azurecr.io`| `charts/<repo>/envs/prod-values.yaml` |
| `pull_request`                   | (no push)             | (no bump — builds + scans only)       |

The chart's env values files hardcode the matching `image.repository`
(env `dev` and `test` → `cr0test.azurecr.io/<repo>`; env `stg` and
`prod` → `cr0shared.azurecr.io/<repo>`), so the same image tag exists
in different registries depending on which branch built it.

On the **first push to `main`** when `charts/<repo>/` does not yet exist
in the chart repo, the pipeline copies `charts/.template/` from
`idvilnius/private-charts` into a new `charts/<repo>/`, generates three
ArgoCD Applications (dev / stg / prod), and opens a single bootstrap PR.
Subsequent pushes follow the routing table above.

## Calling each workflow individually (legacy / advanced)

Most services should _not_ need this — `service-pipeline.yaml` already
wires the pieces together. Drop down to direct calls only if you need
non-standard sequencing (e.g. two ACR registries based on branch, like
`mokauvilniuje`).

```yaml
jobs:
  GenerateImageVersion:
    uses: idvilnius/gha-workflows/.github/workflows/image-version.yaml@main

  Build:
    needs: [GenerateImageVersion]
    uses: idvilnius/gha-workflows/.github/workflows/build-env.yaml@main
    with:
      docker_image: cr0shared.azurecr.io/<your-repo>
      image_tag: ${{ needs.GenerateImageVersion.outputs.image_version }}
      lint_step: |
        echo "No linter yet"
      test_step: |
        echo "No tests yet"
      build_step: |
        docker build -t cr0shared.azurecr.io/<your-repo>:${{ needs.GenerateImageVersion.outputs.image_version }} .

  Security:
    needs: [GenerateImageVersion, Build]
    uses: idvilnius/gha-workflows/.github/workflows/security.yaml@main
    with:
      docker_image: cr0shared.azurecr.io/<your-repo>
      image_tag: ${{ needs.GenerateImageVersion.outputs.image_version }}

  Publish:
    needs: [GenerateImageVersion, Security]
    if: ${{ github.ref == 'refs/heads/main' }}
    uses: idvilnius/gha-workflows/.github/workflows/publish-acr.yaml@main
    with:
      docker_image: cr0shared.azurecr.io/<your-repo>
      image_tag: ${{ needs.GenerateImageVersion.outputs.image_version }}
    secrets:
      acr_username: ${{ secrets.ACR_USERNAME }}
      acr_password: ${{ secrets.ACR_PASSWORD }}

  BumpChartTag:
    needs: [GenerateImageVersion, Publish]
    if: ${{ github.ref == 'refs/heads/main' }}
    uses: idvilnius/gha-workflows/.github/workflows/bump-chart-tag.yaml@main
    with:
      chart_repo: idvilnius/private-charts
      chart_file: charts/<your-repo>/envs/stg-values.yaml
      yaml_path: app.image.tag
      new_tag: ${{ needs.GenerateImageVersion.outputs.image_version }}
      branch_prefix: bump-<your-repo>-stg
      auto_merge: false
    secrets:
      gh_token: ${{ secrets.CHARTS_REPO_TOKEN }}
```

`CHARTS_REPO_TOKEN` must be a PAT (or fine-grained token) with
`contents: write` and `pull_requests: write` on the chart repo.
