# ADO Repository Rules

This document defines repository, branch, worktree, and path constraints.

## 1. Repository Boundary

ADO never implements directly in a user's active development directory.

All automated implementation happens in:

```text
{ADO_DATA_DIR}/projects/{project_key}/worktrees/{component_work_id}/
```

## 2. Branch Model

Protected:

```text
main
integrate
```

ADO branches:

```text
ado/{project_key}/{work_type}/{feature_unit_key}/{component_work_key}
```

Examples:

```text
ado/deci-duel/feature/nickname-change/nickname-contract
ado/ado/feature/core-db-models/core-schema
```

## 3. Base and Target Branches

Rules:

- Work branches are created from `origin/integrate`.
- PR target is `integrate`.
- ADO never targets `main`.
- ADO never pushes directly to `integrate`.

## 4. Worktree Creation

Required preflight:

1. repository registered in DB
2. primary Component and every ComponentWorkScope mapped to the one repository
3. `origin/integrate` exists
4. branch name matches ADO pattern
5. target worktree path is under ADO_DATA_DIR
6. no unexpected existing dirty worktree
7. ComponentWork has immutable scope entries and allowed_paths

## 5. Dirty Worktree Policy

If worktree exists and is dirty:

- if dirty state is expected for current Job, continue
- otherwise mark `human_required`
- record GitSnapshot
- do not overwrite

## 6. Allowed Paths

Every ComponentWork must define allowed_paths.

For a managed Project monorepo, `single` Work may write only its declared
primary root. `coordinated` Work may write its declared ComponentWorkScope
roots plus explicitly human-approved shared paths. See
`MANAGED_PROJECT_MONOREPO_POLICY.md`.

PR creation is blocked if any changed file is outside allowed_paths.

Allowed paths must be explicit enough to prevent accidental broad edits.

Discouraged:

```text
**
src/**
.
```

Allowed only with human approval:

- package manager lockfiles
- generated files
- project config files
- migrations
- CI files
- security-sensitive files

## 7. Git Commands

Allowed by GitWorkspaceManager:

- fetch
- worktree add
- status
- diff
- rev-parse
- branch existence checks
- snapshot collection

Allowed by GitHubPRManager:

- push ADO branch

Forbidden by default:

- reset --hard
- clean -fd
- checkout main for modification
- switch main for modification
- push integrate
- push main
- push --force
- merge
- rebase protected branches
- branch delete user branch

## 8. Commit Policy

ADO commits are Component Work scoped.

Commit message format:

```text
{work_type}({component_work_key}): {feature_unit_title}

ADO-Project: {project_key}
ADO-Feature-Unit: {feature_unit_key}
ADO-Component-Work: {component_work_key}
ADO-Agent-Run: {agent_run_id}
```

ADO does not squash automatically.

## 9. PR Policy

PRs are Component Work level. A coordinated Component Work still creates one
branch and one PR for all of its declared roots. Feature Unit UI groups only
independent related PRs.

Required before PR:

- branch rule valid
- target integrate
- changed paths allowed
- verification passed
- review completed
- ArbiterDecision permits PR
- PullRequestPacket valid
- no stale/quarantined evidence

## 10. AGENTS/CLAUDE Files

Generated instruction files may be placed in ADO worktrees.

Rules:

- Generated `AGENTS.md` is short and points to ContextPacket.
- Generated `CLAUDE.md` may import `AGENTS.md`.
- Generated instruction files are artifacts.
- Generated instruction files include source hash where practical.
- Product repo instruction files are modified only if ComponentWork explicitly scopes that change.

## 11. Recovery

Worktree is not source of truth.

ADO must be able to recover from:

- DB records
- git remote
- branch name
- artifacts
- command logs

If recovery cannot prove state, mark `human_required`.
