# Padrão de assets

Todo dado da plataforma é um **asset**: tem uma identidade única, um contrato e uma linhagem, **independentemente da ferramenta que o produz** (Lambda, dbt ou Airflow). Trocar a ferramenta muda só quem produz; a identidade, os consumidores e a linhagem continuam os mesmos ([ADR-0020](decisoes/0020-padrao-de-assets.md)).

| Peça | Padrão | Onde vive |
|---|---|---|
| **Identidade** | ID `<camada>.<domínio>.<fonte>.<entidade>` e nomes físicos derivados dele | Regra neste documento |
| **Contrato** | **ODCS v3.2.0** (Open Data Contract Standard), mais as extensões da plataforma | `assets/` do repositório produtor |
| **Lib** | `labdata.assets.<camada>.<domínio>.<fonte>`, gerada a partir dos contratos | `src/labdata/assets/` do repositório produtor |
| **Linhagem** | **OpenLineage**, emitido a cada execução | Catálogo de metadados (Marquez local; OpenMetadata ou DataHub na camada 11) |
| **Disparo** | Evento `AssetUpdated` | Barramento `lab-data-<env>-assets` ([eventos](eventos.md)) |

O "lugar único" para ver tudo junto é o **catálogo de metadados**, que recebe o que cada repositório publica. A regra fica central (este documento); os assets ficam com quem os produz.

---

## 1. Identidade

```
<camada>.<domínio>.<fonte>.<entidade>
```

| Parte | Valores | Exemplo |
|---|---|---|
| `camada` | `source` (fonte externa: API, arquivo, banco, FTP), `bronze`, `silver`, `gold`, `dw`, `dashboard` | `bronze` |
| `domínio` | Área de negócio | `financeiro` |
| `fonte` | Origem (nas camadas `source` e `bronze`) ou assunto (nas camadas seguintes) | `bcb`, `mercado` |
| `entidade` | O dado em si | `sgs`, `indicador_diario` |

Tudo em minúsculas e `snake_case`. Exemplos de uma cadeia completa:

```
source.financeiro.bcb.sgs  ──►  bronze.financeiro.bcb.sgs  ──►  silver.financeiro.bcb.indicadores
   (externo)                       (Lambda, data-ingestion)        (dbt, data-lakehouse)
                                                                          │
                       dw.financeiro.mercado.fato_indicador_diario  ◄──  gold.financeiro.mercado.indicador_diario
```

**Unicidade:** cada repositório é dono das fontes que ingere (o `bcb` é do `data-ingestion`), e os nomes físicos são derivados do ID. Dois assets com o mesmo ID quebram o CI do repositório; entre repositórios, aparecem como conflito no catálogo.

## 2. Nomes físicos derivados

Ninguém inventa nome físico: todos saem do ID, por regra.

| A partir de `bronze.financeiro.bcb.sgs` | Regra | Resultado |
|---|---|---|
| Prefixo no S3 | `<domínio>/<fonte>/<entidade>/` no bucket da camada | `s3://lab-data-dev-bronze-<account_id>/financeiro/bcb/sgs/` |
| Tabela no Glue | `lab_data_<env>_<camada>.<fonte>_<entidade>` | `lab_data_dev_bronze.bcb_sgs` |
| URI (Airflow `Asset`, catálogo) | `labdata://<camada>/<domínio>/<fonte>/<entidade>` | `labdata://bronze/financeiro/bcb/sgs` |
| `source` do dbt | `source('<camada>_<domínio>', '<fonte>_<entidade>')` | `source('bronze_financeiro', 'bcb_sgs')` |
| Identidade OpenLineage | `namespace = s3://<bucket>`, `name = <prefixo>` | `s3://lab-data-dev-bronze-<account_id>` · `financeiro/bcb/sgs` |
| Evento | `detail.asset` = ID | `bronze.financeiro.bcb.sgs` |

## 3. Contrato (ODCS)

Cada asset tem **um arquivo** no padrão **ODCS v3.2.0**, no repositório que o produz:

```
assets/<camada>/<domínio>/<fonte>/<entidade>.yml
```

```yaml
apiVersion: v3.2.0
kind: DataContract
id: bronze.financeiro.bcb.sgs          # ID da plataforma
name: Séries do SGS (bronze)
version: 1.0.0                         # versão do contrato (semver, ver governança §8)
status: active
domain: financeiro
team:
  name: data-ingestion                 # repositório dono
description:
  purpose: Séries temporais do Banco Central como vieram da API, por série e data de ingestão.
  usage: Fonte da silver.financeiro.bcb.indicadores.
tags: [bronze, bcb]
servers:
  - server: bronze
    type: s3
    environment: dev
    location: s3://lab-data-dev-bronze-<account_id>/financeiro/bcb/sgs/
    format: json
schema:
  - name: sgs
    physicalName: bcb_sgs
    logicalType: object
    description: Uma observação de uma série do SGS.
    properties:
      - { name: data,  logicalType: string, physicalType: string, required: true, description: "Data da observação (dd/mm/aaaa)." }
      - { name: valor, logicalType: string, physicalType: string, required: true, description: "Valor da série, como texto." }
    quality:
      - { type: library, metric: rowCount, mustBeGreaterThan: 0, description: "O arquivo tem pelo menos uma observação." }
slaProperties:
  - { property: freshness, value: 1, unit: d }      # sem AssetUpdated em 1 dia = alerta
customProperties:                                   # extensões da plataforma (obrigatórias)
  - { property: classificacao, value: publico }
  - { property: criticidade,   value: media }
  - { property: depende_de,    value: [source.financeiro.bcb.sgs] }
  - { property: disparo,       value: todos }
  - property: produzido_por
    value: { ferramenta: lambda, repositorio: data-ingestion, referencia: pipelines/financeiro/bcb/sgs.yml }
```

