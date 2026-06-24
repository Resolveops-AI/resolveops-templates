# Resolveops AI Templates

This repository contains reusable GitHub Actions workflows to implement the Final FitForge-style CI/CD and GitOps flow for Resolveops AI applications.

## Final CI/CD Model

The deployment process follows a strict "Build Once, Promote Everywhere" model:

1. **Dev Push**: Builds Docker images once.
2. **PR Validation**: Does not rebuild images. Verifies and scans the existing ACR images.
3. **Prod Release**: Does not rebuild images. Verifies, scans, and promotes the dev image to a semantic production tag.

## Reusable Workflows

### Dev Workflows
These are used during the Dev phase (on push to `main`):
- `_sast.yml`: SonarQube/SonarCloud SAST.
- `_sca.yml`: Snyk SCA.
- `_dockerfile-check.yml`: Dockerfile security and non-root checks.
- `_image-build-and-scan.yml`: Temporary image build and Trivy vulnerability scan.
- `reusable-docker-build-push.yml`: Pushes the dev image (`dev-<sha>`) to ACR.
- `reusable-gitops-update.yml`: Updates the GitOps manifests with the new dev image tag.
- `reusable-notify.yml`: Sends workflow notifications.

### PR Workflows
These are used during PR validation to ensure the dev image artifact is safe to merge:
- `_verify-acr-image.yml`: Verifies the dev image exists in ACR.
- `_trivy-scan-acr.yml`: Runs Trivy against the existing dev image in ACR without rebuilding.
- `reusable-notify.yml`: Sends workflow notifications.

### Prod Workflows
These are used for Production releases (on tag creation):
- `_verify-acr-image.yml`: Verifies the dev image exists in ACR before promotion.
- `_trivy-scan-acr.yml`: Runs a final Trivy scan against the existing dev image.
- `_promote-acr-image.yml`: Promotes the tested dev image to a semantic version (`vX.Y.Z`) inside ACR.
- `reusable-gitops-update.yml`: Updates the GitOps manifests with the promoted production tag.
- `reusable-notify.yml`: Sends workflow notifications.

## Usage

Please see the `CICD_USAGE_GUIDE.md` or the `resolveops-application` workflows for detailed examples of how to call these reusable workflows.
