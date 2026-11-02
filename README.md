# data-platform

Hub da plataforma de dados **`lab-data`** na AWS: visão do todo, roadmap, convenções, contratos entre repositórios e decisões de arquitetura. Este repositório não cria infraestrutura; cada componente da plataforma vive no seu próprio repositório.

## A plataforma

Plataforma de dados construída do zero, **camada a camada**, com práticas de mercado: infraestrutura como código, arquitetura medalhão, formato de tabela aberto, testes locais, governança e controle de custo.

| Capacidade | Tecnologia |
|---|---|
| **Data Lakehouse** | S3 + Apache Iceberg + Glue Data Catalog + Athena |
| **Data Warehouse** | Redshift Serverless, alimentado pelo lake via Spectrum |
| **Analytics** | Amazon QuickSight |
| **Orquestração** | Airflow em EC2 + Step Functions na ingestão |
| **Transformação** | dbt (Athena e Redshift) |

Primeiro caso de uso: **domínio financeiro** (dados públicos do Banco Central e da CVM).

```
 FONTES       INGESTÃO            LAKEHOUSE                          DW               CONSUMO

 APIs ──► Step Functions ──► S3 bronze ─► silver ─► gold ──► Redshift Serverless ──► QuickSight
          + Lambda           (bruto)     (Iceberg)  (Iceberg)   (modelo dimensional)
                                  └── dbt-athena ───┘  └─ dbt-redshift via Spectrum ─┘
                             Glue Data Catalog · Athena

 ORQUESTRAÇÃO   Airflow (EC2) de ponta a ponta; Step Functions em cada ingestão
 TRANSVERSAIS   qualidade · observabilidade · governança · segurança · Terraform · CI/CD · FinOps
```

## Repositórios

Um repositório Git por componente, todos dentro do workspace `lab-data-engineering/` ([ADR-0010](docs/decisoes/0010-polyrepo.md)).

| Repositório | Papel | Camadas | Status |
|---|---|---|---|
| [data-platform](https://github.com/lab-data-platform/data-platform) | Hub: arquitetura, convenções, contratos, ADRs | — | ✅ |
| [data-foundation](https://github.com/lab-data-platform/data-foundation) | State remoto, budget, alertas, rede base | 1, 6 | ✅ camada 1 aplicada |
| [data-lakehouse](https://github.com/lab-data-platform/data-lakehouse) | S3, Glue, Athena e o dbt do lake | 3, 5 | ⏳ bronze aplicado |
| [data-ingestion](https://github.com/lab-data-platform/data-ingestion) | Lambdas e Step Functions por fonte | 4 | ⬜ |

Os repositórios das camadas 7 em diante (orquestração, warehouse, analytics, governança) são criados quando cada camada começar.

**Dependência:** `data-foundation` → `data-lakehouse` → `data-ingestion`. Os repositórios só conversam pelos [contratos no SSM](docs/contratos.md).

## Roadmap

📍 **Onde estamos, próximos passos e o plano de cada camada: [ROADMAP.md](ROADMAP.md).** Para acompanhar uma plataforma sua, use o modelo [ROADMAP.example.md](ROADMAP.example.md).

## Documentação

| Documento | Conteúdo |
|---|---|
| [ROADMAP.md](ROADMAP.md) | **Planejamento e acompanhamento:** onde estamos, próximos passos, entregáveis e critério de pronto de cada camada, histórico |
| [docs/arquitetura.md](docs/arquitetura.md) | Camadas, repositórios, contratos, componentes plugáveis, testes locais, custo |
| [docs/convencoes.md](docs/convencoes.md) | Repositórios, nomes, tags, camadas de dados, Terraform |
| [docs/contratos.md](docs/contratos.md) | Parâmetros SSM que cada repositório publica |
| [docs/governanca.md](docs/governanca.md) | Políticas do dado: contrato por asset, classificação, colunas técnicas, quarentena, retenção, segredos, evolução de schema, auditoria |
| [docs/eventos.md](docs/eventos.md) | Assets, nome dos assets e o evento `AssetUpdated` |
| [docs/decisoes/](docs/decisoes/README.md) | ADRs: cada decisão e o motivo dela |
| [docs/setup/ambiente-local.md](docs/setup/ambiente-local.md) | Ferramentas locais e acesso à AWS pela linha de comando |

## Testes locais

Todo repositório é testável na máquina local, em três níveis ([ADR-0012](docs/decisoes/0012-testes-locais.md)):

| Nível | Comando | AWS |
|---|---|---|
| 1. Unitário | `make test` | Não |
| 2. Integração local (moto em Docker) | `make local-up` / `make local-test` | Não |
| 3. Integração real | `make plan` / `make apply` | Sim |

## Qualidade

| Verificação | Onde roda | O que cobre |
|---|---|---|
| Pré-commit | Na sua máquina, a cada `git commit` | Segredos (gitleaks, credenciais AWS, chaves privadas), YAML, espaços e fim de arquivo, commit direto na `main` |
| CI | GitHub Actions ([ci.yml](.github/workflows/ci.yml)), em todo PR e push na `main` | O mesmo pre-commit, em todos os arquivos; obrigatório para o merge |

```bash
pre-commit install            # uma vez por clone: ativa os hooks
pre-commit run --all-files    # roda todos os hooks manualmente
```

## Princípios

- **Uma camada por vez**, com escopo confirmado e critério de pronto.
- **Um repositório por componente**, e repositórios que só conversam por contratos.
- **Tudo testável localmente**, até onde o emulador permite, e com o limite documentado.
- **Decisão registrada:** toda escolha vira ADR, e mudar de ideia é permitido.
- **Nenhum segredo no Git** e **custo sob controle**.
