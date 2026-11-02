# ADR-0017 — Trilha de auditoria da conta com CloudTrail

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Fundação (etapa 1b)

## Contexto

Sem uma trilha, o CloudTrail só guarda o histórico de eventos por 90 dias, e apenas para consulta no console. Não há como investigar um incidente antigo nem provar quem alterou um recurso.

## Decisão

- Uma trilha **`lab-data-<env>-audit`**, criada pelo `data-foundation`: eventos de gerenciamento, serviços globais incluídos, validação de integridade dos logs e multirregião.
- Logs num bucket próprio, `lab-data-<env>-audit-<account_id>`: versionado, criptografado, sem acesso público, só TLS, **sem `force_destroy`** e com retenção de 365 dias.
- A trilha multirregião fica numa variável (`audit_multi_region`): no plano gratuito só `us-east-2` está liberada ([ADR-0013](0013-regiao-us-east-2.md)), e a criação multirregião precisa ser confirmada no primeiro apply.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Só o histórico padrão (90 dias) | Grátis, sem configuração | Some em 90 dias; sem integridade nem arquivo próprio |
| Eventos de dados do S3 (*data events*) | Auditoria de cada leitura e gravação de objeto | Cobrados por evento; desnecessários para dados públicos no lab |
| CloudTrail Lake | Consulta SQL nos eventos | Custo por GB ingerido |

## Consequências

- A primeira trilha de eventos de gerenciamento é gratuita; paga-se só o armazenamento dos logs no S3 (centavos).
- O bucket de auditoria não some num `make destroy`: para removê-lo, é preciso esvaziá-lo antes, de propósito.
- Os logs podem ser consultados pelo Athena no futuro (tabela sobre o bucket de auditoria).
