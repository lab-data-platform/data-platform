# ADR-0022 — Idioma dos nomes: código e dados em inglês, texto em português

- **Status:** Aceito
- **Data:** 2026-11-03
- **Camada:** Todas

## Contexto

A primeira pipeline misturava os dois idiomas sem regra: classes em inglês (`Extractor`, `BronzeWriter`) ao lado de funções e campos em português (`data_referencia`, `estatisticas`, `sem_dado`, `classificacao`). Cada pessoa nova teria de adivinhar o idioma de cada nome, e o mesmo conceito podia aparecer nos dois (`partition` no código e `particao` no evento).

## Decisão

| O quê | Idioma | Exemplos |
|---|---|---|
| **Código:** classes, funções, métodos, propriedades, variáveis, módulos, pacotes, nomes dos testes | Inglês | `ExtractResult`, `reference_date`, `labdata.extractors`, `test_404_means_no_data_not_failure` |
| **Dados e configuração:** chaves de YAML/JSON, campos de contrato, evento e manifesto, valores enumerados, partições, colunas técnicas, variáveis de ambiente e do Makefile | Inglês | `contract_version`, `load: replace`, `status: no_data`, `reference_date=2026-10-29`, `classification: public`, `LINEAGE=1` |
| **Colunas e valores da fonte** no bronze | Nome original da fonte | `data`, `valor` (API do SGS) |
| **Texto para pessoas:** documentação, docstrings, comentários, logs, mensagens de erro, descrições dos contratos, commits | Português do Brasil | `"dado em quarentena"`, `description: Taxa DI diária...` |

Os nomes de negócio que formam o ID do asset (`cdi_diario`) e o domínio (`financeiro`) são identificadores de negócio: seguem o nome usado pela área, e mudá-los é decisão de cada domínio, não desta regra.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Tudo em português | Coerente com a documentação | Briga com a linguagem e as bibliotecas (`ExtractResult` ao lado de `get`, `put`); termos sem tradução natural (`upsert`, `stats`) |
| Tudo em inglês, inclusive documentação | Um idioma só | O time e os usuários leem em português; a documentação perderia clareza |
| **Código e dados em inglês, texto em português** (escolhida) | Padrão de mercado; nomes técnicos alinhados com ODCS, OpenLineage e AWS; texto legível pelo time | Duas regras para lembrar (a tabela acima resolve) |

## Consequências

- Os schemas da plataforma (`asset-extensions` e `asset-updated`) passaram a usar campos e valores em inglês. Como nada tinha sido aplicado na conta nem publicado no barramento, não houve migração.
- Os ADRs anteriores ficam como foram escritos; os nomes vigentes estão em [assets.md](../assets.md), [eventos.md](../eventos.md) e [governanca.md](../governanca.md).
- Os repositórios já entregues (`data-foundation`, `data-lakehouse`) ficam de ser revisados pela mesma regra (nomes de testes do Terraform e variáveis dos scripts).
