# cmoreira.dev — HomeLab 🚀

Organização do meu homelab: um cluster **Kubernetes (Talos)** que corre sobre
Proxmox, com workloads e infraestrutura geridos por **GitOps**.

**Como as peças encaixam:**

- **Ingress** via NGINX Gateway Fabric (Gateway API).
- **GitOps** com Argo CD — cada repo `gitops.*` é reconciliado automaticamente.
- **IaC em dois níveis:**
  - *Tier 1* — infra base (AWS + Azure + Proxmox) em `iac.homelab-live-infra`,
    via Terragrunt/OpenTofu (GitHub Actions para a cloud, `terragrunt` local para Proxmox).
  - *Tier 2* — dependências de aplicação, reconciliadas no cluster pelo operador
    **Burrito** a partir da pasta `terraform/` de cada repo GitOps.
- **Secrets** via external-secrets ← AWS SSM Parameter Store.
- **Build & registry** — imagens para ECR, push por GitHub Actions com OIDC (sem chaves estáticas).

📖 Documentação completa: **[docs.cmoreira.dev](https://docs.cmoreira.dev)**

> `🔒` = repositório privado.

---

## 📚 Documentação

| Repo | Descrição |
|------|-----------|
| [docs.cmoreira-dev.github.io](https://github.com/cmoreira-dev/docs.cmoreira-dev.github.io) | Site de documentação (MkDocs) — arquitetura, runbooks e padrões. Publicado em [docs.cmoreira.dev](https://docs.cmoreira.dev). |

## 🏗️ Infraestrutura como Código

| Repo | Descrição |
|------|-----------|
| [iac.homelab-live-infra](https://github.com/cmoreira-dev/iac.homelab-live-infra) `🔒` | **Tier 1** — estado da infra base (AWS + Azure + Proxmox). Terragrunt + OpenTofu. |
| [iac-aws-ecr-pipeline](https://github.com/cmoreira-dev/iac-aws-ecr-pipeline) | Módulo Terraform reutilizável: repositórios ECR (create-on-push) + role OIDC/IAM para o pipeline de build/push. |
| [iac-proxmox-lxc](https://github.com/cmoreira-dev/iac-proxmox-lxc) | Módulo Terraform reutilizável para containers LXC no Proxmox. |
| [homelab-bootsrap-k3s](https://github.com/cmoreira-dev/homelab-bootsrap-k3s) `🔒` | Scripts de bootstrap do cluster e dos addons base. |

## ⚙️ Platform Engineering & GitOps

| Repo | Descrição |
|------|-----------|
| [gitops.core-addons](https://github.com/cmoreira-dev/gitops.core-addons) `🔒` | Addons base do cluster: cert-manager, NGINX Gateway Fabric, external-secrets, Burrito, Renovate. Dependência de quase todos os outros. |
| [gitops.ai-core-addons](https://github.com/cmoreira-dev/gitops.ai-core-addons) `🔒` | Addons de inferência IA/GPU — Ollama, LiteLLM. |
| [gitops.monitoring](https://github.com/cmoreira-dev/gitops.monitoring) `🔒` | Stack de observabilidade (Prometheus, Grafana, Loki). |
| [gitops.cnpg](https://github.com/cmoreira-dev/gitops.cnpg) | Operador CloudNativePG (PostgreSQL). |
| [gitops.headlamp](https://github.com/cmoreira-dev/gitops.headlamp) | UI Headlamp para Kubernetes. |
| [gitops.echoserver](https://github.com/cmoreira-dev/gitops.echoserver) | Echo server — testes de ingress/rede. |
| [gitops.generic-app-chart](https://github.com/cmoreira-dev/gitops.generic-app-chart) | Helm chart de biblioteca partilhado para apps do homelab (Deployment + Service + HTTPRoute + ExternalSecret). |
| [gitops.template](https://github.com/cmoreira-dev/gitops.template) | Template para novos repos GitOps de aplicação. |
| [backstage.homelab](https://github.com/cmoreira-dev/backstage.homelab) | Portal interno de developers (Backstage). |

## 🤖 Aplicações

### teupadel.com — análise de movimento no padel com IA

| Repo | Descrição |
|------|-----------|
| [api.ia.teupadel.com](https://github.com/cmoreira-dev/api.ia.teupadel.com) `🔒` | Backend (inferência IA + lógica de negócio). |
| [ui.ia.teupadel.com](https://github.com/cmoreira-dev/ui.ia.teupadel.com) `🔒` | Frontend (React). |
| [gitops.teupadel.com](https://github.com/cmoreira-dev/gitops.teupadel.com) | Deploy GitOps da solução. |

### Sara — acordes e cifras sem anúncios (`local.cmoreira.dev/sara`)

| Repo | Descrição |
|------|-----------|
| [api.ia.local-sara](https://github.com/cmoreira-dev/api.ia.local-sara) `🔒` | Backend FastAPI que resolve acordes/letras do Cifra Club. |
| [ui.ia.local-sara](https://github.com/cmoreira-dev/ui.ia.local-sara) `🔒` | Player de prática de cifras, sem anúncios. |
| [gitops.local-sara](https://github.com/cmoreira-dev/gitops.local-sara) | Deploy GitOps da Sara. |

## 🧩 Org

| Repo | Descrição |
|------|-----------|
| [.github](https://github.com/cmoreira-dev/.github) | Perfil público da org (este README) e workflows reutilizáveis. |
