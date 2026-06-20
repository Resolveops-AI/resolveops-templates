# CI/CD Usage Guide

This guide explains how to use the reusable templates from `resolveops-templates` in the `resolveops-application` repository.

## Overview

The `resolveops-templates` repository contains reusable GitHub Actions workflows designed to standardize and simplify CI/CD pipelines across the organization. By centralizing the pipeline logic, we ensure:
- Consistent security scanning across all apps.
- Enforced use of OIDC for Azure deployments (no more `KUBECONFIG_DATA` or long-lived secrets).
- Clear separation between application code, CI/CD logic, and Infrastructure as Code.
- Generic workflows that can be called for any application, like QuickHaul or ResolveOps Platform.

## QuickHaul Caller Workflow Example

Below is a complete example of how `resolveops-application` might call these templates to build, scan, and deploy the QuickHaul service.

```yaml
name: QuickHaul CI/CD

on:
  push:
    branches:
      - main
    paths:
      - 'sample-apps/quickhaul/**'

jobs:
  build-push:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      working-directory: "sample-apps/quickhaul"
      image-name: "quickhaul-transits"
      dockerfile-path: "Dockerfile"
      context-path: "."
      acr-login-server: ${{ vars.ACR_LOGIN_SERVER }}
      environment: "dev"
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  trivy-scan:
    needs: build-push
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-trivy-scan.yml@main
    with:
      image-name: "quickhaul-transits"
      acr-login-server: ${{ vars.ACR_LOGIN_SERVER }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  helm-validate:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-helm-validate.yml@main
    with:
      working-directory: "sample-apps/quickhaul/helm"
      helm-chart-path: "quickhaul"
      helm-values-file: "values.yaml"

  deploy-dev:
    needs: [trivy-scan, helm-validate]
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-helm-deploy-aks.yml@main
    with:
      environment: "dev"
      aks-resource-group: ${{ vars.AKS_RESOURCE_GROUP }}
      aks-cluster-name: ${{ vars.AKS_CLUSTER_NAME }}
      namespace: "quickhaul-dev"
      release-name: "quickhaul"
      helm-chart-path: "sample-apps/quickhaul/helm/quickhaul"
      helm-values-file: "values-dev.yaml"
      image-name: "quickhaul-transits"
      acr-login-server: ${{ vars.ACR_LOGIN_SERVER }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## How to Deploy to Different Environments

### `quickhaul-dev`
- **Cluster**: `quickhaul-aks`
- **Namespace**: `quickhaul-dev`
- **Action**: Pass `environment: "dev"` and `namespace: "quickhaul-dev"` to the `reusable-helm-deploy-aks.yml` workflow.

### `quickhaul-prod`
- **Cluster**: `quickhaul-aks`
- **Namespace**: `quickhaul-prod`
- **Action**: Pass `environment: "prod"` and `namespace: "quickhaul-prod"` to the `reusable-helm-deploy-aks.yml` workflow. Usually triggered on tags or main branch.

### `resolveops`
- **Cluster**: `resolveops-aks`
- **Namespace**: `resolveops`
- **Action**: Pass `environment: "prod"`, `namespace: "resolveops"` and appropriate cluster variables to deploy the ResolveOps platform itself.

## Secrets and Variables

### Required Secrets
The following secrets MUST be added to your GitHub repository or organization secrets for OIDC authentication:
- `AZURE_CLIENT_ID`: The client ID of your Azure Managed Identity / App Registration.
- `AZURE_TENANT_ID`: Your Azure AD Tenant ID.
- `AZURE_SUBSCRIPTION_ID`: Your Azure Subscription ID.
- `ARGOCD_AUTH_TOKEN`: (Optional) Only required if using the `reusable-argocd-sync.yml` workflow.

### Required Variables
- `ACR_LOGIN_SERVER`: Example: `myacr.azurecr.io`
- `AKS_RESOURCE_GROUP`: Azure resource group containing the AKS cluster.
- `AKS_CLUSTER_NAME`: Name of the AKS cluster.

### CI/CD Secrets vs Application Runtime Secrets

**CI/CD Secrets** are strictly for allowing GitHub Actions to authenticate with infrastructure components (like Azure, ACR, AKS, and ArgoCD). They belong in GitHub Settings > Secrets. Examples include:
- `AZURE_CLIENT_ID`
- `ARGOCD_AUTH_TOKEN`
- `SONAR_TOKEN` (if used)

**Application Runtime Secrets** are required by your deployed application to function (like MongoDB connection strings, Redis passwords, JWT secrets, SMTP credentials, API keys).
**CRITICAL**: Do NOT store runtime secrets in GitHub Actions. They should be managed externally (e.g., Azure Key Vault, HashiCorp Vault, or Sealed Secrets) and injected directly into the Kubernetes cluster or retrieved by the application at runtime. CI/CD templates should remain generic and unaware of application-specific runtime secrets.
