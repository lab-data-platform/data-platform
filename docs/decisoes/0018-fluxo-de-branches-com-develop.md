# ADR-0018 — Fluxo de branches com develop de integração

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Todas

## Contexto

Com a `main` protegida (PR obrigatório e CI), cada ajuste virava um PR, o que gerava PRs demais para um projeto com um só desenvolvedor. As travas da `main` continuam necessárias, mas o ciclo de desenvolvimento precisa ser mais leve.

## Decisão

- Cada repositório tem uma branch **`develop`** de integração, que recebe **push direto**, com commits no padrão e o pre-commit rodando em todo commit.
- O CI roda em todo push na `develop`, além de PRs e pushes na `main`.
- A `main` só recebe mudanças por **PR `develop → main`**, aberto no **deploy geral**: um PR por repositório, com CI obrigatório e merge por squash (ou merge commit, para preservar os commits da develop).
- O `apply` na conta acontece depois do merge na `main`, no deploy geral.
- O Dependabot abre os PRs de atualização para a `develop`.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Um PR por ajuste direto na `main` | Revisão a cada mudança | PRs demais para um desenvolvedor só |
| Push direto na `main` | Mais rápido | Remove as travas; a `main` deixa de ser confiável |
| Git Flow completo (release, hotfix) | Bem definido | Pesado demais para o lab |

## Consequências

- A `main` continua sendo o estado aplicado na conta: o que está nela, está na AWS.
- A `develop` pode ficar à frente da `main` com vários commits; o deploy geral leva todos de uma vez, depois de o CI passar no PR.
- Uma mudança que atravessa repositórios vai para a `develop` de cada um, e o deploy geral segue a ordem de dependência (`data-foundation` → `data-lakehouse` → `data-ingestion`).
