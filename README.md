# Renovate Config

Shareable [Renovate](https://docs.renovatebot.com/) presets for
`github>donaldgifford/renovate-config`. Compose presets via `"extends"` to get
consistent dependency management across all repositories.

See
[DESIGN-0001](docs/design/0001-shareable-renovate-preset-architecture.md)
for the full architecture and conventions.

## Usage

Pick a base + ecosystem + cross-cutting presets for your repo. The `mise`
preset requires `# renovate:` annotations in your `mise.toml` — see the
[mise preset docs](#mise) below.

### Go repos

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:go",
    "github>donaldgifford/renovate-config:docker",
    "github>donaldgifford/renovate-config:mise",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

### Rust repos

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:rust",
    "github>donaldgifford/renovate-config:mise",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

### Node / TypeScript repos (bun or yarn)

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:node",
    "github>donaldgifford/renovate-config:mise",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

### Terraform / OpenTofu repos (with Terragrunt / Boilerplate)

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:terraform",
    "github>donaldgifford/renovate-config:mise",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

### Repos with Dockerfiles / docker-bake

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:docker",
    "github>donaldgifford/renovate-config:mise",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

### Kustomize repos

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:kustomize",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

### Nix flake repos

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:nix",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

### Helm chart repos

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:helm",
    "github>donaldgifford/renovate-config:docker",
    "github>donaldgifford/renovate-config:ci"
  ]
}
```

## Presets

### Base

| Preset | Description |
|--------|-------------|
| `default.json5` | `config:recommended`, weekly Mon schedule (America/Detroit), automerge non-major, vulnerability alerts, PR limits (5 concurrent / 2 hourly) |

### Ecosystem

| Preset | Description |
|--------|-------------|
| `go.json5` | `gomodTidy`, `gomodUpdateImportPaths`, groups non-major, no automerge on toolchain bumps, `dont-release` for Go config files |
| `rust.json5` | `update-lockfile` range strategy, groups non-major, automerge lockfile maintenance |
| `node.json5` | `update-lockfile`, `yarnDedupeHighest`, groups non-major and `@types/*` separately, no automerge on Node version bumps |
| `terraform.json5` | Terragrunt support, pin provider digests, scoped lockfile maintenance, groups providers/modules separately, regex manager for boilerplate variables |
| `helm.json5` | Scoped to `charts/`, per-chart branch prefixes and commit messages, no automerge, appVersion tracking via Docker tags |
| `kustomize.json5` | No automerge, groups non-major image bumps, `dont-release` labels |
| `nix.json5` | Groups non-major flake inputs, no automerge on major input bumps |

### Cross-cutting

| Preset | Description |
|--------|-------------|
| `ci.json5` | Actions: pin digests, group non-major, no automerge on major. Labels Actions/Dockerfiles/config as `dont-release` |
| `docker.json5` | Pin digests in Dockerfiles, regex manager for `docker-bake.hcl` version variables |
| `mise.json5` | Regex manager for pinned versions in `mise.toml` / `.mise.toml`, requires `# renovate:` annotations |

### Mise annotations {#mise}

The `mise` preset requires annotating each pinned tool in `mise.toml`:

```toml
# renovate: datasource=github-releases depName=google/yamlfmt
yamlfmt = "0.20.0"

# renovate: datasource=pypi depName=yamllint
yamllint = "1.37.1"

# renovate: datasource=npm depName=prettier
prettier = "3.7.4"
```

Tools pinned to `"latest"` don't need annotations.

## Labels

| Label | Applied When |
|-------|-------------|
| `dependencies` | Every Renovate PR |
| `patch` | Non-major updates (minor, patch, digest, lockfile) |
| `minor` | Major updates or runtime version bumps |
| `dont-release` | Infrastructure-only changes (CI, Docker, config, mise) |
| `security` | Vulnerability alert PRs |

## Validation

CI runs `renovate-config-validator --strict` and `prettier --check` against
all `.json5` files on every PR (`.github/workflows/validate.yml`).

To validate locally:

```bash
npx renovate-config-validator --strict <file>
prettier --check "*.json5"
```
