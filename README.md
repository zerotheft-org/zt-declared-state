# zt-declared-state

**GitOps Desired State Repository for ZeroTheft**

This repository is the single source of truth for everything running on ZeroTheft's EKS clusters. It contains YAML manifests describing exactly how applications should look in each environment.

## What This Repo Is

- ✅ Kubernetes manifests (Deployments, Services, ConfigMaps, Ingress)
- ✅ Kustomize overlays per environment (dev, staging, prod)
- ✅ ArgoCD ApplicationSet definitions
- ✅ Environment-specific configurations (replicas, resource limits, image tags)
- ✅ Platform-wide configs (observability, ingress rules, certificates)

## What This Repo Is NOT

- ❌ Application source code (lives in `zt-be-*`, `zt-fe-*`, `zt-mo-app`)
- ❌ Secrets (use ExternalSecrets or SealedSecrets)
- ❌ Directly modified by developers (only via CI PRs or platform team)
- ❌ CI/CD pipeline definitions (live in source repos' `.github/workflows`)

## Repository Structure

```
zt-declared-state/
├── bootstrap/
│   └── argocd-appset.yaml          # ArgoCD App-of-Apps
├── apps/
│   ├── backend/
│   │   ├── auth-service/
│   │   ├── audit-service/
│   │   ├── billing-service/
│   │   └── notification-service/
│   ├── frontend/
│   │   ├── saas-app/
│   │   ├── tenant-app/
│   │   └── onboarding-app/
│   ├── infra/
│   │   └── device-discovery-service/
│   └── mobile/
│       └── flutter-app/
├── platform/
│   ├── observability/
│   │   ├── prometheus-rules/
│   │   └── grafana-dashboards/
│   └── infrastructure/
│       └── aws-load-balancer-controller/
└── environments/
    ├── dev/
    │   └── namespace-configs/
    ├── staging/
    │   └── namespace-configs/
    └── prod/
        ├── namespace-configs/
        ├── ingress-rules.yaml
        └── certificate-configs.yaml
```

## Per-App Structure

Each application follows the Kustomize base + overlay pattern:

```
apps/<category>/<service>/
├── base/
│   ├── deployment.yaml           # Pod spec with image placeholder
│   ├── service.yaml              # K8s service
│   ├── configmap.yaml            # Non-secret configuration
│   └── kustomization.yaml        # Base layer
└── overlays/
    ├── dev/
    │   ├── image-tag.yaml        # ← AUTO-UPDATED by CI (patch)
    │   ├── replica-count.yaml
    │   └── kustomization.yaml
    ├── staging/
    └── prod/
```

## Container Registry

All images are stored in **AWS ECR**:

```
<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/zerotheft/<service-name>:<tag>
```

## Deployment Flow (Future)

1. Developer commits to source repo (e.g., `zt-be-billing-service`)
2. CI builds Docker image and pushes to ECR
3. CD opens PR to this repo updating `image-tag.yaml`
4. Platform team reviews and merges PR
5. ArgoCD detects change and syncs to cluster

## Environments

| Environment | Namespace | Purpose | Replicas |
|---|---|---|---|
| dev | `zerotheft-dev` | Developer testing | 1 |
| staging | `zerotheft-staging` | QA / Pre-prod | 2 |
| prod | `zerotheft-prod` | Production | 3 |

## Modification Policy

- **Platform Team Only**: Direct commits to `main` require platform approval
- **CI Automation**: Image tag patches are auto-generated via PRs
- **Developers**: Submit changes via PR; never commit directly
