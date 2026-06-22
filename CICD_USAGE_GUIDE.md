# CI/CD Usage Guide

This guide explains how to use the reusable workflows in `resolveops-templates` from caller repositories.

## Final CI/CD Flow

The standard FitForge-style CI/CD flow should follow these steps in order:

1. **Security Scan**
2. **Docker Build/Push**
3. **Helm Update** (commits new image tags to the repository)
4. **Argo CD Auto-sync / Optional Manual Sync**
5. **Notify**

## Required Variables and Secrets

### Required Caller Repository Variables
Configure these variables in your caller repositories (e.g., `resolveops-application`):
* `ACR_NAME`: Azure Container Registry name (e.g., `resolveopsacr03`)
* `ACR_LOGIN_SERVER`: Azure Container Registry login server URL (e.g., `resolveopsacr03.azurecr.io`)
* `AKS_CLUSTER_NAME`: Name of the AKS cluster
* `AZURE_RESOURCE_GROUP`: Azure Resource Group name
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

## Versioning Rules

To ensure reliable GitOps deployments:
* **Dev image tags** should use `dev-<short-sha>`, example `dev-a1b2c3d`.
* **Production image tags** should use semantic versions, example `v1.0.0`.
* **Do not use** only `dev` or `prod` as the main GitOps image tag because those are mutable.
* `dev`/`prod` tags can be optional aliases, but Helm values **must** use immutable tags.

## Usage Examples

### 1. ResolveOps CI Example

```yaml
name: ResolveOps CI
on:
  push:
    branches: [ "main" ]

jobs:
  security-scan:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-security-scan.yml@main
    with:
      working_directory: ./resolveops
      run_sonar: true
      run_snyk: true
      run_trivy: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  build-push:
    needs: security-scan
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      image_name: resolveops-api
      dockerfile_path: ./resolveops/Dockerfile
      context_path: ./resolveops
      acr_name: ${{ vars.ACR_NAME }}
      acr_login_server: ${{ vars.ACR_LOGIN_SERVER }}
      image_tag: dev-${{ github.sha }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  helm-update:
    needs: build-push
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-helm-update.yml@main
    with:
      values_file: ./helm/resolveops/values.yaml
      image_repository_key: image.repository
      image_tag_key: image.tag
      image_repository: ${{ vars.ACR_LOGIN_SERVER }}/resolveops-api
      image_tag: dev-${{ github.sha }}
      commit_message: "chore: update resolveops-api image to dev-${{ github.sha }} [skip ci]"

  notify:
    needs: helm-update
    if: always()
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-notify.yml@main
    with:
      service_name: resolveops-api
      environment: dev
      status: ${{ job.status }}
      run_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

### 2. QuickHaul CI Example

```yaml
name: QuickHaul CI
on:
  push:
    branches: [ "main" ]

jobs:
  security-scan:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-security-scan.yml@main
    with:
      working_directory: ./quickhaul
      run_sonar: true
      run_snyk: true
      run_trivy: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  build-push:
    needs: security-scan
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      image_name: quickhaul-app
      dockerfile_path: ./quickhaul/Dockerfile
      context_path: ./quickhaul
      acr_name: ${{ vars.ACR_NAME }}
      acr_login_server: ${{ vars.ACR_LOGIN_SERVER }}
      image_tag: dev-${{ github.sha }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  helm-update:
    needs: build-push
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-helm-update.yml@main
    with:
      values_file: ./helm/quickhaul/values-dev.yaml
      image_tag_key: image.tag
      image_tag: dev-${{ github.sha }}
      commit_message: "chore: update quickhaul-app dev image to dev-${{ github.sha }} [skip ci]"

  notify:
    needs: helm-update
    if: always()
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-notify.yml@main
    with:
      service_name: quickhaul-app
      environment: dev
      status: ${{ job.status }}
      run_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

### 3. QuickHaul Production Promotion Example

```yaml
name: QuickHaul Production Promotion
on:
  release:
    types: [published]

jobs:
  build-push-prod:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      image_name: quickhaul-app
      dockerfile_path: ./quickhaul/Dockerfile
      context_path: ./quickhaul
      acr_name: ${{ vars.ACR_NAME }}
      acr_login_server: ${{ vars.ACR_LOGIN_SERVER }}
      image_tag: ${{ github.event.release.tag_name }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  helm-update-prod:
    needs: build-push-prod
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-helm-update.yml@main
    with:
      values_file: ./helm/quickhaul/values-prod.yaml
      image_tag_key: image.tag
      image_tag: ${{ github.event.release.tag_name }}
      commit_message: "chore: update quickhaul-app prod image to ${{ github.event.release.tag_name }} [skip ci]"

  argocd-sync:
    needs: helm-update-prod
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-argocd-sync.yml@main
    with:
      argocd_app_name: quickhaul-prod
      argocd_server: ${{ vars.ARGOCD_SERVER }}
    secrets:
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}

  notify:
    needs: argocd-sync
    if: always()
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-notify.yml@main
    with:
      service_name: quickhaul-app
      environment: prod
      status: ${{ job.status }}
      run_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

Note: If Argo CD is private inside AKS, GitHub-hosted runners may not reach it, making manual sync from the pipeline impossible. In that case, preferred deployment method is Argo CD auto-sync triggered by the Helm values update.
