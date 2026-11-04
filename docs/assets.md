# Padrão de assets

Todo dado da plataforma é um **asset**: tem uma identidade única, um contrato e uma linhagem, **independentemente da ferramenta que o produz** (Lambda, dbt ou Airflow). Trocar a ferramenta muda só quem produz; a identidade, os consumidores e a linhagem continuam os mesmos ([ADR-0020](decisoes/0020-padrao-de-assets.md); identidade revista no [ADR-0021](decisoes/0021-identidade-de-asset-como-caminho.md)).

| Peça | Padrão | Onde vive |
|---|---|---|
| **Identidade** | ID `<camada>.<origem>[.<nível>...].<entidade>` (um caminho) e nomes físicos derivados dele | Regra neste documento |
| **Contrato** | **ODCS v3.2.0** (Open Data Contract Standard), mais as extensões da plataforma | `assets/` do repositório produtor |
| **Lib** | `labdata.assets.<camada>.<origem>[.<nível>...]`, lida dos contratos | `src/labdata/assets/` do repositório produtor |
| **Linhagem** | **OpenLineage**, emitido a cada execução | Catálogo de metadados (Marquez local; OpenMetadata ou DataHub na camada 11) |
| **Disparo** | Evento `AssetUpdated` | Barramento `lab-data-<env>-assets` ([eventos](eventos.md)) |

O "lugar único" para ver tudo junto é o **catálogo de metadados**, que recebe o que cada repositório publica. A regra fica central (este documento); os assets ficam com quem os produz.

---

## 1. Identidade

O ID de um asset é um **caminho**, de tamanho variável:

```
<camada>.<origem>[.<nível>...].<entidade>
```

| Parte | Obrigatória | O que é | Exemplo |
|---|---|---|---|
| `camada` | sim | `source` (fonte externa: API, arquivo, banco, FTP), `bronze`, `silver`, `gold`, `dw`, `dashboard` | `bronze` |
| `origem` | sim | Quem publica o dado (nas camadas `source` e `bronze`) ou o assunto (nas camadas seguintes) | `bcb`, `cvm`, `mercado` |
| `nível` | de 0 a 4 | Sistema, subsistema ou conjunto dentro da origem, quantos ela precisar | `sgs`; `fi` e `doc` |
| `entidade` | sim | O dado em si | `cdi_diario`, `inf_diario` |

Tudo em minúsculas e `snake_case`. A profundidade acompanha a estrutura de cada origem, sem forçar nenhuma num formato fixo:

| Caso | ID |
|---|---|
| Origem simples | `bronze.ibge.populacao` |
| Origem com sistema | `bronze.bcb.sgs.cdi_diario` |
| Sistema com subsistema | `bronze.cvm.fi.doc.inf_diario` |
| Assunto na gold | `gold.mercado.taxas.indicador_diario` |

Uma cadeia completa:

```
source.bcb.sgs.cdi_diario  ──►  bronze.bcb.sgs.cdi_diario  ──►  silver.bcb.sgs.cdi  ──►  gold.mercado.taxas.indicador_diario
   (externo)                       (Lambda, data-ingestion)       (dbt, data-lakehouse)        (dbt, data-lakehouse)
```

**O domínio não faz parte do ID.** Ele é uma classificação de negócio e pode ser corrigido; o ID precisa ser estável. O domínio fica no campo `domain` do contrato e na tag `Domain` dos recursos: corrigi-lo não renomeia nenhum asset, pasta, tabela ou consumidor.

**Unicidade:** cada repositório é dono das origens que ingere (o `bcb` é do `data-ingestion`), e os nomes físicos são derivados do caminho inteiro. Dois assets com o mesmo ID quebram o CI do repositório; entre repositórios, aparecem como conflito no catálogo.

## 2. Nomes físicos derivados

Ninguém inventa nome físico: todos saem do caminho, pela mesma regra, seja qual for a profundidade.

| Derivado | Regra | `bronze.bcb.sgs.cdi_diario` | `bronze.cvm.fi.doc.inf_diario` |
|---|---|---|---|
| Contrato | `assets/<camada>/<caminho>.yml` | `assets/bronze/bcb/sgs/cdi_diario.yml` | `assets/bronze/cvm/fi/doc/inf_diario.yml` |
| Pipeline (ingestão) | `pipelines/<caminho>.yml` | `pipelines/bcb/sgs/cdi_diario.yml` | `pipelines/cvm/fi/doc/inf_diario.yml` |
| Prefixo no S3 (bucket da camada) | `<caminho>/` | `bcb/sgs/cdi_diario/` | `cvm/fi/doc/inf_diario/` |
| Tabela no Glue (`lab_data_<env>_<camada>`) | caminho com `_` | `bcb_sgs_cdi_diario` | `cvm_fi_doc_inf_diario` |
| URI (Airflow `Asset`, catálogo) | `labdata://<camada>/<caminho>` | `labdata://bronze/bcb/sgs/cdi_diario` | `labdata://bronze/cvm/fi/doc/inf_diario` |
| Lib | `labdata.assets.<camada>.<caminho sem a entidade>` | `labdata.assets.bronze.bcb.sgs` → `cdi_diario` | `labdata.assets.bronze.cvm.fi.doc` → `inf_diario` |
| `source` do dbt | `source('<camada>_<origem>', '<caminho com _>')` | `source('bronze_bcb', 'bcb_sgs_cdi_diario')` | `source('bronze_cvm', 'cvm_fi_doc_inf_diario')` |
| Identidade OpenLineage | `namespace = s3://<bucket>`, `name = <caminho>` | `bcb/sgs/cdi_diario` | `cvm/fi/doc/inf_diario` |
| Evento | `detail.asset` = ID | `bronze.bcb.sgs.cdi_diario` | `bronze.cvm.fi.doc.inf_diario` |

