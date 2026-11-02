# Arquitetura da plataforma `lab-data`

Documento de referência da arquitetura. Descreve **o que** a plataforma é, **como** as camadas se encaixam e **quais** ferramentas implementam cada uma. As decisões e os motivos de cada escolha estão em [decisoes/](decisoes/README.md), e a ordem de construção e o andamento estão no [ROADMAP.md](../ROADMAP.md).

> Documento vivo: quando uma decisão mudar, atualize este arquivo e registre um ADR novo (o antigo fica como *Substituído*).

---

## 1. Objetivo

Construir, de ponta a ponta, uma **plataforma de dados** na AWS com práticas de mercado. A plataforma é a base comum (armazenamento, catálogo, orquestração, qualidade, segurança, custo) em que qualquer fonte de dados é plugada pelo mesmo caminho.

Ela entrega três capacidades de consumo:

| Capacidade | Para quem | Como |
|---|---|---|
| **Data Lakehouse** | Engenharia, analistas avançados, ciência de dados | SQL no Athena sobre tabelas Iceberg |
| **Data Warehouse** | Analistas e áreas de negócio | SQL no Redshift sobre o modelo dimensional |
| **Analytics** | Usuários de negócio | Dashboards no QuickSight |

**Primeiro caso de uso:** domínio **financeiro**, com dados públicos do Banco Central e da CVM. O detalhamento das fontes fica para a camada de ingestão.

---

## 2. Visão geral

```
                                 PLATAFORMA lab-data
 ┌──────────────────────────────────────────────────────────────────────────────────┐
 │                                                                                  │
 │  FONTES        INGESTÃO           LAKEHOUSE                   DW         CONSUMO │
 │                                                                                  │
 │  APIs   ──►  Step Functions ──►  S3 bronze ─► silver ─► gold ─► Redshift ─► QuickSight
 │  arquivos     + Lambda           (raw)      (Iceberg) (Iceberg) Serverless       │
 │                                       └── dbt-athena ──┘   └ dbt-redshift ┘      │
 │                                                              (via Spectrum)      │
 │                          Glue Data Catalog · Athena                              │
 │                                                                                  │
 │  ORQUESTRAÇÃO ─ Airflow (EC2) comanda o fluxo de ponta a ponta                   │
 │                                                                                  │
 │  TRANSVERSAIS ─ qualidade · observabilidade · governança/catálogo · segurança    │
 │                 IaC (Terraform) · CI/CD · FinOps                                 │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

### Fluxo de um dado

1. O **Airflow** agenda a execução da fonte e dispara a state machine de ingestão no **Step Functions**.
2. A **Lambda** extrai da fonte e grava o dado bruto no **bronze**, com os metadados de ingestão.
3. O **dbt-athena** transforma bronze → **silver** (limpo, tipado, deduplicado) → **gold** (entidades de negócio conformadas), em Iceberg.
4. O **dbt-redshift** lê a gold pelo **Redshift Spectrum** e materializa o **modelo dimensional** no Redshift.
5. O **QuickSight** consome os marts do DW. O lake continua consultável pelo Athena.
6. Testes, frescor e alertas rodam em cada passo, e o catálogo registra a linhagem de ponta a ponta.

---

## 3. Camadas

Cada camada tem um papel, as ferramentas que a implementam e um **contrato** com as camadas vizinhas. O contrato é o que permite trocar a ferramenta sem quebrar o resto.

### 3.1 Fundação

| | |
|---|---|
| Papel | Base de tudo: state do Terraform, convenções, tags, controle de custo |
| Ferramentas | Terraform (state remoto no S3 com `use_lockfile`), AWS Budgets |
| Repositório | `data-foundation` |
| Contrato | Prefixo `lab-data-<env>`, tags obrigatórias ([convencoes.md](convencoes.md)), tópico SNS de alertas publicado no SSM |

### 3.2 Rede e segurança base

| | |
|---|---|
| Papel | Rede e identidade compartilhadas pelos componentes que rodam dentro de VPC (EC2 do Airflow, Redshift) |
| Ferramentas | VPC default, security groups, IAM roles por componente, SSM Session Manager |
| Repositório | `data-foundation` (construída só antes do primeiro componente em VPC) |
| Princípios | Sem NAT Gateway (custo), sem porta exposta na internet, uma role por componente com o mínimo de permissão |

### 3.3 Lakehouse

| | |
|---|---|
| Papel | Armazenamento e catálogo de todas as camadas de dado |
| Ferramentas | S3 (um bucket por camada), Apache Iceberg, Glue Data Catalog, Athena (engine v3, workgroup com limite de bytes) |
| Repositório | `data-lakehouse` (infra do lake e, na camada 5, o projeto dbt do lake) |
| Contrato | Database do Glue por camada (`lab_data_<env>_bronze/silver/gold`); toda tabela registrada no catálogo |

| Camada de dado | Conteúdo | Formato | Quem escreve |
|---|---|---|---|
| bronze | Dado como veio da fonte, mais os metadados de ingestão | JSON/CSV comprimido | Ingestão |
| silver | Limpo, tipado, deduplicado, no modelo da fonte | Iceberg | dbt-athena |
| gold | Entidades de negócio conformadas, prontas para consumo | Iceberg | dbt-athena |

### 3.4 Ingestão

| | |
|---|---|
| Papel | Trazer o dado da fonte para o bronze, de forma idempotente e reprocessável |
| Ferramentas | Step Functions (fluxo com retry e validação) + Lambda (extração) |
| Repositório | `data-ingestion` |
| Contrato | Grava em `s3://<bronze>/<fonte>/<entidade>/ingestion_date=YYYY-MM-DD/` e a tabela bronze fica registrada no catálogo |
| Padrão | Um módulo Terraform reutilizável por fonte: uma fonte nova significa uma instância nova do módulo, não código novo de infra |

