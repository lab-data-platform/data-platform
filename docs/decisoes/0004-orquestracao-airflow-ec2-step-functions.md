# ADR-0004 — Airflow em EC2 como orquestrador principal e Step Functions na ingestão

- **Status:** Substituído por [ADR-0019](0019-orquestracao-serverless.md)
- **Data:** 2026-11-02
- **Camada:** Orquestração
- **Substitui:** a decisão inicial (2026-10-30) de usar só Step Functions + EventBridge

## Contexto

O Airflow é o orquestrador mais usado no mercado e há interesse em praticá-lo. Também há interesse em praticar o Step Functions. O MWAA (Airflow gerenciado) custa cerca de US$ 350/mês, o que não cabe numa conta de teste.

## Decisão

- **Airflow 3 em Docker Compose numa EC2** (t3.medium, com agenda de liga/desliga) como orquestrador principal: DAG de ponta a ponta, com o dbt executado via Astronomer Cosmos.
- **Step Functions** no fluxo serverless de cada ingestão (Lambda de extração → validação → registro), disparado pelo Airflow com o `StepFunctionStartExecutionOperator`.
- Acesso à UI do Airflow por port forwarding via SSM, sem porta pública.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| MWAA | Gerenciado, padrão "enterprise" | Cerca de US$ 350/mês |
| Airflow local em Docker | Custo zero | Só roda com a máquina ligada |
| Só Step Functions | Serverless e barato | Não pratica Airflow; DAGs complexas ficam verbosas em ASL |

## Consequências

- Você administra a EC2 (atualizações, disco, backup do banco de metadados).
- Os dois orquestradores têm papéis separados, sem fazer a mesma coisa.
- As DAGs ficam portáveis: migrar para o MWAA depois não exige reescrevê-las.
