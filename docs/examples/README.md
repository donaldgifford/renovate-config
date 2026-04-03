# Example Configs

Copy the appropriate example to your repo as `renovate.json` and you're done.

| Example | Use Case |
|---------|----------|
| [go-project.json](go-project.json) | Go service with Dockerfiles and mise |
| [rust-project.json](rust-project.json) | Rust project with Dockerfiles and mise |
| [node-bun-project.json](node-bun-project.json) | Node/TypeScript project using bun |
| [backstage-yarn-project.json](backstage-yarn-project.json) | Backstage or yarn-based project with Docker |
| [terraform-terragrunt-project.json](terraform-terragrunt-project.json) | Terraform/OpenTofu with Terragrunt and Boilerplate |
| [kustomize-k8s-project.json](kustomize-k8s-project.json) | Kubernetes manifests with Kustomize |
| [helm-charts-repo.json](helm-charts-repo.json) | Helm charts monorepo |
| [go-k8s-operator.json](go-k8s-operator.json) | Go-based K8s operator (Go + Kustomize + Docker) |

## Customizing

These are starting points. Add repo-specific overrides after the
`"extends"` array:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:go",
    "github>donaldgifford/renovate-config:ci"
  ],
  "ignoreDeps": ["github.com/some/fork"],
  "packageRules": [
    {
      "matchPackageNames": ["k8s.io/*"],
      "dependencyDashboardApproval": true
    }
  ]
}
```
