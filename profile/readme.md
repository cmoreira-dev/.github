# cmoreira.dev — HomeLab 🚀

My homelab organization: a **Kubernetes (Talos)** cluster running on Proxmox,
with workloads and infrastructure managed through **GitOps**.

**How the pieces fit together:**

- **Ingress** via NGINX Gateway Fabric (Gateway API).
- **GitOps** with Argo CD — every `gitops.*` repo is reconciled automatically.
- **Two-tier IaC:**
  - *Tier 1* — base infra (AWS + Azure + Proxmox) in `iac.homelab-live-infra`,
    via Terragrunt/OpenTofu (GitHub Actions for the cloud, local `terragrunt` for Proxmox).
  - *Tier 2* — application dependencies, reconciled in-cluster by the **Burrito**
    operator from each GitOps repo's `terraform/` folder.
- **Secrets** via external-secrets ← AWS SSM Parameter Store.
- **Build & registry** — images to ECR, pushed by GitHub Actions with OIDC (no static keys).

📖 Full documentation: **[docs.cmoreira.dev](https://docs.cmoreira.dev)**

> `🔒` = private repository.

---

## 📚 Documentation

| Repo | Description |
|------|-------------|
| [docs.cmoreira-dev.github.io](https://github.com/cmoreira-dev/docs.cmoreira-dev.github.io) | Documentation site (MkDocs) — architecture, runbooks and patterns. Published at [docs.cmoreira.dev](https://docs.cmoreira.dev). |

## 🏗️ Infrastructure as Code

| Repo | Description |
|------|-------------|
| [iac.homelab-live-infra](https://github.com/cmoreira-dev/iac.homelab-live-infra) `🔒` | **Tier 1** — live state of the base infra (AWS + Azure + Proxmox). Terragrunt + OpenTofu. |
| [iac-aws-ecr-pipeline](https://github.com/cmoreira-dev/iac-aws-ecr-pipeline) | Reusable Terraform module: ECR repositories (create-on-push) + the OIDC/IAM role for the build/push pipeline. |
| [iac-proxmox-lxc](https://github.com/cmoreira-dev/iac-proxmox-lxc) | Reusable Terraform module for Proxmox LXC containers. |
| [homelab-bootsrap-k3s](https://github.com/cmoreira-dev/homelab-bootsrap-k3s) `🔒` | Scripts to bootstrap the cluster and its base addons. |

## ⚙️ Platform Engineering & GitOps

| Repo | Description |
|------|-------------|
| [gitops.core-addons](https://github.com/cmoreira-dev/gitops.core-addons) `🔒` | Core cluster addons: cert-manager, NGINX Gateway Fabric, external-secrets, Burrito, Renovate. A dependency of almost every other repo. |
| [gitops.ai-core-addons](https://github.com/cmoreira-dev/gitops.ai-core-addons) `🔒` | AI/GPU inference addons — Ollama, LiteLLM. |
| [gitops.monitoring](https://github.com/cmoreira-dev/gitops.monitoring) `🔒` | Observability stack (Prometheus, Grafana, Loki). |
| [gitops.cnpg](https://github.com/cmoreira-dev/gitops.cnpg) | CloudNativePG operator (PostgreSQL). |
| [gitops.headlamp](https://github.com/cmoreira-dev/gitops.headlamp) | Headlamp Kubernetes UI. |
| [gitops.echoserver](https://github.com/cmoreira-dev/gitops.echoserver) | Echo server — ingress/networking testing. |
| [gitops.generic-app-chart](https://github.com/cmoreira-dev/gitops.generic-app-chart) | Shared library Helm chart for homelab apps (Deployment + Service + HTTPRoute + ExternalSecret). |
| [gitops.template](https://github.com/cmoreira-dev/gitops.template) | Template for new application GitOps repos. |
| [backstage.homelab](https://github.com/cmoreira-dev/backstage.homelab) | Internal developer portal (Backstage). |

## 🤖 Applications

### teupadel.com — AI padel movement analysis

| Repo | Description |
|------|-------------|
| [api.ia.teupadel.com](https://github.com/cmoreira-dev/api.ia.teupadel.com) `🔒` | Backend (AI inference + business logic). |
| [ui.ia.teupadel.com](https://github.com/cmoreira-dev/ui.ia.teupadel.com) `🔒` | Frontend (React). |
| [gitops.teupadel.com](https://github.com/cmoreira-dev/gitops.teupadel.com) | GitOps deployment for the solution. |

### Sara — ad-free chords & lyrics (`local.cmoreira.dev/sara`)

| Repo | Description |
|------|-------------|
| [api.ia.local-sara](https://github.com/cmoreira-dev/api.ia.local-sara) `🔒` | FastAPI backend that resolves Cifra Club chords/lyrics. |
| [ui.ia.local-sara](https://github.com/cmoreira-dev/ui.ia.local-sara) `🔒` | Ad-free chords/lyrics practice player. |
| [gitops.local-sara](https://github.com/cmoreira-dev/gitops.local-sara) | GitOps deployment for Sara. |

## 🧩 Org

| Repo | Description |
|------|-------------|
| [.github](https://github.com/cmoreira-dev/.github) | Org public profile (this README) and reusable workflows. |
