# ADR-0014 — S3 Tables como armazenamento da silver

- **Status:** Proposto
- **Data:** 2026-11-02
- **Camada:** Lakehouse (camada 3) e Transformação (camada 5)

## Contexto

Na camada 3, cada camada de dado ganhou um bucket S3 comum, com as tabelas Iceberg gerenciadas pelo Glue Data Catalog e pelo Athena. O usuário quer manter a silver aberta para outra forma de armazenamento, em especial o **Amazon S3 Tables**: buckets próprios para tabelas Iceberg, com compactação, limpeza de snapshots e remoção de arquivos órfãos feitas pela AWS.

O código já está preparado: a variável `layer_storage` do `data-lakehouse` define a implementação de cada camada. Hoje só aceita `s3`.

Leitura de teste em 2026-11-02: a API do S3 Tables responde em `us-east-2` na conta do plano gratuito (`list-table-buckets` sem erro).

## Decisão (proposta)

Avaliar o S3 Tables para a silver **antes da camada 5** (dbt), respondendo:

1. **Integração com o Athena:** as tabelas do S3 Tables aparecem no Athena pelo catálogo federado `s3tablescatalog`. Essa integração depende do **Lake Formation**? Se sim, ela antecipa parte da camada 11, que foi deixada para depois.
2. **dbt:** o adapter `dbt-athena` cria e atualiza tabelas (incremental com `merge`) num table bucket do S3 Tables?
3. **Teste local:** o moto emula o S3 Tables? Sem isso, a silver perde o nível 2 de teste.
4. **Custo:** além do armazenamento, o S3 Tables cobra pela compactação e por objeto monitorado. Quanto isso representa no volume do lab?
5. **Plano gratuito:** a criação de um table bucket é permitida pela SCP da conta (até agora só a leitura foi testada)?

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| **Bucket S3 comum + Glue Catalog** (implementação atual) | Simples, funciona com dbt-athena e com o moto, sem Lake Formation | Manutenção do Iceberg (`OPTIMIZE`, `VACUUM`) fica por nossa conta |
| **S3 Tables** | Manutenção automática do Iceberg, desempenho de consulta otimizado | Integração com o Athena e o dbt a confirmar, possível dependência do Lake Formation, custo extra de manutenção |

## Consequências

- Se aceito: implementar `layer_storage.silver = "s3tables"` no `data-lakehouse` como uma nova implementação (table bucket + namespace), sem mudar o contrato. O parâmetro `silver_bucket` passaria a apontar para o table bucket, e um ADR novo substitui este.
- Se descartado: a silver segue no bucket comum, e a manutenção do Iceberg entra na orquestração (camada 7).
