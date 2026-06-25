# Project Derivation Guide

This guide explains how to derive project-specific ADO documents from `ADO_MASTER_SPEC.md`.

## 1. Create Project Workspace

Recommended structure:

```text
~/.ado/projects/{project_key}/
  docs/
  artifacts/
  logs/
  worktrees/
```

For repository-local project docs, use:

```text
project-root/
  ADO_PROJECT_SPEC.md
  ADO_ROADMAP.md
  docs/ado/
```

Only create repository-local docs when the project explicitly wants them.

## 2. Define Project

Create a project spec with:

- approved ADO Spec Library revision and manifest hash
- project key
- product/service name
- one public monorepo repository (required in v1)
- selected managed Project file structure profile
- components
- environments
- project constraint profile
- design constraints
- architecture constraints
- verification profile
- protected branches
- base integration branch
- allowed providers
- safety policy
- budget policy

Project constraints are created from `SERVICE_CONSTRAINT_QUESTIONNAIRE.md` and approved by a human before execution.
The Project binds one approved SpecLibraryRevision before its first packet is
generated. A later upgrade is a new human-approved Project configuration
revision; it does not alter prior Feature Units, packets, runs, or PR evidence.

## 3. Import Roadmap

Roadmap is human-provided source input. It is not directly executable.

Planner decomposes it into:

```text
Roadmap -> Feature Unit drafts -> Component Work drafts
```

Human approval is required before execution.

## 4. Define Components

The default file structure profile is `standard_product_monorepo` from
`MANAGED_PROJECT_FILE_STRUCTURE_POLICY.md`.

Typical component types:

- app
- web
- server
- admin
- infra
- design_asset
- docs
- research

Each component maps to a root in the one Project repository and to a
verification profile. Each component also declares an internal structure
profile such as `nextjs_feature_app`, `expo_feature_app`,
`nestjs_module_api`, `pure_shared_package`, `project_ui_package`, or
`transport_contract_package`. Multiple component roots in the same repository
are the normal Project shape.

Each component also inherits relevant project constraints. For example, a web component receives web design and frontend architecture constraints, while a server component receives API, data, and backend verification constraints.

The Platform supplies only the selected Spec Library sections required by that
component's role and work. It never injects a mutable whole-spec checkout.

## 5. Define Feature Units

A Feature Unit must include:

- goal
- scope
- out of scope
- acceptance criteria
- human verification checklist
- required components
- risk level
- dependencies

Feature Unit is the human-verifiable unit.

## 6. Define Component Work

Component Work must include:

- primary component and declared component scopes
- repository mapping
- execution scope (`single` or `coordinated`)
- branch name
- allowed paths
- component contract
- verification profile
- risk level
- expected artifacts

Component Work is the implementation and PR unit. A `coordinated` Work covers
multiple declared monorepo roots on one branch/PR when a Feature Unit cannot be
verified in a partially merged state.

## 7. Execution Policy

Default v1 policy:

- branch from integrate
- PR to integrate
- never touch main
- never auto-merge
- ADO-managed worktree only
- single Component Work PRs when independent
- one coordinated Component Work PR when components must integrate atomically
- Feature Unit groups independent PRs

## 8. Review Policy

Default v1 review:

- verification first
- Local Review Council after verification
- Codex Arbiter after local review
- P0/P1 accepted findings block PR
- human final verification after PR creation

## 9. Claude Policy

Claude is a human-mediated advisor.

Export only external-safe packets. Import responses as CandidateArtifact. Human must promote or reject.

## 10. Implementation Order

For a new project:

1. Create ProjectSpec.
2. Answer Service Constraint Questionnaire.
3. Generate ProjectConstraintProfile.
4. Generate project design, architecture, and verification documents.
5. Human approves ProjectConstraintProfile.
6. Register repositories and components.
7. Import roadmap.
8. Decompose into Feature Units.
9. Human reviews and approves Feature Units.
10. Generate Component Work specs.
11. Execute approved Component Work.
12. Create PRs to integrate.
13. Human verifies and merges outside ADO.
