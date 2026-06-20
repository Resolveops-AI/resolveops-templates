# CI/CD Usage Guide

This guide explains how to use the reusable templates from `resolveops-templates` in the `resolveops-application` repository.

## Overview

The `resolveops-templates` repository contains reusable GitHub Actions workflows designed to standardize and simplify CI/CD pipelines across the organization. By centralizing the pipeline logic, we ensure:
- Consistent security scanning across all apps.
- Enforced use of OIDC for Azure deployments (no more `KUBECONFIG_DATA` or long-lived secrets).
- Clear separation between application code, CI/CD logic, and Infrastructure as Code.

## Workflow Example

Below is a complete example of how `resolveops-application` might call these templates in its `.github/workflows/build-deploy.yml` pipeline.

```yaml
name: Build and Deploy Application

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  # 1. Block the pipeline if security scans fail
  security-scan:
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-security-scan.yml@main
    with:
      service_name: "resolveops-app"
      service_type: "node" # or "python"
      working_directory: "."
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # 2. Build and push image only if security scans passed
  build-push:
    needs: security-scan
    if: github.ref == 'refs/heads/main'
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-docker-build-push.yml@main
    with:
      image_name: "resolveops-app"
      acr_name: ${{ vars.ACR_NAME }}
      acr_login_server: ${{ vars.ACR_LOGIN_SERVER }}
      image_tag: ${{ github.sha }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  # 3. Deploy to AKS
  deploy-aks:
    needs: build-push
    if: github.ref == 'refs/heads/main'
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-deploy-aks.yml@main
    with:
      azure_resource_group: ${{ vars.AZURE_RESOURCE_GROUP }}
      aks_cluster_name: ${{ vars.AKS_CLUSTER_NAME }}
      namespace: ${{ vars.AKS_NAMESPACE }}
      kustomize_path: "k8s/overlays/prod"
      deployment_names: "resolveops-app-deployment"
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  # 4. Notify pipeline status
  notify:
    needs: [security-scan, build-push, deploy-aks]
    if: always()
    uses: Resolveops-AI/resolveops-templates/.github/workflows/reusable-notify.yml@main
    with:
      service_name: "resolveops-app"
      status: ${{ contains(needs.*.result, 'failure') && 'failure' || 'success' }}
      run_url: "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
```

## Debugging Guide

If your pipeline fails, follow these steps to troubleshoot:

1. **Snyk or SonarQube failures**:
   - Check the `reusable-security-scan.yml` step output. Snyk is configured to fail the build automatically on high severity issues (`severity_threshold: high`).
   - If Snyk is blocking the build, you must resolve the vulnerabilities locally, update dependencies, and push again. Soft failing (`continue-on-error`) is explicitly disabled by design.
2. **OIDC Authentication Failures**:
   - Verify that `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` are properly set in your GitHub organization or repository secrets.
   - Ensure the calling repository has `permissions: { id-token: write, contents: read }` in its caller workflow.
   - Verify that the Workload Identity / Federated Credential is properly mapped to the calling repository in Azure.
3. **Kustomize / Deployment Failures**:
   - Ensure that `kustomize_path` points to a valid directory in `resolveops-application` containing a `kustomization.yaml` file.
   - Run `kubectl kustomize <path>` locally to verify there are no syntax errors in your YAML files.
