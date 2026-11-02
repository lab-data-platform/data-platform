# ROADMAP — plataforma `lab-data`

Documento único de planejamento e acompanhamento: **onde estamos, o que vem a seguir, o que cada camada entrega e o que já foi feito**. Atualize ao fim de cada sessão de trabalho.

A plataforma é construída **uma camada por vez**. Cada camada só começa depois que a anterior estiver pronta e o escopo dela for confirmado. A ordem e o conteúdo podem mudar; quando mudarem, atualize este arquivo e registre o motivo num ADR em [docs/decisoes/](docs/decisoes/README.md).

**Legenda:** ✅ concluída · ⏳ em andamento · 🟡 código pronto, falta aplicar · ⬜ não iniciada · ⛔ bloqueada · 💭 em avaliação

---

## 📍 Onde estamos

| | |
|---|---|
| **Camada atual** | 2 — Repositório e CI básico (todos os repositórios) |
| **Situação** | Camada 1 concluída. Os 4 repositórios têm `git init`, mas nenhum commit ainda |
| **Próximo passo** | Primeiro commit de cada repositório e decidir a hospedagem no GitHub |
| **Última atualização** | 2026-11-02 |

## ⏭️ Próximos passos

1. [ ] Camada 2: primeiro commit de cada repositório.
2. [ ] Camada 2: decidir a hospedagem (GitHub público ou privado) e configurar o CI básico.
3. [ ] Camada 3: confirmar o escopo do `data-lakehouse` antes de começar.

## ⚠️ Pendências e bloqueios

| Item | Impacto | Decidir quando |
|---|---|---|
| Redshift Serverless bloqueado no plano gratuito ([ADR-0013](docs/decisoes/0013-regiao-us-east-2.md)) | Camada 8 (DW) | Ao chegar na camada 8: upgrade da conta ou gold no Athena |
| QuickSight bloqueado no plano gratuito | Camada 9 (Analytics) | Ao chegar na camada 9: upgrade da conta ou BI open source |
| Budget de US$ 10/mês | Pode ficar apertado com EC2 (camada 7) | Antes da camada 7 |

---

## 🗺️ Camadas

