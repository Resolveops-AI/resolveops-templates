# CI/CD Usage Guide

This guide explains how to use the reusable workflows in `resolveops-templates` from caller repositories.

## Final CI/CD Flow & Branch Behavior

The templates repo should not control branch triggers directly except through `workflow_call`. Branch behavior must be controlled by `resolveops-application` caller workflows.

### 1. Push to `dev` branch
* Application repo may run lightweight validation only.
* No Docker image build is required from templates on dev push unless caller explicitly calls it.
* Templates are generic and do not force image creation on dev push.
* No ACR push required.
* No Helm values update required.

### 2. Pull request from `dev` to `main`
* Call `ci-reusable-template.yml` for CI/security scanning.
* Call `docker-build-push-template.yml` to build dev images.
* Image tag format should support: `dev-pr-<PR_NUMBER>-<short-sha>`
* Call `helm-updater-template.yml` to update `values-dev.yaml`.
* Optional `cd-reusable-template.yml` only if Argo CD is reachable.

### 3. Merge/push to `main`
* Call `ci-reusable-template.yml` again.
* Calculate semantic version for production image tag.
* Production job should use GitHub Environment manual approval in the caller repo.
* Call `docker-build-push-template.yml` to build/push or retag production release semantic image.
* Semantic version format: `vMAJOR.MINOR.PATCH`
* Call `helm-updater-template.yml` to update `values-prod.yaml`.
* Optional `cd-reusable-template.yml` only if Argo CD is reachable.

## Required Variables and Secrets

### Required Caller Repository Variables
Configure these variables in your caller repositories (e.g., `resolveops-application`):
* `ACR_NAME`: Azure Container Registry name
* `ACR_LOGIN_SERVER`: Azure Container Registry login server URL
* `AZURE_RESOURCE_GROUP`: Azure Resource Group name
* `AKS_CLUSTER_NAME`: Name of the AKS cluster
* `RESOLVEOPS_NAMESPACE`: Target Kubernetes namespace for ResolveOps
* `QUICKHAUL_DEV_NAMESPACE`: Target Kubernetes namespace for QuickHaul Dev
* `QUICKHAUL_PROD_NAMESPACE`: Target Kubernetes namespace for QuickHaul Prod
* `ARGOCD_SERVER` (optional): Argo CD server URL

### Required Secrets
Configure these secrets in your caller repositories:
* `AZURE_CLIENT_ID`: Azure AD Client ID for OIDC
* `AZURE_TENANT_ID`: Azure AD Tenant ID for OIDC
* `AZURE_SUBSCRIPTION_ID`: Azure Subscription ID for OIDC
* `SONAR_TOKEN`: SonarQube token for code analysis
* `SONAR_HOST_URL`: SonarQube server URL
* `SNYK_TOKEN`: Snyk API token for vulnerability scanning
* `ARGOCD_AUTH_TOKEN` (optional): Token for optional Argo CD manual sync

## Versioning Rules & Semantic Versioning Expectation

* Semantic version is calculated in the caller repo.
* Templates only accept the final `image_tag` input.
* Templates should not decide version bump logic.
* Caller may use Conventional Commits:
  * `BREAKING CHANGE` or `!` = major
  * `feat` = minor
  * `fix` = patch
  * `chore`/`refactor`/`docs`/`test` = patch for this project

## Usage Examples

### 1. CI Validation Example (PR)

```yaml
name: CI Validation
on:
  pull_request:
    branches: [ "main" ]

jobs:
  security-scan:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/ci-reusable-template.yml@main
    with:
      working_directory: ./src
      run_sonar: true
      run_snyk: true
      run_trivy: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  build-push:
    needs: security-scan
    uses: Resolveops-AI/resolveops-templates/.github/workflows/docker-build-push-template.yml@main
    with:
      image_name: app
      dockerfile_path: ./src/Dockerfile
      context_path: ./src
      acr_name: ${{ vars.ACR_NAME }}
      acr_login_server: ${{ vars.ACR_LOGIN_SERVER }}
      image_tag: dev-pr-${{ github.event.pull_request.number }}-${{ github.sha }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  helm-update:
    needs: build-push
    uses: Resolveops-AI/resolveops-templates/.github/workflows/helm-updater-template.yml@main
    with:
      values_file: ./helm/values-dev.yaml
      image_tag_key: image.tag
      image_tag: dev-pr-${{ github.event.pull_request.number }}-${{ github.sha }}
      branch_name: main
      commit_message: "chore: update dev image to dev-pr-${{ github.event.pull_request.number }}-${{ github.sha }} [skip ci]"

  notify:
    needs: helm-update
    if: ${{ always() && needs.helm-update.result == 'failure' }}
    uses: Resolveops-AI/resolveops-templates/.github/workflows/notify-template.yml@main
    with:
      service_name: app
      environment: dev
      failed_job_name: helm-update
      status: ${{ needs.helm-update.result }}
      run_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
      branch_name: ${{ github.ref_name }}
      commit_sha: ${{ github.sha }}
      actor: ${{ github.actor }}
      workflow_name: ${{ github.workflow }}
    secrets:
      SMTP_HOST: ${{ secrets.SMTP_HOST }}
      SMTP_PORT: ${{ secrets.SMTP_PORT }}
      SMTP_USERNAME: ${{ secrets.SMTP_USERNAME }}
      SMTP_PASSWORD: ${{ secrets.SMTP_PASSWORD }}
      EMAIL_FROM: ${{ secrets.EMAIL_FROM }}
      EMAIL_TO: ${{ secrets.EMAIL_TO }}
```

Note: `cd-reusable-template.yml` should be called only if Argo CD is reachable from GitHub-hosted runners. Otherwise, let Argo CD automatically sync from the updated Helm values.
