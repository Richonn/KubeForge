# Architecture Decisions

## ADR-001 — Deployment vs ReplicaSet vs Pod

**Date:** 2026-03-19
**Status:** Accepted

### Context

When deploying workloads to Kubernetes, three levels of abstraction are available:
- **Pod**: the atomic unit, runs one or more containers
- **ReplicaSet**: ensures N replicas of a Pod are running at all times
- **Deployment**: wraps a ReplicaSet and adds lifecycle management

### Decision

Use **Deployment** for all three services (api, frontend, worker).

### Rationale

| Option | Self-healing | Rolling updates | Rollback | Verdict |
|--------|-------------|-----------------|----------|---------|
| Pod directly | No | No | No | Too fragile |
| ReplicaSet | Yes | No | No | Missing update strategy |
| Deployment | Yes | Yes | Yes | **Chosen** |

A Deployment is the correct default for any stateless workload. It manages a ReplicaSet internally, enabling zero-downtime rollouts (`RollingUpdate` strategy by default) and easy rollback via `kubectl rollout undo`.

---

## ADR-002 — Sealed Secrets vs native Kubernetes Secrets

**Date:** 2026-03-19
**Status:** Accepted

### Context

The GHCR pull secret (`ghcr-secret`) needs to be stored somewhere. Kubernetes native Secrets are base64-encoded, not encrypted — committing them to Git exposes credentials.

### Decision

Use **Sealed Secrets** (Bitnami) to encrypt secrets before committing them.

### Rationale

| Option | Git-safe | Cluster-managed | Complexity |
|--------|----------|-----------------|------------|
| Native Secret (plain) | No — base64 is not encryption | Yes | Low |
| Native Secret (gitignored) | Yes, but not in Git | Manual apply | Medium |
| Sealed Secrets | Yes — asymmetric encryption | Yes | Medium |
| External Secrets (Vault, AWS SM) | Yes | Yes | High |

Sealed Secrets fits the GitOps model: the encrypted `SealedSecret` is committed to Git, the controller decrypts it on the cluster using its private key. No plaintext ever leaves the cluster.

---

## ADR-003 — NetworkPolicy with Calico CNI

**Date:** 2026-03-19
**Status:** Accepted

### Context

Kubernetes NetworkPolicies are only enforced if a compatible CNI plugin is installed. Minikube's default CNI (bridge) does not enforce them.

### Decision

Use **Calico** as the CNI plugin (`minikube start --cni=calico`).

### Traffic model

```
Internet → NGINX Ingress → frontend (port 80)
Internet → NGINX Ingress → api (port 3000)
api → worker (port 8080)
all pods → DNS (port 53 UDP/TCP)
everything else: denied
```

### Rationale

Default deny-all + explicit allow rules enforces least-privilege networking. Without Calico, NetworkPolicies exist in the API but have no effect — a silent security gap.

---

## ADR-004 — GitHub Actions reusable workflows (3-file split)

**Date:** 2026-03-19
**Status:** Accepted

### Context

The CI/CD pipeline needs to: build images, scan for vulnerabilities, push to GHCR, and bump image tags in the infra repo.

### Decision

Split into 4 reusable workflows called from a single orchestrator:
- `ci.yml` — orchestrator, defines job order and conditions
- `build.yml` — builds and pushes `:ci` tag
- `scan.yml` — Trivy scan on `:ci` tag (blocking)
- `push.yml` — pushes `:latest` + `:sha-*` tags
- `bump.yml` — updates `values.yaml` with new SHA tag

### Rationale

- **Separation of concerns**: each file has one responsibility
- **Reusability**: workflows can be called independently
- **Security gate**: scan runs after build but before final push — a vulnerable image never gets a production tag
- **Loop prevention**: `paths-ignore: infra/**` prevents the bump commit from retriggering the CI

---

## ADR-005 — Helm umbrella chart

**Date:** 2026-03-20
**Status:** Accepted

### Context

Raw Kubernetes manifests are static — image tags, replica counts, and resource limits are hardcoded. Updating them requires editing multiple files.

### Decision

Use a **Helm umbrella chart** with one sub-chart per service.

```
infra/charts/kubeforge/
├── Chart.yaml
├── values.yaml          ← single source of truth for all config
└── charts/
    ├── api/
    ├── frontend/
    └── worker/
```

### Rationale

- **Single `values.yaml`**: image tags bumped in one file by the CI
- **Sub-chart isolation**: each service keeps its own templates
- **ArgoCD native support**: ArgoCD renders Helm charts server-side, no manual `helm install` needed
- **Parameterized HPA**: `minReplicas`, `maxReplicas`, `targetCPU` configurable per service without touching templates

---

## ADR-006 — Image hardening: removing npm from final stage

**Date:** 2026-03-20
**Status:** Accepted

### Context

Trivy scans revealed 22 HIGH vulnerabilities in the `api` and `worker` images. All were in `/usr/local/lib/node_modules/npm/` — npm's own internal dependencies (`tar`, `glob`, `minimatch`). npm `overrides` in `package.json` only affect the app's dependencies, not npm's internal ones.

### Decision

Remove npm from the final stage of all Node.js images after `npm install/ci`:

```dockerfile
RUN npm ci --omit=dev && \
    rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx
```

### Rationale

The production container only needs `node` to run `dist/main`. npm is a build tool with no runtime purpose. Removing it eliminates its entire dependency tree from the attack surface. Result: 0 CRITICAL/HIGH vulnerabilities across all three images.

---

## ADR-007 — GitOps bootstrap with ArgoCD app-of-apps

**Date:** 2026-03-20
**Status:** Accepted

### Context

On every `minikube start`, cert-manager and Sealed Secrets are lost and must be reinstalled manually before ArgoCD can sync the application successfully.

### Decision

Create ArgoCD `Application` resources for cert-manager and Sealed Secrets in `infra/bootstrap/`, applied once after ArgoCD is installed.

### Rationale

- **Self-healing**: ArgoCD reinstalls dependencies if they drift or disappear
- **GitOps consistency**: all cluster state is declared in Git
- **Reduced manual steps**: after a cluster restart, only ArgoCD itself needs to be reinstalled; it handles everything else