| # | Camada | Repositório | Status |
|---|---|---|---|
| 0 | [Ambiente local e acesso à AWS](#camada-0--ambiente-local-e-acesso-à-aws) | — | ✅ |
| 1 | [Fundação](#camada-1--fundação) | `data-foundation` | ✅ |
| 2 | [Repositório e CI básico](#camada-2--repositório-e-ci-básico) | todos | ⏳ |
| 3 | [Lakehouse](#camada-3--lakehouse) | `data-lakehouse` | ⬜ |
| 4 | [Ingestão](#camada-4--ingestão) | `data-ingestion` | ⬜ |
| 5 | [Transformação no lake](#camada-5--transformação-no-lake) | `data-lakehouse` | ⬜ |
| 6 | [Rede e segurança base](#camada-6--rede-e-segurança-base) | `data-foundation` | ⬜ |
| 7 | [Orquestração com Airflow](#camada-7--orquestração-com-airflow) | a criar | ⬜ |
| 8 | [Data Warehouse](#camada-8--data-warehouse) | a criar | ⛔ |
| 9 | [Analytics](#camada-9--analytics) | a criar | ⛔ |
| 10 | [Qualidade e observabilidade](#camada-10--qualidade-e-observabilidade) | todos | ⬜ |
| 11 | [Governança e catálogo](#camada-11--governança-e-catálogo) | a criar | ⬜ |
| 12 | [CI/CD completo](#camada-12--cicd-completo) | todos | ⬜ |
| — | [Implementações alternativas](#implementações-alternativas) | — | 💭 |

**Foco atual: camadas 0 a 4** (até a ingestão). Só os repositórios dessas camadas existem; os das camadas seguintes são criados quando cada uma começar.

**Por que nessa ordem:** cada camada usa o que a anterior entregou. O lake vem antes da ingestão, porque a ingestão grava nele. A rede só entra antes do Airflow e do Redshift, que são os primeiros componentes que rodam dentro de VPC. O CI básico entra cedo, para todo código novo já nascer validado.

---

### Camada 0 — Ambiente local e acesso à AWS

✅ **Concluída.** Guia: [docs/setup/ambiente-local.md](docs/setup/ambiente-local.md).

- [x] AWS CLI v2 e Terraform instalados no WSL
- [x] MFA no root
- [x] Usuário IAM `lab-admin` com access key no profile `lab-data` ([ADR-0007](docs/decisoes/0007-acesso-cli-iam-access-key.md))
- [x] Rotação da chave que apareceu no chat
- [x] `aws sts get-caller-identity` retorna `user/lab-admin`
- [x] Região do profile ajustada para `us-east-2` ([ADR-0013](docs/decisoes/0013-regiao-us-east-2.md))

---

### Camada 1 — Fundação

✅ **Concluída em 2026-11-02.** Repositório: `data-foundation`.

**Entregáveis**
- [x] `bootstrap/`: bucket `lab-data-tfstate-<account_id>`, que guarda o state de todos os repositórios (versionado, criptografado, só TLS, `prevent_destroy`)
- [x] `infra/`: backend S3 com `use_lockfile`, tags automáticas, budget `lab-data-dev-mensal` e tópico SNS de alertas `lab-data-dev-alerts`
- [x] Contrato no SSM: `/lab-data/dev/foundation/*` ([docs/contratos.md](docs/contratos.md))
- [x] `terraform test` com provider simulado (9 casos) e `Makefile`

**Pronto quando**
- [x] `make bootstrap-apply` criou o bucket de state
- [x] `make init` usa o backend remoto
- [x] `make apply` criou o budget, o tópico de alertas e os parâmetros do contrato
- [x] A assinatura de e-mail do tópico foi confirmada

**Em aberto:** valor do budget. US$ 10 serve até a camada 5; antes das camadas 7 e 8, avaliar US$ 25–30.

---

### Camada 2 — Repositório e CI básico

⏳ **Em andamento.** Repositórios: todos.

**Objetivo:** versionar cada repositório e validar automaticamente todo push.

**Entregáveis**
- [x] `git init` local nos 4 repositórios, branch `main` ([ADR-0010](docs/decisoes/0010-polyrepo.md))
- [ ] Primeiro commit de cada repositório
- [ ] Repositórios no GitHub e proteção da branch `main`
- [ ] GitHub Actions: `terraform fmt -check`, `validate` e `test` e, conforme o código existir, lint de Python (ruff) e `dbt parse`
- [ ] `pre-commit` local com os mesmos checks, mais um detector de segredos (ex.: gitleaks)

**Pronto quando:** um PR com Terraform mal formatado falha no CI.

**Em aberto:** repositório público ou privado; o CI usa credenciais da AWS (OIDC) já aqui ou só na camada 12.

---

### Camada 3 — Lakehouse

⬜ **Não iniciada.** Repositório: `data-lakehouse`.

**Objetivo:** criar o armazenamento e o catálogo de todas as camadas de dado.

**Entregáveis**
- [ ] Escopo confirmado
- [ ] Buckets S3 `bronze`, `silver`, `gold` e `athena-results`: criptografia, bloqueio de acesso público, só TLS, lifecycle
- [ ] Databases do Glue: `lab_data_dev_bronze`, `lab_data_dev_silver`, `lab_data_dev_gold`
- [ ] Workgroup do Athena (engine v3, resultados no bucket próprio, limite de bytes por consulta)
- [ ] Contrato no SSM: `/lab-data/dev/lakehouse/*` (buckets, databases, workgroup)
- [ ] Testes: `terraform test` (nível 1), S3 no moto (nível 2) e uma tabela Iceberg de exemplo criada e consultada pelo Athena (nível 3)

**Pronto quando:** dá para criar, consultar e apagar uma tabela Iceberg na silver usando só o workgroup da plataforma.

**Em aberto:** Lake Formation agora ou só na camada 11; um bucket por camada ou um bucket com prefixos.

---

### Camada 4 — Ingestão

⬜ **Não iniciada.** Repositório: `data-ingestion`.

**Objetivo:** trazer as fontes do domínio financeiro para o bronze.

**Entregáveis**
- [ ] Escopo confirmado e **fontes definidas** (é aqui que estudamos o dado: APIs, formato, volume, frequência, histórico)
- [ ] Módulo Terraform reutilizável `infra/modules/source/`: Lambda, role IAM, state machine e tabela bronze. Uma fonte nova é uma instância nova do módulo
- [ ] Consumo dos contratos do `data-foundation` (alertas) e do `data-lakehouse` (bucket e database do bronze)
- [ ] State machine no Step Functions por fonte: extrair → validar → registrar, com retry e alerta de falha
- [ ] Código das Lambdas em `src/financeiro/<fonte>/`
- [ ] Testes: pytest com moto (nível 1), Lambda, S3 e Step Functions no moto em Docker (nível 2) e execução real no `dev` (nível 3)
- [ ] Carga histórica (backfill) e carga incremental, idempotentes

**Pronto quando:** cada fonte tem dados no bronze, consultáveis pelo Athena, e uma reexecução do mesmo dia não duplica nada.

**Em aberto:** quais fontes e entidades entram primeiro; agendamento provisório (EventBridge) até o Airflow existir.

---

### Camada 5 — Transformação no lake

⬜ **Não iniciada.** Repositório: `data-lakehouse` (pasta `dbt/`).

**Objetivo:** transformar bronze → silver → gold com dbt.

**Entregáveis**
- [ ] Projeto dbt com adapter Athena e um target local com dbt-duckdb, para testar sem AWS
- [ ] Silver: modelos incrementais em Iceberg, deduplicados e tipados
- [ ] Gold: entidades de negócio conformadas do domínio financeiro
- [ ] Testes (`unique`, `not_null`, `relationships`, `accepted_values`) e descrição de cada modelo
- [ ] `dbt docs` gerado localmente

**Pronto quando:** `dbt build` roda do zero e de forma incremental, com todos os testes passando.

**Em aberto:** onde o dbt roda antes do Airflow (local ou um container); se algum passo pesado vai para o Glue Spark.

---

### Camada 6 — Rede e segurança base

⬜ **Não iniciada.** Repositório: `data-foundation`.

**Objetivo:** preparar a rede e as identidades para os componentes que rodam em VPC.

**Entregáveis**
- [ ] Rede base sobre a VPC default, publicada no contrato (`vpc_id`, subnets)
- [ ] Security groups do Airflow e do Redshift, sem entrada pública
- [ ] VPC endpoints gratuitos (gateway do S3) e acesso via SSM Session Manager
- [ ] Uma role IAM por componente, com o mínimo de permissão

**Pronto quando:** uma EC2 de teste é acessível por SSM sem nenhuma porta aberta e alcança o S3.

---

### Camada 7 — Orquestração com Airflow

⬜ **Não iniciada.** Repositório: a criar (ex.: `data-orchestration`).

**Objetivo:** um orquestrador comandando o fluxo de ponta a ponta.

**Entregáveis**
- [ ] EC2 t3.medium, role e agenda de liga/desliga
- [ ] Airflow 3 em Docker Compose na EC2, com as DAGs no próprio repositório; o mesmo compose roda localmente
- [ ] DAG de ponta a ponta: dispara o Step Functions da ingestão → `dbt build` do lake (Cosmos) → testes → alerta
- [ ] Deploy das DAGs (sync a partir do repositório)
- [ ] Acesso à UI por port forwarding via SSM

**Pronto quando:** a DAG diária roda sozinha, e uma falha gera alerta.

**Em aberto:** banco de metadados do Airflow (Postgres no próprio Docker ou RDS); como fazer o deploy das DAGs.

---

### Camada 8 — Data Warehouse

⛔ **Bloqueada no plano gratuito.** Repositório: a criar (ex.: `data-warehouse`).

**Objetivo:** o modelo dimensional num DW separado ([ADR-0003](docs/decisoes/0003-data-warehouse-redshift-serverless.md)).

**Entregáveis**
- [ ] Namespace e workgroup do Redshift Serverless (8 RPU base, limite de uso) e role para o Spectrum
- [ ] External schema apontando para a gold no Glue Catalog
- [ ] Projeto dbt com adapter Redshift: staging via Spectrum → dimensões e fatos → marts
- [ ] Task do dbt do DW na DAG do Airflow

**Pronto quando:** os marts do domínio financeiro estão no Redshift, atualizados pela DAG diária.

**Em aberto:** ⚠️ o Redshift Serverless está bloqueado no plano gratuito ([ADR-0013](docs/decisoes/0013-regiao-us-east-2.md)): decidir entre o upgrade da conta e a gold no Athena como DW. Também em aberto: estratégia de SCD nas dimensões; revisar o budget antes de criar.

---

### Camada 9 — Analytics

⛔ **Bloqueada no plano gratuito.** Repositório: a criar.

**Objetivo:** dashboards para o negócio.

**Entregáveis**
- [ ] Assinatura do QuickSight (trial) e conexão com o DW
- [ ] Datasets e dashboards do domínio financeiro, versionados quando possível

**Pronto quando:** um dashboard mostra dados atualizados pela DAG do dia.

**Em aberto:** ⚠️ o QuickSight está bloqueado no plano gratuito ([ADR-0013](docs/decisoes/0013-regiao-us-east-2.md)): decidir entre o upgrade da conta e um BI open source ([implementações alternativas](#implementações-alternativas)).

---

### Camada 10 — Qualidade e observabilidade

⬜ **Não iniciada.** Repositórios: todos.

**Objetivo:** saber quando o dado está errado ou atrasado, antes do usuário.

**Entregáveis**
- [ ] `dbt source freshness` e testes de qualidade de negócio
- [ ] Alarmes no CloudWatch (falha de Lambda, Step Functions e Airflow) publicando no tópico `lab-data-dev-alerts`
- [ ] Dashboard operacional no CloudWatch

**Pronto quando:** atraso ou falha em qualquer camada gera alerta por e-mail.

---

### Camada 11 — Governança e catálogo

⬜ **Não iniciada.** Repositório: a criar.

**Objetivo:** um lugar único para descobrir e entender os dados ([ADR-0008](docs/decisoes/0008-governanca-catalogo.md)).

**Entregáveis**
- [ ] `dbt docs` dos dois projetos publicado como site estático
- [ ] Catálogo de metadados (OpenMetadata ou SageMaker Unified Studio), com conectores para Glue, Athena, DW, dbt e Airflow
- [ ] Linhagem de ponta a ponta, donos e glossário do domínio financeiro
- [ ] Lake Formation, se ainda não tiver entrado na camada 3

**Em aberto:** qual catálogo usar.

---

### Camada 12 — CI/CD completo

⬜ **Não iniciada.** Repositórios: todos.

**Objetivo:** entregar mudanças sem passo manual.

**Entregáveis**
- [ ] Credenciais da AWS no GitHub Actions via OIDC, sem access key
- [ ] `terraform plan` comentado no PR e `apply` após o merge
- [ ] `dbt build` em ambiente de CI (slim CI com `state:modified`)
- [ ] Deploy automático das DAGs e das Lambdas

---

### Implementações alternativas

💭 Opções que podem entrar **ao lado** da implementação padrão, para comparar ou para o usuário escolher ([docs/arquitetura.md §6](docs/arquitetura.md#6-componentes-plugáveis)). Nenhuma está confirmada.

| Capacidade | Alternativa | Quando faria sentido |
|---|---|---|
| Orquestração | Step Functions de ponta a ponta | Comparar orquestração serverless com o Airflow |
| Orquestração | MWAA | Demonstração curta do Airflow gerenciado |
| Data Warehouse | Gold no Athena como DW | Custo quase zero; funciona no plano gratuito |
| Analytics | Metabase ou Superset | Funciona no plano gratuito |
| Processamento | Glue Spark (PySpark) | Praticar Spark ou lidar com volume alto |
| Governança | DataHub | Comparar com o OpenMetadata |

---

## 📜 Histórico

| Data | O que aconteceu |
|---|---|
| 2026-10-30 | Início do projeto. Escolhas iniciais: dados públicos BCB/CVM, Athena + Iceberg, QuickSight, Step Functions |
| 2026-10-31 | Acesso à AWS pelo CLI com o usuário IAM `lab-admin` (conta sem SSO e sem login de usuário IAM no console) |
| 2026-11-02 | Arquitetura revisada: DW separado em Redshift Serverless, Airflow em EC2 + Step Functions, dbt em dois projetos ([ADR-0003](docs/decisoes/0003-data-warehouse-redshift-serverless.md), [ADR-0004](docs/decisoes/0004-orquestracao-airflow-ec2-step-functions.md), [ADR-0005](docs/decisoes/0005-transformacao-dbt.md)) |
| 2026-11-02 | Projeto dividido em repositórios (`data-platform`, `data-foundation`, `data-lakehouse`, `data-ingestion`), com contratos via SSM e testes locais em três níveis ([ADR-0010](docs/decisoes/0010-polyrepo.md) a [0012](docs/decisoes/0012-testes-locais.md)) |
| 2026-11-02 | Descoberta: a conta está no plano gratuito, e só `us-east-2` está liberada; região do projeto trocada ([ADR-0013](docs/decisoes/0013-regiao-us-east-2.md)) |
| 2026-11-02 | Camada 1 aplicada na AWS: bucket de state, budget, tópico de alertas e contrato SSM |
| 2026-11-02 | ✅ Camada 1 concluída: assinatura de e-mail do tópico de alertas confirmada |
| 2026-11-02 | Roadmap unificado: plano e acompanhamento num só arquivo, o `ROADMAP.md` |
| 2026-11-02 | Padrão "Recursos na AWS" nos READMEs: serviço, recurso, finalidade, região, custo, código e link do console |
