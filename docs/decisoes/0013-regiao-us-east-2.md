# ADR-0013 — Região us-east-2, imposta pelo plano gratuito da conta

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Todas

## Contexto

O primeiro `apply` do bootstrap falhou com `AccessDenied ... explicit deny in a service control policy` ao criar um bucket S3 em `us-east-1`.

A investigação mostrou que:

- A conta `<account_id>` está no **plano gratuito (Free plan)**, com US$ 100 de crédito e validade até 30/04/2027, e foi criada pela "nova experiência" da AWS (AWS Settings / projetos).
- Nesse modelo, a AWS administra uma organização em nome do usuário (conta gerenciadora `<conta-gerenciadora>`) e aplica SCPs até que os **advanced features** sejam ativados. Uma delas, a `RegionFloor`, limita as regiões utilizáveis.
- Leituras de teste em 2026-11-02 mostraram que **só `us-east-2` está liberada**: `us-east-1`, `us-west-2`, `sa-east-1` e `eu-west-1` negam até a leitura.
- Em `us-east-2`, responderam: S3, SNS, SSM, Budgets, IAM, Glue, Athena, Lambda, Step Functions, EventBridge, CloudWatch Logs, EC2/VPC e Lake Formation.
- **Bloqueados por SCP:** Redshift Serverless e QuickSight.
- Isso também explica a falta de IAM Identity Center e de login de usuário IAM no console ([ADR-0007](0007-acesso-cli-iam-access-key.md)).

## Decisão

- A região da plataforma passa a ser **`us-east-2` (Ohio)**, no código (variáveis, Makefile), na documentação e no profile `lab-data` do CLI.
- O plano gratuito é mantido enquanto as camadas em foco (0 a 4) funcionarem nele.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Fazer o upgrade para o plano pago e ativar os advanced features | Libera todas as regiões e serviços, incluindo Redshift, QuickSight e Identity Center | Irreversível; remove o spend limit; o que passar do crédito é cobrado |
| Manter `us-east-1` | Região mais comum | Bloqueada pela SCP |

## Consequências

- Preços de `us-east-2` são praticamente os mesmos de `us-east-1`.
- **Camada 8 (DW) e camada 9 (Analytics) ficam impedidas no plano gratuito.** Ao chegar nelas, decidir entre o upgrade e as alternativas já previstas: gold no Athena como DW, Metabase ou Superset como BI ([ADR-0003](0003-data-warehouse-redshift-serverless.md), [ADR-0006](0006-analytics-quicksight.md)).
- Antes de usar um serviço novo, testar uma leitura dele em `us-east-2`.
- Se a conta passar para o plano pago, revisar este ADR e o ADR-0007.
