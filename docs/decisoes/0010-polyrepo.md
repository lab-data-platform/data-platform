# ADR-0010 — Um repositório Git por componente, reunidos num workspace

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Todas

## Contexto

O projeto começou num repositório único, com um stack Terraform central. A preferência é que cada componente da plataforma seja um projeto independente: com a própria arquitetura, o próprio Terraform e o próprio repositório Git, mas pensado como parte do todo. E todos precisam ser testáveis localmente.

## Decisão

- **Polyrepo:** um repositório Git por componente, com o nome `data-<componente>`.
- A pasta `lab-data-engineering/` é o **workspace** que reúne os repositórios. Ela não é versionada.
- O **`data-platform`** é o hub: arquitetura, roadmap, convenções, catálogo de contratos e ADRs globais.
- Cada repositório tem o próprio Terraform, com o próprio state (`<componente>/<env>/terraform.tfstate`) no bucket comum criado pelo `data-foundation`.
- Repositórios atuais: `data-platform`, `data-foundation`, `data-lakehouse`, `data-ingestion`. Os das camadas seguintes são criados quando cada camada começar.
- O dbt fica no repositório do componente em que roda (lake no `data-lakehouse`, DW no repositório do warehouse).

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Monorepo com stack central | Uma mudança atravessa tudo num commit só | Componentes acoplados; um `apply` mexe em tudo |
| Monorepo com um stack por pasta | Isolamento de state sem vários repositórios | Não atende à preferência de um Git por projeto |
| Repositórios separados para o dbt | Separação máxima | Mais repositórios para um projeto solo; dbt e infra do mesmo componente evoluem juntos |

## Consequências

- Os repositórios só dependem uns dos outros pelos contratos no SSM ([ADR-0011](0011-contratos-ssm.md)).
- Convenções e Makefile padronizados em todos ([convencoes.md](../convencoes.md)), para trocar de repositório sem reaprender.
- Uma mudança que atravessa repositórios vira mais de um commit, na ordem de dependência.
- A hospedagem remota (GitHub) é definida na camada 2.
