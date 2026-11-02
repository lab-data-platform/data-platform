# ADR-0011 — Contratos entre repositórios via SSM Parameter Store

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Todas

## Contexto

Com um repositório por componente ([ADR-0010](0010-polyrepo.md)), um repositório precisa de valores criados por outro. Por exemplo, a ingestão precisa do bucket bronze criado pelo lakehouse. Isso não pode acoplar o código nem o state dos dois.

## Decisão

- Cada repositório publica no **SSM Parameter Store** o que os outros precisam, no caminho `/lab-data/<env>/<componente>/<chave>`.
- Quem consome lê com `data "aws_ssm_parameter"` no Terraform, ou pelo SDK no código.
- Os parâmetros são declarados num arquivo `infra/contract.tf` em cada repositório produtor e catalogados em [contratos.md](../contratos.md).
- Todo produtor tem um teste que verifica a lista de parâmetros publicados.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| `terraform_remote_state` | Nativo do Terraform | O consumidor lê o state inteiro do produtor e fica amarrado à estrutura dele |
| Valores fixos no código | Simples | Quebra quando algo muda; dados da conta no repositório |
| Nomes previsíveis por convenção | Sem leitura nenhuma | Funciona para nomes, mas não para ARNs e IDs gerados |

## Consequências

- Parâmetros do tipo *Standard* são gratuitos.
- A ordem de aplicação importa: o produtor precisa existir antes do consumidor.
- Renomear um parâmetro é uma mudança de contrato e exige atualizar os consumidores.
