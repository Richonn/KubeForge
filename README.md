![Build](https://github.com/TON_USER/kubeforge/actions/workflows/ci.yml/badge.svg)
![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Kubernetes](https://img.shields.io/badge/kubernetes-v1.32-326CE5?logo=kubernetes&logoColor=white)

# KubeForge

A hands-on Kubernetes learning project — three minimal services deployed on a local Minikube cluster, progressively hardened with GitOps, security, and autoscaling.

## Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              GitHub Actions CI/CD            │
                        │  push → build → scan (Trivy) → push → bump  │
                        └─────────────────┬───────────────────────────┘
                                          │ git push (infra/charts)
                                          ▼
                        ┌─────────────────────────────────────────────┐
                        │                  ArgoCD                      │
                        │         auto-sync infra/charts/kubeforge     │
                        └─────────────────┬───────────────────────────┘
                                          │ kubectl apply
                                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Minikube cluster                               │
│                                                                         │
│  ┌──────────────┐    ┌──────────────────────────────────────────────┐  │
│  │ NGINX Ingress│    │              default namespace                │  │
│  │              │───▶│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │ /api → :3000 │    │  │   api    │  │ frontend │  │  worker  │  │  │
│  │ /app → :80   │    │  │ NestJS   │  │  React   │  │  Node.js │  │  │
│  └──────────────┘    │  │ HPA 1-3  │  │ HPA 1-3  │  │ HPA 1-3  │  │  │
│                      │  └──────────┘  └──────────┘  └──────────┘  │  │
│  ┌──────────────┐    │                                              │  │
│  │ cert-manager │    │  NetworkPolicy: deny-all + allow explicit    │  │
│  │ TLS self-    │    │  RBAC: ServiceAccount per service            │  │
│  │ signed       │    │  Secrets: Sealed Secrets (kubeseal)          │  │
│  └──────────────┘    └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Stack

| Component | Role |
|-----------|------|
| Minikube | Local Kubernetes cluster |
| NGINX Ingress | HTTP routing by path |
| cert-manager | TLS self-signed certificate |
| Sealed Secrets | Encrypted secrets in Git |
| ArgoCD | GitOps continuous delivery |
| GitHub Actions | CI/CD pipeline |
| Trivy | Container image vulnerability scanning |
| Helm | Kubernetes package manager |
| HPA | Horizontal Pod Autoscaler |

## Services

| Service | Stack | Port |
|---------|-------|------|
| `api` | NestJS | 3000 |
| `frontend` | React + Vite → Nginx | 80 |
| `worker` | Node.js/TS (no HTTP) | — |

## Prerequisites

- Docker
- [Minikube](https://minikube.sigs.k8s.io/)
- kubectl
- helm
- kubeseal
- A GitHub account with a PAT (`write:packages` scope) stored as `CR_PAT` repo secret

## Local setup

### 1. Start the cluster

```bash
minikube start --driver=docker --cni=calico
minikube addons enable ingress
minikube addons enable metrics-server
```

### 2. Install cluster dependencies

```bash
# cert-manager
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true

# Sealed Secrets
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets

# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.0/manifests/install.yaml
```

### 3. Create the GHCR pull secret

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=Richonn \
  --docker-password=<your-PAT> \
  --dry-run=client -o yaml | \
  kubeseal --controller-name=sealed-secrets \
           --controller-namespace=kube-system -o yaml \
  > infra/base/ghcr-sealed-secret.yml
```

### 4. Deploy via ArgoCD

```bash
kubectl apply -f infra/argocd/application.yml
kubectl apply -f infra/bootstrap/
```

### 5. Access the UI

```bash
# Add to /etc/hosts
echo "$(minikube ip) k8s-forge.local" | sudo tee -a /etc/hosts

# ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080 — admin / $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# App
# https://k8s-forge.local/app
# https://k8s-forge.local/api/health
```

## CI/CD Pipeline

```
git push (app/)
    └─▶ build.yml     — docker build + push :ci tag to GHCR
    └─▶ scan.yml      — Trivy scan :ci (CRITICAL/HIGH, blocks on vuln)
    └─▶ push.yml      — push :latest + :sha-<commit> tags
    └─▶ bump.yml      — update image tags in infra/charts/kubeforge/values.yaml
                              └─▶ ArgoCD detects diff → sync → rolling update
```

Changes to `infra/` do not retrigger the CI (prevents bump loop).

## Security checklist

- [x] Multi-stage Docker builds — build tools not present in final image
- [x] npm removed from final stage — eliminates npm's internal dep vulnerabilities
- [x] Alpine packages pinned (`zlib>=1.3.2-r0`, `libexpat>=2.7.5-r0`)
- [x] Trivy scan blocking on CRITICAL/HIGH unfixed vulnerabilities
- [x] `runAsNonRoot: true` on all containers
- [x] `readOnlyRootFilesystem: true` on all containers
- [x] `allowPrivilegeEscalation: false` on all containers
- [x] `capabilities: drop: [ALL]` on all containers
- [x] Resource `requests` and `limits` set on all containers
- [x] NetworkPolicy deny-all + explicit allow rules (Calico CNI)
- [x] RBAC — dedicated ServiceAccount per service, least privilege
- [x] Secrets encrypted at rest with Sealed Secrets (kubeseal)
- [x] No plain secrets committed to Git

## Architecture decisions

See [DECISIONS.md](./DECISIONS.md).
