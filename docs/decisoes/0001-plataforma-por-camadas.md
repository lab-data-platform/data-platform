# ADR-0001 — Plataforma construída por camadas, com componentes plugáveis

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Todas

## Contexto

O projeto começa do zero, numa conta AWS de teste, com o objetivo de construir uma plataforma de dados de ponta a ponta (Lakehouse, Data Warehouse e Analytics). As ferramentas podem mudar ao longo do caminho, e pode ser útil ter mais de uma ferramenta para a mesma finalidade, para comparar ou para o usuário escolher.

## Decisão

1. A plataforma é construída **uma camada por vez**, na ordem do [roadmap](../../ROADMAP.md). Cada camada só começa com o escopo confirmado.
2. Cada **capacidade** (orquestração, DW, analytics, governança...) pode ter **mais de uma implementação**, cada uma num repositório ou módulo próprio, que se liga aplicando e se desliga destruindo.
3. As camadas conversam por **contratos estáveis**: layout no S3, tabelas no Glue Catalog e projetos dbt. O orquestrador chama as peças, mas não contém regra de negócio.
4. O nome da plataforma é `lab-data`, e o prefixo dos recursos é `lab-data-<env>`.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Construir tudo de uma vez | Resultado completo mais rápido | Difícil de entender, revisar e mudar de rumo |
| Uma ferramenta fixa por capacidade | Mais simples | Trocar de ferramenta exige refazer as camadas vizinhas |

## Consequências

- Cada camada precisa de critério de pronto e de documentação antes da próxima.
- Implementações alternativas custam mais código (um repositório ou módulo por ferramenta), mas não geram custo de nuvem enquanto estiverem desligadas.
- A organização em repositórios foi definida depois, no [ADR-0010](0010-polyrepo.md).
