# Deprecated: ADO Django Architecture

> Historical Django-only draft. `DJANGO_NEXT_PLATFORM_ARCHITECTURE.md` is the
> canonical implementation architecture for ADO v1. This file is retained only
> for early design context and must not be used to create new ADO code,
> schemas, commands, or tests.

This document defines the Django architecture for implementing Agent Development Orchestrator (ADO).

## 1. Architecture Goal

ADO is a control system, not a chat app.

The architecture must make these things easy:

- see what is happening
- stop unsafe work
- retry failed work
- prove why state changed
- generate narrow context packets
- inspect logs, artifacts, reviews, PRs, and human decisions

## 2. Recommended Repository Layout

```text
ado/
  manage.py
  config/
    settings/
      base.py
      local.py
      test.py
      production.py
    urls.py
    asgi.py
    wsgi.py
  apps/
    core/
    projects/
    planning/
    components/
    artifacts/
    documents/
    execution/
    verification/
    reviews/
    policy/
    state/
    gitops/
    integrations/
    audit/
    ui/
  templates/
  static/
  tests/
```

This layout is a default. ADO may split into packages later, but v1 should keep the system simple enough to inspect.

## 3. App Responsibilities

### 3.1 `core`

Shared primitives:

- base models
- time utilities
- typed IDs
- result objects
- domain exceptions
- redaction helpers
- hash helpers

`core` must not depend on feature apps.

### 3.2 `projects`

Owns:

- Project
- Repository
- Component
- EnvironmentProfile
- ProjectConstraintProfile
- project-specific generated constraint documents

This app is responsible for converting a generic ADO installation into a specific product workspace.

### 3.3 `planning`

Owns:

- Roadmap
- RoadmapSource
- FeatureUnit
- AcceptanceCriterion
- HumanVerificationItem
- FeatureUnitRelation

Planning creates executable drafts. Human approval makes them executable.

### 3.4 `components`

Owns:

- ComponentWork
- ComponentContract
- ComponentWorkRelation
- allowed paths
- expected artifacts
- component-specific verification binding

Component Work is the implementation and PR unit.

### 3.5 `artifacts`

Owns:

- Artifact
- ArtifactSourceRef
- ArtifactEmbedding
- artifact status
- artifact hashes
- redaction metadata

Artifacts are claims or evidence. They are not automatically truth.

### 3.6 `documents`

Owns:

- DocumentArtifact
- ContextPacket
- ReviewPacket
- markdown generation
- stale detection
- manual edit detection

Documents are generated from database state unless explicitly imported.

### 3.7 `execution`

Owns:

- Job
- AgentRun
- CommandRun
- runner leasing
- heartbeat
- timeout handling
- retry handling

Execution records what happened. It does not approve what happened.

### 3.8 `verification`

Owns:

- VerificationProfile
- VerificationRun
- command plans
- deterministic evidence
- human verification checklist results

Verification must distinguish command success from product correctness.

### 3.9 `reviews`

Owns:

- ReviewGroup
- ReviewResult
- ReviewFinding
- ArbiterDecision
- RevisionTask

Local models review independently. Codex Arbiter synthesizes. Policy and evidence decide whether PR creation is allowed.

### 3.10 `policy`

Owns:

- PolicyDecision
- SafetyEvent
- BudgetPolicy
- command allow/deny decisions
- external transfer decisions

Policy code should be mostly pure and easy to test.

### 3.11 `state`

Owns:

- TransitionRequest
- StateTransition
- state-machine application services

Only this app applies important business state changes.

### 3.12 `gitops`

Owns:

- GitWorktree
- GitSnapshot
- PullRequest
- branch naming
- worktree lifecycle
- GitHub PR synchronization

Git operations must respect repository rules.

### 3.13 `integrations`

Owns adapters for external systems:

- Codex CLI
- Ollama/local models
- GitHub CLI/API
- Claude import/export packets
- notification providers

Integrations return structured results. They do not mutate orchestration state directly.

### 3.14 `audit`

Owns:

- AuditEvent
- IncidentReport
- operation timeline exports

Audit must be append-only in normal operation.

### 3.15 `ui`

Owns Django views, templates, and API endpoints for:

- dashboard
- project setup
- roadmap approval
- feature unit approval
- job logs
- phase progress
- PR status
- human verification
- manual override with reason

UI code must call services. It must not implement orchestration rules.

## 4. Standard App Module Shape

Small apps may start with files:

```text
apps/{app_name}/
  __init__.py
  apps.py
  models.py
  admin.py
  selectors.py
  services.py
  policies.py
  validators.py
  types.py
  tests/
```

Large apps should split by concern:

```text
apps/execution/
  models/
  services/
  selectors/
  runners/
  tests/
```

Do not split early only for style. Split when the file is becoming hard to review.

## 5. Dependency Direction

Allowed dependency direction:

```text
ui -> services -> policies/selectors/models
worker -> services -> runners
runners -> integrations
services -> state/policy/audit/artifacts
```

Forbidden dependency direction:

```text
models -> services
policy -> ui
state -> ui
integrations -> ui
runners -> state mutation
```

## 6. Transaction Boundaries

Use `transaction.atomic()` around database state changes that must commit together.

Do not run external commands inside a transaction.

Correct pattern:

```text
1. create command run record
2. commit
3. run external command
4. persist result in short transaction
5. request state transition
```

Incorrect pattern:

```text
1. open transaction
2. run codex exec
3. run tests
4. call GitHub
5. commit
```

## 7. Admin And UI

Django admin may be used for internal inspection, but it is not the primary control room.

ADO requires first-class UI screens for:

- project overview
- feature unit board
- component work detail
- job queue
- agent run detail
- command logs
- artifact explorer
- review findings
- PR status
- human verification
- incident review

Admin actions that change state must call the same services and policy checks as the normal UI.

## 8. Settings

Settings must be environment-specific.

Required settings groups:

- database
- allowed repositories
- worktree root
- artifact root
- log root
- command allowlist
- Codex runner settings
- local model runner settings
- GitHub settings
- external transfer policy
- budget policy

Secrets must come from environment variables or a secret manager. They must not be stored in project specs, Markdown, or logs.

## 9. Management Commands

v1-alpha worker entry points are Django management commands.

Recommended commands:

```text
ado_worker
ado_enqueue
ado_publish_outbox
ado_worker_status
ado_recover_workers
ado_reconcile_runtime
ado_rebuild_documents
ado_validate_artifacts
ado_sync_prs
ado_collect_stale_jobs
ado_export_claude_packet
ado_import_claude_response
```

Commands must be thin wrappers around services.

The command and Worker lifecycle contract is defined in
`WORKER_EXECUTION_CONTRACT.md` and `WORKER_OPERATIONS.md`.

## 10. Migration Rules

- Every model change must include a migration.
- Critical invariants must use database constraints where practical.
- Data migrations must be idempotent.
- Migration code must not call external services.
- Enum changes must preserve old audit records.

## 11. Project-Specific Constraint Support

ADO must support generated project constraint profiles.

At minimum, the `projects` app stores:

- generic ADO policy version
- project identity
- component map
- design constraints
- architecture constraints
- verification profile
- human-approved status
- source questionnaire version
- generated document hashes

Feature Unit and Component Work packets must include only the project constraints relevant to the target work.