### 3.5 Transformação

| | |
|---|---|
| Papel | Transformar o dado entre as camadas, com testes, documentação e linhagem |
| Ferramentas | dbt, em **dois projetos**: `dbt-athena` (bronze → silver → gold) e `dbt-redshift` (gold → modelo dimensional) |
| Contrato | Modelos organizados por domínio; todo modelo com testes básicos (`unique`, `not_null`) e descrição |

### 3.6 Data Warehouse

| | |
|---|---|
| Papel | Modelo dimensional (star schema, SCD, chaves surrogate) e marts de consumo |
| Ferramentas | Redshift Serverless (base de 8 RPU, com limite de uso) |
| Ponte com o lake | **Redshift Spectrum**: um *external schema* aponta para o Glue Catalog, e o dbt-redshift lê a gold como `source`. Não usamos `COPY` porque ele ignora os snapshots do Iceberg |
| Divisão com o lake | A gold guarda **entidades de negócio**; o DW guarda **a modelagem dimensional** e os marts do BI |

### 3.7 Orquestração

| | |
|---|---|
| Papel | Decidir quando e em que ordem tudo roda, com retry, histórico e alerta de falha |
| Ferramentas | **Airflow em EC2** como orquestrador principal (DAG de ponta a ponta, com o dbt via Astronomer Cosmos) e **Step Functions** no fluxo serverless de cada ingestão, disparado pelo Airflow |
| Acesso | UI do Airflow por port forwarding via SSM (`localhost:8080`), sem porta pública |
| Custo | t3.medium com agenda de liga/desliga |

### 3.8 Analytics

| | |
|---|---|
| Papel | Dashboards e autoatendimento para o negócio |
| Ferramentas | Amazon QuickSight, conectado ao Redshift (marts) e opcionalmente ao Athena |

### 3.9 Qualidade e observabilidade

| | |
|---|---|
| Papel | Garantir que o dado está correto e atualizado, e que falhas são percebidas |
| Ferramentas | dbt tests e `source freshness`, alertas do Airflow, CloudWatch (logs, métricas, alarmes), SNS para notificação |

### 3.10 Governança e catálogo

