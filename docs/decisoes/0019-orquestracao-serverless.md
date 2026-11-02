# ADR-0019 — Orquestração serverless: Step Functions, Lambda e eventos

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Orquestração e Ingestão
- **Substitui:** [ADR-0004](0004-orquestracao-airflow-ec2-step-functions.md) (Airflow em EC2 como orquestrador principal)

## Contexto

O ADR-0004 previa o Airflow em EC2 como orquestrador principal, com o Step Functions só na ingestão. Desde então:

- a plataforma passou a ser **orientada a assets e eventos** ([ADR-0015](0015-assets-e-eventos.md)), e o EventBridge já faz o papel de "agendar por asset";
- a conta está no **plano gratuito** ([ADR-0013](0013-regiao-us-east-2.md)), com budget de US$ 10/mês, e uma EC2 ligada consome os créditos;
- o usuário já usa Airflow 3 com Assets no dia a dia, e o lab rende mais exercitando a AWS nativa.

Lambda e Airflow não são alternativas diretas: a Lambda **executa** código e o Airflow **orquestra**. A escolha é entre orquestrar com Airflow ou com Step Functions + EventBridge.

## Decisão

- **Orquestração serverless:** EventBridge Scheduler (agenda) e eventos `AssetUpdated` (dependências) disparam **state machines do Step Functions**; o código roda em **Lambda**.
- O código de extração é Python puro (`labdata.sdk`), com o runtime isolado em `sdk/runtime/`: a mesma fonte pode rodar no Airflow trocando só o runtime.
- O **Airflow vira implementação alternativa**, adotada se houver necessidade, como mais um consumidor dos eventos `AssetUpdated` (via SQS e *AssetWatcher*), sem mudar os produtores.
- A forma de rodar o dbt (camada 5) fica para a camada 5: Lambda em container ou task no ECS Fargate disparada pelo Step Functions.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| **Serverless** (escolhida) | ~US$ 0, sem servidor, encaixa nos eventos de assets, tudo liberado no plano gratuito | Grafo precisa ser gerado; dbt mais trabalhoso de rodar |
| Airflow em EC2 | Assets nativos do Airflow 3, UI do grafo, backfill confortável, dbt com Cosmos | US$ 10–30/mês, servidor para manter, precisa ficar ligado |
| Híbrido (ADR-0004) | Pratica as duas | Dobro da complexidade e dois orquestradores |

## Consequências

- A camada 7 (Airflow) deixa de ser obrigatória no roadmap e vira implementação alternativa.
- A visão do grafo vem da linhagem OpenLineage e do catálogo ([ADR-0020](0020-padrao-de-assets.md)), e não da UI de um orquestrador.
- Backfill: pelo input da execução (janela de datas) ou pelo replay dos eventos do archive.
