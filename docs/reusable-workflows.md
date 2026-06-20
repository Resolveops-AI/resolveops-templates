# Reusable Workflows Reference

This document is a reference for all reusable GitHub Actions workflows provided by `resolveops-templates`.

---

## Trigger Pattern — All Caller Workflows

All caller workflows in the `resolveops-application` repository **must** use the following trigger pattern:

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

**Do not use `pull_request` triggers.** Pull requests generate an OIDC subject like:
`repo:Resolveops-AI/resolveops-application:pull_request`

This does NOT match the branch-based federated credential and will cause Azure OIDC login failures.

---

## Azure OIDC Federated Credential Configuration

These reusable workflows authenticate to Azure using **OIDC with branch-based federated credentials only**. No GitHub Environments are used at the job level, which keeps the OIDC subject stable and predictable.

### Expected OIDC Subjects

| Repository | Expected Subject |
|---|---|
| `resolveops-infrastructure` | `repo:Resolveops-AI/resolveops-infrastructure:ref:refs/heads/main` |
| `resolveops-application` | `repo:Resolveops-AI/resolveops-application:ref:refs/heads/main` |

### What NOT to configure

- ❌ Do **not** configure environment-scoped federated credentials (e.g., subject: `environment:dev`)
- ❌ Do **not** configure pull_request federated credentials
- ✅ Use only branch-based subjects matching the table above

> **Why**: If a reusable workflow job has `environment: <name>` at the job level, the OIDC subject automatically becomes `repo:<owner>/<repo>:environment:<name>`, which won't match a branch-based federated credential. All `environment:` job-level keys have been removed from these reusable templates to prevent this issue.

---

## Architecture — ResolveOps vs QuickHaul

| Component | Cluster | Namespace(s) | Environments |
|---|---|---|---|
| ResolveOps AI Platform | `resolveops-aks` | `resolveops` | None — single platform |
| QuickHaul Transits | `quickhaul-aks` | `quickhaul-dev`, `quickhaul-prod` | dev + prod |

**ResolveOps has no dev/prod environments.** It runs once as a platform. Do not pass `environment: dev` or `environment: prod` when calling workflows for ResolveOps. The `environment` input on these workflows is a **display label only** — it does not gate deployments.

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
| `environment-name` | ❌ | Display label only. Defaults to `N/A` |
| `repository-name` | ✅ | Full repo name (e.g., `Resolveops-AI/resolveops-application`) |
| `branch-name` | ✅ | Branch where the failure occurred |
| `commit-sha` | ✅ | The commit SHA that triggered the pipeline |
| `run-url` | ✅ | Direct URL to the failing GitHub Actions run |
| `triggered-by` | ✅ | GitHub actor (username) who triggered the workflow |

### Secrets Required

**None.** This workflow is completely secret-free.

### Example: QuickHaul Caller Workflow

```yaml
name: QuickHaul CI/CD

on:
  push:
    branches:
      - main
    paths:
      - 'sample-apps/quickhaul/**'
  workflow_dispatch:

jobs:
  build-push:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      working-directory: "sample-apps/quickhaul"
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
      environment: "dev"          # Display label only — not a GitHub Environment gate
      aks-cluster-name: ${{ vars.QUICKHAUL_AKS_CLUSTER_NAME }}
      aks-resource-group: ${{ vars.QUICKHAUL_AKS_RESOURCE_GROUP }}
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

### Example: ResolveOps Platform Caller Workflow

ResolveOps has **no dev/prod environments**. It deploys once to `resolveops-aks` in the `resolveops` namespace.

```yaml
name: ResolveOps Platform Deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build-push:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      image-name: "resolveops-platform"
      acr-login-server: ${{ vars.ACR_LOGIN_SERVER }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy:
    needs: build-push
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-helm-deploy-aks.yml@main
    with:
      # No environment: input — ResolveOps is a single platform, no dev/prod split
      aks-cluster-name: ${{ vars.RESOLVEOPS_AKS_CLUSTER_NAME }}
      aks-resource-group: ${{ vars.RESOLVEOPS_AKS_RESOURCE_GROUP }}
      namespace: "resolveops"
      release-name: "resolveops"
      helm-chart-path: "helm/resolveops"
      image-name: "resolveops-platform"
      acr-login-server: ${{ vars.ACR_LOGIN_SERVER }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  notify-on-failure:
    needs: [build-push, deploy]
    if: failure()
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-failure-notify.yml@main
    with:
      workflow-name: "ResolveOps Platform Deploy"
      repository-name: ${{ github.repository }}
      branch-name: ${{ github.ref_name }}
      commit-sha: ${{ github.sha }}
      run-url: "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
      triggered-by: ${{ github.actor }}
```

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
