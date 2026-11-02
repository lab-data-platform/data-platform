# Catálogo de contratos

Parâmetros que cada repositório publica no **SSM Parameter Store** para os outros consumirem ([ADR-0011](decisoes/0011-contratos-ssm.md)).

**Caminho:** `/lab-data/<env>/<componente>/<chave>`

**Regras**
- Só entra aqui o que **outro repositório precisa**. Detalhe interno fica no output do Terraform.
- Renomear ou remover um parâmetro é **mudança de contrato**: atualize este catálogo, o teste do contrato no repositório produtor e os consumidores, nessa ordem.
- Adicionar um parâmetro novo não quebra ninguém.

## data-foundation

Publicado por `data-foundation/infra/contract.tf`.

| Parâmetro | Conteúdo | Consumidores |
|---|---|---|
| `/lab-data/<env>/foundation/name_prefix` | Prefixo dos recursos (`lab-data-<env>`) | todos |
| `/lab-data/<env>/foundation/account_id` | ID da conta AWS | todos |
| `/lab-data/<env>/foundation/region` | Região da plataforma | todos |
| `/lab-data/<env>/foundation/alerts_topic_arn` | ARN do tópico SNS de alertas | `data-ingestion` (falhas de Lambda e Step Functions) |

## data-lakehouse

Publicado por `data-lakehouse/infra/contract.tf` (camada 3).

| Parâmetro | Conteúdo | Consumidores |
|---|---|---|
| `/lab-data/<env>/lakehouse/bronze_bucket` | Bucket da camada bronze | `data-ingestion` |
| `/lab-data/<env>/lakehouse/silver_bucket` | Bucket da camada silver | dbt do lake |
| `/lab-data/<env>/lakehouse/gold_bucket` | Bucket da camada gold | dbt do lake, DW |
| `/lab-data/<env>/lakehouse/glue_database_bronze` | Database do Glue do bronze | `data-ingestion` |
| `/lab-data/<env>/lakehouse/glue_database_silver` | Database do Glue da silver | dbt do lake |
| `/lab-data/<env>/lakehouse/glue_database_gold` | Database do Glue da gold | dbt do lake, DW |
| `/lab-data/<env>/lakehouse/athena_workgroup` | Workgroup do Athena (`lab-data-<env>`) | `data-ingestion`, dbt do lake, analytics |
| `/lab-data/<env>/lakehouse/athena_results_bucket` | Bucket de resultados do Athena | quem precisar ler resultados de consulta direto do S3 |

**Consome:** `/lab-data/<env>/foundation/name_prefix` e `account_id`.

## data-ingestion

⬜ Planejado (camada 4). A ingestão é consumidora; ela publica só o que o orquestrador vai precisar (ex.: o ARN de cada state machine).
