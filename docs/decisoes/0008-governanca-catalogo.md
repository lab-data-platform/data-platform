# ADR-0008 — Governança com catálogo de metadados, sem front próprio

- **Status:** Proposto
- **Data:** 2026-11-02
- **Camada:** Governança e catálogo

## Contexto

Cada ferramenta da plataforma tem a própria interface (Airflow, dbt docs, Glue, Redshift, QuickSight, CloudWatch). Para quem consome os dados, é preciso um lugar único para responder: que dado existe, de onde vem, está atualizado, posso confiar, quem é o dono.

## Decisão (proposta)

- **Não construir um front próprio.** Esse papel é de um **catálogo de metadados**, que coleta informação de todas as ferramentas.
- Começar com o **dbt docs** dos dois projetos, publicado como site estático.
- Na camada 11, adotar um catálogo completo: **OpenMetadata** ou **SageMaker Unified Studio**. A escolha fica para essa camada.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| OpenMetadata | Open source, conectores para Airflow, dbt, Glue, Athena, Redshift e QuickSight | Precisa de cerca de 8 GB de RAM (EC2 de US$ 30–60/mês) |
| SageMaker Unified Studio (ex-DataZone) | Gerenciado e nativo da AWS | Menos conectores fora da AWS; modelo de custo a avaliar |
| DataHub | Open source, maduro | Mais pesado que o OpenMetadata |
| Front próprio | Controle total | Muito esforço para reproduzir o que os catálogos já fazem |

## Consequências

- Todos os componentes precisam ter metadados bons (descrições no dbt, tags, donos) desde as primeiras camadas, para o catálogo ter o que mostrar.
- A decisão final entre OpenMetadata e SageMaker Unified Studio vira um ADR novo na camada 11.
