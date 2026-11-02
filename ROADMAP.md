# ROADMAP — plataforma `lab-data`

Documento único de planejamento e acompanhamento: **onde estamos, o que vem a seguir, o que cada camada entrega e o que já foi feito**. Atualize ao fim de cada sessão de trabalho.

A plataforma é construída **uma camada por vez**. Cada camada só começa depois que a anterior estiver pronta e o escopo dela for confirmado. A ordem e o conteúdo podem mudar; quando mudarem, atualize este arquivo e registre o motivo num ADR em [docs/decisoes/](docs/decisoes/README.md).

**Legenda:** ✅ concluída · ⏳ em andamento · 🟡 código pronto, falta aplicar · ⬜ não iniciada · ⛔ bloqueada · 💭 em avaliação

---

## 📍 Onde estamos

| | |
|---|---|
| **Camada atual** | 3 — Lakehouse (`data-lakehouse`) |
| **Situação** | ✅ Etapa 3a (bronze) aplicada na conta e aprovada no smoke test. Silver (3b) e gold (3c) ainda não ligadas |
| **Próximo passo** | Escolher a próxima etapa: ingestão (camada 4), que já pode gravar no bronze, ou silver (3b), que depende do ADR-0014 |
| **Última atualização** | 2026-11-02 |

## ⏭️ Próximos passos

1. [x] Camada 2: GitHub Actions com `terraform fmt -check`, `validate` e `test` no `data-foundation`.
2. [x] Camada 2: proteção da `main` no `data-foundation` (ruleset `proteger-main`).
3. [x] Camada 2: `pre-commit` com detector de segredos e padrão de qualidade nos 4 repositórios.
4. [ ] Camada 3: confirmar o escopo do `data-lakehouse` antes de começar.

## ⚠️ Pendências e bloqueios

| Item | Impacto | Decidir quando |
|---|---|---|
| Redshift Serverless bloqueado no plano gratuito ([ADR-0013](docs/decisoes/0013-regiao-us-east-2.md)) | Camada 8 (DW) | Ao chegar na camada 8: upgrade da conta ou gold no Athena |
| QuickSight bloqueado no plano gratuito | Camada 9 (Analytics) | Ao chegar na camada 9: upgrade da conta ou BI open source |
| Budget de US$ 10/mês | Pode ficar apertado com EC2 (camada 7) | Antes da camada 7 |
| OIDC bloqueado no plano gratuito (SCP nega `iam:*OpenIDConnectProvider*`) | Deploy automático pelo GitHub Actions (camada 12) sem credencial temporária | Quando o deploy automático for retomado: usuário IAM dedicado ao CI ou upgrade da conta |

---

## 🗺️ Camadas

