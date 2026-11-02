# Convenções da plataforma `lab-data`

Regras que todo repositório, recurso, tabela e fonte de dados da plataforma segue. Um repositório ou uma fonte nova só entra na plataforma se respeitar este documento.

## Repositórios

| Regra | Padrão |
|---|---|
| Nome | `data-<componente>`, por exemplo `data-foundation` e `data-ingestion` |
| Local | Dentro do workspace `lab-data-engineering/`, que não é versionado |
| Estrutura | `infra/` (Terraform + `tests/`), `src/`, `tests/`, `local/`, `docs/`, `Makefile`, `CLAUDE.md`, `README.md` ([arquitetura.md §4](arquitetura.md#4-repositórios)) |
| Makefile | Alvos padrão: `help`, `whoami`, `fmt`, `validate`, `test`, `init`, `plan`, `apply`, `destroy`; e `local-up`, `local-test`, `local-down` quando houver ambiente local |
| Dependência entre repositórios | Só pelos contratos no SSM ([contratos.md](contratos.md)), nunca pelo state ou pelo código de outro repositório |
| Branch principal | `main` |
| README | Sempre com a seção **"Recursos na AWS"** (ver abaixo) |

### Seção "Recursos na AWS" no README

Todo repositório que cria recursos na AWS lista, no README, **tudo o que cria**, nesta tabela e com estas colunas:

| Serviço AWS | Recurso | Para que serve | Região | Custo | Código | Onde ver no console |
|---|---|---|---|---|---|---|
| Nome oficial do serviço (ex.: **Amazon S3**) | Tipo e nome do recurso | Uma frase | `us-east-2` ou global | Estimativa | Arquivo `.tf` | Link do console, já na região |

Regras:
- Uma linha por recurso **que aparece no console**. Configurações de um recurso (versionamento, criptografia) ficam na frase do próprio recurso, e não em linhas separadas.
- Logo abaixo da tabela: o que está **previsto** (camadas futuras), o que foi criado **fora do Terraform** e o comando para listar os recursos pela tag `Repository`.
- Atualize a tabela no mesmo commit em que o recurso entra ou sai do código.
- Antes de aplicar, a tabela fica como **"Recursos na AWS (previstos)"**.
- Referência: [data-foundation/README.md](../../data-foundation/README.md#recursos-na-aws).

**Plataforma × domínio:** a plataforma é o que todo domínio usa (buckets, catálogo, workgroup, orquestrador, alertas). O domínio é o que é específico de uma área (fontes, transformações e modelos de dados dela). Dentro de cada repositório, o código é organizado por domínio, por exemplo `data-ingestion/src/financeiro/bcb/`.

## Commits

Padrão [Conventional Commits](https://www.conventionalcommits.org/pt-br/), com a descrição em **português do Brasil**.

```
<tipo>(<escopo opcional>): <descrição no imperativo, minúscula, sem ponto final>

<corpo opcional: o que mudou e por quê>
```

| Tipo | Quando usar | Exemplo |
|---|---|---|
| `feat` | Recurso ou capacidade nova | `feat(infra): adiciona tópico SNS de alertas` |
| `fix` | Correção de erro | `fix(bootstrap): corrige região do bucket de state` |
| `docs` | Só documentação | `docs(adr): registra decisão sobre a região us-east-2` |
| `test` | Testes novos ou ajustados | `test(infra): valida os parâmetros do contrato` |
| `refactor` | Mudança de código sem mudar comportamento | `refactor(infra): extrai locals do contrato` |
| `chore` | Manutenção: configuração, `.gitignore`, dependências | `chore: adiciona .gitignore` |
| `ci` | Pipeline de CI/CD | `ci: valida terraform fmt no push` |
| `build` | Empacotamento e build | `build(lambda): empacota dependências da ingestão` |
| `perf` | Melhoria de desempenho | `perf(dbt): particiona a tabela por mês` |
| `revert` | Desfaz um commit anterior | `revert: desfaz feat(infra): ...` |

Regras:
- **Descrição no imperativo:** "adiciona", "corrige", "remove", e não "adicionado" ou "adicionando".
- **Escopo** = a parte do repositório afetada: `bootstrap`, `infra`, `adr`, `roadmap`, `dbt`, `lambda`...
- **Um assunto por commit.** Mudanças que atravessam repositórios viram um commit em cada um, na ordem de dependência.
- **Mudança que quebra contrato** leva `!` depois do tipo, por exemplo `feat(infra)!: renomeia parâmetro do contrato`, e explica o impacto no corpo.

## Nomes de recursos

| Recurso | Padrão | Exemplo |
|---|---|---|
| Prefixo geral | `<platform>-<env>` | `lab-data-dev` |
| Bucket S3 | `<platform>-<env>-<camada>-<account_id>` | `lab-data-dev-bronze-<account_id>` |
| Database do Glue | `<platform>_<env>_<camada>` (com `_`, porque o Athena não aceita `-`) | `lab_data_dev_silver` |
| Tabela | `<fonte>_<entidade>` em `snake_case` | `cvm_inf_diario` |
| Lambda / State machine | `<prefixo>-<função>` | `lab-data-dev-ingest-bcb` |
| Tópico SNS | `<prefixo>-<finalidade>` | `lab-data-dev-alerts` |
| Budget | `<prefixo>-mensal` | `lab-data-dev-mensal` |
| Parâmetro SSM (contrato) | `/<platform>/<env>/<componente>/<chave>` | `/lab-data/dev/foundation/alerts_topic_arn` |

O account id entra no nome dos buckets porque nomes de bucket são globais na AWS. O nome da plataforma nos recursos continua `lab-data`, independentemente do nome das pastas.

## Tags

Aplicadas automaticamente pelo `default_tags` do provider de cada repositório. Nenhum recurso é criado sem elas.

| Tag | Valor | Uso |
|---|---|---|
| `Platform` | `lab-data` | Identifica o que pertence à plataforma |
| `Environment` | `dev` / `prd` | Separa os ambientes |
| `Component` | `foundation`, `lakehouse`, `ingestion`... | Agrupa o custo por componente |
| `ManagedBy` | `terraform` | Indica que não deve ser alterado no console |
| `Repository` | nome do repositório, ex.: `data-foundation` | Aponta para o código de origem |
| `Domain` | ex.: `financeiro` | Agrupa o custo por domínio (só em recursos de domínio) |

## Camadas de dados (arquitetura medalhão)

| Camada | Conteúdo | Formato | Quem escreve |
|---|---|---|---|
| **bronze** | Dado como veio da fonte, mais os metadados de ingestão | Arquivo bruto (JSON/CSV, comprimido) | `data-ingestion` |
| **silver** | Dado limpo, tipado e deduplicado, ainda no modelo da fonte | Apache Iceberg | dbt do lake |
| **gold** | Entidades de negócio conformadas, prontas para consumo e para o DW | Apache Iceberg | dbt do lake |

A modelagem dimensional (fatos e dimensões) fica no Data Warehouse, e não na gold ([ADR-0003](decisoes/0003-data-warehouse-redshift-serverless.md)).

## Terraform

- **State remoto** no bucket `<platform>-tfstate-<account_id>`, criado pelo `data-foundation/bootstrap`, com versionamento e lock nativo do S3 (`use_lockfile`).
- **Key do state:** `<componente>/<env>/terraform.tfstate`, por exemplo `foundation/dev/terraform.tfstate`.
- **Nada da conta fica fixo no código:** o account id vem do `aws_caller_identity`, o backend é configurado no `make init` e os valores pessoais (e-mail) ficam em `terraform.tfvars`, que o Git ignora.
- **Todo repositório tem `terraform test`** com provider simulado (`make test`).
- **`make destroy` precisa funcionar sem passo manual** em todo repositório. O bootstrap é a exceção (`prevent_destroy`), porque guarda o state de todos.
- **Destruir na ordem inversa da dependência:** `data-ingestion` → `data-lakehouse` → `data-foundation`.
- **Sempre `make plan` antes de `make apply`.**

## Custo

- Serviços serverless, que só cobram pelo uso. Não usar recurso sempre ligado (NAT Gateway, Redshift provisionado, MWAA) sem uma decisão registrada.
- Budget mensal da conta com alertas em 50%, 80% e 100%, mais um alerta de previsão.
