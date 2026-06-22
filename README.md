# ResolveOps CI/CD Templates

This repository contains **only reusable GitHub Actions workflows** for the FitForge-style shared CI/CD setup. 

## Architectural Guidelines

This repository enforces strict boundaries:
- **NO** application code
- **NO** Terraform configurations
- **NO** Kubernetes manifests
- **NO** Helm charts
- **NO** Argo CD Application YAMLs
- **NO** secrets

For Infrastructure as Code, see the `resolveops-infrastructure` repository.
For Application code, Helm charts, and Argo CD apps, see the `resolveops-application` repository.

## Final Workflow List

The following reusable workflows are provided in the `.github/workflows/` directory:

* `reusable-security-scan.yml`
* `reusable-docker-build-push.yml`
* `reusable-helm-update.yml`
* `reusable-argocd-sync.yml`
* `reusable-notify.yml`

For detailed usage, examples, required secrets, variables, and versioning rules, please read the [CICD_USAGE_GUIDE.md](CICD_USAGE_GUIDE.md).
