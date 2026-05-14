---
id: DESIGN-0002
title: "Local Skills for Maintaining Renovate Presets"
status: Draft
author: Donald Gifford
created: 2026-05-13
---

<!-- markdownlint-disable-file MD025 MD041 -->

# DESIGN 0002: Local Skills for Maintaining Renovate Presets

**Status:** Draft **Author:** Donald Gifford **Date:** 2026-05-13

<!--toc:start-->

- [Overview](#overview)
- [Goals and Non-Goals](#goals-and-non-goals)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Background](#background)
- [Detailed Design](#detailed-design)
  - [Skill Inventory](#skill-inventory)
  - [Skill: preset-author](#skill-preset-author)
  - [Skill: preset-reviewer](#skill-preset-reviewer)
  - [Skill: preset-dry-run](#skill-preset-dry-run)
  - [Skill: preset-branch-test](#skill-preset-branch-test)
  - [Layout in .claude/skills/](#layout-in-claudeskills)
  - [Shared References](#shared-references)
- [API / Interface Changes](#api--interface-changes)
- [Data Model](#data-model)
- [Testing Strategy](#testing-strategy)
- [Migration / Rollout Plan](#migration--rollout-plan)
- [Open Questions](#open-questions)
- [References](#references)
<!--toc:end-->

## Overview

This document designs a set of Claude Code **local skills** living inside this
repo (`.claude/skills/`) that automate the maintenance work for our shareable
Renovate presets. These skills are scoped to _this_ repo's workflow — authoring
new presets, reviewing changes, testing unmerged presets against real consumer
repos. They are intentionally distinct from the portable skills in DESIGN-0003,
which target consumer repos.

## Goals and Non-Goals

### Goals

- Codify the conventions in `CLAUDE.md` (label scheme, group name pattern,
  scoped `matchManagers`, paired packageRules for custom managers) as skill
  procedures rather than prose checklists humans have to remember
- Make `renovate-config-validator --strict` + `prettier --check` the default
  first-and-last action of every preset edit
- Make the `#<branch>` test loop and local `--dry-run=full` invocations
  one-command operations
- Keep the README, CLAUDE.md, and DESIGN-0001 preset inventory automatically in
  sync when a preset is added or renamed

### Non-Goals

- Replacing CI (`.github/workflows/validate.yml`) — skills augment local work,
  CI is still the source of truth
- Helping consumer repos configure Renovate (covered in DESIGN-0003)
- Auto-merging or otherwise operating on PRs in this repo
- Generating Renovate bot deployment/hosting config

## Background

This repo has accumulated a non-trivial set of conventions:

- JSON5 with `// --- Description ---` rule separators
- Only `default.json5` sets `$schema`, `extends`, and global automerge
- Every preset must label non-major (`patch`) and major (`minor`) updates
- PR by default — `automerge: true` is a narrow opt-in per CLAUDE.md
- Custom regex managers need paired `packageRules` matching `custom.regex` and
  scoped by `matchFileNames`
- Group name pattern: `<ecosystem> <thing> (non-major)`
- Schedule: weekly Monday before 6am, `America/Detroit`

A net-new preset touches at minimum: the `.json5` file, `README.md`,
`CLAUDE.md`, and the DESIGN-0001 inventory table. Forgetting any of these is how
drift starts, and we've already paid for several conventions twice (e.g.
`matchPackagePatterns` deprecation, `cargoUpdate` not existing, custom managers
without paired rules silently dropping updates).

Codifying these as skills moves the convention-enforcement from "remember to
follow CLAUDE.md" to "run the skill."

## Detailed Design

### Skill Inventory

| Skill                | Trigger                                        | What It Does                                                                                                  |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `preset-author`      | Adding a new ecosystem or cross-cutting preset | Scaffolds `<name>.json5`, updates README/CLAUDE.md/DESIGN-0001, runs validator + prettier                     |
| `preset-reviewer`    | Reviewing a preset change (local or PR)        | Checks deprecated options, label coverage, scope, custom-manager pairing, group-name pattern, redundant rules |
| `preset-dry-run`     | Previewing effect of preset changes            | Wraps `renovate --dry-run=full` against a target consumer repo and summarizes what would change               |
| `preset-branch-test` | Testing unmerged presets on a real repo        | Switches a target consumer repo's `extends` to `#<branch>` form, optionally reverts after testing             |

### Skill: preset-author

**Trigger:** "add a preset for X", "scaffold a new preset", `/preset-author`.

**Workflow:**

1. **Gather inputs** (interactive if missing):
   - Preset name (file stem, e.g. `python`)
   - Layer: ecosystem or cross-cutting
   - Renovate manager(s) covered (`gomod`, `pip_requirements`, etc.)
   - Whether a custom regex manager is needed
   - Whether anything in this preset is safe to automerge (default: no)
2. **Scaffold the JSON5 file** using the convention skeleton:
   - `// --- Group non-major <ecosystem> updates ---` with
     `addLabels: ["patch"]` and `groupName: "<ecosystem> <thing> (non-major)"`
   - `// --- Major bumps get individual PRs ---` with `addLabels: ["minor"]`
   - If custom regex manager: paired `packageRules` entry with
     `matchManagers: ["custom.regex"]` and `matchFileNames` scoped to this
     preset's files
   - No `$schema`, no `extends`, no redundant `automerge: false`
3. **Update docs in lockstep:**
   - `README.md`: add usage example block and preset table entry
   - `CLAUDE.md`: add bullet under "Preset Layers" in the right layer
   - `docs/design/0001-shareable-renovate-preset-architecture.md`: add row to
     the preset inventory table
4. **Validate and format:**
   - `npx --yes --package=renovate -- renovate-config-validator --strict <file>`
   - `prettier --write "*.json5" *.md`
5. **Report**: list of files changed, validator output, prettier output.

**Anti-patterns it must refuse:**

- Setting `$schema` or `extends` outside `default.json5`
- Adding `automerge: true` without an explicit justification in the rule's
  preceding `// ---` comment
- Using `matchPackagePatterns` (deprecated — must use `matchPackageNames` with
  glob)
- Using `cargoUpdate` postUpdateOption (doesn't exist)
- Creating a custom regex manager without a paired `packageRules` entry
- Group name that doesn't follow `<ecosystem> <thing> (non-major)`

**Reference files (loaded on demand):**

- `references/preset-skeleton.json5` — the canonical scaffold
- `references/custom-manager-skeleton.json5` — scaffold for regex managers
- `references/conventions.md` — distilled checklist from CLAUDE.md

### Skill: preset-reviewer

**Trigger:** "review this preset", `/preset-reviewer`, or invoked automatically
against staged JSON5 changes.

**Workflow:**

1. **Scope detection:** identify which `.json5` files are touched (staged or
   passed as arguments).
2. **Per-file checks:**
   - **Deprecated options:** grep for `matchPackagePatterns`, `matchPaths`
     (deprecated in favor of `matchFileNames`), `cargoUpdate` in
     `postUpdateOptions`
   - **Schema discipline:** `$schema` and `extends` only in `default.json5`
   - **Label coverage:** every preset must have a non-major-with-`patch` rule
     and a major-with-`minor` rule
   - **Automerge discipline:** every `automerge: true` rule has a preceding
     `// ---` comment explaining the opt-in
   - **Scope:** every rule has `matchManagers` (or matches via `customType`)
   - **Group naming:** group names match `^<ecosystem> .+ \(non-major\)$`
   - **Custom manager pairing:** for every `customManagers` entry, a
     `packageRules` entry exists with `matchManagers: ["custom.regex"]` AND
     `matchFileNames` matching the custom manager's `managerFilePatterns`
   - **Redundant `automerge: false`:** the base default already blocks
     automerge, so explicit `automerge: false` rules are noise
3. **Run validator and prettier check:**
   - `renovate-config-validator --strict` on each touched file
   - `prettier --check` on each touched file
4. **Report findings** with severity (`error`, `warning`, `info`) and the
   file:line where applicable.

**Output format:**

```
preset-reviewer report for node.json5:
  error  L18: matchPackagePatterns is deprecated — use matchPackageNames with glob
  warn   L25: automerge: true rule has no preceding `// ---` comment explaining opt-in
  info   group name "node packages (non-major)" matches convention
```

### Skill: preset-dry-run

**Trigger:** "dry-run this preset against `<repo>`", `/preset-dry-run <repo>`.

**Workflow:**

1. **Verify prerequisites:**
   - `gh auth token` returns a valid token
   - Branch is pushed to origin (so Renovate can fetch via
     `github>donaldgifford/renovate-config#<branch>`)
2. **Run the command:**

   ```bash
   npx --package=renovate -- renovate \
     --platform=github \
     --token=$(gh auth token) \
     --dry-run=full \
     --autodiscover=false \
     <target-repo>
   ```

3. **Capture and summarize stdout/stderr:** parse the dry-run output, surface:
   - Which managers detected files
   - Which packages would update (name, current → new)
   - Which `packageRules` matched each update
   - Any custom regex managers that fired with zero matches (silent bug warning)

**Why summarize:** the raw Renovate dry-run output is verbose and not ergonomic
to scan. The skill surfaces the signal — what would change, and whether our
custom managers actually matched anything.

### Skill: preset-branch-test

**Trigger:** "test this branch on `<repo>`", `/preset-branch-test <repo>`.

**Workflow:**

1. **Resolve target:** clone or `cd` into the target consumer repo (skill
   parameter is a `owner/repo` slug or local path)
2. **Detect current extends:** read the target's `renovate.json` and identify
   any `github>donaldgifford/renovate-config[:preset]` entries
3. **Rewrite to branch form:** append `#<current-branch>` to each matching
   extends entry
4. **Commit on a throwaway branch in the target repo** (e.g.
   `test/renovate-config-<branch>`) and push
5. **Trigger Renovate** (option A: rely on the schedule; option B: invoke the
   dry-run skill against the rewritten config)
6. **Cleanup mode:** when re-invoked with `--revert`, restore the unsuffixed
   extends form

**Safety:** never modifies the target repo's `main` branch directly. The skill
checks out the target repo into a git worktree (sibling directory) so the user's
existing checkout is undisturbed. All edits happen in the worktree, get
committed to a side branch, and pushed from there. The worktree is removed on
`--revert` or after successful cleanup.

### Layout in `.claude/skills/`

Following the structure used by claude-md plugin skills:

```
.claude/
  skills/
    preset-author/
      SKILL.md
      references/
        preset-skeleton.json5
        custom-manager-skeleton.json5
        conventions.md
    preset-reviewer/
      SKILL.md
      references/
        deprecated-options.md
        review-checklist.md
    preset-dry-run/
      SKILL.md
    preset-branch-test/
      SKILL.md
```

Each `SKILL.md` is short prose describing the workflow plus a `## Workflow`
section. Heavy reference data (skeletons, checklists) lives in `references/` and
is loaded only when the skill needs it.

### Shared References

Several skills need the same convention data. Rather than duplicate, the
project's `CLAUDE.md` remains the canonical statement and skills can `@import`
or grep it. The `references/` directories in each skill carry only data specific
to that skill (e.g., the preset skeleton).

## API / Interface Changes

No changes to the JSON5 presets themselves. Skills are pure tooling — they read
and write the same files a human would. The skills do introduce two new slash
commands surfaced to Claude Code users in this repo:

- `/preset-author` — scaffold a new preset
- `/preset-reviewer` — audit a preset change

`/preset-dry-run` and `/preset-branch-test` are also slash-invokable but more
commonly called by name in conversation.

## Data Model

No persisted state. Skills are stateless and operate on the repo's working tree
plus shell command output.

## Testing Strategy

Skills are validated by their own outputs:

1. **`preset-author` self-test:** run it against a sample input (e.g. "scaffold
   a `python` preset"), then run `preset-reviewer` on the result. If the
   reviewer flags any errors, the author skill is broken.
2. **`preset-reviewer` regression suite:** keep a small set of known-bad fixture
   presets under `.claude/skills/preset-reviewer/fixtures/`. The reviewer must
   flag every documented violation.
3. **`preset-dry-run` smoke test:** run against a known-stable consumer repo on
   `main` and verify the summary structure renders.
4. **`preset-branch-test` smoke test:** run with `--revert` on a clean target
   and verify it's a no-op.

CI changes: none. `preset-reviewer` is local-only by design — CI's job is strict
validation (`renovate-config-validator --strict` + `prettier --check`), and
`preset-reviewer` is the looser convention-coach for humans editing locally.
Keeping them separate avoids muddying CI signal with stylistic guidance.

## Migration / Rollout Plan

1. **Phase 1 — `preset-reviewer` only.** Lowest risk; pure read. Ship it,
   exercise it on the existing presets, capture any false positives.
2. **Phase 2 — `preset-author`.** Once the reviewer is trustworthy, the author
   skill uses it as its post-scaffold gate.
3. **Phase 3 — `preset-dry-run` and `preset-branch-test`.** These touch the
   network and modify a target repo's branch state, so they ship last and
   default to dry-run / interactive confirmation.

Each phase is its own PR.

## Open Questions

_None at this time. Resolved during design review:_

- _Doc sync (README, CLAUDE.md, DESIGN-0001 inventory) is bundled into
  `preset-author` rather than split into a separate skill._
- _`preset-reviewer` stays local-only — not wired into CI._
- _`preset-branch-test` uses a git worktree to keep the user's checkout
  undisturbed._

## References

- [Renovate Shareable Config Presets](https://docs.renovatebot.com/config-presets/)
- [Renovate Custom Managers](https://docs.renovatebot.com/modules/manager/regex/)
- [Claude Code Skills documentation](https://docs.claude.com/en/docs/claude-code/skills)
- DESIGN-0001: Shareable Renovate Preset Architecture
- DESIGN-0003: Portable Skills for Configuring Repos with Renovate
- `CLAUDE.md` — canonical conventions reference
