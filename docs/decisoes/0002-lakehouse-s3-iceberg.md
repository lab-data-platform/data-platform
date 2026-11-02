# ADR-0002 — Lakehouse em S3 + Apache Iceberg + Glue Catalog + Athena

- **Status:** Aceito
- **Data:** 2026-10-30
- **Camada:** Lakehouse

## Contexto

O lake precisa guardar todas as camadas de dado (bronze, silver, gold) com baixo custo, suportar atualização e deduplicação (`MERGE`) e ser consultável por SQL sem servidor ligado.

## Decisão

- **S3** como armazenamento, com um bucket por camada de dado.
- **Apache Iceberg** como formato de tabela na silver e na gold. O bronze guarda o arquivo bruto da fonte.
- **Glue Data Catalog** como catálogo único, usado por Athena, dbt, Redshift Spectrum e QuickSight.
- **Athena (engine v3)** como motor de consulta do lake.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Parquet puro (sem formato de tabela) | Mais simples | Sem `MERGE`, sem ACID, deduplicação difícil |
| Delta Lake | Muito usado com Databricks | Suporte menos nativo no Athena e no Glue |
| Apache Hudi | Bom para upsert em streaming | Mais complexo de operar; menos adoção recente |
| S3 Tables (Iceberg gerenciado) | Compactação automática | Mais novo e com custo de manutenção próprio; pode ser avaliado depois |

## Consequências

- `MERGE`, time travel e evolução de schema ficam disponíveis no lake.
- O Redshift lê as tabelas Iceberg pelo Spectrum ([ADR-0003](0003-data-warehouse-redshift-serverless.md)).
- Tabelas Iceberg precisam de manutenção periódica (`OPTIMIZE` e `VACUUM`), que entra na orquestração.
