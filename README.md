# ResolveOps CI/CD Templates

This repository contains **only reusable GitHub Actions workflows** for the FitForge-style shared CI/CD setup. 

## Architectural Guidelines

This repository enforces strict boundaries:
- **NO** application code
- **NO** Terraform configurations
- **NO** Kubernetes manifests
- **NO** Helm charts
- **NO** Argo CD Application YAMLs
- **NO** HashiCorp Vault
- **NO** deployment-specific hardcoding

For Infrastructure as Code, see the `resolveops-infrastructure` repository.
For Application code, Helm charts, and Argo CD apps, see the `resolveops-application` repository.

## Final Workflow List

The following reusable workflows are provided in the `.github/workflows/` directory:

* `ci-reusable-template.yml`
* `docker-build-push-template.yml`
* `helm-updater-template.yml`
* `cd-reusable-template.yml`
* `notify-template.yml`

For detailed usage, branch behaviors, examples, required secrets, variables, and versioning rules, please read the [CICD_USAGE_GUIDE.md](CICD_USAGE_GUIDE.md).
