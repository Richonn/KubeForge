# K8s-Forge — TODO

Plateforme de déploiement auto-hébergée pour microservices.  
Stack : minikube · NGINX Ingress · cert-manager · Sealed Secrets · ArgoCD · GitHub Actions · Helm · Trivy · k6

---

## Phase 1 — Fondations

### Environnement
- [x] Installer `minikube` et `kubectl`
- [x] Démarrer un cluster minikube (`minikube start --driver=docker`)
- [x] Vérifier que `kubectl get nodes` retourne le nœud prêt

### Services applicatifs
- [x] Créer 3 services minimalistes : `api` (ex. FastAPI/Express), `frontend` (ex. Nginx static), `worker` (ex. script Python/Node)
- [x] Écrire un `Dockerfile` pour chacun
- [x] Builder et pousser les images sur GitHub Container Registry (`ghcr.io`)

### Manifests Kubernetes de base
- [ ] Écrire un `Deployment` pour chaque service
- [ ] Écrire un `Service` (ClusterIP) pour chaque service
- [ ] Créer un `ConfigMap` pour les variables de configuration non-sensibles
- [ ] Appliquer les manifests et vérifier `kubectl get pods`
- [ ] Comprendre et documenter la différence Pod / ReplicaSet / Deployment dans `DECISIONS.md`

---

## Phase 2 — Networking & Ingress

### Ingress NGINX
- [ ] Activer l'addon ingress de minikube (`minikube addons enable ingress`)
- [ ] Écrire une ressource `Ingress` avec routing par path (`/api`, `/app`)
- [ ] Ajouter `k8s-forge.local` dans `/etc/hosts` pointant vers `minikube ip`
- [ ] Vérifier l'accès HTTP aux deux services via le nom de domaine

### TLS avec cert-manager
- [ ] Installer `cert-manager` via Helm
- [ ] Créer un `Issuer` self-signed
- [ ] Configurer le `Certificate` et l'annotation TLS dans l'`Ingress`
- [ ] Vérifier l'accès HTTPS sur `https://k8s-forge.local`

### NetworkPolicy
- [ ] Écrire une `NetworkPolicy` deny-all par défaut sur le namespace
- [ ] Ouvrir uniquement les flux nécessaires (ex. ingress → api, api → worker)
- [ ] Tester l'isolation avec `kubectl exec` + `curl`
- [ ] Documenter le modèle de flux dans `DECISIONS.md`

---

## Phase 3 — Sécurité

### RBAC
- [ ] Créer un `ServiceAccount` dédié pour chaque service
- [ ] Écrire des `Role` et `RoleBinding` au principe du moindre privilège
- [ ] Créer un `ClusterRole` de lecture seule pour le monitoring (optionnel)
- [ ] Vérifier les permissions avec `kubectl auth can-i`

### Gestion des secrets
- [ ] Installer `Sealed Secrets` (controller + CLI `kubeseal`)
- [ ] Chiffrer un secret avec `kubeseal` et commiter le `SealedSecret` dans le repo
- [ ] Supprimer tout `Secret` en clair du repo et du cluster
- [ ] Documenter pourquoi Sealed Secrets plutôt que Secret natif dans `DECISIONS.md`

### Hardening des Pods
- [ ] Ajouter un `securityContext` sur chaque Deployment :
  - [ ] `runAsNonRoot: true`
  - [ ] `readOnlyRootFilesystem: true`
  - [ ] `allowPrivilegeEscalation: false`
  - [ ] `drop: [ALL]` sur les capabilities
- [ ] Définir des `resources.requests` et `resources.limits` sur chaque container

### Scan de sécurité
- [ ] Installer `Trivy`
- [ ] Scanner les 3 images (`trivy image <image>`)
- [ ] Corriger les vulnérabilités critiques/high
- [ ] Ajouter le scan Trivy dans le pipeline CI (Phase 4)
- [ ] Rédiger une checklist de sécurité dans le `README.md`

---

## Phase 4 — GitOps & CI/CD

### Structure du repo
- [ ] Organiser le repo en deux dossiers : `app/` (code source) et `infra/` (manifests K8s / Helm)
- [ ] Ajouter un `.gitignore` propre

### GitHub Actions — CI
- [ ] Créer un workflow `ci.yml` : lint + tests unitaires à chaque push
- [ ] Ajouter un job de build et push de l'image Docker sur `ghcr.io`
- [ ] Intégrer le scan Trivy comme job bloquant
- [ ] Ajouter un job qui met à jour le tag d'image dans `infra/` (commit automatique)

### ArgoCD — CD
- [ ] Installer ArgoCD sur le cluster (`kubectl apply -n argocd -f ...`)
- [ ] Accéder à l'UI ArgoCD via port-forward
- [ ] Créer une `Application` ArgoCD pointant sur le dossier `infra/` du repo
- [ ] Configurer le sync automatique (auto-sync + self-heal)
- [ ] Vérifier qu'un `git push` sur `app/` déclenche un déploiement complet sans intervention manuelle

### Tests end-to-end du pipeline
- [ ] Faire un changement visible (ex. modifier une route API)
- [ ] Vérifier la chaîne complète : push → CI → nouvelle image → bump infra → ArgoCD sync → pod redémarré

---

## Phase 5 — Scaling & Polish

### Helm
- [ ] Créer un chart Helm pour chaque service (ou un chart umbrella)
- [ ] Paramétrer les valeurs variables dans `values.yaml` (image tag, replicas, resources)
- [ ] Remplacer les manifests bruts par le chart dans `infra/`
- [ ] Vérifier le déploiement via `helm install` / `helm upgrade`

### Autoscaling
- [ ] Activer le metrics-server sur minikube (`minikube addons enable metrics-server`)
- [ ] Écrire un `HorizontalPodAutoscaler` (HPA) sur le service `api` (cible CPU 50%)
- [ ] Simuler une charge avec `k6` ou `hey`
- [ ] Observer le scaling avec `kubectl get hpa -w`

### Documentation & présentation
- [ ] Écrire un `README.md` complet :
  - [ ] Description du projet et objectifs
  - [ ] Schéma d'architecture (ASCII ou image)
  - [ ] Instructions d'installation pas à pas
  - [ ] Description de chaque composant et son rôle
- [ ] Compléter `DECISIONS.md` avec toutes les décisions techniques justifiées
- [ ] Épingler le repo sur le profil GitHub

---

## Fichiers clés à créer

```
k8s-forge/
├── app/
│   ├── api/
│   ├── frontend/
│   └── worker/
├── infra/
│   ├── base/          # Manifests bruts (phases 1-3)
│   └── charts/        # Charts Helm (phase 5)
├── .github/
│   └── workflows/
│       └── ci.yml
├── DECISIONS.md
└── README.md
```
