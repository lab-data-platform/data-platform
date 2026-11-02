# ADR-0007 — Acesso do CLI com usuário IAM e access key

- **Status:** Aceito
- **Data:** 2026-10-31
- **Camada:** Ambiente local

## Contexto

O CLI, o Terraform e o dbt precisam de credenciais da AWS. O tipo de conta usada **não tem IAM Identity Center (SSO)** e **não permite login de usuários IAM no console**, então o `aws login` também não está disponível. A causa foi descoberta depois: a conta está no plano gratuito, sem os advanced features ativados ([ADR-0013](0013-regiao-us-east-2.md)).

## Decisão

- Usuário IAM `lab-admin` com a policy `AdministratorAccess` e uma access key, configurada no profile `lab-data`.
- O root fica só para administração da conta, com MFA e sem access key.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| IAM Identity Center (SSO) | Credenciais temporárias | Indisponível no tipo de conta |
| `aws login` | Credenciais temporárias a partir do console | Exige login do usuário IAM no console, que a conta não permite |
| Access key do root | Nenhum | Prática proibida |

## Consequências

- A chave não expira: `chmod 600` nos arquivos do `~/.aws`, rotação a cada 90 dias, nunca no repositório.
- No CI (camada 12), usar OIDC do GitHub em vez de access key.
- Se a conta passar a permitir login no console, migrar para o `aws login` e registrar um ADR novo.
