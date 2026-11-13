# workflows

Workflows reutilizáveis dos repositórios de `argel-sosa90`.

## `build-push-deploy.yml`

Build da imagem Docker no runner do GitHub, push para
`registry.asolutions.dev` (tags `latest` e `sha-<commit>`) e deploy no Dokploy
pelo webhook. Sem cache de build, para não consumir a cota do plano gratuito.

Os segredos vêm do **Infisical**, projeto `ci`, environment `prod`, autenticando
por **OIDC** (nenhum segredo fica nos secrets do GitHub):

| Pasta | Chaves |
|---|---|
| `/` | `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` |
| `/<nome do repositório>` | `DOKPLOY_WEBHOOK_URL` (um por serviço, se houver mais de um) |

### Uso

`.github/workflows/deploy.yml` no repositório do app:

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    uses: argel-sosa90/workflows/.github/workflows/build-push-deploy.yml@main
    with:
      image: hostinger-lite
```

Repositório com mais de uma imagem (ex.: bills-manager), um job por imagem:

```yaml
jobs:
  backend:
    uses: argel-sosa90/workflows/.github/workflows/build-push-deploy.yml@main
    with:
      image: bills-manager-backend
      dockerfile: backend/Dockerfile
      webhook-key: DOKPLOY_WEBHOOK_URL_BACKEND
  frontend:
    uses: argel-sosa90/workflows/.github/workflows/build-push-deploy.yml@main
    with:
      image: bills-manager-frontend
      context: frontend
      webhook-key: DOKPLOY_WEBHOOK_URL_FRONTEND
```

| Input | Padrão | Descrição |
|---|---|---|
| `image` | (obrigatório) | Nome da imagem no registry |
| `context` | `.` | Contexto do build |
| `dockerfile` | `<context>/Dockerfile` | Caminho do Dockerfile |
| `build-args` | vazio | Um `KEY=valor` por linha |
| `secret-path` | `/<nome do repositório>` | Pasta do repositório no projeto `ci` |
| `webhook-key` | `DOKPLOY_WEBHOOK_URL` | Chave do webhook dentro da pasta |

O `permissions` com `id-token: write` no repositório que chama é obrigatório:
sem ele, o workflow não consegue o token OIDC para entrar no Infisical.

## Configuração (uma vez)

1. **Este repositório** é privado, então libere o uso pelos outros: *Settings →
   Actions → General → Access → "Accessible from repositories owned by the
   user"*.
2. **Infisical:** identidade de organização `github-actions` com **OIDC Auth**

   | Campo | Valor |
   |---|---|
   | OIDC Discovery URL | `https://token.actions.githubusercontent.com` |
   | Issuer | `https://token.actions.githubusercontent.com` |
   | Audiences | `https://github.com/argel-sosa90` (padrão do token do GitHub) |
   | Claims | `repository_owner_id` = `204069241` e `ref` = `refs/heads/main` |
   | Subject | em branco |

   A restrição fica nas claims, não no subject: repositórios criados depois de
   15/07/2026 usam um subject com IDs numéricos
   (`repo:argel-sosa90@<id>/<repo>@<id>:…`), e um curinga no nome do dono
   (`argel-sosa90*`) aceitaria também outras contas, como `argel-sosa90x`. O
   `repository_owner_id` é imutável. Com isso, só pushes na `main` de
   repositórios desta conta autenticam.

   Papel **Viewer** só no projeto `ci`.
3. Preencha em `build-push-deploy.yml` o `INFISICAL_IDENTITY_ID` (ID da
   identidade) e o `INFISICAL_PROJECT_ID` (ID do projeto `ci`). Não são segredos.

## Atualizações

As actions usam a major mais recente (`@vN`). O `Infisical/secrets-action` não
publica tag de major: está fixado na última versão e precisa ser atualizado à
mão quando sair uma nova.
