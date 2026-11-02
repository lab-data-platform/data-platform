# ADR-0009 — Infraestrutura como código com Terraform e state remoto no S3

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Fundação

## Contexto

Toda a infraestrutura precisa ser reproduzível: criada, destruída e recriada sem passo manual, para controlar o custo do lab e documentar o ambiente.

## Decisão

- **Terraform** (≥ 1.10) com o provider AWS `~> 6.0`.
- **State remoto** no bucket `lab-data-tfstate-<account_id>`, criado pelo `data-foundation/bootstrap` (que mantém o próprio state local) e usado por todos os repositórios, cada um com a key `<componente>/<env>/terraform.tfstate`.
- **Lock nativo do S3** (`use_lockfile`), sem tabela DynamoDB.
- **Backend com configuração parcial**, preenchida pelo `make init`, para nenhum dado da conta ficar fixo no código.
- Recursos com tags automáticas via `default_tags`.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| AWS CDK | Infra em Python/TypeScript | Mais acoplado à AWS; menos usado em times de dados |
| CloudFormation | Nativo, sem state para gerenciar | Verboso; menos portável |
| Lock com DynamoDB | Padrão antigo | Desnecessário desde o Terraform 1.10 |

## Consequências

- O `make destroy` de cada repositório precisa funcionar sempre. O bucket de state é a exceção, protegido por `prevent_destroy`.
- Se o state local do bootstrap se perder, ele é recuperado com `terraform import`.
