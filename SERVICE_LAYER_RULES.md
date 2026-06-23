# ADO Service Layer Rules

This document defines how ADO business logic is implemented.

## 1. Rule

All orchestration behavior lives in application use cases, policies, query
services, runners, and the state machine.

Controllers, Next.js UI actions, Worker CLI entry points, and agent outputs
must not contain hidden business logic.

## 2. Layer Contracts

### 2.1 Entities And Repositories

TypeORM entities and repositories define:

- fields
- relationships
- indexes
- constraints
- simple derived properties

They must not:

- call external services
- run commands
- create PRs
- apply orchestration state transitions
- generate agent prompts

### 2.2 Query Services

Query services read data.

They may:

- build typed repository/SQL queries
- apply dashboard filters
- prefetch related records
- return read DTOs

They must not:

- mutate rows
- create artifacts
- enqueue jobs
- call runners

### 2.3 Application Use Cases

Application use cases execute use cases.

They may:

- create and update domain records
- call policies
- call selectors
- create artifacts
- enqueue jobs
- request state transitions
- call runners through explicit adapter boundaries

They must:

- be idempotent where jobs can retry
- record audit events for important actions
- use transactions for related database changes
- return structured results

### 2.4 Policies

Policies decide whether something is allowed.

They must return structured decisions:

```text
allowed: true|false
code: stable_machine_code
reasons: list
required_evidence: list
```

Policies must not mutate business state.

### 2.5 State Machine

The state machine is the only layer that applies important state transitions.

It must:

- validate current state
- validate target state
- validate actor role
- validate policy decision
- validate required evidence
- write StateTransition
- write AuditEvent
- apply the state change in the same transaction

### 2.6 Runners

Runners execute outside systems:

- Codex CLI
- local models
- shell commands
- tests
- Git
- GitHub

Runners return structured results. They do not decide whether the result is acceptable.

## 3. Standard Service Shape

Use explicit request/result objects when a service has more than a few inputs.

TypeScript example:

```ts
interface StartComponentWorkRequest {
  componentWorkId: string;
  actorId: string;
  reason: string;
}

interface StartComponentWorkResult {
  componentWorkId: string;
  jobId: string;
  transitionId: string;
}
```

Services should be callable from:

- UI
- Nest Worker CLI operation
- worker
- tests

## 4. Important Use Cases

ADO v1 should implement these use cases as services:

- create project from project spec
- generate project constraint questionnaire
- approve project constraint profile
- import roadmap
- decompose roadmap into Feature Units
- approve Feature Units
- create Component Work
- create branch and worktree
- generate ContextPacket
- run Codex implementation
- run verification
- run local review council
- run Codex arbiter
- create revision tasks
- apply revision loop
- create PR to integrate
- record human verification
- close Feature Unit

## 5. Project Constraint Generation

Project-specific constraints are generated and managed by services.

Required flow:

```text
Project intake answers
-> ProjectConstraintProfile draft
-> generated project constraint docs
-> human review
-> approved ProjectConstraintProfile
-> ContextPacket inclusion by scope
```

ADO common constraints always win over project constraints.

Precedence:

```text
ADO common constraints
> project constraints
> feature unit constraints
> component work constraints
> agent suggestions
```

If a lower layer conflicts with a higher layer, the lower layer is rejected or marked as requiring human override.

## 6. Worker Job Pattern

Every worker job handler follows:

```text
1. lease job
2. heartbeat
3. load minimal DB context
4. run pre-policy check
5. create run record
6. execute runner or service
7. validate output schema when applicable
8. persist artifacts
9. request state transition
10. terminalize attempt/job through services
11. publish approved JobOutboxEvents after commit
```

The job handler must not skip policy and state-machine services.

`WORKER_EXECUTION_CONTRACT.md` and `JOB_HANDLER_CATALOG.md` define the
canonical detailed handler contract. This sequence is its service-layer summary.

## 7. Revision Loop Rules

Revision is work, not a comment.

Every accepted finding that blocks progress must become a `RevisionTask`.

RevisionTask must include:

- source finding
- severity
- affected component
- expected fix
- verification required
- status

Codex Implementer receives revision tasks through a new ContextPacket, not through loose chat history.

## 8. Human Decision Rules

Human decisions are first-class records.

Required fields:

- actor
- decision type
- target object
- decision
- reason
- timestamp
- source screen or command

Manual override requires a reason and creates an audit event.

## 9. Service Test Requirements

Every service that changes state must test:

- successful path
- policy denial
- invalid transition
- missing evidence
- idempotent retry where applicable
- audit creation
- transaction behavior for failure

## 10. Service Completion Standard

A service is production-ready when:

- inputs are validated
- output is structured
- state changes use state machine
- policy checks are explicit
- audit events are written
- failures are recorded
- tests cover the contract