O limite de 4 níveis intermediários existe para os nomes físicos não ficarem longos demais; se uma origem precisar de mais, muda uma linha no schema.

## 3. Contrato (ODCS)

Cada asset tem **um arquivo** no padrão **ODCS v3.2.0**, no repositório que o produz:

```
assets/<camada>/<origem>/<níveis...>/<entidade>.yml
```

```yaml
apiVersion: v3.2.0
kind: DataContract
id: bronze.bcb.sgs.cdi_diario          # ID da plataforma (o caminho)
name: CDI diário (bronze)
version: 1.0.0                         # versão do contrato (semver, ver governança §8)
status: active
domain: financeiro                     # classificação de negócio: fica aqui, e não no ID
team:
  name: data-ingestion                 # repositório dono
description:
  purpose: Observação diária do CDI (SGS série 12) exatamente como veio da API do Banco Central.
  usage: Fonte da silver do domínio financeiro.
tags: [bronze, bcb]
servers:
  - server: bronze
    type: s3
    environment: dev
    location: s3://lab-data-dev-bronze-<account_id>/bcb/sgs/cdi_diario/
    format: json
schema:
  - name: cdi_diario
    physicalName: bcb_sgs_cdi_diario
    logicalType: object
    description: A observação do dia da partição.
    properties:
      - { name: data,  logicalType: string, physicalType: string, required: true, description: "Data da observação (dd/mm/aaaa)." }
      - { name: valor, logicalType: string, physicalType: string, required: true, description: "Valor da série, como texto." }
    quality:
      - { type: library, metric: rowCount, mustBe: 1, description: "Cada partição tem exatamente a observação do dia." }
slaProperties:
  - { property: freshness, value: 4, unit: d }      # sem AssetUpdated em 4 dias = alerta (cobre feriado prolongado)
customProperties:                                   # extensões da plataforma (obrigatórias)
  - { property: classification, value: public }
  - { property: criticality,    value: medium }
  - { property: depends_on,     value: [source.bcb.sgs.cdi_diario] }
  - { property: trigger,        value: all }
  - property: produced_by
    value: { tool: lambda, repository: data-ingestion, reference: pipelines/bcb/sgs/cdi_diario.yml }
```

### Extensões da plataforma (`customProperties`)

O ODCS prevê `customProperties` para o que é específico de cada organização. Na plataforma, estas são **obrigatórias**:

| Propriedade | Valores | Para quê |
|---|---|---|
| `classification` | `public`, `internal`, `confidential`, `personal` | Acesso e retenção ([governança §2](governanca.md#2-classificação)) |
| `criticality` | `high`, `medium`, `low` | Prioridade dos alertas de falha e de frescor |
| `depends_on` | Lista de IDs (vazia para `source`) | Linhagem declarada e regras de disparo |
| `trigger` | `all` (padrão) ou `any` | Quando há várias dependências: esperar todas ou reagir a qualquer uma |
| `produced_by` | `{tool, repository, reference}`, com `tool` = `lambda`, `dbt`, `airflow` ou `external` (obrigatório em `source`) | Quem produz, sem prender o asset à ferramenta |

O frescor esperado usa o campo padrão do ODCS (`slaProperties`, `property: freshness`).

### Validação

| Verificação | Ferramenta | Onde |
|---|---|---|
| Formato ODCS | `datacontract lint` (datacontract-cli) | pre-commit e CI do repositório produtor |
| Extensões e regras da plataforma (ID em caminho de 2 a 6 partes, `domain` declarado e fora do ID, propriedades obrigatórias) | [schemas/asset-extensions.schema.json](schemas/asset-extensions.schema.json) | pre-commit e CI |
| Unicidade e dependências internas do repositório | teste em `tests/contracts/` | CI |

## 4. Lib de assets

Os contratos viram **constantes importáveis**, para nenhum código escrever um ID ou um nome físico à mão:

```python
from labdata.assets.bronze.bcb import sgs

sgs.cdi_diario.id          # "bronze.bcb.sgs.cdi_diario"
sgs.cdi_diario.s3_prefix   # "bcb/sgs/cdi_diario/"
sgs.cdi_diario.table       # "bcb_sgs_cdi_diario"
sgs.cdi_diario.uri         # "labdata://bronze/bcb/sgs/cdi_diario"
sgs.cdi_diario.domain      # "financeiro" (do contrato)
sgs.cdi_diario.depends_on  # ("source.bcb.sgs.cdi_diario",)
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
