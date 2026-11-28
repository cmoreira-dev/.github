# app.* monorepo CI

An `app.<name>` repo holds one application: code (`api/`, `ui/`, ...), its cloud infra (`iac/`, Tier 2) and its
deployment (`gitops/`). Argo CD discovers it through the `app-repos` ApplicationSet (live-infra `cluster/argocd`)
once `gitops/argocd/` exists and is not just `.gitkeep`.

Start from the org workflow template **app.* monorepo CI** (`workflow-templates/app-ci.yml`). It wires:

| Reusable workflow | Role |
|---|---|
| `app-changes.yml` | Which top-level folders changed; output `changed` is a JSON map (`fromJSON(...).api`). Unknown diff base or a change under `.github/`, `scripts/`, `Makefile` runs everything |
| `build-push-ecr.yml` | Image publish on `main` (`context` = the folder). The ECR push role trusts the repo only if it is in `github_repos` (live-infra `aws/.../ecr`) |
| `terraform-gitops.yml` | Plan on PR, apply on `main` behind `production`, one job per root under `iac/`. See [terraform-gitops.md](terraform-gitops.md) |
| `app-gitops-validate.yml` | helm lint + template + kubeconform for `gitops/helm/*`, kustomize, and `gitops/argocd/` |
| `app-ci-gate.yml` | The single required check (`ci-ok / ci-ok`): skipped jobs pass, failed or cancelled ones fail |

Component tests (pytest, npm, service containers such as Postgres) stay in the repo's own `ci.yml`, since they differ per
application.

Rules:
- Keep Argo Application names when moving an application here: `ImageUpdater` matches them by name, and its
  `writeBackTarget` is relative to the Application's `source.path`, so the write-back follows the new repo.
- The image-updater commit touches only `gitops/`, so it never rebuilds images.
- `renovate.json` at the root must disable `gitops/**` and re-enable only `helmv3` under `gitops/helm/**`
  (copy it from `app.sara`): otherwise the `helm-values` manager fights argocd-image-updater over image tags and
  Renovate starts bumping pinned manifests such as CNPG images.
- `AWS_REGION` and `AWS_ROLE_ARN` come from the org; set only `PUBLISH_ENABLED` on the repo.
- Migration plan from the split `api.*` / `ui.*` / `gitops.*` repos: [monorepo-migration.md](monorepo-migration.md).
