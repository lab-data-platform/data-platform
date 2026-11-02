# Governança de dados da plataforma `lab-data`

Políticas que valem para **todo dado** da plataforma. As [convenções](convencoes.md) dizem como o código é organizado; este documento diz **como o dado é tratado**: quem é o dono, o que ele significa, quem pode ver, quanto tempo fica e como se audita.

> Regra geral: um asset só entra na plataforma com o **contrato de dados** declarado (seção 1). Sem contrato, não há ingestão.

---

## 1. Contrato de dados por asset

Cada repositório declara os assets que produz num arquivo `assets.yml`, versionado junto com o código ([ADR-0016](decisoes/0016-contrato-de-dados-por-asset.md)). O arquivo é o **contrato de dados**: o que o asset é, quem responde por ele e o que os consumidores podem esperar.

```yaml
assets:
  - id: bronze.bcb.sgs                    # <camada>.<fonte>.<entidade>
    tipo: bronze                          # fonte-externa | bronze | silver | gold | dw | dashboard
    descricao: Séries do SGS como vieram da API, por série e data de ingestão
    dono: data-ingestion                  # repositório responsável
    classificacao: publico                # ver seção 2
    depende_de: [source.bcb.sgs]
    produzido_por: lab-data-dev-ingest-bcb
    local: s3://lab-data-dev-bronze-<account_id>/bcb/sgs/
    tabela: lab_data_dev_bronze.bcb_sgs
    frescor_esperado: 1d                  # sem evento AssetUpdated nesse prazo = alerta
    versao_contrato: 1.0.0                # ver seção 8
    schema:
      - { nome: data,  tipo: string, descricao: "Data da observação (dd/mm/aaaa)", obrigatorio: true }
      - { nome: valor, tipo: string, descricao: "Valor da série, como texto",        obrigatorio: true }
    qualidade:
      - { regra: nao_vazio,   descricao: "O arquivo tem pelo menos uma observação" }
      - { regra: obrigatorio, coluna: data }
```

| Campo | Obrigatório | Para quê |
|---|---|---|
| `id`, `tipo`, `descricao`, `dono` | sim | Identificar o asset e quem responde por ele |
| `classificacao` | sim | Controle de acesso e retenção (seções 2 e 6) |
| `depende_de` | sim (pode ser vazio) | Linhagem: o grafo da plataforma sai daqui |
| `frescor_esperado` | sim, exceto fonte externa | Alerta de atraso (camada 10) |
| `schema`, `qualidade` | sim, exceto fonte externa | O que o consumidor pode esperar e o que a validação confere |
| `versao_contrato` | sim | Evolução do schema (seção 8) |

O formato é validado por um JSON Schema: [schemas/assets.schema.json](schemas/assets.schema.json).

## 2. Classificação

Todo asset tem uma classificação, que vai no `assets.yml` e na tag `Classification` dos recursos dele.

| Classificação | O que é | Exemplo | Regras |
|---|---|---|---|
| `publico` | Dado já público na origem | Séries do BCB, dados abertos da CVM | Sem restrição de leitura na plataforma |
| `interno` | Dado derivado, de uso da plataforma | Métricas operacionais, tabelas intermediárias | Leitura só por componentes da plataforma |
| `confidencial` | Dado de negócio sensível | Carteira de um cliente | Acesso nominal; entra no Lake Formation (camada 11) |
| `pessoal` | Dado pessoal (LGPD) | CPF, nome, e-mail | Como `confidencial`, mais mascaramento fora da silver e retenção mínima |

Hoje só há dados `publico`. As regras de `confidencial` e `pessoal` existem antes do primeiro dado desse tipo, e não depois.

## 3. Colunas técnicas

Toda tabela, do bronze em diante, carrega as mesmas colunas de controle. No bronze, que guarda o arquivo original sem alterar, elas vêm do **caminho do arquivo** e de um **manifesto** gravado ao lado dele.

| Coluna | Conteúdo | Para quê |
|---|---|---|
| `_ingested_at` | Momento da ingestão (UTC, ISO 8601) | Saber quando o dado chegou |
| `_source` | Fonte e entidade (ex.: `bcb.sgs`) | Saber de onde veio |
| `_run_id` | ID da execução que produziu o dado (ARN da execução no Step Functions) | Rastrear até o log da execução |
| `_source_file` | Caminho do arquivo de origem no bronze | Linhagem por linha e reprocessamento |

## 4. Validação e quarentena

- Toda ingestão valida o dado **antes** de publicá-lo, conforme as regras de `qualidade` do contrato.
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
| Bronze | Indefinida para dado `publico`; versões antigas por 30 dias | É a fonte de verdade para reprocessar. Revisar quando o volume crescer (transição para Glacier) |
| Silver e gold | Indefinida; o Iceberg mantém os snapshots conforme a manutenção (`expire_snapshots`) | Recriáveis a partir do bronze |
| Resultados do Athena | 7 dias | Só servem para quem acabou de consultar |
| Eventos de assets (archive) | 90 dias | Auditoria e replay de curto prazo |
| Logs de auditoria (CloudTrail) | 365 dias | Investigação de incidentes |
| Dado `pessoal` | O mínimo necessário à finalidade, definido no contrato | LGPD |

## 7. Segredos

- **Nunca** no código, em variável de ambiente fixa, em `.tfvars` versionado ou em log.
- Chave de API de fonte vai para o **SSM Parameter Store (`SecureString`)** ou o **Secrets Manager**, lida em tempo de execução pela Lambda, com permissão só para aquele parâmetro.
- O detector de segredos (gitleaks) roda em todo commit e no CI.
- As fontes atuais (BCB e CVM) são públicas e não usam chave.

## 8. Evolução de schema

`versao_contrato` segue o versionamento semântico:

| Mudança | Tipo | Versão | O que fazer |
|---|---|---|---|
| Coluna nova, opcional | compatível | `1.0.0 → 1.1.0` | Atualizar o contrato |
| Descrição ou regra de qualidade mais branda | compatível | `1.0.0 → 1.0.1` | Atualizar o contrato |
| Coluna removida, renomeada ou com tipo alterado | **quebra** | `1.0.0 → 2.0.0` | Avisar os consumidores (`depende_de`), commit com `!` (`feat!:`), período de convivência das duas versões |
| Mudança na fonte que quebra a validação | detectada | — | Quarentena + alerta; o contrato é revisado antes de liberar |

## 9. Auditoria

- **CloudTrail:** trilha `lab-data-<env>-audit`, com validação de integridade, grava toda ação na conta por 365 dias ([ADR-0017](decisoes/0017-auditoria-cloudtrail.md)).
- **Eventos de assets:** o archive do barramento guarda cada atualização de asset, com a execução que a produziu.
- **Código:** toda mudança passa por commit no padrão, CI e, na `main`, PR ([convenções](convencoes.md#branches)).

## 10. Custo

- Tags `Platform`, `Environment`, `Component`, `Repository` e `Layer` em todo recurso.
- ⚠️ No plano gratuito, a **ativação das tags de custo** no Billing é negada à conta (só a conta gerenciadora da organização pode ativá-las). O custo por tag no Cost Explorer fica indisponível até um eventual upgrade; a busca por tag (`resourcegroupstaggingapi`) continua funcionando.
- Budget mensal com alertas e limite por consulta no Athena.

## 11. Runbooks

Cada repositório mantém, em `docs/runbooks/`, um runbook por procedimento operacional: o que fazer quando uma ingestão falha, como reprocessar um período, como rotacionar uma credencial. Formato: **sintoma → diagnóstico → correção → verificação**.
