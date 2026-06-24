# Resolveops AI Templates

This repository contains reusable GitHub Actions workflows to implement the FitForge-style CI/CD and GitOps flow for Resolveops AI applications.

## Reusable Workflows

### 1. `reusable-pr-validation.yml`
Performs comprehensive validation for Pull Requests:
- SonarQube/SonarCloud SAST
- Snyk SCA
- Trivy Image Scanning (fails on HIGH/CRITICAL vulnerabilities)
- Ensures strict security standards before merge.

### 2. `reusable-docker-build-push.yml`
Builds and pushes Docker images to Azure Container Registry (ACR):
- Uses Azure OIDC login for secure authentication without long-lived credentials.
- Supports immutable tags.
- Disables `latest` tag by default to enforce explicit versioning.

### 3. `reusable-gitops-update.yml`
Updates the GitOps manifest repository (Helm or Kustomize) with the new image tags:
- Automatically commits with `[skip ci]` to prevent infinite CI loops.
- Avoids hardcoded secrets, pushing back to the configured `target_branch`.

### 4. `reusable-argocd-sync.yml`
Optional Argo CD sync workflow:
- Forces a sync using the Argo CD CLI.
- Runs only if `ARGOCD_SERVER` and `ARGOCD_AUTH_TOKEN` are provided. Not strictly required since Argo CD can auto-sync based on Git updates.

### 5. `reusable-notify.yml`
Provides notifications for workflow runs:
- Writes to GitHub Step Summary.
- Optionally sends a webhook/email notification without failing the main workflow on error.

## Usage

Please see the [CICD_USAGE_GUIDE.md](CICD_USAGE_GUIDE.md) for detailed examples of how to call these reusable workflows from your application repositories.
