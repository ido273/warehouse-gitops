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
  secrets/
    secret-store.yaml            # ClusterSecretStore: aws-secrets-manager
    mysql-external-secret.yaml   # ExternalSecret -> mysql-secret
    app-external-secret.yaml     # ExternalSecret -> app-secrets
    frontend-external-secret.yaml # ExternalSecret -> frontend-secret
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
- `apps/secrets/*.yaml` — **not** ArgoCD `Application` manifests, just plain Kubernetes/ESO custom resources (`ClusterSecretStore`, `ExternalSecret`). Picked up the same way as the `Application` files: the root-app watches `apps/` with `directory.recurse: true`, so anything under `apps/secrets/` gets applied automatically too, no separate wiring needed. See "Secrets management" below.
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

## Secrets management: `apps/secrets/`

`mysql-secret`, `app-secrets`, and `frontend-secret` (the Kubernetes Secrets the Helm charts' `secretKeyRef`s point at) are **not** written by Terraform or by any Helm chart in this repo. They're synced from AWS Secrets Manager by [External Secrets Operator (ESO)](https://external-secrets.io/), which `warehouse-infra` installs into the cluster. This directory is the sync configuration.

### `secret-store.yaml` — `ClusterSecretStore`

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

A `SecretStore` tells ESO *where* and *how* to authenticate to a secret backend — here, AWS Secrets Manager, authenticating as the `external-secrets` ServiceAccount in the `external-secrets` namespace (IRSA-bound by `warehouse-infra`, no static AWS credentials anywhere). It's a **`ClusterSecretStore`** (cluster-scoped), not a namespaced `SecretStore`, specifically because the IRSA ServiceAccount lives in the `external-secrets` namespace while the `ExternalSecret`s that use it live in `warehouse` — a namespaced `SecretStore` can only be referenced by `ExternalSecret`s in its *own* namespace, so cross-namespace auth needs the cluster-scoped kind plus an explicit `serviceAccountRef.namespace`.

### `*-external-secret.yaml` — `ExternalSecret`

Each one maps one AWS Secrets Manager secret (or a subset of its keys) onto one Kubernetes Secret, e.g.:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mysql-secret
  namespace: warehouse
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: mysql-secret
    creationPolicy: Owner
  data:
    - secretKey: root-password
      remoteRef:
        key: warehouse/mysql
        property: root_password
```

- `secretStoreRef` points at the `ClusterSecretStore` above.
- `target.name` / `creationPolicy: Owner` — ESO creates and owns a Kubernetes Secret with this name; if the `ExternalSecret` is deleted, the Secret is deleted too.
- `data[].remoteRef.key` — the AWS Secrets Manager secret name (`warehouse/mysql`, `warehouse/app-secrets`); `.property` — which JSON key inside that secret's value to pull, mapped to `.secretKey` (the key inside the resulting k8s Secret).
- `refreshInterval: 1h` on all three — ESO re-polls AWS Secrets Manager and updates the k8s Secret if the source value changed, without needing a redeploy. (Pods using the Secret as env vars still need a restart to pick up a changed value — ESO updates the Secret object, it doesn't reach into running containers.)

| ExternalSecret | Target k8s Secret | AWS source | Keys |
|---|---|---|---|
| `mysql-external-secret.yaml` | `mysql-secret` | `warehouse/mysql` | `root-password`, `password` |
| `app-external-secret.yaml` | `app-secrets` | `warehouse/app-secrets` | `jwt-secret`, `flask-secret`, `database-url` |
| `frontend-external-secret.yaml` | `frontend-secret` | `warehouse/app-secrets` | `secret-key` (reuses `flask-secret`) |

`warehouse/app-secrets` doesn't get created by anything automated — it must exist in AWS Secrets Manager before these `ExternalSecret`s can sync (see the comment in `app-external-secret.yaml` for the exact `aws secretsmanager create-secret` command and required keys).

## Force-syncing an Application

Normally sync is automatic (`prune: true`, `selfHeal: true` on every Application). To force an immediate sync without waiting for ArgoCD's next reconciliation:

```bash
kubectl patch application <app-name> -n argocd --type merge -p \
  '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {"revision": "HEAD", "prune": true}}}'
```

Replace `<app-name>` with one of `ai-tagging`, `auth-service`, `backend`, `frontend`, `mysql`, or `root-app` itself (to re-discover changes under `apps/`, including new files under `apps/secrets/`). Equivalent with the ArgoCD CLI: `argocd app sync <app-name> --force`.

This only applies to actual `Application`-kind resources — `apps/secrets/*.yaml` (`ClusterSecretStore`/`ExternalSecret`) aren't Applications, so there's no `<app-name>` for them; `root-app` syncing re-applies their YAML as-is (same as any other file under `apps/`), but that doesn't make ESO re-fetch from AWS Secrets Manager early. To force *that*, bypass the 1h `refreshInterval` on a specific `ExternalSecret` directly:

```bash
kubectl annotate externalsecret <name> -n warehouse \
  force-sync=$(date +%s) --overwrite
```

Replace `<name>` with `mysql-secret`, `app-secrets`, or `frontend-secret`.

## Environment structure

**Production only.** There is exactly one environment directory, `envs/production/` — no `envs/staging` or `envs/dev` exists. Adding a second environment would mean a new `envs/<name>/` directory with its own values files, plus either parameterizing the `Application` manifests or adding a second set of `apps/<service>-<env>.yaml` files pointing at the new values path.