### Extensões da plataforma (`customProperties`)

O ODCS prevê `customProperties` para o que é específico de cada organização. Na plataforma, estas são **obrigatórias**:

| Propriedade | Valores | Para quê |
|---|---|---|
| `classificacao` | `publico`, `interno`, `confidencial`, `pessoal` | Acesso e retenção ([governança §2](governanca.md#2-classificação)) |
| `criticidade` | `alta`, `media`, `baixa` | Prioridade dos alertas de falha e de frescor |
| `depende_de` | Lista de IDs (vazia para `source`) | Linhagem declarada e regras de disparo |
| `disparo` | `todos` (padrão) ou `qualquer` | Quando há várias dependências: esperar todas ou reagir a qualquer uma |
| `produzido_por` | `{ferramenta, repositorio, referencia}`, com `ferramenta` = `lambda`, `dbt`, `airflow` ou `externo` | Quem produz, sem prender o asset à ferramenta |

O frescor esperado usa o campo padrão do ODCS (`slaProperties`, `property: freshness`).

### Validação

| Verificação | Ferramenta | Onde |
|---|---|---|
| Formato ODCS | `datacontract lint` (datacontract-cli) | pre-commit e CI do repositório produtor |
| Extensões e regras da plataforma (ID de 4 partes, `domain` igual ao do ID, propriedades obrigatórias) | [schemas/asset-extensions.schema.json](schemas/asset-extensions.schema.json) | pre-commit e CI |
| Unicidade e dependências internas do repositório | teste em `tests/contracts/` | CI |

## 4. Lib de assets

Os contratos viram **constantes importáveis**, para nenhum código escrever um ID ou um nome físico à mão:

```python
from labdata.assets.bronze.financeiro import bcb

bcb.sgs.id          # "bronze.financeiro.bcb.sgs"
bcb.sgs.s3_prefix   # "financeiro/bcb/sgs/"
bcb.sgs.table       # "bcb_sgs"
bcb.sgs.uri         # "labdata://bronze/financeiro/bcb/sgs"
bcb.sgs.depende_de  # ("source.financeiro.bcb.sgs",)
```

| Regra | Por quê |
|---|---|
| **O YAML em `assets/` é a fonte da verdade**; a lib só o lê (os YAMLs vão no pacote como *package data*) | Uma definição só, sem cópias para divergir |
| **Cada repositório publica a sua parte** do mesmo `labdata.assets` (*namespace package*): o `data-ingestion` publica `source` e `bronze`; o `data-lakehouse`, `silver` e `gold` | Um import único para a plataforma, com cada asset no repositório de quem o produz |
| **Sem `__init__.py` em `labdata/` e em `labdata/assets/`** | É o que permite ao Python juntar as partes de pacotes diferentes |
| **Distribuição por tag do Git** (`labdata-ingestion @ git+https://github.com/lab-data-platform/data-ingestion@v1.2.0`) | O consumidor fixa a versão; mudar um asset nunca quebra ninguém sem aviso |
| **Quem não é Python consome um artefato gerado** (ex.: o `sources.yml` do dbt) | O dbt não importa Python |

Erro de digitação num ID vira `ImportError`, e não um asset fantasma.

## 5. Linhagem (OpenLineage)

A cada execução, quem produz o asset emite eventos **OpenLineage** (`START`, `COMPLETE`, `FAIL`) com o job, as entradas e as saídas, usando a identidade da seção 2.

| Ferramenta | Como emite |
|---|---|
| Lambda (`data-ingestion`) | Cliente `openlineage-python`, chamado pelo `sdk` em toda execução |
| dbt | `dbt-ol` (nativo) |
| Airflow | Provider `apache-airflow-providers-openlineage` (nativo) |

| Ambiente | Para onde vão os eventos |
|---|---|
| Local | **Marquez** em Docker, com interface para ver o grafo |
| Conta | Catálogo de metadados da camada 11 (OpenMetadata ou DataHub) |

**OpenLineage × `AssetUpdated`:** os dois coexistem. O OpenLineage **registra** a linhagem e as execuções (observabilidade); o `AssetUpdated` **dispara** os consumidores. Um não substitui o outro.
