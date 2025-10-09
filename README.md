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
