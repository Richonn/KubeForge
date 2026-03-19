# KubeForge

A hands-on Kubernetes learning project — three minimal services deployed on a local Minikube cluster, progressively hardened with GitOps, RBAC, and sealed secrets.

## Services

| Service | Stack | Port |
|---------|-------|------|
| `api` | NestJS | 3000 |
| `frontend` | React + Vite → Nginx | 80 |
| `worker` | Node.js/TS (no HTTP) | — |

## Prerequisites

- Docker
- Minikube: `minikube start --driver=docker`
- kubectl

## Local setup

```bash
# Start cluster
minikube start --driver=docker
kubectl get nodes

# Apply manifests
kubectl apply -f infra/base/configmap.yml
kubectl apply -f infra/base/api/
kubectl apply -f infra/base/frontend/
kubectl apply -f infra/base/worker/

# Check pods
kubectl get pods -A
```

## Build & push images

```bash
docker login ghcr.io

docker build -t ghcr.io/richonn/kubeforge-api:latest app/api/
docker build -t ghcr.io/richonn/kubeforge-frontend:latest app/frontend/
docker build -t ghcr.io/richonn/kubeforge-worker:latest app/worker/

docker push ghcr.io/richonn/kubeforge-api:latest
docker push ghcr.io/richonn/kubeforge-frontend:latest
docker push ghcr.io/richonn/kubeforge-worker:latest
```

## Architecture decisions

See [DECISIONS.md](./DECISIONS.md).
