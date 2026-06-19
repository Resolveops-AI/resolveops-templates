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
