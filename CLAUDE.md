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

Presets are JSON5 files in the repo root, organized in three layers:

**Base:** `default.json5` — global defaults including `automerge: false`.

**Ecosystem (pick one or more):**

- `go.json5` — `gomodTidy`, `gomodUpdateImportPaths`, `dont-release` for
  `.goreleaser.yml`/`.golangci.yml`
- `rust.json5` — `update-lockfile`, **automerges lockfile maintenance**
- `node.json5` — `update-lockfile`, `yarnDedupeHighest`, **automerges
  `@types/*`**
- `terraform.json5` — terragrunt support, pin provider digests, **automerges
  lockfile maintenance**
- `helm.json5` — scoped to `charts/`, per-chart branches, appVersion tracking
  via Docker tags (custom regex manager)
- `kustomize.json5` — `dont-release` labels
- `nix.json5` — groups non-major flake inputs
- `argocd.json5` — ArgoCD Applications, `dont-release`
- `tflint.json5` — TFLint plugin updates (compose with `terraform`)
- `homebrew.json5` — `Brewfile` formulae, `dont-release`
- `typst.json5` — regex manager for `.typ` files (needs `// renovate:`
  annotations); supports both `#let` and `#import "@preview/..."` patterns
- `hugo.json5` — enables `git-submodules` for Hugo theme tracking;
  `dont-release` labels
- `tool-versions.json5` — regex manager for asdf `.tool-versions` (needs
  `# renovate:` annotations); `dont-release` labels

**Cross-cutting (compose as needed):**

- `ci.json5` — Actions (pin digests, group non-major, **automerges trusted
  publishers** `actions/*`, `github/*`, `docker/*`), `dont-release` labels for
  Actions/Dockerfiles/config files
- `docker.json5` — pin Dockerfile digests, regex manager for `docker-bake.hcl`
  (with grouping/labels for those updates)
- `mise.json5` — regex manager for `mise.toml` / `.mise.toml` only (needs
  `# renovate:` annotations). `.tool-versions` is **not** covered here — compose
  `tool-versions.json5` for asdf-style tool files.

## Validation & CI

CI runs on every PR via `.github/workflows/validate.yml`:

1. `renovate-config-validator --strict <file>` — validates syntax, semantics,
   deprecated options against every `.json5` file
2. `prettier --check "*.json5"` — enforces consistent formatting

To validate locally (the validator is bundled inside the `renovate` package, so
install via `--package=renovate`):

```bash
npx --yes --package=renovate -- renovate-config-validator --strict <file>
prettier --check "*.json5"
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
- **JSON5:** `prettier`, `renovate-config-validator`

## Conventions for Adding/Editing Presets

- **JSON5** with `// --- Description ---` comment separators
- Only `default.json5` sets `$schema`, `extends`, and global automerge
- Every preset must include: group non-major (`addLabels: ["patch"]`) and major
  bumps (`addLabels: ["minor"]`)
- **PR by default** — only set `automerge: true` for explicitly low-risk
  categories (lockfile maintenance, `@types/*`, trusted Actions publishers)
- Don't add redundant `automerge: false` — base default already blocks it
- Scope all rules with `matchManagers` to the relevant manager(s)
- Group name pattern: `"<ecosystem> <thing> (non-major)"`
- Labels: `dependencies` (base), `patch`, `minor`, `dont-release`, `security`
- Schedule: weekly Monday before 6am, `America/Detroit`
- Run `prettier --write "*.json5"` after editing — CI fails otherwise

## Custom Managers

Custom regex managers need a paired `packageRules` entry that matches
`custom.regex` and scopes by `matchFileNames`, otherwise their updates land
unlabeled and ungrouped. Annotation comment is `# renovate:` (or `// renovate:`
for typst).

```json5
{
  customManagers: [
    {
      customType: "regex",
      managerFilePatterns: ["/(^|/)mise\\.toml$/"], // regex, not glob
      matchStrings: ["..."],
      // ...
    },
  ],
  packageRules: [
    {
      matchManagers: ["custom.regex"],
      matchFileNames: ["mise.toml", ".mise.toml"], // scope to this preset's files
      matchUpdateTypes: ["minor", "patch", "digest"],
      groupName: "mise tools (non-major)",
      addLabels: ["patch"],
    },
  ],
}
```

## Common Pitfalls

- `matchPackagePatterns` is deprecated — use `matchPackageNames` with glob
  (`["@types/**"]`)
- `cargoUpdate` postUpdateOption doesn't exist — use
  `rangeStrategy: "update-lockfile"`
- `managerFilePatterns` uses regex syntax wrapped in `/.../`, not glob
- `lockFileMaintenance` is configured per packageRule via
  `matchUpdateTypes: ["lockFileMaintenance"]`, not as a global key
- `terragrunt` is a separate manager from `terraform` — match both when rules
  apply to either
- A custom regex manager whose `matchStrings` doesn't match the file format
  (e.g. HCL syntax against a YAML file) silently catches nothing
