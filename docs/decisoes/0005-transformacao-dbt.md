# ADR-0005 — Transformação com dbt, em dois projetos (lake e DW)

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Transformação

## Contexto

As transformações precisam de versionamento, testes, documentação e linhagem. Elas acontecem em dois motores: o Athena (lake) e o Redshift (DW).

## Decisão

- **dbt** para todas as transformações em SQL.
- **Dois projetos**, um por motor:
  - `transform/dbt_lakehouse/` (adapter Athena): bronze → silver → gold, em Iceberg
  - `transform/dbt_warehouse/` (adapter Redshift): gold (via Spectrum) → dimensões, fatos e marts
- Modelos organizados por domínio dentro de cada projeto.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Um projeto dbt com dois targets | Um só lugar | Cada modelo só roda num motor; a separação ficaria confusa |
| Glue Spark (PySpark) | Escala para volume alto | Mais código e custo; fica como [alternativa](../../ROADMAP.md#implementações-alternativas) |
| SQL solto orquestrado pelo Airflow | Simples | Sem testes, documentação nem linhagem |

## Consequências

- O dbt docs dos dois projetos vira a primeira fonte de catálogo e linhagem ([ADR-0008](0008-governanca-catalogo.md)).
- A gold do lake é a fronteira entre os dois projetos: o `dbt_warehouse` declara as tabelas da gold como `sources`.
