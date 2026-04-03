---
id: DESIGN-0001
title: "Shareable Renovate Preset Architecture"
status: Draft
author: Donald Gifford
created: 2026-04-03
---
<!-- markdownlint-disable-file MD025 MD041 -->

# DESIGN 0001: Shareable Renovate Preset Architecture

**Status:** Draft
**Author:** Donald Gifford
**Date:** 2026-04-03

<!--toc:start-->
- [Overview](#overview)
- [Goals and Non-Goals](#goals-and-non-goals)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Background](#background)
- [Detailed Design](#detailed-design)
- [API / Interface Changes](#api--interface-changes)
- [Data Model](#data-model)
- [Testing Strategy](#testing-strategy)
- [Migration / Rollout Plan](#migration--rollout-plan)
- [Open Questions](#open-questions)
- [References](#references)
<!--toc:end-->

## Overview

This document defines the architecture and conventions for
`github>donaldgifford/renovate-config`, a centralized repository of shareable
Renovate presets. Consumer repos compose presets via `"extends"` in their
`renovate.json` to get consistent dependency management across all
repositories without duplicating configuration.

## Goals and Non-Goals

### Goals

- Single source of truth for Renovate configuration across all repos
- Composable presets that can be mixed and matched per repo's tech stack
- Consistent labeling, grouping, and automerge behavior everywhere
- Easy onboarding: a new repo picks 2-4 presets and is fully configured
- Clear conventions so new presets follow the same patterns

### Non-Goals

- Per-repo customization within this repo (repos can override locally)
- Managing Renovate bot deployment/hosting (that's a separate concern)
- Covering every possible Renovate manager (only ecosystems we use)

## Background

We maintain repositories across multiple ecosystems: Go, Rust, Node/TypeScript
(bun and yarn), Terraform/OpenTofu (with Terragrunt and Boilerplate), Helm
charts, Kustomize, Docker (with docker-bake), and GitHub Actions. Each repo
previously managed its own Renovate config, leading to drift, inconsistency,
and duplicated effort. This repo centralizes those configs into composable
presets.

## Detailed Design

### Preset Architecture

Presets are organized into three layers:

```
Layer 1: Base        default.json5
Layer 2: Ecosystem   go / rust / node / terraform / helm / kustomize
Layer 3: Cross-cut   ci / docker / mise
```

**Layer 1 (Base)** provides global defaults that every repo needs:
`config:recommended`, schedule, automerge policy, PR limits, vulnerability
alerts, and the `dependencies` label.

**Layer 2 (Ecosystem)** provides language/tool-specific configuration: manager
options, post-update commands, grouping rules, and ecosystem-specific labels.
A repo picks exactly one ecosystem preset for its primary language.

**Layer 3 (Cross-cutting)** provides concerns that apply across ecosystems:
CI labeling, Docker digest pinning, mise tool version tracking. Repos compose
as many of these as needed.

### Composition Model

Consumer repos compose presets via `"extends"`. Renovate merges them in order,
with later presets overriding earlier ones for conflicting keys. `packageRules`
arrays are concatenated (not overridden), so rules from all presets apply.

Example for a Go repo with Docker and mise:

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

### Preset Inventory

| Preset | Layer | Managers / Scope | Key Behavior |
|--------|-------|-----------------|--------------|
| `default.json5` | Base | All | `config:recommended`, weekly Mon schedule (America/Detroit), automerge non-major, vuln alerts, PR limits (5 concurrent / 2 hourly) |
| `go.json5` | Ecosystem | `gomod` | `gomodTidy`, `gomodUpdateImportPaths`, group non-major, no automerge on toolchain bumps, `dont-release` for `.goreleaser.yml`/`.golangci.yml` |
| `rust.json5` | Ecosystem | `cargo` | `rangeStrategy: update-lockfile`, group non-major, automerge lockfile maintenance |
| `node.json5` | Ecosystem | `npm`, `nodenv`, `nvm` | `rangeStrategy: update-lockfile`, `yarnDedupeHighest`, group `@types/*` separately, no automerge on Node version bumps |
| `terraform.json5` | Ecosystem | `terraform`, `terragrunt` | Pin provider digests, scoped lockfile maintenance, group providers and modules separately, regex manager for boilerplate variables |
| `helm.json5` | Ecosystem | `helmv3` | Scoped to `charts/`, per-chart branch prefixes and commit messages, no automerge, appVersion tracking via Docker tags |
| `kustomize.json5` | Ecosystem | `kustomize` | No automerge, group non-major image bumps, `dont-release` labels |
| `ci.json5` | Cross-cut | `github-actions`, `dockerfile`, file patterns | Pin Actions digests, group non-major Actions, no automerge on major Actions bumps, `dont-release` labels for Actions/Dockerfiles/config |
| `docker.json5` | Cross-cut | `dockerfile`, custom regex | Pin digests in Dockerfiles, regex manager for `docker-bake.hcl` version variables |
| `mise.json5` | Cross-cut | custom regex | Regex manager for pinned versions in `mise.toml`/`.mise.toml`, requires `# renovate:` annotations |

### Labeling Convention

All presets follow a consistent labeling scheme tied to semantic release:

| Label | Applied When |
|-------|-------------|
| `dependencies` | Every Renovate PR (set in `default.json5`) |
| `patch` | Non-major dependency updates (minor, patch, digest, lockfile) |
| `minor` | Major dependency updates or runtime version bumps |
| `dont-release` | Infrastructure-only changes (CI, Docker, config files, mise tools) |
| `security` | Vulnerability alert PRs |

### Grouping Convention

Every ecosystem preset groups non-major updates into a single PR to reduce
noise. Major bumps always get individual PRs for visibility. The group name
follows the pattern: `<ecosystem> <thing> (non-major)`.

### Automerge Convention

- **Non-major dependency updates**: automerge (from `default.json5`)
- **Major dependency updates**: never automerge (from `default.json5`)
- **Runtime/toolchain version bumps** (Go, Node): never automerge
- **Infrastructure manifests** (Helm, Kustomize): never automerge
- **Lockfile maintenance** (Rust, Terraform): automerge

### Custom Manager Convention

For tools without native Renovate managers (mise, docker-bake, boilerplate),
we use regex custom managers with `# renovate:` comment annotations. The
annotation tells Renovate the datasource and dependency name:

```toml
# renovate: datasource=github-releases depName=google/yamlfmt
yamlfmt = "0.20.0"
```

```hcl
# renovate: image=golang
variable "GO_IMAGE_TAG" { default = "1.23.2-bookworm" }
```

This pattern is consistent across all custom managers and is self-documenting
in the consuming repo's source files.

## API / Interface Changes

Consumer repos interact with this repo solely through `"extends"` in their
`renovate.json`. The "API" is the set of preset names (file stems):

```
github>donaldgifford/renovate-config           → default.json5
github>donaldgifford/renovate-config:<name>     → <name>.json5
```

Adding a new preset is backwards-compatible. Changing an existing preset
affects all consumers on their next Renovate run.

## Testing Strategy

Renovate configs are validated using:

1. `renovate-config-validator --strict` — validates syntax, semantics, and
   deprecated options against every `.json5` file
2. `prettier --check "*.json5"` — enforces consistent formatting

CI runs both on every PR via `.github/workflows/validate.yml`.

## Adding a New Preset

When adding a new ecosystem or cross-cutting preset, follow these rules:

1. **One file per concern** — don't mix ecosystems or bundle unrelated managers
2. **JSON5 format** — use comments to explain non-obvious rules
3. **No `$schema` or `extends`** — only `default.json5` sets these
4. **Include these package rules at minimum**:
   - Group non-major updates with `addLabels: ["patch"]`
   - Major bumps with `addLabels: ["minor"]`
   - Automerge override if the ecosystem warrants it
5. **Use `matchManagers`** to scope all rules to the relevant manager(s)
6. **Comment style**: `// --- Description ---` for rule separators
7. **Group name pattern**: `"<ecosystem> <thing> (non-major)"`
8. **Custom managers**: use `# renovate:` annotation pattern, document the
   expected comment format in the preset comments
9. **Update README.md**: add usage example and preset table entry
10. **Update CLAUDE.md**: add preset description

## Migration / Rollout Plan

For existing repos already using Renovate:

1. Replace inline config with `"extends"` pointing to this repo's presets
2. Remove any duplicated `packageRules` now covered by presets
3. Keep repo-specific overrides (e.g., `ignoreDeps`, custom `packageRules`)
4. Merge and verify the Dependency Dashboard reflects the new config

For new repos:

1. Choose the ecosystem preset(s) matching the repo's tech stack
2. Add cross-cutting presets as needed (`ci`, `docker`, `mise`)
3. Create `renovate.json` with the `"extends"` array
4. Merge and review the onboarding PR

## Open Questions

- Should we add a `python.json5` preset for future Python repos?
- Should lockfile maintenance be scoped per-preset or enabled globally in
  `default.json5`?

## References

- [Renovate Shareable Config Presets](https://docs.renovatebot.com/config-presets/)
- [Renovate Configuration Options](https://docs.renovatebot.com/configuration-options/)
- [Renovate Managers](https://docs.renovatebot.com/modules/manager/)
- [Renovate Custom Managers](https://docs.renovatebot.com/modules/manager/regex/)
