# ResolveOps AI — Reusable Workflow Templates

This repository provides **org-level reusable GitHub Actions workflows** for all services in the [Resolveops-AI](https://github.com/Resolveops-AI) organization.

## Architecture & Role

This repository is part of a 3-repo architecture designed for maintainability and separation of concerns:
1. **resolveops-application**: Contains the source code and Kubernetes manifests (`kustomization.yaml`, `deployment.yaml`, etc.). It invokes these templates in its `build-deploy.yml` pipeline.
2. **resolveops-templates** (this repo): Contains reusable CI/CD templates. It is kept intentionally simple, and does not hold any infrastructure configuration or application logic.
3. **resolveops-infrastructure**: Contains the Terraform code required to provision the underlying Azure infrastructure (AKS, ACR, Key Vault, etc.). Terraform workflows belong *only* in that repository.

---

## Available Templates

| Workflow | File | Purpose |
|---|---|---|
| **Build & Push** | `reusable-docker-build-push.yml` | Docker build and push to Azure Container Registry (ACR) via OIDC. |
| **Trivy Scan** | `reusable-trivy-scan.yml` | Trivy vulnerability scan on a container image in ACR. |
| **Helm Validate** | `reusable-helm-validate.yml` | Helm lint and `helm template` validation. |
| **Helm Deploy AKS** | `reusable-helm-deploy-aks.yml` | Deploys using Helm to an AKS cluster via OIDC. |
| **ArgoCD Sync** | `reusable-argocd-sync.yml` | Triggers an ArgoCD application sync and waits for rollout. |
| **Failure Notify** | `reusable-failure-notify.yml` | Writes a failure summary to the GitHub Actions job summary. No secrets required. |

---

## Required Secrets & Variables

### Secrets (Set at Org or Repo Level)
- `AZURE_CLIENT_ID`: Azure Client ID for OIDC authentication.
- `AZURE_TENANT_ID`: Azure Tenant ID for OIDC authentication.
- `AZURE_SUBSCRIPTION_ID`: Azure Subscription ID for OIDC authentication.
- `SONAR_TOKEN`: Token for SonarQube authentication.
- `SONAR_HOST_URL`: URL of the SonarQube server.
- `SNYK_TOKEN`: API token for Snyk vulnerability scanning.

### Variables
- `ACR_LOGIN_SERVER`: Azure Container Registry login server (e.g., `resolveopsai.azurecr.io`).
- `ACR_NAME`: Azure Container Registry name (e.g., `resolveopsai`).
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

For a full reference of all reusable workflows, inputs, and example caller patterns, see [docs/reusable-workflows.md](docs/reusable-workflows.md).
