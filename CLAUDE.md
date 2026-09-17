# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Project Overview

Shareable [Renovate](https://docs.renovatebot.com/) preset configurations,
consumed by other repos via `"extends"` in their `renovate.json`. The repo is
hosted at `github>donaldgifford/renovate-config`.

See `docs/design/0001-shareable-renovate-preset-architecture.md` for the full
architecture and design conventions.

## Preset Layers

Presets are `.json` files in the repo root (Renovate resolves the bare shorthand
to `default.json` only — `.json5` is not used here). Organized in three layers.

**Base:** `default.json` — global defaults including `automerge: false`.

**Ecosystem (pick one or more):**

- `go.json` — `gomodTidy`, `gomodUpdateImportPaths`, toolchain bumps labeled
  `minor`
- `rust.json` — `cargo` (with `update-lockfile`) + `rust-toolchain` managers,
  enables + **automerges lockfile maintenance**
- `node.json` — `npm` + `bun` managers (both `update-lockfile`),
  `yarnDedupeHighest`, **automerges `@types/*`**
- `python.json` — `pip_requirements`, `pip_setup`, `pipenv`, `poetry`, `pep621`
  (uv/hatch/PDM), `pip-compile` (lockfile strategies for the last three), and
  `pyenv` (`.python-version`)
- `terraform.json` — Terraform modules repos. Pin provider digests, group
  providers/modules separately, enables + **automerges lockfile maintenance**.
  Does **not** match `terragrunt`.
- `terragrunt.json` — Terragrunt deployment repos (Boilerplate-generated,
  Atlantis-applied). Groups non-major module bumps. No `dont-release` label —
  deployments still get released via Atlantis.
- `helm.json` — scoped to `charts/`, per-chart branches, appVersion tracking via
  Docker tags (custom regex manager)
- `kustomize.json` — `dont-release` labels
- `kubernetes.json` — raw K8s manifest `image:` refs under `k8s/`, `manifests/`,
  `deploy/`, `deployments/`. `dont-release` labels.
- `nix.json` — groups non-major flake inputs
- `argocd.json` — ArgoCD Applications, `dont-release`
- `tflint.json` — TFLint plugin updates (compose with `terraform`)
- `homebrew.json` — `Brewfile` formulae, `dont-release`
- `typst.json` — regex manager for `.typ` files (needs `// renovate:`
  annotations); supports both `#let` and `#import "@preview/..."` patterns
- `hugo.json` — enables `git-submodules` for Hugo theme tracking; `dont-release`
  labels
- `tool-versions.json` — regex manager for asdf `.tool-versions` (only needed
  for pure-asdf repos; `:mise` covers `.tool-versions` when `mise.toml` also
  present)

**Cross-cutting (compose as needed):**

- `ci.json` — GitHub Actions (pin digests, group non-major, **automerges trusted
  publishers** `actions/*`, `github/*`, `docker/*`), forgejo-actions support
  with `forgejo.fartlab.dev` registry, `dont-release` labels for
  Actions/Dockerfiles/config files
- `docker.json` — native `dockerfile` + `docker-compose` managers (pin digests,
  group non-major), regex manager for `docker-bake.hcl`
- `mise.json` — native `mise` manager. Auto-detects tools in `mise.toml`,
  `.mise.toml`, and `.tool-versions`. Annotations only needed for unknown tools.
- `renovate-config.json` — native `renovate-config` manager. Bumps pinned
  `extends` refs in downstream repos.
- `devcontainer.json` — native `devcontainer` manager. Updates features and base
  image in `.devcontainer/devcontainer.json`.

## Validation & CI

CI runs on every PR via `.github/workflows/validate.yml`:

1. `renovate-config-validator --strict <file>` — validates syntax, semantics,
   deprecated options against every `.json` file
2. `prettier --check "*.json"` — enforces consistent formatting

To validate locally (the validator is bundled inside the `renovate` package, so
install via `--package=renovate`):

```bash
npx --yes --package=renovate -- renovate-config-validator --strict <file>
prettier --check "*.json"
```

## Testing Presets on Real Repos

To test unmerged preset changes on a real consumer repo, append `#<branch>` to
the extends reference. Renovate will fetch the preset from that branch on its
next run:

```json
{
  "extends": [
    "github>donaldgifford/renovate-config#feat/my-branch",
    "github>donaldgifford/renovate-config:go#feat/my-branch"
  ]
}
```

After merging the PR, switch the consumer back to the unsuffixed form.

For a no-PR dry-run that shows what Renovate would do without creating anything:

```bash
npx --package=renovate -- renovate \
  --platform=github \
  --token=$(gh auth token) \
  --dry-run=full \
  --autodiscover=false \
  donaldgifford/<target-repo>
```

## Linting & Formatting

Tools managed via [mise](https://mise.jdx.dev/) (`mise.toml`):

- **Markdown:** `markdownlint-cli2`, `prettier` (proseWrap: always, 80 cols)
- **YAML:** `yamlfmt`, `yamllint`
- **JSON:** `prettier`, `renovate-config-validator`

## Conventions for Adding/Editing Presets

- **JSON** files only. `.json5` is not resolved by Renovate for the bare preset
  shorthand — stick with `.json`.
- Only `default.json` sets `extends` and global automerge. Every preset sets
  `$schema` (harmless, helps editor completion).
- **Exactly one semver label per PR** (`major`/`minor`/`patch`/`dont-release`) —
  release tooling requires it. Release-relevant ecosystem presets (go, rust,
  node, python, terraform, terragrunt, helm): group non-major with `patch`,
  majors get `minor`. Infra presets (ci, docker, mise, kustomize, kubernetes,
  argocd, tflint, homebrew, hugo, tool-versions): one rule labels **all**
  updates `dont-release`, a second rule groups non-major with **no** labels.
  Never stack `dont-release` with `patch`/`minor`.
- Runtime deps (`go`, `node`, `bun`) are excluded from group rules via
  `matchDepNames: ["!go"]`-style negation so the runtime rule's `minor` label
  doesn't stack with the group's `patch`
- **PR by default** — only set `automerge: true` for explicitly low-risk
  categories (lockfile maintenance, `@types/*`, trusted Actions publishers)
- Don't add redundant `automerge: false` — base default already blocks it
- Scope all rules with `matchManagers` to the relevant manager(s)
- Group name pattern: `"<ecosystem> <thing> (non-major)"`
- Labels: `dependencies` (base), `patch`, `minor`, `dont-release`, `security`
- Schedule: weekly Monday before 6am, `America/Detroit`
- Run `prettier --write "*.json"` after editing — CI fails otherwise

## Custom Managers

Custom regex managers need a paired `packageRules` entry that matches
`custom.regex` and scopes by `matchFileNames`, otherwise their updates land
unlabeled and ungrouped. Annotation comment is `# renovate:` (or `// renovate:`
for typst).

Prefer **native managers** where available (mise, dockerfile, docker-compose,
kubernetes, devcontainer, renovate-config, rust-toolchain, bun) over custom
regex. Custom regex is a last resort for tools without native support (typst,
`docker-bake.hcl`).

## Common Pitfalls

- `matchPackagePatterns` is deprecated — use `matchPackageNames` with glob
  (`["@types/**"]`)
- `cargoUpdate` postUpdateOption doesn't exist — use
  `rangeStrategy: "update-lockfile"`
- `managerFilePatterns` uses regex syntax wrapped in `/.../`, not glob
- `lockFileMaintenance` is **disabled by default** — a
  `matchUpdateTypes: ["lockFileMaintenance"]` packageRule does nothing unless
  the preset also sets `"lockFileMaintenance": { "enabled": true }` (top-level).
  The packageRule only labels/automerges what enablement generates.
- `terragrunt` is a separate manager from `terraform` — use `:terraform` for
  modules repos, `:terragrunt` for deployment repos, both if the repo mixes raw
  `.tf` files with `terragrunt.hcl`
- `registryUrls` cannot be combined with `matchUpdateTypes` in the same
  packageRule — split into two rules (one sets the URL, one does the
  grouping/labeling)
- The `renovate-config` manager is named `renovate-config`, not
  `renovate-config-presets` (the latter is auto-migrated but flagged)
- A custom regex manager whose `matchStrings` doesn't match the file format
  (e.g. HCL syntax against a YAML file) silently catches nothing