| # | Camada | Repositório | Status |
|---|---|---|---|
| 0 | [Ambiente local e acesso à AWS](#camada-0--ambiente-local-e-acesso-à-aws) | — | ✅ |
| 1 | [Fundação](#camada-1--fundação) | `data-foundation` | ✅ |
| 2 | [Repositório e CI básico](#camada-2--repositório-e-ci-básico) | todos | ✅ |
| 3 | [Lakehouse](#camada-3--lakehouse) | `data-lakehouse` | ⏳ |
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

✅ **Concluída em 2026-11-02.** Repositórios: todos.

**Objetivo:** versionar cada repositório e validar automaticamente todo push.

**Entregáveis**
- [x] `git init` local nos 4 repositórios, branch `main` ([ADR-0010](docs/decisoes/0010-polyrepo.md))
- [x] Primeiro commit de cada repositório, no padrão Conventional Commits em português
- [x] Identidade Git pessoal separada da do trabalho (`includeIf` no `~/.gitconfig` + chave SSH própria)
- [x] Repositórios **públicos** na organização [lab-data-platform](https://github.com/lab-data-platform)
- [x] Proteção da `main` no `data-foundation`: ruleset `proteger-main` (só por PR, CI obrigatório, sem force push, sem apagar), testada com um push direto recusado
- [x] GitHub Actions no `data-foundation`: `terraform fmt -check`, `make validate` e `make test` em todo PR e push na `main` (PR #1)
- [x] Padrão de qualidade nos 4 repositórios: job `pre-commit` no CI e ruleset `proteger-main` exigindo os checks ([convenções](docs/convencoes.md#padrão-de-qualidade))
- [ ] Jobs por linguagem nos próximos repositórios, quando houver código (ruff para Python, `dbt parse`)
- [x] `pre-commit` com gitleaks, credenciais AWS, chave privada e verificações gerais; testado com um token falso bloqueado

**Pronto quando:** um PR com Terraform mal formatado falha no CI.

**Em aberto:** o CI usa credenciais da AWS (OIDC) já aqui ou só na camada 12.

---

### Camada 3 — Lakehouse

⏳ **Bronze no ar; silver e gold a ligar.** Repositório: `data-lakehouse`.

**Objetivo:** criar o armazenamento e o catálogo das camadas de dado, **uma de cada vez**: primeiro o bronze, depois silver e gold.

**Entregáveis**
**3a — Bronze** ✅ (2026-11-02)
- [x] Escopo confirmado: um bucket por camada, camada a camada (começando pelo bronze), Lake Formation só na camada 11, limite de 1 GB por consulta no Athena
- [x] Módulo `data-bucket` reutilizável; bucket do bronze (versionado) e `athena-results` (expira em 7 dias), com criptografia, sem acesso público, só TLS, lifecycle
- [x] Database do Glue `lab_data_dev_bronze`
- [x] Workgroup `lab-data-dev` (engine v3, resultados no bucket próprio, limite de 1 GB por consulta)
- [x] Contrato no SSM: 4 parâmetros em `/lab-data/dev/lakehouse/` ([contratos](docs/contratos.md#data-lakehouse)); consome o contrato do foundation
- [x] Nível 1: `terraform test` (9 casos, incluindo silver e gold ligadas)
- [x] Nível 2: stack aplicado no moto (19 recursos, 8 verificações), também no CI
- [x] `make apply` na conta real: 19 recursos (2 buckets, 1 database, 1 workgroup, 4 parâmetros)
- [x] Nível 3: `make smoke-test` passou (arquivo JSON no bronze, tabela no Glue, consulta pelo workgroup, limpeza completa)

**3b — Silver** ⬜ (antes da camada 5)
- [ ] Decidir o armazenamento: bucket comum ou S3 Tables ([ADR-0014](docs/decisoes/0014-s3-tables-na-silver.md))
- [ ] Ligar a silver em `layers` e aplicar

**3c — Gold** ⬜ (antes da camada 5)
- [ ] Ligar a gold em `layers` e aplicar

**Pronto quando:** cada camada ligada tem bucket, database e contrato, e o smoke test passa. Para o bronze: um arquivo gravado nele vira tabela consultável pelo workgroup da plataforma.

**Em aberto:** S3 Tables na silver ([ADR-0014](docs/decisoes/0014-s3-tables-na-silver.md)), a decidir antes da camada 5.

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
- [ ] Credenciais da AWS no GitHub Actions. ⚠️ O OIDC está bloqueado no plano gratuito (testado em 2026-11-02); alternativas: usuário IAM dedicado ao CI, com permissão mínima e secret num Environment protegido, ou upgrade da conta
- [ ] `terraform plan` comentado no PR e `apply` após o merge, com aprovação manual (Environment `dev`) e smoke test depois do apply
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
| 2026-11-02 | Repositórios publicados na organização [lab-data-platform](https://github.com/lab-data-platform), com commits no padrão Conventional Commits |
| 2026-11-02 | Primeiro CI (GitHub Actions) e proteção da `main` no `data-foundation`, feitos pelo usuário com acompanhamento |
| 2026-11-02 | ✅ Camada 2 concluída: pre-commit com gitleaks, job `pre-commit` no CI e `main` protegida nos 4 repositórios |
| 2026-11-02 | ✅ Etapa 3a concluída: bronze aplicado na conta (19 recursos) e aprovado no smoke test |
| 2026-11-02 | Deploy automático pelo GitHub Actions avaliado e adiado: o OIDC está bloqueado no plano gratuito |
| 2026-11-02 | Camada 3: escopo confirmado; o lakehouse passa a ser construído camada a camada, começando pelo bronze (código pronto e testado no moto). S3 Tables na silver registrado como ADR-0014 (proposto) |
