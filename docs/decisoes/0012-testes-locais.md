# ADR-0012 — Testes locais em três níveis, com moto como emulador

- **Status:** Aceito
- **Data:** 2026-11-02
- **Camada:** Todas

## Contexto

Todo repositório precisa ser testável na máquina local, sem depender da conta AWS para cada mudança. Nem todo serviço da AWS tem emulador local confiável.

Em 23/03/2026, o LocalStack encerrou a edição Community: a imagem passou a exigir conta e auth token, e o plano gratuito ficou restrito a uso não comercial.

## Decisão

Três níveis de teste, com os mesmos alvos de `make` em todo repositório:

| Nível | Ferramentas | Alvo |
|---|---|---|
| 1. Unitário | `terraform test` com `mock_provider`; pytest com moto (em memória) | `make test` |
| 2. Integração local | **moto em modo servidor** (Docker) e as ferramentas reais em Docker (Airflow, DuckDB) | `make local-up`, `make local-test`, `make local-down` |
| 3. Integração real | Conta AWS, ambiente `dev` | `make plan`, `make apply` |

- O **moto** é o emulador padrão: open source (Apache 2.0) e sem cadastro.
- O nível 2 só existe onde o emulador é confiável. Onde não é, o repositório documenta o limite e fica nos níveis 1 e 3.

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| LocalStack | Emulação mais completa, integração com Terraform (`tflocal`) | Exige conta e token; plano gratuito só para uso não comercial; Glue, Athena e Redshift só na versão paga |
| Só a conta AWS | Comportamento real | Lento, com custo, e exige credencial para qualquer teste |

## Consequências

- **data-foundation:** fica nos níveis 1 e 3. Testado em 2026-11-02, o moto não emula a API S3 Control que o provider AWS v6 usa nas tags do bucket, e a criação de budget não completa.
- **data-ingestion:** é onde o nível 2 tem mais valor (Lambda, S3 e Step Functions têm bom suporte no moto).
- **dbt:** o teste local usa dbt-duckdb, com SQL compatível; o que for específico do Athena ou do Redshift fica no nível 3.
- Se o LocalStack voltar a ser uma opção viável, registrar um ADR novo.
