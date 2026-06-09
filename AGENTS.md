# AGENTS.md

## Repo purpose

Flux GitOps repository for Kubernetes cluster config (currently `staging` only). Flux watches this repo and reconciles changes automatically.

## Directory layout

- `clusters/staging/` — Flux Kustomization resources (the reconciliation entrypoints). Order: `flux-system/ → infra-configs → infra-addons → apps → workflows`
- `infrastructure/staging/configs/` — cluster-level configs (issuers, storage classes, DB init jobs, notifications)
- `infrastructure/staging/pg-bouncer/` — infra addons (remaining non-TF addon)
- `infrastructure/controllers/` — static controller manifests (cert-manager, ingress-nginx, nvidia, metrics-server). **Note:** these are referenced but deployment has been migrated to Terraform (`terraform-iac/cluster-addons/`)
- `apps/base/<app>/` — shared app definitions (HelmRepository, HelmRelease, base configs)
- `apps/staging/<app>/` — environment-specific overlays (patches, ingress, values)
- `workflows/templates/` — Argo WorkflowTemplates (STAC indexing, OWS updates, etc.)
- `workflows/cronwf/` — scheduled Argo CronWorkflows
- `workflows/once/` — one-off Argo Workflow manifests
- `testing/` — ad-hoc test manifests

## Validation & CI

CI runs three jobs on push/PR to `main` (`.github/workflows/ci.yaml`):

1. **pre-commit** — runs `.pre-commit-config.yaml` hooks (trailing whitespace, YAML lint, detect-secrets with `.secrets.baseline`)
2. **checkov** — `checkov --framework kubernetes` with `--soft-fail` and `--skip-check CKV_K8S_21` (CPU limits)
3. **flux-validation** — `flux check --kustomization-file` on every Flux Kustomization YAML

To run locally before pushing:

```bash
pre-commit run --all-files
```

## YAML conventions

- 2-space indentation, max line length 200
- `document-start` rule is disabled
- Flux-generated files are yamllint-ignored: `gotk-*.yaml`, `*-gotk.yaml`, `templates.yaml`, `flux-system/`
- Multi-document YAML allowed (`--allow-multiple-documents` in check-yaml)

## Secrets handling

- `detect-secrets` with baseline `.secrets.baseline` runs in pre-commit and CI
- `.env` is gitignored and excluded from detect-secrets scanning
- Many flagged entries are templated references (`secretKeyRef`, etc.) — verify before marking as false positives

## Flux Kustomization dependency chain

All apps depend on `infra-configs`. Workflows depend on `argo-workflows`:

```
flux-system → infra-configs → infra-addons (pg-bouncer)
                           → jupyterhub, argo, explorer, ows, monitoring, terria
                                          argo-workflows → workflow-templates, cron-jobs
```

## Argo Workflows

Submitted via `argo submit` with `--from workflowtemplate/<name>` in namespace `argo-workflows` with serviceaccount `argo-workflows-executor`. See `workflows/templates/README.md` for example commands.

## Gotchas

- `infrastructure/controllers/` manifests exist in repo but deployment is managed by Terraform — do not edit expecting Flux reconciliation
- `apps/staging/kustomization.yaml` lists resources but Flux uses per-app Kustomizations in `clusters/staging/apps.yaml` instead (the kustomization.yaml is not directly reconciled by Flux)
- `_task/` is gitignored; used for local task notes
