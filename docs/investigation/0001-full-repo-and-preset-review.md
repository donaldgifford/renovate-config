---
id: INV-0001
title: "Full Repo and Preset Review"
status: Open
author: Donald Gifford
created: 2026-07-07
---

<!-- markdownlint-disable-file MD025 MD041 -->

# INV 0001: Full Repo and Preset Review

**Status:** Open **Author:** Donald Gifford **Date:** 2026-07-07

<!--toc:start-->

- [Question](#question)
- [Hypothesis](#hypothesis)
- [Context](#context)
- [Approach](#approach)
- [Environment](#environment)
- [Findings](#findings)
  - [P1 — Bugs (behavior differs from intent)](#p1--bugs-behavior-differs-from-intent)
    - [1.1 Lockfile-maintenance automerge never fires (rust, terraform)](#11-lockfile-maintenance-automerge-never-fires-rust-terraform)
    - [1.2 Label stacking violates the "exactly one semver label" contract](#12-label-stacking-violates-the-exactly-one-semver-label-contract)
    - [1.3 docs/examples/terraform-terragrunt-project.json broken by the split](#13-docsexamplesterraform-terragrunt-projectjson-broken-by-the-split)
    - [1.4 go.json dead rule for .goreleaser.yml/.golangci.yml](#14-gojson-dead-rule-for-goreleaserymlgolangciyml)
    - [1.5 mise.toml has a copy-paste annotation bug](#15-misetoml-has-a-copy-paste-annotation-bug)
  - [P2 — Inconsistencies](#p2--inconsistencies)
    - [2.1 helm.json appVersion rule mislabels majors](#21-helmjson-appversion-rule-mislabels-majors)
    - [2.2 helm.json appVersion uses versioningTemplate: "semver"](#22-helmjson-appversion-uses-versioningtemplate-semver)
    - [2.3 tflint.json and homebrew.json majors lose dont-release](#23-tflintjson-and-homebrewjson-majors-lose-dont-release)
    - [2.4 node.json runtime rule misses the bun-version manager](#24-nodejson-runtime-rule-misses-the-bun-version-manager)
    - [2.5 mise.json redundant "enabled": true](#25-misejson-redundant-enabled-true)
    - [2.6 terraform.json pinDigests: true is questionable](#26-terraformjson-pindigests-true-is-questionable)
    - [2.7 Forgejo rules unverified against a real repo](#27-forgejo-rules-unverified-against-a-real-repo)
  - [P3 — Repo hygiene and infrastructure](#p3--repo-hygiene-and-infrastructure)
    - [3.1 This repo doesn't dogfood Renovate](#31-this-repo-doesnt-dogfood-renovate)
    - [3.2 validate.yml doesn't validate docs/examples/\*.json](#32-validateyml-doesnt-validate-docsexamplesjson)
    - [3.3 Examples dir is missing 7 of 22 presets](#33-examples-dir-is-missing-7-of-22-presets)
    - [3.4 Design docs still describe the .json5 world](#34-design-docs-still-describe-the-json5-world)
    - [3.5 scripts/ is untracked](#35-scripts-is-untracked)
    - [3.6 validate.yml installs unpinned renovate on every run](#36-validateyml-installs-unpinned-renovate-on-every-run)
  - [Verified-good](#verified-good)
- [Conclusion](#conclusion)
- [Recommendation](#recommendation)
- [References](#references)
<!--toc:end-->

## Question

After the `.json5` → `.json` migration, native-manager refactor, and the
terraform/terragrunt split, do all 22 presets actually behave as documented in
CLAUDE.md/README, and is the repo's own infrastructure (CI, examples, design
docs) consistent with the current state?

## Hypothesis

The presets validate cleanly (CI is green), but semantic bugs that
`renovate-config-validator` cannot catch — dead rules, label conflicts,
never-firing automerge — have likely accumulated across the recent large
refactors. Validator-green ≠ behavior-correct.

## Context

**Triggered by:** post-merge review after PR #11 (native managers, drop .json5)
and PR #12 (python preset). The repo now has 22 presets, a labels workflow,
examples dir, and three design docs written across several sessions — worth a
full-sweep audit before pointing more consumer repos at it.

## Approach

1. Read every preset `.json` in the repo root against the documented conventions
   (CLAUDE.md "Conventions" + "Common Pitfalls")
2. Cross-check labels each rule emits vs. the `pr-labels.yml` "exactly one
   semver label" contract
3. Trace automerge opt-ins to confirm the update types they match can actually
   occur
4. Check repo infra: validate workflow coverage, examples dir accuracy, design
   doc staleness, untracked files, dogfooding
5. Verify mise.toml annotations against the native mise manager behavior

## Environment

| Component    | Version / Value                     |
| ------------ | ----------------------------------- |
| Renovate     | 43.x (validator via `npx renovate`) |
| Repo HEAD    | `afadebb` (feat: Add python preset) |
| Preset count | 22 `.json` files in repo root       |

## Findings

### P1 — Bugs (behavior differs from intent)

#### 1.1 Lockfile-maintenance automerge never fires (rust, terraform)

`rust.json` and `terraform.json` automerge
`matchUpdateTypes: ["lockFileMaintenance"]`, and both README and DESIGN-0001
advertise this. But **lockfile maintenance is disabled by default in Renovate**
and nothing in the preset chain enables it. `config:recommended` does not turn
it on. The packageRules can only relabel/automerge updates that are generated —
none are.

**Fix:** add a scoped enablement in each preset that advertises it:

```json
{
  "lockFileMaintenance": { "enabled": true }
}
```

(top-level in `rust.json`/`terraform.json`, or per-manager via packageRules if
we want it narrower). Schedule inherits the weekly Monday window.

#### 1.2 Label stacking violates the "exactly one semver label" contract

`pr-labels.yml` (and the same convention consumers use for release automation)
requires **exactly one** of `major`/`minor`/`patch`/ `dont-release`. Multiple
presets stack two:

| Preset               | Rule collision                                                                         | Resulting labels       |
| -------------------- | -------------------------------------------------------------------------------------- | ---------------------- |
| `kustomize.json`     | rule 1 adds `dont-release` to **all** kustomize updates; rules 2/3 add `patch`/`minor` | `dont-release`+`patch` |
| `argocd.json`        | same pattern                                                                           | `dont-release`+`patch` |
| `ci.json`            | rule 1 adds `dont-release` to all github-actions; rule 2 adds `patch` to non-major     | `dont-release`+`patch` |
| `tflint.json`        | non-major rule adds `["patch", "dont-release"]` in one rule                            | both                   |
| `homebrew.json`      | same                                                                                   | both                   |
| `mise.json`          | same (`["patch", "dont-release"]`, `["minor", "dont-release"]`)                        | both                   |
| `docker.json`        | same                                                                                   | both                   |
| `kubernetes.json`    | same                                                                                   | both                   |
| `tool-versions.json` | same                                                                                   | both                   |
| `hugo.json`          | same                                                                                   | both                   |
| `go.json`            | Go toolchain minor bump matches group rule (`patch`) **and** runtime rule (`minor`)    | `patch`+`minor`        |
| `node.json`          | node/bun minor bump matches group rule (`patch`) **and** runtime rule (`minor`)        | `patch`+`minor`        |

**Decision needed:** either (a) `dont-release` becomes the _only_ semver label
on infra PRs — drop `patch`/`minor` from those rules; or (b) the label check
treats `dont-release` as overriding. (a) is cleaner and requires no
consumer-side changes. For the go/node runtime double-match, add
`matchUpdateTypes: ["major", "minor"]` (or exclude the runtime deps from the
group rule via `matchDepNames: ["!go"]` style excludes).

#### 1.3 `docs/examples/terraform-terragrunt-project.json` broken by the split

The example still extends only `:terraform`, but since PR #11 the terraform
preset **no longer matches the terragrunt manager**. A Terragrunt repo using
this example gets no grouping/labels on `terragrunt.hcl` updates. The example
README row still says "Terraform/OpenTofu with Terragrunt and Boilerplate".

**Fix:** add `:terragrunt` to that example (or split into two examples matching
the modules-repo vs deployment-repo distinction).

#### 1.4 `go.json` dead rule for `.goreleaser.yml`/`.golangci.yml`

```json
{
  "matchFileNames": [".goreleaser.yml", ".golangci.yml"],
  "addLabels": ["dont-release"]
}
```

No manager in the preset chain extracts dependencies from those files, so this
rule matches nothing. It also violates our own "scope all rules with
matchManagers" convention. Either delete it, or make it real with a custom regex
manager that tracks tool versions in those files (e.g. golangci-lint plugin
versions). Also: only matches `.yml`, not `.yaml` variants.

#### 1.5 `mise.toml` has a copy-paste annotation bug

```toml
# renovate: datasource=github-releases depName=donaldgifford/makefmt
"github:donaldgifford/forge" = "latest"
```

`forge` is annotated with `depName=donaldgifford/makefmt`. Harmless today
(pinned to `latest`, so nothing tracks it), but the first time it's pinned
Renovate would bump forge using makefmt's releases. With the native mise manager
now in use, `github:` backend tools may not need annotations at all — verify and
clean up.

### P2 — Inconsistencies

#### 2.1 `helm.json` appVersion rule mislabels majors

The `custom.regex` packageRule applies `addLabels: ["patch"]` with **no
matchUpdateTypes filter** — a major appVersion bump gets labeled `patch`. Every
other preset splits non-major/major. Split into two rules per convention.

#### 2.2 `helm.json` appVersion uses `versioningTemplate: "semver"`

The docker-bake regex manager in `docker.json` uses `"docker"` versioning for
image tags; helm's appVersion tracker uses `"semver"` for the same kind of value
(a Docker tag). Docker tags aren't guaranteed semver-parseable
(`1.2.3-bookworm`). Align on `"docker"` versioning.

#### 2.3 `tflint.json` and `homebrew.json` majors lose `dont-release`

Non-major rules add `["patch", "dont-release"]`, but the major rules add only
`["minor"]`. A major TFLint-plugin or Brewfile bump is still dev tooling/infra —
it should keep `dont-release` (subject to the 1.2 decision on label stacking).

#### 2.4 `node.json` runtime rule misses the `bun-version` manager

Rule 4 matches `["nodenv", "nvm", "npm", "bun"]` for node/bun runtime bumps.
Renovate also has a `bun-version` manager for `.bun-version` files — add it to
the list for repos that pin bun that way.

#### 2.5 `mise.json` redundant `"enabled": true`

The native mise manager is enabled by default; `"mise": {"enabled": true}` is a
no-op. Harmless, but the convention says don't state defaults (same reason we
don't write `automerge: false` in presets).

#### 2.6 `terraform.json` `pinDigests: true` is questionable

`pinDigests` is a Docker-datasource concept. On the terraform manager it does
nothing for registry providers/modules (they're not digest-pinnable). Either
remove it or document why it's there (git-ref module sources?). Needs a dry-run
against a real modules repo to confirm it's a no-op.

#### 2.7 Forgejo rules unverified against a real repo

`ci.json` matches `forgejo-tags`/`forgejo-releases` datasources with
`registryUrls: ["https://forgejo.fartlab.dev/"]`. Whether the github-actions
manager actually emits those datasources for `.forgejo/workflows/` files (and
for `uses:` refs pointing at the forgejo host) hasn't been tested on a real
repo. The README test-plan checkbox from PR #11 is still open. Run the
documented dry-run against a forgejo-actions repo.

### P3 — Repo hygiene and infrastructure

#### 3.1 This repo doesn't dogfood Renovate

There is **no `renovate.json` in this repo**. The repo that ships Renovate
presets isn't running Renovate. It has real update surface: the two GitHub
Actions workflows (checkout, setup-node, required-labels action) and mise.toml
tools. Add:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>donaldgifford/renovate-config",
    "github>donaldgifford/renovate-config:ci",
    "github>donaldgifford/renovate-config:mise"
  ]
}
```

This is also the cheapest possible integration test — the presets get exercised
weekly on this repo itself.

#### 3.2 `validate.yml` doesn't validate `docs/examples/*.json`

Finding 1.3 (broken example) would have been caught if examples were validated.
They're plain renovate configs — the validator accepts them. Add a second loop
(or glob) over `docs/examples/*.json`. Note: validating examples against
`--strict` also catches future preset renames that orphan examples, since
`extends` refs to nonexistent presets fail resolution only at runtime — consider
a smoke test that greps example `extends` entries against the preset file list.

#### 3.3 Examples dir is missing 7 of 22 presets

No examples for: `python`, `terragrunt` (deployment repo), `kubernetes`, `hugo`,
`devcontainer`, `renovate-config`, `tool-versions`. The examples README table is
equally stale.

#### 3.4 Design docs still describe the `.json5` world

DESIGN-0001 references `.json5` filenames, JSON5 comment conventions, and the
"no `$schema` outside default" rule that has since flipped (every preset now
sets `$schema`). DESIGN-0002/0003 (skills docs) inherited the same conventions
in their skill designs (`preset-skeleton.json5` references, JSON5-only rules).
Since DESIGN-0003 is being ported to the claude-skills repo, correcting the
conventions there matters before implementation starts.

#### 3.5 `scripts/` is untracked

`scripts/labels.sh` and `scripts/goheader.tmpl` have sat untracked through ~6
PRs. Commit them (labels.sh looks like the label-bootstrap script the workflows
depend on conceptually) or delete them.

#### 3.6 `validate.yml` installs unpinned renovate on every run

`npm install -g renovate` pulls latest every CI run — a renovate release can
break CI with no repo change (and nearly did: the transient
`ETARGET renovate@43.216.1` failure seen during PR #10 work). Pin the version
and let Renovate (per 3.1) bump it.

### Verified-good

For completeness, these all check out:

- All 22 presets pass `renovate-config-validator --strict` with zero warnings
  (including no config-migration warnings after the `renovate-config-presets` →
  `renovate-config` fix)
- Every preset scopes rules with `matchManagers` (except the dead go.json rule,
  finding 1.4)
- Custom regex managers (docker-bake, helm appVersion, typst, tool-versions) all
  have paired packageRules with `matchFileNames` scoping
- No deprecated options anywhere (`matchPackagePatterns`, `matchPaths`,
  `fileMatch` all gone)
- Automerge opt-ins are exactly the documented four categories; nothing else
  sets `automerge: true`
- `default.json` is the only file with `extends`; global `automerge: false` with
  `platformAutomerge` for the opt-ins
- Group-name convention `<ecosystem> <thing> (non-major)` holds everywhere
- The typst regex correctly handles both `#let` and `#import "@preview/..."`
  forms; the tool-versions regex correctly strips `v` prefixes via
  `extractVersionTemplate`

## Conclusion

**Answer: No** — the presets are validator-clean but not behavior-correct.

Five P1 findings change actual runtime behavior: the advertised
lockfile-maintenance automerge never fires (1.1), eleven presets emit label
combinations that violate the exactly-one-semver-label contract (1.2), the
terragrunt example silently lost coverage in the split (1.3), go.json carries a
dead rule (1.4), and mise.toml has a mis-annotated tool (1.5). None of these are
detectable by `renovate-config-validator` — they're all semantic. The P3 items
compound the risk: no dogfooding and no example validation means these classes
of bug recur silently.

## Recommendation

Ordered by leverage:

1. **Fix P1 now** (one PR): enable lockFileMaintenance in rust/terraform, decide
   the label-stacking policy (recommend: `dont-release` is exclusive — strip
   `patch`/`minor` from infra rules), fix the terragrunt example, drop or
   implement the go.json dead rule, fix the mise.toml annotation.
2. **Dogfood + CI hardening** (one PR): add `renovate.json` to this repo,
   validate `docs/examples/*.json` in CI, pin renovate in validate.yml, commit
   or delete `scripts/`.
3. **P2 consistency pass** (one PR): helm appVersion major split + docker
   versioning, tflint/homebrew major labels, bun-version manager, drop redundant
   mise `enabled`, resolve terraform `pinDigests`.
4. **Docs refresh** (one PR): examples for the 7 uncovered presets, DESIGN doc
   `.json5` → `.json` errata (and sync DESIGN-0003 before the claude-skills
   port).
5. **Verify forgejo on a real repo** using the CLAUDE.md dry-run procedure;
   record the result in this INV and close it.

## References

- DESIGN-0001: Shareable Renovate Preset Architecture
- DESIGN-0002 / DESIGN-0003: skills design docs (convention staleness, 3.4)
- PR #11 (native managers, .json5 removal), PR #12 (python preset)
- [Renovate lockFileMaintenance](https://docs.renovatebot.com/configuration-options/#lockfilemaintenance)
- [Renovate mise manager](https://docs.renovatebot.com/modules/manager/mise/)
- `.github/workflows/pr-labels.yml` — the exactly-one-label contract
