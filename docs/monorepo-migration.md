# Plano: um repo por produto (`app.<produto>`)

Status: proposta (2026-11-27). Substitui o split `api.*` / `ui.*` / `gitops.*` por produto.
Escopo: só os repos aplicacionais, ou seja, código nosso com `api.*`/`ui.*` + o seu `gitops.*` (sara, teupadel,
psirenatalima, rapporthub, documents, matrix-agent). Os repos de plataforma ficam como estão por agora: addons e
charts de terceiros (`gitops.core-addons`, `gitops.monitoring`, `gitops.cnpg`, `gitops.headlamp`, `gitops.kagent`,
`gitops.matrix`, `gitops.n8n`, `gitops.echoserver`, `gitops.generic-app-chart`, `gitops.template`, ...). A única mexida
de plataforma é a Wave 0, que só prepara o terreno.

## Objetivo

Um repo `app.<produto>` com tudo do produto, cada pasta com o seu pipeline:

```
app.<produto>/
├── api/  ui/  processor/      código; cada um com o seu Dockerfile
├── iac/<raiz>/                Terraform Tier 2 (S3, IAM, SSM do produto)
├── gitops/
│   ├── argocd/                Applications; é o que o ApplicationSet descobre
│   ├── helm/<componente>/     wrapper sobre gitops.generic-app-chart
│   └── cnpg/                  quando houver base de dados
├── docs/  mkdocs.yml          docs do produto (fundem os docs/ dos repos antigos)
├── catalog-info.yaml          System + Components + Resources (multi-documento)
├── renovate.json  CLAUDE.md  README.md
└── .github/workflows/ci.yml   filtro por pasta + gate único `ci-ok`
```

O padrão já existe e está provado em `app.convert-to-3d` (ADR 0001 desse repo): CI por pasta, `build-push-ecr.yml` com
`context`/`file`/`cache_scope`, write-back do image-updater em `gitops/helm/*/values.yaml` sem rebuild.
O que falta é tirar o registo manual do Argo e generalizar o resto.

## Pronto quando

- Cada produto vive num só `app.*`, deploy a partir de `main`, Argo Synced/Healthy, image-updater a fazer write-back
  no repo novo.
- Os repos antigos estão arquivados (não apagados) e nada no cluster, na AWS, no Backstage ou no Renovate aponta para eles.
- Docs (workspace `CLAUDE.md`, `conventions/repos`, profile README, golden path do Backstage) descrevem `app.*` como o
  padrão para serviços novos.

## Acoplamentos ao nome do repo (o que tem de mudar)

| Onde | Hoje | Impacto |
|---|---|---|
| `ApplicationSet/gitops-repos` (aplicado à mão, sem dono no Argo; verificado ao vivo 2026-11-27) | `scmProvider` com `^gitops\..*` + `pathsDoNotExist: argocd/.gitkeep`, `path: argocd`, `project: default`, requeue 3000 s, token `github-org-via-token`, sem `syncPolicy` | `app.*` não é descoberto; `app.convert-to-3d` tem root Application à mão; o ficheiro em `homelab-bootsrap-k3s` está desatualizado face ao live |
| ECR OIDC (`iac.homelab-live-infra/aws/.../ecr`, `github_repos`) | allowlist por nome de repo | repo novo não consegue publicar imagem |
| Tier 2 (`.../gitops-terraform-roles`, mapa `repos`) | chave = nome do repo; `state_key_prefix` explícito | registar o repo novo; o prefixo de state mantém-se, logo sem mover state |
| `ImageUpdater/cmoreira-dev-apps` (`gitops.core-addons`) | `namePattern` = nome da Application; `writeBackTarget` relativo ao `source.path` | nada muda se o nome da Application se mantiver; só o secret git tem de ter acesso ao repo novo |
| Backstage `cmoreiraDevProvider` | lê `/catalog-info.yaml` de cada repo | um ficheiro multi-documento na raiz; entidades antigas saem quando o repo é arquivado (confirmar) |
| Golden path (`backstage.homelab-golden-path/templates`) | gera `gitops.*` | passar a gerar `app.*` |
| Renovate | `onboarding: true` org-wide | repo novo recebe o PR "Configure Renovate"; regras do processor (deps que andam juntas) têm de vir junto |

## Wave 0 — plataforma (sem migrar nenhum produto)

Um PR por mudança, validado antes do merge (padrão de cutover).

Estado (2026-11-28, Wave 0 concluída):

