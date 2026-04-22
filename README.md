# gha-workflows

Reusable GitHub Actions workflows shared across idvilnius service repos.
All workflows live under `.github/workflows/` and are invoked from caller
workflows via `uses: idvilnius/gha-workflows/.github/workflows/<file>@main`.

| Workflow | Purpose |
| --- | --- |
| `image-version.yaml` | Derive an image version from git sha (+ optional tag). |
| `build-env.yaml` | Lint, test, build a Docker image, upload as artifact. |
| `publish-acr.yaml` | Pull the artifact and push the image to Azure Container Registry. |
| `bump-chart-tag.yaml` | Open (or auto-merge) a PR in a separate Helm chart repo that updates a single image tag field. |

See individual files for the full input/secret schema. A minimal caller
workflow looks like:

```yaml
name: Build and push workflow

on:
  push:
    branches:
      - main
  pull_request: {}

env:
  AZURE_CONTAINER_REGISTRY: cr0shared.azurecr.io/samples/hello-world
  ACR_IMAGE: ${{ env.AZURE_CONTAINER_REGISTRY }}

jobs:
  GenerateImageVersion:
    name: ID Vilnius Generate Image Version
    uses: idvilnius/gha-workflows/.github/workflows/image-version.yaml@main

  Build:
    name: ID Vilnius Build image
    needs: [GenerateImageVersion]
    uses: idvilnius/gha-workflows/.github/workflows/build-env.yaml@main
    with:
      docker_image: ${{ env.ACR_IMAGE }}
      image_tag: ${{ needs.GenerateImageVersion.outputs.image_version }}
      lint_step: |
        echo "No linter yet"
      test_step: |
        echo "No tests yet"
      build_step: |
        docker build -t "${{ env.ACR_IMAGE }}:${{ needs.GenerateImageVersion.outputs.image_version }}" .
    secrets: inherit

  Security:
    name: 🔐 ID Vilnius Security Scan
    needs: [GenerateImageVersion, Build]
    uses: idvilnius/.github/workflows/security.yaml@main
    with:
      docker_image: ${{ env.ACR_IMAGE }}
      image_tag: ${{ needs.GenerateImageVersion.outputs.image_version }}

  PublishDev:
    name: 📦 ID Vilnius Publish to ACR
    needs: [GenerateImageVersion, Security]
    if: ${{ github.ref == 'refs/heads/staging-sview-infra' }}
    uses: idvilnius/gha-workflows/.github/workflows/publish-acr.yaml@main
    with:
      docker_image: ${{ env.ACR_IMAGE }}
      image_tag: ${{ needs.GenerateImageVersion.outputs.image_version }}
    secrets:
      acr_username: ${{ secrets.ACR_USERNAME }}
      acr_password: ${{ secrets.ACR_PASSWORD }}
```

## Bumping a Helm chart after publish

Once the image is in ACR, `bump-chart-tag.yaml` opens a PR in the chart repo
that points the deployment at the new tag. It takes a dot-path to the YAML
field to edit, so the same workflow works for `app.image.tag` in wrapper
charts or `image.tag` in legacy ones.

```yaml
  BumpChartTag:
    name: 🔖 Bump chart tag
    needs: [GenerateImageVersion, PublishProd]
    uses: idvilnius/gha-workflows/.github/workflows/bump-chart-tag.yaml@main
    with:
      chart_repo:    idvilnius/private-charts
      chart_file:    charts/schools-internal/values.yaml
      yaml_path:     app.image.tag
      new_tag:       ${{ needs.GenerateImageVersion.outputs.image_version }}
      branch_prefix: bump-schools-internal
      auto_merge:    true
    secrets:
      gh_token: ${{ secrets.CHARTS_REPO_TOKEN }}
```

`CHARTS_REPO_TOKEN` must be a PAT (or fine-grained token) with `contents: write`
and `pull_requests: write` on the chart repo. Leave `auto_merge: false` while
bedding in the flow, then flip it on once you trust it.
