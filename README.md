# ResolveOps AI — Reusable Workflow Templates

This repository provides **org-level reusable GitHub Actions workflows** for all services
in the [Resolveops-AI](https://github.com/Resolveops-AI) organization.

Modeled after the [QuickHaulTransits/quickhaul-templates](https://github.com/QuickHaulTransits) pattern.

---

## Available Templates

| Workflow | File | Purpose |
|---|---|---|
| Build, Scan & Push | `reusable-build-scan-push.yml` | Docker build + ACR push |
| Deploy to AKS | `reusable-deploy.yml` | Rolling deploy via kubectl |
| Email Notify | `reusable-email-notify.yml` | Pipeline status notification |

---

## Usage

In any service repo's `.github/workflows/ci.yml`:

```yaml
jobs:
  build-and-push:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-build-scan-push.yml@main
    with:
      service_name: my-service
      docker_context: .
      image_tag: ${{ github.sha }}
    secrets:
      ACR_LOGIN_SERVER: ${{ secrets.ACR_LOGIN_SERVER }}
      ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
      ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
```

---

## Required Secrets (Set at Org Level)

| Secret | Description |
|---|---|
| `ACR_LOGIN_SERVER` | Azure Container Registry login server (`resolveopsai.azurecr.io`) |
| `ACR_USERNAME` | ACR username |
| `ACR_PASSWORD` | ACR password or service principal secret |
| `KUBECONFIG_DATA` | Base64-encoded AKS kubeconfig (for deploy workflow) |

Set these at: **GitHub Org → Settings → Secrets → Actions**

---

## ACR Login Server

All workflows use the `ACR_LOGIN_SERVER` secret — **never hardcode** the registry URL in any workflow file.

This allows the org to migrate to a different ACR instance by updating a single secret.

## CI/CD Pipeline Configuration

This repository uses a central reusable workflow for continuous integration and delivery.

### Required GitHub Secrets
The following secrets must be configured in the GitHub repository settings (Settings > Secrets and variables > Actions):

* SONAR_TOKEN: Token for SonarQube authentication
* SONAR_HOST_URL: URL of the SonarQube server
* SNYK_TOKEN: API token for Snyk vulnerability scanning
* AZURE_CLIENT_ID: Azure Client ID for OIDC authentication
* AZURE_TENANT_ID: Azure Tenant ID for OIDC authentication
* AZURE_SUBSCRIPTION_ID: Azure Subscription ID for OIDC authentication

### Required GitHub Variables
The following variables must be configured (Settings > Secrets and variables > Actions > Variables):

* ACR_LOGIN_SERVER: Azure Container Registry login server (e.g., esolveopsai.azurecr.io)