| Item | PR | Estado |
|---|---|---|
| ApplicationSets no módulo (`applicationsets`, `adopt_applicationsets`) | iac-argocd#3 | merged, tag `v0.2.0` |
| `cluster/argocd`: adotar `gitops-repos`, criar `app-repos`, `preserveResourcesOnDeletion` | iac.homelab-live-infra#70, #72, #73 | aplicado 2026-11-28 (run 46132573378); #73 esvazia `adopt_applicationsets` |
| ECR: `app.*` na trust do role de push | iac.homelab-live-infra#71 | aplicado e verificado |
| Workflows reutilizáveis `app-changes`, `app-gitops-validate`, `app-ci-gate` + template `app-ci` | .github#22 | merged |

Descobertas que mudaram o plano:
- O `platform-bootstrap` foi arquivado em 2026-10-19; a camada do cluster vive em `iac.homelab-live-infra/cluster/*`
  (módulo `iac-argocd`). Os ApplicationSets passaram a ser donos dali.
- As Applications-pai `gitops.*` têm `resources-finalizer`: sem `preserveResourcesOnDeletion`, arquivar um repo apaga
  em cascata as filhas. O #70 tem de estar aplicado antes do primeiro cutover.
- Repos novos recebem o subject OIDC imutável (`repo:org@<id>/<repo>@<repo_id>`): o registo Tier 2 do `app.documents`
  só pode ser feito depois de o repo existir (Wave 5). O ECR aceita as duas formas, por isso foi pré-registado.
