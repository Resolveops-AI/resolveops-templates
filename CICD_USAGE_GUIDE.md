# CI/CD Usage Guide

This guide explains how to use the reusable workflows provided in the `resolveops-templates` repository for your application CI/CD pipelines following the FitForge GitOps architecture.

## Overview

The CI/CD process is split into several distinct phases:
1. **PR Validation**: Validates code and docker images on Pull Requests to `main`.
2. **Dev Deployment**: Builds and pushes Docker image to ACR (with a `dev-<sha>` tag) on pushes to `main`, and optionally updates the dev environment GitOps manifests.
3. **Prod Release Deployment**: Builds and pushes Docker image with a release tag (e.g. `v1.0.0`), updates the prod GitOps manifests, and optionally triggers Argo CD.
4. **GitOps Update**: Updates the Git repository where Kubernetes manifests (Helm or Kustomize) are stored so that Argo CD can detect the change and apply it.
5. **Argo CD Sync**: Optional manual trigger for Argo CD to sync immediately. Auto-sync should normally handle this.

## Required Secrets

Your caller repository (or organization) must provide the following secrets:
- `SONAR_TOKEN`: For SonarCloud/SonarQube scanning.
- `SNYK_TOKEN`: For Snyk SCA scanning.
- `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`: For Azure OIDC login to push to ACR.
- `WEBHOOK_URL` (optional): For Teams/Slack/Email webhook notifications.
- `ARGOCD_AUTH_TOKEN` (optional): For manual Argo CD sync.

## Example: Microservice Caller Workflow

Create a file in your application repository at `.github/workflows/ci-cd.yml`:

```yaml
name: Microservice CI/CD

on:
  push:
    branches:
      - main
    tags:
      - 'v*.*.*'
  pull_request:
    branches:
      - main

permissions:
  id-token: write
  contents: write

jobs:
  # 1. PR Validation
  pr-validation:
    if: github.event_name == 'pull_request'
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-pr-validation.yml@main
    with:
      dockerfile_path: 'Dockerfile'
      context_path: '.'
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # 2. Build and Push (Dev or Prod)
  build-push:
    if: github.event_name == 'push'
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      image_name: 'my-microservice'
      # Tag logic: use short sha for main branch, or the actual tag if a tag was pushed
      image_tag: ${{ startsWith(github.ref, 'refs/tags/') && github.ref_name || format('dev-{0}', github.sha) }}
      acr_name: 'myacr'
      acr_login_server: 'myacr.azurecr.io'
      push_latest: false
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  # 3. GitOps Update
  gitops-update:
    needs: build-push
    if: github.event_name == 'push'
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-gitops-update.yml@main
    with:
      manifest_type: 'helm'
      values_file: 'helm/my-microservice/values.yaml'
      image_tag_key: '.image.tag'
      image_name: 'my-microservice'
      image_tag: ${{ startsWith(github.ref, 'refs/tags/') && github.ref_name || format('dev-{0}', github.sha) }}
      target_branch: 'main'

  # 4. Argo CD Sync (Optional)
  argocd-sync:
    needs: gitops-update
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-argocd-sync.yml@main
    with:
      app_name: 'my-microservice-prod'
      argocd_server: 'argocd.example.com'
    secrets:
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}

  # 5. Notifications
  notify:
    needs: [pr-validation, build-push, gitops-update, argocd-sync]
    if: always()
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-notify.yml@main
    with:
      status: ${{ job.status }}
      message: "Workflow completed for my-microservice"
    secrets:
      WEBHOOK_URL: ${{ secrets.WEBHOOK_URL }}
```
