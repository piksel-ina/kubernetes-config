# AGENTS.md

Flux GitOps repo for the Piksel staging Kubernetes cluster. Flux reconciles this
repo automatically; edit manifests here rather than applying to the cluster directly.

## Commands

| Task | Command | Notes |
|------|---------|-------|
| Install lint hooks (once/clone) | `pre-commit install` | not auto-installed; without it commits skip the checks |
| Lint all files | `pre-commit run --all-files` | check-yaml/json, large-files, mixed-line-ending, detect-secrets |
| Validate an overlay renders | `kustomize build apps/staging/<app>` | the real local check before pushing |
| Security scan | `checkov --framework kubernetes --directory . --skip-check CKV_K8S_21 --soft-fail` | CI job; soft-fail, non-blocking |
| Run a workflow | `argo submit --from workflowtemplate/<name> -n argo-workflows --serviceaccount argo-workflows-executor` | see `workflows/templates/README.md` |

## Setup & gotchas

- **`infrastructure/controllers/` is reference-only.** Those addons (cert-manager,
  ingress-nginx, nvidia, metrics-server) are deployed by Terraform
  (`terraform-iac/cluster-addons/`), not Flux — editing them here does nothing.
- **`apps/staging/kustomization.yaml` is not reconciled by Flux.** Flux uses the
  per-app Kustomizations in `clusters/staging/apps.yaml` instead.
- **CI flux-validation is a no-op.** `ci.yaml` calls `flux check --kustomization-file`
  (a flag absent in current flux) and swallows the error, so the job always passes.
  Don't trust it for correctness — validate with `kustomize build` locally.
- **detect-secrets:** many flagged entries are templated refs (`secretKeyRef`), not
  real secrets. Re-run the hook to sync `.secrets.baseline`; never inline a secret.

## Conventions

- **Flux Kustomization names ≠ overlay directory names.** In `clusters/staging/apps.yaml`:
  `jhub`→`jupyterhub`, `explorer`→`datacube-explorer`, `ows`→`datacube-ows`,
  `argo-workflows`→`argo`. Others match.
- **base/overlay:** `apps/base/<app>/` holds shared manifests; `apps/staging/<app>/`
  patches image tags, ingress, and env-specific config.
- **Reconcile order** (`apps.yaml` `dependsOn`): `infra-configs` → `infra-addons`;
  most apps depend on `infra-configs`; `workflows-templates` and `cron-jobs` depend
  on `argo-workflows`. `terria` and `tileserver` have no `dependsOn` (deploy immediately).

## Project map

- `clusters/staging/` — Flux reconciliation entrypoints (`apps.yaml`, `infrastructures.yaml`, `flux-system/`)
- `apps/base/<app>/`, `apps/staging/<app>/` — app definitions + staging overlays
- `infrastructure/staging/` — cluster configs (issuers, DB init, notifications) + pg-bouncer
- `workflows/{templates,cronwf,once}/` — Argo WorkflowTemplates, CronWorkflows, one-off jobs
- Golden sample: `apps/staging/tileserver/` — small, complete overlay (uses `configMapGenerator`)

## Boundaries

- Never `kubectl apply`, edit cluster state, or run direct DB operations — changes
  flow through Flux, or Argo/Terraform for DB and addons.
