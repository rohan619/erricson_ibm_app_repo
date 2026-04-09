# Ericsson IBM 3-Repo Runbook (ArgoCD Flow)

This runbook documents how to operate the full delivery system across:

- `erricson_ibm_K8s_Infra` (infrastructure + ARC runners + ArgoCD platform setup)
- `erricson_ibm_poc` (application CI/security/build/sign + GitOps digest update)
- `erricson_ibm_app_repo` (Kubernetes manifests consumed by ArgoCD)

Use this document for onboarding, day-1 setup, daily releases, and troubleshooting.

## 1. System Overview

Secure GitOps delivery chain:

1. Infra repo provisions GKE and self-hosted GitHub runners.
2. App repo (`erricson_ibm_poc`) runs tests/scans, builds image, signs attestation.
3. App repo updates immutable image digest in `erricson_ibm_app_repo/base/kustomization.yaml`.
4. ArgoCD detects repo changes and syncs manifests to cluster.

In the current setup, `staging` and `prod` are separate namespaces in the same GKE cluster.

## 2. Repo Responsibilities

### `erricson_ibm_K8s_Infra`

Purpose:
- Provision cloud infrastructure and Kubernetes cluster.
- Install/manage ARC runner scale set.
- (Recommended) Install and configure ArgoCD.

### `erricson_ibm_poc`

Purpose:
- Own application code and DevSecOps checks.
- Build container image in Artifact Registry.
- Create Binary Authorization attestation.
- Update GitOps repo image digest.

Primary workflow:
- `.github/workflows/main.yml`

### `erricson_ibm_app_repo`

Purpose:
- Store Kubernetes manifests and overlays for environments.
- Provide GitOps source for ArgoCD reconciliation.

Key files:
- `base/kustomization.yaml`
- `base/deployment.yaml`
- `base/service.yaml`
- `overlays/staging/*`
- `overlays/prod/*`

## 3. End-to-End Flow

1. Code push in `erricson_ibm_poc` triggers CI.
2. Quality gates run (tests/scans).
3. Image builds and pushes to Artifact Registry.
4. Digest is resolved and attested.
5. `erricson_ibm_poc` updates `erricson_ibm_app_repo/base/kustomization.yaml` with immutable digest.
6. ArgoCD syncs the updated manifests to Kubernetes.

## 4. Required Prerequisites

- GCP project with:
  - GKE
  - Artifact Registry
  - Cloud Build
  - Binary Authorization
  - IAM Workload Identity Federation
- ArgoCD installed and connected to cluster.
- GitHub repositories created and accessible:
  - `rohan619/erricson_ibm_K8s_Infra`
  - `rohan619/erricson_ibm_poc`
  - `rohan619/erricson_ibm_app_repo`

## 5. GitHub Secrets Checklist

### A) `erricson_ibm_K8s_Infra`

- `WIF_PROVIDER`
- `GCP_SERVICE_ACCOUNT`
- ARC related credentials/secrets (as per infra setup)

### B) `erricson_ibm_poc`

- `SONAR_TOKEN`
- `FORTIFY_SCA_IMAGE` (optional)
- `WIF_PROVIDER`
- `GCP_SERVICE_ACCOUNT`
- `ARTIFACT_REGION`
- `GCP_PROJECT_ID`
- `ARTIFACT_REPO`
- `GITOPS_PAT` (write access to `erricson_ibm_app_repo`)

### C) `erricson_ibm_app_repo`

- No deployment-orchestrator-specific secrets required in this repo.
- Optional: repository secrets needed by any future lint/validation workflows.

## 6. Day-1 Bootstrap Procedure

1. Provision infra from `erricson_ibm_K8s_Infra`.
2. Deploy ARC runners and validate workflow execution.
3. Install/configure ArgoCD to track `erricson_ibm_app_repo`.
4. Validate GitOps render locally:

```bash
cd /home/rohan619/erricson_ibm_app_repo
kubectl kustomize overlays/staging
kubectl kustomize overlays/prod
```

5. Trigger first app pipeline in `erricson_ibm_poc` and confirm digest update commit lands in this repo.
6. Confirm ArgoCD syncs the new revision successfully.

## 7. Standard Release Procedure

1. Merge code to `main` in `erricson_ibm_poc`.
2. Confirm CI/test/security/build/sign jobs succeed.
3. Confirm GitOps digest update commit lands in `erricson_ibm_app_repo`.
4. Confirm ArgoCD sync status is healthy.
5. Validate workloads in staging/prod namespaces.

## 8. Manual/Operational Commands

Render checks:

```bash
kubectl kustomize overlays/staging | rg "image:|namespace:|name:"
kubectl kustomize overlays/prod | rg "image:|namespace:|name:"
```

Cluster state checks:

```bash
kubectl get all -n staging
kubectl get all -n prod
```

## 9. Troubleshooting

### A) CI pipeline fails in `erricson_ibm_poc`

Checks:
1. Validate required secrets and WIF setup.
2. Verify Cloud Build and Artifact Registry permissions.
3. Verify BinAuthz signing permissions and attestor names.

### B) GitOps update does not land in `erricson_ibm_app_repo`

Checks:
1. Verify `GITOPS_PAT` is valid and has repository write permission.
2. Confirm target branch is correct (`main` by default).
3. Confirm `base/kustomization.yaml` contains expected image mapping.

### C) ArgoCD does not apply latest commit

Checks:
1. Confirm ArgoCD Application points to correct repo URL, branch, and path.
2. Confirm repo credentials are valid in ArgoCD.
3. Check app sync/health status and controller logs.
4. Force sync and inspect resulting events.

## 10. Ownership Boundaries

- App build/sign/security controls live in `erricson_ibm_poc`.
- Deployment intent (manifests and env overlays) lives in `erricson_ibm_app_repo`.
- Cluster platform/runtime controls live in `erricson_ibm_K8s_Infra`.

Keeping these boundaries clean makes ArgoCD operations predictable and auditable.
