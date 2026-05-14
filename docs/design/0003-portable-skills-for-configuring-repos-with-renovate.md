---
id: DESIGN-0003
title: "Portable Skills for Configuring Repos with Renovate"
status: Draft
author: Donald Gifford
created: 2026-05-13
---

<!-- markdownlint-disable-file MD025 MD041 -->

# DESIGN 0003: Portable Skills for Configuring Repos with Renovate

**Status:** Draft **Author:** Donald Gifford **Date:** 2026-05-13

<!--toc:start-->

- [Overview](#overview)
- [Goals and Non-Goals](#goals-and-non-goals)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Background](#background)
- [Detailed Design](#detailed-design)
  - [Skill Inventory](#skill-inventory)
  - [Skill: renovate-onboard](#skill-renovate-onboard)
  - [Skill: renovate-annotate](#skill-renovate-annotate)
  - [Skill: renovate-audit](#skill-renovate-audit)
  - [Skill: renovate-triage](#skill-renovate-triage)
  - [Detection Reference](#detection-reference)
  - [Annotation Reference](#annotation-reference)
  - [Packaging](#packaging)
- [API / Interface Changes](#api--interface-changes)
- [Data Model](#data-model)
- [Testing Strategy](#testing-strategy)
- [Migration / Rollout Plan](#migration--rollout-plan)
- [Open Questions](#open-questions)
- [References](#references)
<!--toc:end-->

## Overview

This document designs a set of **portable Claude Code skills** that help any
consumer repo adopt and maintain a Renovate configuration. The skills are
written to be reusable across repositories and will live in a separate
`claude-skills` repository (not in this preset repo). They target the common
pain points of configuring Renovate correctly: picking the right preset(s) to
extend, adding the `# renovate:` annotations that custom regex managers depend
on, auditing an existing config for drift and deprecated options, and triaging
open Renovate PRs by risk class.

## Goals and Non-Goals

### Goals

- Make initial Renovate adoption a one-skill operation: scan repo → write
  `renovate.json` with the right `extends`
- Make `# renovate: datasource=X depName=Y` annotation discovery automatic for
  `mise.toml`, `docker-bake.hcl`, `Chart.yaml` `appVersion`, and `.typ` files
- Catch the deprecated-options trap (`matchPackagePatterns`, `cargoUpdate`)
  before it bites consumer repos the way it bit this one
- Give humans a fast, structured way to triage a backlog of open Renovate PRs
- Be portable: skills must not depend on this preset repo being checked out
  locally; they reference its presets only via the well-known
  `github>donaldgifford/renovate-config` slug

### Non-Goals

- Maintaining the presets themselves (covered in DESIGN-0002)
- Running Renovate as a bot — these skills augment human work, not replace the
  scheduled Renovate run
- Auto-merging PRs — even the triage skill stops at categorization
- Generating Renovate cloud config or app installation steps

## Background

Setting up a repo for Renovate has a steep first-time cliff:

1. **Picking presets.** Which `extends` entries from
   `github>donaldgifford/renovate-config` apply? A repo with `go.mod`,
   `mise.toml`, `Dockerfile`, and `.github/workflows/` needs four. The mapping
   is mechanical but easy to get wrong.
2. **Annotations.** Custom regex managers (`mise`, `docker`, `helm`, `typst`)
   only fire when the source file has the right `# renovate:` comment. A
   `mise.toml` without `# renovate: datasource=github-releases depName=...` will
   never update those tools. There is no error — just silence.
3. **Datasource mapping confusion.** For mise, the `datasource` and `depName`
   fields don't always match what's in the source file. `yamlfmt = "0.20.0"`
   needs `datasource=github-releases depName=google/yamlfmt`; the GitHub org
   isn't visible in the line itself. Humans get this wrong.
4. **Existing configs drift.** Repos still using `matchPackagePatterns`,
   `matchPaths`, or `cargoUpdate` will run on current Renovate but the rules are
   deprecated and may silently misbehave.
5. **Renovate PR review fatigue.** A repo with 15 open Renovate PRs gets
   ignored. Humans need a triage layer that surfaces the 2 that matter and
   batches the rest.

Each of these is a discrete, repeatable task with a clear input/output — ideal
skill territory.

## Detailed Design

### Skill Inventory

| Skill               | Trigger                                                  | What It Does                                                                                                          |
| ------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `renovate-onboard`  | Greenfield Renovate adoption on a repo                   | Scans repo for manifest files, emits `renovate.json` with appropriate `extends` array                                 |
| `renovate-annotate` | Repo has manifest files but custom managers won't fire   | Adds `# renovate:` annotations to `mise.toml`, `docker-bake.hcl`, `Chart.yaml`, `.typ` files                          |
| `renovate-audit`    | Existing repo with Renovate config that may have drifted | Flags deprecated options, missing annotations, redundant overrides, presets that could be added or removed            |
| `renovate-triage`   | Repo has open Renovate PRs needing review                | Classifies open PRs by risk class (runtime vs dev, major vs non-major, automergeable category) and surfaces a summary |

### Skill: renovate-onboard

**Trigger:** "set up Renovate for this repo", "add a `renovate.json`",
`/renovate-onboard`.

**Workflow:**

1. **Scan repo root and subdirectories** for ecosystem signals (see
   [Detection Reference](#detection-reference)).
2. **Map signals to presets.** Build the `extends` array starting with the base
   preset, then ecosystem presets, then cross-cutting presets.
3. **Check for custom-manager-required files.** If `mise.toml`,
   `docker-bake.hcl`, `Chart.yaml`, or `.typ` files are detected, surface a
   prompt: "Also run `renovate-annotate` after this to add the required
   `# renovate:` comments."
4. **Emit `renovate.json`:**

   ```json
   {
     "$schema": "https://docs.renovatebot.com/renovate-schema.json",
     "extends": [
       "github>donaldgifford/renovate-config",
       "github>donaldgifford/renovate-config:<ecosystem>"
     ]
   }
   ```

5. **Refuse to overwrite** an existing `renovate.json` without explicit
   confirmation. If one exists, suggest running `renovate-audit` instead.

**Scope assumption: `main` is the default branch.** The skill targets the
default branch and does not configure `baseBranches`. Repos with non-`main`
default branches or multi-branch Renovate setups are out of scope — those are
rare enough to warrant hand-editing rather than skill complexity.

**Output:** the proposed `renovate.json`, a list of detected signals, and a
follow-up checklist (annotate, configure Renovate app/webhook).

### Skill: renovate-annotate

**Trigger:** "add Renovate annotations", `/renovate-annotate`, or auto-suggested
by `renovate-onboard` and `renovate-audit`.

**Workflow:**

1. **Find files needing annotations** (mise/asdf, docker-bake, helm, typst — see
   [Annotation Reference](#annotation-reference)).
2. **For each file, for each candidate line:**
   - Skip if a `# renovate:` (or `// renovate:`) comment already sits directly
     above the line
   - Otherwise, infer `datasource` + `depName`:
     - **mise tools:** look up the tool in `references/mise-datasource-map.json`
       (baked-in map of common tools → their `datasource` + `depName`). Unknown
       tools prompt the user.
     - **docker-bake variables:** scan the rendered docker bake target for
       `image = "..."` references and extract the image name.
     - **Helm `appVersion`:** look at the chart's existing `image:` field in
       templates; surface candidate image names for confirmation.
     - **Typst:** if `#import "@preview/foo:x.y.z"`, `depName=foo` and
       `datasource=...` per typst registry conventions.
3. **Insert the annotation** as a comment immediately above the target line.
4. **Report:** files modified, annotations added, annotations the skill couldn't
   infer (left for human).

**Why this matters and what it must get right:** the annotation is what makes
the custom regex manager work. Wrong `datasource` → Renovate queries the wrong
registry → no updates. Wrong `depName` → 404s.

**Default behavior: prompt on ambiguity.** Whenever the skill cannot confidently
infer both `datasource` and `depName` from the file plus the baked-in reference
data, it asks the user rather than guessing. A `--non-interactive` flag is
accepted (e.g. for scripted use) and causes ambiguity to error out instead of
prompting — the skill never silently fills in a guess.

**Reference data (baked into the skill, loaded on demand):**

- `references/mise-datasource-map.json` — structured map of common mise tools to
  their `datasource` + `depName`. JSON rather than markdown because the skill
  loads it as data (lookup keyed on the tool name) and humans can still edit it
  directly. Updates ship via a new release of the skill.
- `references/annotation-formats.md` — exact comment syntax for each file type
- `references/typst-package-conventions.md` — typst-specific notes

### Skill: renovate-audit

**Trigger:** "audit my Renovate config", `/renovate-audit`.

**Workflow:**

1. **Read `renovate.json`** (or `.json5` / `.renovaterc` / `package.json`
   `renovate` key — handle all the valid locations).
2. **Static checks:**
   - **Deprecated options:** `matchPackagePatterns`, `matchPaths`, `cargoUpdate`
     in `postUpdateOptions`, `regexManagers` (renamed to `customManagers`)
   - **Schema validity:** valid `$schema` URL, JSON5 syntax if applicable
   - **Redundant `automerge: false`:** if extending
     `github>donaldgifford/renovate-config`, the base default already blocks
     automerge
3. **Coverage checks:**
   - Detected manifest files vs. extended presets — flag a repo with
     `Cargo.toml` but no `:rust` extend, etc.
   - Custom-manager files present (`mise.toml`, etc.) — verify the corresponding
     cross-cutting preset is extended AND the annotations exist
4. **Drift checks:**
   - Local `packageRules` that duplicate preset behavior (e.g., a rule grouping
     non-major `gomod` updates when `:go` already does this)
5. **Run `renovate-config-validator --strict`** if available locally (via
   `npx --package=renovate`); otherwise skip with a note.
6. **Report findings** with severity and remediation hint:

   ```
   audit report:
     error   matchPackagePatterns at L18 — replace with matchPackageNames + glob
     warn    Cargo.toml detected but :rust preset not extended
     warn    mise.toml has 3 entries without `# renovate:` annotations
     info    local rule at L42 duplicates :go preset behavior — can be removed
   ```

### Skill: renovate-triage

**Trigger:** "triage open Renovate PRs", `/renovate-triage`.

**Workflow:**

1. **Fetch open PRs** authored by Renovate
   (`gh pr list --author renovate[bot] --state open --json ...`)
2. **Categorize each PR** using its labels (set by our presets) and title:
   - **Auto-merge candidates:** PRs whose updates fall in our opt-in categories
     (lockfile maintenance, `@types/*`, trusted Actions publishers) but somehow
     didn't auto-merge — surface why
   - **Low-risk batch:** non-major, dev-tool, infra (`dont-release`) updates
   - **Needs review:** major bumps (`minor` label) and runtime deps
   - **Security:** any PR with the `security` label — surface first
3. **Output a structured summary:**

   ```
   triage summary (12 PRs):
     security (1):
       #142  bump axios 1.6.0 → 1.6.8  (CVE-2024-12345)
     needs review (3):
       #138  bump go 1.22 → 1.23  (runtime)
     low-risk batch (7):
       node packages (non-major), terraform modules (non-major)
     auto-merge candidates that didn't (1):
       #119  @types/node 20.10.0 → 20.10.5  — investigate why automerge skipped
   ```

The skill **does not merge or close** PRs. It surfaces signal for a human to act
on.

**Optional: post the summary to a GitHub issue.** With
`--post-to-issue <number>` (or `--post-to-issue new` to open a fresh one), the
skill writes the summary as a comment on the specified issue. This is useful for
repos that use an ongoing "Renovate triage" issue as a working doc, but is
strictly opt-in — default behavior is to print to the terminal only.

### Detection Reference

| Signal File / Path                                | Implies Preset |
| ------------------------------------------------- | -------------- |
| `go.mod`                                          | `:go`          |
| `Cargo.toml`                                      | `:rust`        |
| `package.json`, `bun.lockb`, `yarn.lock`          | `:node`        |
| `*.tf`, `*.tofu`, `terragrunt.hcl`                | `:terraform`   |
| `.tflint.hcl`                                     | `:tflint`      |
| `charts/*/Chart.yaml`                             | `:helm`        |
| `kustomization.yaml`                              | `:kustomize`   |
| `flake.nix`                                       | `:nix`         |
| `Brewfile`                                        | `:homebrew`    |
| `*.typ`                                           | `:typst`       |
| `argocd/`, `apps/*.yaml` with `kind: Application` | `:argocd`      |
| `.github/workflows/*.yml`                         | `:ci`          |
| `Dockerfile`, `docker-bake.hcl`                   | `:docker`      |
| `mise.toml`, `.mise.toml`, `.tool-versions`       | `:mise`        |

Always include the base preset (`github>donaldgifford/renovate-config`) as the
first entry.

### Annotation Reference

| File                           | Annotation Syntax                            | Notes                                                              |
| ------------------------------ | -------------------------------------------- | ------------------------------------------------------------------ |
| `mise.toml` / `.tool-versions` | `# renovate: datasource=<ds> depName=<repo>` | `<ds>` is usually `github-releases`; `<repo>` is the GitHub slug   |
| `docker-bake.hcl`              | `# renovate: image=<image-name>`             | `<image-name>` matches the rendered `image = "..."` reference      |
| `Chart.yaml` `appVersion`      | `# renovate: image=<image-name>`             | Tracks the Docker tag the chart deploys                            |
| `.typ`                         | `// renovate: datasource=<ds> depName=<pkg>` | `#import "@preview/foo:x"` and `#let foo = "x.y.z"` both supported |

The skill carries a curated mise tool → GitHub slug map (yamlfmt →
`google/yamlfmt`, etc.) because this is the most common confusion point. Unknown
tools default to prompting the user.

### Packaging

Skills live in a separate `donaldgifford/claude-skills` repository (existing).
Layout will mirror the claude-md plugin's:

```
skills/
  renovate-onboard/
    SKILL.md
    references/
      detection-signals.md
      preset-mapping.md
  renovate-annotate/
    SKILL.md
    references/
      mise-datasource-map.json
      annotation-formats.md
      typst-package-conventions.md
  renovate-audit/
    SKILL.md
    references/
      deprecated-options.md
      audit-checklist.md
  renovate-triage/
    SKILL.md
    references/
      label-taxonomy.md
```

Distribution is via the existing plugin marketplace mechanism in the
`claude-skills` repo.

## API / Interface Changes

These skills are pure tooling against external state (the consumer repo's files
and GitHub API). No changes to the preset repo's `.json5` files. No required
changes to consumer repos beyond what the skills propose.

Slash commands surfaced once installed:

- `/renovate-onboard`
- `/renovate-annotate`
- `/renovate-audit`
- `/renovate-triage`

## Data Model

No persisted state. Each skill is stateless and operates on the working tree of
the consumer repo plus, where needed, GitHub API responses (PR list, repo
metadata).

## Testing Strategy

Each skill carries a `fixtures/` directory under its `references/` (or a sibling
`tests/`) containing minimal example repo trees. Test runs invoke the skill
against the fixture and diff the output against a golden file.

| Skill               | Fixture Type                                                 |
| ------------------- | ------------------------------------------------------------ |
| `renovate-onboard`  | Stripped-down repo trees per ecosystem combination           |
| `renovate-annotate` | `mise.toml`, `docker-bake.hcl`, `Chart.yaml`, `.typ` samples |
| `renovate-audit`    | Known-bad `renovate.json` files with documented violations   |
| `renovate-triage`   | Mocked `gh pr list` JSON output                              |

The deprecation list (`matchPackagePatterns`, etc.) used by `renovate-audit` is
the same canonical list referenced by DESIGN-0002's `preset-reviewer`. We
extract it to a single source under `references/deprecated-options.md` that both
skill sets can pull from.

## Migration / Rollout Plan

1. **Phase 1 — `renovate-onboard` and `renovate-annotate`.** Together these
   handle the greenfield case end-to-end. Ship first.
2. **Phase 2 — `renovate-audit`.** Once we have onboarded enough repos to have
   real config drift to detect, audit becomes valuable.
3. **Phase 3 — `renovate-triage`.** Lowest urgency, highest convenience.

Each phase is its own PR in `claude-skills` and gets a corresponding entry in
that repo's marketplace manifest.

## Open Questions

_None at this time. Resolved during design review:_

- _`renovate-onboard` targets `main` only — non-default branches are out of
  scope._
- _`renovate-annotate` prompts by default on ambiguity; a `--non-interactive`
  flag errors instead of guessing._
- _Mise datasource map is baked into the skill as
  `references/mise-datasource-map.json` and ships with each skill release._
- _`renovate-triage` supports an opt-in `--post-to-issue` flag to comment the
  summary onto a GitHub issue._

## References

- [Renovate Shareable Config Presets](https://docs.renovatebot.com/config-presets/)
- [Renovate Custom Managers](https://docs.renovatebot.com/modules/manager/regex/)
- [Renovate Datasources](https://docs.renovatebot.com/modules/datasource/)
- [Claude Code Skills documentation](https://docs.claude.com/en/docs/claude-code/skills)
- DESIGN-0001: Shareable Renovate Preset Architecture
- DESIGN-0002: Local Skills for Maintaining Renovate Presets
