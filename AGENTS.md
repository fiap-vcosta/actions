# Tech Challenge — Guia para agentes (`actions`)

Reusable workflows de CI/CD da org [fiap-vcosta](https://github.com/fiap-vcosta). Callers nos demais repositórios usam `fiap-vcosta/actions/.github/workflows/….yml@main`.

## Antes de mudar código

1. Manter callers finos: o miolo mora aqui; não copiar jobs de volta para os serviços
2. Não adicionar `tf-apply` / `tf-destroy` consumíveis pelo `infra-bootstrap`
3. Não inventar secrets no Git; org vars e secrets ficam no GitHub / GCP
4. **Git:** nunca commit/push direto em `main` — branch → PR → merge (ver [`.cursor/rules/git-workflow.mdc`](.cursor/rules/git-workflow.mdc))

## Responsabilidade

| Peça | Papel |
|------|--------|
| `ci-dotnet` | Lint, unit (gate 80%), integração, SonarCloud |
| `ci-terraform` | fmt / validate / plan opcional em PR |
| `build-push` | Imagem no Artifact Registry |
| `deploy` | Secret k8s (SM + org), manifests `k8s/`, rollout, smoke |
| `tf-apply` / `tf-destroy` | Apply/destroy manuais via caller `workflow_dispatch` |

## Regras canônicas (resumo)

- Push nesta `main` atualiza todos os `@main`
- Este repo está fora de `ci_repositories` (sem WIF próprio)
- `building-blocks` não consome estes reusables
- Extras do `infra-k8s` (Gateway / cleanup) ficam no caller

## Comandos

Não há build de aplicação neste repositório. Validação = revisão dos YAML + CI verde no primeiro caller após o merge.
