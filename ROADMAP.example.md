# ROADMAP — acompanhamento da plataforma `<nome-da-plataforma>`

> **Modelo.** Copie para `ROADMAP.md` e preencha. O `ROADMAP.md` desta plataforma é um exemplo preenchido.
>
> ```bash
> cp -n ROADMAP.example.md ROADMAP.md   # -n: não sobrescreve um ROADMAP.md existente
> ```

Painel de progresso: **onde estamos, o que vem a seguir e o que já foi feito**. Atualize ao fim de cada sessão de trabalho.

Documento único: planejamento (objetivo, entregáveis e critério de pronto de cada camada) e acompanhamento (onde estamos, próximos passos, histórico).

**Legenda:** ✅ concluída · ⏳ em andamento · 🟡 código pronto, falta aplicar · ⬜ não iniciada · ⛔ bloqueada

---

## 📍 Onde estamos

| | |
|---|---|
| **Camada atual** | `<número> — <nome> (<repositório>)` |
| **Situação** | `<uma frase: o que já está feito e o que falta nesta camada>` |
| **Próximo passo** | `<a próxima ação concreta>` |
| **Última atualização** | `AAAA-MM-DD` |

## ⏭️ Próximos passos

Só as próximas 3 a 5 ações, em ordem.

1. [ ] `<ação>`
2. [ ] `<ação>`
3. [ ] `<ação>`

## ⚠️ Pendências e bloqueios

| Item | Impacto | Decidir quando |
|---|---|---|
| `<o que está pendente ou bloqueado>` | `<o que isso impede>` | `<agora / ao chegar na camada N>` |

---

## 🗺️ Camadas

| # | Camada | Repositório | Status |
|---|---|---|---|
| 0 | Ambiente local e acesso à AWS | — | ⬜ |
| 1 | Fundação | `data-foundation` | ⬜ |
| 2 | Repositório e CI básico | todos | ⬜ |
| 3 | Lakehouse | `data-lakehouse` | ⬜ |
| 4 | Ingestão | `data-ingestion` | ⬜ |
| 5 | Transformação no lake (dbt) | `data-lakehouse` | ⬜ |
| 6 | Rede e segurança base | `data-foundation` | ⬜ |
| 7 | Orquestração com Airflow | a criar | ⬜ |
| 8 | Data Warehouse | a criar | ⬜ |
| 9 | Analytics | a criar | ⬜ |
| 10 | Qualidade e observabilidade | todos | ⬜ |
| 11 | Governança e catálogo | a criar | ⬜ |
| 12 | CI/CD completo | todos | ⬜ |

### Camada 0 — Ambiente local e acesso à AWS ⬜

Guia: [docs/setup/ambiente-local.md](docs/setup/ambiente-local.md)

- [ ] AWS CLI v2 e Terraform instalados
- [ ] MFA no root
- [ ] Usuário IAM com acesso pelo CLI, no profile da plataforma
- [ ] `aws sts get-caller-identity` retorna o usuário IAM, e não o root
- [ ] Regiões e serviços liberados na conta verificados (contas no plano gratuito têm restrições, ver ADR-0013)

### Camada 1 — Fundação ⬜

Repositório: `data-foundation`

- [ ] `make test` passando
- [ ] `make bootstrap-apply`: bucket de state criado
- [ ] `infra/terraform.tfvars` criado (com `cp -n`) e com o seu e-mail
- [ ] `make init` e `make apply`: budget, tópico de alertas e contrato SSM
- [ ] Assinatura de e-mail do tópico confirmada

### Camada 2 — Repositório e CI básico ⬜

- [ ] Primeiro commit de cada repositório
- [ ] Hospedagem remota (GitHub)
- [ ] CI com `terraform fmt -check`, `validate` e `test`
- [ ] `pre-commit` com detector de segredos

### Camada 3 — Lakehouse ⬜

Repositório: `data-lakehouse`

- [ ] Escopo confirmado
- [ ] Buckets, databases do Glue e workgroup do Athena
- [ ] Contrato publicado no SSM
- [ ] Testes nos níveis 1, 2 e 3

### Camada 4 — Ingestão ⬜

Repositório: `data-ingestion`

- [ ] Escopo confirmado e fontes definidas
- [ ] Módulo reutilizável por fonte
- [ ] Lambdas e state machines
- [ ] Testes nos níveis 1, 2 e 3

<!-- Acrescente as camadas seguintes quando chegar nelas. -->

---

## 📜 Histórico

Uma linha por marco: decisão importante, camada aplicada, problema resolvido.

| Data | O que aconteceu |
|---|---|
| `AAAA-MM-DD` | `<marco>` |
