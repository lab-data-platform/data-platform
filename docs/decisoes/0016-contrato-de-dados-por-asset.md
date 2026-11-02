# ADR-0016 — Contrato de dados por asset, declarado em assets.yml

- **Status:** Substituído por [ADR-0020](0020-padrao-de-assets.md)
- **Data:** 2026-11-02
- **Camada:** Todas (começa na Ingestão)

## Contexto

Os contratos de infraestrutura (SSM, [ADR-0011](0011-contratos-ssm.md)) dizem *onde* está cada recurso, mas não *o que* o dado é. Sem um contrato de dados, uma mudança na fonte quebra as camadas seguintes em silêncio, ninguém sabe quem responde por um dado e não há como cobrar frescor nem qualidade.

## Decisão

- Cada asset tem um **contrato de dados** no `assets.yml` do repositório produtor: descrição, dono, classificação, dependências, local, frescor esperado, schema, regras de qualidade e versão do contrato ([governança](../governanca.md#1-contrato-de-dados-por-asset)).
- O formato é validado por JSON Schema (o `assets.schema.json`, removido pelo ADR-0020), no pre-commit e no CI de cada repositório que tiver `assets.yml`.
- Um asset só entra na plataforma com o contrato declarado.
- A versão do contrato segue o versionamento semântico; mudança que quebra exige aviso aos consumidores e commit com `!`.
- As políticas que o contrato referencia (classificação, colunas técnicas, quarentena, retenção, segredos) ficam no documento de [governança](../governanca.md).

## Alternativas consideradas

| Opção | Prós | Contras |
|---|---|---|
| Só o schema do Glue Catalog | Já existe | Não tem dono, frescor, classificação nem qualidade; não cobre fontes externas |
| Contrato direto num catálogo (OpenMetadata) | Interface pronta | O catálogo só chega na camada 11; o contrato precisa nascer com o código |
| Formato aberto (ex.: Open Data Contract Standard) | Padrão de mercado | Mais verboso do que o lab precisa agora; pode ser adotado depois, mapeando os campos |

## Consequências

- O `assets.yml` alimenta a linhagem (`depende_de`), os alarmes de frescor (camada 10) e o catálogo (camada 11).
- A validação da ingestão confere as regras de `qualidade` do contrato.
- Mudar o schema de um asset vira uma decisão explícita, versionada e revisada.
