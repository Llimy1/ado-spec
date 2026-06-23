# ADO Coding Standards

This document defines coding standards for implementing Agent Development Orchestrator (ADO).

ADO code must be boring, explicit, auditable, and easy to stop safely.

## 1. Design Basis

ADO follows these foundations:

- NestJS module design: clear boundaries, explicit providers, and controllers
  that stay at the transport boundary.
- TypeScript monorepo model: packages have one directional dependency graph and
  apps compose concrete adapters only at their edges.
- TypeORM/PostgreSQL transaction model: business state changes use explicit
  `QueryRunner` transaction boundaries.
- TypeScript testing model: deterministic unit, module, integration, database,
  and critical UI tests prove behavior.
- PostgreSQL model: important invariants must be represented with constraints, indexes, and transactions.
- Harness engineering model: LLMs propose actions, but the harness enforces permissions, schemas, state transitions, evidence checks, and audit records.

## 2. Core Principles

### 2.1 Explicit Over Implicit

Do not hide state changes in signals, model `save()` overrides, middleware side effects, or admin hooks.

Allowed:

- explicit service function
- explicit transaction
- explicit state transition request
- explicit audit event

Forbidden by default:

- business-critical ORM subscribers/listeners
- implicit status updates in model methods
- background jobs that mutate state without a transition request
- broad `except Exception` that marks work as successful

### 2.2 Database-Backed Truth

If a fact controls execution, safety, state, review, verification, cost, or branch behavior, it must be stored in the database.

Markdown, logs, command output, and model responses are artifacts. They may inform decisions, but they do not become truth until imported, validated, and linked to a database record.

### 2.3 Narrow Context

Agent prompts and packets must contain only the context required for the current role and work unit.

Large project documents are referenced by stable IDs and hashes. They are not copied into every prompt unless the role needs them.

### 2.4 State Is Special

State changes are never ordinary field assignments.

All important state changes must use:

```text
TransitionRequest -> PolicyDecision -> StateMachine -> StateTransition + AuditEvent
```

### 2.5 Agents Are Untrusted Contributors

Codex, local models, Claude imports, and any future LLM output are treated as claims.

Claims must pass:

- schema validation
- role validation
- policy validation
- evidence validation
- state-machine validation

## 3. Project Code Style

### 3.1 TypeScript

- Use clear names over abbreviations and strict TypeScript types.
- Prefer simple functions, small modules, and explicit request/result types.
- Define interfaces for service, query, policy, repository, and runner ports.
- Keep comments rare and useful.
- Raise domain exceptions for expected failures.
- Never swallow errors that should create `SafetyEvent`, `IncidentReport`, `JobFailure`, or `VerificationRun` records.

### 3.2 NestJS And TypeORM

- Controllers and control-room actions are thin.
- TypeORM entities define data shape and relations; reviewed migrations define
  PostgreSQL constraints and indexes.
- Application use cases perform mutations.
- Query services perform read queries.
- Policies make allow/deny decisions.
- Runners interact with external tools.
- State machine code is the only code that applies state transitions.

### 3.3 PostgreSQL

Use database constraints for invariants that must survive bugs, worker retries, and concurrent jobs.

Required by default:

- UUID primary keys.
- Unique constraints for human-readable keys scoped by project.
- Foreign keys for ownership and lineage.
- Check constraints for bounded enums where practical.
- Indexes for queue leasing, state dashboards, artifact lookup, and audit timelines.

Avoid:

- business-critical facts stored only in JSON fields.
- unindexed dashboard filters.
- long transactions around external processes.

## 4. Naming Rules

### 4.1 Modules And Packages

Use plural domain names for Nest modules and workspace packages unless the
domain is naturally singular.

Examples:

- `projects`
- `planning`
- `components`
- `execution`
- `verification`
- `reviews`
- `policy`
- `state`
- `gitops`
- `artifacts`
- `documents`
- `audit`

### 4.2 Services

Service functions use verb-first names:

- `create_project_from_spec`
- `decompose_roadmap`
- `request_state_transition`
- `lease_next_job`
- `record_verification_result`
- `generate_context_packet`
- `create_pull_request`

### 4.3 Selectors

Selector functions use read-oriented names:

- `get_project_overview`
- `list_ready_component_work`
- `find_open_revision_tasks`
- `load_context_packet_sources`

### 4.4 Policies

Policy functions return structured decisions, not booleans:

- `can_start_component_work`
- `can_export_external_packet`
- `can_create_pr`
- `can_promote_claude_import`

## 5. Error Handling

Expected domain failures should become typed exceptions or structured result objects.

Examples:

- `PolicyDenied`
- `InvalidTransition`
- `MissingEvidence`
- `SchemaValidationFailed`
- `RunnerTimeout`
- `UnsafeCommandDenied`
- `RepositoryStateMismatch`

Unexpected failures must be recorded with enough context to reproduce the failure without exposing secrets.

## 6. Logging And Audit

Logs are operational traces. Audit records are product facts.

Every meaningful automated action must produce an audit event:

- who or what initiated it
- what was requested
- what policy decided
- what evidence was used
- what state changed
- what artifacts were created

Raw logs may be rotated. Audit records must remain queryable.

## 7. Idempotency

Worker actions must be safe to retry.

Every job handler must define:

- idempotency key
- retry policy
- timeout policy
- lock or lease strategy
- duplicate-output handling
- failure record behavior

External side effects require recorded external IDs when available.

Examples:

- PR URL
- branch name
- worktree path
- command run ID
- artifact hash

## 8. Forbidden Patterns

ADO implementation must not use:

- direct status assignment outside the state machine
- automatic merge
- direct push to `main` or `integrate`
- production secret capture
- model signals for core orchestration
- unbounded agent context packets
- model output accepted without schema validation
- long DB transactions around CLI calls
- shell command construction from unvalidated user text
- deleting worktrees or branches without a recorded policy decision

## 9. Completion Standard

A code change is complete only when:

- behavior is implemented in the correct layer
- state changes go through the state machine
- policy checks exist where needed
- database constraints protect critical invariants
- tests cover success and important failure cases
- artifacts and audit records are produced where required
- generated documents remain derived from database state
