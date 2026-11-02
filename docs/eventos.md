# Assets e eventos

A plataforma é **orientada a assets** ([ADR-0015](decisoes/0015-assets-e-eventos.md)): cada dado tem nome, dono e lugar, e quando ele é atualizado, **publica um evento**. Quem depende dele reage ao evento, sem depender de horário.

```
asset[source.bcb.sgs] ──► asset[bronze.bcb.sgs] ──► asset[silver.bcb.indicadores] ──► asset[gold.indicador_diario]
   (externo)            (data-ingestion)           (dbt, camada 5)                    (dbt, camada 5)
```

## Nome de um asset

`<camada>.<fonte>.<entidade>`, em minúsculas e `snake_case`:

| Camada | Exemplo | Produzido por |
|---|---|---|
| `source` (fonte externa: API, arquivo, banco, FTP...) | `source.bcb.sgs` | ninguém da plataforma |
| `bronze` | `bronze.bcb.sgs` | `data-ingestion` |
| `silver` | `silver.bcb.indicadores` | dbt do lake |
| `gold` | `gold.indicador_diario` | dbt do lake |
| `dw` | `dw.fato_indicador_diario` | dbt do DW |

A declaração completa de cada asset (contrato de dados) está na [governança](governanca.md#1-contrato-de-dados-por-asset).

## O evento `AssetUpdated`

Publicado no barramento **`lab-data-<env>-assets`** (EventBridge, criado pelo `data-foundation`; nome e ARN no [contrato](contratos.md#data-foundation)) quando um asset fica **pronto**: gravado e validado.

```json
{
  "source": "lab-data.ingestion",
  "detail-type": "AssetUpdated",
  "detail": {
    "asset": "bronze.bcb.sgs",
    "versao_contrato": "1.0.0",
    "particao": "2026-11-02",
    "registros": 120,
    "local": "s3://lab-data-dev-bronze-<account_id>/bcb/sgs/ingestion_date=2026-11-02/",
    "run_id": "arn:aws:states:us-east-2:<account_id>:execution:lab-data-dev-ingest-bcb:<id>",
    "produzido_em": "2026-11-02T08:00:05Z"
  }
}
```

| Campo | Regra |
|---|---|
| `source` | `lab-data.<componente>`: quem publicou (`lab-data.ingestion`, `lab-data.lakehouse`...) |
| `detail-type` | Sempre `AssetUpdated` |
| `detail.asset` | `id` do asset, igual ao do `assets.yml` |
| `detail.versao_contrato` | Versão do contrato do asset no momento da publicação |
| `detail.particao` | Partição atualizada (data, ou `full` para cargas completas) |
| `detail.registros` | Quantidade de registros, quando conhecida |
| `detail.local` | Onde o dado está |
| `detail.run_id` | Execução que produziu o dado (ligação com o log e com `_run_id`) |
| `detail.produzido_em` | Momento da publicação (UTC, ISO 8601) |

Validado por [schemas/asset-updated.schema.json](schemas/asset-updated.schema.json).

**Quando não publicar:** dado em quarentena, execução com falha ou execução sem dado novo.

## Como reagir a um asset

O **consumidor** cria, no próprio repositório, uma regra no barramento para os assets de que depende:

```hcl
resource "aws_cloudwatch_event_rule" "bronze_bcb_atualizado" {
  event_bus_name = data.aws_ssm_parameter.assets_event_bus_name.insecure_value
  event_pattern = jsonencode({
    "detail-type" = ["AssetUpdated"]
    detail        = { asset = ["bronze.bcb.sgs"] }
  })
}
```

O alvo da regra pode ser uma state machine, uma fila SQS (lida pelo Airflow 3 com *AssetWatcher*, na camada 7) ou outra Lambda. O produtor não sabe quem consome; ele só publica.

## Replay

O archive do barramento guarda os eventos por 90 dias. Para reprocessar um período, inicia-se um *replay* dos eventos daquele intervalo: os consumidores recebem os eventos de novo e refazem o processamento.

## Eventos do Glue Catalog

O Glue publica sozinho eventos de mudança no catálogo (`Glue Data Catalog Table State Change`) no barramento padrão da conta. Eles **não** substituem o `AssetUpdated` (não disparam em arquivo novo numa partição existente, nem em tabelas com *partition projection*, e não carregam contexto). Ficam para **observabilidade e auditoria** na camada 10: por exemplo, alertar quando uma tabela for alterada fora de um pipeline.
