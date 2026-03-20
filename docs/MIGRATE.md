# Migrating a Go Repo from Dependabot to Renovate

A practical guide for replacing Dependabot with self-hosted or GitHub-hosted Renovate (FOSS) on Go repositories.

---

## Why Migrate

Dependabot's Go support is functional but limited. Renovate gives you grouped PRs (monorepo-aware `go.mod` updates), regex managers for non-standard version references (Dockerfiles, Makefiles, `.tool-versions`), auto-merge with configurable stability policies, and a single config language that works across Go modules, container images, GitHub Actions, Helm charts, and Terraform in the same repo. For Go specifically, Renovate understands `go.mod` replace directives, handles major version bumps with import path changes, and can pin digests on container images referenced in your Go builds.

---

## Step 0: Understand Renovate's Execution Models

Renovate FOSS runs in two modes — pick the one that fits your setup.

**GitHub App (Mend-hosted):** Install the [Mend Renovate App](https://github.com/apps/renovate) from the GitHub Marketplace. Zero infrastructure — Mend runs the workers. You just add `renovate.json` to each repo. This is the simplest path for personal and small-org repos.

**Self-hosted:** Run `renovate` as a container or binary on your own infra (GitHub Actions workflow, Kubernetes CronJob, etc.). You control scheduling, concurrency, and credentials. Required if you need to hit private registries behind a VPN or want full control over execution.

For personal repos (public and private), the GitHub App is the path of least resistance. The config file is identical either way.

---

## Step 1: Disable Dependabot

Remove or rename `.github/dependabot.yml` so both tools aren't racing to open PRs for the same dependencies.

```bash
# Option A: delete it
git rm .github/dependabot.yml

# Option B: keep it around for reference during migration
git mv .github/dependabot.yml .github/dependabot.yml.disabled
```

Close any open Dependabot PRs you don't intend to merge — Renovate will re-create equivalent ones under its own management.

If you're using Dependabot security alerts (not version updates), note that those are a separate GitHub feature and continue to work independently of `dependabot.yml`. You lose nothing on the security alert side.

---

## Step 2: Install Renovate

### GitHub App (recommended for personal repos)

1. Go to [https://github.com/apps/renovate](https://github.com/apps/renovate) and install it.
2. Grant access to the specific repos you want, or all repos.
3. Renovate will open an onboarding PR in each repo with a default `renovate.json`. You can merge it as-is or replace the contents before merging.

### Self-Hosted via GitHub Actions

If you prefer to run it yourself (useful if you want a single workflow across many repos), create `.github/workflows/renovate.yml`:

```yaml
name: Renovate
on:
  schedule:
    - cron: "0 */6 * * *" # every 6 hours
  workflow_dispatch: {}

jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: renovatebot/github-action@v41
        with:
          configurationFile: renovate.json
          token: ${{ secrets.RENOVATE_TOKEN }}
```

The `RENOVATE_TOKEN` needs `repo` and `workflow` scopes. A fine-grained PAT scoped to the target repos is ideal.

---

## Step 3: Create the Renovate Config

Drop a `renovate.json` (or `renovate.json5` if you want comments) in the repo root.

### Minimal Go Config

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"]
}
```

This alone gives you `go.mod` dependency updates, GitHub Actions updates, and Dockerfile base image updates using Renovate's built-in manager detection. The `config:recommended` preset includes sane defaults: respect semver, separate major/minor/patch PRs, and a dependency dashboard issue.

### Production Go Config

A more opinionated config that maps common Dependabot patterns to Renovate equivalents:

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: [
    "config:recommended",
    "docker:enableMajor",
    ":automergeDigest",
    ":automergePatch",
    "group:allNonMajor",
  ],

  // --- Schedule ---
  // Equivalent to dependabot's `schedule.interval: weekly`
  schedule: ["before 6am on monday"],
  timezone: "America/Detroit",

  // --- Go-specific ---
  gomod: {
    // Post-update: run `go mod tidy` after updating go.mod
    postUpdateOptions: ["gomodTidy"],
    // Group all non-major go module updates into one PR
    groupName: "go modules (non-major)",
    groupSlug: "go-modules-non-major",
    matchUpdateTypes: ["minor", "patch", "digest"],
  },

  // --- Major version bumps get individual PRs ---
  major: {
    automerge: false,
  },

  // --- Auto-merge settings ---
  automerge: true,
  automergeType: "pr",
  platformAutomerge: true,

  // --- PR behavior ---
  prConcurrentLimit: 5,
  prHourlyLimit: 2,
  labels: ["dependencies"],
  assignees: [],

  // --- Vulnerability alerts ---
  // Renovate can create PRs for vulnerabilities from the Go vuln DB
  vulnerabilityAlerts: {
    enabled: true,
    labels: ["security"],
  },

  // --- Pin GitHub Actions digests ---
  "github-actions": {
    pinDigests: true,
  },
}
```

---

## Step 4: Handle Go-Specific Scenarios

### `go.mod` replace directives

Renovate respects `replace` directives in `go.mod`. If you have local replacements (e.g., `replace github.com/foo/bar => ../bar`), Renovate will skip those — it only updates replacements that point to versioned modules.

### Go toolchain directive

Go 1.21+ added the `toolchain` directive to `go.mod`. Renovate has a dedicated `gomod` manager that handles this. If you want Renovate to update the Go version itself:

```json5
{
  gomod: {
    postUpdateOptions: ["gomodTidy", "gomodUpdateImportPaths"],
  },
  // Renovate detects the `go` and `toolchain` directives automatically
}
```

The `gomodUpdateImportPaths` option is important for major version bumps — it rewrites import paths when a dependency moves from `v1` to `v2`.

### Private Go modules

If your `go.mod` references private modules (e.g., `github.com/your-org/private-lib`), Renovate needs credentials. For the GitHub App, it already has access to repos you've granted. For self-hosted, set `GOPRIVATE` and provide a token:

```json5
{
  hostRules: [
    {
      matchHost: "github.com",
      token: "{{ secrets.GITHUB_TOKEN }}",
    },
  ],
  env: {
    GOPRIVATE: "github.com/your-org/*",
  },
}
```

For self-hosted via GitHub Actions, pass the token through the environment:

```yaml
env:
  GOPRIVATE: "github.com/your-org/*"
  GONOSUMCHECK: "github.com/your-org/*"
```

### Goreleaser / Go binaries in Dockerfiles

If your Dockerfile has a Go version pinned (e.g., `FROM golang:1.23.2-bookworm`), Renovate's `dockerfile` manager handles this automatically — no extra config needed. Same for multi-stage builds.

### Go tool dependencies (tools.go pattern)

If you use a `tools.go` file with `//go:build tools` for pinning CLI tools (like `golangci-lint`, `buf`, `wire`, etc.), Renovate treats these as normal `go.mod` dependencies since they appear in `go.sum`. They'll be updated as part of the `gomod` manager.

### Custom version references (regex manager)

If you have version strings in Makefiles, scripts, or other non-standard locations, use `customManagers`:

```json5
{
  customManagers: [
    {
      customType: "regex",
      fileMatch: ["^Makefile$", "^scripts/.+\\.sh$"],
      matchStrings: [
        "GOLANGCI_LINT_VERSION\\s*[:?]?=\\s*v?(?<currentValue>\\S+)",
      ],
      depNameTemplate: "golangci/golangci-lint",
      datasourceTemplate: "github-releases",
      extractVersionTemplate: "^v(?<version>.*)$",
    },
  ],
}
```

---

## Step 5: Map Dependabot Features to Renovate

| Dependabot (`dependabot.yml`)      | Renovate equivalent                                                           |
| ---------------------------------- | ----------------------------------------------------------------------------- |
| `package-ecosystem: gomod`         | Built-in `gomod` manager (auto-detected)                                      |
| `schedule.interval: weekly`        | `"schedule": ["before 6am on monday"]`                                        |
| `open-pull-requests-limit: 5`      | `"prConcurrentLimit": 5`                                                      |
| `reviewers: ["user"]`              | `"reviewers": ["user"]`                                                       |
| `assignees: ["user"]`              | `"assignees": ["user"]`                                                       |
| `labels: ["dependencies"]`         | `"labels": ["dependencies"]`                                                  |
| `ignore` (skip specific deps)      | `"ignoreDeps": ["github.com/foo/bar"]`                                        |
| `ignore` (skip versions)           | `"packageRules": [{"matchPackageNames": [...], "allowedVersions": "<2.0.0"}]` |
| `allow` (only specific deps)       | `"enabledManagers": [...]` or `packageRules` with `enabled`                   |
| `insecure-external-code-execution` | `"allowCustomCrateRegistries": true` / `allowScripts`                         |
| `versioning-strategy: increase`    | `"rangeStrategy": "bump"` (default for Go)                                    |
| Grouped updates (not supported)    | `"group:allNonMajor"` or custom `packageRules` with `groupName`               |
| Auto-merge (not native)            | `"automerge": true, "platformAutomerge": true`                                |

---

## Step 6: Validate and Merge

Before merging your config, validate it locally:

```bash
# Install the renovate-config-validator
npx --yes renovate --validate-config=strict renovate.json
```

Or if you prefer not to install anything, the onboarding PR from the Renovate GitHub App will show validation results in the PR body.

Once merged, Renovate will:

1. Create a **Dependency Dashboard** issue in your repo (a checklist tracking all pending updates).
2. Open PRs according to your schedule and grouping rules.
3. Auto-merge anything that matches your automerge criteria (if CI passes).

---

## Step 7: Clean Up

After Renovate is running and you're satisfied:

1. Delete `.github/dependabot.yml` (or the `.disabled` copy).
2. Close any remaining stale Dependabot PRs.
3. If you had Dependabot security updates enabled separately and want Renovate to handle those too, verify `"vulnerabilityAlerts": { "enabled": true }` is in your config. Note: Renovate's vulnerability detection for Go uses the Go vulnerability database and OSV, which is generally comprehensive.

---

## Quick Reference: Common Config Patterns

**Pin a dependency to a specific major version:**

```json5
{
  packageRules: [
    {
      matchPackageNames: ["google.golang.org/grpc"],
      allowedVersions: "<2.0.0",
    },
  ],
}
```

**Exclude a dependency entirely:**

```json5
{
  ignoreDeps: ["github.com/some/fork"],
}
```

**Separate PRs for a critical dependency:**

```json5
{
  packageRules: [
    {
      matchPackageNames: ["github.com/hashicorp/vault/api"],
      groupName: null,
      automerge: false,
      reviewers: ["team:security"],
    },
  ],
}
```

**Auto-merge everything except major bumps:**

```json5
{
  extends: ["config:recommended"],
  automerge: true,
  major: { automerge: false },
}
```

**Dashboard approval for specific deps (manual trigger via checkbox):**

```json5
{
  packageRules: [
    {
      matchPackageNames: ["k8s.io/*"],
      dependencyDashboardApproval: true,
    },
  ],
}
```