- A root `convert-to-3d` é adotada pelo pai `app.convert-to-3d` que o `app-repos` cria; não é preciso apagá-la.
- O `gitops-repos` vivo tinha um `template` vazio por defeito dentro do generator; o provider `kubernetes` não o
  consegue remover num update (a CRD exige `metadata`/`destination`), por isso o YAML declara-o tal como está (#72).
- O runner `rasp-srv-02` falhou com "No space left on device": tmpfs do `_work` subido de 256 MB para 512 MB.

Verificado no cluster (2026-11-28): os dois ApplicationSets com `preserveResourcesOnDeletion: true` e geridos pelo
OpenTofu; todos os pais `gitops.*` Synced/Healthy; `app.convert-to-3d` criado pelo `app-repos` e a rastrear a root
`convert-to-3d`. **Wave 0 concluída**; a Wave 1 pode começar.
- Renovate tem acesso a todos os repos (`selection=all`). O token do image-updater só fica provado no write-back da Wave 1.

1. **ApplicationSets com dono e a descobrir `app.*`** (iac-argocd#3 + live-infra#70). O `gitops-repos` não tinha dono
   (aplicado à mão); passa a ser do OpenTofu em `cluster/argocd`, a partir do spec **live**. Um segundo ApplicationSet
   `app-repos` (`^app\..*`, `path: gitops/argocd`, mesmos filtros de `.gitkeep`). Ambos com
   `syncPolicy.preserveResourcesOnDeletion: true`. Não em `gitops.core-addons`: um ApplicationSet gerido por uma
   Application que ele próprio gera seria circular.
2. **`app.convert-to-3d`** fica adotado automaticamente pelo `app-repos` (o pai novo adota a root existente). Achatar
   `gitops/argocd/apps/` para `gitops/argocd/` é opcional e pode ficar para depois.
3. **Workflow de CI reutilizável.** Extrair do `ci.yml` do convert-to-3d o que é genérico (`changed-folders.sh`,
   build de PR sem push, publish via `build-push-ecr.yml`, `ci-ok`) para um template/composite no `.github`, para cada
   repo novo ter um `ci.yml` curto. Incluir o caller do `terraform-gitops.yml` filtrado por `iac/**`.
   Pin de `build-push-ecr.yml` por SHA nos callers (hoje é `@main`).
4. **Tier 1** (live-infra#71, merge só com a tua confirmação, dispara apply): `ecr/terragrunt.hcl` com todos os
   `cmoreira-dev/app.*` previstos em `github_repos` (os antigos ficam até à Wave 6). O Tier 2 do `app.documents` fica
   para a Wave 5 (precisa do ID do repo).
5. **Credenciais.** Confirmar que o secret `argocd-image-updater-git-creds` e a GitHub App do Renovate cobrem repos novos
   (se a instalação for "selected repositories", adicionar ao criar cada repo).
6. **Backstage.** Exemplo de `catalog-info.yaml` multi-documento (System + Components com
   `backstage.io/source-location` por pasta + o label `homelab.cmoreira.dev/catalog-component` já usado pelo plugin do Argo);
   confirmar que repos arquivados saem do catálogo.
7. **Docs/convenção.** ADR na org (esta proposta, aceite) + `conventions/repos`: `app.*` passa a ser o padrão
   para produtos; o "não é descoberto pelo ApplicationSet" do `CLAUDE.md` do workspace deixa de ser verdade.

Saída da Wave 0: `app.convert-to-3d` deployado via ApplicationSet, nenhum produto mexido.

## Receita por wave (igual para cada produto)

**Preparar (sem tocar no cluster)**
1. Congelar merges nos repos antigos do produto (avisar, ou branch protection só-leitura).
2. Criar `cmoreira-dev/app.<produto>` (privado). Importar cada repo antigo **com histórico** numa subpasta
   (`git filter-repo --to-subdirectory-filter api`, depois `git merge --allow-unrelated-histories`): `api.*` → `api/`,
   `ui.*` → `ui/`, `gitops.*` → `gitops/` (o `argocd/` antigo vai para `gitops/argocd/`, `terraform/` para `iac/`).
3. Nos manifests de `gitops/argocd/*.yaml`: `repoURL` → repo novo, `path: helm/x` → `gitops/helm/x`. Os **nomes das
   Applications ficam iguais** (o ImageUpdater e os dashboards dependem deles).
4. `ci.yml` do template com o filtro por pasta; PRs passam a ter CI em todas as pastas (hoje só a API do teupadel tem).
   `PUBLISH_ENABLED=false` até ao cutover.
5. Juntar `docs/`, `CLAUDE.md`, `catalog-info.yaml`, `renovate.json` (com as regras específicas dos repos antigos).
6. Validar localmente: `helm template` de cada `gitops/helm/*` igual ao do repo antigo (diff vazio), builds Docker,
   testes. Push em `main` com `gitops/argocd/` só com `.gitkeep`: pelo filtro `pathsDoNotExist` o ApplicationSet ainda
   **não** descobre o repo, logo nada é criado no cluster. A descoberta acontece quando o `.gitkeep` sai no passo 8
   (até 50 min de requeue; forçar com refresh do ApplicationSet se for preciso).

**Cutover**
7. No repo `gitops.<produto>` antigo: adicionar `argocd.argoproj.io/sync-options: Prune=false` às Applications filhas
   e esperar que esteja **live** nas Applications.
8. No repo antigo: esvaziar `argocd/` (só `.gitkeep`). O pai antigo fica OutOfSync mas não apaga nada (`Prune=false`) e
   deixa de aplicar as filhas; no requeue seguinte o `gitops-repos` apaga o pai sem cascata
   (`preserveResourcesOnDeletion`). Tem de vir **antes** do passo 9: com os dois pais a declarar a mesma Application,
   com `selfHeal`, ficariam a alternar o `repoURL`.
9. No repo novo: preencher `gitops/argocd/` (mesmos nomes, `repoURL`/`path` novos) e apagar o `.gitkeep`. No requeue
   do `app-repos` (até 50 min) o pai novo adota as Applications; com render idêntico não há pods novos. Conferir
   Synced/Healthy.
10. `PUBLISH_ENABLED=true`; um commit trivial por componente para provar a cadeia: build → ECR → write-back do
    image-updater em `gitops/helm/*/values.yaml` → sync, e que o commit do image-updater só corre o `gitops-validate`.
    Depois arquivar os repos antigos (descrição "ARCHIVED <data>: moved to ...").

**Fechar**
11. Workspace local: clonar em `<pasta-do-produto>/app.<produto>`, apagar os clones antigos.
12. Atualizar `CLAUDE.md` do workspace (layout §2, fluxo §3), `.github/profile/readme.md`, docs do produto, memória.

**Rollback** até ao passo 10: repor `repoURL`/`path` antigos nas Applications (o repo antigo continua intacto) e
reverter o PR do repo novo. Depois do arquivo: desarquivar e fazer o mesmo.

## Waves de migração (do menor risco ao maior)

| Wave | Produto | Repos de origem | Notas |
|---|---|---|---|
| 1 | `app.psirenatalima` | `ui.psirenatalima.com`, `gitops.psirenatalima.com` | piloto: só UI, sem BD, sem iac |
| 2 | `app.rapporthub` | `ui.rapporthub`, `gitops.rapporthub` | igual à 1; confirma a receita sem surpresas |
| 3 | `app.sara` | `api.ia.local-sara`, `ui.ia.local-sara`, `gitops.local-sara` | primeira com API + UI; ver o `docker-compose.yml` solto na pasta `sara/` (não versionado) |
| 4 | `app.teupadel` | `api.ia.teupadel.com`, `ui.ia.teupadel.com`, `api.ia.pose-estimation`, `gitops.teupadel.com` | o mais crítico: CNPG (`db-teupadel`), processor GPU (`processor/`, imagem amd64 só, deps acopladas no Renovate), testes com Postgres no CI, telemetria (nomes de serviço OTel mantêm-se). Fazer fora de horário de uso |
| 5 | serviços aplicacionais | `api.documents`+`gitops.documents` → `app.documents`; `api.matrix-agent`+`gitops.matrix-agent` → `app.matrix-agent` | `app.documents` é o único com Tier 2 hoje: `terraform/` → `iac/`, mesmo backend `key`; registar `app.documents` em `gitops-terraform-roles` (com `oidc_subject_prefix` do repo novo) depois de o criar |
| 6 | limpeza | — | tirar os repos antigos de `github_repos` e do mapa Tier 2 (PR Tier 1), remover `Prune=false` que sobrar, atualizar o golden path, `catalog-info` do `gitops.template` |

Cada wave é independente; dá para parar depois de qualquer uma com tudo a funcionar.

### Estado das waves

| Wave | Estado |
|---|---|
| 1 `app.psirenatalima` | **concluída 2026-11-28**: adotada sem restart, cadeia provada (imagem `6f03607`, write-back `5068a15`), repos antigos arquivados |
| 2 `app.rapporthub` | **concluída 2026-11-28**: idem (imagem `6142c3a`); chart em `gitops/helm/site` |
| 3 `app.sara` | **concluída 2026-11-28**: idem (api e ui `1325499`); TechDocs por pasta (`techdocs-ref: dir:./api` etc.) |
| 4 `app.teupadel` | repo criado e CI verde (testes da API com Postgres, testes do processor, gitops), render Helm e kustomize do CNPG idênticos; **cutover por fazer** (fora de horário de uso; PRs humanos abertos: api.ia.teupadel.com#41, api.ia.pose-estimation#43) |
| 5 documents, matrix-agent | por começar: o painel Esteira tem worktrees ativos sobre PRs destes repos; congelar primeiro |
| 6 limpeza | por começar |

Lições das waves 1–3:
- O `renovate.json` da raiz tem de desligar `gitops/**` exceto `helmv3` em `gitops/helm/**` (os repos gitops antigos
  tinham `enabledManagers: [helmv3]`); sem isso o manager `helm-values` disputa as tags com o image-updater.
- O token do image-updater e o `repo-creds` do Argo (`https://github.com/cmoreira-dev`) já cobrem repos novos; as
  variáveis `AWS_REGION`/`AWS_ROLE_ARN` vêm da org, só `PUBLISH_ENABLED` é do repo.
- Nos pushes para `main` os jobs `*-build` não correm (só em PR): o `docker build` só fica provado no PR do passo 10.

## Segurança (itens abertos, fora destas waves)

- Os pais gerados correm em `project: default` e as filhas em `homelab` (amplo): quem cria um repo `gitops.*` ou
  `app.*` na org consegue deployar qualquer coisa. Isto já é verdade hoje para `gitops.*`; o `app-repos` não alarga.
  Fecho: AppProjects por app (destinos só no namespace da app, sem kinds `Application`/`AppProject`), ou
  apps-in-any-namespace.
- A org deixa membros criarem repos (2 membros). Restringir a criação a owners fecha tanto o ponto acima como o
  pré-registo de nomes `app.*` no ECR.
- Branch protection + CODEOWNERS em `gitops/argocd/` dos `app.*`.

## Decisões em aberto

- **Nomes**: proposta sem TLD, como `app.convert-to-3d` (`app.teupadel`, `app.sara`, `app.rapporthub`, `app.psirenatalima`).
  O nome do repo vira o nome da Application-pai no Argo.
- **Histórico**: importar com histórico (proposto) ou snapshot limpo (mais simples, perde `git blame`).
- **Backstage**: fora do escopo (portal de plataforma), como os repos `gitops.*` de addons.
- **Ambientes/aprovação**: o `production` environment do Tier 2 passa a viver no repo do produto; quem aprova o apply
  continua a ser só tu.