| | |
|---|---|
| Papel | Um lugar único para descobrir dados: catálogo, linhagem de ponta a ponta (fonte → lake → DW → dashboard), dono, qualidade, glossário |
| Ferramentas | Começa com o **dbt docs** publicado no S3; depois **OpenMetadata** (open source) ou **SageMaker Unified Studio** (gerenciado pela AWS) |
| Observação | Não existe um front próprio: o catálogo de metadados cumpre esse papel ([ADR-0008](decisoes/0008-governanca-catalogo.md)) |

### 3.11 CI/CD

| | |
|---|---|
| Papel | Validar e entregar mudanças com segurança |
| Ferramentas | GitHub Actions: `terraform fmt/validate/plan`, lint e testes de Python, `dbt build` em ambiente de CI |

---

## 4. Repositórios

A plataforma é dividida em **um repositório Git por componente** ([ADR-0010](decisoes/0010-polyrepo.md)). Cada repositório tem o próprio Terraform, o próprio state, os próprios testes e a própria documentação, e pode ser aplicado e destruído sozinho. A pasta `lab-data-engineering` é só o **workspace** que reúne todos eles e não é versionada.

```
lab-data-engineering/            ← workspace (não versionado)
├── data-platform/     (git)     ← hub: visão do todo, convenções, contratos, ADRs globais
├── data-foundation/   (git)     ← state remoto, budget, alertas, rede base
├── data-lakehouse/    (git)     ← S3, Glue, Athena (+ dbt do lake na camada 5)
└── data-ingestion/    (git)     ← Lambdas + Step Functions por fonte
```

Os repositórios das camadas seguintes (orquestração, warehouse, analytics, governança) são criados quando cada camada começar.

**Ordem de dependência:** `data-foundation` → `data-lakehouse` → `data-ingestion`. Um repositório só depende dos **contratos** dos anteriores, nunca do código ou do state deles.

### Estrutura padrão de um repositório

```
data-<componente>/
├── infra/          # Terraform do componente (state: <componente>/<env>/terraform.tfstate)
│   └── tests/      # terraform test com provider simulado
├── src/            # código (quando houver), organizado por domínio
├── tests/          # testes do código
├── local/          # ambiente local em Docker (quando o componente puder ser emulado)
├── docs/           # decisões específicas do componente
├── Makefile        # alvos padrão: help, fmt, validate, test, init, plan, apply, destroy
├── CLAUDE.md
└── README.md
```

**Domínios:** dentro de cada repositório, o código é organizado por domínio (ex.: `data-ingestion/src/financeiro/bcb/`). Um domínio novo segue a mesma estrutura sem criar repositório novo.

---

## 5. Contratos entre repositórios

Cada repositório publica no **SSM Parameter Store** o que os outros precisam saber dele, num caminho padrão ([ADR-0011](decisoes/0011-contratos-ssm.md)):

```
/lab-data/<env>/<componente>/<chave>
```

Quem consome lê o parâmetro, sem acessar o state nem o código do outro repositório:

```hcl
data "aws_ssm_parameter" "alerts_topic_arn" {
  name = "/lab-data/dev/foundation/alerts_topic_arn"
}
```

O catálogo de todos os parâmetros está em [contratos.md](contratos.md). Mudar ou remover um parâmetro é **mudança de contrato**: atualize o catálogo e avise os consumidores.

---

## 6. Componentes plugáveis

Cada capacidade pode ter **mais de uma implementação**, para trocar de ferramenta sem refazer as camadas vizinhas ou para manter duas lado a lado.

| Capacidade | Implementação padrão | Alternativas previstas |
|---|---|---|
| Orquestração | Airflow (EC2) | Step Functions de ponta a ponta, MWAA |
| Data Warehouse | Redshift Serverless | Gold no Athena (custo quase zero) |
| Analytics | QuickSight | Metabase ou Superset (Docker) |
| Governança | dbt docs → OpenMetadata | SageMaker Unified Studio, DataHub |
| Processamento | dbt (SQL) | Glue Spark (PySpark) para cargas pesadas |

**Como isso funciona com vários repositórios:**

