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

Com `enable_github_packages: true`, o caller precisa de `packages: read` no job para o restore no GitHub Packages.


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

```yaml
name: build-push

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  build-push:
    uses: fiap-vcosta/actions/.github/workflows/build-push.yml@main
    with:
      image_name: administrativo-service
```

### deploy

O reusable aplica `k8s/namespace.yaml`, `serviceaccount.yaml`, `configmap.yaml` (envsubst `DD_SITE`), `deployment.yaml` (envsubst `IMAGE` / `DD_VERSION`), `service.yaml`, `hpa.yaml` e `httproute.yaml` quando existirem; faz rollout e smoke. **Não** cria secrets da aplicação — o caller garante os secrets no cluster antes (job prévio ou outro fluxo).

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

## Fora de escopo

- Apply/destroy do `infra-bootstrap`
- CI próprio do pacote `building-blocks` (não usa estes reusables)
- Código de aplicação, Terraform de nuvem e manifests dos serviços
