# store-app-manifests

Helm chart for deploying a microservices e-commerce application on Amazon EKS using a GitOps workflow powered by ArgoCD.

## Architecture Overview

```
GitHub Actions (CI)          ArgoCD (CD)               EKS Cluster
─────────────────────        ──────────────────        ──────────────────────────
 Push to main branch   →     Detects repo change  →    Deploys to namespace:
 Build Docker images         Syncs Helm chart           store-app
 Push to Docker Hub          Applies manifests
 Update values.yaml
```

The pipeline follows a **GitOps model**: this repository is the single source of truth for the cluster state. No manual `kubectl` changes are made directly to the cluster.

## Repository Structure

```
store-app-manifests/
├── Chart.yaml                        # Helm chart metadata
├── values.yaml                       # Image tags and replica counts
├── application.yaml                  # ArgoCD Application manifest
├── iam_policy.json                   # IAM policy for AWS integrations
├── my-alerts.yml                     # Prometheus alerting rules
├── templates/
│   ├── deployment.yaml               # Deployments: python-store & my-db
│   ├── services.yaml                 # ClusterIP/NodePort services
│   ├── ingress.yaml                  # AWS ALB Ingress (internet-facing)
│   ├── storageclass-gp3.yaml         # Default gp3 StorageClass (EBS CSI)
│   └── pv.yml                        # PersistentVolumeClaim for PostgreSQL
└── prometheus-manifests/
    └── prometheus/                   # Prometheus Helm subchart + custom alerts
```

## Services

| Service | Image | Port | Replicas |
|---|---|---|---|
| `python-store` (Flask frontend) | `andresafag/python-frontend` | 5000 | 2 |
| `my-db` (PostgreSQL) | `andresafag/postgres-db` | 5432 | 1 |

Traffic flow: `Internet → ALB Ingress → python-store-service:80 → pod:5000`

## Prerequisites

- EKS cluster provisioned (see `../infrastructure/`)
- ArgoCD installed in the `argocd` namespace
- AWS Load Balancer Controller installed in the cluster
- EBS CSI Driver installed and configured
- Kubernetes secret `postgres-secret` created in the `store-app` namespace:

```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=<user> \
  --from-literal=POSTGRES_PASSWORD=<password> \
  --from-literal=POSTGRES_DB=<dbname> \
  --from-literal=DATABASE_URL=<connection_string> \
  -n store-app
```

## Deployment

ArgoCD manages the full lifecycle automatically. To register the application:

```bash
kubectl apply -f application.yaml
```

ArgoCD will then:
1. Watch the `main` branch of this repository
2. Sync the Helm chart to the `store-app` namespace
3. Auto-heal any configuration drift
4. Prune resources removed from Git


## Configuration

All environment-specific values live in `values.yaml`:

```yaml
frontend:
  replicaCount: 2
  image:
    repository: andresafag/python-frontend
    tag: "v9"          # Updated automatically by CI pipeline
    pullPolicy: Always

database:
  replicaCount: 1
  image:
    repository: andresafag/postgres-db
    tag: "v9"
    pullPolicy: Always
```

> Image tags are updated automatically by the GitHub Actions CI pipeline on every push to `main`.

## CI/CD Pipeline

The CI pipeline lives in `../frontend-db/.github/workflows/ci-cd.yml` and runs on every push to `main`:

1. **Build & Push** — Builds Docker images for `frontend` and `db`, tags them with the GitHub run number (`v<N>`), and pushes to Docker Hub.
2. **Update Manifests** — Checks out this repository and updates the `tag` field in `values.yaml` via `sed`.
3. **Commit & Push** — Commits the tag change back to `main`, which triggers ArgoCD to perform a rolling update.

## Storage

PostgreSQL data is persisted using an EBS-backed PVC:

- **StorageClass**: `gp3` (encrypted, provisioned by `ebs.csi.aws.com`)
- **PVC**: `ebs-pvc` — 10Gi, `ReadWriteOnce`
- **Binding mode**: `WaitForFirstConsumer`

## Observability

Prometheus is deployed via the `prometheus-manifests/` subchart. A custom alert group `PruebaDevOps` is configured in `my-alerts.yml` to validate that node metrics are being scraped correctly.

## Health Checks

The Flask frontend exposes two endpoints used by Kubernetes probes:

| Probe | Path | Port |
|---|---|---|
| Readiness | `/ready` | 5000 |
| Liveness | `/healthz` | 5000 |

Pods will not receive traffic until the readiness probe passes.

## Infrastructure

The EKS cluster is provisioned with Terraform under `../infrastructure/`. Key components:

- **EKS 1.35** with managed node groups
- **Cluster Autoscaler** via IRSA
- **ArgoCD** deployed via Helm (`argo-cd` chart v5.51.6)
- **VPC** with public/private subnets
- **ALB** for external traffic ingress
