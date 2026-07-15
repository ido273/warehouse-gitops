# warehouse-gitops

GitOps manifests for the Smart Warehouse project. This repo holds no application code and no Terraform — just ArgoCD `Application` definitions and per-environment Helm values. ArgoCD watches it and reconciles the cluster to match.

## App of Apps pattern

The actual root/bootstrap `Application` is **not** in this repo — it's provisioned by `warehouse-infra` (Terraform, `modules/argocd/main.tf`, via `kubectl_manifest.root_app`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/ido273/warehouse-gitops.git
    targetRevision: master
    path: apps
    directory:
      recurse: true
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

That root Application watches this repo's `apps/` directory with `recurse: true`. Every `Application` manifest placed under `apps/` is automatically discovered and applied — that's the "App of Apps" part: one root Application manages a set of child Applications, one per microservice, without ArgoCD needing to be reconfigured each time a service is added.

## Directory structure

```
apps/
  ai-tagging.yaml       # Application: ai-tagging
  auth-service.yaml     # Application: auth-service
  backend.yaml          # Application: backend
  frontend.yaml         # Application: frontend
  mysql.yaml            # Application: mysql
envs/
  production/
    ai-tagging-values.yaml
    auth-service-values.yaml
    backend-values.yaml
    frontend-values.yaml
    mysql-values.yaml
ingress.yaml
```

- `apps/*.yaml` — one ArgoCD `Application` per service. These reference the Helm chart (which lives in `warehouse-app`) and the values file for the target environment (in this repo).
- `envs/production/*-values.yaml` — the actual environment-specific Helm values (image tag, replica count, env vars, ingress config, IRSA role ARNs, etc). These are what `cd.yaml` in `warehouse-app` updates automatically on every merge to `master` (bumping the `tag:` field).

## How ArgoCD uses this repo: multi-source Applications

Each `apps/<service>.yaml` is a **multi-source** Application — it pulls from two repos at once:

```yaml
spec:
  sources:
    - repoURL: https://github.com/ido273/warehouse-app.git
      targetRevision: master
      path: <service>/helm
      helm:
        valueFiles:
          - values.yaml
          - $values/envs/production/<service>-values.yaml
    - repoURL: https://github.com/ido273/warehouse-gitops.git
      targetRevision: master
      ref: values
```

- Source 1 (`warehouse-app`) supplies the Helm **chart** (`<service>/helm/`) and its chart-default `values.yaml`.
- Source 2 (this repo) is referenced as `$values` — it doesn't get deployed itself, it just makes `envs/production/<service>-values.yaml` available to source 1's `helm.valueFiles`.

This is what lets image tags and environment config live in `warehouse-gitops` (fast-changing, CI-driven) while the chart templates stay in `warehouse-app` (slow-changing, reviewed with the app code) — one Application, two repos, no chart duplication.

## Force-syncing an Application

Normally sync is automatic (`prune: true`, `selfHeal: true` on every Application). To force an immediate sync without waiting for ArgoCD's next reconciliation:

```bash
kubectl patch application <app-name> -n argocd --type merge -p \
  '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {"revision": "HEAD", "prune": true}}}'
```

Replace `<app-name>` with one of `ai-tagging`, `auth-service`, `backend`, `frontend`, `mysql`, or `root-app` itself (to re-discover changes under `apps/`). Equivalent with the ArgoCD CLI: `argocd app sync <app-name> --force`.

## Environment structure

**Production only.** There is exactly one environment directory, `envs/production/` — no `envs/staging` or `envs/dev` exists. Adding a second environment would mean a new `envs/<name>/` directory with its own values files, plus either parameterizing the `Application` manifests or adding a second set of `apps/<service>-<env>.yaml` files pointing at the new values path.
