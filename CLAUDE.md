# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Project Overview

Shareable [Renovate](https://docs.renovatebot.com/) preset configurations,
consumed by other repos via `"extends"` in their `renovate.json`. The repo
is hosted at `github>donaldgifford/renovate-config`.

See `docs/design/0001-shareable-renovate-preset-architecture.md` for the
full architecture and design conventions.

## Preset Layers

Presets are JSON5 files in the repo root, organized in three layers:

**Base:** `default.json5` — global defaults every repo needs.

**Ecosystem (pick one or more):**

- `go.json5` — `gomodTidy`, `gomodUpdateImportPaths`, no automerge on
  toolchain bumps, `dont-release` for `.goreleaser.yml`/`.golangci.yml`
- `rust.json5` — `update-lockfile`, automerge lockfile maintenance
- `node.json5` — `update-lockfile`, `yarnDedupeHighest`, groups `@types/*`,
  no automerge on Node version bumps
- `terraform.json5` — terragrunt, pin provider digests, scoped lockfile
  maintenance, regex manager for boilerplate variables
- `helm.json5` — scoped to `charts/`, per-chart branches, appVersion
  tracking via Docker tags
- `kustomize.json5` — no automerge, `dont-release` labels
- `actions.json5` — pin digests, group non-major, no automerge on major

**Cross-cutting (compose as needed):**

- `ci.json5` — `dont-release` labels for Actions, Dockerfiles, config files
- `docker.json5` — pin Dockerfile digests, regex for `docker-bake.hcl`
- `mise.json5` — regex manager for `mise.toml` (needs `# renovate:`
  annotations)

## Validation & CI

CI runs on every PR via `.github/workflows/validate.yml`:

1. `renovate-config-validator --strict <file>` — validates syntax, semantics,
   deprecated options against every `.json5` file
2. `prettier --check "*.json5"` — enforces consistent formatting

To validate locally:

```bash
npx renovate-config-validator --strict <file>
prettier --check "*.json5"
```

## Linting & Formatting

Tools managed via [mise](https://mise.jdx.dev/) (`mise.toml`):

- **Markdown:** `markdownlint-cli2`, `prettier` (proseWrap: always, 80 cols)
- **YAML:** `yamlfmt`, `yamllint`
- **JSON5:** `prettier`, `renovate-config-validator`

## Conventions for Adding/Editing Presets

- **JSON5** with `// --- Description ---` comment separators
- Only `default.json5` sets `$schema` and `extends`
- Every preset must include: group non-major (`addLabels: ["patch"]`),
  major bumps (`addLabels: ["minor"]`), and automerge override if needed
- Scope all rules with `matchManagers` to the relevant manager(s)
- Group name pattern: `"<ecosystem> <thing> (non-major)"`
- Custom managers use `# renovate:` annotation pattern
- Labels: `dependencies` (base), `patch`, `minor`, `dont-release`, `security`
- Schedule: weekly Monday before 6am, `America/Detroit`
- Major bumps are never automerged
