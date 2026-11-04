# ADR-0020 — Padrão de assets: identidade, ODCS, lib e OpenLineage

- **Status:** Aceito; o item 1 (identidade) foi substituído pelo [ADR-0021](0021-identidade-de-asset-como-caminho.md)
- **Data:** 2026-11-02
- **Camada:** Todas
- **Substitui:** [ADR-0016](0016-contrato-de-dados-por-asset.md) (contrato em formato próprio, `assets.yml`)

## Contexto

A plataforma pode usar Lambda, dbt ou Airflow, e trocar de ferramenta ao longo do tempo. O objetivo é uma **linhagem completa**, com todos os assets seguindo o mesmo padrão e com unicidade. Três caminhos foram avaliados:

- **registro central no `data-platform`**: mistura o hub de documentação com configuração viva usada por código, com versões e ritmos diferentes;
- **repositório próprio de catálogo**: unicidade forte, mas foge do que o mercado faz para esse porte;
- **padrões de mercado**: identidade por convenção de nomes, contrato no repositório do produtor num formato aberto e linhagem por eventos de um padrão aberto, juntados num catálogo de metadados.

## Decisão

1. **Identidade:** ID `<camada>.<domínio>.<fonte>.<entidade>`, com a camada `source` para fontes externas de qualquer tipo. Todos os nomes físicos (S3, Glue, URI, dbt, OpenLineage) são **derivados do ID** ([assets](../assets.md#2-nomes-físicos-derivados)).
2. **Contrato:** um arquivo **ODCS v3.2.0** por asset, em `assets/<camada>/<domínio>/<fonte>/<entidade>.yml`, no **repositório produtor**. As regras da plataforma vão em `customProperties` obrigatórias (`classificacao`, `criticidade`, `depende_de`, `disparo`, `produzido_por`), e o frescor, no `slaProperties` padrão.
3. **Validação:** `datacontract lint` para o formato ODCS e [asset-extensions.schema.json](../schemas/asset-extensions.schema.json) para as regras da plataforma, no pre-commit e no CI.
4. **Lib:** os contratos viram constantes em `labdata.assets.<camada>.<domínio>.<fonte>`, lidas dos YAMLs (fonte da verdade). Cada repositório publica a sua parte do mesmo *namespace package* e distribui por tag do Git.
5. **Linhagem:** cada execução emite eventos **OpenLineage**, com a identidade derivada do ID. Localmente, o **Marquez** mostra o grafo; na camada 11, o catálogo de metadados (OpenMetadata ou DataHub) recebe os mesmos eventos.
6. **Disparo:** continua com o `AssetUpdated` ([ADR-0015](0015-assets-e-eventos.md)), que ganha `carga` (substituir, upsert, acrescentar) e `estatisticas`.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Registro central no `data-platform` | Lista única | Hub sobrecarregado; versões de documentação e de catálogo misturadas |
| Repositório `data-catalog` | Unicidade garantida antes do deploy | Dois commits por asset; incomum nesse porte |
| Formato próprio (`assets.yml`, ADR-0016) | Simples | Sem ferramentas de mercado; cada ferramenta acabaria criando o seu |
| **Padrões de mercado** (escolhida) | ODCS e OpenLineage abertos (Linux Foundation), com ferramentas prontas; funciona com Lambda, dbt e Airflow | A unicidade entre repositórios é verificada no catálogo, e não antes |

## Consequências

- O "lugar único" para ver tudo junto é o **catálogo de metadados**, e não um repositório.
- Adicionar um asset = um contrato ODCS no repositório produtor; a lib, a validação e a linhagem vêm do mesmo arquivo.
- A ingestão (camada 4) já nasce com contrato ODCS, lib de assets, OpenLineage e Marquez local.
- O ADR-0015 continua valendo para o evento `AssetUpdated`; os nomes de asset passam a ter 4 partes.
