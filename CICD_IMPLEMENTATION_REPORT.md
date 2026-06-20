# CI/CD Implementation Report

## Architecture Simplification & Restructuring

The `resolveops-templates` repository has been completely refactored to align with the final 3-repo architecture (`resolveops-application`, `resolveops-templates`, and `resolveops-infrastructure`). 

### Key Structural Changes

1. **Consolidation of CI Workflows**:
   - Removed all redundant workflows (`reusable-ci.yml`, `reusable-sonarqube.yml`, `reusable-snyk.yml`, `reusable-build-scan-push.yml`).
   - Replaced them with cleanly delineated responsibilities: `reusable-security-scan.yml`, `reusable-docker-build-push.yml`, `reusable-deploy-aks.yml`, and `reusable-notify.yml`.

2. **Removal of External Dependencies and Config Repos**:
   - **No more `resolveops-config`**: The deploy workflow now relies on the `resolveops-application` repository to provide its own Kubernetes manifests via Kustomize (`kustomize_path` input). It no longer checks out an external configuration repository.
   - **No more Docker Hub**: All Docker images are built and pushed strictly to Azure Container Registry (ACR), parameterized using the `acr_login_server` and `acr_name` inputs.

### Security and Best Practices

1. **Deprecation of `KUBECONFIG_DATA`**:
   - Previously, deployments relied on a long-lived base64-encoded Kubeconfig stored as a secret. This has been removed.
   - **OIDC Replacement**: We now strictly use Azure Workload Identity Federation (OIDC) through `azure/login@v2` and `az aks get-credentials` to securely access the AKS cluster.

2. **Strict Security Scans**:
   - **No Soft Fails**: `continue-on-error` configurations have been entirely removed from the security scanning pipeline. If a critical or high vulnerability is detected, the pipeline is blocked.
   - **No Unsafe Install Commands**: The `pip install -r requirements.txt || true` workaround has been eliminated to ensure true pipeline integrity. Build dependencies must resolve cleanly, or the scan fails.
   - **Enforced Build Order**: Scans must occur *before* the Docker image is built and pushed. This order is managed via `needs:` in the caller workflow.

3. **Standardized YAML Format**:
   - All workflow files have been reformatted to clean, multiline YAML syntax to ensure high readability, simple code reviews, and guaranteed compatibility with GitHub Actions syntax.
