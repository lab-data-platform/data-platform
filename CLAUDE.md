# CLAUDE.md — data-platform (regras gerais da plataforma)

Contexto para o assistente de IA trabalhar em **qualquer** repositório da plataforma `lab-data`. Cada repositório tem um CLAUDE.md próprio, que aponta para este e acrescenta só o que é específico dele.

## O que é

Plataforma de dados `lab-data` na AWS, construída **camada a camada** e dividida em **um repositório Git por componente** dentro do workspace `~/projects/lab-data-engineering/`. A plataforma tem Lakehouse (S3 + Iceberg + Glue + Athena), Data Warehouse (Redshift Serverless via Spectrum) e Analytics (QuickSight), com orquestração por Airflow em EC2 e Step Functions, e transformação em dbt. Primeiro domínio: financeiro (BCB e CVM).

## Onde se localizar

| Pergunta | Arquivo (neste repositório) |
|---|---|
| Onde estamos? Próximo passo? Escopo e critério de pronto de cada camada? | [ROADMAP.md](ROADMAP.md) (documento único de planejamento e acompanhamento) |
| Como as peças e os repositórios se encaixam? | [docs/arquitetura.md](docs/arquitetura.md) |
| O que cada repositório publica para os outros? | [docs/contratos.md](docs/contratos.md) |
| Por que escolhemos cada ferramenta? | [docs/decisoes/](docs/decisoes/README.md) |
| Nomes, tags, estrutura de repositório, Terraform | [docs/convencoes.md](docs/convencoes.md) |
| Setup local e acesso à AWS | [docs/setup/ambiente-local.md](docs/setup/ambiente-local.md) |

## Como trabalhar

1. **Uma camada por vez.** Confirme o escopo com o usuário antes de construir. Não antecipe camadas futuras: nem repositório, nem pasta, nem investigação de dado.
2. **Discutir antes de construir.** Quando o usuário estiver decidindo arquitetura, discuta e recomende; não escreva código até ele confirmar.
3. **Foco atual: camadas 0 a 4** (até a ingestão). Os repositórios das camadas seguintes só são criados quando a camada começar.
4. **Decisão nova ou alterada vira ADR** em `docs/decisoes/` (template incluído). ADR antigo não se apaga: fica como *Substituído*. Atualize o índice e o `arquitetura.md`.
5. **Ao fim de cada sessão,** atualize o [ROADMAP.md](ROADMAP.md): "Onde estamos", próximos passos, checklists e histórico. **Ao concluir uma camada,** atualize também o status no README do repositório da camada. Não crie outro arquivo de roadmap.
6. **Repositório novo segue a estrutura padrão** (`convencoes.md`): `infra/` com `tests/`, `Makefile` com os alvos padrão, `README.md`, `CLAUDE.md` e `.gitignore`. O README tem sempre as seções **"Recursos na AWS"** e **"Qualidade"**, no formato padrão; ao criar ou remover um recurso, atualize a tabela no mesmo commit. Todo repositório segue o **padrão de qualidade** (pre-commit + CI + ruleset `proteger-main`) de `convencoes.md`.
7. **Contrato:** tudo o que um repositório precisa de outro passa pelo SSM (`/lab-data/<env>/<componente>/<chave>`). Ao mudar um contrato, atualize `docs/contratos.md`.
8. **Teste local:** todo repositório tem `make test` (nível 1). O nível 2 (moto) só onde o emulador é confiável ([ADR-0012](docs/decisoes/0012-testes-locais.md)).
9. **Git:** não faça commit nem push sem o usuário pedir. A `main` de todos os repositórios é protegida: toda mudança vai por branch + PR, com CI ✅ e merge por squash com `--subject` no padrão. Commits seguem o Conventional Commits em português (`feat`, `fix`, `docs`, `test`, `chore`...; ver `convencoes.md`), um assunto por commit. A identidade pessoal vem do `includeIf` no `~/.gitconfig`; confira com `git config user.email` antes de commitar.
10. **Idioma:** documentação, comentários e mensagens em português do Brasil, com acentuação correta.

## Regras com a AWS

- **Profile `lab-data`, região `us-east-2`** (única liberada no plano gratuito, [ADR-0013](docs/decisoes/0013-regiao-us-east-2.md)). Confira com `make whoami` (esperado: `user/lab-admin`).
- **A conta está no plano gratuito (Free plan)**, numa organização gerenciada pela AWS. SCPs bloqueiam outras regiões, **Redshift Serverless** e **QuickSight**. Antes de usar um serviço novo, teste uma leitura dele em `us-east-2`.
- **Nunca rode `apply`, `destroy` ou comandos que criem ou alterem recursos sem confirmação explícita do usuário.** `plan`, `validate`, `test` e consultas de leitura podem rodar.
- **Nunca exiba segredos.** Não crie access key por comando (o secret apareceria na saída). Ao listar chaves, mascare o ID. Não leia `../.secrets/`.
- **Restrições da conta:** sem IAM Identity Center e sem login de usuário IAM no console. Não sugerir SSO nem `aws login` ([ADR-0007](docs/decisoes/0007-acesso-cli-iam-access-key.md)).
- **Custo:** budget de US$ 10/mês. Não crie recurso sempre ligado (NAT Gateway, Redshift provisionado, MWAA) sem decisão registrada. Avise o custo estimado antes de criar EC2, Redshift ou QuickSight.