1. **Contratos estáveis.** As implementações trocam sempre as mesmas coisas: o layout no S3, as tabelas no Glue Catalog e os parâmetros no SSM. O orquestrador só chama essas peças; ele não contém regra de negócio.
2. **Uma implementação alternativa é um repositório ou um módulo novo**, que publica o mesmo contrato da implementação padrão (ex.: `data-orchestration-airflow` e `data-orchestration-stepfunctions`).
3. **Ligar e desligar é aplicar ou destruir** o repositório da implementação. Um componente que não foi aplicado não gera custo.

---

## 7. Testes locais

Todo repositório precisa ser testável na máquina local, em três níveis ([ADR-0012](decisoes/0012-testes-locais.md)):

| Nível | Como | AWS |
|---|---|---|
| **1. Unitário** | `terraform test` com provider simulado; pytest com moto para o código | Não |
| **2. Integração local** | Emulador da AWS (moto em modo servidor) e ferramentas reais em Docker, com `make local-up` | Não |
| **3. Integração real** | `make plan` e `make apply` no ambiente `dev` | Sim |

| Repositório | Nível 1 | Nível 2 | Limite |
|---|---|---|---|
| data-foundation | ✅ `terraform test` | — | Budgets e S3 Control não são emulados pelo moto |
| data-lakehouse | `terraform test` | S3 no moto; dbt-duckdb lendo os mesmos arquivos (camada 5) | Glue e Athena só no nível 3 |
| data-ingestion | `terraform test` + pytest com moto | Lambda, S3 e Step Functions no moto | Execução real da Lambda só no nível 3 |

---

## 8. Assets, eventos e governança

A plataforma é **orientada a assets** ([ADR-0015](decisoes/0015-assets-e-eventos.md)): cada dado tem nome (`<camada>.<fonte>.<entidade>`), dono e contrato, e publica o evento `AssetUpdated` no barramento `lab-data-<env>-assets` quando fica pronto. Quem depende dele reage ao evento.

```
Step Functions (ingestão) ──AssetUpdated──► EventBridge lab-data-<env>-assets ──regra──► consumidores
                                                    │ archive (90 dias, replay)           (dbt, Airflow via SQS)
```

- **Contrato de dados por asset** no `assets.yml` de cada repositório ([ADR-0016](decisoes/0016-contrato-de-dados-por-asset.md)).
- **Políticas** de classificação, colunas técnicas, quarentena, retenção, segredos e evolução de schema: [governanca.md](governanca.md).
- **Auditoria** da conta com CloudTrail ([ADR-0017](decisoes/0017-auditoria-cloudtrail.md)).
- Detalhes do evento e de como reagir a ele: [eventos.md](eventos.md).

---

## 9. Ambientes

| Ambiente | Uso | Estado |
|---|---|---|
| `dev` | Desenvolvimento e o lab em si | Único ambiente por enquanto |
| `prd` | Previsto na variável `environment` | Só se o projeto evoluir para isso |

Cada repositório tem um state por ambiente (`<componente>/<env>/terraform.tfstate`), todos no bucket `lab-data-tfstate-<account_id>` criado pelo `data-foundation`.

---

## 10. Custo estimado

Valores aproximados por mês, no ambiente `dev`, com uso de laboratório.

| Componente | Custo | Como controlar |
|---|---|---|
| S3, Glue Catalog, Lambda, Step Functions | centavos | Lifecycle no S3 |
| Athena | centavos (US$ 5 por TB escaneado) | Limite de bytes por consulta no workgroup, Iceberg particionado |
| EC2 do Airflow (t3.medium) | US$ 10 a 30 | Agenda de liga/desliga |
| Redshift Serverless | depende do uso (8 RPU a cerca de US$ 3/hora ativa) | Base de 8 RPU, limite de RPU-hora por dia, trial de crédito |
| QuickSight | trial de 30 dias, depois cerca de US$ 24 por autor | Avaliar ao fim do trial |
| OpenMetadata | US$ 30 a 60 (EC2 com 8 GB) | Ligar só na camada de governança |

O budget atual (US$ 10/mês) cobre as camadas iniciais. Antes de criar a EC2 e o Redshift, revisar o limite (ver [ROADMAP.md](../ROADMAP.md)).
