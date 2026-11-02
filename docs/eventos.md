# Assets e eventos

A plataforma é **orientada a assets** ([ADR-0015](decisoes/0015-assets-e-eventos.md)): cada dado tem nome, dono e lugar, e quando ele é atualizado, **publica um evento**. Quem depende dele reage ao evento, sem depender de horário.

```
asset[source.financeiro.bcb.sgs] ──► asset[bronze.financeiro.bcb.sgs] ──► asset[silver.financeiro.bcb.indicadores] ──► ...
        (externo)                        (data-ingestion)                       (dbt, camada 5)
```

O padrão completo dos assets (identidade de 4 partes, nomes físicos derivados, contrato ODCS, lib e linhagem OpenLineage) está em [assets.md](assets.md). Este documento trata só do **evento** que avisa quando um asset fica pronto.

## O evento `AssetUpdated`

Publicado no barramento **`lab-data-<env>-assets`** (EventBridge, criado pelo `data-foundation`; nome e ARN no [contrato](contratos.md#data-foundation)) quando um asset fica **pronto**: gravado e validado.

```json
{
  "source": "lab-data.ingestion",
  "detail-type": "AssetUpdated",
  "detail": {
    "asset": "bronze.financeiro.bcb.sgs",
    "versao_contrato": "1.0.0",
    "particao": "2026-11-02",
    "carga": "substituir",
    "estatisticas": { "registros": 120, "arquivos": 4, "bytes": 5321, "hash": "sha256:9f2c..." },
    "local": "s3://lab-data-dev-bronze-<account_id>/financeiro/bcb/sgs/",
    "run_id": "arn:aws:states:us-east-2:<account_id>:execution:lab-data-dev-ingest-bcb-sgs:<id>",
    "produzido_em": "2026-11-02T08:00:05Z"
  }
}
```

| Campo | Regra |
|---|---|
| `source` | `lab-data.<componente>`: quem publicou (`lab-data.ingestion`, `lab-data.lakehouse`...) |
| `detail-type` | Sempre `AssetUpdated` |
| `detail.asset` | ID do asset (4 partes), igual ao `id` do contrato ODCS |
| `detail.versao_contrato` | Versão do contrato do asset no momento da publicação |
| `detail.particao` | Partição atualizada (data, ou `full` para cargas completas) |
| `detail.carga` | Como o consumidor aplica a atualização: `substituir` (troca a partição), `upsert` (atualiza por chave) ou `acrescentar` |
| `detail.estatisticas` | Opcional: `registros`, `arquivos`, `bytes` e `hash` do conteúdo (o consumidor pode ignorar um hash repetido) |
| `detail.local` | Onde o dado está |
| `detail.run_id` | Execução que produziu o dado (ligação com o log e com `_run_id`) |
| `detail.produzido_em` | Momento da publicação (UTC, ISO 8601) |

Validado por [schemas/asset-updated.schema.json](schemas/asset-updated.schema.json).

**Quando não publicar:** dado em quarentena, execução com falha ou execução sem dado novo.

## Como reagir a um asset

O **consumidor** cria, no próprio repositório, uma regra no barramento para os assets de que depende:

```hcl
resource "aws_cloudwatch_event_rule" "bronze_financeiro_bcb_sgs" {
  event_bus_name = data.aws_ssm_parameter.assets_event_bus_name.insecure_value
  event_pattern = jsonencode({
    "detail-type" = ["AssetUpdated"]
    detail        = { asset = ["bronze.financeiro.bcb.sgs"] }
  })
}
```

Com vários assets em `depende_de`, o `disparo` do contrato decide: `qualquer` = uma regra que reage a qualquer um; `todos` = o consumidor só roda quando todos tiverem evento novo na mesma janela (controle feito pelo próprio consumidor).

O alvo da regra pode ser uma state machine, uma fila SQS (lida pelo Airflow 3 com *AssetWatcher*, na camada 7) ou outra Lambda. O produtor não sabe quem consome; ele só publica.

## Replay

O archive do barramento guarda os eventos por 90 dias. Para reprocessar um período, inicia-se um *replay* dos eventos daquele intervalo: os consumidores recebem os eventos de novo e refazem o processamento.

## `AssetUpdated` × OpenLineage

| | `AssetUpdated` | OpenLineage |
|---|---|---|
| Papel | **Disparar** quem depende do asset | **Registrar** a linhagem e as execuções |
| Quando | Só quando o asset fica pronto (gravado e validado) | Em toda execução: início, fim, falha |
| Para onde | Barramento `lab-data-<env>-assets` | Marquez (local) ou catálogo (camada 11) |

Os dois são emitidos pela mesma execução, com o mesmo `run_id`.

## Eventos do Glue Catalog

O Glue publica sozinho eventos de mudança no catálogo (`Glue Data Catalog Table State Change`) no barramento padrão da conta. Eles **não** substituem o `AssetUpdated` (não disparam em arquivo novo numa partição existente, nem em tabelas com *partition projection*, e não carregam contexto). Ficam para **observabilidade e auditoria** na camada 10: por exemplo, alertar quando uma tabela for alterada fora de um pipeline.
