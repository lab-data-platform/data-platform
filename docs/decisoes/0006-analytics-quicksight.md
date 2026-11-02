# ADR-0006 — Analytics com Amazon QuickSight

- **Status:** Aceito
- **Data:** 2026-10-30
- **Camada:** Analytics

> ⚠️ **Bloqueado no plano gratuito** (2026-11-02): uma SCP da conta nega o QuickSight. Ao chegar na camada 9, decidir entre o upgrade da conta e a alternativa prevista ([ADR-0013](0013-regiao-us-east-2.md)).

## Contexto

A plataforma precisa de uma camada de BI para usuários de negócio, de preferência nativa da AWS.

## Decisão

**Amazon QuickSight** (parte do Amazon Quick Suite), conectado ao Redshift (marts) e, opcionalmente, ao Athena.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Metabase / Superset (Docker) | Gratuito, open source | Não é nativo da AWS; precisa de servidor |
| Power BI / Tableau | Muito usados no mercado | Fora da AWS, com licença própria |

## Consequências

- Trial de 30 dias para um autor; depois disso, cerca de US$ 24 por autor ao mês.
- Ao fim do trial, avaliar manter o QuickSight ou ligar uma alternativa open source ([roadmap](../../ROADMAP.md#implementações-alternativas)).
- A assinatura do QuickSight é feita no console; o Terraform gerencia as fontes de dados e os datasets.
