# actions

Reusable workflows de CI/CD da org [fiap-vcosta](https://github.com/fiap-vcosta).

Os repositórios consumidores ficam só com callers finos apontando para `@main`. Um push nesta `main` atualiza todos. Este repositório **não** entra em `ci_repositories` (sem WIF próprio) e **não** expõe `tf-apply` / `tf-destroy` para o `infra-bootstrap` (apply do bootstrap continua só local, por owner).

## Workflows

| Arquivo | Uso |
|---------|-----|
| [`ci-dotnet.yml`](.github/workflows/ci-dotnet.yml) | Lint (InspectCode), unitários (gate 80% line/branch), integração, SonarCloud |
| [`ci-terraform.yml`](.github/workflows/ci-terraform.yml) | `fmt` / `validate`; `plan` opcional em PR |
| [`build-push.yml`](.github/workflows/build-push.yml) | Build e push da imagem no Artifact Registry |
| [`deploy.yml`](.github/workflows/deploy.yml) | Deploy no GKE (manifests em `k8s/`, rollout, smoke HTTPS) |
| [`tf-apply.yml`](.github/workflows/tf-apply.yml) | `terraform apply` manual (`workflow_dispatch` no caller) |
| [`tf-destroy.yml`](.github/workflows/tf-destroy.yml) | `terraform destroy` manual (`workflow_dispatch` no caller) |

Pin padrão do Terraform: **1.16.1**. Working directory padrão: **`terraform`**.

## Exemplos de caller

### CI .NET (serviço)

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  ci:
    uses: fiap-vcosta/actions/.github/workflows/ci-dotnet.yml@main
    permissions:
      contents: read
      packages: read
    with:
      sonar_project_key: fiap-vcosta_administrativo-service
      enable_github_packages: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Com `enable_github_packages: true`, o caller precisa de `packages: read` no job para o restore no GitHub Packages. O reusable exporta `NUGET_AUTH_TOKEN=${{ github.token }}` nos jobs .NET para o `%NUGET_AUTH_TOKEN%` do `nuget.config` do serviço expandir no `dotnet restore`/`build`/`test` (não só no `setup-dotnet`).


### CI Terraform (com plan em PR)

```yaml
name: CI

on:
  push:
  pull_request:

permissions:
  contents: read
  id-token: write

jobs:
  terraform:
    uses: fiap-vcosta/actions/.github/workflows/ci-terraform.yml@main
    with:
      run_plan: true
```

### CI Terraform (só fmt/validate — bootstrap)

```yaml
jobs:
  terraform:
    uses: fiap-vcosta/actions/.github/workflows/ci-terraform.yml@main
    with:
      run_plan: false
```

### build-push

O reusable faz `docker build` com BuildKit e monta o secret `nuget_token` a partir de `GITHUB_TOKEN` (`packages: read`), para o restore do pacote `building-blocks` no GitHub Packages. O `Dockerfile` do serviço precisa seguir o contrato do [`building-blocks`](https://github.com/fiap-vcosta/building-blocks#docker-build-com-secret-mount):

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=nuget_token \
    dotnet nuget update source github \
      --username github \
      --password-stdin \
      --store-password-in-clear-text < /run/secrets/nuget_token \
 && dotnet restore
```

O caller precisa conceder `packages: read` no job (o reusable já declara a mesma permissão).

```yaml
name: build-push

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write
  packages: read

jobs:
  build-push:
    uses: fiap-vcosta/actions/.github/workflows/build-push.yml@main
    with:
      image_name: administrativo-service
```

### deploy

O reusable aplica `k8s/namespace.yaml`, `serviceaccount.yaml`, sincroniza o **Secret Kubernetes** da app (Secret Manager + org secrets), depois `configmap.yaml` (envsubst `DD_SITE`), `deployment.yaml` (envsubst `IMAGE` / `DD_VERSION`), `service.yaml`, `hpa.yaml`, `httproute.yaml` e `healthcheckpolicy.yaml` quando existirem; faz rollout e smoke.

Por padrão (`sync_app_secret: true`) o Secret no cluster fica com:

| Key | Origem |
|-----|--------|
| `DD_API_KEY` | org/repo secret `DD_API_KEY` |
| `ConnectionStrings__DefaultConnection` | Secret Manager (`connection_string_sm_secret`), se informado |
| `ServiceAuth__Key` | Secret Manager (`service_auth_sm_secret`) ou fallback org secret `SERVICE_AUTH_KEY` |
| `Jwt__SigningKeyPem` | Secret Manager (`jwt_signing_key_sm_secret`), se informado |
| `MercadoPago__AccessToken` | org/repo secret `MERCADO_PAGO_ACCESS_TOKEN`, se informado |
| `MercadoPago__WebhookSecret` | org/repo secret `MERCADO_PAGO_WEBHOOK_SECRET`, se informado |

```yaml
name: deploy

on:
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    uses: fiap-vcosta/actions/.github/workflows/deploy.yml@main
    with:
      image_name: administrativo-service
      deployment_name: administrativo-service
      smoke_url: https://api.example.com/administrativo/health
      connection_string_sm_secret: administrativo-service-connection-string
      service_auth_sm_secret: administrativo-service-service-auth-key
    secrets:
      DD_API_KEY: ${{ secrets.DD_API_KEY }}
```

### tf-apply / tf-destroy

A **concurrency fica no caller** (não no reusable). O mesmo grupo no caller e no reusable deadlocka o job `uses:` — o reusable nunca sobe.

```yaml
name: tf-apply

on:
  workflow_dispatch:

concurrency:
  group: terraform-infra-db
  cancel-in-progress: false

permissions:
  contents: read
  id-token: write

jobs:
  apply:
    uses: fiap-vcosta/actions/.github/workflows/tf-apply.yml@main
```

Passos extras do `infra-k8s` (Gateway pós-apply, cleanup pré-destroy) ficam em **jobs do caller**, depois ou antes do reusable, cobertos pelo mesmo `concurrency` do workflow.

## Inputs principais

### `ci-dotnet`

| Input | Default | Obrigatório |
|-------|---------|-------------|
| `sonar_project_key` | — | sim |
| `solution` | `TechChallenge.sln` | não |
| `unit_project` | `tests/UnitTests/UnitTests.csproj` | não |
| `integration_project` | `tests/IntegrationTests/IntegrationTests.csproj` | não |
| `coverage_threshold` | `80` | não |
| `global_json_file` | `global.json` | não |
| `enable_github_packages` | `false` | não |

Secret: `SONAR_TOKEN`.

### `ci-terraform` / `tf-apply` / `tf-destroy`

| Input | Default | Notas |
|-------|---------|-------|
| `terraform_version` | `1.16.1` | |
| `working_directory` | `terraform` | |
| `run_plan` | `true` | só CI |
| `require_k8s_vars` | `false` | checa projeto/região/cluster e exporta `TF_VAR_*` |

Concurrency de apply/destroy: declarar no workflow caller, não nestes reusables.

### `build-push` / `deploy`

| Input | Obrigatório |
|-------|-------------|
| `image_name` | sim |
| `deployment_name` | deploy |
| `smoke_url` | deploy |
| `k8s_namespace` | não (`tech-challenge`) |
| `image_tag` | não (`latest`) |
| `jwt_signing_key_sm_secret` | não (vazio = omitir `Jwt__SigningKeyPem`) |

Secrets opcionais do `deploy` (além de `DD_API_KEY` / `SERVICE_AUTH_KEY`):

| Secret | Papel |
|--------|--------|
| `MERCADO_PAGO_ACCESS_TOKEN` | `MercadoPago__AccessToken` (vazio = omitir) |
| `MERCADO_PAGO_WEBHOOK_SECRET` | `MercadoPago__WebhookSecret` (vazio = omitir) |

`build-push` usa BuildKit secret `id=nuget_token` (`NUGET_AUTH_TOKEN` = `github.token`). Caller e reusable precisam de `packages: read`.

## Fora de escopo

- Apply/destroy do `infra-bootstrap`
- CI próprio do pacote `building-blocks` (não usa estes reusables)
- Código de aplicação, Terraform de nuvem e manifests dos serviços
