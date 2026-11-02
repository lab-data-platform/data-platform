# Decisões de arquitetura (ADRs)

Cada decisão relevante vira um **ADR** (*Architecture Decision Record*): um registro curto com o contexto, a escolha, as alternativas e as consequências.

**Regras**
- ADR não se apaga. Se a decisão mudar, crie um ADR novo e marque o antigo como **Substituído por ADR-XXXX**.
- Uma decisão por arquivo: `NNNN-titulo-curto.md`, numeração sequencial.
- Use o [template](template.md).

**Status possíveis:** Proposto · Aceito · Substituído · Descartado

| ADR | Decisão | Status |
|---|---|---|
| [0001](0001-plataforma-por-camadas.md) | Plataforma construída por camadas, com componentes plugáveis | Aceito |
| [0002](0002-lakehouse-s3-iceberg.md) | Lakehouse em S3 + Apache Iceberg + Glue Catalog + Athena | Aceito |
| [0003](0003-data-warehouse-redshift-serverless.md) | Data Warehouse separado em Redshift Serverless, alimentado via Spectrum | Aceito |
| [0004](0004-orquestracao-airflow-ec2-step-functions.md) | Airflow em EC2 como orquestrador principal e Step Functions na ingestão | Aceito |
| [0005](0005-transformacao-dbt.md) | Transformação com dbt, em dois projetos (lake e DW) | Aceito |
| [0006](0006-analytics-quicksight.md) | Analytics com Amazon QuickSight | Aceito |
| [0007](0007-acesso-cli-iam-access-key.md) | Acesso do CLI com usuário IAM e access key | Aceito |
| [0008](0008-governanca-catalogo.md) | Governança com catálogo de metadados, sem front próprio | Proposto |
| [0009](0009-iac-terraform.md) | Infraestrutura como código com Terraform e state remoto no S3 | Aceito |
| [0010](0010-polyrepo.md) | Um repositório Git por componente, reunidos num workspace | Aceito |
| [0011](0011-contratos-ssm.md) | Contratos entre repositórios via SSM Parameter Store | Aceito |
| [0012](0012-testes-locais.md) | Testes locais em três níveis, com moto como emulador | Aceito |
| [0013](0013-regiao-us-east-2.md) | Região us-east-2, imposta pelo plano gratuito da conta | Aceito |
