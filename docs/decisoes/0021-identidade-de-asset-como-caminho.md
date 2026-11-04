# ADR-0021 — Identidade de asset como caminho, sem domínio

- **Status:** Aceito
- **Data:** 2026-11-03
- **Camada:** Todas
- **Substitui:** o item 1 (identidade) do [ADR-0020](0020-padrao-de-assets.md); os demais itens do ADR-0020 continuam valendo

## Contexto

O ADR-0020 definiu o ID com 4 partes fixas, `<camada>.<domínio>.<fonte>.<entidade>`. Dois problemas apareceram antes do primeiro asset ir para a conta:

1. **O domínio no ID trava a correção.** Domínio é uma classificação de negócio e pode estar errado ou mudar. Com ele no ID, corrigir o domínio obriga a renomear o asset, a pasta no S3, a tabela, os consumidores e a linhagem.
2. **Profundidade fixa não cabe em toda origem.** Uma origem com sistema e subsistema (ex.: CVM → fundos → documentos → informe diário) não cabe em 4 partes, e a saída seria "espremer" dois níveis num nome só.

## Decisão

- O ID é um **caminho de tamanho variável**: `<camada>.<origem>[.<nível>...].<entidade>`, com de 0 a 4 níveis entre a origem e a entidade.
- **O domínio sai do ID** e fica só no campo `domain` do contrato ODCS e na tag `Domain` dos recursos.
- Todos os nomes físicos são derivados do caminho, pela mesma regra em qualquer profundidade ([assets](../assets.md#2-nomes-físicos-derivados)): contrato `assets/<camada>/<caminho>.yml`, S3 `<caminho>/`, tabela `<caminho com _>`, URI `labdata://<camada>/<caminho>`, lib `labdata.assets.<camada>.<caminho sem a entidade>`.
- A hierarquia de pastas de todo repositório segue `<origem>/<níveis...>/<entidade>`, sem domínio.

| Antes (ADR-0020) | Depois |
|---|---|
| `bronze.financeiro.bcb.cdi_diario` | `bronze.bcb.sgs.cdi_diario` |
| `assets/bronze/financeiro/bcb/cdi_diario.yml` | `assets/bronze/bcb/sgs/cdi_diario.yml` |
| S3 `financeiro/bcb/cdi_diario/` | S3 `bcb/sgs/cdi_diario/` |
| Tabela `bcb_cdi_diario` | Tabela `bcb_sgs_cdi_diario` |

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Manter 4 partes com domínio (ADR-0020) | Formato fixo, previsível | Domínio errado vira renomeação em cascata; não cabe origem com subsistema |
| 4 partes fixas sem domínio (`<camada>.<origem>.<sistema>.<entidade>`) | Simples | Continua travado: origem sem sistema ou com subsistema não cabe |
| Espelhar o caminho completo da API (ex.: `bcb.bcdata.sgs`) | Fiel à URL | Partes que não diferenciam nada (o `bcdata` está em toda URL do BCB); a URL já fica em `servers.location` do contrato e na linhagem |
| **Caminho variável sem domínio** (escolhida) | Acompanha a estrutura de cada origem; domínio corrigível sem renomear | Nomes físicos mais longos em origens profundas (limitado a 4 níveis) |

## Consequências

- Uma origem com estrutura diferente não exige mudança de código, schema ou convenção: usa os níveis de que precisar.
- A tabela concatena o caminho inteiro, então duas origens nunca geram o mesmo nome.
- Segue o modelo de *namespaces* hierárquicos usado no mercado (namespaces aninhados do Iceberg, caminhos de dataset do OpenLineage e do DataHub).
- Como nenhum asset tinha sido aplicado na conta, a mudança não exigiu migração.
