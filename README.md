# ResolveOps AI — Reusable Workflow Templates

This repository provides **org-level reusable GitHub Actions workflows** for all services in the [Resolveops-AI](https://github.com/Resolveops-AI) organization.

## Architecture & Role

This repository is part of a 3-repo architecture designed for maintainability and separation of concerns:
1. **resolveops-application**: Contains the source code and Kubernetes manifests (`kustomization.yaml`, `deployment.yaml`, etc.). It invokes these templates in its `build-deploy.yml` pipeline.
2. **resolveops-templates** (this repo): Contains reusable CI/CD templates. It is kept intentionally simple, and does not hold any infrastructure configuration or application logic.
3. **resolveops-infrastructure**: Contains the Terraform code required to provision the underlying Azure infrastructure (AKS, ACR, Key Vault, etc.). Terraform workflows belong *only* in that repository.

---

## Available Templates

> **Note:** These reusable workflows do not normally run by themselves; they are designed to be called by application or infrastructure repositories using `workflow_call`.

| Workflow | File | Purpose |
|---|---|---|
| **Build & Push** | `reusable-docker-build-push.yml` | Standardized Docker build and push to Azure Container Registry (ACR) via OIDC. |
| **Helm Deploy to AKS** | `reusable-helm-deploy-aks.yml` | Deploys using Helm to an AKS cluster via OIDC. |
| **Plain K8s Deploy** | `reusable-kubernetes-deploy.yml` | Deploys plain Kubernetes manifests to AKS via OIDC. |
| **Helm Validate** | `reusable-helm-validate.yml` | Validates Helm charts using lint and template. |
| **Trivy Scan** | `reusable-trivy-scan.yml` | Vulnerability scanning for Docker images. |
| **ArgoCD Sync** | `reusable-argocd-sync.yml` | Triggers synchronization for ArgoCD applications. |
| **Notify** | `reusable-notify.yml` | Provides GitHub step summaries and optional webhook notifications. |

---

## Required Secrets & Variables

### Secrets (Set at Org or Repo Level)
- `AZURE_CLIENT_ID`: Azure Client ID for OIDC authentication.
- `AZURE_TENANT_ID`: Azure Tenant ID for OIDC authentication.
- `AZURE_SUBSCRIPTION_ID`: Azure Subscription ID for OIDC authentication.
- `SONAR_TOKEN`: Token for SonarCloud/SonarQube authentication (Note: SonarCloud uses `SONAR_TOKEN` and does not require `SONAR_HOST_URL`).
- `SNYK_TOKEN`: API token for Snyk vulnerability scanning (if applicable).

> **Note:** We no longer require legacy secrets such as `ACR_USERNAME`, `ACR_PASSWORD`, or `KUBECONFIG_DATA`.

### Variables
- `ACR_NAME`: Azure Container Registry name (e.g., `resolveopsai`).
- `ACR_LOGIN_SERVER`: Azure Container Registry login server (e.g., `resolveopsai.azurecr.io`).
- `AZURE_RESOURCE_GROUP`: Azure Resource Group for the AKS cluster.
- `AKS_CLUSTER_NAME`: Name of the AKS cluster.
- `AKS_NAMESPACE`: Default Kubernetes namespace for deployment.

---

## Why are these workflows simple?

- **Separation of Concerns**: Templates do not contain application-specific logic or infrastructure definitions.
- **Security First**: Security scans are blocking operations (`continue-on-error: false`, no `|| true` fallbacks). Docker pushes will only execute *after* scans have passed in the calling workflow.
- **OIDC Authentication**: We avoid long-lived credentials (`KUBECONFIG_DATA`) and utilize temporary, federated OIDC credentials for Azure operations.
- **Deployment Manifests**: Deploy workflows expect the application repository to provide the Kubernetes deployment manifests (via Kustomize).

For detailed instructions on how to use these templates in your application repo, refer to the [CICD_USAGE_GUIDE.md](CICD_USAGE_GUIDE.md).

## Plain Kubernetes Deploy Example

For deploying plain Kubernetes manifests (such as for the ResolveOps platform itself), you can use the `reusable-kubernetes-deploy.yml` template:

```yaml
deploy:
  needs: build
  uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-kubernetes-deploy.yml@main
  secrets: inherit
  with:
    aks_resource_group: ${{ vars.RESOLVEOPS_AKS_RESOURCE_GROUP }}
    aks_cluster_name: ${{ vars.RESOLVEOPS_AKS_NAME }}
    namespace: ${{ vars.RESOLVEOPS_NAMESPACE }}
    manifest_path: kubernetes
    deployment_name: frontend
    kubectl_timeout: 180s
```
