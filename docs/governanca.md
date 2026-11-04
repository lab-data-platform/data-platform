# Governança de dados da plataforma `lab-data`

Políticas que valem para **todo dado** da plataforma. As [convenções](convencoes.md) dizem como o código é organizado; este documento diz **como o dado é tratado**: quem é o dono, o que ele significa, quem pode ver, quanto tempo fica e como se audita.

> Regra geral: um asset só entra na plataforma com o **contrato de dados** declarado (seção 1). Sem contrato, não há ingestão.

---

## 1. Contrato de dados por asset

Cada asset tem um **contrato de dados no padrão ODCS v3.2.0**, num arquivo `assets/<camada>/<origem>/<níveis...>/<entidade>.yml` do repositório que o produz ([ADR-0020](decisoes/0020-padrao-de-assets.md)). O contrato diz o que o asset é, quem responde por ele e o que os consumidores podem esperar.

| O contrato declara | Campo ODCS |
|---|---|
| ID, nome, descrição, dono, domínio | `id`, `name`, `description`, `team`, `domain` |
| Onde o dado está | `servers` |
| Schema e regras de qualidade | `schema[].properties`, `schema[].quality` |
| Frescor esperado | `slaProperties` (`freshness`) |
| Versão do contrato | `version` (ver seção 8) |
| Classificação, criticidade, dependências, disparo e quem produz | `customProperties` obrigatórias da plataforma |

