# Ericsson IBM App Repo (GitOps Manifests)

This repository contains Kubernetes manifests for deploying the Python application with Kustomize overlays.

It is now intended to be reconciled by ArgoCD.

For the complete 3-repo operating guide, see [RUNBOOK.md](RUNBOOK.md).

## Purpose

This repo is the GitOps source of truth for runtime deployment configuration:

1. `erricson_ibm_poc` builds and signs the application image.
2. `erricson_ibm_poc` updates the image digest in this repo.
3. ArgoCD detects and syncs changes from this repo to Kubernetes.

## Repository Structure

- `base/`
  - Base Kubernetes manifests (`Deployment`, `Service`) used by all environments.
- `overlays/staging/`
  - Adds `staging-` resource name prefix and deploys into `staging` namespace.
- `overlays/prod/`
  - Adds `prod-` prefix, deploys into `prod` namespace, scales replicas to 3, and changes Service `nodePort` to `30081`.
- `skaffold.yaml`
  - Optional local render config for staging/prod overlays.

## Environment Behavior

### Staging

- Resource name prefix: `staging-`
- Namespace: `staging`
- Uses base replica count (2)
- Service NodePort from base: `30080`

### Production

- Resource name prefix: `prod-`
- Namespace: `prod`
- Replica override: `3`
- Service NodePort override: `30081`

## Prerequisites

- Kubernetes cluster with namespaces `staging` and `prod`
- ArgoCD installed and configured to watch this repository/branch
- `kubectl`, `kustomize` (and optionally `skaffold`) for local validation

## Typical Workflow

1. Update image reference in `base/kustomization.yaml` (typically automated by `erricson_ibm_poc`).
2. Commit/push to `main`.
3. ArgoCD syncs manifests to the target cluster(s).
4. Verify rollout in staging and production.

## Useful Commands

Render manifests locally:

```bash
kubectl kustomize overlays/staging
kubectl kustomize overlays/prod
```

Render with Skaffold profile:

```bash
skaffold render -p staging
skaffold render -p prod
```

Cluster checks:

```bash
kubectl get all -n staging
kubectl get all -n prod
```

## Notes

- This repo no longer includes any platform-specific deploy orchestrator config.
- If you use ArgoCD auto-sync, commits to `main` are applied automatically.
