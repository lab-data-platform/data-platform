# ADR-0003 — Data Warehouse separado em Redshift Serverless, alimentado via Spectrum

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Data Warehouse
- **Substitui:** a decisão inicial (2026-10-30) de usar a gold no Athena como DW

> ⚠️ **Bloqueado no plano gratuito** (2026-11-02): uma SCP da conta nega o Redshift Serverless. Ao chegar na camada 8, decidir entre o upgrade da conta e a alternativa prevista ([ADR-0013](0013-regiao-us-east-2.md)).

## Contexto

A plataforma deve oferecer **Lakehouse e Data Warehouse** como componentes distintos. Usar só a gold do lake como DW deixaria o usuário de negócio sem um DW dedicado.

## Decisão

- **Redshift Serverless** como DW, com capacidade base de 8 RPU e limite de uso configurado.
- O DW é alimentado pelo **Redshift Spectrum**: um *external schema* aponta para a gold no Glue Catalog, e o projeto **dbt-redshift** materializa dimensões, fatos e marts como tabelas nativas.
- **Divisão de papéis:** a gold guarda as entidades de negócio conformadas; o DW guarda a modelagem dimensional (star schema, SCD, chaves surrogate) e os marts do BI.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Gold no Athena como DW | Custo quase zero | Lake e DW não aparecem separados; segue como [alternativa](../../ROADMAP.md#implementações-alternativas) |
| Carga com `COPY` | Comando clássico do Redshift | Lê os arquivos Parquet soltos e ignora os snapshots do Iceberg, o que pode trazer linhas já apagadas |
| Redshift provisionado | Custo previsível | Fica ligado o tempo todo; caro para o lab |
| RDS Postgres como DW | Barato e familiar | Não é colunar; não representa um DW analítico de mercado |

## Consequências

- O lake e o DW ficam consultáveis, cada um para o seu público.
- O Redshift Serverless precisa de VPC (camada 6) e de uma role IAM com acesso ao Glue e ao S3.
- O custo depende do uso. Antes de criar o DW, revisar o budget e configurar o limite de RPU-hora.
