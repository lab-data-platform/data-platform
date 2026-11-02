# ADR-0015 — Plataforma orientada a assets, com eventos AssetUpdated no EventBridge

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Todas (começa na Fundação e na Ingestão)

## Contexto

Os dados da plataforma formam uma cadeia de dependências: `api.bcb.sgs → bronze.bcb.sgs → silver → gold → DW`. Disparar cada passo por horário acopla tudo à hora em que o anterior *costuma* terminar. O usuário quer definir os assets e disparar eventos quando um asset é atualizado, de forma organizada.

O Glue tem recursos parecidos, mas cada um cobre só uma parte: os eventos do Data Catalog são técnicos (mudança de estrutura, e não de dado), os Glue Workflows só orquestram jobs do Glue e os assets do SageMaker Unified Studio são de governança, e não de disparo.

## Decisão

- **Asset** = todo dado com nome, dono e lugar, nomeado `<camada>.<fonte>.<entidade>` ([eventos](../eventos.md)).
- Cada repositório declara seus assets num `assets.yml`, que também é o contrato de dados ([ADR-0016](0016-contrato-de-dados-por-asset.md)).
- Quando um asset fica pronto (gravado e validado), o produtor publica o evento **`AssetUpdated`** no barramento EventBridge **`lab-data-<env>-assets`**, criado pelo `data-foundation`. Na Step Functions, é o último passo, com a integração nativa (sem Lambda).
- O consumidor cria, no próprio repositório, a regra para os assets de que depende.
- O barramento tem **archive** de 90 dias, para auditoria e replay.
- Os eventos do Glue Data Catalog ficam para observabilidade e auditoria (camada 10), e não para disparo.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Agendamento por horário | Simples | Acopla ao tempo; atraso de um passo quebra os seguintes |
| Eventos do Glue Data Catalog | Gratuitos, sem código | Não disparam em arquivo novo numa partição existente nem com *partition projection*; sem contexto (registros, execução) |
| Glue Workflows | Nativo do Glue | Só orquestra jobs e crawlers do Glue |
| Eventos do S3 (Object Created) | Nativos | Um evento por arquivo, e não por asset pronto; dispara antes da validação |
| Só Airflow Assets | Linhagem e disparo no próprio orquestrador | Prende a plataforma ao Airflow; a ingestão (Step Functions) não teria como avisar |

## Consequências

- Produtores e consumidores ficam desacoplados: o produtor não sabe quem consome.
- O Airflow 3 (camada 7) entra como mais um consumidor, lendo os eventos por uma fila SQS (*AssetWatcher*), sem mudar nada nos produtores.
- O formato do evento vira contrato, validado por JSON Schema ([schemas/asset-updated.schema.json](../schemas/asset-updated.schema.json)).
- Custo: US$ 1 por milhão de eventos; no lab, praticamente zero.
