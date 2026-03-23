# Ericsson IBM App Repo (Deployment Config)

This repository contains the Kubernetes deployment and release orchestration for a Python app.

The main goal here is to promote one application image through **staging** and then **production** using:
- **Kustomize** for environment-specific manifests
- **Skaffold** profiles for render/deploy config
- **Google Cloud Deploy** for staged rollout
- **GitHub Actions** to trigger releases automatically

## What We Are Doing Here

This repo is acting as the **deployment/config repo** in a GitOps-style flow:

1. Another repo (referred to in comments as "Repo 1") builds/scans/signs the app image.
2. That repo updates `base/kustomization.yaml` with the exact image digest/tag.
3. A push to `main` that changes `base/kustomization.yaml` triggers this repo’s workflow.
4. GitHub Actions applies `clouddeploy.yaml` and creates a Cloud Deploy release.
5. Cloud Deploy promotes to:
   1. `staging-cluster`
   2. `prod-cluster` (with a **manual approval gate**)

So the responsibility of this repository is safe, repeatable environment promotion rather than application source code.

## Repository Structure

- `base/`
  - Base Kubernetes manifests (`Deployment`, `Service`) used by all environments.
- `overlays/staging/`
  - Adds `staging-` resource name prefix and deploys into `staging` namespace.
- `overlays/prod/`
  - Adds `prod-` prefix, deploys into `prod` namespace, scales replicas to 3, and changes Service `nodePort` to `30081`.
- `skaffold.yaml`
  - Defines `staging` and `prod` profiles mapped to overlays.
- `clouddeploy.yaml`
  - Defines Cloud Deploy delivery pipeline and targets.
  - Includes `requireApproval: true` for production.
- `.github/workflows/deploy.yaml`
  - Triggered on `main` when `base/kustomization.yaml` changes.
  - Authenticates to GCP via Workload Identity Federation.
  - Applies Cloud Deploy pipeline + creates a release.

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
- Promotion requires manual approval in Cloud Deploy

## Prerequisites

- GCP project with:
  - GKE cluster(s)
  - Cloud Deploy enabled
  - Artifact Registry configured
- GitHub repository secrets configured:
  - `WIF_PROVIDER`
  - `GCP_SERVICE_ACCOUNT`
  - `ARTIFACT_REGION`
  - `GCP_PROJECT_ID`
- `gcloud`, `kubectl`, `kustomize`, and optionally `skaffold` for local validation

## Typical Workflow

1. Update image reference in `base/kustomization.yaml` (often automated by upstream pipeline).
2. Merge/push to `main`.
3. GitHub Action creates a Cloud Deploy release.
4. Verify rollout in staging.
5. Approve production promotion in Cloud Deploy UI/CLI.

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

Apply Cloud Deploy pipeline config manually:

```bash
gcloud deploy apply --file=clouddeploy.yaml --region=<REGION> --project=<PROJECT_ID>
```

## Notes

- Some values in this repo are still template-like or PoC-oriented (for example `YOUR_PROJECT_ID` in `base/kustomization.yaml`).
- Before production use, confirm cluster IDs, image registry path, service exposure strategy (`NodePort` vs `LoadBalancer`), and security/compliance settings.
