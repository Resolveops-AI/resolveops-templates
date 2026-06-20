# Reusable Workflows Reference

This document is a reference for all reusable GitHub Actions workflows provided by `resolveops-templates`.

---

## Available Workflows

| Workflow File | Purpose |
|---|---|
| `reusable-docker-build-push.yml` | Build and push Docker images to Azure Container Registry |
| `reusable-trivy-scan.yml` | Trivy vulnerability scan on a container image |
| `reusable-helm-validate.yml` | Helm lint and template validation |
| `reusable-helm-deploy-aks.yml` | Deploy using Helm to an AKS cluster |
| `reusable-argocd-sync.yml` | Trigger an ArgoCD application sync |
| `reusable-failure-notify.yml` | Write a failure summary to the GitHub Actions job summary |

---

## `reusable-failure-notify.yml`

### Overview

This workflow writes a structured failure notification to the **GitHub Actions step summary**. When a job in a caller workflow fails, this template can be triggered in a follow-up job using `if: failure()`. GitHub natively surfaces this as a failed job in the Actions UI and can notify repository watchers automatically based on their notification preferences — **no external services or secrets are required**.

### Inputs

| Input | Required | Description |
|---|---|---|
| `workflow-name` | ✅ | Name of the workflow that failed |
| `environment-name` | ❌ | Target environment (e.g., `dev`, `prod`). Defaults to `N/A` |
| `repository-name` | ✅ | Full repo name (e.g., `Resolveops-AI/resolveops-application`) |
| `branch-name` | ✅ | Branch where the failure occurred |
| `commit-sha` | ✅ | The commit SHA that triggered the pipeline |
| `run-url` | ✅ | Direct URL to the failing GitHub Actions run |
| `triggered-by` | ✅ | GitHub actor (username) who triggered the workflow |

### Secrets Required

**None.** This workflow is completely secret-free.

### Example Caller Workflow

The following is an example of how any repository can call this reusable workflow on failure.

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
      image-name: "quickhaul-transits"
      acr-login-server: ${{ vars.ACR_LOGIN_SERVER }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy-dev:
    needs: build-push
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-helm-deploy-aks.yml@main
    with:
      environment: "dev"
      aks-cluster-name: ${{ vars.AKS_CLUSTER_NAME }}
      aks-resource-group: ${{ vars.AKS_RESOURCE_GROUP }}
      namespace: "quickhaul-dev"
      release-name: "quickhaul"
      helm-chart-path: "sample-apps/quickhaul/helm/quickhaul"
      image-name: "quickhaul-transits"
      acr-login-server: ${{ vars.ACR_LOGIN_SERVER }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  # ✅ Failure notification — called automatically if any upstream job fails
  notify-on-failure:
    needs: [build-push, deploy-dev]
    if: failure()
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-failure-notify.yml@main
    with:
      workflow-name: "QuickHaul CI/CD"
      environment-name: "dev"
      repository-name: ${{ github.repository }}
      branch-name: ${{ github.ref_name }}
      commit-sha: ${{ github.sha }}
      run-url: "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
      triggered-by: ${{ github.actor }}
```

> **Key pattern**: `if: failure()` on the `needs` list ensures the notification job only runs when any of the upstream jobs fail. It does not run on success.

---

## Enabling GitHub-Native Failure Notifications

GitHub natively emails or notifies you when a workflow fails — no extra setup is needed for the workflow itself. To configure your personal notification preferences:

1. Go to **GitHub → Settings → Notifications**.
2. Scroll to the **System** section.
3. Under **Actions**, select **"Only notify for failed workflows"** (or your preferred option).

This means every collaborator can control how they receive failure alerts through their own GitHub account settings.

> ℹ️ For organization-wide defaults, organization admins can configure notification settings under **Organization Settings → Notifications**.

---

## CI/CD Secrets vs Application Runtime Secrets

### CI/CD Secrets (belong in GitHub Secrets)
Only secrets required for the CI/CD pipeline itself should be stored as GitHub secrets:

| Secret | Purpose |
|---|---|
| `AZURE_CLIENT_ID` | OIDC auth to Azure for Docker/AKS operations |
| `AZURE_TENANT_ID` | OIDC auth to Azure |
| `AZURE_SUBSCRIPTION_ID` | OIDC auth to Azure |
| `ARGOCD_AUTH_TOKEN` | ArgoCD sync (only if using `reusable-argocd-sync.yml`) |

### Application Runtime Secrets (do NOT belong in GitHub Secrets)
The following are runtime secrets required by the deployed application. They must **never** be added to GitHub CI/CD workflows or templates:

- MongoDB connection strings
- Redis passwords
- JWT secrets
- SMTP credentials
- Payment API keys
- AWS credentials
- Any user-data or per-tenant secrets

These should be managed using **Azure Key Vault**, **Sealed Secrets**, or **External Secrets Operator**, and injected into Kubernetes at runtime — not at CI/CD time.