O formato completo, um exemplo validado e as regras de validação estão em [assets.md](assets.md#3-contrato-odcs). **Sem contrato, não há ingestão.**

## 2. Classificação

Todo asset tem uma classificação, que vai na `customProperties` `classification` do contrato e na tag `Classification` dos recursos dele.

| Classificação | O que é | Exemplo | Regras |
|---|---|---|---|
| `public` | Dado já público na origem | Séries do BCB, dados abertos da CVM | Sem restrição de leitura na plataforma |
| `internal` | Dado derivado, de uso da plataforma | Métricas operacionais, tabelas intermediárias | Leitura só por componentes da plataforma |
| `confidential` | Dado de negócio sensível | Carteira de um cliente | Acesso nominal; entra no Lake Formation (camada 11) |
| `personal` | Dado pessoal (LGPD) | CPF, nome, e-mail | Como `confidential`, mais mascaramento fora da silver e retenção mínima |

Hoje só há dados `public`. As regras de `confidential` e `personal` existem antes do primeiro dado desse tipo, e não depois.

## 3. Colunas técnicas

Toda tabela, do bronze em diante, carrega as mesmas colunas de controle. No bronze, que guarda o arquivo original sem alterar, elas vêm do **caminho do arquivo** e de um **manifesto** gravado ao lado dele.

| Coluna | Conteúdo | Para quê |
|---|---|---|
| `_ingested_at` | Momento da ingestão (UTC, ISO 8601) | Saber quando o dado chegou |
| `_source` | Fonte e entidade (ex.: `bcb.sgs`) | Saber de onde veio |
| `_run_id` | ID da execução que produziu o dado (ARN da execução no Step Functions) | Rastrear até o log da execução |
| `_source_file` | Caminho do arquivo de origem no bronze | Linhagem por linha e reprocessamento |
| `_partition` | Partição de onde a linha veio (ex.: `reference_date=2026-11-02`) | Reprocessar e substituir só a partição certa |

## 4. Validação e quarentena

- Toda ingestão valida o dado **antes** de publicá-lo, conforme as regras de `quality` do contrato ODCS.
- O que falha vai para `s3://<bronze>/_quarantine/<fonte>/<entidade>/ingestion_date=<data>/`, com um arquivo explicando o motivo, e gera alerta no tópico `lab-data-<env>-alerts`.
- Dado em quarentena **não** dispara `AssetUpdated` e **não** é lido pelas camadas seguintes.
- Nada se apaga da quarentena automaticamente antes da análise; a retenção dela é a do bronze.

## 5. Idempotência e reprocessamento

- **Rodar de novo a mesma partição substitui, e nunca duplica.** O caminho de gravação é determinístico: `<fonte>/<entidade>/ingestion_date=<data>/`.
- **Carga histórica** usa o mesmo fluxo da incremental, com outra janela de datas no input da execução.
- **Reprocessar um período** = reexecutar as ingestões daquele período, ou reenviar os eventos dele pelo archive do barramento ([eventos](eventos.md)).
- O bronze é versionado: uma partição substituída mantém a versão anterior por 30 dias.

## 6. Retenção por camada

| Camada | Retenção | Por quê |
|---|---|---|
| Bronze | Indefinida para dado `public`; versões antigas por 30 dias | É a fonte de verdade para reprocessar. Revisar quando o volume crescer (transição para Glacier) |
| Silver e gold | Indefinida; o Iceberg mantém os snapshots conforme a manutenção (`expire_snapshots`) | Recriáveis a partir do bronze |
| Resultados do Athena | 7 dias | Só servem para quem acabou de consultar |
| Eventos de assets (archive) | 90 dias | Auditoria e replay de curto prazo |
| Logs de auditoria (CloudTrail) | 365 dias | Investigação de incidentes |
| Dado `personal` | O mínimo necessário à finalidade, definido no contrato | LGPD |

## 7. Segredos

- **Nunca** no código, em variável de ambiente fixa, em `.tfvars` versionado ou em log.
- Chave de API de fonte vai para o **SSM Parameter Store (`SecureString`)** ou o **Secrets Manager**, lida em tempo de execução pela Lambda, com permissão só para aquele parâmetro.
- O detector de segredos (gitleaks) roda em todo commit e no CI.
- As fontes atuais (BCB e CVM) são públicas e não usam chave.

## 8. Evolução de schema

O campo `version` do contrato ODCS (publicado como `contract_version` no evento) segue o versionamento semântico:

| Mudança | Tipo | Versão | O que fazer |
|---|---|---|---|
| Coluna nova, opcional | compatível | `1.0.0 → 1.1.0` | Atualizar o contrato |
| Descrição ou regra de qualidade mais branda | compatível | `1.0.0 → 1.0.1` | Atualizar o contrato |
| Coluna removida, renomeada ou com tipo alterado | **quebra** | `1.0.0 → 2.0.0` | Avisar os consumidores (quem tem o asset no `depends_on`), commit com `!` (`feat!:`), período de convivência das duas versões |
| Mudança na fonte que quebra a validação | detectada | — | Quarentena + alerta; o contrato é revisado antes de liberar |

## 9. Auditoria

- **CloudTrail:** trilha `lab-data-<env>-audit`, com validação de integridade, grava toda ação na conta por 365 dias ([ADR-0017](decisoes/0017-auditoria-cloudtrail.md)).
- **Eventos de assets:** o archive do barramento guarda cada atualização de asset, com a execução que a produziu.
- **Linhagem:** os eventos OpenLineage de cada execução registram entradas, saídas e o job ([assets](assets.md#5-linhagem-openlineage)).
- **Código:** toda mudança passa por commit no padrão, CI e, na `main`, PR ([convenções](convencoes.md#branches)).

## 10. Custo

- Tags `Platform`, `Environment`, `Component`, `Repository` e `Layer` em todo recurso.
- ⚠️ No plano gratuito, a **ativação das tags de custo** no Billing é negada à conta (só a conta gerenciadora da organização pode ativá-las). O custo por tag no Cost Explorer fica indisponível até um eventual upgrade; a busca por tag (`resourcegroupstaggingapi`) continua funcionando.
- Budget mensal com alertas e limite por consulta no Athena.

## 11. Runbooks

Cada repositório mantém, em `docs/runbooks/`, um runbook por procedimento operacional: o que fazer quando uma ingestão falha, como reprocessar um período, como rotacionar uma credencial. Formato: **sintoma → diagnóstico → correção → verificação**.
