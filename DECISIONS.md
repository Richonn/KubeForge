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
